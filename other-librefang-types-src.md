# Other — librefang-types-src

# librefang-types-src: Model Catalog Types

Shared data structures for the model and provider registry. This crate defines the canonical types that flow between the TOML catalog files on disk, the runtime catalog loader in `librefang-runtime`, the metering/cost-estimation layer in `librefang-kernel-metering`, and the provider management routes.

Nothing here performs I/O or network calls — this is purely a type and serialization module.

## Data Flow

```mermaid
graph LR
    TOML["providers/*.toml"] -->|deserialize| MCF[ModelCatalogFile]
    MCF -->|provider field| PCT[ProviderCatalogToml]
    PCT -->|.into()| PI[ProviderInfo]
    MCF -->|models vec| MCE[ModelCatalogEntry]
    MCE -->|validate()| Loader["Runtime catalog loader"]
    PI --> Runtime["librefang-runtime"]
    MCE --> Runtime
    MCE --> Metering["librefang-kernel-metering"]
    Runtime --> Routes["routes/providers.rs"]
    Routes -->|add_custom_model| MCE
```

## Core Enums

### `ModelTier`

Capability classification for a model. Determines sorting and selection logic when users request models by quality level.

| Variant | Semantic | Examples |
|---------|----------|---------|
| `Frontier` | Cutting-edge, most capable | Claude Opus, GPT-4.1 |
| `Smart` | High capability, cost-effective | Claude Sonnet, Gemini 2.5 Flash |
| `Balanced` | Speed/cost tradeoff (**default**) | GPT-4o-mini, Groq Llama |
| `Fast` | Cheapest, fastest for simple tasks | — |
| `Local` | Locally-hosted models | Ollama, vLLM, LM Studio |
| `Custom` | User-added at runtime | — |

All variants serialize to lowercase strings (`"frontier"`, `"smart"`, etc.). The enum is `#[non_exhaustive]` — new tiers may be added without a semver break.

### `AuthStatus`

Provider authentication state, detected at runtime. The default is `Missing`.

**Availability check** — `is_available()` returns `true` for states where the provider can actually serve requests:

- `ValidatedKey` — key confirmed valid via live probe
- `Configured` — key present, not yet validated
- `AutoDetected` — key found via fallback env var
- `ConfiguredCli` — CLI tool available as fallback
- `NotRequired` — local provider, no key needed

Returns `false` for `InvalidKey`, `Missing`, `CliNotInstalled`, and `LocalOffline`. Note that `InvalidKey` means a key *exists* but the provider rejected it (HTTP 401/403).

**`LocalOffline` lifecycle**: This state is set when a local provider's port is not listening. It differs from `Missing` in that `detect_auth()` will *not* reset it — only the background probe can transition it back to `NotRequired` when the service comes up.

### `Modality`

What kind of output the model produces. Directly affects which fields in `ModelCatalogEntry` are required.

| Variant | `context_window` / `max_output_tokens` required? |
|---------|--------------------------------------------------|
| `Text` (default) | **Yes** — `validate()` rejects `0` values |
| `Image` | No |
| `Audio` | No |
| `Video` | No |
| `Music` | No |

Non-text models may have `context_window = 0` because they're priced per-call or per-asset rather than per-token.

### `ModelType`

Classification used in `ModelOverrides` — `Chat` (default), `Speech`, or `Embedding`.

## Key Structs

### `ModelCatalogEntry`

A single model in the catalog. This is the central type — it appears in TOML files, is passed to the runtime loader, and feeds into cost estimation.

**Critical fields:**

- `id` — Canonical model identifier (e.g. `"claude-sonnet-4-20250514"`)
- `provider` — Provider identifier (e.g. `"anthropic"`)
- `tier` / `modality` — Classification enums
- `context_window` / `max_output_tokens` — Token limits, **must be non-zero for `Text` models**
- `input_cost_per_m` / `output_cost_per_m` — Cost per million tokens in USD
- `image_input_cost_per_m` / `image_output_cost_per_m` — `Option<f64>`, only set for image/multimodal models where pixels are priced separately
- `supports_tools` / `supports_vision` / `supports_streaming` / `supports_thinking` — Capability flags
- `aliases` — Short names that resolve to this model

**Zero-value handling rule**: Both `context_window` and `max_output_tokens` default to `0`. Consumers **must not** feed `0` into compaction thresholds or budget math — treat it as "unknown" and supply a sensible default at the call site.

#### `validate()`

```rust
pub fn validate(&self) -> Result<(), String>
```

