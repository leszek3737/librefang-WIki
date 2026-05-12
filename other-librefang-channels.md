# Other — librefang-channels

# librefang-channels

Channel Bridge Layer for LibreFang. Provides 40+ pluggable messaging integrations that convert inbound platform messages into unified `ChannelMessage` events for the kernel and route agent replies back out.

## Architecture

```mermaid
graph LR
    subgraph "External Platforms"
        T[Telegram]
        D[Discord]
        S[Slack]
        W[Webhook]
        O[37+ more]
    end

    subgraph "librefang-channels"
        A[ChannelAdapter trait]
        R[Router / Dispatch]
        C[Core modules]
    end

    K[librefang-kernel]

    T -->|HTTP webhook| A
    D -->|HTTP webhook| A
    S -->|HTTP webhook| A
    W -->|HTTP POST| A
    O -->|various| A

    A --> R
    R -->|ChannelMessage| K
    K -->|reply| R
    R -->|send()| A
    A -->|HTTP/WS/SMTP| T
    A -->|HTTP/WS/SMTP| D
```

Channels sit below the kernel. The kernel calls into channels through dispatch; channels never import the kernel. Each adapter implements the `ChannelAdapter` trait, and the router/dispatch layer fans messages in and replies out.

## Cargo Features

**`default = []`**. Every workspace consumer (`librefang-api`, `librefang-cli`, `librefang-desktop`) sets `default-features = false` and forwards an explicit subset. There is no default channel set.

| Feature flag | Platform |
|---|---|
| `channel-telegram` | Telegram |
| `channel-discord` | Discord |
| `channel-slack` | Slack |
| `channel-matrix` | Matrix (pulls in `pulldown-cmark`) |
| `channel-email` | IMAP/SMTP (pulls in `lettre`, `imap`, `mailparse`, `rustls-*`) |
| `channel-webhook` | Generic outbound webhook |
| `channel-whatsapp` | WhatsApp |
| `channel-signal` | Signal |
| `channel-teams` | Microsoft Teams |
| `channel-mattermost` | Mattermost |
| `channel-irc` | IRC |
| `channel-google-chat` | Google Chat (pulls in `rsa`) |
| `channel-mqtt` | MQTT (pulls in `rumqttc`) |
| `channel-bluesky` | Bluesky |
| `channel-nostr` | Nostr (pulls in `k256`) |
| `channel-feishu` | Feishu/Lark (pulls in `aes`, `cbc`) |
| `channel-wecom` | WeCom (pulls in `aes`, `cbc`) |
| `channel-ntfy` | ntfy |
| `all-channels` | Aggregates every adapter above. Used by release CI only. |

See `Cargo.toml` for the complete feature matrix (44 adapters total).

## Always-Compiled Core

The trait definition, dispatch glue, and shared utilities compile unconditionally. Only individual adapter implementations are feature-gated.

**Core modules:**

| Module | Responsibility |
|---|---|
| `types` | `ChannelMessage`, session IDs, shared types |
| `bridge` | Adapter registration and lifecycle |
| `router` | Inbound message routing to the kernel |
| `commands` | Channel-side command handling |
| `formatter` | Message formatting across platforms |
| `sanitizer` | Input sanitization |
| `message_journal` | Message persistence/journaling |
| `message_truncator` | Platform-aware message splitting |
| `rate_limiter` | Per-channel rate limiting |
| `roster` | Contact/participant management |
| `sidecar` | Auxiliary channel processes |
| `thread_ownership` | Thread/conversation ownership tracking |
| `attachment_enrich` | Attachment metadata extraction |

**Re-exports for platform limits:**

- `split_to_utf16_chunks`, `truncate_to_utf16_limit`, `utf16_len` — UTF-16 aware text utilities for platforms that count in UTF-16 code units
- `DISCORD_MESSAGE_LIMIT` — 2000 characters
- `TELEGRAM_MESSAGE_LIMIT` — 4096 characters
- `TELEGRAM_CAPTION_LIMIT` — 1024 characters

## Boundary

### What this crate owns

- `ChannelAdapter` trait and all adapter implementations under `src/<channel>/`
- `ChannelMessage` event type
- Message routing and dispatch logic
- Attachment enrichment, sanitization, formatting, truncation

