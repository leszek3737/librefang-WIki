# Other — librefang-types-src

# Model Catalog Types (`librefang-types::model_catalog`)

Shared data structures for the model registry — the single source of truth for how providers, models, authentication states, and inference overrides are represented across the codebase.

## Purpose

Every subsystem that needs to know "what models exist, who provides them, and are they available?" consumes types from this module. The catalog is loaded from TOML files shipped in the `librefang-registry` and merged with user-defined providers at runtime. The types here are intentionally free of I/O or business logic — they are pure data with lightweight helper methods.

## Architecture

```mermaid
graph TD
    TOML["providers/*.toml<br/>(catalog files)"] -->|deserialize| MCF[ModelCatalogFile]
    MCF -->|provider field| PCT[ProviderCatalogToml]
    MCF -->|models array| MCE[ModelCatalogEntry]
    PCT -->|.into()| PI[ProviderInfo]
    ALIASES["aliases.toml"] -->|deserialize| ACF[AliasesCatalogFile]
    PI -->|consumed by| RUNTIME["librefang-runtime<br/>model_catalog"]
    MCE -->|consumed by| RUNTIME
    MCE -->|consumed by| METERING["librefang-kernel-metering"]
```

## Key Types

### `ModelTier`

Capability tier classification for models. Serde uses `lowercase` renaming (`"frontier"`, `"smart"`, etc.).

| Variant | Semantics | Default |
|---------|-----------|---------|
| `Frontier` | Most capable, highest cost (e.g. Claude Opus, GPT-4.1) | |
| `Smart` | Cost-effective intelligence (e.g. Claude Sonnet, Gemini 2.5 Flash) | |
| `Balanced` | Speed/cost trade-off (e.g. GPT-4o-mini, Groq Llama) | **yes** |
| `Fast` | Cheapest, fastest for simple tasks | |
| `Local` | Self-hosted (Ollama, vLLM, LM Studio) | |
| `Custom` | User-defined at runtime | |

### `AuthStatus`

Provider authentication state, detected at runtime by the catalog loader's `detect_auth()` probe.

The `is_available()` method returns `true` for states where the provider is usable — `ValidatedKey`, `Configured`, `AutoDetected`, `ConfiguredCli`, and `NotRequired`. Notably, `InvalidKey` returns `false` because the key exists but was rejected (HTTP 401/403).

Key states that require careful handling:

- **`LocalOffline`** — Set when a local provider's port is not listening. Unlike `Missing`, `detect_auth()` will not reset this; only a successful probe transitions back to `NotRequired`.
- **`AutoDetected`** — Key found via a fallback environment variable. Usable but may not match the actual provider; the UI should prompt verification.

### `ModelCatalogEntry`

A single model's metadata. All boolean capability flags default to `false` via `#[serde(default)]`, so community catalog files only need to set capabilities they support.

Fields of note:
- **`provider`** — When empty in a catalog TOML, the runtime loader infers it from the enclosing `[provider].id` section during merge.
- **`aliases`** — Short names that resolve to this model (e.g. `["sonnet", "claude-sonnet"]`).
- **`input_cost_per_m` / `output_cost_per_m`** — Cost in USD per million tokens, consumed by the metering subsystem.

### `ModelOverrides`

Per-model inference parameter overrides persisted to `~/.librefang/model_overrides.json`, keyed by `provider:model_id`. Every field is `Option` — `None` means "use the agent's or system default." The override resolution order is:

1. Agent-level `ModelConfig` (highest priority)
2. `ModelOverrides` from this file
3. System defaults

Use `is_empty()` to check whether any overrides are set. The struct uses `#[serde(skip_serializing_if = "Option::is_none")]` to keep the persisted file clean.

### `ProviderInfo` vs `ProviderCatalogToml`

Two structs represent the same provider at different stages:

| | `ProviderCatalogToml` | `ProviderInfo` |
|---|---|---|
| **Source** | Parsed directly from `providers/*.toml` | Runtime state after catalog loading |
| **Runtime fields** | None | `auth_status`, `model_count`, `available_models`, `is_custom`, `proxy_url` |
| **Conversion** | `ProviderCatalogToml` → `ProviderInfo` via `From` impl | — |

The `From<ProviderCatalogToml>` conversion initializes runtime fields to safe defaults: `AuthStatus::Missing`, `model_count: 0`, empty `available_models`, `is_custom: false`.

The `is_custom` flag on `ProviderInfo` controls dashboard behavior — built-in providers (from the registry) can only be deconfigured, not deleted, because registry sync would recreate their TOML on next boot.

### `RegionConfig` and Regional Endpoints

Providers can define regional endpoint overrides in their TOML:

```toml
[provider.regions.us]
base_url = "https://dashscope-us.aliyuncs.com/compatible-mode/v1"
api_key_env = "DASHSCOPE_US_API_KEY"  # optional override
```

Each `RegionConfig` has a `base_url` (required) and an optional `api_key_env` that overrides the provider-level key for that region. At runtime, region selection replaces the provider's `base_url` — if no region is selected, the provider-level URL is used.

### File-Level Structs

**`ModelCatalogFile`** — The top-level deserialization target for a single provider TOML. Contains an optional `[provider]` section and a `[[models]]` array. Community catalog files include the provider section; registry files may omit it.

**`AliasesCatalogFile`** — A separate `aliases.toml` mapping short names to canonical model IDs. Used for global aliases that cross provider boundaries.

## TOML Format Reference

A complete provider catalog file:

```toml
[provider]
id = "anthropic"
display_name = "Anthropic"
api_key_env = "ANTHROPIC_API_KEY"
base_url = "https://api.anthropic.com"
key_required = true
signup_url = "https://console.anthropic.com/settings/keys"

[provider.regions.eu]
base_url = "https://api-eu.anthropic.com"

[[models]]
id = "claude-sonnet-4-20250514"
display_name = "Claude Sonnet 4"
provider = "anthropic"
tier = "smart"
context_window = 200000
max_output_tokens = 64000
input_cost_per_m = 3.0
output_cost_per_m = 15.0
supports_tools = true
supports_vision = true
supports_streaming = true
supports_thinking = true
aliases = ["sonnet", "claude-sonnet"]
```

An aliases-only file:

```toml
[aliases]
sonnet = "claude-sonnet-4-20250514"
haiku = "claude-haiku-4-5-20251001"
```

## Consumers

- **`librefang-runtime::model_catalog`** — Calls `merge_discovered_models` which produces `ModelCatalogEntry` instances by merging catalog data with live API probe results.
- **`librefang-kernel-metering`** — Reads `ModelCatalogFile` to look up per-model pricing (`input_cost_per_m`, `output_cost_per_m`) for cost estimation, with a legacy budget-rate fallback when pricing is zero.

## Serde Conventions

- `ModelTier` and `AuthStatus` use `#[serde(rename_all = "lowercase")]` and `#[serde(rename_all = "snake_case")]` respectively.
- Boolean fields on `ModelCatalogEntry` use `#[serde(default)]` so they default to `false` when absent from TOML.
- `ModelOverrides` skips serializing `None` fields to keep the JSON file minimal.
- `ProviderInfo` skips serializing empty `available_models` vectors.