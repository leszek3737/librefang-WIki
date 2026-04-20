# Other — librefang-channels

# librefang-channels

**Channel Bridge Layer** — pluggable messaging integrations for LibreFang.

This crate provides a uniform interface for sending and receiving messages across 43+ messaging platforms. Each platform is compiled behind a Cargo feature flag, allowing deployments to include only the channels they need.

## Architecture

```
┌─────────────────────────────────────────────────┐
│              librefang-channels                  │
│                                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐        │
│  │ Telegram │ │ Discord  │ │  Slack   │  ...    │
│  │  Driver  │ │  Driver  │ │  Driver  │         │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘        │
│       │             │             │              │
│       └─────────────┼─────────────┘              │
│                     │                            │
│              ┌──────▼──────┐                     │
│              │  Dispatch   │                     │
│              │   / Router  │                     │
│              └──────┬──────┘                     │
│                     │                            │
└─────────────────────┼────────────────────────────┘
                      │
              ┌───────▼───────┐
              │ librefang-    │
              │   types       │
              └───────────────┘
```

Each channel driver translates between platform-specific APIs (REST, WebSocket, IMAP, etc.) and the shared types defined in `librefang-types`. A central dispatch/router normalizes inbound messages and routes outbound messages to the correct driver.

## Supported Channels

| Channel | Feature Flag | Extra Dependencies |
|---------|-------------|-------------------|
| Telegram | `channel-telegram` | — |
| Discord | `channel-discord` | — |
| Slack | `channel-slack` | — |
| Matrix | `channel-matrix` | — |
| Email | `channel-email` | `lettre`, `imap`, `rustls-connector`, `mailparse` |
| Webhook | `channel-webhook` | — |
| WhatsApp | `channel-whatsapp` | — |
| Signal | `channel-signal` | — |
| Microsoft Teams | `channel-teams` | — |
| Mattermost | `channel-mattermost` | — |
| IRC | `channel-irc` | — |
| Google Chat | `channel-google-chat` | `rsa` |
| Twitch | `channel-twitch` | — |
| Rocket.Chat | `channel-rocketchat` | — |
| Zulip | `channel-zulip` | — |
| XMPP | `channel-xmpp` | — |
| Bluesky | `channel-bluesky` | — |
| Feishu (Lark) | `channel-feishu` | `aes`, `cbc` |
| LINE | `channel-line` | — |
| Mastodon | `channel-mastodon` | — |
| Messenger (Facebook) | `channel-messenger` | — |
| Reddit | `channel-reddit` | — |
| Revolt | `channel-revolt` | — |
| Viber | `channel-viber` | — |
| Voice | `channel-voice` | — |
| Flock | `channel-flock` | — |
| Guilded | `channel-guilded` | — |
| Keybase | `channel-keybase` | — |
| Nextcloud Talk | `channel-nextcloud` | — |
| Nostr | `channel-nostr` | `k256` |
| Pumble | `channel-pumble` | — |
| Threema | `channel-threema` | — |
| Twist | `channel-twist` | — |
| Webex | `channel-webex` | — |
| DingTalk | `channel-dingtalk` | — |
| Discourse | `channel-discourse` | — |
| Gitter | `channel-gitter` | — |
| Gotify | `channel-gotify` | — |
| LinkedIn | `channel-linkedin` | — |
| Mumble | `channel-mumble` | — |
| ntfy | `channel-ntfy` | — |
| QQ | `channel-qq` | — |
| WeChat | `channel-wechat` | — |
| WeCom | `channel-wecom` | `aes`, `cbc`, `roxmltree` |
| MQTT | `channel-mqtt` | `rumqttc` |

## Feature Flags

### Default

The `default` feature enables all channels **except** `channel-mqtt`:

```toml
[dependencies]
librefang-channels = { path = "../librefang-channels" }
```

### Selective inclusion

To include only specific channels, disable default features and list the ones you need:

```toml
[dependencies]
librefang-channels = {
    path = "../librefang-channels",
    default-features = false,
    features = ["channel-telegram", "channel-discord"],
}
```

### All channels

The `all-channels` feature includes every channel, including `channel-mqtt`:

```toml
[dependencies]
librefang-channels = {
    path = "../librefang-channels",
    default-features = false,
    features = ["all-channels"],
}
```

## Key Dependencies and Their Roles

| Dependency | Purpose |
|-----------|---------|
| `librefang-types` | Shared domain types (messages, channel configs, events) |
| `tokio` | Async runtime for all channel I/O |
| `reqwest` | HTTP client for REST-based channel APIs |
| `tokio-tungstenite` | WebSocket transport (used by several real-time channels) |
| `axum` | HTTP server for receiving webhook callbacks |
| `serde` / `serde_json` | Serialization of API payloads and config |
| `dashmap` | Concurrent hashmap for managing channel state |
| `async-trait` | Async trait definitions for the channel driver interface |
| `image` | Image processing (JPEG, PNG, WebP) for media handling |
| `hmac` / `sha2` / `sha1` | HMAC-SHA signature verification for webhook authentication |
| `zeroize` | Secure clearing of sensitive credentials from memory |
| `rustls` | TLS via Rustls (no OpenSSL dependency) |

### Channel-specific cryptography

- **`channel-email`**: `lettre` (SMTP sending), `imap` + `rustls-connector` + `mailparse` (IMAP receiving and parsing).
- **`channel-google-chat`**: `rsa` for service account authentication (JWT signing).
- **`channel-feishu` / `channel-wecom`**: `aes` + `cbc` for decrypting callback payloads. WeCom additionally uses `roxmltree` for XML-based API responses.
- **`channel-nostr`**: `k256` for secp256k1 key management and event signing per the Nostr protocol.
- **`channel-mqtt`**: `rumqttc` as the MQTT v5 client.

## Benchmarking

A dispatch benchmark is available under `benches/dispatch`:

```bash
cargo bench --bench dispatch
```

This measures message routing throughput through the dispatch layer, useful when evaluating the overhead of adding many concurrent channel drivers.

## Integration with LibreFang

This crate sits between the core application logic and external messaging platforms:

1. **Inbound**: A channel driver receives a message (via webhook, polling, or persistent connection), normalizes it into a `librefang-types` message structure, and passes it upstream for processing.
2. **Outbound**: The core application sends a normalized message to the dispatch layer, which routes it to the appropriate channel driver. The driver translates it into the platform's native format and delivers it.

All credential storage, message normalization, and event routing rely on types from `librefang-types`, ensuring consistency across the entire system.

## Adding a New Channel

1. Add a new feature flag in `Cargo.toml`:
   ```toml
   channel-mychannel = ["dep:some-crate"]
   ```
2. Add the feature to both the `default` and `all-channels` lists.
3. Implement the channel driver following the async trait interface defined in this crate.
4. Register the driver with the dispatch router so it can be selected at runtime based on configuration from `librefang-types`.