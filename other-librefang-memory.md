# Other — librefang-memory

# librefang-memory

Memory substrate for the LibreFang Agent OS.

## Overview

`librefang-memory` provides the persistence and state management layer for LibreFang agents. It abstracts storage operations behind a substrate that other modules can rely on for reading, writing, querying, and managing agent state across restarts and sessions.

The crate acts as a foundational library — higher-level agent logic depends on it, but it does not itself depend on other LibreFang crates beyond `librefang-types`.

## Role in the System

```
┌──────────────────────────┐
│    Agent Logic Layer      │
├──────────────────────────┤
│  librefang-memory (this)  │
├──────────────────────────┤
│    librefang-types        │
├──────────────────────────┤
│   SQLite / Filesystem     │
└──────────────────────────┘
```

This module sits between the agent logic and the underlying storage. It is responsible for durable state, working memory, and any structured data the agent needs to persist or recall.

## Key Dependencies and What They Suggest

| Dependency | Purpose in This Module |
|---|---|
| `rusqlite` | SQLite-backed persistent storage. Likely used for structured queries, indexes, and transactional state management. |
| `serde` / `serde_json` / `rmp-serde` | Serialization in both JSON and MessagePack formats. Suggests the module handles serialization of arbitrary agent state, possibly with different format choices for different storage contexts. |
| `sha2` | Cryptographic hashing. Used for content-addressed storage, integrity checks, or deduplication of stored entries. |
| `uuid` | Unique identifier generation for memory entries, sessions, or transactions. |
| `chrono` | Timestamps on memory entries, enabling time-based queries and expiration. |
| `tokio` / `async-trait` | Async runtime support. Storage operations are non-blocking to integrate with the agent's async event loop. |
| `reqwest` | HTTP client capability. May be used for remote memory backends, syncing state to a server, or fetching external data to cache locally. |
| `tracing` | Instrumented logging for storage operations, useful for debugging agent behavior. |
| `thiserror` | Typed error definitions for storage failures, corruption, or query errors. |

## Architecture

The module likely exposes one or more async traits or structs that represent a memory store. Consumers interact with these abstractions rather than raw SQL or file I/O.

### Expected Capabilities

Based on the dependency profile:

- **Key-value or document storage** backed by SQLite, with serialization handled transparently.
- **Content-addressed entries** using SHA-2 hashing for deduplication or integrity verification.
- **Time-indexed records** via `chrono` timestamps, supporting queries like "most recent" or "all entries since X."
- **UUID-tagged entries** for unique identification across distributed or multi-agent scenarios.
- **Error handling** with structured error types rather than panics or raw `Result<T, Box<dyn Error>>`.

## Integration Points

This crate depends on `librefang-types` for shared type definitions — likely the shapes of memory entries, agent identifiers, or configuration structs that flow between modules.

No other LibreFang crates depend on this module at the code level based on available call graph data, but at runtime, any module that needs durable state would bring it in as a dependency.

## Development

Run tests with:

```bash
cargo test -p librefang-memory
```

The `tempfile` dev-dependency is used in tests to create isolated temporary databases, ensuring tests don't pollute the filesystem or interfere with each other.