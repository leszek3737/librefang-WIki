# Other — librefang-types-tests

# librefang-types-tests

Integration tests for the `librefang-types` crate. Two concerns live here: **round-trip contract tests** that lock the dashboard's TOML serializer to the kernel's deserializer, and **schema-dump diagnostics** for eyeballing `schemars`-generated JSON Schema output.

## Why these tests exist

The dashboard (TypeScript) produces TOML via a visual agent-editor form. The kernel (Rust) reads that TOML into `AgentManifest`. If either side renames a field, changes an enum variant, or alters defaulting behavior, agents silently break at runtime. These tests catch that drift at build time by parsing the *exact* TOML the dashboard's serializer would emit and asserting the resulting Rust structs.

The schema-dump tests serve a different purpose: they're a low-friction way to inspect what `schemars` produces for key config types (`BudgetConfig`, `VaultConfig`, `KernelConfig`, `ResponseFormat`) without wiring up a full HTTP endpoint.

---

## Test Files

### `agent_form_roundtrip.rs`

Mirror of the serializer rules in `crates/librefang-api/dashboard/src/lib/agentManifest.ts`. Each test constructs a TOML string matching what the dashboard form emits, deserializes it into `AgentManifest`, and asserts field values.

| Test | What it covers |
|---|---|
| `parses_form_minimum_viable_output` | Bare-minimum manifest: name, version, module, and a model block with provider + model name. |
| `parses_form_full_output_with_capabilities_and_resources` | All basic sections populated: description, tags, skills, model options (temperature, max_tokens, system_prompt), resource quotas, and capabilities (network, shell, agent_spawn). |
| `parses_form_with_advanced_sections` | Every advanced section filled: priority, session_mode, web_search_augmentation, schedule, exec_policy, thinking budget, autonomous config, routing tiers, fallback_models (array-of-tables), context_injection, and extended capability globs (memory_read, memory_write, agent_message, ofp_connect). |
| `parses_form_response_format_json_schema` | Inline `ResponseFormat::JsonSchema` variant with name, schema, and strict flag. Validates that the tagged-enum TOML representation round-trips correctly. |
| `omitting_optional_sections_uses_defaults` | Manifest with no resources or capabilities sections. Asserts that `ResourceQuota` and `ManifestCapabilities` fields fall back to their defaults (empty vecs, `None` for optional limits, `false` for booleans). |

**Key types validated across these tests:**

- `AgentManifest` — top-level config struct
- `Priority` — enum (e.g. `High`)
- `SessionMode` — enum (e.g. `New`)
- `ResponseFormat::JsonSchema` — tagged enum variant carrying `serde_json::Value`
- Optional advanced sections: `thinking`, `autonomous`, `routing` — each deserializes as `Option<T>`, so their absence must produce `None`

### `schemars_poc.rs`

Run with `--nocapture` to see output:

```bash
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

| Test | What it dumps |
|---|---|
| `dump_budget_config_schema` | `BudgetConfig` — straightforward struct, baseline for size comparison. |
| `dump_vault_config_schema` | `VaultConfig` — contains `Option<PathBuf>`, useful for checking how filesystem paths render in JSON Schema. |
| `full_kernel_config_schema_generates` | `KernelConfig` — end-to-end sanity check. Asserts >50 top-level properties and >50 nested definitions, confirming the full config surface is representable. |
| `dump_response_format_schema` | `ResponseFormat` — tagged enum with a variant carrying `serde_json::Value`. This is a high-risk point for schema fidelity. |

These are not strict pass/fail contract tests (except `full_kernel_config_schema_generates`, which asserts minimum field counts). They exist for developer inspection during schema evolution.

---

## Relationship to the codebase

```
Dashboard (TS)                 Kernel (Rust)
┌──────────────────┐           ┌─────────────────────┐
│ agentManifest.ts  │──TOML──▶│ AgentManifest        │
│ (serializer)      │          │ (serde deserializer) │
└──────────────────┘           └─────────────────────┘
        │                              │
        └──── contract tested by ──────┘
             agent_form_roundtrip.rs
```

- **Upstream dependency**: `librefang-types` provides the `AgentManifest`, `Priority`, `SessionMode`, `ResponseFormat`, and all supporting structs.
- **Cross-repo contract**: The TOML strings in these tests must be kept in sync with `agentManifest.ts`. If the dashboard serializer changes, the corresponding test here should be updated (or should fail, signaling the break).
- **No outbound calls**: These tests are pure deserialization + assertion. No network, no filesystem, no database.

## Adding a new round-trip test

When the dashboard form gains a new field or section:

1. Add a test case (or extend an existing one) with the exact TOML the dashboard will emit.
2. Assert the deserialized field values and any defaulting behavior.
3. If the field uses a new enum variant or struct, verify it appears correctly in the parsed output.

For schema work, add a `dump_*` test to `schemars_poc.rs` and inspect the output. If the type is already covered by `KernelConfig`, the `full_kernel_config_schema_generates` test will pick up new definitions automatically.