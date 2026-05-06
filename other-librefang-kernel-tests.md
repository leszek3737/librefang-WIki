# Other — librefang-kernel-tests

# librefang-kernel Tests

Integration and contract test suite for the `librefang-kernel` crate. Tests exercise the kernel through its public APIs — `LibreFangKernel`, `KernelHandle` trait, and the `purge_sentinels` CLI binary — without mocking internal modules.

## Test Infrastructure

### Kernel Boot Helpers (`common/mod.rs`)

Two shared helpers boot a real kernel against a temporary directory:

- **`boot_kernel()`** — boots with default config and no users. Returns `(LibreFangKernel, TempDir)`.
- **`boot_kernel_with_users(users)`** — boots with a custom `Vec<UserConfig>` for RBAC tests.

Both create the required directory skeleton (`data/`, `skills/`, `workspaces/agents/`, `workspaces/hands/`) and configure SQLite at `data/test.db` with networking disabled.

### MockKernelBuilder (from `librefang-testing`)

Many tests use `MockKernelBuilder` instead of the raw boot helpers. The builder provides `with_config(|c| ...)` to mutate `KernelConfig` before `build()` returns a `(LibreFangKernel, TempDir)` pair. This is preferred when tests need to tweak retention settings, model defaults, or audit configuration.

### Runtime Flavor

Tests that call `start_background_agents()` or exercise deep async paths (agent loop, WASM execution, cron scheduling) require `#[tokio::test(flavor = "multi_thread")]`. The default current-thread runtime panics when the kernel calls `tokio::task::block_in_place` during synchronous substrate touches.

## Test Suites by Subsystem

### RBAC & Tool Policy

