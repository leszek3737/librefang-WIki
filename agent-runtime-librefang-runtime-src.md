# Agent Runtime — librefang-runtime-src

# Agent Runtime — `librefang-runtime-src`

The agent runtime is the execution core of LibreFang. It houses the agent loop (the iterative LLM–tool–response cycle), the A2A interoperability layer, per-turn context loading, message-history management, and the end-of-turn finalization pipeline that persists memories and folds stale tool results.

## Architecture Overview

```mermaid
graph TD
    subgraph "Agent Loop (run_agent_loop / run_agent_loop_streaming)"
        PROMPT[Prompt Assembly] --> LLM[LLM Completion]
        LLM -->|tool_calls| TOOLS[Tool Execution]
        LLM -->|text| END[End-Turn Finalize]
        TOOLS --> LLM
    end

    subgraph "Per-Turn Context"
        CTX[context.md Loader] --> PROMPT
    end

    subgraph "End-Turn Pipeline"
        END --> FOLD[Fold Stale Tool Results]
        FOLD --> MEM[Persist Session + Memory]
        MEM --> PM[Proactive Memory auto_memorize]
        PM --> HOOKS[Hook Callbacks]
    end

    subgraph "A2A Interop"
        AC[A2aClient] -->|discover| CARD[/.well-known/agent.json]
        AC -->|send_task| EXT[External A2A Agent]
        STORE[A2aTaskStore] -->|insert/get/update| DB[(SQLite + In-Memory)]
        BUILD[build_agent_card] --> MANIFEST[AgentManifest → AgentCard]
    end

    LLM -.->|response| STORE
```

---

## Module Index

| Module | File | Purpose |
|--------|------|---------|
| A2A Protocol | `a2a.rs` | Cross-framework agent discovery, task exchange, and SSRF-safe HTTP client |
| Agent Context | `agent_context.rs` | Per-turn `context.md` loading with caching and symlink rejection |
| Agent Loop — End Turn | `agent_loop/end_turn.rs` | Post-turn finalization: memory persistence, history folding, proactive memory |
| Agent Loop — History | `agent_loop/history.rs` | Message-history trim-cap resolution and clamping |
| Agent Loop — Message | `agent_loop/message.rs` | Message classifiers, sanitizers, accumulated-text buffer, safe trimming |

---

## A2A Protocol — `a2a.rs`

Implements Google's A2A specification for cross-framework agent interoperability via **Agent Cards** and **Task-based coordination**.

### Core Types

**`AgentCard`** — A JSON capability manifest served at `/.well-known/agent.json`. Built from an `AgentManifest` via `build_agent_card()`, which converts tool names into A2A skill descriptors. Fields include `name`, `description`, `url`, `capabilities` (streaming, push notifications, state transition history), and `skills`.

**`A2aTask`** — The unit of work exchanged between agents. Contains:
- `id` — unique task identifier
- `session_id` — optional conversation continuity key
- `status` — an `A2aTaskStatusWrapper` (see below)
- `messages` — `Vec<A2aMessage>` conversation transcript
- `artifacts` — `Vec<A2aArtifact>` produced outputs
- `agent_id` / `caller_a2a_agent_id` — routing and audit fields

**`A2aTaskStatus`** — Enum with states: `Submitted`, `Working`, `InputRequired`, `Completed`, `Cancelled`, `Failed`.

**`A2aTaskStatusWrapper`** — Accepts both the bare string form (`"completed"`) and the object form (`{"state": "completed", "message": null}`) used by different A2A implementations. Use `.state()` to extract the underlying status regardless of encoding.

### `A2aTaskStore` — Task Lifecycle Tracking

An in-memory + SQLite-backed store for tracking A2A tasks through their lifecycle.

**Construction:**
- `A2aTaskStore::new(max_tasks)` — in-memory only, default 24h TTL
- `A2aTaskStore::with_ttl(max_tasks, duration)` — custom TTL
- `A2aTaskStore::with_persistence(max_tasks, db_path)` — SQLite-backed, loads survivors on startup

**Eviction policy** (applied lazily on `insert`):
1. TTL sweep: remove all tasks whose `updated_at` exceeds the TTL, regardless of state
2. Capacity eviction: if still at capacity, evict the oldest terminal-state task (`Completed`/`Failed`/`Cancelled`), then fall back to the oldest task overall

**Persistence model:**
- Every mutation is written to SQLite (`db_upsert`) before the in-memory map is updated
- On boot, `db_load_into_memory` loads the `max_tasks` most recently updated rows; older rows remain queryable via the `get()` SQLite fallback
- Rows older than 7 days are pruned on startup
- Persistence is **best-effort** — SQLite write failures log a warning and degrade to in-memory-only

**Key methods:**
- `insert(task)` — persist + insert with eviction
- `get(task_id)` — fast-path in-memory lookup, falls back to SQLite for evicted tasks
- `update_status(task_id, status)` — transition a task's state
- `complete(task_id, response, artifacts)` — append response message, mark `Completed`
- `fail(task_id, error_message)` — append error, mark `Failed`
- `cancel(task_id)` — mark `Cancelled`

