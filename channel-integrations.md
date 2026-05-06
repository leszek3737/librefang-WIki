# Channel Integrations

# Channel Integrations

The `librefang-channels` crate is the messaging backbone that connects external chat platforms (Telegram, Discord, Slack, Bluesky, WhatsApp, and many others) to the LibreFang agent kernel. It provides a uniform adapter abstraction, message dispatch pipeline, input sanitization, debouncing, group-trigger detection, and content-aware attachment enrichment.

## Architecture Overview

```mermaid
graph TD
    subgraph "External Platforms"
        TG[Telegram]
        DC[Discord]
        BS[Bluesky]
        WA[WhatsApp]
        OT[...others]
    end

    subgraph "librefang-channels"
        CA[ChannelAdapter trait]
        BM[BridgeManager]
        MDB[MessageDebouncer]
        RL[RateLimiter]
        SA[InputSanitizer]
        AR[AgentRouter]
        AE[AttachmentEnrichment]
        TO[ThreadOwnership]
    end

    subgraph "Kernel (librefang-api)"
        CBH[ChannelBridgeHandle]
        AK[Agent Loop]
    end

    TG & DC & BS & WA & OT -->|ChannelMessage stream| CA
    CA -->|start()| BM
    BM --> MDB
    MDB -->|flush| SA
    SA -->|clean| AR
    AR -->|route to agent| CBH
    CBH -->|send_message_streaming| AK
    AK -->|ReplyEnvelope| CBH
    CBH -->|send()| CA
    AE -->|ContentBlock| CBH
    TO -->|dedup claims| BM
```

## Core Abstractions

### `ChannelAdapter` Trait

Defined in `crate::types`, this is the interface every platform adapter implements. Key methods:

| Method | Purpose |
|--------|---------|
| `name()` | Adapter identifier string (e.g. `"telegram"`, `"bluesky"`) |
| `channel_type()` | Returns the `ChannelType` enum variant |
| `start()` | Returns a `Stream<Item = ChannelMessage>` of inbound messages |
| `send(user, content)` | Deliver an outbound message to a user |
| `stop()` | Graceful shutdown |
| `create_webhook_routes()` | Optional — returns `(Router, Stream)` for webhook-based platforms |
| `typing_events()` | Optional — returns a receiver for typing indicator events |

### `ChannelMessage`

The unified message envelope all adapters emit. Fields include:

- `channel` — `ChannelType` identifying the source platform
- `platform_message_id` — Platform-native message ID (URI, message ID, etc.)
- `sender` — `ChannelUser` with `platform_id`, `display_name`, optional `librefang_user`
- `content` — `ChannelContent` enum (Text, Command, Image, File, Voice, Video, Location, etc.)
- `is_group` — Whether the message came from a group/chat context
- `thread_id` — For forum-topic or thread-scoped routing
- `metadata` — `HashMap<String, serde_json::Value>` carrying platform-specific data (URIs, reply refs, account_id for multi-bot routing)

### `ChannelContent`

A comprehensive enum covering every content type the system handles:

```rust
pub enum ChannelContent {
    Text(String),
    Command { name: String, args: Vec<String> },
    Image { url, caption, mime_type },
    File { url, filename },
    FileData { filename, data, mime_type },  // In-memory binary
    Voice { url, duration_seconds, caption },
    Video { url, caption, duration_seconds, mime_type },
    Audio { url, caption, duration_seconds, mime_type },
    Location { lat, lon },
    Interactive { text, buttons },
    ButtonCallback { action, data },
    EditInteractive { message_id, text, buttons },
    DeleteMessage { message_id },
    Animation { url, caption, duration_seconds },
    Sticker { file_id },
    MediaGroup { items },
    Poll { question, options, is_anonymous },
    PollAnswer { poll_id, option_ids },
}
```

The helper `content_to_text()` collapses any variant into a human-readable string for logging and debouncer merging.

## `BridgeManager`

The central orchestrator. It owns all running adapters and manages the dispatch pipeline.

### Construction

```rust
let manager = BridgeManager::new(kernel_handle, agent_router)
    .with_sanitizer(&sanitize_config)
    .with_journal(journal);
```

