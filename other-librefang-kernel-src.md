# Other — librefang-kernel-src

# Kernel Integration Tests (`kernel/tests.rs`)

## Purpose

This is the primary integration and regression test suite for `librefang-kernel`. It validates the kernel's core behaviors—agent lifecycle, capability enforcement, hand activation, skill management, provider routing, persistence, and background task orchestration—by booting real `LibreFangKernel` instances against temporary directories and asserting on in-memory state, SQLite rows, and file-system artifacts.

The tests double as **executable specifications**: many are annotated with issue numbers (e.g., `#3135`, `#2923`, `#2380`) and include doc comments explaining the exact regression they guard against.

## Test Infrastructure

### Kernel Boot Helper

Most tests follow a three-step pattern:

```
1. tempfile::tempdir()          → isolated home_dir
2. KernelConfig { home_dir, … } → minimal configuration
3. LibreFangKernel::boot_with_config(config) → live kernel
```

The kernel boots with a SQLite substrate, an empty agent registry, default skill registry, and no external LLM provider unless explicitly configured. Tests that need async operations use `#[tokio::test(flavor = "multi_thread")]`.

### Environment Variable Isolation

Tests that depend on environment variables (e.g., key rotation) use `set_test_env` / `EnvVarGuard`:

```rust
fn set_test_env(key: &'static str, value: &str) -> EnvVarGuard
```

The guard removes the variable on drop. Tests use **unique env-var names** (e.g., `LIBREFANG_TEST_ROTATION_PRIMARY_KEY_A`) and rely on the single-threaded default test runner for serialization. **Do not** use common names like `OPENAI_API_KEY` in tests.

### Recording Channel Adapter

`RecordingChannelAdapter` is a test double for `ChannelAdapter` that captures sent messages in an `Arc<Mutex<Vec<String>>>`. It's used in notification routing tests (e.g., `test_notify_escalated_approval_prefers_request_route_to`) by inserting it into `kernel.mesh.channel_adapters`.

### Skill Installation Helper

`install_test_skill` writes a minimal valid `skill.toml` + `prompt_context.md` under a given parent directory, enabling tests that exercise skill registry loading without depending on external skill packages.

## Behavioral Domains

### Agent Lifecycle & Capability Enforcement

```mermaid
graph TD
    A[spawn_agent_inner] --> B{parent given?}
    B -->|No| C[Register with manifest caps]
    B -->|Yes| D{parent exists in registry?}
    D -->|No| E[Fail closed: "not registered"]
    D -->|Yes| F{child caps ⊆ parent caps?}
    F -->|No| G[Fail: "Privilege escalation denied"]
    F -->|Yes| C
```

