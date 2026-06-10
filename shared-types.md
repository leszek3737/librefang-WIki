# Shared Types

# Agent Types (`librefang-types::agent`)

Central type definitions for LibreFang's agent system — identifiers, manifests, lifecycle states, session resolution, scheduling, resource quotas, and skill workshop configuration. Every other crate in the workspace depends on these types rather than defining their own, which keeps the wire format and serialization contract in one place.

## Identifier System

Three newtype wrappers — `UserId`, `AgentId`, `SessionId` — all wrap a `Uuid` and implement `Display`, `FromStr`, `Serialize`/`Deserialize`, and the standard equality/hash traits.

### Deterministic vs. Random IDs

Random (`::new()`) IDs are appropriate for ephemeral or one-shot use. For anything that must survive a daemon restart — audit log correlation, session history, cron job deduplication — use the deterministic UUID v5 derivation methods:

| Type | Method | Input | Namespace |
|---|---|---|---|
| `UserId` | `from_name(name)` | user name string | `LIBREFANG_USER_NAMESPACE` |
| `AgentId` | `from_name(name)` | `"agent:{name}"` | `AgentId::NAMESPACE` |
| `AgentId` | `from_hand_id(hand_id)` | bare `hand_id` string | `AgentId::NAMESPACE` |
| `AgentId` | `from_hand_agent(hand_id, role, instance_id)` | `"{hand_id}:{role}"` or `"{hand_id}:{role}:{instance_id}"` | `AgentId::NAMESPACE` |
| `SessionId` | `for_channel(agent_id, channel)` | `"{agent_id}:{channel}"` | `CHANNEL_SESSION_NAMESPACE` |
| `SessionId` | `for_sender_scope(agent_id, channel, chat_id)` | composed scope via `compose_sender_scope` | `CHANNEL_SESSION_NAMESPACE` |
| `SessionId` | `for_cron_run(agent_id, run_key)` | `"{agent_id}:{run_key}"` | `CRON_RUN_SESSION_NAMESPACE` |
| `SessionId` | `for_trigger_fire(agent_id, trigger_id, fire_time)` | `"{agent_id}:{trigger_id}:{rfc3339}"` | `TRIGGER_FIRE_SESSION_NAMESPACE` |
| `SessionId` | `from_route_key(agent_id, channel, account, conversation)` | varies (see below) | `CHANNEL_SESSION_NAMESPACE` |

The namespaces are deliberately disjoint so a session derived for a channel can never collide with one derived for a cron run or trigger fire, even if the input strings happen to coincide.

**Backward compatibility notes:**

- `AgentId::from_hand_agent` with `instance_id: None` produces the legacy `"{hand_id}:{role}"` hash so existing single-instance hands keep their IDs.
- `SessionId::from_route_key` with an empty `account` collapses to `for_channel` semantics; with a non-empty `account`, a `"v2:"` byte prefix keeps the hash space disjoint from legacy sessions.
- `AgentId::from_hand_id` uses the bare `hand_id` (no prefix) for backward compat with existing hand IDs.

### Scope Composition

`compose_sender_scope(channel, chat_id)` is the single source of truth for how `(channel, chat_id)` pairs map to scope strings:

- Both non-empty → `"{channel}:{chat_id}"`
- Chat ID absent/empty → `"{channel}"`
- Channel empty → `None`

Both the kernel's session-id derivation and the runtime's memory-scope filter call this helper. If they ever disagree, memory written under one chat leaks into another chat's session — exactly the regression that motivated extracting this function.

### Name Validation

