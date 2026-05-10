# Other — librefang-channels

# librefang-channels

Pluggable messaging bridge layer for LibreFang. Converts inbound platform messages into unified `ChannelMessage` events consumed by the kernel, and routes agent replies back out through the originating channel.

## Architecture

Each messaging platform is implemented as an adapter behind the `ChannelAdapter` trait. The trait and all dispatch infrastructure compile unconditionally — only individual adapters are gated by cargo features.

```mermaid
graph LR
    subgraph Platforms
        T[Telegram]
        D[Discord]
        S[Slack]
        W[Webhook]
        E[email / IMAP]
        O["40+ more"]
    end
    subgraph librefang-channels
        A["ChannelAdapter trait"]
        R["router / dispatch"]
    end
    K["librefang-kernel"]
    T -->|inbound| A
    D -->|inbound| A
    S -->|inbound| A
    W -->|inbound| A
    E -->|inbound| A
    O -->|inbound| A
    A -->|ChannelMessage| R
    R -->|events| K
    K -->|agent reply| R
    R -->|send()| A
    A -->|outbound| T
    A -->|outbound| D
    A -->|outbound| S
    A -->|outbound| W
    A -->|outbound| E
```

Sessions are derived deterministically via `SessionId::for_channel(agent, "channel:chat")`. The kernel owns the per-`(agent, session)` lock; channels never touch it directly.

## Cargo Features

`default = []`. Every workspace consumer sets `default-features = false` and forwards an explicit feature subset.

| Feature | Description |
|---|---|
| `all-channels` | Aggregates every adapter. Used by release CI and packaging. |
| `channel-telegram` | Telegram Bot API |
| `channel-discord` | Discord via gateway |
| `channel-slack` | Slack Events API |
| `channel-matrix` | Matrix (heavy; pulls in `matrix-sdk`) |
| `channel-email` | SMTP send + IMAP receive (adds `lettre`, `imap`, `rustls-connector`, `mailparse`) |
| `channel-webhook` | Generic outbound webhooks with SSRF guard |
| `channel-mqtt` | MQTT pub/sub (adds `rumqttc`) |
| `channel-bluesky` | Bluesky/AT Protocol |
| `channel-nostr` | Nostr relays (adds `k256`) |
| `channel-ntfy` | ntfy.sh push notifications |
| `channel-feishu` / `channel-wecom` | Adds `aes` + `cbc` for payload decryption |
| `channel-google-chat` | Adds `rsa` for signature verification |

See `Cargo.toml` for the full 45-feature matrix. Adding a new adapter requires a matching entry in the `all-channels` list.

## Always-Compiled Core

These modules are available regardless of which channel features are enabled:

- **`types`** — `ChannelMessage`, session identifiers, shared enums
- **`router`** — dispatch routing from adapter → kernel event bus
- **`bridge`** — top-level adapter registry and lifecycle
- **`formatter`** — message formatting across platform constraints
- **`sanitizer`** — input sanitization for inbound payloads
- **`commands`** — channel command parsing
- **`roster`** — contact/participant tracking
- **`rate_limiter`** — per-channel rate limiting
- **`message_journal`** — inbound/outbound message logging
- **`message_truncator`** — platform-aware message splitting
- **`attachment_enrich`** — attachment metadata extraction (uses `image`, `pdf-extract`)
- **`thread_ownership`** — conversation threading semantics
- **`sidecar`** — auxiliary channel-side processing

**Re-exports:** `split_to_utf16_chunks`, `truncate_to_utf16_limit`, `utf16_len`, `DISCORD_MESSAGE_LIMIT` (2000), `TELEGRAM_CAPTION_LIMIT` (1024), `TELEGRAM_MESSAGE_LIMIT` (4096).

## Adding a New Channel Adapter

1. **Create `src/<channel>/mod.rs`** implementing the `ChannelAdapter` trait.
2. **Add a cargo feature** `channel-<name>` in `Cargo.toml`. If it needs extra crate dependencies, add them as optional deps and wire `dep:xxx` in the feature.
3. **Add the feature to `all-channels`** in the same `Cargo.toml`.
4. **Wire HTTP webhook route** (if the platform uses webhooks) in `librefang-api/src/routes/channels.rs`.
5. **Write `tests/<channel>_wiremock.rs`** covering at least the `send()` happy path and one error response (e.g., 429 rate limit, 500 server error).
6. **Document required environment variables** in the adapter module's doc comment.

### ChannelAdapter Trait Contract

Every adapter implements:

- **Inbound parsing** — transform platform-specific webhook payloads or streaming events into `ChannelMessage`.
- **`send()`** — transmit an agent reply back to the platform. Must be testable via wiremock.
- **Lifecycle hooks** — startup validation (credential checks, SSRF guard), graceful shutdown.

## Webhook Security

HMAC/signature verification is **mandatory** for platforms that provide it. There is no bypass.

| Platform | Verification | Config key |
|---|---|---|
| Messenger | HMAC-SHA1 with App Secret | `app_secret_env` in `[channels.messenger]` |
| Teams | Base64 outgoing-webhook token | `security_token_env` in `[channels.teams]` |
| LINE | Platform signature header | Auto-detected |
| Viber | Platform signature header | Auto-detected |
| DingTalk | Platform signature header | Auto-detected |

Missing signature → **400**. Mismatched signature → **401**. Health-check probes (curl, monitoring) without the header will receive 4xx — this is intentional.

If you cannot implement verification for a platform, the adapter must refuse to start.

## Outbound Webhook SSRF Guard

`[channels.webhook] callback_url` must resolve to a publicly routable IP. The guard rejects:

- Private ranges: 10/8, 172.16/12, 192.168/16
- CGN: 100.64/10
- Loopback: 127/8, ::1
- Link-local, multicast, cloud metadata (169.254/16)
- IPv6 short forms (`[::]`), IPv4-mapped (`[::ffff:127.0.0.1]`), NAT64, trailing-dot FQDNs

Adapters refuse to start if the callback URL violates these rules. For local development, use a public tunnel (ngrok, cloudflared) or omit `callback_url` entirely.

## Testing Requirements

Inbound parsing has extensive coverage (795 tests). Outbound `send()` has historically been undertested.

**All new adapters and `send()` changes must include a wiremock test** in `tests/<channel>_wiremock.rs`. Minimum coverage:

- Happy-path `send()` with expected request shape
- At least one error response (HTTP 4xx/5xx) verifying the adapter handles it gracefully

PRs without `send()` tests will be rejected.

## Dependency Boundaries

### Allowed dependencies
- `librefang-types`
- `librefang-extensions` (vault for secrets, `http_client::shared_client()` for HTTP)
- `librefang-http`
- Channel-specific SDKs (gated per feature)

### Forbidden dependencies
- **`librefang-kernel`** — channels sit below the kernel. The kernel calls into channels through dispatch; never the reverse.
- **`librefang-runtime`** — same boundary reason.

### Shared HTTP client

Do not construct a bespoke `reqwest::Client`. Always use `librefang-extensions::http_client::shared_client()` to reuse connection pools and respect workspace-wide TLS/proxy configuration.