| Test | What it guards |
|------|---------------|
| `test_spawn_child_exceeding_parent_is_rejected` | Children cannot exceed parent capabilities — prevents privilege escalation through agent spawning (#3044-era hardening) |
| `test_spawn_child_with_subset_capabilities_is_allowed` | Positive counterpart: subset inheritance works |
| `test_spawn_with_unknown_parent_fails_closed` | Stale `AgentId` references fail rather than silently degrading to non-parent path |
| `test_spawn_agent_applies_local_default_model_override` | `"default"/"default"` provider/model stored at spawn; concrete resolution deferred to `execute_llm_agent` |

### Provider & Model Switching

`test_set_agent_model_clears_overrides_when_provider_changes` is the regression test for issue #2380. When `set_agent_model` switches an agent's provider (e.g., from `"cloudverse"` to `"openrouter"`), it **must** clear per-agent `api_key_env` and `base_url` overrides. Without this, the agent would continue hitting the old endpoint with stale credentials.

The test also verifies the **same-provider branch**: when only the model name changes within the same provider, existing overrides are preserved.

### Hand Activation, Runtime Overrides & Persistence

Hands are the kernel's pre-configured multi-agent bundles. The test suite covers:

**Activation basics:**
- `test_hand_activation_does_not_seed_runtime_tool_filters` — tool allowlist/blocklist remain empty so skill and MCP tools stay visible.
- `test_hand_skills_propagate_to_derived_agent_manifest` (#3135) — hand-level `skills = [...]` propagates into each role agent's `AgentManifest.skills`.
- `test_hand_skills_intersect_per_role_overrides` — when a per-role agent also declares `skills`, the effective list is the intersection.

**Runtime overrides & reactivation:**
- `test_hand_reactivation_rebuilds_same_runtime_profile` — deactivate + reactivate rebuilds from the hand definition, not from prior runtime overrides.
- `reactivate_builds_from_hand_toml_not_override` — every override field (model, provider, api_key_env, base_url, max_tokens, temperature, web_search_augmentation) is reset on fresh activation.

**Persistence across restarts:**
- `hand_runtime_override_survives_restart_via_activate_hand_with_id` — deterministically replays the `activate_hand_with_id` restore path and asserts all overrides round-trip through `hand_state.json`.
- `boot_drift_preserves_hand_settings_tail` / `boot_drift_preserves_skill_and_team_tails` — system prompt tails (`## User Configuration`, `## Reference Knowledge`, `## Your Team`) survive restart because `activate_hand_with_id` re-renders them from the HAND.tomL definition.

**Cleanup:**
- `deactivate_hand_removes_hand_agent_rows_from_sqlite` — after deactivation, SQLite rows are scrubbed even when the in-memory registry has already evicted the agents (the post-restart scenario).

### Tool Availability & Filtering

| Test | Behavior |
|------|----------|
| `test_available_tools_returns_empty_when_tools_disabled` | `tools_disabled: true` suppresses all builtin, skill, and MCP tools |
| `test_available_tools_glob_pattern_matches_mcp_tools` | Glob patterns like `"file_*"` correctly match `file_read`, `file_write`, `file_list` |
| `test_shell_exec_available_when_declared_in_tools_without_explicit_exec_policy` | Agents declaring `shell_exec` in `capabilities.tools` get their exec_policy auto-promoted to `Full` instead of inheriting the global `Deny` default |
| `test_skill_evolve_tools_default_available_to_restricted_agent` | Skill evolution tools (`skill_evolve_create`, etc.) are available to every agent regardless of declared capabilities |

### Background Sweeps & Task Board

Two idempotent background loops are tested:

- `test_spawn_approval_sweep_task_is_idempotent` — calling twice only starts one loop; the atomic flag `approval_sweep_started` prevents duplication.
- `test_spawn_task_board_sweep_task_is_idempotent` — same pattern for task-board sweeper (#2923).
- `test_task_board_sweep_resets_stuck_in_progress_task` — an in-progress task whose `claimed_at` exceeds the TTL is reset to `pending` with `assigned_to = ""`.

### Session Interrupt Cascade (Parent /stop)

When a parent agent is stopped mid-turn, the cancellation must propagate to child agents:

```
parent_interrupt.cancel()  →  child_interrupt.is_cancelled() == true
child_interrupt.cancel()   →  parent_interrupt.is_cancelled() == false  (one-way)
```

Key tests:
- `cascade_primitives_via_session_interrupts_dashmap` — validates `SessionInterrupt::new_with_upstream` cascade semantics via the `session_interrupts` DashMap.
- `no_upstream_when_parent_has_no_active_turn` — missing interrupt returns `None` without erroring.
- `send_to_agent_as_tolerates_unregistered_parent_uuid` — parent id resolution falls back to UUID parsing when the registry entry is gone.
- `send_to_agent_as_rejects_unparseable_parent_id` — garbage parent ids surface errors rather than panicking.

### Atomic TOML Persistence

`atomic_write_toml` replaces the previous plain `fs::write` with a write-to-temp-then-rename strategy:

```rust
// Stages to sibling .tmp, then fs::rename (atomic on same filesystem)
super::atomic_write_toml(&path, &content)
```

Tests verify:
- Content replacement works correctly.
- No `.tmp` files remain after success.
- Concurrent writers never produce a partial/corrupt file observable by readers.

### JSON Extraction from LLM Responses

`extract_json_from_llm_response` handles multiple LLM output formats:

| Format | Example |
|--------|---------|
| JSON code block | ` ```json\n{...}\n``` ` |
| Bare JSON object | `{...}` |
| JSON embedded in prose | Text before/after |
| Nested braces in strings | `{"k": "use {x} syntax"}` |
| Multiple code blocks | First valid block wins |
| Malformed JSON | Returns `None` |

### Sanitization for Background Review

`sanitize_reviewer_block` and `sanitize_reviewer_line` prevent a compromised prior LLM response from injecting fake instructions into the reviewer's context:

- Triple backticks are neutralized (prevent fake JSON blocks).
- `</data>` and `<data>` envelope tags are stripped.
- `[EXTERNAL SKILL CONTEXT]` brackets become parentheses.
- Null bytes and control characters are removed.
- Truncation is char-based (not byte-based) with a `…[truncated]` marker.

### Condition Evaluation

`evaluate_condition` supports a simple DSL for agent routing:

```rust
"agent.tags contains 'chat'"  → true if agent has the "chat" tag
"" or None                    → true (unconditional)
"some.unknown.expression"     → false (strict default — prevents injection)
```

### Peer-Scoped Key Namespacing

`peer_scoped_key` constructs namespaced storage keys with security validations (#5119, #5120):

```rust
peer_scoped_key("car", Some("user-123"))  → Ok("peer:user-123:car")
peer_scoped_key("car", None)              → Ok("car")  // global scope
peer_scoped_key("car", Some("u:456"))     → Err  // colon in peer_id
peer_scoped_key("car", Some(""))          → Err  // empty peer_id
peer_scoped_key("peer:victim:x", None)    → Err  // reserved prefix
```

### Cron Job Peer Context

`test_cron_create_preserves_peer_id` — the regression test for OFP-triggered cron jobs losing peer context. `cron_create` reads `peer_id` from `job_json` and persists it; without the fix, it always stored `None`.

## Conventions for Adding Tests

1. **Use `tempfile::tempdir()`** for every test. Never write to a shared or global path.
2. **Always call `kernel.shutdown()`** before the test returns, even on error paths. This cleans up background tasks and releases file locks.
3. **Use `set_test_env`** for environment variables. Never use `std::env::set_var` directly.
4. **Annotate regressions** with issue numbers in both the test name and doc comment. Future readers need the context.
5. **Handle graceful skips** for tests that depend on optional hands (like `"apitester"`):
   ```rust
   Err(e) if e.to_string().contains("unsatisfied requirements") => {
       eprintln!("Skipping test: {e}");
       kernel.shutdown();
       return;
   }
   ```
6. **Prefer synchronous tests** (`#[test]`) when possible. Use `#[tokio::test(flavor = "multi_thread")]` only when the code under test is `async`.
7. **The `#[ignore]` attribute** marks tests that require network or are inherently flaky in sandboxed CI. Run them locally with `cargo test -p librefang-kernel -- --ignored`.