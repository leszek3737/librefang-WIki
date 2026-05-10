# Other — librefang-types-tests

# librefang-types Tests

Integration tests that guard the contract between serialized TOML configs and their Rust type definitions. The suite catches three categories of silent regressions that are easy to introduce but hard to notice in production: dashboard↔kernel serialization drift, `Default`/`serde(default)` divergence, and schema generation failures.

## Test Files

### `agent_form_roundtrip.rs` — Dashboard Serializer ↔ Kernel Deserializer

The dashboard's TypeScript visual editor (`crates/librefang-api/dashboard/src/lib/agentManifest.ts`) serializes agent configuration to TOML. The kernel deserializes the same TOML using `librefang_types::agent::AgentManifest`. These tests mirror the exact serializer output at various levels of complexity and assert the kernel reads every field back correctly.

**When to update these tests:** Any change to the dashboard's TOML serializer or to `AgentManifest`'s serde attributes (field renames, type changes, new required fields) must be accompanied by a corresponding test update. CI failure here means the dashboard will produce TOML the kernel cannot read.

| Test | What it covers |
|------|----------------|
| `parses_form_minimum_viable_output` | Bare-minimum manifest with `name`, `version`, `module`, and `[model]` |
| `parses_form_full_output_with_capabilities_and_resources` | All standard sections: tags, skills, temperature, resource quotas, network/shell capabilities |
| `parses_form_with_advanced_sections` | Every advanced section filled: priority, session mode, thinking budget, autonomous config, routing, fallback models, context injection, schedule, exec policy |
| `parses_form_response_format_json_schema` | Inline `ResponseFormat::JsonSchema` variant — tagged enum with nested `schema` table |
| `omitting_optional_sections_uses_defaults` | Missing `resources`/`capabilities` sections fall back to struct defaults (empty vecs, `None` options) |

### `config_default_roundtrip.rs` — Default↔Serde Consistency (Issue #3404)

This is the largest test file and addresses a specific bug class: when a struct field is annotated with `#[serde(default)]` but the manual `impl Default` is not updated to match, the two sources of truth diverge silently. Empty TOML round-trips successfully, but programmatic `T::default()` produces different values.

#### Bug class mechanism

```
┌─────────────────────────┐       ┌──────────────────────────────┐
│  impl Default for T     │       │  #[serde(default)] on fields  │
│  (manual code)          │       │  (attribute-driven)           │
│                         │       │                                │
│  field_x: 0             │  vs   │  field_x: FieldX::default()   │
│  // forgot new_field!   │       │  new_field: NewField::default()│
└─────────────────────────┘       └──────────────────────────────┘
         │                                     │
         ▼                                     ▼
   T::default()                        toml::from_str("")
         │                                     │
         └──────── both succeed ───────────────┘
                   but produce different values
```

#### Test helpers

**`assert_default_roundtrip::<T>(label)`** — For most config types. Asserts two properties:

1. `T::default()` serializes to the same TOML as deserializing an empty string and re-serializing the result.
2. `T::default()` round-trips losslessly through TOML serialization.

Comparison is done by comparing TOML string representations rather than requiring `PartialEq` on every type — this avoids cascading derive requirements through the entire nested config tree.

**`assert_default_roundtrip_with::<T>(label, normalize)`** — For types with a known legitimate divergence. The `normalize` closure adjusts a single field before comparison. Every *other* field is still required to match exactly.

#### The `KernelConfig` exception

`KernelConfig` has one field where `Default` and serde-empty legitimately differ: `config_version`.

- `KernelConfig::default()` sets `config_version` to `CONFIG_VERSION` (currently `2`) — fresh in-memory configs need no migration.
- The serde helper `default_config_version()` returns `1` — a legacy TOML that omits `config_version` is by definition pre-versioning, and `run_migrations` lifts it forward.

The test normalizes `config_version` to the `Default` value before comparison, so every other field is still checked for consistency.

#### Covered types

Over 45 config structs are tested. The full list (as of this writing):

