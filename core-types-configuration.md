# Core Types & Configuration

# Core Types & Configuration (`librefang-types`)

The `librefang-types` crate defines the shared vocabulary of the LibreFang agent OS — every struct, enum, and identifier that crosses a crate boundary or persists to disk lives here. Downstream crates (`librefang-runtime`, `librefang-api`, `librefang-memory`, channel adapters) depend on these types but never redefine them.

Two modules form the backbone:

- **`agent`** — identity, manifests, lifecycle state, scheduling, resource quotas, and session resolution.
- **`approval`** — the human-in-the-loop approval gate that guards dangerous tool execution.

---

## Identity & Deterministic IDs

All entity IDs are newtype wrappers around `Uuid`. Where stability across restarts matters, the module uses **UUID v5** (SHA-1 + namespace) so the same logical name always maps to the same ID.

### `UserId`

```rust
pub struct UserId(pub Uuid);
```

| Constructor | UUID version | Use case |
|---|---|---|
| `UserId::new()` | v4 (random) | Transient/guest users |
| `UserId::from_name(name)` | v5 | Config-defined users — survives daemon restarts, preserves audit trail |

The namespace constant `LIBREFANG_USER_NAMESPACE` is frozen. Changing it would rotate every existing `UserId` and break cross-restart audit correlation.

### `AgentId`

```rust
pub struct AgentId(pub Uuid);
```

All deterministic variants use a single `AgentId::NAMESPACE` with typed prefixes to avoid collisions:

| Constructor | Hash input | Purpose |
|---|---|---|
| `AgentId::new()` | random v4 | Ephemeral agents |
| `AgentId::from_name(name)` | `"agent:{name}"` | Named agents from config |
| `AgentId::from_hand_id(hand_id)` | `hand_id` (bare, backward-compat) | Multi-agent hands |
| `AgentId::from_hand_agent(hand_id, role, None)` | `"{hand_id}:{role}"` | Legacy single-instance hands |
| `AgentId::from_hand_agent(hand_id, role, Some(id))` | `"{hand_id}:{role}:{id}"` | Multi-instance hands |

### `SessionId`

```rust
pub struct SessionId(pub Uuid);
```

Session resolution is the most nuanced part of the ID system. Three derivation paths serve different dispatch sources:

```
SessionId::for_channel(agent, channel)         — channel messages, cron (channel="cron")
SessionId::for_cron_run(agent, run_key)        — isolated per-fire cron sessions
SessionId::from_route_key(agent, ch, acct, conv) — multi-tenant channel routing
```

**Session resolution precedence** (highest to lowest):

1. **Explicit override** from the dispatch caller (HTTP API, fork plumbing) — always wins.
2. **Per-trigger `session_mode`** on the `Trigger` definition.
3. **Channel branch** — when a non-empty `SenderContext.channel` is present, always uses `SessionId::for_channel(agent, "channel:chat")`, overriding both per-trigger and manifest values.
4. **Cron branch** — synthesizes `SenderContext{channel:"cron"}`, but the cron dispatcher can pass an explicit `session_id_override` for per-fire isolation.
5. **Manifest `session_mode`** — final fallback.

`from_route_key` is backward-compatible with `for_channel`: when `account` is empty, it produces identical IDs. When `account` is non-empty, a `v2:` prefix ensures the hash space is disjoint.

---

## Agent Manifest (`AgentManifest`)

The manifest is the complete declarative specification of an agent — read from `agent.toml` at boot, stored in the kernel registry, and serialized to JSON for the API.

### Core Fields

| Field | Type | Default | Purpose |
|---|---|---|---|
| `name` | `String` | `"unnamed"` | Human-readable identifier |
| `module` | `String` | `"builtin:chat"` | WASM/Python module path |
| `schedule` | `ScheduleMode` | `Reactive` | How the agent wakes up |
| `session_mode` | `SessionMode` | `Persistent` | Session reuse vs. per-invocation |
| `model` | `ModelConfig` | provider=`"default"` | LLM configuration |
| `fallback_models` | `Vec<FallbackModel>` | `[]` | Chain tried on primary failure |
| `resources` | `ResourceQuota` | 256 MB, 30s CPU | Resource limits |
| `priority` | `Priority` | `Normal` | Scheduling priority |
| `capabilities` | `ManifestCapabilities` | all empty | Granted capabilities |
| `profile` | `Option<ToolProfile>` | `None` | Named tool preset |

### Scheduling (`ScheduleMode`)

```rust
pub enum ScheduleMode {
    Reactive,                                    // wake on message/event
    Periodic { cron: String },                   // cron schedule
    Proactive { conditions: Vec<String> },       // condition monitors
    Continuous { check_interval_secs: u64 },     // persistent loop (default 300s)
}
```

### Permission Mode (`AgentMode`)

Controls tool access at runtime, independent of the manifest-level allowlist:

