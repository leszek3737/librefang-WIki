# Memory & Storage — librefang-memory-src

# librefang-memory

The memory substrate for the LibreFang Agent Operating System. Provides a unified persistence layer over SQLite for structured data, semantic vector search, knowledge graphs, session history, and proactive (mem0-style) memory management.

All storage shares a single `r2d2` connection pool backed by one SQLite file in WAL mode. The module is consumed by the kernel, runtime, API routes, and the agent loop.

## Architecture

```mermaid
graph TD
    subgraph "MemorySubstrate (unified entry point)"
        direction LR
        A[StructuredStore] B[SemanticStore] C[KnowledgeStore]
        D[SessionStore] E[PromptStore] F[WorkflowStore]
        G[UsageStore] H[IdempotencyStore]
    end

    Pool[r2d2::Pool&lt;SqliteConnectionManager&gt;] --> A
    Pool --> B
    Pool --> C
    Pool --> D
    Pool --> E
    Pool --> F
    Pool --> G
    Pool --> H

    I[ProactiveMemoryStore] --> A
    I --> B
    I --> C

    J[ConsolidationEngine] --> Pool
    K[decay::run_decay] --> Pool

    B -->|delegates| L[VectorStore trait]
    L -->|impl| M[SqliteVectorStore]
    L -->|impl| N[HttpVectorStore]
```

## Module Layout

| File | Purpose |
|------|---------|
| `substrate.rs` | `MemorySubstrate` — owns the connection pool, wires all sub-stores together |
| `migration.rs` | Schema creation and migration ladder (v1–v41) |
| `session.rs` / `session_store.rs` | Session persistence, FTS, JSONL mirroring |
| `structured.rs` | Key-value store for agent state and config |
| `semantic.rs` | Semantic memory with embedding-based recall (`SqliteVectorStore`) |
| `http_vector_store.rs` | `VectorStore` impl delegating to a remote HTTP service |
| `knowledge.rs` | Entity-relation knowledge graph |
| `proactive.rs` | Unified proactive memory (search, add, auto-memorize, auto-retrieve) |
| `consolidation.rs` | Memory deduplication and confidence decay |
| `decay.rs` | TTL-based soft-deletion of stale memories |
| `chunker.rs` | Text chunking with overlap for embedding |
| `prompt.rs` | Prompt versioning and experiment management |
| `namespace_acl.rs` | Namespace-scoped read/write guards |
| `idempotency.rs` | Idempotency-key cache for state-creating POSTs |
| `workflow_store.rs` | Workflow run persistence |
| `usage.rs` | Token/cost usage tracking |
| `roster_store.rs` | Agent roster storage |
| `provider.rs` | Pluggable memory provider trait (`MemoryProvider`) |

## Text Chunking — `chunker`

Splits long documents into overlapping chunks suitable for embedding-based retrieval. The splitting cascade is:

1. **Paragraph boundaries** (`\n\n`) — preferred, preserves coherent blocks.
2. **Sentence boundaries** — when a single paragraph exceeds `max_size`. Recognizes `. ` / `.\n`, `。`, `？`, `！`.
3. **Hard character split** — last resort when a single sentence is still too long.

```rust
pub fn chunk_text(text: &str, max_size: usize, overlap: usize) -> Vec<String>
```

Overlap is applied by prepending the last `overlap` characters of the previous chunk to the next one, joined by `\n\n`. If the overlap + new segment exceeds `max_size`, the overlap is dropped.

Internally: `build_segments` → `split_sentences` → `split_by_char_limit` produces atomic segments, then `pack_with_overlap` greedily packs them into chunks.

All character counting uses `str::chars().count()` — Unicode-safe. The `char_boundaries` helper ensures hard splits never slice inside a multi-byte character.

## Memory Consolidation — `consolidation`

`ConsolidationEngine` runs periodic consolidation cycles that:

1. **Decay confidence** of memories not accessed in 7 days, multiplying by `(1 - decay_rate)` with a floor of `0.1`.
2. **Merge duplicates** — memories with >90% text similarity (Jaccard on lowercased words) are merged per-agent.

### Merge Semantics

When two memories are merged, the higher-confidence row (the *keeper*) absorbs the lower-confidence row (the *loser*) which is soft-deleted:

| Field | Merge rule |
|-------|-----------|
| `confidence` | `max(keeper, loser)` |
| `access_count` | Sum of both |
| `metadata` | JSON object union; keeper wins key conflicts. Non-object payloads (arrays, strings, malformed) on either side cause the keeper to be preserved verbatim. |
| `embedding` | Confidence-weighted average using a **running accumulated weight** (see below) |

### Running Weighted Average for Embeddings

A keeper that absorbs multiple losers tracks an accumulated weight that grows with each merge:

