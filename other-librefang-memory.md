# Other — librefang-memory

# librefang-memory

**Memory substrate for the LibreFang Agent OS**

## Purpose

`librefang-memory` provides the persistence and retrieval layer for agent state within the LibreFang Agent OS. It serves as the memory backbone—responsible for storing, querying, and managing the data that agents produce and consume during their lifecycle. This includes conversation history, task state, configuration snapshots, and any other structured data that must survive across sessions or be shared between agents.

The crate is designed as a **substrate**, meaning it offers low-level storage primitives that higher-level agent logic builds upon, rather than prescribing a specific memory model.

## Architecture

The module draws on several key dependencies to fulfill its role:

| Dependency | Role in this crate |
|---|---|
| `rusqlite` | Primary on-disk storage engine via embedded SQLite |
| `tokio` + `async-trait` | Async I/O interface for non-blocking storage operations |
| `serde` / `serde_json` / `rmp-serde` | Serialization of memory entries (JSON and MessagePack formats) |
| `sha2` | Content-addressable hashing for deduplication or integrity verification |
| `chrono` | Timestamping memory entries |
| `uuid` | Unique identification of memory records |
| `reqwest` | Remote memory synchronization or retrieval over HTTP |
| `thiserror` | Typed error definitions |
| `tracing` | Instrumentation and diagnostic logging |

### Storage Model

The embedded SQLite database provides the core storage mechanism. This choice gives the substrate:

- **Zero-configuration persistence** — no external database service required.
- **Transactional safety** — writes are atomic, supporting crash recovery.
- **Query flexibility** — complex lookups via SQL without writing custom index logic.

Serialization support for both JSON and MessagePack (`rmp-serde`) suggests the substrate can store entries in either a human-readable form (useful for debugging and interoperability) or a compact binary form (useful for performance-sensitive paths).

### Content Addressing

The inclusion of `sha2` indicates that memory entries may be content-addressed—identified by their cryptographic hash rather than solely by UUID. This enables:

- Deduplication of identical entries.
- Integrity checks on retrieval.
- Stable references to immutable data.

### Remote Capabilities

The `reqwest` dependency implies the memory substrate is not purely local. Possible use cases include:

- Syncing agent memory to a remote store.
- Fetching shared memory or configuration from a central service.
- Retrieving pre-seeded memory snapshots for agent initialization.

## Integration with the Codebase

```
librefang-types  ←  librefang-memory
                        ↑
                   Higher-level crates
                   (agent runtime, etc.)
```

`librefang-memory` depends on `librefang-types` for shared type definitions (error types, domain models, configuration structs). It exposes storage APIs that higher-level crates in the workspace consume to persist and retrieve agent data.

Because no incoming or outgoing internal calls were detected, this module operates as a **leaf dependency** within the workspace—other crates call into it, but it does not call out to sibling LibreFang crates beyond `librefang-types`.

## Development

### Building

```sh
cargo build -p librefang-memory
```

### Testing

Tests use `tempfile` (listed in dev-dependencies) for isolated, ephemeral SQLite databases:

```sh
cargo test -p librefang-memory
```

Each test should create a temporary directory, initialize a fresh database instance, run assertions, and let the temporary resource clean up on drop. This avoids test pollution and eliminates the need for manual cleanup logic.

### Adding New Storage Operations

When extending the substrate:

1. **Define the schema** — add any new tables or indexes as migrations run at database initialization.
2. **Use async wrappers** — expose operations through `async-trait` implementations so callers remain non-blocking.
3. **Serialize with serde** — any complex Rust type stored in the database should derive `Serialize` / `Deserialize`.
4. **Trace operations** — annotate significant storage calls with `tracing` spans for observability in production.
5. **Return typed errors** — use `thiserror`-derived error enums rather than generic `Box<dyn Error>`.