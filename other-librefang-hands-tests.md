# Other — librefang-hands-tests

# librefang-hands-tests — Registry Smoke Tests

## Purpose

This module contains integration smoke tests that validate the `HandRegistry` public API behaves correctly when its methods are composed together. These tests are not unit tests for individual methods — they verify **cross-method invariants** and **disk persistence contracts** that previous production bugs violated (e.g., definitions present but no workspace directory, instances lingering after uninstall).

The tests require no LLM, no kernel, and no running services. They exercise pure tool-dispatch and persistence behaviour against a temporary filesystem.

## Fixtures

Two constant strings provide the minimal valid inputs needed to install a hand:

- **`SMOKE_HAND_TOML`** — A complete `HAND.toml` with `id = "smoke-hand"`, a `routing` section with an alias, and an `[agent]` block. The `category = "data"` value satisfies schema validation without pulling in additional dependencies.
- **`SMOKE_SKILL_MD`** — A trivial Markdown file that serves as the skill body written to `SKILL.md`.

## Test Cases

### `install_activate_deactivate_uninstall_lifecycle`

End-to-end exercise of the full hand lifecycle. The call sequence and the invariants checked at each stage are:

```
HandRegistry::new()
  ├── list_definitions / list_instances  →  both empty
  │
  ├── install_from_content_persisted(home, toml, md)
  │     ├── returns def with id "smoke-hand"
  │     ├── creates home/workspaces/smoke-hand/HAND.toml on disk
  │     ├── creates home/workspaces/smoke-hand/SKILL.md on disk
  │     └── get_definition("smoke-hand") → Some
  │
  ├── activate("smoke-hand", HashMap::new())
  │     ├── returns instance with status HandStatus::Active
  │     └── list_instances → exactly one entry matching the instance
  │
  ├── uninstall_hand(home, "smoke-hand")
  │     ├── must ERR (refused — live instance exists)
  │     ├── get_definition still Some (state unperturbed)
  │     └── list_instances still length 1 (state unperturbed)
  │
  ├── deactivate(instance_id)
  │     ├── returns deactivated instance
  │     └── list_instances → empty
  │
  └── uninstall_hand(home, "smoke-hand")
        ├── succeeds (no live instances)
        ├── get_definition → None
        └── workspace directory physically removed from disk
```

The key cross-method invariants this test locks in:

| Invariant | Why it matters |
|---|---|
| Files persist to `workspaces/<id>/` after install | Daemon restarts must be able to reload state from disk |
| `uninstall_hand` refuses while instances exist | The `DELETE /api/hands/{id}` route depends on this guard |
| A refused uninstall leaves in-memory state untouched | Prevents partial corruption on error paths |
| Deactivation removes the instance from `list_instances` | Ensures cleanup is complete, not just a status change |
| Successful uninstall removes the workspace directory | Prevents orphaned disk usage |

The empty `HashMap::new()` passed to `activate` exercises the explicit-default path — equivalent to a dashboard "activate with defaults" action for a hand that declares no required settings.

### `definitions_round_trip_through_a_disk_reload`

Validates the disk-as-source-of-truth contract:

1. One `HandRegistry` installs a hand via `install_from_content_persisted`.
2. A second, independent `HandRegistry` is created (starts empty).
3. `reload_from_disk(home)` discovers and loads the previously installed hand.
4. `get_definition("smoke-hand")` returns the correct definition on the fresh registry.

This test ensures that a daemon process restart — where a new `HandRegistry` is constructed and `reload_from_disk` is called — re-discovers all previously installed hands without needing to replay an in-memory log. The `reload_from_disk` return value `(loaded, _failed)` is asserted to report at least one successful load.

## Disk Layout Contract

Both tests depend on the following filesystem layout established by `install_from_content_persisted`:

```
<home>/
└── workspaces/
    └── <hand-id>/
        ├── HAND.toml
        └── SKILL.md
```

This layout is the contract between `install_from_content_persisted` (write side) and `reload_from_disk` (read side). Changing it requires updating both the registry implementation and these tests.

## Dependencies

| Crate / Module | Usage |
|---|---|
| `librefang_hands::registry::HandRegistry` | System under test |
| `librefang_hands::HandStatus` | Asserting instance status after activation |
| `tempfile` | Isolated temporary directories per test |

## Running

```sh
cargo test -p librefang-hands-tests --test registry_smoke
```

These tests are safe to run in parallel — each creates its own `tempfile::tempdir` with no shared state.