```
step 1: accum_weight = keeper.confidence
        merged = (keeper.emb × accum_weight + loser.emb × loser.conf) / (accum_weight + loser.conf)
        accum_weight += loser.confidence

step 2: merged = (merged.emb × accum_weight + loser2.emb × loser2.conf) / (accum_weight + loser2.conf)
        accum_weight += loser2.confidence
```

This produces a true average over all absorbed vectors rather than a chain of pairwise blends biased toward the last loser. Dimension-mismatched losers are skipped (keeper bytes preserved verbatim). When the keeper has no embedding but the loser does, the loser's vector is adopted (asymmetric — some embedding is better than none).

### Transaction Strategy

A single outer transaction wraps every merge in a consolidation run. This collapses up to `MAX_MERGES_PER_RUN` (100) fsyncs into one. Per-pair atomicity (loser soft-delete + keeper update) is preserved because both writes land in the same transaction. If any step fails, the `?` propagation rolls the entire batch back; consolidation is idempotent, so the next run picks up where it left off.

### Tenant Isolation

Memories are grouped by `agent_id` before comparison. Cross-tenant merges never occur, even when content is identical. The `SELECT` in the merge loop filters `WHERE deleted = 0 AND agent_id = ?1`.

## Time-Based Decay — `decay`

Scope-based TTL rules:

| Scope | Behavior |
|-------|----------|
| `user_memory` | Never decays — permanent user knowledge |
| `session_memory` | Soft-deleted after `session_ttl_days` of no access |
| `agent_memory` | Soft-deleted after `agent_ttl_days` of no access |

```rust
pub fn run_decay(pool: &Pool<SqliteConnectionManager>, config: &MemoryDecayConfig) -> LibreFangResult<usize>
```

Decay performs a **soft delete** (`deleted = 1`, `deleted_at = <unix timestamp>`) rather than a hard `DELETE`. Hard removal happens later via:

```rust
pub fn prune_soft_deleted_memories(pool: &Pool<SqliteConnectionManager>, older_than_days: u64) -> LibreFangResult<usize>
```

`prune_soft_deleted_memories` reclaims rows (and their embedding BLOBs) that have been soft-deleted for longer than the specified window. Rows with `deleted_at = NULL` are ignored.

Timestamp comparisons use `datetime()` on both sides so SQLite parses them properly rather than relying on lexicographic string comparison (which breaks across offset formats and fractional-second precision).

## HTTP Vector Store — `http_vector_store`

A `VectorStore` implementation that delegates all vector operations to a remote HTTP service, allowing external vector databases (Qdrant, Weaviate, custom microservices) without linking native clients.

### API Contract

| Method | Path | Request | Response |
|--------|------|---------|----------|
| POST | `/insert` | `{ id, embedding, payload, metadata }` | `{}` |
| POST | `/search` | `{ query_embedding, limit, filter? }` | `[{ id, payload, score, metadata }]` |
| DELETE | `/delete` | `{ id }` | `{}` |
| POST | `/get_embeddings` | `{ ids }` | `{ "<id>": [f32, ...], ... }` |

```rust
let store = HttpVectorStore::new("http://localhost:6333/collections/memories");
```

Trailing slashes are stripped. All errors are mapped to `LibreFangError::Internal` with the HTTP status and response body.

## Idempotency — `idempotency`

SQLite-backed idempotency-key cache for state-creating POSTs. Shares the substrate connection pool so no separate database file is needed.

- **TTL**: 24 hours (`TTL_SECONDS = 86400`)
- **First-writer-wins**: `INSERT OR IGNORE` — concurrent writes under the same key silently keep whichever landed first.
- **Self-trimming**: `lookup` deletes expired rows in place and reports them as a miss. `prune_expired` is called opportunistically by the middleware.

```rust
pub trait IdempotencyStore: Send + Sync {
    fn lookup(&self, key: &str) -> Result<Option<StoredRecord>, IdempotencyError>;
    fn put(&self, key: &str, body_hash: &str, response: &CachedResponse) -> Result<(), IdempotencyError>;
    fn prune_expired(&self) -> Result<(), IdempotencyError>;
}
```

`SqliteIdempotencyStore` is the production implementation. The trait enables in-memory substitution for unit tests in the API crate. Schema is created by migration v34.

Pool exhaustion is tracked via `metrics::counter!("librefang_memory_pool_get_failed_total")`.

## Knowledge Graph — `knowledge`

Entity-relation graph backed by SQLite. Supports:

- `add_entity` — upsert with conflict resolution (updates name/properties on conflict)
- `add_relation` — creates a typed, confidence-scored edge between two entities
- `query_graph` — pattern matching against source/relation/target with a 100-row limit
- `delete_by_agent` — transactional deletion of all entities and relations for an agent (relations deleted first to avoid orphans)
- `has_relation` — existence check, resolving by both ID and name

Relations can reference entities by either ID or name (the MCP tool uses names). The `query_graph` JOIN handles both:

