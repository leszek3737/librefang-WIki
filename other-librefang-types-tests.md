# Other — librefang-types-tests

# librefang-types Tests

Integration tests that guard the TOML serialization boundary between the dashboard frontend and the kernel's config/agent type system.

## Purpose

The `librefang-types` crate defines every configuration and agent manifest struct that flows between three distinct producers/consumers:

- **On-disk TOML files** — the kernel reads `AgentManifest` and `KernelConfig` from disk at startup.
- **Dashboard visual editor** — a TypeScript serializer in `agentManifest.ts` emits TOML that the kernel must accept.
- **In-process `Default` construction** — `KernelConfig::default()` and friends build values programmatically.

This test suite catches a specific class of silent regression: a field annotated with `#[serde(default)]` whose value diverges from what the manual `impl Default` produces. It also validates that the dashboard's TOML output shape stays parseable by the kernel deserializer.

---

## Test Files

### `agent_form_roundtrip.rs`

Validates that the exact TOML the dashboard's visual editor emits can be deserialized into `AgentManifest`. Each test mirrors a different form-output profile:

| Test | What it covers |
|------|---------------|
| `parses_form_minimum_viable_output` | Bare-minimum fields: `name`, `version`, `module`, and a `[model]` section |
| `parses_form_full_output_with_capabilities_and_resources` | All primary sections filled: tags, skills, model tuning, `[resources]`, `[capabilities]` |
| `parses_form_with_advanced_sections` | Every advanced section: `priority`, `session_mode`, `thinking`, `autonomous`, `routing`, `fallback_models`, `context_injection` |
| `parses_form_response_format_json_schema` | The `response_format` field with inline JSON-schema table |
| `omitting_optional_sections_uses_defaults` | Sections omitted entirely — verifies that `ResourceQuota` and `ManifestCapabilities` fall back to sensible defaults |

The contract being tested is: **if the dashboard form emits TOML that passes client-side validation, the kernel's `toml::from_str` must succeed and produce the expected values.**

### `config_default_roundtrip.rs`

