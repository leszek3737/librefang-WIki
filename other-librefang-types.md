# Other — librefang-types

# librefang-types

Shared data structures for the LibreFang Agent OS. Every other workspace crate depends on this one. **Contains no business logic** — pure types, derive-only helpers, and error enumerations.

## Position in the dependency graph

This crate sits at the absolute bottom. It imports external libraries only (`serde`, `chrono`, `uuid`, `thiserror`, `schemars`, etc.) and zero workspace crates. Every other `librefang-*` crate may import from here; this crate imports from nothing in the workspace.

```
┌──────────────────────┐
│   librefang-api      │
│   librefang-kernel   │
│   librefang-runtime  │
│   ...                │
└────────┬─────────────┘
         │ imports
         ▼
┌──────────────────────┐
│  librefang-types     │  ← bottom of DAG
└────────┬─────────────┘
         │ imports
         ▼
   serde, chrono, uuid,
   thiserror, schemars, ...
```

## Public modules

| Module | Domain |
|---|---|
| `agent` | Agent identity and descriptors |
| `approval` | Human-approval gating types |
| `capability` | Permission/capability tokens |
| `comms` | Inter-agent communication types |
| `config` | Configuration structs (including `KernelConfig`) |
| `error` | `LibreFangError` and related error enums |
| `event` | Event types emitted by the kernel |
| `goal` | Goal/task representation |
| `i18n` | Internationalization types |
| `manifest_signing` | Signed manifest verification types |
| `media` | Media/attachment types |
| `memory` | Memory substrate types |
| `message` | Chat/message types |
| `model_catalog` | Model registry types |
| `oauth` | OAuth credential types |
| `registry_schema` | Registry payload schemas |
| `scheduler` | Task scheduler types |
| `serde_compat` | Serde helpers for cross-format compatibility |
| `subagent` | Sub-agent spawning types |
| `taint` | Taint-tracking types |
| `tool` / `tool_class` | Tool definition and classification |

## Constants

- **`VERSION: &str`** — workspace version, compiled from `CARGO_PKG_VERSION`.

## Key dependencies

| Dependency | Role |
|---|---|
| `serde` / `serde_json` | Serialization framework |
| `chrono` | Timestamps (`DateTime<Utc>`) |
| `uuid` | Identifiers |
| `thiserror` | Error enum derives |
| `schemars` | JSON Schema generation (config types) |
| `toml` | TOML deserialization (config types) |
| `dirs` | Default path resolution |
| `ed25519-dalek` / `sha2` / `hex` / `zeroize` | Manifest signing |
| `fluent` / `unic-langid` | i18n |
| `regex-lite` | Lightweight regex for validation |
| `url` | Parsed URL types |
| `tracing` | Log-level annotations |

Dev-dependencies: `rmp-serde` (MessagePack round-trip tests), `tempfile`.

---

## Adding a new type

1. **Pick the right module.** Place the type in the matching submodule. If no module fits, create one — but first verify this is truly a cross-crate type and not something that belongs in the consuming crate.
2. **Derive the standard quartet:** `Debug`, `Clone`, `Serialize`, `Deserialize`. Add `PartialEq`, `Eq`, or `Hash` only when a downstream consumer needs them.
3. **OpenAPI surface types:** also derive `utoipa::ToSchema`.
4. **Configuration types:** also derive `schemars::JsonSchema`.
5. **Map/set fields in prompt-bound types:** use `BTreeMap` / `BTreeSet`, never `HashMap` / `HashSet`. Deterministic ordering matters for LLM prompts.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MyNewType {
    pub id: uuid::Uuid,
    pub label: String,
}
```

## Adding a field to a config struct

Every config struct change has downstream effects. Follow all four steps:

1. **Add the field** with `#[serde(default)]` so existing TOML files continue to parse.
2. **Update the `Default` impl.** The build breaks if you forget.
3. **Add a doc comment.** `schemars` surfaces doc comments as the `description` field in the generated JSON Schema.
4. **Regenerate the golden fixture** in `librefang-api`'s test suite. CI will catch this if you don't.

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct KernelConfig {
    /// Maximum concurrent tool invocations per agent.
    #[serde(default = "default_max_concurrent_tools")]
    pub max_concurrent_tools: usize,
    // ...
}
```

## Schema-mirror invariant

`librefang-types` *defines* the schema; `librefang-api` *guards* it. The test `kernel_config_schema_matches_golden_fixture` compares the live `JsonSchema` output against a checked-in golden file in `librefang-api/tests`.

CI enforces this through the **changed-lanes rule**: any PR that touches `librefang-types` automatically pulls `librefang-api` into the affected test set. You cannot bypass this.

When you change a `KernelConfig` field (add, rename, retype), you **must** regenerate the golden fixture. The canonical OpenAPI and TOML baselines live under `xtask/baselines/`.

## Error types

This crate owns `LibreFangError` and related error enums. The codebase is migrating away from `Result<_, String>` and `anyhow::Error` in trait boundaries (refs #3541, #3711). New error variants go here.

When adding a variant:

- Preserve the `source()` chain so backtraces remain useful.
- Use `#[from]` on wrapped enums — this is the standard idiom.

```rust
#[derive(Debug, thiserror::Error)]
pub enum LibreFangError {
    #[error("configuration error: {0}")]
    Config(#[from] ConfigError),

    #[error("tool execution failed: {name}")]
    ToolFailed { name: String, source: ToolError },
}
```

## Hard rules

| Rule | Why |
|---|---|
| No `tokio` | Sync types only. Async runtime belongs in consumers. |
| No `reqwest` | HTTP code lives in consumer crates. Types are data-only. |
| No `librefang-*` imports | This is the bottom of the DAG. Reverse the dependency. |
| No function bodies > 5 lines | If it *does* something, it belongs elsewhere. |
| No `HashMap` in prompt-bound types | Non-deterministic iteration order breaks LLM reproducibility (#3298). Use `BTreeMap`. |
| No silently dropped serde fields | Use `#[serde(default)]` explicitly, or let it fail at compile time. |