# Other — librefang-types-tests

# librefang-types Tests

Integration tests that guard serialization correctness between the TypeScript dashboard, on-disk TOML config files, and the Rust kernel's type system.

## Purpose

This module exists to catch a specific class of silent bugs: **drift between how the dashboard serializes data, how the kernel deserializes it, and how Rust `Default` implementations compare to serde's empty-document behavior**. Without these tests, a field added with `#[serde(default)]` but forgotten in a manual `Default` impl would cause programmatic config construction to differ from file-loaded configs — with no compile-time or runtime error.

## Test Files

### `agent_form_roundtrip.rs`

Validates that the TOML emitted by the dashboard's visual editor (`crates/librefang-api/dashboard/src/lib/agentManifest.ts`) parses correctly into `AgentManifest`.

**What it catches:** Renamed fields, changed enum variants, or missing struct fields in `AgentManifest` that would cause the dashboard's output to be rejected by the kernel.

| Test | Scope |
|------|-------|
| `parses_form_minimum_viable_output` | Required fields only: `name`, `version`, `module`, `[model]` |
| `parses_form_full_output_with_capabilities_and_resources` | All standard sections: tags, skills, temperature, resources, capabilities |
| `parses_form_with_advanced_sections` | Every advanced section filled: priority, session mode, thinking, autonomous, routing, fallback models, context injection |
| `parses_form_response_format_json_schema` | `ResponseFormat::JsonSchema` variant with inline schema table |
| `omitting_optional_sections_uses_defaults` | Sections omitted entirely — verifies `ResourceQuota` and `ManifestCapabilities` defaults |

### `config_default_roundtrip.rs`

Regression suite for [issue #3404]. Validates two properties for every config struct that has both `#[serde(default)]` and a manual `impl Default`:

1. **Empty-TOML equality:** `T::default()` must produce the same TOML as deserializing an empty string and re-serializing.
2. **Round-trip idempotency:** `T::default()` → serialize → deserialize → serialize must be lossless.

**Why compare serialized TOML strings instead of deriving `PartialEq`:** Deriving `PartialEq` would cascade through the entire nested config tree and add a maintenance burden. TOML string comparison is sufficient and requires no trait changes on production types.

#### Core helpers

```
assert_default_roundtrip::<T>(label)
assert_default_roundtrip_with::<T>(label, normalize)
```

- `assert_default_roundtrip` — the common case: every field must agree between `T::default()` and the serde-empty result.
- `assert_default_roundtrip_with` — for types with a known legitimate divergence. The `normalize` closure copies the canonical value before comparison. Only `KernelConfig` uses this variant (for `config_version`, explained below).

#### The `config_version` normalization

`KernelConfig` has one field where `Default` and serde legitimately disagree:

| Source | `config_version` value | Reason |
|--------|----------------------|--------|
| `KernelConfig::default()` | `CONFIG_VERSION` (currently `2`) | Fresh in-memory configs need no migration |
| serde empty-document | `1` (via `default_config_version()`) | Legacy on-disk TOML files without a version field are pre-versioning; `run_migrations` lifts them forward |

The test normalizes this single field and then asserts all other fields match exactly — so a new `#[serde(default)]` field missing from the `Default` impl will still be caught.

#### Covered types

Over 40 config structs are tested, including: `QueueConfig`, `BudgetConfig`, `SessionConfig`, `MemoryConfig`, `NetworkConfig`, `VaultConfig`, `ChannelsConfig`, `BroadcastConfig`, `AgentManifest`, `TtsConfig`, `DockerSandboxConfig`, `PairingConfig`, `SanitizeConfig`, `TerminalConfig`, and all search provider configs.

#### Pinned-value regression: `channels_config_default_has_50mb_max`

A standalone assertion ([issue #4436]) that `ChannelsConfig::default().file_download_max_bytes` equals 50 MiB. This guards against the scenario where both `Default` and the serde helper are silently zeroed in tandem — which would pass the round-trip test but break the channel bridge at runtime.

### `schemars_poc.rs`

Diagnostic tests that dump `schemars`-generated JSON Schema (draft-07) to stdout. These are not assertions on schema correctness — they exist for developer inspection of edge cases.

Run with stdout visible:

```bash
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

| Test | What it exercises |
|------|-------------------|
| `dump_budget_config_schema` | Simple flat struct |
| `dump_vault_config_schema` | `Option<PathBuf>` — how schemars renders filesystem paths |
| `full_kernel_config_schema_generates` | End-to-end sanity: asserts >50 top-level properties and >50 definitions; prints size |
| `dump_response_format_schema` | Tagged enum carrying `serde_json::Value` — a major risk point for schema generation |

## When to extend these tests

**Add an `agent_form_roundtrip` test** when:
- You rename, add, or remove a field on `AgentManifest` or its nested structs.
- You change how the dashboard serializer formats TOML (e.g., array syntax, inline tables).
- You add a new enum variant that the form can produce.

**Add a `config_default_roundtrip` test** when:
- You add a new config struct with `#[serde(default)]` and a manual `Default` impl.
- You add a `#[serde(default)]` field to an existing struct that already has a manual `Default`.

For most cases, a single call is sufficient:

```rust
#[test]
fn my_config_default_roundtrips_through_toml() {
    assert_default_roundtrip::<MyConfig>("MyConfig");
}
```

If the type has a field where `Default` and serde intentionally diverge (like `config_version`), use `assert_default_roundtrip_with` and normalize only that field.