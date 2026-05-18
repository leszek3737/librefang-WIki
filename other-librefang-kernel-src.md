# Other — librefang-kernel-src

# librefang-kernel Test Suite

The `kernel::tests` module is the primary integration test surface for the LibreFang kernel. It exercises the full `LibreFangKernel` lifecycle — boot, agent spawning, hand activation/deactivation, background task sweeps, skill management, session interrupt cascades, and clean shutdown — against ephemeral tempdir-backed instances with no external network dependencies.

## Architecture

Every test follows the same contract:

1. Create a `tempfile::TempDir` for `home_dir`.
2. Build a `KernelConfig` pointing at it.
3. Call `LibreFangKernel::boot_with_config` (or a helper that wraps it).
4. Assert invariants against in-memory registries, substrate queries, or file system state.
5. Call `kernel.shutdown()`.

A small set of reusable helpers reduce boilerplate:

| Helper | Purpose |
|---|---|
| `set_test_env` / `EnvVarGuard` | Sets an env var for the test's lifetime; removes on drop. Uses unique prefixed keys per test to avoid cross-test collisions under the default serial test runner. |
| `test_manifest` | Produces a minimal `AgentManifest` with configurable name, description, and tags. |
| `install_test_skill` | Writes a valid `skill.toml` + `prompt_context.md` under a given parent directory so the registry's `load_skill` accepts it. |
| `RecordingChannelAdapter` | A `ChannelAdapter` impl that captures sent text into an `Arc<Mutex<Vec<String>>>` for assertion, with an empty inbound stream. |
| `cascade_test_kernel` | Boots a kernel with a leaked tempdir (process-lifetime) for session-interrupt cascade tests that need `Arc<LibreFangKernel>`. |

## Test Coverage by Subsystem

### API Key Rotation

`collect_rotation_key_specs` is tested for:

- **Deduplication** — when the primary key matches a profile's `api_key_env` value, that profile is marked `use_primary_driver: true` and the primary is not double-listed.
- **Missing profile skip** — profiles whose env var is unset are silently excluded.
- **Prepending** — a distinct primary key gets its own `RotationKeySpec` entry at position zero.

### Approval Escalation Routing

`test_notify_escalated_approval_prefers_request_route_to` validates the priority chain: a per-request `route_to` on an `ApprovalRequest` with `escalation_count > 0` wins over routing rules, agent notification rules, and the global `approval_channels` config. Uses `RecordingChannelAdapter` to confirm only the explicit recipient receives the message.

### Agent Manifest → Capabilities

`manifest_to_capabilities` is exercised across several scenarios:

- **Explicit tools** — `capabilities.tools` maps to `Capability::ToolInvoke` entries.
- **ToolProfile expansion** — a profile like `ToolProfile::Coding` expands to file tools + shell_exec + web_fetch + derived `ShellExec` / `NetConnect` capabilities.
- **Explicit tools override profile** — when both are set, only the explicit list is used; the profile is ignored.

### Agent Registry

The `AgentRegistry` is tested for:

- **Name lookup** via `find_by_name`.
- **UUID lookup** via `get`.
- **Tag filtering** — client-side iteration filtering on `tags` and `name` substrings.

### Agent Spawning & Lineage Enforcement

`spawn_agent_inner` is the low-level spawn path. Tests enforce:

| Test | Invariant |
|---|---|
| `test_spawn_child_exceeding_parent_is_rejected` | A child whose declared capabilities (tools, shell, network) exceed its parent's is rejected with `"Privilege escalation denied"`. The child is never registered. |
| `test_spawn_child_with_subset_capabilities_is_allowed` | A child requesting a strict subset of the parent's tools spawns successfully and records the parent link. |
| `test_spawn_with_unknown_parent_fails_closed` | Passing a stale `AgentId` as parent (not in the registry) fails with `"not registered"` — prevents silent privilege bypass via ghost parents. |
| `test_spawn_agent_applies_local_default_model_override` | An agent with `provider: "default"` / `model: "default"` resolves the concrete provider at execution time, not spawn time. The manifest stores the placeholder values. |

### Provider Switching & Stale Override Cleanup

