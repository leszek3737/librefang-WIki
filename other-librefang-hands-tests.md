# Other — librefang-hands-tests

# librefang-hands/tests/registry_smoke

Integration smoke tests that exercise the `HandRegistry` public API lifecycle end-to-end. These tests operate against a temporary directory with no LLM, no kernel, and no network — they validate pure tool-dispatch and persistence behaviour.

## Purpose

Every historical regression in the hand registry has been a **cross-method invariant violation**: definitions present without a workspace directory, instances lingering after uninstall, or a refused uninstall corrupting in-memory state. Unit tests in `registry.rs` cover individual method contracts; these smoke tests catch the composition bugs that slip through when methods are called in sequence.

## Test Cases

### `install_activate_deactivate_uninstall_lifecycle`

Walks the complete lifecycle a custom hand goes through in production:

1. **Fresh state** — A new `HandRegistry` has zero definitions and zero instances.
2. **Install** — `install_from_content_persisted` writes `HAND.toml` and `SKILL.md` to `home/workspaces/<id>/` and registers the definition in memory. Asserts both files exist on disk and `list_definitions` / `get_definition` reflect the new hand.
3. **Activate** — `activate("smoke-hand", HashMap::new())` creates a live instance with status `HandStatus::Active`. Verifies the instance appears in `list_instances` and is retrievable via `get_instance`.
4. **Uninstall while active (must fail)** — `uninstall_hand` returns an error. Critically, asserts that the failed call does **not** perturb in-memory state: the definition and instance are still present. This is the invariant the DELETE `/api/hands/{id}` route depends on.
5. **Deactivate** — `deactivate(instance_id)` removes the instance from `list_instances`.
6. **Uninstall after deactivation (must succeed)** — `uninstall_hand` now succeeds. Asserts the in-memory definition is gone and the workspace directory is physically removed from disk.

### `definitions_round_trip_through_a_disk_reload`

Validates that `home/workspaces/<id>/HAND.toml` is the source of truth for persistence:

1. One `HandRegistry` installs a hand via `install_from_content_persisted`.
2. A second, independent `HandRegistry` is created (simulating a daemon restart). It starts empty.
3. `reload_from_disk(home)` discovers and loads the hand from the filesystem.
4. Asserts `get_definition` returns the correct data after reload.

This test locks in the contract that a process restart can fully recover state from disk without replaying any in-memory log.

## Fixtures

| Constant | Purpose |
|---|---|
| `SMOKE_HAND_TOML` | Minimal valid `HAND.toml` with id `smoke-hand`, category `data`, and a single routing alias. |
| `SMOKE_SKILL_MD` | Trivial Markdown body for the skill file. |

Both fixtures are embedded as string constants — no external files required.

## API Surface Under Test

```
HandRegistry::new()
HandRegistry::install_from_content_persisted(home, toml, skill_md)
HandRegistry::list_definitions()
HandRegistry::get_definition(id)
HandRegistry::activate(hand_id, config_map)
HandRegistry::list_instances()
HandRegistry::get_instance(instance_id)
HandRegistry::deactivate(instance_id)
HandRegistry::uninstall_hand(home, hand_id)
HandRegistry::reload_from_disk(home)
```

## Dependencies

- `tempfile` — creates isolated temporary directories so tests never touch real state.
- `librefang_hands::registry::HandRegistry` — the system under test.
- `librefang_hands::HandStatus` — enum for asserting instance status after activation.

## Running

```sh
cargo test -p librefang-hands --test registry_smoke
```

No environment variables, external services, or setup required.