`KernelConfig`, `QueueConfig`, `QueueConcurrencyConfig`, `BudgetConfig`, `SessionConfig`, `CompactionTomlConfig`, `TaskBoardConfig`, `TriggersConfig`, `WebhookTriggerConfig`, `WebConfig`, `WebFetchConfig`, `BrowserConfig`, `BraveSearchConfig`, `TavilySearchConfig`, `PerplexitySearchConfig`, `JinaSearchConfig`, `ReloadConfig`, `RateLimitConfig`, `SkillsConfig`, `ExtensionsConfig`, `VaultConfig`, `AutoReplyConfig`, `InboxConfig`, `TelemetryConfig`, `PromptIntelligenceConfig`, `CanvasConfig`, `ThinkingConfig`, `ContextEngineTomlConfig`, `ExternalAuthConfig`, `AuditConfig`, `PrivacyConfig`, `HealthCheckConfig`, `HeartbeatTomlConfig`, `AutoDreamConfig`, `RegistryConfig`, `MemoryConfig`, `MemoryDecayConfig`, `ChunkConfig`, `NetworkConfig`, `TtsConfig`, `DockerSandboxConfig`, `PairingConfig`, `SanitizeConfig`, `ParallelToolsConfig`, `TerminalConfig`, `VoiceConfig`, `LinkedInConfig`, `AgentManifest`, `ChannelsConfig`, `BroadcastConfig`

#### Adding a new config type

For any new config struct `FooConfig` that has both `#[serde(default)]` fields and a manual `Default` impl (or reaches one transitively), add:

```rust
#[test]
fn foo_config_default_roundtrips_through_toml() {
    assert_default_roundtrip::<FooConfig>("FooConfig");
}
```

If `FooConfig` has a known legitimate divergence on a field, use `assert_default_roundtrip_with` and pass a normalizer closure that sets only that field.

#### Pinned-value regression tests

Some types have additional tests that pin a specific default value, independent of the roundtrip property. These catch the case where both `Default` and the serde helper are updated simultaneously to the same wrong value (keeping them consistent but wrong):

- **`channels_config_default_has_50mb_max`** — `ChannelsConfig::default().file_download_max_bytes` must be 50 MiB (issue #4436). A previous `#[derive(Default)]` zeroed this field via `u64::default()`, causing the channel bridge to reject all attachments when the config was built programmatically.

### `schemars_poc.rs` — Schema Generation Smoke Tests

Development/debugging tests that invoke `schemars::schema_for!` on representative types and dump the resulting JSON Schema (draft-07) to stdout. Not part of the normal CI gate — run with `--nocapture` to inspect output.

```bash
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

| Test | Type | Why it's representative |
|------|------|------------------------|
| `dump_budget_config_schema` | `BudgetConfig` | Simple struct, baseline |
| `dump_vault_config_schema` | `VaultConfig` | Contains `Option<PathBuf>` — tests filesystem path rendering |
| `full_kernel_config_schema_generates` | `KernelConfig` | End-to-end sanity: asserts >50 top-level properties and >50 nested definitions, validates well-formed JSON |
| `dump_response_format_schema` | `ResponseFormat` | Tagged enum with a variant carrying `serde_json::Value` — major risk point for schema correctness |

## Running the Tests

```bash
# Full suite
cargo test -p librefang-types

# Only dashboard roundtrip tests
cargo test -p librefang-types --test agent_form_roundtrip

# Only default-consistency tests
cargo test -p librefang-types --test config_default_roundtrip

# Schema dumps (requires --nocapture for visible output)
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

## Relationship to the Codebase

```
librefang-types (crate)
├── src/agent.rs          ← AgentManifest definition
├── src/config/types.rs   ← KernelConfig + all nested config structs
├── src/config/version.rs ← default_config_version() helper
└── tests/
    ├── agent_form_roundtrip.rs      ← validates AgentManifest TOML contract
    ├── config_default_roundtrip.rs  ← validates Default↔serde consistency
    └── schemars_poc.rs              ← schema generation smoke tests

crates/librefang-api/dashboard/
└── src/lib/agentManifest.ts         ← TypeScript serializer (mirrored by agent_form_roundtrip tests)
```

The tests in this module are the primary safety net for two cross-language contracts: the TypeScript↔Rust TOML serialization boundary, and the Rust `Default` trait↔`serde(default)` attribute boundary. Failures here indicate that either the kernel will reject valid dashboard output, or programmatic config construction will produce subtly wrong defaults.