```sql
JOIN entities s ON (r.source_entity = s.id OR (r.source_entity = s.name AND s.agent_id = r.agent_id))
```

### Corrupt JSON Handling

`parse_entity` and `parse_relation` **do not** silently default corrupt `properties` blobs to empty maps. A parse failure returns `LibreFangError::Serialization` and logs the row ID, table, and column. This makes corruption visible in logs and API responses rather than disguising it as "no properties."

## Schema Migration — `migration`

Manages the SQLite schema from version 1 through 41. Current version constant: `SCHEMA_VERSION = 41`.

### Ladder Execution

Each step runs in its own transaction that bundles the DDL, an audit-row `INSERT INTO migrations`, and the `user_version` pragma bump. The `run_step!` macro wraps this:

```rust
macro_rules! run_step {
    ($version:expr, $migrate_fn:expr) => {
        if current_version < $version {
            let tx = conn.unchecked_transaction()?;
            $migrate_fn(&tx)?;
            set_schema_version(&tx, $version)?;
            tx.commit()?;
        }
    };
}
```

### Safety Checks

1. **Downgrade refusal**: If `user_version > SCHEMA_VERSION`, migration aborts with a descriptive error — silently lowering `user_version` would cause subsequent ALTERs to corrupt the schema.

2. **Ladder consistency**: Before any DDL runs, the system verifies that `MAX(migrations.version)` does not exceed `pragma user_version`. If it does, the ladder is in an inconsistent state (audit rows exist for unapplied DDL), and migration refuses to proceed with recovery instructions.

3. **Audit-trail backfill**: After all steps complete, any missing audit rows are backfilled and a single `warn!` summarizes the rescue. This self-heals databases where earlier versions applied DDL without recording audit rows.

### Notable Migrations

| Version | Purpose |
|---------|---------|
| v1 | Core tables: agents, sessions, events, memories, structured_kv, pending_approvals |
| v29 | Adds `deleted_at` column for decay soft-deletes |
| v32 | Denormalized `sessions.message_count` for list performance |
| v33 | Rebuilds `sessions_fts` with unicode61 tokenizer + backfill |
| v34 | Idempotency-key cache table |
| v35–36 | Tool-use ID and deferred execution persistence for ACP |
| v37 | Workflow run persistence (replaces JSON file) |
| v38 | Backfill `approval_audit.second_factor_used` for pre-TOTP databases |
| v39 | Per-session model override |
| v40 | Fold bolted-on `ALTER TABLE agents ADD COLUMN` calls into the ladder |
| v41 | Composite indexes on `sessions(agent_id, updated_at)` and `audit_entries(agent_id, timestamp)` |

## Proactive Memory — `proactive`

Unified mem0-style API layer that sits atop the structured, semantic, and knowledge stores. Provides:

- **Auto-memorize**: Hook that extracts memories from conversation turns
- **Auto-retrieve**: Hook that injects relevant memories before response generation
- **Deduplication**: Cross-chat dedup at the user level (configurable)
- **Consolidation integration**: Delegates to `ConsolidationEngine`

Exposed types: `ProactiveMemory`, `ProactiveMemoryConfig`, `ProactiveMemoryHooks`, `ProactiveMemoryStore`, `MemoryExportItem`, `MemoryStats`.

## Re-exports

`lib.rs` re-exports the primary public API:

```rust
pub use substrate::MemorySubstrate;
pub use session_store::SessionStore;
pub use workflow_store::{WorkflowRunRow, WorkflowStore};
pub use semantic::SqliteVectorStore;
pub use http_vector_store::HttpVectorStore;
pub use namespace_acl::{MemoryNamespaceGuard, NamespaceGate};
pub use proactive::{MemoryExportItem, MemoryStats, ProactiveMemoryStore};
pub use prompt::PromptStore;
pub use provider::{MemoryError, MemoryManager, MemoryProvider, NullMemoryProvider};
```

Plus type re-exports from `librefang_types::memory`: `MemoryFilter`, `MemoryItem`, `VectorSearchResult`, `VectorStore`, `ExtractionResult`, etc.

## Integration Points

**Kernel** (`librefang-kernel`): calls `run_migrations` at boot, uses `MemorySubstrate` for all persistence, invokes `decay::run_decay` and `prune_soft_deleted_memories` on scheduled intervals.

**Agent Loop**: uses `ProactiveMemoryHooks` for auto-memorize/auto-retrieve, `Session` for conversation persistence.

**API Routes** (`librefang-api`): uses `IdempotencyStore` for POST dedup, `ProactiveMemoryStore` for the memory management endpoints, `NamespaceGate` for ACL enforcement.

**Metering** (`librefang-kernel-metering`): records `UsageRecord` entries for token/cost tracking.

**Runtime** (`librefang-runtime`): uses `Session` for compaction decisions and message preparation.