### What this crate does NOT own

- HTTP webhook route handlers — those live in `librefang-api/src/routes/channels.rs`
- Kernel's per-`(agent, session)` lock — channel messages always derive `SessionId::for_channel(agent, "channel:chat")`

### Dependency direction

| Depends on | Does NOT depend on |
|---|---|
| `librefang-types` | `librefang-kernel` |
| `librefang-extensions` (vault, shared HTTP client) | `librefang-runtime` |
| `librefang-http` | |

## Inbound Message Flow

1. External platform sends a webhook/event to `librefang-api`.
2. The API route handler in `librefang-api/src/routes/channels.rs` invokes the channel adapter.
3. The adapter parses the platform-specific payload into a `ChannelMessage`.
4. The router dispatches the `ChannelMessage` to the kernel.
5. The kernel processes the message, producing a reply.
6. The kernel calls back into the adapter's `send()` method.
7. The adapter translates the reply into the platform's API format and delivers it.

## Webhook Security

HMAC/signature verification is **mandatory** for these platforms. Adapters must refuse to start if verification cannot be configured.

| Platform | Environment variable | Config key | Behavior |
|---|---|---|---|
| Messenger | `MESSENGER_APP_SECRET` | `[channels.messenger] app_secret_env` | Missing header → 400, mismatch → 401 |
| Teams | `TEAMS_SECURITY_TOKEN` | `[channels.teams] security_token_env` | Base64 outgoing-webhook token verification |
| LINE | Platform signature header | — | HMAC-SHA256 on request body |
| Viber | Platform signature header | — | HMAC-SHA256 on request body |
| DingTalk | Platform signature header | — | Platform-specific signature |

Health check probes (curl, monitoring) that omit the signature header will receive 4xx responses. This is intentional.

## Outbound Webhook SSRF Guard

The `[channels.webhook] callback_url` is validated at adapter startup. The URL must resolve to a public IP. The following are rejected:

- **Private ranges:** 10/8, 172.16/12, 192.168/16
- **CGN:** 100.64/10
- **Loopback:** 127/8, ::1
- **Link-local and multicast**
- **Cloud metadata endpoints** (169.254/16)
- **IPv6 short forms:** `[::]`
- **IPv4-mapped IPv6:** `[::ffff:127.0.0.1]`
- **NAT64 prefixes**
- **Trailing-dot FQDNs**

For local development, use a public tunnel (ngrok, cloudflared) or omit `callback_url` entirely.

## Adding a New Channel Adapter

1. **Create the adapter:** New file `src/<channel>/mod.rs` implementing the `ChannelAdapter` trait.
2. **Register the feature:** Add `channel-<name> = []` to `Cargo.toml` `[features]`. Add `"channel-<name>"` to the `all-channels` list.
3. **Default is off:** Every adapter ships off-by-default. Never add a new channel to `default`.
4. **Wire the webhook route** (if the platform uses push delivery) in `librefang-api/src/routes/channels.rs`.
5. **Write send-path tests:** Create `tests/<channel>_wiremock.rs` with at minimum:
   - A `send()` happy-path test against a wiremock server
   - One error-response test (non-2xx from the platform API)
6. **Document environment variables** in the adapter module's doc comment.

PRs without a `send()` wiremock test will be rejected.

## Testing

- **Inbound parsing:** 795 existing tests covering platform payload parsing.
- **Outbound `send()`:** Historically undertested. All new work must include wiremock-based tests in `tests/<channel>_wiremock.rs`.
- **Benchmarks:** Dispatch benchmarks live in `benches/dispatch`.

## Rules and Taboos

| Rule | Reason |
|---|---|
| No `librefang-kernel` import | Channels are below the kernel in the dependency graph. The kernel calls into channels; channels never call up. |
| No bespoke `reqwest::Client` | Use `librefang-extensions::http_client::shared_client()` for connection pooling and consistent TLS configuration. |
| No `default = ["all-channels"]` | The default feature set is and must remain empty. |
| No bypassing HMAC verification | If a platform requires signature verification, the adapter must implement it or refuse to start. Never silently skip. |
| No raw `callback_url` without SSRF guard | Use the existing SSRF validation guard for all outbound webhook URLs. |