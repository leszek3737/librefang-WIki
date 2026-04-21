# Shared Types & Configuration — librefang-types-src

# librefang-types — Shared Types & Configuration

## Purpose

`librefang-types` is the foundational crate for the LibreFang Agent Operating System. It defines every shared data structure used across the kernel, runtime, memory substrate, channel adapters, wire protocol, and dashboard — but contains **no business logic**. Every other crate in the workspace depends on this one, making it the single source of truth for configuration shape, error taxonomy, and cross-cutting type definitions.

```mermaid
graph TD
    subgraph "librefang-types"
        CFG["config/"]
        ERR["error"]
        AGENT["agent"]
        EVENT["event"]
        MSG["message"]
        MEM["memory"]
        TOOL["tool / tool_policy"]
        SCHED["scheduler"]
        SIGN["manifest_signing"]
        I18N["i18n"]
        OTHER["capability, comms, goal, media, …"]
    end

    CLI["librefang-cli"] --> CFG
    CLI --> ERR
    RT["librefang-runtime"] --> CFG
    RT --> ERR
    RT --> TOOL
    CH["librefang-channels"] --> MSG
    CH --> CFG
    API["librefang-api"] --> ERR
    API --> CFG

    style CFG fill:#f9f,stroke:#333
    style ERR fill:#f66,stroke:#333
```

## Module Layout

| Module | Responsibility |
|--------|---------------|
| `config/` | All kernel configuration types, serde helpers, validation, version tracking |
| `error` | `LibreFangError` enum and `LibreFangResult<T>` alias |
| `agent` | Agent manifest and descriptor types |
| `approval` | Execution approval policy and notification configuration |
| `capability` | Agent capability definitions |
| `comms` | Inter-component communication types |
| `event` | System event types for the event bus |
| `goal` | Agent goal tracking structures |
| `i18n` | Internationalization (Fluent-based) helpers |
| `manifest_signing` | Ed25519 manifest signing and verification |
| `media` | Media understanding and link understanding configuration |
| `memory` | Memory substrate and proactive memory (mem0-style) configuration |
| `message` | Wire-protocol message types |
| `model_catalog` | LLM model catalog entries |
| `registry_schema` | Plugin/skill registry schema types |
| `scheduler` | Cron and scheduled-job types with validation |
| `serde_compat` | Lenient deserialization helpers (`vec_lenient`) |
| `subagent` | Sub-agent spawning and orchestration types |
| `taint` | Security taint-tracking types |
| `tool` / `tool_compat` | Tool descriptor types |
| `tool_policy` | Global deny/allow tool policy rules |
| `webhook` | Webhook trigger payload types and validation |
| `workflow_template` | Workflow template definitions |

---

## Error Handling — `error`

The central error type is [`LibreFangError`](src/error.rs), a `thiserror`-derived enum covering every failure mode in the system:

```rust
pub type LibreFangResult<T> = Result<T, LibreFangError>;
```

Key variants and when they arise:

| Variant | Typical source |
|---------|---------------|
| `AgentNotFound` / `AgentAlreadyExists` | Agent CRUD operations |
| `CapabilityDenied` | Permission checks before tool execution |
| `InvalidState { current, operation }` | State-machine transitions (e.g. stopping an already-stopped agent) |
| `ToolExecution { tool_id, reason }` | Tool runner failures |
| `MaxIterationsExceeded` | Agent loop guard |
| `RepeatedToolFailures { iterations, error_count }` | Consecutive tool failure circuit-breaker |
| `ShuttingDown` | Kernel shutdown barrier |
| `Sandbox` | WASM / Docker sandbox violations |
| `MeteringError` | Cost tracking failures |

Most variants carry descriptive strings; `Io` wraps `std::io::Error` via `#[from]`.

---

## Configuration — `config/`

The configuration system is the largest part of this crate. It is organized into four submodules, all re-exported from `config/mod.rs`:

### Submodule Structure

```
config/
├── mod.rs            # Re-exports, constants (DEFAULT_API_PORT, DEFAULT_API_LISTEN)
├── types.rs          # All struct/enum definitions + Default impls
├── serde_helpers.rs  # OneOrMany<T>, deserialize_string_or_int_vec
├── validation.rs     # KernelConfig::validate(), clamp_bounds(), unknown-field detection
└── version.rs        # Config schema version tracking
```

