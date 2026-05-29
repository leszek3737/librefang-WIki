# Other — librefang-import-tests

# Idempotency & Forward-Compatibility Tests (`tests/idempotency.rs`)

End-to-end integration tests that verify the **filesystem-level** contract of the `librefang-import` migration crate. While unit tests inside `src/openclaw.rs` check that `report.imported.is_empty()` on a second run, these tests assert the stronger guarantee that the actual bytes on disk remain unchanged.

## What Is Being Tested

Three distinct properties:

| Property | Why It Matters |
|---|---|
| **Second-run is a no-op** | Callers expect to be able to re-drive a migration safely (e.g., in CI or retry loops). The destination tree must be byte-identical after a redundant run. |
| **Partial-write recovery** | If a process is killed mid-migration, re-running must recreate missing files without clobbering surviving ones. |
| **Legacy config forward-compat** | A `config_version = 1` TOML file must still deserialize into the current `KernelConfig` and round-trip through `run_migrations` to reach `CONFIG_VERSION`. |

## Test Inventory

### OpenClaw Idempotency

- **`openclaw_second_run_is_byte_identical`** — Runs `openclaw::migrate` twice against the same source/destination pair. Snapshots the entire destination tree after each run and asserts byte equality. The openclaw migrator uses a `.openclaw_migrated` marker file to short-circuit before any writes on the second run.

- **`openclaw_partial_write_is_recoverable`** — After a clean first run, deletes a specific file (the `coder/agent.toml` manifest, or a non-marker fallback) and removes the marker file, simulating a crash. Re-runs migration and asserts: (1) the deleted file is recreated with identical content, and (2) every other surviving file is unchanged. This validates the "never-clobber" semantics in `promote_staging` (issue #3795).

### OpenFang Idempotency

- **`openfang_second_run_is_byte_identical`** — Same shape as the openclaw version, but openfang has no marker file. Instead, each source file's existence at the destination is checked individually (`dest_path.exists()`). The second run must report zero imports and must list every first-run file as skipped.

- **`openfang_partial_write_is_recoverable`** — Deletes `agents/coder/agent.toml` after a clean run, then re-runs. Asserts the rewritten file is recreated with identical bytes and no other file was touched.

### Legacy Config Forward-Compatibility

- **`legacy_v1_config_parses_into_current_kernel_config`** — Loads `tests/fixtures/legacy_config/config_v1.toml` (which declares `config_version = 1` with the old `[api]` subtable). Asserts it deserializes into the current `KernelConfig` without error (relying on `#[serde(default)]` and unknown-field ignoring).

- **`legacy_v1_config_migrates_forward_to_current_version`** — Passes the same v1 fixture through `run_migrations(&mut raw, 1)` and asserts it reaches `CONFIG_VERSION`, the `[api]` table is removed, and `api_key`, `api_listen`, and `log_level` are hoisted to root level.

## Helper Functions

### `snapshot_tree(root: &Path) -> BTreeMap<PathBuf, Vec<u8>>`

Reads every regular file under `root` and returns a sorted map of relative paths to byte contents. Uses `BTreeMap` intentionally so that `assert_eq!` failures point at the first differing path in a deterministic order, rather than depending on `HashMap` insertion order.

### `walkdir_iter(root: &Path) -> Vec<PathBuf>`

A minimal recursive directory walker built on `std::fs::read_dir`. Exists to avoid pulling the `walkdir` crate into the test crate's dev-dependencies. Symlinks are deliberately ignored since neither migrator produces them and they reduce cross-platform snapshot stability.

### `write_openclaw_workspace(dir: &Path)`

Creates a minimal openclaw source fixture containing:
- `openclaw.json` — agent definitions (coder + researcher), channel config, memory/session settings
- `memory/coder/MEMORY.md` — per-agent memory
- `sessions/agent_coder_main.jsonl` — a session file (must not be duplicated on re-run)

Mirrors the shape from the in-crate `create_json5_workspace` helper in `src/openclaw.rs` but trimmed for readability when snapshots appear in test failure diffs.

### `write_openfang_workspace(dir: &Path)`

Creates a minimal openfang source fixture containing:
- `config.toml` — with `config_version`, `api_listen`, `log_level`, and `[default_model]`
- `secrets.env` — verbatim environment variable passthrough
- `agents/coder/agent.toml` — agent manifest (exercises rewrite path on recovery)
- `data/index.db` — binary blob

### `opts(source, src, dst) -> MigrateOptions`

Convenience constructor that builds a `MigrateOptions` with `dry_run: false`.

## Idempotency Mechanisms by Source Type

```mermaid
flowchart TD
    A["migrate() called"] --> B{Source type?}
    B -->|OpenClaw| C{Marker exists?}
    C -->|Yes| D["Short-circuit: return empty report"]
    C -->|No| E["Write all files, create marker"]
    B -->|OpenFang| F["For each source file"]
    F --> G{dest_path.exists()?}
    G -->|Yes| H["Skip (add to skipped list)"]
    G -->|No| I["Copy/rewrite to destination"]
```

## Dependencies on Other Crates

| Crate | Usage |
|---|---|
| `librefang-import` | The crate under test. Imports `openclaw`, `openfang`, `MigrateOptions`, `MigrateSource`. |
| `librefang-types` | Imports `config::KernelConfig`, `config::run_migrations`, `config::CONFIG_VERSION` for the forward-compat tests. |
| `tempfile` | `TempDir` for isolated source/destination directories. |

## Fixture Files

The legacy config fixture lives at `tests/fixtures/legacy_config/config_v1.toml` and is intentionally minimal — it contains only `config_version = 1` plus the `[api]` subtable with `api_key`, `api_listen`, and `log_level`. This is the smallest representation that exercises both raw deserialization into the current `KernelConfig` and the v1→v2 migration hoist. A complete v1 config is deliberately NOT constructed because the schema surface area has grown too large to safely reconstruct legacy defaults from this side.

## Relationship to Unit Tests

These tests complement — not replace — the in-crate unit tests in `src/openclaw.rs`. The unit tests assert on the `MigrateReport` structure (`imported.is_empty()`, `skipped` counts). The integration tests here assert on the actual filesystem state, catching bugs where the report says "nothing happened" but a file was silently rewritten (e.g., a timestamp regeneration in the marker body or a config normalization pass).