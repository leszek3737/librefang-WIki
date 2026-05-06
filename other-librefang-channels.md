# Other — librefang-channels

# librefang-channels

Channel Bridge Layer for LibreFang — 40+ pluggable messaging integrations that convert platform messages into unified `ChannelMessage` events and route agent replies back out.

## Architecture

```mermaid
graph TD
    Platform[Platform: Telegram, Discord, Slack, ...]
    Adapter[ChannelAdapter impl]
    Core[Trait + Dispatch Core]
    Kernel[librefang-kernel]
    API[librefang-api webhook routes]

    Platform -->|inbound message| Adapter
    Adapter -->|ChannelMessage| Core
    Core -->|dispatch| Kernel
    Kernel -->|agent reply| Core
    Core -->|send()| Adapter
    Adapter -->|outbound| Platform
    API -->|HTTP POST| Adapter
```

Channels sit below the kernel. The kernel calls into channels through dispatch — channels never import the kernel. Inbound messages from platforms are normalized into `ChannelMessage` events. Outbound replies flow back through the same adapter's `send()` method.

## Cargo Features

**`default = []`** — intentionally empty. Every workspace consumer (`librefang-api`, `librefang-cli`, `librefang-desktop`) sets `default-features = false` and forwards an explicit subset. Always pick features explicitly when depending on this crate.

- **`all-channels`** — enables every adapter. Used by release CI and packaging pipelines. Not for everyday development.
- **Per-adapter features** — `channel-telegram`, `channel-discord`, `channel-slack`, `channel-webhook`, `channel-ntfy`, etc. See `Cargo.toml` for the full list of 45 features.

Some adapters pull in additional dependencies:
- `channel-email` → `lettre`, `imap`, `rustls-connector`, `mailparse`
- `channel-google-chat` → `rsa`
- `channel-feishu` / `channel-wecom` → `aes`, `cbc`
- `channel-nostr` → `k256`
- `channel-mqtt` → `rumqttc`

## Always-Compiled Core

The trait definitions, dispatch glue, and shared utilities compile unconditionally. Only adapter implementations are feature-gated.

Core modules:
- **`types`** — `ChannelMessage` event type and related shared types
- **`bridge`** — dispatch and routing infrastructure
- **`router`** — message routing logic
- **`attachment_enrich`** — attachment processing
- **`commands`** — channel command handling
- **`formatter`** — message formatting for platform-specific constraints
- **`message_journal`** — message logging/journaling
- **`message_truncator`** — message length truncation utilities
- **`rate_limiter`** — per-channel rate limiting
- **`roster`** — contact/participant management
- **`sanitizer`** — input sanitization
- **`sidecar`** — auxiliary channel processing
- **`thread_ownership`** — conversation thread tracking

Re-exported utilities for platform message limits:
- `split_to_utf16_chunks`, `truncate_to_utf16_limit`, `utf16_len`
- `DISCORD_MESSAGE_LIMIT` (2000 characters)
- `TELEGRAM_MESSAGE_LIMIT` (4096 characters)
- `TELEGRAM_CAPTION_LIMIT` (1024 characters)

## Channel Adapter Trait

Each adapter lives in `src/<channel>/mod.rs` and implements `ChannelAdapter`. The trait defines the interface for:

1. **Inbound parsing** — receive a platform-specific webhook payload, validate it, and produce a `ChannelMessage`.
2. **Outbound sending** — take an agent reply and deliver it to the platform via the adapter's `send()` method.

Sessions are derived deterministically: `SessionId::for_channel(agent, "channel:chat")`. The kernel owns the per-`(agent, session)` lock; channels never touch it.

## Security

### HMAC Verification (Mandatory)

HMAC signature verification is **mandatory** for platforms that provide it. There is no bypass.

| Platform | Environment Variable | Config Key | Failure Behavior |
|----------|---------------------|------------|------------------|
| Messenger | `MESSENGER_APP_SECRET` | `[channels.messenger] app_secret_env` | 400 (missing), 401 (mismatch) |
| Teams | `TEAMS_SECURITY_TOKEN` | `[channels.teams] security_token_env` | 400 (missing), 401 (mismatch) |
| LINE | Platform-specific header | — | 400/401 |
| Viber | Platform-specific header | — | 400/401 |
| DingTalk | Platform-specific header | — | 400/401 |

Health checks and probes without the correct signature header receive 4xx responses. This is intentional.

### Outbound SSRF Guard

`[channels.webhook] callback_url` must resolve to a public IP. The adapter refuses to start if the URL targets:

- Private ranges: `10/8`, `172.16/12`, `192.168/16`
- Carrier-grade NAT: `100.64/10`
- Loopback: `127/8`, `::1`
- Link-local, multicast, cloud metadata endpoints
- IPv6 short forms (`[::]`), IPv4-mapped IPv6 (`[::ffff:127.0.0.1]`), NAT64, trailing-dot FQDNs

For local development, use a public tunnel (ngrok, cloudflared) or omit `callback_url`.

## Dependencies

Direct dependencies:
- `librefang-types` — shared type definitions
- `librefang-extensions` — vault access, `http_client::shared_client()`
- `librefang-http` — HTTP utilities

Explicitly **not** depended on:
- `librefang-kernel` — channels are below kernel in the layer graph
- `librefang-runtime` — no direct runtime dependency

HTTP requests must use `librefang-extensions::http_client::shared_client()`. Do not create bespoke `reqwest::Client` instances.

## Testing

Inbound parsing has extensive coverage (795+ tests). Outbound `send()` has historically been undertested.

**Requirement:** Any new adapter or `send()` work must include a wiremock test in `tests/<channel>_wiremock.rs`. Minimum coverage:

- `send()` happy path
- One error response scenario

PRs adding an adapter without a `send()` test will be rejected.

The crate also includes a criterion benchmark at `benches/dispatch`.

## Adding a New Channel

1. Create `src/<channel>/mod.rs` implementing the `ChannelAdapter` trait.
2. Add a `channel-<name>` feature to `Cargo.toml` (empty feature flag, or with optional dependencies).
3. Add the feature to the `all-channels` list in `Cargo.toml`.
4. If the channel uses webhooks, wire the HTTP route in `librefang-api/src/routes/channels.rs`.
5. Create `tests/<channel>_wiremock.rs` with at minimum a `send()` happy-path test and one error case.
6. Document required environment variables in the adapter's module-level doc comment.

All channels ship off-by-default. Do not add new channels to `default`.

## Constraints

- **No `librefang-kernel` import.** The kernel calls down into channels; channels never reach up.
- **No bespoke `reqwest::Client`.** Always use the shared client from `librefang-extensions`.
- **No `default = ["all-channels"]`.** The default feature set is and remains empty.
- **No bypassing HMAC verification.** Implement it fully or refuse to start the adapter.
- **No raw `callback_url` without SSRF validation.** Use the existing guard.