### `OneOrMany<T>` — Multi-Instance Channel Config

The [`OneOrMany<T>`](src/config/serde_helpers.rs) wrapper is the key to supporting multiple bot accounts per channel while maintaining backward compatibility with single-instance configs.

In TOML, a single bot is a bare table:
```toml
[channels.telegram]
bot_token_env = "TG_TOKEN_1"
```

Multiple bots use array-of-tables:
```toml
[[channels.telegram]]
bot_token_env = "TG_TOKEN_1"
account_id = "bot1"

[[channels.telegram]]
bot_token_env = "TG_TOKEN_2"
account_id = "bot2"
```

The API mirrors `Option<T>` for backward compatibility:

```rust
let tg = &config.channels.telegram;
if tg.is_some() {
    let first = tg.first().unwrap();
}
for bot in tg.iter() { /* ... */ }
```

Serialization behavior:
- **0 elements** → `null`
- **1 element** → bare value/table
- **2+ elements** → array

### `KernelConfig` — The Top-Level Struct

[`KernelConfig`](src/config/types.rs) holds the entire daemon configuration. It is deserialized from `config.toml` and provides `Default` for every field, so a minimal config file is valid.

Major configuration sections within `KernelConfig`:

| Field | Type | Purpose |
|-------|------|---------|
| `api_listen` | `String` | HTTP API bind address (default: `127.0.0.1:4545`) |
| `mode` | `KernelMode` | `Stable`, `Default`, or `Dev` operating mode |
| `channels` | `ChannelsConfig` | All channel adapter configs (20+ platforms) |
| `default_model` | `DefaultModelConfig` | Primary LLM provider/model |
| `memory` | `MemoryConfig` | Memory substrate backend and parameters |
| `web` | `WebConfig` | Web search + fetch (Brave, Tavily, Jina, Perplexity, DuckDuckGo) |
| `browser` | `BrowserConfig` | Headless Chromium automation |
| `docker` | `DockerSandboxConfig` | Container sandbox for code execution |
| `exec_policy` | `ExecPolicy` | Shell command allowlist/deny/security mode |
| `session` | `SessionConfig` | Retention, cleanup, context injection |
| `compaction` | `CompactionTomlConfig` | LLM-based history summarization thresholds |
| `queue` | `QueueConfig` | Agent command queue depth and concurrency |
| `proxy` | `ProxyConfig` | HTTP/HTTPS proxy with credential redaction |
| `auth_profiles` | `HashMap<String, Vec<AuthProfile>>` | Multi-key rotation per provider |
| `thinking` | `Option<ThinkingConfig>` | Extended thinking budget (Claude-style) |
| `triggers` | `TriggersConfig` | Event-driven trigger cooldowns and depth limits |
| `approval` | `ApprovalPolicy` | Human-in-the-loop execution approval |
| `tool_policy` | `ToolPolicy` | Global tool deny/allow rules |
| `provider_urls` | `HashMap<String, String>` | Per-provider base URL overrides |
| `provider_api_keys` | `HashMap<String, String>` | Per-provider API key env var overrides |
| `provider_regions` | `HashMap<String, String>` | Per-provider regional endpoint selection |
| `vault` | `VaultConfig` | Encrypted credential vault |
| `skills` | `SkillsConfig` | Skill loading, extra dirs, disabled list |
| `bindings` | `Vec<AgentBinding>` | Route specific channels/peers to agents |
| `tts` | `TtsConfig` | Text-to-speech (OpenAI, ElevenLabs, Google) |
| `canvas` | `CanvasConfig` | Agent-to-UI HTML rendering |
| `sidecar_channels` | `Vec<SidecarChannelConfig>` | External-process channel adapters |

### Key Enums

**`ExecSecurityMode`** — controls shell execution policy:
- `Deny` — block all shell commands
- `Allowlist` (default) — only `safe_bins` + `allowed_commands`
- `Full` — unrestricted (dev only)

Aliases like `"none"`, `"restricted"`, `"unrestricted"` are accepted via `#[serde(alias)]`.