| Mode | Behavior |
|---|---|
| `Observe` | No tools — agent can only read LLM output |
| `Assist` | Read-only tools only: `file_read`, `file_list`, `memory_list`, `memory_recall`, `web_fetch`, `web_search`, `agent_list` |
| `Full` | All granted tools (default) |

`AgentMode::filter_tools()` applies the mode to a `Vec<ToolDefinition>` at dispatch time.

### Tool Profiles (`ToolProfile`)

Named presets that expand into a tool list *and* derived `ManifestCapabilities`:

```
Minimal    → file_read, file_list
Coding     → file_read, file_write, file_list, shell_exec, web_fetch
Research   → web_fetch, web_search, file_read, file_write
Messaging  → agent_send, agent_list, channel_send, memory_store, memory_list, memory_recall
Automation → all of Coding + Messaging
Full/Custom → ["*"] (all tools)
```

`ToolProfile::implied_capabilities()` derives network, shell, agent_spawn, and memory permissions from the tool list.

### Workspaces (`WorkspaceDecl`)

Named shared directories agents can access:

```toml
[workspaces]
library = { path = "shared/library", mode = "rw" }
vault   = { mount = "/Users/me/Obsidian", mode = "r" }
```

- `path` — relative to `workspaces_dir`, auto-created by the kernel.
- `mount` — absolute host path (e.g., an Obsidian vault). Must be whitelisted in `config.toml: allowed_mount_roots`.

### Concurrency Control (`max_concurrent_invocations`)

Caps concurrent trigger-dispatch invocations per agent. Only affects event-trigger fan-out (`TaskPosted`, `MessageReceived`, etc.) — channel messages, cron, and `agent_send` are not throttled by this knob.

- Caps > 1 require `session_mode = "new"` on the manifest (parallel writes to a persistent session are undefined).
- The semaphore is sized on first dispatch and not invalidated by hot-reload; a restart is needed to pick up changes.

### Other Notable Fields

| Field | Purpose |
|---|---|
| `tool_allowlist` / `tool_blocklist` | Post-expansion tool filtering (blocklist wins) |
| `tools_disabled` | Nuclear option — disables all tools regardless of profile |
| `exec_policy` | Per-agent override for shell execution policy |
| `response_format` | Structured LLM output (plain JSON or schema-constrained) |
| `thinking` | Per-agent extended thinking config, overrides global `[thinking]` |
| `context_injection` | Merged with global session injections |
| `web_search_augmentation` | Auto-inject web results for models without tool support |
| `auto_dream_enabled` | Opt-in to background memory consolidation |
| `show_progress` | Surface `🔧 tool_name` progress lines in channel replies |
| `auto_evolve` | Background skill evolution after each turn (default: true) |
| `cache_context` | Read `context.md` once per session vs. every turn |
| `inherit_parent_context` | Subagent context inheritance (default: true) |
| `channel_overrides` | Per-agent dm_policy, group_policy overrides |

---

## Agent Entry (`AgentEntry`)

The runtime representation of a registered agent in the kernel's in-memory registry. Wraps the manifest with lifecycle state:

```rust
pub struct AgentEntry {
    pub id: AgentId,
    pub name: String,
    pub manifest: AgentManifest,
    pub state: AgentState,          // Created | Running | Suspended | Terminated | Crashed
    pub mode: AgentMode,            // Observe | Assist | Full
    pub session_id: SessionId,      // Active session
    pub parent: Option<AgentId>,    // Spawned-by chain
    pub children: Vec<AgentId>,
    pub identity: AgentIdentity,    // Emoji, avatar, color, archetype, vibe
    pub force_session_wipe: bool,   // Hard reset on next turn
    pub resume_pending: bool,       // Interrupted but recoverable
    pub has_processed_message: bool,// Distinguishes alive vs. never-dispatched
    // ...
}
```

`has_processed_message` is critical for the heartbeat monitor: agents that were spawned but never received work must not be flagged as unresponsive (which would cause a crash-recover loop).

---

## Model Configuration

### `ModelConfig`

```rust
pub struct ModelConfig {
    pub provider: String,
    pub model: String,            // alias: "name" in TOML
    pub max_tokens: u32,          // 4096
    pub temperature: f32,         // 0.7
    pub system_prompt: String,
    pub api_key_env: Option<String>,
    pub base_url: Option<String>,
    pub context_window: Option<u64>,     // override registry value
    pub max_output_tokens: Option<u64>,  // override catalog default
    pub extra_params: HashMap<String, serde_json::Value>, // flattened into API body
}
```

`extra_params` uses `#[serde(flatten)]` — keys are merged directly into the LLM API request body. Useful for provider-specific features like Qwen's `enable_memory`.

### `ModelRoutingConfig`

Auto-selects cheap/mid/expensive models by token count:

| Threshold | Default Model |
|---|---|
| Below `simple_threshold` (100) | `claude-haiku-4-5-20251001` |
| Between thresholds | `claude-sonnet-4-20250514` |
| Above `complex_threshold` (500) | `claude-sonnet-4-20250514` |

