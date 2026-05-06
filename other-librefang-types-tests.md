# Other — librefang-types-tests

# librefang-types Tests

Integration tests that guard against silent drift between three sources of truth: the dashboard's TOML serializer, the kernel's serde deserialization, and manual `Default` implementations on config types.

## Architecture

```mermaid
graph LR
    subgraph "External"
        D[Dashboard agentManifest.ts]
    end
    subgraph "librefang-types (src)"
        S[Serde Deserialize structs]
        DF[Manual Default impls]
        SD["#[serde(default = ...)"]' helpers]
    end
    subgraph "librefang-types (tests)"
        A[agent_form_roundtrip]
        C[config_default_roundtrip]
        P[schemars_poc]
    end
    D -- "mirrors serializer" --> A
    A -- "parses via" --> S
    C -- "compares" --> DF
    C -- "compares" --> SD
    P -- "generates JSON Schema" --> S
```

## Test Files

### `agent_form_roundtrip.rs`

Catches drift between the dashboard's visual editor (TypeScript serializer in `crates/librefang-api/dashboard/src/lib/agentManifest.ts`) and the kernel's TOML deserializer. The dashboard emits TOML; the kernel must read it back. Any renamed field, changed enum variant, or moved section that isn't mirrored on both sides will fail here.

**What it tests:**

| Test function | Scope |
|---|---|
| `parses_form_minimum_viable_output` | Minimal manifest: `name`, `module`, `[model]` with `provider`/`model` |
| `parses_form_full_output_with_capabilities_and_resources` | All standard sections: tags, skills, `[resources]`, `[capabilities]`, model tuning (temperature, max_tokens) |
| `parses_form_with_advanced_sections` | Every advanced section the form can emit: `priority`, `session_mode`, `web_search_augmentation`, `schedule`, `exec_policy`, `[thinking]`, `[autonomous]`, `[routing]`, `[[fallback_models]]`, `[[context_injection]]`, capability ACLs (`memory_read`, `memory_write`, `agent_message`, `ofp_connect`) |
| `parses_form_response_format_json_schema` | `ResponseFormat::JsonSchema` variant with inline schema, name, and strict flag |
| `omitting_optional_sections_uses_defaults` | Sections left out by the form produce correct defaults (empty capability lists, `None` resource quotas, `agent_spawn = false`) |

**When to update:** Any time `agentManifest.ts` changes its serializer output, or `AgentManifest` / nested structs gain/rename fields.

### `config_default_roundtrip.rs`

