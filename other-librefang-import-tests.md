# Other — librefang-import-tests

# librefang-import/tests/idempotency.rs

End-to-end idempotency and forward-compatibility tests for the `librefang-import` migration crate. Filed against issue #3407.

## Why This File Exists

The unit-level idempotency checks in `src/openclaw.rs` assert that `report.imported.is_empty()` on a second migration run. That's necessary but not sufficient. Callers depend on a stronger filesystem-level contract: running the same migration twice must produce a **byte-identical** destination tree — no duplicate sessions, no clobbered configs, no rewritten timestamps.

This test module sits one level above the unit tests and verifies three properties:

1. **Idempotency** — a clean second run is a complete no-op on disk.
2. **Crash recovery** — a partially-completed migration (simulating a killed process) can be re-driven to a correct state without corrupting surviving entries.
3. **Forward compatibility** — the prior major version's `KernelConfig` shape still deserializes into the current type and round-trips through `run_migrations`.

## Test Categories

### A. Byte-level idempotency

| Test | Source type | Short-circuit mechanism |
|---|---|---|
| `openclaw_second_run_is_byte_identical` | `MigrateSource::OpenClaw` | `.openclaw_migrated` marker file |
| `openfang_second_run_is_byte_identical` | `MigrateSource::OpenFang` | Per-entry `dest_path.exists()` skip |

Both tests:
1. Write a representative fixture workspace into a temp directory.
2. Run the migration once, snapshot the entire destination tree.
3. Run the migration again, snapshot a second time.
4. Assert `snapshot_a == snapshot_b` (byte-identical contents, not just file existence).

For OpenFang, the second run additionally verifies that every previously-imported entry appears in `report.skipped` (the count must match the first run's `report.imported` count).

### B. Partial-write recovery

| Test | Source type | Crash simulation |
|---|---|---|
| `openclaw_partial_write_is_recoverable` | `MigrateSource::OpenClaw` | Delete victim file + marker |
| `openfang_partial_write_is_recoverable` | `MigrateSource::OpenFang` | Delete victim file only |

These tests simulate a migration that died mid-write by:
1. Running a full migration and snapshotting the baseline.
2. Deleting a single "victim" file (and the `.openclaw_migrated` marker for OpenClaw, since it blocks re-runs).
3. Re-running the migration.
4. Asserting the victim file is recreated with identical byte content.
5. Asserting no surviving file was clobbered (the `promote_staging` never-clobber semantics from #3795).

The OpenClaw variant selects the victim deterministically: it prefers an agent manifest (`agent.toml` containing "coder"), falling back to any non-marker file.

The OpenFang variant always targets `agents/coder/agent.toml` — a rewritten file — to exercise the rewrite path during recovery.

### C. Forward compatibility

| Test | What it asserts |
|---|---|
| `legacy_v1_config_parses_into_current_kernel_config` | v1 TOML deserialises into current `KernelConfig` via `#[serde(default)]` |
| `legacy_v1_config_migrates_forward_to_current_version` | `run_migrations(_, 1)` hoists `[api]` fields to root and reaches `CONFIG_VERSION` |

The fixture lives at `tests/fixtures/legacy_config/config_v1.toml` and contains only the minimum fields needed to exercise the v1→v2 migration path (`config_version` plus the `[api]` table with `api_key`, `api_listen`, `log_level`).

The migration test asserts that `run_migrations`:
- Removes the `[api]` table.
- Hoists `api_key`, `api_listen`, and `log_level` to root level.
- Returns the current `CONFIG_VERSION`.

## Internal Helpers

### `snapshot_tree(root: &Path) -> BTreeMap<PathBuf, Vec<u8>>`

Reads every regular file under `root` and returns a sorted map of relative path → byte contents. Uses `BTreeMap` so that `assert_eq!` failures point at the first differing path deterministically.

### `walkdir_iter(root: &Path) -> Vec<PathBuf>`

A minimal recursive directory walker built on `std::fs::read_dir`. Exists to avoid pulling `walkdir` into the test crate's dev-dependencies. Symlinks are deliberately ignored since neither migrator produces them and they'd introduce platform-dependent snapshot instability.

### `write_openclaw_workspace(dir: &Path)`

Creates a minimal but representative OpenClaw source workspace:
- `openclaw.json` — JSON5 agent/channel/memory/session config with two agents ("coder" and "researcher")
- `memory/coder/MEMORY.md` — per-agent memory file
- `sessions/agent_coder_main.jsonl` — a session file (idempotency must not duplicate it)

### `write_openfang_workspace(dir: &Path)`

Creates a minimal OpenFang source workspace:
- `config.toml` — config with version, listen address, log level, and a default model
- `secrets.env` — environment file that must be preserved verbatim
- `agents/coder/agent.toml` — agent manifest (rewritten during migration)
- `data/index.db` — binary blob (exercises the raw-copy path)

### `opts(source, src, dst) -> MigrateOptions`

Constructs `MigrateOptions` with `dry_run: false` for the given source type and directory pair.

## Relationships to Other Crates

```
┌──────────────────────────────────┐
│  tests/idempotency.rs            │
│  (this file)                     │
├──────────┬───────────────────────┤
│  calls   │  calls                │
│  down    │  across               │
└────┬─────┴──────────┬────────────┘
     ▼                ▼
┌─────────────┐  ┌──────────────────────┐
│ librefang-  │  │ librefang-types      │
│ import      │  │  config::KernelConfig│
│  openclaw:: │  │  config::run_migr... │
│  migrate()  │  │  config::CONFIG_VER. │
│  openfang:: │  └──────────────────────┘
│  migrate()  │
│  MigrateOpt.│
│  MigrateSrc │
└─────────────┘
```

- **`librefang-import`**: provides `openclaw::migrate`, `openfang::migrate`, `MigrateOptions`, and `MigrateSource` — the public migration API under test.
- **`librefang-types`**: provides `KernelConfig`, `run_migrations`, and `CONFIG_VERSION` — used by the forward-compat tests to verify v1 configs still parse and migrate correctly.

## Running

```sh
# All idempotency tests
cargo test -p librefang-import --test idempotency

# Individual test
cargo test -p librefang-import --test idempotency -- openclaw_second_run_is_byte_identical
```