**`DmPolicy`** / **`GroupPolicy`** — per-channel message routing:
- DM: `Respond` (default), `AllowedOnly`, `Ignore`
- Group: `MentionOnly` (default), `All`, `CommandsOnly`, `Ignore`

**`ResponseFormat`** — LLM output constraint:
- `Text` — free-form (default)
- `Json` — valid JSON, no schema
- `JsonSchema { name, schema, strict }` — schema-constrained JSON output

**`KernelMode`** — operating mode affecting auto-updates and experimental features.

### Validation

`KernelConfig` provides several validation methods:

- **`validate(&self) -> Vec<String>`** — returns warnings for structural issues: invalid ports, unrecognized log levels, missing env vars, network-without-secret.
- **`clamp_bounds(&mut self)`** — enforces minimums on dangerous fields (browser timeout, max sessions, fetch limits). Called during config loading to prevent zero-value footguns.
- **`detect_unknown_fields(raw: &toml::Value) -> Vec<String>`** — compares raw TOML keys against `known_top_level_fields()` to catch typos. When `strict_config = true`, unknown fields prevent startup.

The `strict_config` field controls the behavior:
- `false` (default) — unknown fields produce warnings, daemon boots normally
- `true` — unknown fields cause a startup failure

### Configuration Includes and Aliases

`KernelConfig` supports `include` for splitting config across files (relative paths only, no `..` traversal). Field aliases allow backward-compatible renames:

```rust
#[serde(alias = "listen_addr")]
pub api_listen: String,

#[serde(alias = "approval_policy")]
pub approval: ApprovalPolicy,
```

### Proxy Credential Redaction

`ProxyConfig` has a custom `Debug` impl that strips credentials:

```rust
let cfg = ProxyConfig {
    http_proxy: Some("http://admin:secret@proxy:8080".into()),
    ...
};
format!("{:?}", cfg);
// → "ProxyConfig { http_proxy: Some(\"http://***@proxy:8080\"), ... }"
```

The standalone function `redact_proxy_url()` handles the actual redaction and is available for use in logging elsewhere.

### API Key Resolution Order

`KernelConfig::resolve_api_key_env(provider)` resolves the environment variable name for a provider's API key using this precedence:

1. **Explicit mapping** in `provider_api_keys` — highest priority
2. **Auth profile** — the highest-priority profile's `api_key_env`
3. **Convention** — `{PROVIDER_UPPER}_API_KEY` (e.g., `"nvidia"` → `"NVIDIA_API_KEY"`)

### Config Schema Versioning

`config_version` (defaults to `1`) tracks the configuration schema version for future automatic migration. Set via `default_config_version()` in `version.rs`.

---

## Utility Functions

### `truncate_str`

```rust
pub fn truncate_str(s: &str, max_bytes: usize) -> &str
```

Truncates a string to at most `max_bytes` without splitting a UTF-8 character. Backs off to the previous character boundary when the cut point lands mid-character. Originally introduced to fix issue #104 where em dashes (3-byte UTF-8) caused panics in `kernel.rs` and `session.rs`.

Used by `subagent::truncate_output_preview()` and anywhere bounded output is needed.

### `VERSION`

```rust
pub const VERSION: &str = env!("CARGO_PKG_VERSION");
```

Injected from the workspace `Cargo.toml` at compile time. Available to all crates for consistent version reporting.

---

## How Other Crates Use This

The call graph reveals the primary usage patterns:

**Configuration access** — nearly every crate reads `KernelConfig`. The `OneOrMany` methods (`is_some`, `is_none`, `first`) appear across the runtime, channel adapters, skill loader, MCP integration, and API layer for checking whether channel/provider configs are present.

**Error propagation** — `LibreFangError` and `LibreFangResult<T>` are the standard error type returned by all kernel and runtime functions.

**Type-only contract** — because this crate contains zero business logic, it compiles quickly and doesn't pull in heavy dependencies. Other crates add `librefang-types` as a dependency to share type definitions without creating circular module coupling.

**Validation reuse** — the scheduler module's `validate()` and webhook's `validate()` are called from runtime and API handlers respectively, keeping validation logic co-located with the type definitions it guards.