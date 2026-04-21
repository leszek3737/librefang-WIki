# Other — librefang-channels

# librefang-channels

Channel Bridge Layer — pluggable messaging integrations for the LibreFang alerting platform.

## Purpose

This crate provides a unified abstraction over 40+ messaging and notification platforms. It translates LibreFang's internal alert format into platform-specific API calls, WebSocket messages, or webhook payloads, allowing alerts to be dispatched to any supported channel through a single interface.

## Architecture

```
┌─────────────────────────────────────────────────┐
│              librefang-channels                  │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │         Channel Trait (core)              │   │
│  │  dispatch() / validate() / health()       │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │Telegram │ │ Discord │ │ Slack   │  ...       │
│  └─────────┘ └─────────┘ └─────────┘           │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐           │
│  │ Matrix  │ │  Email  │ │ Webhook │  ...       │
│  └─────────┘ └─────────┘ └─────────┘           │
│                                                  │
│  Shared: HTTP / WebSocket / Crypto / Parsing     │
└──────────────┬──────────────────────────────────┘
               │ depends on
        librefang-types
```

Each channel is compiled conditionally behind a Cargo feature flag. The `default` feature enables all 44 stable channels. Consumers that want a minimal build can disable default features and enable only the channels they need.

## Core Dependencies

| Dependency | Role |
|---|---|
| `librefang-types` | Shared types defining alerts, channel configurations, and result types |
| `tokio` | Async runtime for concurrent dispatch |
| `reqwest` | HTTP client for REST-based channel APIs |
| `tokio-tungstenite` | WebSocket support for real-time channels (e.g., Discord, Slack, Matrix) |
| `axum` | HTTP server for receiving inbound webhooks (delivery confirmations, callbacks) |
| `dashmap` | Lock-free concurrent map for channel registry and state |
| `async-trait` | Trait object support for async channel methods |
| `hmac`, `sha2`, `sha1` | Signature verification for inbound webhook authentication |
| `image` | Image processing for channels with media constraints |
| `serde`, `serde_json` | Serialization for API payloads |
| `tracing` | Structured logging across all channel implementations |
| `uuid` | Correlation IDs for dispatch tracking |

## Feature Flags

### Meta-features

- **`default`** — Enables all 44 stable channels.
- **`all-channels`** — Enables all stable channels **plus** `channel-mqtt`. Use this when you want every available integration including experimental ones.

### Enabling specific channels

Disable default features and list only what you need:

```toml
[dependencies]
librefang-channels = { path = "../librefang-channels", default-features = false, features = [
    "channel-telegram",
    "channel-slack",
    "channel-email",
] }
```

### Channel-specific optional dependencies

Most channels are implemented using the shared `reqwest`/`tokio-tungstenite` infrastructure and require no additional crates. The following channels pull in specialized dependencies:

| Feature | Additional dependencies | Reason |
|---|---|---|
| `channel-email` | `lettre`, `imap`, `rustls-connector`, `mailparse` | SMTP sending, IMAP receiving, TLS, MIME parsing |
| `channel-google-chat` | `rsa` | RSA key-based authentication (service accounts) |
| `channel-feishu` | `aes`, `cbc` | AES-CBC encryption for callback verification |
| `channel-wecom` | `aes`, `cbc`, `roxmltree` | AES-CBC for message encryption, XML parsing for API responses |
| `channel-nostr` | `k256` | secp256k1 key generation and event signing |
| `channel-mqtt` | `rumqttc` | MQTT v5 client for IoT/push channels |

## Channel List

```
channel-telegram       channel-discord         channel-slack
channel-matrix         channel-email           channel-webhook
channel-whatsapp       channel-signal          channel-teams
channel-mattermost     channel-irc             channel-google-chat
channel-twitch         channel-rocketchat      channel-zulip
channel-xmpp           channel-bluesky         channel-feishu
channel-line           channel-mastodon        channel-messenger
channel-reddit         channel-revolt          channel-viber
channel-voice          channel-flock           channel-guilded
channel-keybase        channel-nextcloud       channel-nostr
channel-pumble         channel-threema         channel-twist
channel-webex          channel-dingtalk        channel-discourse
channel-gitter         channel-gotify          channel-linkedin
channel-mumble         channel-ntfy            channel-qq
channel-wechat         channel-wecom           channel-mqtt
```

## Benchmarks

The crate includes a Criterion benchmark suite targeting the dispatch path:

```bash
cargo bench --bench dispatch
```

Use this to measure throughput and latency impact when adding or modifying a channel implementation.