Modality-aware schema check. For `Modality::Text` entries, rejects `context_window == 0` or `max_output_tokens == 0`. Non-text entries pass unconditionally.

**Callers**: `from_sources()` in the runtime catalog loader and `add_custom_model()` in the provider routes. Catalog loaders **must** call this and reject entries that fail — this is the guardrail that prevents `0` from leaking into downstream calculations.

#### `is_image_generation()`

Shorthand: `self.modality == Modality::Image`.

### `ProviderInfo`

Runtime provider metadata. Includes both persisted fields (from TOML) and runtime-only fields that are populated during catalog loading and authentication probing.

**Runtime-only fields** (not in TOML):

| Field | Source |
|-------|--------|
| `auth_status` | Background auth probe |
| `model_count` | Counted from loaded catalog entries |
| `available_models` | Populated by live API probe |
| `is_custom` | Set by catalog loader based on file location |
| `proxy_url` | Per-provider proxy override, persisted separately |

**Region support**: The `regions` field maps region names to `RegionConfig` structs, each with their own `base_url` and optional `api_key_env` override. When a user selects a region, consumers resolve the effective base URL by checking `regions.get(selected)` first, falling back to `provider.base_url`.

**`is_custom` flag**: Drives whether the dashboard shows a real "Delete" control. Built-in providers (from `librefang-registry`) can only be deconfigured (key removed), not deleted, because the registry sync would re-create their TOML on next boot.

### `ProviderCatalogToml`

The on-disk format — maps 1:1 to the `[provider]` section in `providers/<name>.toml`. Omits runtime-only fields (`auth_status`, `model_count`, `available_models`, `is_custom`, `proxy_url`).

Converts to `ProviderInfo` via `From<ProviderCatalogToml> for ProviderInfo`. The conversion sets runtime fields to their defaults (`auth_status: Missing`, `model_count: 0`, `is_custom: false`, etc.).

### `ModelCatalogFile`

Top-level catalog file structure containing an optional `[provider]` section and a `[[models]]` array. Used by both the main repository catalog and the community model-catalog repository.

The `provider` field is `Option<ProviderCatalogToml>` — community catalog files include it, but individual model entries can exist without it (the provider is inferred from context during merge).

### `AliasesCatalogFile`

Separate alias file mapping short names to canonical model IDs:

```toml
[aliases]
sonnet = "claude-sonnet-4-20250514"
```

### `ModelOverrides`

Per-model inference parameter overrides, persisted to `~/.librefang/model_overrides.json` keyed by `provider:model_id`. All fields are `Option` — `None` means "use the agent's or system default".

**Override resolution order** (highest to lowest priority):
1. Agent-level `ModelConfig`
2. `ModelOverrides` (this struct)
3. System defaults

Notable fields:

| Field | Purpose |
|-------|---------|
| `use_max_completion_tokens` | Some providers use `max_completion_tokens` instead of `max_tokens` |
| `no_system_role` | Model doesn't support the `system` role message |
| `force_max_tokens` | Send `max_tokens` even when the provider doesn't require it |
| `reasoning_effort` | `"low"`, `"medium"`, or `"high"` for reasoning models |

`is_empty()` returns `true` when all fields are `None`.

### `RegionConfig`

Per-region endpoint override with a `base_url` and optional `api_key_env`. When `api_key_env` is `None`, the provider-level key is used.

## Serialization Conventions

- All enums use `#[serde(rename_all = "lowercase")]` or `snake_case` — matches the TOML field conventions
- `ModelCatalogEntry` uses `#[serde(default)]` on most fields so non-text models can omit `context_window`, `max_output_tokens`, and capability flags
- Optional cost fields (`image_input_cost_per_m`, `image_output_cost_per_m`) use `skip_serializing_if = "Option::is_none"` to keep serialized output clean
- `ProviderInfo.available_models` uses `skip_serializing_if = "Vec::is_empty"`

## Downstream Consumers

| Consumer | What it uses |
|----------|-------------|
| `librefang-runtime` `model_catalog.rs` | `ModelCatalogFile`, `ModelCatalogEntry::validate()` via `from_sources()` and `merge_discovered_models()` |
| `librefang-runtime` `model_metadata.rs` | `ModelCatalogEntry` via `entry()` and `synthesize_entry()` |
| `librefang-kernel-metering` | `ModelCatalogFile` for cost estimation (e.g. `synthetic_priced_catalog()`) |
| `routes/providers.rs` | `validate()` via `add_custom_model()` when users add custom models through the dashboard |