Regression guard for [issue #3404](#bug-class-3404). Every config struct that has both `#[serde(default)]` attributes and a manual `impl Default` is covered here.

#### Bug class #3404

When a developer adds a field with `#[serde(default)]` but forgets to add it to the manual `Default` impl (or vice versa), two different default values exist for the same field:

- **Serde path:** Empty TOML → `#[serde(default)]` fills the field with `Field::default()` or the named helper
- **In-memory path:** `T::default()` returns whatever the manual `impl Default` produces

Both paths produce valid values, so no error surfaces — but they disagree silently.

#### How the tests catch it

Each test calls one of two helper functions:

**`assert_default_roundtrip::<T>(label)`** — for types where `T::default()` and serde-empty deserialization must agree on every field:

1. Serialize `T::default()` to TOML
2. Deserialize an empty string `""` into `T`, serialize that to TOML
3. Assert the two TOML strings are identical

It also round-trips the serialized default back through deserialization to catch non-idempotent serialization.

**`assert_default_roundtrip_with::<T>(label, normalize)`** — for types with a known legitimate divergence. The `normalize` closure patches the divergent field before comparison. Every other field is still checked exactly.

#### The `KernelConfig` special case

`KernelConfig` is the only type using `assert_default_roundtrip_with`. The divergence is on `config_version`:

- `KernelConfig::default()` sets `config_version` to `CONFIG_VERSION` (currently `2`) — fresh in-memory configs need no migration
- `default_config_version()` (the serde helper) returns `1` — a legacy on-disk TOML omitting `config_version` is pre-versioning, and `run_migrations` will upgrade it

The test normalizes `config_version` to the canonical value so all other fields are still checked.

#### Covered types (50 structs)

Every config struct with both `#[serde(default)]` and a manual `Default` impl. The full list:

`KernelConfig`, `QueueConfig`, `QueueConcurrencyConfig`, `BudgetConfig`, `SessionConfig`, `CompactionTomlConfig`, `TaskBoardConfig`, `TriggersConfig`, `WebhookTriggerConfig`, `WebConfig`, `WebFetchConfig`, `BrowserConfig`, `BraveSearchConfig`, `TavilySearchConfig`, `PerplexitySearchConfig`, `JinaSearchConfig`, `ReloadConfig`, `RateLimitConfig`, `SkillsConfig`, `ExtensionsConfig`, `VaultConfig`, `AutoReplyConfig`, `InboxConfig`, `TelemetryConfig`, `PromptIntelligenceConfig`, `CanvasConfig`, `ThinkingConfig`, `ContextEngineTomlConfig`, `ExternalAuthConfig`, `AuditConfig`, `PrivacyConfig`, `HealthCheckConfig`, `HeartbeatTomlConfig`, `AutoDreamConfig`, `RegistryConfig`, `MemoryConfig`, `MemoryDecayConfig`, `ChunkConfig`, `NetworkConfig`, `TtsConfig`, `DockerSandboxConfig`, `PairingConfig`, `SanitizeConfig`, `ParallelToolsConfig`, `TerminalConfig`, `VoiceConfig`, `LinkedInConfig`, `AgentManifest`, `ChannelsConfig`, `BroadcastConfig`

#### Adding a new type

For most config structs, add a new test:

```rust
#[test]
fn my_config_default_roundtrips_through_toml() {
    assert_default_roundtrip::<MyConfig>("MyConfig");
}
```

If the type has a field where `Default` and serde legitimately diverge, use `assert_default_roundtrip_with` and supply a normalizer closure. Document why the divergence exists.

#### Pinned-value tests

Some types also have an independent pinned-value test that checks a specific default value. These protect against a future change that silently zeroes both the `Default` impl *and* the serde helper (keeping them consistent but wrong):

- **`channels_config_default_has_50mb_max`** — asserts `ChannelsConfig::default().file_download_max_bytes == 50 * 1024 * 1024` (issue #4436)

### `schemars_poc.rs`

Diagnostic tests that print schemars-generated JSON Schema (draft-07) for representative types. Not assertions-heavy — the primary value is the stdout output for manual review.

Run with output visible:

```bash
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

**What it exercises:**

| Test | Type | Why |
|---|---|---|
| `dump_budget_config_schema` | `BudgetConfig` | Representative config struct |
| `dump_vault_config_schema` | `VaultConfig` | Contains `Option<PathBuf>` — tests filesystem path rendering |
| `full_kernel_config_schema_generates` | `KernelConfig` | End-to-end sanity: asserts >50 top-level properties and >50 nested definitions |
| `dump_response_format_schema` | `ResponseFormat` | Tagged enum carrying `serde_json::Value` — major risk point for schema correctness |

## Running

```bash
# All tests in this module
cargo test -p librefang-types

# Only agent form round-trip tests
cargo test -p librefang-types --test agent_form_roundtrip

# Only config default round-trip tests
cargo test -p librefang-types --test config_default_roundtrip

# Schema PoC (must see output)
cargo test -p librefang-types --test schemars_poc -- --nocapture
```

## Relationships to other crates

- **`librefang-types` (src):** Defines all config structs, `AgentManifest`, `ResponseFormat`, and their serde/Default impls. Tests live in this crate's `tests/` directory.
- **`librefang-api/dashboard`:** Contains `agentManifest.ts` whose serializer output the agent-form round-trip tests mirror. Changes there may require updating the TOML literals in `agent_form_roundtrip.rs`.
- **Migration system (`config/version.rs`):** `default_config_version()` returns `1` as the migration tripwire; `CONFIG_VERSION` is the current version. The `KernelConfig` test normalizes across this boundary.