# Other — librefang-memory

# librefang-memory

Memory substrate for the LibreFang Agent OS.

## Overview

`librefang-memory` provides persistent storage and state management for the LibreFang agent system. It acts as the substrate layer that other LibreFang components build upon to read, write, query, and manage durable agent state across sessions.

The module is positioned as a foundational crate — higher-level agent logic depends on it, not the other way around.

## Role in the System

The crate's description as a "memory substrate" means it is responsible for the mechanisms by which agents remember and recall information. This includes persistent data storage, serialization of agent state, and retrieval interfaces that other LibreFang modules consume.

```
┌─────────────────────────────────────┐
│        LibreFang Agent OS           │
│                                     │
│  ┌───────────┐    ┌──────────────┐  │
│  │ Agent     │───▶│  librefang-  │  │
│  │ Logic     │    │  memory      │  │
│  └───────────┘    └──────┬───────┘  │
│                          │          │
│                   ┌──────▼───────┐  │
│                   │  librefang-  │  │
│                   │  types       │  │
│                   └──────────────┘  │
└─────────────────────────────────────┘
```

## Dependency Analysis

The crate's dependencies reveal its architectural responsibilities:

### Storage

| Dependency | Purpose |
|---|---|
| `rusqlite` | Embedded SQLite database for structured, queryable persistent storage |

### Serialization

| Dependency | Purpose |
|---|---|
| `serde` | Serialization framework (core trait derivations) |
| `serde_json` | JSON serialization for human-readable or interoperable state |
| `rmp-serde` | MessagePack serialization for compact binary storage |

The dual serialization support (JSON and MessagePack) suggests the module supports both human-inspectable storage formats and space-efficient binary formats, likely selectable per context.

### Async Runtime

| Dependency | Purpose |
|---|---|
| `tokio` | Async runtime for non-blocking storage operations |
| `async-trait` | Async trait definitions for storage backend abstractions |

### Identity and Integrity

| Dependency | Purpose |
|---|---|
| `uuid` | Unique identifiers for memory entries, sessions, or records |
| `sha2` | Cryptographic hashing for data integrity verification or content-addressed storage |
| `chrono` | Timestamp management for temporal queries and record ordering |

### Networking

| Dependency | Purpose |
|---|---|
| `reqwest` | HTTP client — likely for remote memory backends or synchronization with external services |

### Observability and Error Handling

| Dependency | Purpose |
|---|---|
| `tracing` | Structured logging and instrumentation of storage operations |
| `thiserror` | Derived error types for idiomatic error propagation |

### Internal Dependency

| Dependency | Purpose |
|---|---|
| `librefang-types` | Shared type definitions used across the LibreFang workspace |

## Build Configuration

The module is part of the LibreFang workspace and inherits its root-level settings for version, edition, and license.

```toml
version.workspace = true
edition.workspace = true
license.workspace = true
```

### Test Dependencies

| Dependency | Purpose |
|---|---|
| `tokio-test` | Async test utilities |
| `tempfile` | Temporary directories and files for isolated database tests |

The presence of `tempfile` as a test dependency confirms that tests create isolated SQLite databases in temporary locations rather than touching the filesystem.

## Development Notes

- **Backend abstraction**: The combination of `async-trait` and `rusqlite` suggests the module likely defines async storage traits with a SQLite-backed implementation. The `reqwest` dependency hints at potential remote backend support or synchronization.

- **Content-addressing**: The `sha2` dependency, combined with `uuid` and `serde`, suggests some memory entries may be content-addressed — keyed by hash rather than just sequential or random identifiers.

- **Testing strategy**: Use `tempfile` to create ephemeral database paths when writing tests. Initialize a fresh database for each test case to ensure isolation.