- `kernel_handle: Arc<dyn ChannelBridgeHandle>` — the kernel-side API
- `agent_router: Arc<AgentRouter>` — resolves messages to the correct agent
- Optional `MessageJournal` for crash recovery of in-flight messages

### `start_adapter()`

Registers an adapter and spawns a dispatch loop:

1. **Webhook preference**: If the adapter provides `create_webhook_routes()`, those routes are collected for mounting on the main API server under `/channels/{name}/webhook`. Otherwise falls back to adapter-managed polling/sockets.
2. **Download directory cleanup**: Sweeps stale files (>24h) from the upload directory once on first adapter start.
3. **Dispatch mode selection**: Reads `message_debounce_ms` from channel overrides:
   - **Fast path (debounce_ms == 0)**: Each inbound message immediately spawns a concurrent dispatch task.
   - **Debounce path**: Messages are buffered per-sender and merged on flush triggers.
4. **Concurrency control**: A semaphore (default 32 permits) caps concurrent dispatch tasks to prevent unbounded memory under burst traffic.

### `push_message()`

Proactive outbound messages (used by `POST /api/agents/:id/push`):

```rust
manager.push_message("telegram", "chat_id_123", "Hello!", None).await
```

Routes through `ChannelBridgeHandle::send_channel_push`.

### Shutdown

`stop()` signals all dispatch loops, stops each adapter (releasing ports/connections), and awaits all tasks.

## `ChannelBridgeHandle` Trait

This trait is the contract between `librefang-channels` and the kernel. Defined in the channels crate to avoid circular dependencies; implemented in `librefang-api` on the real kernel.

### Core Methods

| Method | Description |
|--------|-------------|
| `send_message(agent_id, message)` | Basic text send to an agent, returns response string |
| `send_message_with_blocks(agent_id, blocks)` | Multimodal send (text + images) |
| `send_message_with_sender(agent_id, message, sender)` | Send with `SenderContext` (identity + channel info) |
| `send_message_streaming(agent_id, message)` | Returns `mpsc::Receiver<String>` for progressive token display |
| `send_message_streaming_with_sender_status(...)` | Full streaming + terminal status via oneshot channel |
| `find_agent_by_name(name)` | Agent lookup |
| `list_agents()` | Running agents list |
| `spawn_agent_by_name(manifest_name)` | Dynamic agent creation |

### Channel Configuration Access

| Method | Description |
|--------|-------------|
| `channel_overrides(channel_type, account_id)` | Per-channel `ChannelOverrides` from config |
| `agent_channel_overrides(agent_id)` | Per-agent overrides from `agent.toml` |
| `get_agent_group_trigger_patterns(agent_id)` | Regex patterns for group mention detection |
| `effective_channels_download_dir()` | File download directory (config or temp fallback) |
| `channels_download_max_bytes()` | Max download size limit |

### Extended Operations

The trait includes a large surface area for features like session management (`reset_session`, `reboot_session`, `compact_session`), model selection (`set_model`, `list_models_text`), RBAC (`authorize_channel_user`), automation (`list_workflows_text`, `run_workflow_text`, `manage_schedule_text`, `resolve_approval_text`), and observability (`session_usage`, `budget_text`, `record_delivery`).

Most methods have sensible defaults (stub responses or no-ops) so adapter implementations only need to override what they support.

## Message Debouncing

The `MessageDebouncer` batches rapid-fire messages from the same sender (common in Telegram where users send multiple short messages instead of one long one).

### Configuration (via `ChannelOverrides`)

| Field | Default | Description |
|-------|---------|-------------|
| `message_debounce_ms` | 0 (disabled) | Delay before flushing buffered messages |
| `message_debounce_max_ms` | 30000 | Hard cap — flush after this time regardless |
| `message_debounce_max_buffer` | 64 | Max messages before immediate flush |

### Flush Triggers