The `a2a_tasks_v2` SQLite schema stores `messages` and `artifacts` as full JSON arrays (preserving chronological ordering and interleaved user/agent roles). The earlier v1 schema that split messages into `input`/`output` columns is dropped unconditionally on startup.

### `A2aClient` — Outbound A2A Communication

Discovers and interacts with external A2A agents. Each outbound request builds a fresh `reqwest::Client` with:

1. **SSRF protection** — `web_fetch::check_ssrf` resolves DNS once, validates addresses against private-IP ranges, and returns pinned addresses
2. **DNS pinning** — the validated IPs are installed via `ClientBuilder::resolve` to prevent DNS-rebinding TOCTOU attacks (#3563)
3. **Redirect blocking** — `Policy::none()` prevents all redirects; 3xx responses are explicitly rejected (#3782)
4. **Body-size caps** — `MAX_AGENT_CARD_BYTES` (256 KiB) for discovery, `MAX_A2A_TASK_BYTES` (1 MiB) for task RPCs. Enforced via streaming chunk reads, not `Response::json()` (#3785)
5. **Transport decompression disabled** — prevents gzip/brotli bombs from evading the size cap

**Methods:**
- `discover(url)` — fetch `/.well-known/agent.json`, return `AgentCard`
- `send_task(url, message, session_id)` — JSON-RPC `tasks/send` to an external agent
- `get_task(url, task_id)` — JSON-RPC `tasks/get` to poll task status

### URL Canonicalization

`canonicalize_a2a_url(url)` normalizes peer URLs for trust-list comparison by lowercasing scheme and host, stripping fragments, empty query strings, and default ports (80 for HTTP, 443 for HTTPS). This ensures `https://Example.COM:443/` and `https://example.com` match the same trust entry.

### Boot-Time Discovery

`discover_external_agents(agents)` is called during kernel boot to fetch Agent Cards from all configured `ExternalAgent` entries and populate the known-agents list.

---

## Agent Context — `agent_context.rs`

Loads per-turn `context.md` files that external tools (cron jobs, scripts) update between turns. This replaced the old behavior of reading the file once and caching it for the session lifetime.

### File Resolution

```
{workspace}/.identity/context.md   ← preferred (new layout)
{workspace}/context.md             ← fallback (legacy)
```

`resolve_context_path` checks the `.identity/` location first; if absent, falls back to the workspace root.

### Loading Behavior

**`load_context_md(workspace, cache_context)`** (sync) and **`load_context_md_async(workspace, cache_context)`** (async):

- When `cache_context = false` (default): re-reads from disk every turn
- When `cache_context = true`: returns the first successful read on all subsequent calls
- On read failure after a prior success: falls back to cached content with a `warn!`
- Returns `None` when no file exists or content is whitespace-only

### Security Constraints

- **Symlink rejection**: uses `symlink_metadata` and explicitly refuses to follow symlinks — prevents an attacker from pointing `context.md` at `/etc/passwd` and having its contents injected into the LLM prompt
- **Size cap**: `MAX_CONTEXT_BYTES` = 32 KiB. Reads are capped at this limit; the read itself stops at `cap + 4` bytes (UTF-8 boundary slop) so multi-GB files are never fully loaded
- **UTF-8 boundary trimming**: if the cap lands mid-codepoint, bytes are truncated to the last valid UTF-8 boundary. Files with zero valid UTF-8 prefix surface as I/O errors so the caller falls back to cached content

---

## Agent Loop — End Turn — `agent_loop/end_turn.rs`

Handles post-turn finalization after the LLM produces a text response (as opposed to tool calls).

### Retry Classification

`classify_end_turn_retry` inspects the response and determines whether the loop should retry rather than finalize:

| Condition | Retry Variant | Trigger |
|-----------|---------------|---------|
| Empty text + no tool calls, iteration 0 or zero tokens | `EmptyResponse` | LLM returned nothing usable |
| Non-empty text, no tool calls, looks like hallucinated action verb | `HallucinatedAction` | Model described an action it didn't take |
| Non-empty text, no tool calls, user message has action intent | `ActionIntent` | User asked for action but model just talked |

Each retry type is gated to fire at most once per turn via `hallucination_retried` / `action_nudge_retried` flags.

### Empty-Response Fallback

`finalize_end_turn_text` handles the case where the final EndTurn iteration produces empty text:
1. If `accumulated_text` from intermediate tool-use iterations is non-empty, use it as the response
2. If tools were executed, return a generic completion notice
3. Otherwise, return an error message suggesting the user retry

### Finalization Pipeline

`finalize_successful_end_turn` executes after the retry guard passes:

1. **Append assistant response** to session messages
2. **Prune heartbeat turns** (for autonomous agents)
3. **Persist session** to the memory substrate (skipped for fork/incognito turns)
4. **Write episodic memory** — sanitized user/response pair as a `[Past exchange]` block (skipped for fork/incognito)
5. **Context engine `after_turn`** callback
6. **Proactive memory `auto_memorize`** — extracts memories from the new message slice
7. **Fire hooks** — `AgentLoopEnd` event via the hook registry
8. **Build `AgentLoopResult`** with usage stats, directives, memories, and the `skill_evolution_suggested` flag (set when ≥5 tool calls occurred)

Fork and incognito turns skip persistence, memory writes, and context engine updates to prevent ephemeral conversations from leaking into long-term state.

### History Folding

`maybe_fold_stale_tool_results` runs a shared fold pass that summarizes old tool results to compress the context window. It:
- Fast-paths when `fold_after_turns == 0` or the message count is too low
- Calls `history_fold::fold_stale_tool_results` on the working copy
- Replays fold rewrites onto the durable `session.messages` (matched by `tool_use_id`) so subsequent turns don't re-summarize

### Proactive Memory Gating

`gated_proactive_memory_for_retrieve` and `gated_proactive_memory_for_memorize` check per-agent overrides in `manifest.proactive_memory` against the global config. This allows sub-agents (e.g., cron tasks) to opt out of automatic memory retrieval or extraction without disabling the feature globally.

---

## Agent Loop — History — `agent_loop/history.rs`

Manages the message-history trim cap that prevents conversations from growing without bound.

### Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `DEFAULT_MAX_HISTORY_MESSAGES` | 60 | ~7–10 real turns; keeps prompt-cache prefix stable |
| `MAX_HISTORY_MESSAGES` | 500 | Hard ceiling; operator overrides above this are clamped |
| `MIN_HISTORY_MESSAGES` | 4 | Floor; values below defeat the safe-trim repair heuristic |

### Resolution

`resolve_max_history(manifest, opts)` determines the effective cap:
1. `manifest.max_history_messages` (per-agent override)
2. `opts.max_history_messages` (kernel config override)
3. `DEFAULT_MAX_HISTORY_MESSAGES` (fallback)

The result is clamped into `[MIN_HISTORY_MESSAGES, MAX_HISTORY_MESSAGES]` with a warning log for out-of-range values.

---

## Agent Loop — Message — `agent_loop/message.rs`

Message-shape utilities used across the agent loop.

### Accumulated Text Buffer

`push_accumulated_text` appends intermediate text (emitted alongside tool_use blocks) to a fallback buffer capped at `ACCUMULATED_TEXT_MAX_BYTES` (64 KiB). Once capped, the buffer is sealed and further appends are dropped. This prevents unbounded memory growth in long-running autonomous agents.

### Classifiers

| Function | Purpose |
|----------|---------|
| `is_no_reply(text)` | Delegates to `silent_response::is_silent_response` — detects `NO_REPLY`, `[no reply needed]`, etc. |
| `is_progress_text_leak(text)` | Catches short ellipsis-terminated preambles the model emits before a tool call that never arrives (≤120 chars, ends with `...` or `…`) |

### Memory Sanitization

`sanitize_for_memory(text)` strips channel-envelope prefixes (e.g., `[Group message from X]`) from messages before persisting them as episodic memory. Returns `None` when nothing meaningful remains, preventing empty half-exchanges from being stored.

### Image Stripping

- `strip_prior_image_data` — removes image data from messages before the current turn (reduces token cost)
- `strip_processed_image_data` — removes images from already-processed turns

### Tool-Result Sanitization

Tool result content is sanitized through injection-marker removal and dynamic truncation (via `context_budget::truncate_tool_result_dynamic`) to keep tool outputs within the context budget.

### Safe Trimming

`safe_trim_messages` truncates the message history to the resolved cap, then validates the trimmed window via `validate_and_repair` (synthesizes a minimal user message when fewer than 2 messages survive).

---

## Cross-Module Data Flow

### Typical Agent Turn

```
1. Kernel receives inbound message
2. load_context_md()          → reads context.md into prompt assembly
3. resolve_max_history()       → determines trim cap
4. safe_trim_messages()        → trims session history
5. Prompt assembly             → system prompt + context + memories + history
6. LLM completion              → tool_calls or text
7. [If tool_calls] → execute tools → goto step 6
8. classify_end_turn_retry()   → empty/hallucination/action-intent check
9. finalize_end_turn_text()    → resolve final response text
10. finalize_successful_end_turn():
    a. Append response to session
    b. Prune heartbeat turns
    c. Persist session to memory substrate
    d. Write episodic memory (sanitized)
    e. Context engine after_turn
    f. Proactive memory auto_memorize
    g. maybe_fold_stale_tool_results()
    h. Fire AgentLoopEnd hooks
11. Return AgentLoopResult
```

### A2A Outbound Task Flow

```
1. Agent calls tool_a2a_send
2. A2aClient.send_task(url, message, session_id)
3. build_client_for_url():
   a. check_ssrf() → validate + pin DNS
   b. Policy::none() → block redirects
   c. Disable decompression
4. POST JSON-RPC tasks/send (body capped at 1 MiB)
5. Parse response → A2aTask
6. A2aTaskStore.insert(task) → in-memory + SQLite upsert
```

### A2A Task Polling

```
1. Poller calls tasks/get
2. A2aTaskStore.get(task_id):
   a. Fast path: in-memory HashMap lookup
   b. Slow path: SQLite SELECT for evicted tasks
3. Return A2aTask with current status/messages/artifacts
```