### `AutonomousConfig`

Guardrails for 24/7 autonomous agents:

- `max_iterations` — per-invocation LLM loop cap (default: 50)
- `max_restarts` — before permanent stop (default: 10)
- `heartbeat_interval_secs` — liveness check interval (default: 30)
- `heartbeat_timeout_secs` — optional per-agent timeout override
- `quiet_hours` — cron expression for suppression windows

---

## Approval System (`approval`)

The approval module implements a human-in-the-loop gate for dangerous tool operations. When an agent attempts a gated tool (e.g., `shell_exec`, `file_write`), the kernel creates an `ApprovalRequest`, pauses the agent, and waits for an `ApprovalResponse`.

### Core Types

**`ApprovalDecision`** — custom serialization that handles both simple strings and structured objects:

| Variant | Wire format |
|---|---|
| `Approved` | `"approved"` |
| `Denied` | `"denied"` |
| `TimedOut` | `"timed_out"` |
| `Skipped` | `"skipped"` |
| `ModifyAndRetry { feedback }` | `{"type": "modify_and_retry", "feedback": "..."}` |

**`RiskLevel`** — with display emojis:

| Level | Emoji |
|---|---|
| Low | ℹ️ |
| Medium | ⚠️ |
| High | 🚨 |
| Critical | ☠️ |

**`ApprovalRequest`** — includes validation constraints:
- `tool_name`: 1–64 chars, alphanumeric + underscores only
- `description`: max 1024 chars
- `action_summary`: max 512 chars
- `timeout_secs`: 10–300 range
- `session_id`: optional, enables `resolve_all_for_session` atomic resolution

### `ApprovalPolicy`

The master configuration for the approval gate:

```toml
[approval]
require_approval = ["shell_exec", "file_write", "file_delete", "apply_patch", "skill_evolve_*"]
timeout_secs = 60
auto_approve_autonomous = false
auto_approve = false           # shorthand: true → clear require_approval list

trusted_senders = ["user_123"]
timeout_fallback = "deny"      # deny | skip | { escalate = { extra_timeout_secs = 120 } }
second_factor = "totp"         # none | totp | login | both
totp_grace_period_secs = 300
totp_tools = ["shell_exec"]    # empty = all gated tools need TOTP

[[approval.channel_rules]]
channel = "telegram"
allowed_tools = ["file_read", "web_search"]
denied_tools = ["shell_exec"]

[[approval.routing]]
tool_pattern = "shell_*"
route_to = [{ channel_type = "telegram", recipient = "-10012345" }]
```

The `require_approval` field accepts either a list of tool names (with glob patterns like `skill_evolve_*`) or a boolean shorthand.

### Channel Tool Rules

Per-channel authorization via `ChannelToolRule`. Rules are evaluated in order; first match wins. Deny takes precedence over allow within a rule. Tool names support glob patterns via `glob_matches()`.

### Timeout Behavior (`TimeoutFallback`)

| Variant | Behavior |
|---|---|
| `Deny` | Auto-deny (default) |
| `Skip` | Skip the tool, agent continues |
| `Escalate` | Re-notify with extra timeout |

### Second Factor (`SecondFactor`)

| Variant | Login TOTP | Approval TOTP |
|---|---|---|
| `None` | ✗ | ✗ |
| `Totp` | ✗ | ✓ |
| `Login` | ✓ | ✗ |
| `Both` | ✓ | ✓ |

`tool_requires_totp()` checks both the global `second_factor` setting and the `totp_tools` allowlist.

### Audit

Every approval decision is recorded in an `ApprovalAuditEntry` with full context (tool, agent, risk level, decision, feedback, second-factor usage). Retention is controlled by `audit_retention_days` (default: 90 days).

---

## Key Design Principles

1. **Determinism where it matters.** `UserId`, `AgentId`, and `SessionId` all use UUID v5 derivation so the same logical input produces the same ID across restarts. This preserves session history, audit trails, and cron job associations.

2. **Backward compatibility in derivation.** `SessionId::from_route_key` produces identical IDs to `for_channel` when the account dimension is absent. `AgentId::from_hand_agent` with `instance_id: None` uses the legacy hash format.

3. **Layered tool filtering.** An agent's effective tool set is the intersection of: manifest `capabilities.tools`, `ToolProfile` expansion, `tool_allowlist`, `tool_blocklist`, `AgentMode` filtering, and the approval gate. Each layer is applied independently.

4. **Validation at the boundary.** Both `ApprovalRequest` and `ApprovalPolicy` implement `validate()` that checks field lengths, character sets, and range constraints. These run at deserialization time to reject malformed config early.

5. **Serde flexibility.** The approval module uses custom `Deserialize`/`Serialize` impls to handle polymorphic wire formats (string-or-object decisions, boolean-or-list require_approval) without breaking existing API consumers.