# Other — librefang-hands-tests

# librefang-hands — Registry Smoke Tests

## Purpose

These integration smoke tests validate that the public API methods on `HandRegistry` compose correctly across the full hand lifecycle. They exist because every historical bug in this area was a **cross-method invariant violation** — for example, a definition remaining in memory after uninstall, or a workspace directory existing on disk without a corresponding in-memory definition.

No LLM or kernel is involved. The tests exercise pure tool-dispatch and persistence behaviour against a temporary filesystem.

## Test Fixtures

Two inline constants provide the minimal valid input for `install_from_content_persisted`:

- **`SMOKE_HAND_TOML`** — a valid `HAND.toml` with `id = "smoke-hand"`, one routing alias `"smoke"`, and a minimal `[agent]` block.
- **`SMOKE_SKILL_MD`** — a trivial `SKILL.md` body (`# Smoke Skill`).

Both are passed as string content; no fixture files on disk are required.

## Test Coverage

### `install_activate_deactivate_uninstall_lifecycle`

End-to-end lifecycle test that exercises every state transition on a fresh `HandRegistry` backed by a temporary home directory.

```mermaid
stateDiagram-v2
    [*] --> Empty : HandRegistry::new()
    Empty --> Installed : install_from_content_persisted()
    Installed --> Active : activate()
    Active --> Active : uninstall_hand() refused
    Active --> Installed : deactivate()
    Installed --> Empty : uninstall_hand()
```

The test asserts invariants at each transition:

| Step | Method called | Assertions |
|---|---|---|
| Fresh state | `new()` | `list_definitions()` and `list_instances()` are empty |
| Install | `install_from_content_persisted(home, toml, md)` | Returns definition with correct id/name; `HAND.toml` and `SKILL.md` exist at `home/workspaces/smoke-hand/`; definition visible via `list_definitions()` and `get_definition()` |
| Activate | `activate("smoke-hand", HashMap::new())` | Returns instance with `status: Active`; exactly one instance in `list_instances()`; retrievable via `get_instance()` |
| Refused uninstall | `uninstall_hand(home, "smoke-hand")` | Returns `Err`; definition and instance survive unchanged |
| Deactivate | `deactivate(instance_id)` | Returns the deactivated instance; `list_instances()` is now empty |
| Uninstall | `uninstall_hand(home, "smoke-hand")` | Returns `Ok`; `get_definition()` returns `None`; workspace directory removed from disk |

The **refused uninstall** step is critical: it validates the invariant that the `DELETE /api/hands/{id}` route depends on — a hand cannot be uninstalled while it has live instances. The test deliberately does not pin the error variant (that's covered by unit tests in `registry.rs`) but asserts that in-memory state is not perturbed by the refusal.

### `definitions_round_trip_through_a_disk_reload`

Validates the persistence contract: the directory layout `home/workspaces/<id>/HAND.toml` is the source of truth, and a fresh `HandRegistry` process can recover all installed hands by scanning disk — no in-memory replay log is needed.

**Flow:**

1. `installer` registry calls `install_from_content_persisted` against a temp home.
2. A second, empty `fresh` registry is created.
3. `fresh.reload_from_disk(home)` is called.
4. Asserts that at least one hand was loaded and `get_definition("smoke-hand")` returns the correct definition.

This locks in the daemon-restart scenario: if the process dies and restarts, `reload_from_disk` on the same home directory recovers all previously installed hands.

## Relationship to the Codebase

These tests call into `librefang_hands::registry::HandRegistry` exclusively. The `HandRegistry` methods exercised are:

- **Writes:** `install_from_content_persisted`, `activate`, `deactivate`, `uninstall_hand`
- **Reads:** `list_definitions`, `get_definition`, `list_instances`, `get_instance`
- **Recovery:** `reload_from_disk`

The tests do not touch the HTTP layer, the LLM runtime, or the kernel. They validate the state machine and persistence contracts that the API routes depend on.

## Running

```bash
cargo test -p librefang-hands --test registry_smoke
```

Both tests create isolated temporary directories via `tempfile::tempdir()` and leave no side effects.