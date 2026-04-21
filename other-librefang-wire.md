# Other — librefang-wire

# librefang-wire

Agent-to-agent networking layer for the LibreFang Protocol (OFP). This crate handles message framing, serialization, authenticated transport, and connection management between LibreFang agents.

## Role in the System

`librefang-wire` sits between the high-level agent logic and the raw network I/O. It is responsible for taking typed messages, authenticating them, framing them for the wire, and delivering them over async TCP connections. It depends on `librefang-types` for shared message definitions and type primitives used across the LibreFang ecosystem.

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────┐
│  Agent Logic │────▶│ librefang-wire   │────▶│  TCP (tokio)  │
│              │◀────│ (framing, auth,  │◀────│               │
│              │     │  routing)        │     │               │
└──────────────┘     └──────────────────┘     └───────────────┘
                            │
                            ▼
                     ┌──────────────┐
                     │librefang-types│
                     │(shared types) │
                     └──────────────┘
```

## Core Responsibilities

### Protocol Framing

Messages are serialized to JSON (via `serde_json`) and framed for transmission over TCP streams. The framing protocol handles:

- Delimiting message boundaries on a byte stream
- Ensuring complete messages are received before deserialization
- Multiplexing message types over a single connection

### Authenticated Messaging

The crate implements HMAC-based message authentication using the following dependency chain:

| Dependency | Purpose |
|---|---|
| `hmac` + `sha2` | HMAC-SHA256 computation for message integrity |
| `subtle` | Constant-time comparison to prevent timing attacks during verification |
| `rand` | Secure random generation for nonces and challenge tokens |
| `hex` | Encoding/decoding of HMAC digests |

Every message on the wire carries an authentication tag. Receivers verify the tag before deserializing the payload, ensuring that only agents sharing a valid key can communicate.

### Concurrent Connection State

`dashmap` provides a lock-free concurrent hashmap, used internally to track active connections, peer state, and session metadata across async tasks without blocking the Tokio runtime.

### Error Handling

All fallible operations return structured errors derived via `thiserror`. Errors distinguish between:

- Protocol-level failures (malformed frames, bad magic bytes)
- Authentication failures (invalid HMAC, expired challenges)
- I/O failures (disconnected peers, timeouts)

### Observability

`tracing` spans and events are emitted throughout the transport lifecycle — connection establishment, message send/receive, authentication handshake steps, and error conditions.

## Key Dependencies and Their Usage

| Crate | Usage |
|---|---|
| `tokio` | Async runtime; TCP stream handling, task spawning |
| `serde` / `serde_json` | Message serialization and deserialization |
| `uuid` | Unique message and session identifiers |
| `chrono` | Timestamps for messages and session tracking |
| `async-trait` | Async trait definitions for transport abstractions |
| `librefang-types` | Shared protocol types (message enums, payloads, IDs) |

## Integration Points

Other crates in the workspace consume `librefang-wire` to establish agent-to-agent communication channels without dealing with framing, authentication, or connection lifecycle management directly. The crate exposes an async API compatible with the Tokio runtime used throughout LibreFang.

To use this crate, add it as a dependency:

```toml
[dependencies]
librefang-wire = { path = "../librefang-wire" }
```

## Testing

The `tokio-test` dev-dependency is used for writing async tests that exercise connection setup, message round-trips, and authentication handshake behavior without requiring a live network.