`validate_agent_name(name)` rejects names starting with `_operator:`, the reserved prefix used by the workflow engine for synthetic operator-node step results (issue #4980). This prevents ambiguity between built-in workflow steps and user-defined agents.

## Agent Manifest (`AgentManifest`)

The complete declarative configuration for an agent, typically loaded from `agent.toml`. It is a large struct with `#[serde(default)]` so partial TOML files fill in compiled defaults for missing fields.

### Key Field Groups

**Identity & metadata:** `name`, `version`, `description`, `author`, `tags`, `metadata`

**LLM configuration:** `model` (`ModelConfig`), `fallback_models` (three-state: `None` = inherit global, `Some([])` = disable, `Some([…])` = explicit chain), `routing` (`ModelRoutingConfig`), `pinned_model`, `thinking`, `response_format`

**Session & scheduling:** `schedule` (`ScheduleMode`), `session_mode` (`SessionMode`), `max_concurrent_invocations`

**Tool & capability surface:** `profile` (`ToolProfile`), `tools` (per-tool params), `tool_allowlist`, `tool_blocklist`, `tools_disabled`, `capabilities` (`ManifestCapabilities`), `mcp_servers`, `mcp_disabled`, `allowed_plugins`

**Resource limits:** `resources` (`ResourceQuota`), `priority` (`Priority`)

**Autonomous operation:** `autonomous` (`AutonomousConfig`), `web_search_augmentation`

**Workspace:** `workspace`, `workspaces` (named shared workspaces), `generate_identity_files`

**Skill & memory:** `skills`, `skills_disabled`, `skill_workshop` (`SkillWorkshopConfig`), `auto_evolve`, `proactive_memory`, `auto_dream_enabled`, `auto_dream_min_hours`, `auto_dream_min_sessions`

**Event triggers:** `triggers` (declarative `ManifestTrigger` list), `reconcile_orphans` (`OrphanPolicy`)

**Advanced overrides:** `exec_policy`, `tool_exec_backend`, `channel_overrides`, `max_history_messages`, `compaction` (`CompactionOverrides`), `rl_export` (`RlExportOverride`), `context_injection`, `inherit_parent_context`, `cache_context`, `show_progress`, `async_tasks` (`AsyncTasksConfig`)

### Session Mode Resolution Precedence

When the kernel dispatches a message, the session ID is resolved with this priority:

1. **Explicit `session_id_override`** from the dispatch caller (HTTP, multi-tab UI, fork plumbing) — always wins.
2. **Per-trigger override** (`ManifestTrigger::session_mode`) — honored for event triggers and `agent_send`.
3. **Channel branch** — always uses `SessionId::for_channel(agent, "channel:chat")` when a non-empty `SenderContext.channel` is present (and `use_canonical_session` is false).
4. **Cron** — synthesizes `SenderContext { channel: "cron" }` and takes the channel branch, unless the per-job `session_mode` is `New`, in which case the dispatcher passes an explicit `session_id_override` derived from the job ID and fire timestamp.
5. **Manifest `session_mode`** — final fallback for event triggers and `agent_send`.

The kernel emits `tracing::debug!` with `resolution_source` when a higher-priority path overrides the manifest setting, so operators can verify why a `session_mode = "new"` declaration is or isn't being honored.

## Session Modes (`SessionMode`)

```rust
pub enum SessionMode {
    Persistent,  // reuse agent's persistent session (default)
    New,         // fresh session per invocation
}
```

**Strict deserialization:** unknown variant strings produce a hard error — there is no `#[serde(other)]` fallback. This is intentional: silently mapping a typo like `"New"` (capitalized) to `Persistent` would violate the concurrent-write safety contract for persistent sessions.

## Agent State & Modes

### Lifecycle States (`AgentState`)

```
Created → Running → Suspended → Running  (resume)
                  → Terminated           (final)
                  → Crashed              (awaiting recovery)
```

### Permission Modes (`AgentMode`)

| Mode | Behavior |
|---|---|
| `Observe` | All tools removed |
| `Assist` | Only read-only tools pass: `file_read`, `file_list`, `memory_list`, `memory_recall`, `web_fetch`, `web_search`, `agent_list` |
| `Full` | All granted tools pass (default) |

`AgentMode::filter_tools()` applies the filter to a `Vec<ToolDefinition>`.

## Scheduling (`ScheduleMode`)

```rust
pub enum ScheduleMode {
    Reactive,                                              // event-driven (default)
    Periodic { cron: String },                             // cron schedule
    Proactive { conditions: Vec<String> },                 // condition monitoring
    Continuous { check_interval_secs: u64 },               // persistent loop (default: 300s)
}
```

## Model Configuration (`ModelConfig`)

Defines the LLM provider, model identifier, generation parameters, and optional overrides. Notable fields:

- **`context_window` / `max_output_tokens`**: Optional overrides when the runtime-probed value differs from reality (e.g., self-hosted Ollama with custom `num_ctx`).
- **`extra_params`**: A `BTreeMap<String, serde_json::Value>` flattened into the API request body via `#[serde(flatten)]`. `BTreeMap` ensures deterministic key order for prompt-cache stability (issue #3298). Conflicting keys in `extra_params` take precedence over standard fields.
- **TOML alias**: `name` is accepted as an alias for `model` in TOML for backward compatibility.

### Model Routing (`ModelRoutingConfig`)

Auto-selects between `simple_model`, `medium_model`, and `complex_model` based on token count thresholds. Defaults to Claude Haiku (simple) and Claude Sonnet (medium/complex).

### Fallback Chain (`FallbackModel`)

`AgentManifest::fallback_models` is a three-state option:
- `None` → inherit global `fallback_providers`
- `Some([])` → explicitly disable fallbacks for this agent
- `Some([model1, model2, ...])` → try in order on primary failure

The custom deserializer `option_vec_lenient` ensures the three states round-trip correctly through both TOML and JSON.

## Resource Quotas (`ResourceQuota`)

Per-agent resource limits with sensible compiled defaults:

| Field | Default |
|---|---|
| `max_memory_bytes` | 256 MB |
| `max_cpu_time_ms` | 30,000 (30s) |
| `max_tool_calls_per_minute` | 60 |
| `max_llm_tokens_per_hour` | `None` (inherit global) |
| `burst_ratio` | `None` (compiled default: 0.2 = 1/5 of hourly budget per minute) |
| `max_network_bytes_per_hour` | 100 MB |
| `max_cost_per_{hour,day,month}_usd` | 0.0 (unlimited) |

**Token limit resolution:**
- `effective_token_limit()`: `None` and `Some(0)` both yield `0` (unlimited). `Some(n)` yields `n`.
- `effective_burst_ratio(global_default)`: agent override > global default > 0.2. Clamped to `[0.01, 1.0]`. NaN/Inf fall back to 0.2.

## Tool Profiles (`ToolProfile`)

Named presets that expand to tool lists and derived capabilities:

| Profile | Tools |
|---|---|
| `Minimal` | `file_read`, `file_list` |
| `Coding` | `file_read`, `file_write`, `file_list`, `shell_exec`, `web_fetch` |
| `Research` | `web_fetch`, `web_search`, `file_read`, `file_write` |
| `Messaging` | `agent_send`, `agent_list`, `channel_send`, `memory_store`, `memory_list`, `memory_recall` |
| `Automation` | all of the above (12 tools) |
| `Full` / `Custom` | `["*"]` (wildcard) |

`implied_capabilities()` derives a `ManifestCapabilities` from the tool list — inferring network access from `web_*` tools, shell from `shell_exec`, agent messaging from `agent_*`, etc.

## Skill Workshop (`SkillWorkshopConfig`)

Passive after-turn capture of reusable workflows. Default is **disabled** (opt-in per agent).

| Field | Default | Purpose |
|---|---|---|
| `enabled` | `false` | Master switch |
| `auto_capture` | `true` | Run heuristic/LLM scan on each turn |
| `review_mode` | `Heuristic` | Classification mode (no LLM cost) |
| `approval_policy` | `Pending` | Candidates wait for human review |
| `max_pending` | `20` | LRU eviction cap per agent |
| `max_pending_age_days` | `None` | Optional TTL for pending candidates |
| `evolution_mode` | `Free` | Background evolution routing |

### Review Modes

- **`Heuristic`** — regex/pattern matching only, zero token cost
- **`ThresholdLlm`** — heuristic gate first, then a cheap auxiliary LLM confirms
- **`None`** — scanner runs but drops all hits (test-only)

### Evolution Modes

- **`Free`** (default) — creates go to pending queue, updates/patches apply directly
- **`Controlled`** — all background mutations (create, update, patch) route through pending queue for human approval

### Approval Policies

- **`Pending`** — write to `pending/`, await human review via CLI or dashboard
- **`Auto`** — write to `pending/` (for audit trail and injection scan), then auto-promote to `active/`

## Declarative Triggers (`ManifestTrigger`)

Symmetric to runtime triggers created via `POST /api/triggers`, but declared in `agent.toml`. On spawn/reload the kernel reconciles this list against the trigger store: missing entries are created, drifted entries are updated in place (TOML wins), runtime-only triggers are handled by `reconcile_orphans`:

| Policy | Behavior |
|---|---|
| `Keep` (default) | Runtime-only triggers preserved — conservative |
| `Warn` | Log orphan trigger IDs, keep them |
| `Delete` | Remove runtime-only triggers from store |

## Compaction Overrides (`CompactionOverrides`)

Per-agent override for the kernel-global `[compaction]` configuration. Each field is `Option<_>` — when `Some`, it supersedes the global value for this agent. The `resolve(global)` method merges overrides onto the global config, producing a fresh config. The `token_threshold_ratio` override is clamped to `[0.0, 1.0]` to prevent misconfiguration.

Resolution order: agent override → global config → compiled defaults.

## Agent Entry (`AgentEntry`)

The runtime registry entry for a live agent — combines identity (`AgentId`, `name`), configuration (`AgentManifest`), lifecycle state (`AgentState`, `AgentMode`), and operational flags:

- **`force_session_wipe`** — next execution clears message history (operator action or stuck-loop recovery)
- **`resume_pending`** — agent was interrupted by restart, expects to continue on the same transcript
- **`has_processed_message`** — sticky flag set after first real message dispatch; prevents the heartbeat monitor from flagging never-used agents as unresponsive

## Agent Identity (`AgentIdentity`)

Visual metadata for dashboard display: emoji, avatar URL, hex color, archetype, vibe, and greeting style. All fields optional.

## Session Labels (`SessionLabel`)

Validated string wrapper (1–128 chars, alphanumeric + spaces + hyphens + underscores). Created via `SessionLabel::new(label)` which returns `Result`.

## Hook Events (`HookEvent`)

Interceptable lifecycle points:

- `BeforeToolCall` — handler can block the call
- `AfterToolCall` — post-execution hook
- `TransformToolResult` — rewrite tool output (first `Ok(Some(s))` wins)
- `BeforePromptBuild` — pre-system-prompt construction
- `AgentLoopEnd` — after agent loop completes

## Autonomous Configuration (`AutonomousConfig`)

Guardrails for 24/7 autonomous agents:

| Field | Default | Purpose |
|---|---|---|
| `max_iterations` | 50 | Hard cap on LLM iterations per invocation |
| `max_restarts` | 10 | Crashes before permanent stop |
| `heartbeat_interval_secs` | 30 | Health check cadence |
| `heartbeat_timeout_secs` | `None` | Per-agent timeout override (default: `interval × 2`) |
| `heartbeat_keep_recent` | `None` | How many NO_REPLY heartbeat turns to keep |
| `block_stall_degrade_after` | 2 | Consecutive block-only iterations before forced tools-stripped completion (issue #5979) |

## Autonomous Web Search (`WebSearchAugmentationMode`)

Controls pre-call web search injection for models without tool support:

| Mode | Behavior |
|---|---|
| `Off` | Disabled |
| `Auto` (default) | Search only when model catalog reports `supports_tools == false` |
| `Always` | Search before every LLM call |

## Workspace Declarations (`WorkspaceDecl`)

Named shared workspaces with either a `path` (relative to `workspaces_dir`, auto-created) or `mount` (absolute host path, must be whitelisted in `config.toml: allowed_mount_roots`). Access mode is `ReadWrite` (default) or `ReadOnly`.

## Async Task Configuration (`AsyncTasksConfig`)

| Field | Default | Purpose |
|---|---|---|
| `default_timeout_secs` | `None` | Wall-clock timeout for async tasks |
| `notify_on_timeout` | `true` | Inject synthetic `TaskCompletionEvent` on timeout |

## Concurrency Control (`max_concurrent_invocations`)

Per-agent cap on concurrent trigger-dispatch fan-out. Channel messages, cron, and `agent_send` are **not** throttled by this knob. Caps > 1 require manifest-level `session_mode = "new"` — concurrent writes to a persistent session are undefined. The per-agent semaphore is sized on first dispatch and is not invalidated by manifest hot-reload.

## Diagram: Session ID Derivation

```mermaid
graph TD
    A[Dispatch arrives] --> B{session_id_override?}
    B -->|Yes| Z[Use override]
    B -->|No| C{Trigger override?}
    C -->|Yes| Z2[Use trigger session_mode]
    C -->|No| D{Has channel?}
    D -->|Yes| E[for_sender_scope]
    D -->|No| F{Cron?}
    F -->|Yes, session_mode=New| G[for_cron_run]
    F -->|Yes, persistent| H[for_channel agent "cron"]
    F -->|No| I{Manifest session_mode}
    I -->|Persistent| J[Persistent session]
    I -->|New| K[SessionId::new]
```

## Constants

| Constant | Value | Purpose |
|---|---|---|
| `STABLE_PREFIX_MODE_METADATA_KEY` | `"stable_prefix_mode"` | Metadata key for stable prefix mode |
| `LIBREFANG_USER_NAMESPACE` | Fixed UUID | Namespace for `UserId::from_name` — changing it breaks all existing user IDs |
| `RESERVED_OPERATOR_AGENT_NAME_PREFIX` | `"_operator:"` | Reserved for workflow engine synthetic agents |

**Warning:** `LIBREFANG_USER_NAMESPACE` and `AgentId::NAMESPACE` must never be changed. Rotating either constant re-keys every existing derived ID and breaks audit-log correlation across the fleet.