# Other — librefang-hands-tests

# librefang-hands/tests — Registry Smoke Tests

## Purpose

Integration smoke tests that exercise the `HandRegistry` public API end-to-end, catching cross-method invariant violations that unit tests alone often miss. Every historical regression in this area was a compositional bug — definitions present without a workspace, instances lingering after uninstall, etc. — so these tests treat the registry as a black box and verify that the full lifecycle composes correctly.

No LLM, no kernel — these tests validate purely tool-dispatch and persistence behaviour.

## Test Fixtures

Two inline constants define the minimal hand content needed for installation:

| Constant | Format | Purpose |
|---|---|---|
| `SMOKE_HAND_TOML` | TOML (id, name, description, category, routing, agent) | A complete `HAND.toml` for a hand named `"smoke-hand"` |
| `SMOKE_SKILL_MD` | Markdown | A minimal `SKILL.md` body |

The hand declares no required settings, which is significant — it means activation can succeed with an empty `HashMap::new()` config, testing the explicit-default code path.

## Test Functions

### `install_activate_deactivate_uninstall_lifecycle`

Walks the complete state machine of a hand from installation through cleanup. Each step asserts not just its own outcome but the observable side effects visible through read-side APIs.

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌────────────┐    ┌───────────┐
│ install  │───▶│ activate │───▶│ uninstall │───▶│ deactivate │───▶│ uninstall  │
│          │    │          │    │ (refused) │    │            │    │ (succeeds) │
└─────────┘    └─────────┘    └──────────┘    └────────────┘    └───────────┘
```

**Steps and invariants verified:**

1. **Fresh registry is empty** — `list_definitions()` and `list_instances()` both return empty collections.
2. **`install_from_content_persisted(home, toml, md)`** — Writes `HAND.toml` and `SKILL.md` to `home/workspaces/smoke-hand/`, returns the definition with correct `id` and `name`. Both files are asserted to exist on disk. The definition becomes visible via `list_definitions()` and `get_definition("smoke-hand")`.
3. **`activate("smoke-hand", HashMap::new())`** — Creates an active instance (`HandStatus::Active`). The instance is visible in `list_instances()` (exactly one) and retrievable via `get_instance(id)`.
4. **`uninstall_hand(home, "smoke-hand")` while active** — Must fail. The test asserts the error without pinning a specific error variant (unit tests cover that). Critically, it also asserts that the refused uninstall does **not** corrupt in-memory state: the definition and instance both survive untouched.
5. **`deactivate(instance_id)`** — Returns the deactivated instance. `list_instances()` must now be empty.
6. **`uninstall_hand(home, "smoke-hand")` with no live instances** — Must succeed. Removes the in-memory definition (confirmed via `get_definition` returning `None`) and physically deletes the workspace directory from disk.

### `definitions_round_trip_through_a_disk_reload`

Validates that the `home/workspaces/<id>/HAND.toml` layout is the authoritative source of truth. After one registry instance writes a hand to disk, a completely independent `HandRegistry` can discover it via `reload_from_disk`.

**Steps:**

1. `installer` registry calls `install_from_content_persisted` into a temp home.
2. `fresh` — a new, empty `HandRegistry` — starts with zero definitions.
3. `fresh.reload_from_disk(home)` returns `(loaded, _failed)` where `loaded >= 1`.
4. `fresh.get_definition("smoke-hand")` returns the definition with `name == "Smoke Hand"`.

This test locks in the contract that a daemon restart can fully rehydrate state from disk without replaying any in-memory log.

## Dependencies on `HandRegistry` API

The tests exercise every primary method on the public registry surface:

| Method | Called by |
|---|---|
| `HandRegistry::new()` | Both tests |
| `install_from_content_persisted(home, toml, md)` | Both tests |
| `list_definitions()` | Lifecycle test |
| `get_definition(id)` | Both tests |
| `activate(hand_id, config)` | Lifecycle test |
| `list_instances()` | Lifecycle test |
| `get_instance(instance_id)` | Lifecycle test |
| `deactivate(instance_id)` | Lifecycle test |
| `uninstall_hand(home, hand_id)` | Lifecycle test |
| `reload_from_disk(home)` | Reload test |

## Running

```sh
# Run just these smoke tests
cargo test -p librefang-hands --test registry_smoke

# Run with output for debugging
cargo test -p librefang-hands --test registry_smoke -- --nocapture
```

No external services, network, or environment variables are required. The tests create isolated temp directories via `tempfile::tempdir()` that are cleaned up automatically on drop.