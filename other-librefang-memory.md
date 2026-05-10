# Other — librefang-memory

# librefang-memory

Memory substrate for the LibreFang Agent OS. Provides persistent, queryable memory for agents through a unified trait that abstracts over three storage backends: structured key-value (SQLite), semantic text search, and a knowledge graph.

## Architecture

Agents interact with a single `Memory` trait rather than individual storage backends. Implementations of this trait compose the three stores into one `MemorySubstrate`.

```
┌─────────────────────────────────┐
│           Memory trait          │
│  search · add · get · list · …  │
└──────────────┬──────────────────┘
               │
       MemorySubstrate
               │
   ┌───────────┼───────────┐
   ▼           ▼           ▼
┌──────┐  ┌────────┐  ┌──────────┐
│Structured│Semantic │  │Knowledge │
│ (SQLite) │(LIKE /  │  │ (SQLite) │
│          │Qdrant)  │  │          │
└──────────┘└────────┘  └──────────┘
```

## Storage Backends

### Structured Store (`structured`)

SQLite-backed storage for:

- **Key/value pairs** — arbitrary agent state serialized via serde (JSON or MessagePack via `rmp-serde`).
- **Sessions** — conversation and interaction context scoped to an agent run or user session.
- **Audit trail** — append-only log of agent actions for traceability.

Uses `r2d2` connection pooling (`r2d2_sqlite`) for concurrent access.

### Semantic Store (`semantic`, `http_vector_store`)

Full-text search over stored text. Two paths:

- **Current** — `LIKE`-based search in SQLite. Suitable for development and small deployments.
- **Vector path** — `http_vector_store` targeting Qdrant for embedding-based similarity search. Not yet the default but available for integration.

### Knowledge Graph (`knowledge`)

SQLite-backed entity-relation store. Agents can record entities (nodes) and typed relations (edges) between them, then query the graph for multi-hop reasoning.

## Proactive Memory (`proactive`)

Inspired by mem0, proactive memory automatically decides what to store, when to consolidate, and what to forget. This is the primary high-level API agents should use.

### Core Types

| Type | Role |
|---|---|
| `ProactiveMemory` | Unified API: `search`, `add`, `get`, `list`. The main entry point. |
| `ProactiveMemoryHooks` | Auto-memorize and auto-retrieve hooks. Wire these into an agent's request/response pipeline so memory operations happen transparently. |
| `ProactiveMemoryStore` | Concrete implementation backed by `MemorySubstrate`. |

### Supporting Modules

- **`chunker`** — Splits incoming text into memory-sized chunks before storage.
- **`consolidation`** — Merges redundant or near-duplicate memories over time to keep the store clean.
- **`decay`** — Time-based relevance scoring. Older or less-accessed memories lose priority and may be pruned.
- **`migration`** — Schema migrations for the underlying SQLite databases. Run on startup to bring the store to the current version.
- **`namespace_acl`** — Access control per namespace. Ensures agents can only read/write memory they are authorized for.
- **`prompt`** — Prompt templates for memory-related LLM calls (e.g., deciding whether a piece of text is worth storing).
- **`provider`** — Abstraction over the LLM provider used by consolidation and relevance scoring.
- **`roster_store`** — Tracks which agents exist and their capabilities, used for cross-agent memory sharing decisions.
- **`session`** — Session lifecycle management tied to the structured store.

## Dependencies

| Crate | Purpose |
|---|---|
| `librefang-types` | Shared domain types exchanged across workspace crates |
| `tokio` | Async runtime |
| `rusqlite` (FTS5) | SQLite engine for structured, knowledge, and semantic stores |
| `r2d2` / `r2d2_sqlite` | Connection pooling |
| `serde` / `serde_json` / `rmp-serde` | Serialization (JSON and MessagePack) |
| `chrono` | Timestamps for audit trail and decay |
| `uuid` | Unique IDs for entities, relations, memory entries |
| `sha2` | Content hashing for deduplication |
| `reqwest` | HTTP client for the Qdrant vector store path |
| `tracing` | Structured logging |
| `metrics` | Instrumentation counters and histograms |

## Usage

```rust
use librefang_memory::proactive::{ProactiveMemory, ProactiveMemoryStore, ProactiveMemoryHooks};

// Create the substrate (backed by SQLite on disk)
let substrate = MemorySubstrate::open("agent.db")?;
let store = ProactiveMemoryStore::new(substrate);

// Add a memory
store.add("agent-1", "The user prefers dark mode", None).await?;

// Search
let results = store.search("agent-1", "user preferences").await?;

// Wire hooks into your agent pipeline for automatic memory
let hooks = ProactiveMemoryHooks::new(store);
// hooks.on_request(...) → auto-retrieve
// hooks.on_response(...) → auto-memorize
```

## Database Schema

All SQLite-backed stores are initialized via the `migration` module on first open. The module tracks the current schema version in a `_meta` table and applies pending migrations sequentially. No manual setup is required — calling `MemorySubstrate::open(path)` handles everything.

## Testing

Tests use the `tempfile` crate to create throwaway database files:

```rust
let dir = tempfile::tempdir()?;
let db_path = dir.path().join("test.db");
let substrate = MemorySubstrate::open(&db_path)?;
// ... run assertions
```

## Relationship to Other Crates

`librefang-memory` sits below the agent runtime and above `librefang-types`. Agents import the `Memory` trait or `ProactiveMemory` and remain unaware of which backend handles their request. The `librefang-types` crate defines the data structures (entries, entities, relations) that flow through the memory API.