# Other — librefang-runtime-audit-src

# librefang-runtime-audit

Tamper-evident audit trail for the Librefang runtime. Every agent action — tool invocation, shell execution, role change, network access — is recorded into a hash-linked Merkle chain that can be verified at any time. The chain is persisted to SQLite and optionally anchored to an external file, making both in-place tampering and full-history rewrites detectable.

## Architecture

```mermaid
graph TD
    A[AuditLog::record] --> B[Read tip hash]
    B --> C[Build AuditEntry with prev_hash = tip]
    C --> D[compute_entry_hash over all fields]
    D --> E[Append to in-memory entries]
    E --> F[INSERT into SQLite inside BEGIN IMMEDIATE]
    F --> G[Write anchor file via atomic rename]
    G --> H[Advance tip]

    subgraph Verification
        V[verify_integrity] --> V1[Walk entries: each.prev_hash == prev_hash["prev.hash"]]
        V1 --> V2[Check chain_anchor or genesis sentinel at head]
        V2 --> V3[Compare tip against anchor file]
    end
```

## Core Types

### AuditLog

The primary interface. Construct it in one of three modes:

| Constructor | Persistence | Anchor | Use case |
|---|---|---|---|
| `AuditLog::new()` | In-memory only | None | Testing, ephemeral runs |
| `AuditLog::with_db(pool)` | SQLite via r2d2 pool | None | Standard daemon |
| `AuditLog::with_db_anchored(pool, path)` | SQLite | External file | Hardened deployments |

**Recording entries:**

```rust
// Minimal — no RBAC attribution
log.record("agent-1", AuditAction::ToolInvoke, "read_file /etc/hosts", "ok");

// With user and channel attribution (RBAC-aware)
log.record_with_context(
    "agent-1",
    AuditAction::PermissionDenied,
    "/api/admin/users",
    "denied",
    Some(UserId::from_name("Alice")),
    Some("api".to_string()),
);
```

Both methods return the hash of the newly appended entry.

### AuditEntry

Each entry carries:

- `seq: u64` — monotonically increasing sequence number
- `timestamp: String` — RFC 3339
- `agent_id: String`
- `action: AuditAction`
- `detail: String`
- `outcome: String` — typically `"ok"` or `"denied"`
- `user_id: Option<UserId>` — the human who triggered the action
- `channel: Option<String>` — how the request arrived (e.g. `"api"`, `"telegram"`)
- `prev_hash: String` — hash of the preceding entry (64-hex-char SHA-256)
- `hash: String` — hash of *this* entry over all fields including `prev_hash`

The first entry's `prev_hash` is the 64-zero genesis sentinel: `"0000…0000"`.

### AuditAction

Locked enum — the `Display` representation of every variant is part of the committed hash. Renaming any variant invalidates all persisted entries that reference it.

| Variant | Category |
|---|---|
| `ToolInvoke` | Agent operations |
| `ShellExec` | |
| `AgentSpawn` | |
| `AgentKill` | |
| `AgentMessage` | |
| `NetworkAccess` | |
| `MemoryAccess` | |
| `ConfigChange` | |
| `UserLogin` | RBAC (M5) |
| `RoleChange` | |
| `PermissionDenied` | |
| `BudgetExceeded` | |
| `RetentionTrim` | Self-audit (M7) |

### UserId

Deterministic identifier derived from a human-readable name via `UserId::from_name("Alice")`. The mapping is stable across process restarts, so audit attribution survives daemon cycling.

## Merkle Chain Integrity

### How hashing works

`compute_entry_hash` hashes all fields of an entry — `seq`, `timestamp`, `agent_id`, `action`, `detail`, `outcome`, `user_id`, `channel`, and `prev_hash` — into a single SHA-256 digest. Any change to any field produces a different hash, breaking the chain at that point.

### Verification

`verify_integrity()` walks the chain and checks:

1. The first surviving entry's `prev_hash` equals either the genesis sentinel or the in-memory `chain_anchor` (set after prefix trims).
2. Every subsequent entry's `prev_hash` equals the preceding entry's `hash`.
3. If an anchor file is configured, the on-disk tip matches the computed tip.

Returns `Ok(())` on success, or `Err(String)` describing where the break occurred (e.g. `"hash mismatch at seq 1"`, `"audit anchor mismatch"`).

### External anchor file

The linked-list structure alone proves internal consistency but cannot detect a full rewrite: an attacker who can write to the SQLite database could wipe every row, fabricate a new history, and recompute all hashes from the genesis sentinel forward. The anchor file defeats this by storing the chain tip out-of-band (outside the database).

Behavior:

- On first boot with an existing DB but no anchor file, `with_db_anchored` **seeds** the anchor from the current tip — this is the upgrade path.
- On subsequent boots, the anchor must exist and match the DB tip.
- If the anchor file is missing after being seeded, verification **fails closed** (returns an error containing `"missing"`).
- The anchor is updated on every `record` call via atomic rename (write to `.tmp`, then `fs::rename`).

### Concurrent writes

The INSERT path runs inside `BEGIN IMMEDIATE` to serialize chain appends at the SQLite layer. This prevents races where two threads read the same tip, build entries with the same `prev_hash`, and create a chain fork. The r2d2 pool may have multiple connections, but SQLite's write serialization ensures only one append is in flight at a time.

