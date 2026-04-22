# Other — librefang-memory

# librefang-memory

Memory substrate for the LibreFang Agent OS.

## Purpose

`librefang-memory` provides the persistence and retrieval layer that underpins the agent's ability to store, query, and recall information across sessions. It abstracts the details of durable storage behind a substrate that the rest of the agent OS can rely on for any state that needs to outlive a single request or process lifecycle.

## Role in the Architecture

This crate sits below the agent logic and above the raw storage engine. Other modules in LibreFang consume the memory substrate when they need to persist conversation history, task state, configuration snapshots, or any derived data that should survive restarts.

```
┌─────────────────────────┐
│   Agent Logic / Core    │
├─────────────────────────┤
│   librefang-memory      │  ← this crate
├─────────────────────────┤
│   librefang-types       │
├─────────────────────────┤
│   SQLite (rusqlite)     │
└─────────────────────────┘
```

## Key Dependencies and What They Enable

| Dependency | Role in this crate |
|---|---|
| `rusqlite` | Primary storage backend. SQLite provides an embedded, transactional database without external service dependencies, making it suitable for single-node agent deployments. |
| `serde` / `serde_json` / `rmp-serde` | Serialization of structured memory entries. JSON supports human-readable inspection and debugging; MessagePack (`rmp-serde`) provides compact binary encoding for larger or more performance-sensitive payloads. |
| `sha2` | Content-addressable hashing. Likely used to fingerprint memory entries for deduplication or integrity verification. |
| `uuid` | Unique identification of memory records, ensuring globally unique references even across distributed instances. |
| `chrono` | Timestamping of memory entries for temporal queries and ordering. |
| `tokio` / `async-trait` | Async runtime integration. The memory substrate exposes async interfaces so that I/O-bound storage operations do not block the agent's task scheduler. |
| `reqwest` | HTTP client, likely used for remote memory backends or synchronization with external services. |
| `tracing` | Structured logging and diagnostics for storage operations. |
| `thiserror` | Ergonomic error type definitions for substrate-specific failures. |

## Design Expectations

### Storage Model

The substrate is built on SQLite via `rusqlite`, indicating a local, file-based persistence model. This choice avoids the operational overhead of running a separate database server while still providing ACID guarantees for memory records.

### Serialization Strategy

The presence of both `serde_json` and `rmp-serde` suggests that the module supports multiple encoding formats. A typical pattern would be:

- **JSON** for entries that benefit from human readability and tooling compatibility (configuration, metadata).
- **MessagePack** for high-volume or binary-heavy data where storage size and serialization speed matter.

### Content Addressing

The `sha2` dependency points to content-addressable storage patterns. Memory entries are likely hashed so that identical content maps to the same identifier, enabling deduplication and integrity checks on retrieval.

### Async Interface

`async-trait` combined with `tokio` indicates that the substrate defines async trait bounds for its storage operations. This allows consumers to `await` reads and writes without blocking, and makes it possible to swap backends (local SQLite vs. remote HTTP) behind a common interface.

## Relationship to `librefang-types`

This crate depends on `librefang-types` for shared data structures. Memory entries, error types, and configuration shapes are defined in the types crate, keeping this module focused on storage mechanics rather than domain modeling.

## Testing

Dev-dependencies include `tokio-test` for async test harnessing and `tempfile` for creating ephemeral SQLite databases during tests. This allows tests to spin up a full substrate backed by a temporary file, exercise it, and clean up without side effects.

## Integration Points

Other LibreFang modules import `librefang-memory` when they need to:

- **Persist state** that must survive process restarts.
- **Recall prior context** to maintain continuity across agent interactions.
- **Query historical data** for reporting, debugging, or decision-making.

The substrate hides whether data lives locally in SQLite or is fetched over HTTP via `reqwest`, presenting a uniform async interface to consumers.