1. **Debounce timer expiry** — `debounce_ms` after the last message
2. **Max timer expiry** — `debounce_max_ms` after the first message
3. **Buffer full** — `max_buffer` messages accumulated
4. **Typing stop** — Sender stopped typing (when `typing_events()` is available)
5. **Stream end** — Adapter stream closed; all buffers drained

### Merging Logic

When multiple messages are buffered:

- **All same type (non-command)**: Text parts joined with `\n`
- **All same command name**: Arguments concatenated into a single `Command`
- **Mixed types**: Everything collapsed to `\n`-separated text

Image blocks are collected across all buffered messages and forwarded alongside the merged text.

### Backpressure

The flush channel is bounded at 1024 entries (`FLUSH_CHANNEL_CAP`). If the dispatcher stalls (rate-limited adapter, paused agent), the debouncer drops buffered messages and logs a warning rather than growing unbounded — this prevents OOM under sustained burst traffic (issue #3580).

Double-fire guards (issue #3742) ensure that if both a manual flush and a timer task enqueue the same key, the second `drain()` call returns `None` harmlessly.

## Group Message Processing

Group messages require careful filtering to avoid the bot responding to every message in a busy chat.

### Group Policy (`GroupPolicy` enum)

| Variant | Behavior |
|---------|----------|
| `Ignore` | All group messages dropped |
| `CommandsOnly` | Only slash-commands processed |
| `MentionOnly` | Only mentions/triggers processed |
| `All` | All messages processed |

### Trigger Detection

For `MentionOnly` mode, the system checks (in order):

1. **`was_mentioned` metadata** — Set by the adapter when the platform reports a direct mention
2. **Bot name substring** — Configured trigger names appear in the message text
3. **`group_trigger_patterns`** — Regex patterns from agent config, compiled and cached via `RegexSet` with a `DashMap` cache keyed on the pattern string
4. **Reply to bot** — Message is a reply to a previous bot message
5. **LLM classification** — `classify_reply_intent()` sends the message to a lightweight LLM call (configurable model)

### Vocative / Addressee Guard

When the `LIBREFANG_GROUP_ADDRESSEE_GUARD=on` environment variable is set, the system performs positional vocative analysis to avoid false positives:

- **`leading_vocative_name(text)`** — Detects opening patterns like `"Caterina, chiedi..."` where the turn is addressed to another participant
- **`is_vocative_trigger(text, pattern)`** — Strict positional matching: the trigger pattern must appear at turn start or after sentence boundaries, and must NOT be preceded by another capitalized vocative
- **`is_addressed_to_other_participant(text, participants, agent_name)`** — Cross-references the detected vocative against the group roster

This prevents the bot from responding when someone says `"Caterina, ask the Bot about X"` — the bot's trigger word appears, but the turn is addressed to Caterina.

## Command Routing

### Slash Commands

Messages starting with `/` are parsed into `ChannelContent::Command { name, args }`. Built-in commands are dispatched to `BridgeManager` handlers that call `ChannelBridgeHandle` methods. Unknown commands are forwarded to the agent as text.

### Command Access Control

Three levels of command restriction (via `ChannelOverrides`):

1. **`disable_commands: true`** — Blocks all commands
2. **`allowed_commands`** (whitelist) — Only listed commands are processed; others forwarded to agent
3. **`blocked_commands`** (blacklist) — Listed commands blocked; all others allowed

Config entries can include or omit the leading `/` (both `"agent"` and `"/agent"` match).

Blocked commands are reconstructed as quoted text (`"/agent admin"`) and forwarded to the agent so the user's original input isn't silently swallowed.

## Attachment Enrichment

The `attachment_enrich` module provides content-aware extraction for channel-downloaded files so the LLM receives useful text instead of just a file path.

### `enrich_saved_file(saved_path, media_type, filename) -> Vec<ContentBlock>`

Called after the bridge streams a download to disk. Returns additional `ContentBlock::Text` entries inserted *before* the existing `[File: name] saved to /path` block — tools that need the raw bytes still work.

| Input Type | Output |
|------------|--------|
| `application/pdf` | `[Attached PDF: name (N bytes)]\n\n<extracted text>` |
| `text/*`, code extensions | `[Attached file: name (N bytes[, truncated])]\n\n<file text>` |
| Images, audio, video | Empty vec (handled elsewhere or intentionally skipped) |
| Unknown binary | Empty vec (caller emits path block alone) |

### PDF Detection

PDFs often arrive with incorrect MIME types (Telegram uses `application/octet-stream`). The detection cascade:

1. **MIME check** — `application/pdf` matches directly
2. **Extension fallback** — `.pdf` extension when MIME is ambiguous (`octet-stream`, empty, `binary/octet-stream`)
3. **Magic bytes** — Checks for `%PDF-` header when both MIME and extension are unhelpful

Explicit MIME types (e.g. `image/png` on a `.pdf`-named file) are never overridden by heuristics.

### PDF Extraction Safety

PDF parsing (`pdf_extract`/`lopdpdf`) can panic on malformed or encrypted documents. The module wraps extraction in `std::panic::catch_unwind(AssertUnwindSafe(...))` to isolate panics and surface a `[Could not extract text: PDF parser panicked]` note instead of crashing the bridge.

### Text Truncation

`MAX_ENRICHED_TEXT_CHARS` (200,000 chars) mirrors the dashboard upload limit. Oversized text is truncated with a human-readable marker:

```
[…file truncated at 200K chars; content continues beyond this point…]
```

### Text-Like Detection

`is_text_like(content_type, filename)` mirrors the dashboard's upload detection. Falls back to extension matching when MIME is generic — supports 70+ extensions covering source code (`.rs`, `.py`, `.go`, `.ts`, `.jsx`), data formats (`.json`, `.yaml`, `.toml`, `.csv`), config files (`.env`, `.ini`, `.conf`), and markup (`.md`, `.html`, `.css`).

## Bluesky Adapter

An example adapter implementing the AT Protocol (Bluesky) integration.

### Authentication

- Uses app passwords (not account passwords) via `com.atproto.server.createSession`
- Returns `accessJwt`, `refreshJwt`, and `did`
- Sessions cached in `Arc<RwLock<Option<BlueskySession>>>`
- Auto-refreshes 5 minutes before expiry (sessions last ~2 hours)
- Falls back to new session creation on refresh failure

### Inbound Messages

Polls `app.bsky.notification.listNotifications` every 5 seconds. Only processes `mention` and `reply` reasons. Parses commands from text starting with `/`:

```
/status check  →  Command { name: "status", args: ["check"] }
hello there    →  Text("hello there")
```

The `reply_ref` metadata is extracted for thread-aware replies.

### Outbound Messages

Creates records via `com.atproto.repo.createRecord` with the `app.bsky.feed.post` lexicon. Long messages are split at 300 grapheme clusters using `split_message()`. Non-text content types fall back to a placeholder `"(Unsupported content type)"`.

### Multi-Bot Support

`with_account_id(account_id)` injects the account ID into message metadata for multi-bot routing on shared channels.

## `ReplyEnvelope`

Outbound responses use a two-channel envelope:

```rust
pub struct ReplyEnvelope {
    pub public: Option<String>,       // Visible to the chat
    pub owner_notice: Option<String>, // Private DM to operator
}
```

- `from_public(s)` — public reply only
- `silent()` — no output at all
- `public_or_empty()` — legacy compatibility for adapters not yet routing owner notices

## Thread Ownership

`ThreadOwnershipRegistry` prevents duplicate replies in shared group threads where multiple agents might respond to the same message. Agents claim a thread before responding; subsequent claims are rejected.

## Webhook Route Collection

Adapters that use webhooks (Telegram, Slack, etc.) provide `create_webhook_routes()` returning `Option<(axum::Router, Stream)>`. `BridgeManager` collects these routes and exposes `take_webhook_router()` which nests them under `/{adapter_name}`. The caller mounts the combined router under `/channels` on the main API server, without auth middleware (adapters handle their own signature verification).

## Message Journal

Optional crash-recovery journal for in-flight messages:

- `with_journal(journal)` attaches a journal to the bridge
- `recover_pending()` returns messages that were being processed when the daemon crashed
- `compact_journal()` flushes and compacts on shutdown