### DB failure handling

If the SQLite INSERT fails (e.g. table dropped, disk full), the in-memory chain **does not advance**. The entry is not added to the in-memory buffer, and the tip hash stays at its previous value. This prevents chain-break-on-restart bugs where an in-memory-only entry's hash was referenced by a later persisted entry whose predecessor never reached disk.

## Retention and Pruning

### Per-action retention (`trim`)

`trim(policy: &AuditRetentionConfig, now: DateTime<Utc>)` drops aged entries according to per-action day counts:

```rust
let mut policy = AuditRetentionConfig::default();
policy.retention_days_by_action.insert("ToolInvoke".to_string(), 7);
let report = log.trim(&policy, chrono::Utc::now());
```

`AuditRetentionConfig` fields:

| Field | Type | Default | Purpose |
|---|---|---|---|
| `retention_days_by_action` | `HashMap<String, u64>` | empty (keep forever) | Days to retain per action name |
| `max_in_memory_entries` | `Option<usize>` | `None` | Hard cap on in-memory window |

**Trim is prefix-only.** It scans from the head of the log and drops consecutive entries that exceed their per-action retention window. It stops at the first entry that should be kept (because its action has no rule, or it's within its window). This means entries *after* a kept entry are never dropped even if they're older than their own rule — this preserves the chain structure.

When a prefix is dropped, `chain_anchor` is set to the hash of the last dropped entry. The first surviving entry's `prev_hash` still points to that dropped entry's hash, and `verify_integrity()` uses the anchor as the trusted starting point instead of requiring the genesis sentinel.

`TrimReport` fields:
- `total_dropped: usize`
- `dropped_by_action: HashMap<String, usize>`
- `new_chain_anchor: Option<String>` — hash of the last dropped entry

The `trim()` method does **not** write a self-audit row. The caller (the kernel periodic task) is responsible for recording a `RetentionTrim` entry after a non-empty trim.

### Legacy day-based pruning (`prune`)

`prune(max_age_days: u64) -> usize` drops entries older than `max_age_days` regardless of action type. It also updates `chain_anchor` and persists deletions to SQLite, matching the semantics of `trim`.

### In-memory soft cap

When `max_in_memory_entries` is configured, the append path enforces a soft cap at `configured × 1.5` (integer arithmetic: `cap * 3 / 2`). If the in-memory buffer exceeds this threshold during `record`, the oldest prefix is evicted and `chain_anchor` is advanced. This prevents unbounded memory growth between periodic `trim` calls.

When `max_in_memory_entries` is `None` or `0`, the hard cap of 10,000 entries applies (`MAX_AUDIT_ENTRIES`).

### Persistence across restart

`with_db` recovers the `chain_anchor` from the surviving rows on load: if the first entry's `prev_hash` is not the genesis sentinel, it becomes the anchor. This means a daemon restart after a trim that dropped the entire prefix will still pass `verify_integrity()` without an external anchor file.

The "drop everything" edge case — where every entry is older than its retention rule — is handled explicitly: the DB is fully cleared, no orphan tail row remains, and the next `record` call anchors against the in-memory `chain_anchor`.

## Cursor-based Streaming

### `since_seq(cursor: u64) -> Vec<AuditEntry>`

Returns all entries with `seq > cursor`. Used by the SSE log stream (`/api/logs/stream`) to deliver incremental updates to consumers.

Key semantics:
- The cursor represents the highest seq the consumer has already received.
- `since_seq(0)` does **not** return `seq == 0` — initial backfill is handled by `recent()`.
- Returns an empty vec when the cursor is at or past the current tail.
- No burst-size limit: delivers every entry after the cursor regardless of volume.

### `recent(n: usize) -> Vec<AuditEntry>`

Returns the last `n` entries. Used for initial backfill in the SSE handler and for inspection in tests.

## SQLite Schema

```sql
CREATE TABLE audit_entries (
    seq INTEGER PRIMARY KEY,
    timestamp TEXT NOT NULL,
    agent_id TEXT NOT NULL,
    action TEXT NOT NULL,       -- AuditAction Display string
    detail TEXT NOT NULL,
    outcome TEXT NOT NULL,
    user_id TEXT,               -- UserId Display string, nullable
    channel TEXT,               -- nullable
    prev_hash TEXT NOT NULL,    -- 64-hex-char SHA-256
    hash TEXT NOT NULL          -- 64-hex-char SHA-256
);
```

The recommended connection initialization:

```rust
SqliteConnectionManager::file(db_path).with_init(|c| {
    c.execute_batch(
        "PRAGMA journal_mode=WAL;
         PRAGMA busy_timeout=5000;
         PRAGMA synchronous=NORMAL;"
    )
})
```

## Threat Model Summary

| Attack | Detection mechanism |
|---|---|
| Modify a single entry's field | Hash mismatch at that seq during `verify_integrity()` |
| Delete a middle entry | Chain break — `prev_hash` of the next entry dangles |
| Full DB rewrite with recomputed hashes | External anchor file mismatch |
| Delete the anchor file | Fails closed — `"missing"` error |
| Race concurrent appends to fork the chain | `BEGIN IMMEDIATE` serializes writes |
| DB INSERT failure leaves phantom in-memory entry | In-memory chain does not advance on failure |