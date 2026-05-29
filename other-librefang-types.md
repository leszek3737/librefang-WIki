# Other — librefang-types

# librefang-types

Core type definitions for the LibreFang Agent OS. Pure data structures — no business logic. Every workspace crate depends on this one; it depends on none of them.

## Position in the workspace

```
┌─────────────────────────────────────────────┐
│  librefang-api, librefang-kernel, runtime,  │
│  memory substrate, wire protocol, etc.      │
│                                             │
│          ┌───────────────┐                  │
│          │  librefang-   │                  │
│          │   types       │ ← bottom of DAG │
│          └───────────────┘                  │
└─────────────────────────────────────────────┘
```

No `librefang-*` crate may appear in this crate's `Cargo.toml`. If a type is needed by two or more crates, it lives here. If only one crate uses it, it belongs in that crate instead.

## Public modules

| Module | Domain |
|---|---|
| `agent` | Agent identity and descriptor types |
| `approval` | Human-in-the-loop approval workflows |
| `capability` | Permission and capability tokens |
| `comms` | Inter-agent communication envelopes |
| `config` | Kernel and runtime configuration (`KernelConfig`, TOML-facing structs) |
| `error` | `LibreFangError` and domain-specific error enums |
| `event` | Event types emitted by the kernel and runtime |
| `goal` | Goal and objective structures |
| `i18n` | Internationalization types (backed by `fluent` / `unic-langid`) |
| `manifest_signing` | Ed25519 manifest signature and verification types |
| `media` | Media attachment and content types |
| `memory` | Memory substrate records and queries |
| `message` | Chat/message envelope types |
| `model_catalog` | Model registry and catalog entries |
| `oauth` | OAuth2 token and flow types |
| `registry_schema` | Agent registry schema definitions |
| `scheduler` | Task scheduling and cron types |
| `serde_compat` | Serde helpers for cross-format compatibility |
| `subagent` | Sub-agent spawning and lifecycle types |
| `taint` | Taint tracking for untrusted data |
| `tool` | Tool descriptor and invocation types |
| `tool_class` | Tool classification taxonomy |

## Constants

- **`VERSION: &str`** — workspace version injected at compile time from `CARGO_PKG_VERSION`.

## Adding a new type

1. **Pick the right module.** If no existing module fits, create a new one — but first confirm the type is genuinely cross-crate. Single-consumer types don't belong here.
2. **Derive the standard quartet:** `Debug`, `Clone`, `Serialize`, `Deserialize`. Add `PartialEq`, `Eq`, `Hash` only when a downstream consumer requires them.
3. **OpenAPI surface types:** also derive `utoipa::ToSchema`.
4. **Configuration types:** also derive `schemars::JsonSchema` (this drives the kernel-config golden fixture).
5. **Prompt-bound types:** use `BTreeMap` / `BTreeSet` instead of `HashMap` / `HashSet`. Deterministic serialization matters when the data reaches an LLM prompt.

## Adding a configuration field

Every field added to a config struct (anything deriving `JsonSchema` and consumed by `KernelConfig`) follows this sequence:

1. Add the field annotated with `#[serde(default)]` — this preserves forward-compatibility with existing TOML files.
2. Add the corresponding entry in the struct's `Default` impl. The build breaks without it.
3. Write a doc comment. `schemars` lifts doc comments into the `description` field of the generated JSON Schema.
4. Regenerate the golden fixture in `librefang-api` (test: `kernel_config_schema_matches_golden_fixture`). CI will fail until this is done.

### Why CI catches schema drift

The changed-lanes rule in CI detects that a `librefang-types` change can affect `librefang-api`. A types-only PR automatically pulls the API test suite into the affected set. The golden-file test then compares the generated schema against the committed fixture. Any mismatch fails the build.

## Adding an error variant

The workspace is migrating away from `Result<_, String>` and `anyhow::Error` in trait boundaries. New error variants go in this crate's `error` module.

When adding a variant:

- Preserve the `source()` chain so that downstream callers can inspect the root cause. Use `#[from]` on a wrapped error type — this is the standard idiom and ensures `std::error::Error::source()` is implemented correctly.
- Example pattern:

```rust
#[derive(Debug, thiserror::Error)]
pub enum LibreFangError {
    #[error("configuration parse failed")]
    ConfigParse(#[from] ConfigError),

    #[error("unknown model: {0}")]
    UnknownModel(String),
}
```

## Key external dependencies

| Crate | Purpose |
|---|---|
| `serde` / `serde_json` | Serialization framework |
| `chrono` | Timestamps with timezone support |
| `uuid` | Unique identifiers |
| `thiserror` | Error enum derives |
| `dirs` | Platform-specific directory paths (used in config defaults) |
| `toml` | TOML parsing for config types |
| `schemars` | JSON Schema generation from Rust types |
| `ed25519-dalek` | Manifest signing types |
| `sha2` / `hex` | Hashing utilities |
| `zeroize` | Secure memory clearing for sensitive types |
| `fluent` / `unic-langid` | i18n message types |
| `url` | Parsed URL types |

## What does NOT belong here

- **`tokio`** — only synchronous types. This crate must compile in `no_std`-adjacent contexts without pulling in a runtime.
- **`reqwest`** — wire types carry HTTP-shaped data, but no HTTP client code.
- **Any `librefang-*` crate** — this crate is the bottom of the dependency graph. Reverse the dependency.
- **Business logic** — if a function body exceeds ~5 lines, it belongs in a consumer crate.
- **`HashMap` / `HashSet` in prompt-bound types** — use `BTreeMap` / `BTreeSet` for deterministic serialization.
- **Silently dropped serde fields** — every field either has `#[serde(default)]` or must be present in the input. No `#[serde(skip)]` on data that matters.

## `serde_compat` module

Handles serialization edge cases across formats (JSON, TOML, MessagePack). Provides custom serde helpers for types like `chrono::DateTime`, `uuid::Uuid`, and enum tag formats that differ between wire protocols. Consumer crates import these helpers rather than reimplementing format-specific logic.