Regression suite for [issue #3404](#bug-class-3404). For every config struct `T` that has both `#[serde(default)]` annotations and a manual `Default` impl, these tests assert two properties:

1. **Empty-TOML equivalence** — `toml::from_str::<T>("")` produces identical TOML when re-serialized as `T::default()`.
2. **Round-trip idempotency** — `toml::from_str::<T>(&toml::to_string(&T::default()).unwrap())` produces the same TOML again.

Comparison is done by serializing both sides to TOML strings and comparing those strings — this avoids requiring `PartialEq` on the entire nested config tree.

#### Helper functions

- **`assert_default_roundtrip::<T>(label)`** — For types where `Default::default()` and serde-empty should agree on every field. This is the common case; most config structs use this.

- **`assert_default_roundtrip_with::<T>(label, normalize)`** — For types with a known legitimate divergence. The `normalize` closure adjusts the serde-derived value before comparison. Only `KernelConfig` currently needs this, due to its `config_version` migration sentinel (see below).

#### KernelConfig and config_version normalization

`KernelConfig` is the one struct where the two default sources intentionally differ on a single field:

| Source | `config_version` value | Reason |
|--------|----------------------|--------|
| `KernelConfig::default()` | `CONFIG_VERSION` (currently `2`) | Fresh in-memory configs need no migration |
| `default_config_version()` (serde) | `1` | Legacy TOML files that omit this field are pre-versioning; `run_migrations` lifts them forward |

The test normalizes both sides to the canonical version before comparing, so every *other* field is still checked exactly.

#### Covered types

Over 45 config structs are tested, including but not limited to: `QueueConfig`, `BudgetConfig`, `SessionConfig`, `WebConfig`, `MemoryConfig`, `TelemetryConfig`, `ThinkingConfig`, `DockerSandboxConfig`, `ChannelsConfig`, `BroadcastConfig`, `AgentManifest`, and all nested sub-configs.

#### The #4436 regression

`ChannelsConfig` had a prior bug where `#[derive(Default)]` set `file_download_max_bytes` to `0` while `#[serde(default = "default_file_download_max_bytes")]` returned 50 MiB. The test `channels_config_default_has_50mb_max` is a pinned-value assertion that catches this even if both sources are changed to the same wrong value simultaneously.

### `schemars_poc.rs`

Diagnostic tests that print `schemars`-generated JSON Schema (draft-07) for representative types. Not a correctness gate — these are for developer inspection.

Run with stdout visible:

```bash
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

| Test | Type | Why it's interesting |
|------|------|---------------------|
| `dump_budget_config_schema` | `BudgetConfig` | Representative flat config |
| `dump_vault_config_schema` | `VaultConfig` | Contains `Option<PathBuf>` — tests filesystem path rendering |
| `full_kernel_config_schema_generates` | `KernelConfig` | End-to-end sanity: asserts >50 top-level properties and >50 definitions, confirms the full schema is well-formed JSON |
| `dump_response_format_schema` | `ResponseFormat` | Tagged enum with `serde_json::Value` — major schema edge case |

---

## Bug Class #3404

The core insight: when you add a field to a config struct with `#[serde(default)]` but forget to add it to the hand-written `Default` impl, the two construction paths silently diverge. The schemars golden-schema test doesn't catch this because schemars reads the serde attribute, not the `Default` impl body.

This manifests as: "config works when loaded from TOML file, but breaks when constructed programmatically via `Default`" — exactly the kind of bug that's invisible until a specific runtime path is hit.

The round-trip test pattern catches this by forcing both paths to produce TOML and comparing them character-by-character.

---

## Adding Coverage for a New Config Type

When a new config struct is added to `librefang-types`:

1. If it has `#[serde(default)]` on any field and a manual `Default` impl (or transitively contains such a struct), add a test to `config_default_roundtrip.rs`:

   ```rust
   #[test]
   fn my_new_config_default_roundtrips_through_toml() {
       assert_default_roundtrip::<MyNewConfig>("MyNewConfig");
   }
   ```

2. If `Default::default()` and the serde empty-fill legitimately differ on a specific field, use `assert_default_roundtrip_with` and pass a normalizer:

   ```rust
   #[test]
   fn my_config_default_roundtrips_through_toml() {
       let canonical = MyConfig::default().special_field;
       assert_default_roundtrip_with::<MyConfig>("MyConfig", move |c| {
           c.special_field = canonical;
       });
   }
   ```

3. If the type is part of `AgentManifest` and the dashboard emits it, add a round-trip case to `agent_form_roundtrip.rs` covering the minimum and full output profiles.

---

## Connections to the Codebase

```mermaid
graph LR
    subgraph Dashboard
        A[agentManifest.ts<br/>TOML serializer]
    end
    subgraph librefang-types
        B[AgentManifest struct]
        C[KernelConfig / config structs]
    end
    subgraph Kernel
        D[Config loader<br/>toml::from_str]
    end
    subgraph Tests (this crate)
        E[agent_form_roundtrip.rs]
        F[config_default_roundtrip.rs]
        G[schemars_poc.rs]
    end
    A -- "TOML output" --> E
    E -- "parses via" --> B
    D -- "reads from disk" --> C
    F -- "Default vs serde-empty" --> C
    F -- "Default vs serde-empty" --> B
    G -- "schema_for!" --> C
```

- **`crates/librefang-api/dashboard/src/lib/agentManifest.ts`** — The TypeScript serializer whose output is mirrored by `agent_form_roundtrip.rs`. Drift between this file and the Rust structs is exactly what the tests catch.
- **`crates/librefang-types/src/config/types.rs`** — Contains the manual `Default` impls validated by `config_default_roundtrip.rs`.
- **`crates/librefang-types/src/config/version.rs`** — Contains `default_config_version()` returning `1`, the sentinel value normalized away in the `KernelConfig` test.