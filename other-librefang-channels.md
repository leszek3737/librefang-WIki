# Other — librefang-channels

# librefang-channels

Channel Bridge Layer — pluggable messaging integrations for LibreFang.

## Purpose

`librefang-channels` provides a unified abstraction over dozens of messaging platforms and notification services. It translates LibreFang's internal message types into protocol-specific formats and handles the transmission, allowing the rest of the system to dispatch alerts and messages without knowing which platform will receive them.

## Architecture

The crate is built around a channel-trait pattern. Each supported platform implements a common async interface, and features are used to compile in only the integrations you need.

```
┌──────────────────────────────┐
│       librefang-types        │
│   (shared message types)     │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│    librefang-channels        │
│                              │
│  ┌──────────┐  ┌──────────┐  │
│  │ Telegram │  │ Discord  │  │
│  └──────────┘  └──────────┘  │
│  ┌──────────┐  ┌──────────┐  │
│  │  Slack   │  │  Matrix  │  │
│  └──────────┘  └──────────┘  │
│         ...43 total...       │
└──────────────────────────────┘
```

The module depends on `librefang-types` for the shared message structures that flow through the system.

## Supported Channels

All 43 channels and their feature flags:

| Feature Flag | Platform | Extra Dependencies |
|---|---|---|
| `channel-telegram` | Telegram Bot API | — |
| `channel-discord` | Discord Webhooks / Bot | — |
| `channel-slack` | Slack Webhooks / API | — |
| `channel-matrix` | Matrix (Element) | — |
| `channel-email` | Email (SMTP/IMAP) | `lettre`, `imap`, `rustls-connector`, `mailparse` |
| `channel-webhook` | Generic Webhooks | — |
| `channel-whatsapp` | WhatsApp Business API | — |
| `channel-signal` | Signal | — |
| `channel-teams` | Microsoft Teams | — |
| `channel-mattermost` | Mattermost | — |
| `channel-irc` | IRC | — |
| `channel-google-chat` | Google Chat | `rsa` |
| `channel-twitch` | Twitch | — |
| `channel-rocketchat` | Rocket.Chat | — |
| `channel-zulip` | Zulip | — |
| `channel-xmpp` | XMPP (Jabber) | — |
| `channel-bluesky` | Bluesky (AT Protocol) | — |
| `channel-feishu` | Feishu (Lark) | `aes`, `cbc` |
| `channel-line` | LINE | — |
| `channel-mastodon` | Mastodon | — |
| `channel-messenger` | Facebook Messenger | — |
| `channel-reddit` | Reddit | — |
| `channel-revolt` | Revolt | — |
| `channel-viber` | Viber | — |
| `channel-voice` | Voice / TTS calls | — |
| `channel-flock` | Flock | — |
| `channel-guilded` | Guilded | — |
| `channel-keybase` | Keybase | — |
| `channel-nextcloud` | Nextcloud Talk | — |
| `channel-nostr` | Nostr | `k256` |
| `channel-pumble` | Pumble | — |
| `channel-threema` | Threema | — |
| `channel-twist` | Twist | — |
| `channel-webex` | Cisco Webex | — |
| `channel-dingtalk` | DingTalk | — |
| `channel-discourse` | Discourse | — |
| `channel-gitter` | Gitter | — |
| `channel-gotify` | Gotify | — |
| `channel-linkedin` | LinkedIn | — |
| `channel-mumble` | Mumble | — |
| `channel-ntfy` | ntfy.sh | — |
| `channel-qq` | QQ | — |
| `channel-wechat` | WeChat | — |
| `channel-wecom` | WeCom (WeChat Work) | `aes`, `cbc`, `roxmltree` |
| `channel-mqtt` | MQTT | `rumqttc` |

**Note:** `channel-mqtt` is available under the `all-channels` feature but is **not** included in the `default` feature set.

## Feature Flags

### Default Behavior

By default, all channels except `channel-mqtt` are enabled. This means a bare dependency:

```toml
[dependencies]
librefang-channels = { path = "../librefang-channels" }
```

compiles with 42 channels. If you only need a subset, disable default features and select explicitly:

```toml
[dependencies]
librefang-channels = { path = "../librefang-channels", default-features = false, features = [
    "channel-telegram",
    "channel-discord",
    "channel-slack",
] }
```

### `all-channels`

The `all-channels` feature enables every channel including `channel-mqtt`. Use this when you want a complete build without listing each feature individually:

```toml
[dependencies]
librefang-channels = { path = "../librefang-channels", default-features = false, features = ["all-channels"] }
```

### Channel-Specific Dependencies

Six channels pull in additional crates when enabled. These are expressed as optional dependencies so they are only compiled when the corresponding feature is active:

- **Email** — `lettre` (SMTP), `imap`, `rustls-connector`, `mailparse`
- **Google Chat** — `rsa` (for service account JWT signing)
- **Feishu** — `aes`, `cbc` (for encryption of event verification)
- **WeCom** — `aes`, `cbc`, `roxmltree` (for message encryption and XML parsing)
- **Nostr** — `k256` (for secp256k1 key handling)
- **MQTT** — `rumqttc` (MQTT v5 client)

## Core Dependencies

The crate relies on these shared workspace dependencies:

| Crate | Role |
|---|---|
| `tokio` | Async runtime |
| `reqwest` | HTTP client for REST-based channels |
| `tokio-tungstenite` | WebSocket client for real-time channels |
| `axum` | HTTP server for webhook receiver endpoints |
| `serde` / `serde_json` | Serialization for API payloads |
| `dashmap` | Concurrent channel state maps |
| `async-trait` | Async trait definitions for channel interface |
| `hmac` / `sha2` / `sha1` | Signature verification for webhook authenticity |
| `tracing` | Structured logging |
| `image` | Image processing (JPEG, PNG, WebP) |
| `url` | URL parsing and construction |

## Benchmarking

A Criterion benchmark suite is configured under `benches/dispatch`. Run it with:

```bash
cargo bench -p librefang-channels
```

This measures message dispatch throughput across the channel abstraction layer.

## Adding a New Channel

To add a new messaging integration:

1. **Add the feature flag** in `[features]`:
   ```toml
   channel-myplatform = []
   ```
   If the platform needs extra crates:
   ```toml
   channel-myplatform = ["dep:some-crate"]
   ```

2. **Add the feature to both lists** — append it to `default` and `all-channels`.

3. **Add the optional dependency** (if any) under `[dependencies]`:
   ```toml
   some-crate = { workspace = true, optional = true }
   ```

4. **Implement the channel trait** in a new module under `src/`, gated by `#[cfg(feature = "channel-myplatform")]`.

5. **Register the channel** in the dispatch/registry logic so it can be looked up at runtime.

## Relationship to Other Modules

This crate sits between `librefang-types` (which defines the canonical message format) and the outside world. The core LibreFang engine or API layer calls into this module to send messages; it does not call back into other modules. This unidirectional dependency keeps the channel layer self-contained and testable in isolation.