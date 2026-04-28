# Other — librefang-wire

# librefang-wire

LibreFang Protocol (OFP) — agent-to-agent networking layer.

## Purpose

`librefang-wire` defines and implements the wire protocol used for secure, authenticated communication between LibreFang agents. It is responsible for message framing, serialization, cryptographic authentication, and the async transport abstractions that carry OFP messages over the network.

This crate does **not** implement application-level RPC or command logic. It provides the foundational transport and protocol primitives that higher-level agent communication builds upon.

## Role in the Architecture

```
┌─────────────────────────────────────────────┐
│              Agent Application              │
│         (commands, RPC, session logic)      │
├─────────────────────────────────────────────┤
│            librefang-wire (OFP)             │◄── you are here
│  framing · auth · serialization · transport │
├─────────────────────────────────────────────┤
│            librefang-types                  │
│        shared domain types & errors         │
└─────────────────────────────────────────────┘
```

`librefang-wire` sits directly above `librefang-types`, consuming shared domain types and error definitions. It exposes async transport and protocol primitives to the rest of the codebase.

## Key Capabilities

### Message Authentication (HMAC-SHA256)

The crate depends on `hmac`, `sha2`, `hex`, and `subtle`, indicating that every message on the wire is authenticated using HMAC-SHA256. The `subtle` crate provides constant-time comparison, which prevents timing side-channel attacks during signature verification.

This means:
- Outbound messages are signed with a shared secret key.
- Inbound messages are verified before any processing occurs.
- Signature comparison is constant-time to resist timing attacks.

### Async Transport

Built on `tokio`, all I/O is fully asynchronous. The `async-trait` dependency suggests that transport behaviors are defined as async traits, allowing different underlying implementations (TCP, TLS, Unix sockets, etc.) to be swapped in behind a common interface.

### Concurrent Connection Management

The `dashmap` dependency indicates that the module maintains a concurrent map of active connections or sessions. `DashMap` provides lock-free concurrent access, which is critical for agents handling many simultaneous peer connections without contention bottlenecks.

### Serialization

Messages are serialized to JSON via `serde` and `serde_json`. Each message carries a `uuid` for correlation and `chrono` timestamps for ordering and replay protection.

### Error Handling

Errors are defined using `thiserror`, producing typed, ergonomic error variants that integrate cleanly with the `?` operator and the broader LibreFang error hierarchy in `librefang-types`.

## Dependency Breakdown

| Dependency | Role in `librefang-wire` |
|---|---|
| `librefang-types` | Shared domain types, message envelopes, error types |
| `tokio` | Async runtime for all network I/O |
| `serde` / `serde_json` | Message serialization and deserialization |
| `uuid` | Unique message and correlation IDs |
| `chrono` | Timestamps for message ordering |
| `thiserror` | Typed error definitions |
| `tracing` | Structured logging of protocol events |
| `async-trait` | Async trait definitions for transport abstraction |
| `hmac` / `sha2` | HMAC-SHA256 message authentication |
| `hex` | Hex encoding/decoding of signatures and keys |
| `subtle` | Constant-time comparison for signature verification |
| `dashmap` | Concurrent map for connection/session tracking |

## Security Considerations

- **Authentication**: HMAC-SHA256 ensures message integrity and authenticity. Messages that fail verification are rejected before deserialization of any payload.
- **Constant-time comparison**: The `subtle` crate ensures that HMAC verification does not leak information through timing differences.
- **No plaintext secrets in logs**: The `tracing` integration should be used to log protocol events without exposing key material or raw message bodies.

## Integration Points

When consuming `librefang-wire` from other crates:

1. Depend on `librefang-wire` in `Cargo.toml`.
2. Use the transport trait implementations to establish connections to peer agents.
3. Construct messages using types from `librefang-types`, then pass them to the wire layer for signing and transmission.
4. Incoming messages arrive authenticated and deserialized, ready for application-level handling.