| File | Focus |
|---|---|
| `kernel_handle_contract_rbac.rs` | User-to-channel binding, memory ACL resolution, tool approval delegation |
| `rbac_m3_evaluate_tool_call.rs` | End-to-end RBAC M3 (#3054) — deny short-circuits, category groups, reload hot-pickup, fail-closed regression guards |

Key behaviors verified:

- **Channel-binding specificity** — a sender ID only matches its bound channel (Telegram `"111"` ≠ Discord `"111"`).
- **Fail-closed guest gate** — unrecognised senders get `NeedsApproval` for unsafe tools, not `Allow`. The `(None, None)` sender/channel pair also fails closed; only `Some("cron")` retains the system-call escape hatch.
- **Tool categories** — `UserToolCategories` resolves against `[tool_policy.groups]` in `KernelConfig`, supporting bulk allow/deny by group name.
- **Hot reload** — `AuthManager::reload(&new_users, &[])` invalidates cached policy so subsequent calls reflect updated deny lists.
- **`force_human` override** — `DeferredToolExecution.force_human=true` prevents the hand-agent auto-approve carve-out even for `is_hand=true` agents.

### Memory Isolation

`kernel_handle_contract_memory.rs` verifies that `memory_store` / `memory_recall` / `memory_list` maintain separate namespaces:

- `peer_id = None` → global namespace
- `peer_id = Some("peer-a")` → peer-scoped namespace
- Keys stored in one namespace are invisible to the other; `memory_list` returns only keys belonging to the requested scope.

### Agent Lifecycle & Hand Management

`multi_agent_test.rs` is the largest file (~900 lines) covering the full hand lifecycle:

- **Activation** — `activate_hand("test-clip", HashMap::new())` spawns agents, applies tool lists from the hand definition, tags agents with `hand:<id>` and `hand_instance:<uuid>`, and resolves `"default"` provider/model sentinels to the kernel's configured provider.
- **Deterministic IDs** — `AgentId::from_hand_agent(hand_id, role, None)` produces stable IDs so reactivation after deactivation reuses the same agent identity (legacy single-instance format).
- **Coordinator roles** — multi-agent hands (like `HAND_C` with `[agents.planner] coordinator = true`) use the declared coordinator for routing, persisted in `hand_state.json` as `coordinator_role`.
- **Pause/Resume** — paused agents remain in the registry; only `deactivate_hand` kills them.
- **Settings schema** — `[[settings]]` blocks declare typed keys with defaults. Activation seeds missing keys from schema defaults, preserves user overrides, and backfills new keys on reactivation against older state files.
- **State persistence** — `hand_state.json` (version 5) stores instance metadata including `agent_ids` map, `status`, `activated_at`, `updated_at`, and `config`.

### Task System

`kernel_handle_contract_task.rs` exercises the task lifecycle through `KernelHandle`:

1. **`task_post`** — creates a task with optional `assigned_to` (agent ID) and `created_by` (user ID). Both fields are persisted as nullable.
2. **`task_claim`** — returns the next assigned task for an agent.
3. **`task_complete`** — sets `status = "completed"` and stores the result string.

### Cron Scheduling

`kernel_handle_contract_cron_spawn.rs` validates:

- `cron_create` preserves `peer_id` when provided and stores `null` when omitted.
- `spawn_agent` returns a non-empty `(id, name)` tuple and the agent appears in `list_agents` / `find_agents`.

### Capability-Checked Spawning

`kernel_handle_contract_spawn_checked.rs` tests `spawn_agent_checked(manifest, parent_id, parent_caps)`:

- Spawns successfully when the child manifest doesn't exceed parent capabilities.
- **Rejects capability escalation** — a child requesting `tools = ["shell_exec"]` when the parent only has `FileRead("/data/*")` returns an error containing "escalation" or "denied".

### Workflow Engine

`workflow_integration_test.rs` covers:

- **Agent resolution** — `StepAgent::ByName { name }` resolves against the agent registry at registration time; `StepAgent::ById { id }` defers to execute time.
- **Run lifecycle** — `create_run` → `get_run` → `list_runs` round-trip. Step results include per-step token counts.
- **E2E with Groq** — a two-step analyst→writer pipeline runs through the real LLM when `GROQ_API_KEY` is set, verifying output flows between steps.
- **Trigger registration** — `register_trigger` / `list_triggers` / `remove_trigger` with `TriggerPattern::Lifecycle` and `TriggerPattern::SystemKeyword`.

### WASM Agent Execution

`wasm_agent_integration_test.rs` tests real WASM modules (compiled from WAT):

- **Echo module** — returns input JSON as-is; kernel extracts the response.
- **Hello module** — returns a fixed `{"response":"hello from wasm"}` string.
- **Infinite loop module** — triggers fuel exhaustion; kernel reports a fuel-related error.
- **Host-call proxy** — forwards input to `librefang.host_call` for host-side dispatch.
- **Streaming** — `send_message_streaming` produces at least `TextDelta` + `ContentComplete` events, falling back from SSE to a single response.
- **Mixed fleets** — WASM and LLM agents coexist in the same registry; killing one doesn't affect the other.

### Audit Retention

`audit_retention_test.rs` (M7) verifies:

- Kernel boots a periodic trim task when `audit.retention.trim_interval_secs` is configured.
- After seeding 50 entries against a cap of 10, the trim job collapses the log to ≤20 entries.
- A `RetentionTrim` self-audit row is written after each trim cycle.
- Chain integrity (`verify_integrity()`) survives trimming.

### Broader KernelHandle Contract

`kernel_handle_contract_broader.rs` covers remaining `KernelHandle` methods:

- **Roster** — `roster_upsert` / `roster_members` / `roster_remove_member` round-trip with channel-scoped membership.
- **Goals** — `goal_list_active(None)` returns empty for a fresh kernel.
- **A2A** — `list_a2a_agents()` and `get_a2a_agent_url()` return empty/None defaults.
- **Kill unknown** — `kill_agent("nonexistent-id")` returns an error.
- **Event publishing** — `publish_event` succeeds on a booted kernel.

### CLI Binary: purge_sentinels

`purge_sentinels_test.rs` drives the compiled binary via `std::process::Command`:

- **`--dry-run`** — reports removal counts without modifying files or creating `.bak`.
- **`--apply`** — creates `.bak` backups, removes whole-line sentinels (`NO_REPLY`, `[no reply needed]`, `no_reply` with whitespace), preserves sentence-embedded sentinels.
- **Idempotency** — second apply reports `removed=0` and leaves files unchanged.
- **Stale `.bak` detection** — aborts with an error if an existing `.bak` doesn't match the current file.
- **Error handling** — nonexistent path exits non-zero with a diagnostic message.

## Recursion Limit

Two test files set `#![recursion_limit = "256"]`:

- `audit_retention_test.rs` — `start_background_agents()` spawns 17 closures; after `TriggerId` gained `PartialOrd + Ord`, the compiler's type-resolution query exceeded the default limit of 128.
- `workflow_integration_test.rs` — deeply-nested futures from the kernel→runtime→agent_loop call chain, amplified by `LoopOptions` / `SessionInterrupt` fields.

## Running Tests

```bash
# All tests (fast, no external dependencies)
cargo test -p librefang-kernel

# Include live Groq LLM tests
GROQ_API_KEY=gsk_... cargo test -p librefang-kernel -- --include-ignored

# Single test file
cargo test -p librefang-kernel --test multi_agent_test

# Verbose output for integration tests
cargo test -p librefang-kernel --test integration_test -- --nocapture
```