`test_set_agent_model_clears_overrides_when_provider_changes` (regression for issue #2380) verifies that switching an agent's provider via `set_agent_model` clears `api_key_env` and `base_url` overrides from the previous provider, but a same-provider model-only swap preserves existing per-agent overrides.

### Hand Lifecycle

Hand activation and deactivation is one of the most heavily tested areas:

**Activation**

- `test_hand_activation_does_not_seed_runtime_tool_filters` — the `tool_allowlist` and `tool_blocklist` on the derived agent manifest remain empty so skill/MCP tools stay visible.
- `test_hand_skills_propagate_to_derived_agent_manifest` (issue #3135) — a hand-level `skills = [...]` allowlist propagates into each derived per-role agent's `AgentManifest.skills`.
- `test_hand_skills_intersect_per_role_overrides` — when both the hand and the per-role agent declare skills lists, the effective list is their intersection.

**Reactivation**

- `test_hand_reactivation_rebuilds_same_runtime_profile` — deactivating and reactivating a hand rebuilds the same tool set, profile, allowlist, blocklist, and MCP server assignments.
- `reactivate_builds_from_hand_toml_not_override` — runtime overrides (model, provider, max_tokens, temperature, web_search_augmentation) applied via `update_hand_agent_runtime_override` do NOT survive a deactivate/reactivate cycle. The fresh activation resolves from the HAND.tomL definition.

**Persistence across restarts**

- `hand_runtime_override_survives_restart_via_activate_hand_with_id` — after persisting hand state and rebooting a fresh kernel, replaying the `activate_hand_with_id` restore path re-applies the persisted `HandAgentRuntimeOverride` to the derived agent manifest.
- `boot_drift_preserves_hand_settings_tail` — `[[settings]]` from HAND.toml are rendered into the `## User Configuration` section of the system prompt after a simulated restart.
- `boot_drift_preserves_skill_and_team_tails` — `## Reference Knowledge` (from SKILL.md) and `## Your Team` (peer roster) tails survive restart via `activate_hand_with_id`.

**Deactivation cleanup**

- `deactivate_hand_removes_hand_agent_rows_from_sqlite` — even when hand agents are not in the in-memory registry (the post-restart scenario), `deactivate_hand` still scrubs their SQLite rows.

### Tool Availability

| Test | Invariant |
|---|---|
| `test_available_tools_returns_empty_when_tools_disabled` | `tools_disabled: true` on the manifest suppresses all builtin, skill, and MCP tools. |
| `test_available_tools_glob_pattern_matches_mcp_tools` | Declared tools like `"file_*"` use glob matching, not exact equality — regression for MCP tools being silently dropped. |
| `test_shell_exec_available_when_declared_in_tools_without_explicit_exec_policy` | When `shell_exec` is in `capabilities.tools` and `shell: ["*"]` is set but no `exec_policy` is provided, the kernel auto-promotes `exec_policy.mode` to `Full`. |
| `test_skill_evolve_tools_default_available_to_restricted_agent` | The `skill_evolve_*` surface is always visible to agents regardless of their declared `capabilities.tools` — enabling self-evolving skills for all agents. |

### Skill Registry Configuration

- `test_skills_config_disabled_list_filters_at_boot` — `config.skills.disabled` excludes named skills from loading even when their directories exist on disk.
- `test_skills_config_extra_dirs_loaded_as_overlay` — skills from `extra_dirs` are loaded on top of the primary directory; local installations with the same name win over external overlays.
- `test_reload_skills_preserves_disabled_and_extra_dirs` — hot-reload via `reload_skills()` re-applies the disabled list and extra_dirs overlay (regression for silent policy loss).
- `test_stable_mode_freezes_registry_and_skips_review_gate` — `KernelMode::Stable` freezes the registry, blocking new mutations and preventing the background-review gate from spawning reviews.

### Session Interrupt Cascade (Issue #3044)

Parent `/stop` propagates to in-flight child turns via `SessionInterrupt::new_with_upstream`:

```
cascade_primitives_via_session_interrupts_dashmap:
  parent_interrupt.cancel()  →  child_interrupt.is_cancelled() == true

  sibling_child.cancel()     →  sibling_parent.is_cancelled() == false  (reverse does NOT hold)
```

Additional tests verify:
- `no_upstream_when_parent_has_no_active_turn` — lookup returns `None` for idle parents; the call proceeds without cascade.
- `send_to_agent_as_tolerates_unregistered_parent_uuid` — the parent-id resolver falls back to UUID parsing when the registry lookup misses, preventing `"Agent not found"` from masking the real child-not-found error.
- `send_to_agent_as_rejects_unparseable_parent_id` — garbage parent IDs produce a clear error rather than a panic.

### Background Task Sweeps

- `test_spawn_approval_sweep_task_is_idempotent` / `test_spawn_task_board_sweep_task_is_idempotent` — atomic guards prevent double-spawning the sweep loops. After `shutdown()`, the flag resets.
- `test_task_board_sweep_resets_stuck_in_progress_task` — a claimed task whose `claimed_at` exceeds the TTL is reset to `pending` with `assigned_to: ""` via `task_reset_stuck`.

### Peer-Scoped Keys

`peer_scoped_key` is validated for security (issues #5119, #5120):

| Input | Expected |
|---|---|
| `("car", Some("user-123"))` | `"peer:user-123:car"` |
| `("car", None)` | `"car"` |
| `("car", Some("u:456"))` | `Err(InvalidInput)` — colon in peer_id |
| `("car", Some(""))` | `Err(InvalidInput)` — empty peer_id |
| `("peer:victim:user_name", None)` | `Err(InvalidInput)` — reserved prefix collision |

### JSON Extraction from LLM Responses

`extract_json_from_llm_response` handles:

- ` ```json ... ``` ` code blocks (first valid block wins)
- Bare `{...}` objects surrounded by prose
- Nested braces inside string values (the naive find/rfind approach was replaced)
- Malformed JSON → `None`
- No JSON present → `None`

### Background Skill Review Sanitization

`sanitize_reviewer_block` and `sanitize_reviewer_line` prevent a compromised prior LLM response from injecting fake instructions into the reviewer:

- Triple backticks are neutralized (prevents forged code blocks).
- `</data>` / `<data>` envelope tags are stripped (prevents escape from the prompt envelope).
- Control characters (`\x00`, `\x07`) are removed; whitespace and tabs preserved.
- Truncation is char-based, not byte-based, with a `…[truncated]` marker.

`is_transient_review_error` classifies errors so only timeouts, 429s, and network issues trigger retries — parse errors and security blocks are permanent.

### Cron Job Peer Context

`test_cron_create_preserves_peer_id` (regression for OFP-triggered cron jobs losing peer context): `cron_create` reads `peer_id` from `job_json` and persists it. Jobs created without `peer_id` store `null`.

### Atomic TOML Writes

`atomic_write_toml` stages content in a sibling `.tmp` file and atomically renames it into place:

- Replaces existing content cleanly.
- Leaves no `.tmp` artifacts on success.
- Under concurrent writes from two threads (50 iterations each), every intermediate read sees either the seed value or one of the two complete payloads — never a truncated mix.

### Thinking Override

`apply_thinking_override` toggles the `thinking` field on an agent manifest:

| Override | Existing `thinking` | Result |
|---|---|---|
| `None` | Any | Unchanged |
| `Some(false)` | Any | `thinking = None` |
| `Some(true)` | `None` | Default `ThinkingConfig` inserted |
| `Some(true)` | `Some(existing)` | Preserved with original `budget_tokens` |

### Message Routing

- `should_reuse_cached_route` — short follow-ups ("fix that", "继续") return `true`; acknowledgements ("thanks") and substantive requests return `false`.
- `assistant_route_key` — incorporates channel, user_id, and thread_id into a scoped key; `None` sender produces a different key.
- `test_boot_spawns_assistant_as_default_agent` — a fresh kernel boot auto-spawns an agent named `"assistant"`.
- `send_message_ephemeral` — errors on unknown agents and does not modify the real session's message history.

### Condition Evaluation

`evaluate_condition` supports `agent.tags contains 'value'` syntax. Empty/`None` conditions evaluate to `true`. Unknown formats evaluate to `false` (strict default).

## Running the Tests

```bash
# Full suite (no network required)
cargo test -p librefang-kernel

# Only the hand lifecycle tests
cargo test -p librefang-kernel -- hand

# The ignored end-to-end hand restart test (requires longer timeout)
cargo test -p librefang-kernel -- --ignored hand_runtime_override_survives_restart_via_start_background_agents
```

All tests create isolated tempdir homes and clean up on completion. The `EnvVarGuard` pattern ensures environment variable mutations never leak across test boundaries.