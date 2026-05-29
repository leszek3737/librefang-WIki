# Other — librefang-hands-tests

# librefang-hands Tests — Registry Smoke Tests

## Purpose

This module (`registry_smoke.rs`) contains integration smoke tests that exercise the `HandRegistry` public API end-to-end. Unlike unit tests that validate individual methods in isolation, these tests verify that the registry methods **compose correctly** across a full lifecycle. The historical motivation is explicit: every previous bug in this area was a cross-method invariant violation (e.g., definitions present without a workspace, instances surviving uninstall).

There is no LLM involvement and no kernel — these tests validate tool-dispatch and persistence behavior only.

## Test Fixtures

Two inline constants provide the fixture data, avoiding any dependency on external files or network resources:

- **`SMOKE_HAND_TOML`** — A minimal valid `HAND.toml` defining a hand with id `smoke-hand`, category `data`, routing alias `smoke`, and a stub agent with an echo-style system prompt.
- **`SMOKE_SKILL_MD`** — A trivial `SKILL.md` body ("Integration-test skill body.").

Both are passed to `install_from_content_persisted` which writes them to disk under `home/workspaces/<id>/`.

## Test Cases

### `install_activate_deactivate_uninstall_lifecycle`

This is the primary regression test. It drives the complete state machine of a hand through every public transition:

```mermaid
stateDiagram-v2
    [*] --> Empty : HandRegistry::new()
    Empty --> Defined : install_from_content_persisted()
    Defined --> Active : activate()
    Active --> Defined : deactivate()
    Active --> Active : uninstall_hand() → ERR
    Defined --> Empty : uninstall_hand()
```

**What it verifies at each step:**

| Step | Assertions |
|------|-----------|
| Fresh registry | `list_definitions()` and `list_instances()` return empty |
| Install | Returns definition with correct id/name; `HAND.toml` and `SKILL.md` exist on disk at `workspaces/<id>/`; definition visible via `list_definitions()` and `get_definition()` |
| Activate | Returns instance with `HandStatus::Active`; instance appears in `list_instances()` and is retrievable via `get_instance()` |
| Uninstall while active | Must fail (returns `Err`); definition and instance must remain unchanged — no partial state corruption |
| Deactivate | Returns the same instance; `list_instances()` must be empty afterward |
| Uninstall after deactivate | Must succeed; definition gone from memory (`get_definition()` returns `None`); workspace directory physically removed from disk |

The test uses an empty `HashMap::new()` for config on activation — this is the explicit-default path for a hand with no required settings, equivalent to a dashboard "activate with defaults" action.

### `definitions_round_trip_through_a_disk_reload`

Validates the disk-as-source-of-truth contract. After one registry writes a hand via `install_from_content_persisted`, a **completely separate** `HandRegistry` instance starts empty and calls `reload_from_disk(home)` on the same directory. It must recover the hand definition.

This locks in the guarantee that a daemon restart can re-discover all installed hands from `home/workspaces/<id>/HAND.toml` without replaying any in-memory log.

Key assertions:

- The second registry starts empty (`list_definitions()` is empty before reload)
- `reload_from_disk` returns a `loaded` count ≥ 1 and no unexpected failures
- `get_definition("smoke-hand")` returns a definition with `name == "Smoke Hand"`

## Relationship to the Rest of the Codebase

These tests sit above `librefang_hands::registry` and consume its public API exclusively:

- `HandRegistry::new()` — constructor
- `install_from_content_persisted()` — writes to disk + registers in memory
- `activate()` / `deactivate()` — instance lifecycle
- `list_definitions()` / `list_instances()` — read-side queries
- `get_definition()` / `get_instance()` — point lookups
- `uninstall_hand()` — guarded removal (refuses while instances exist)
- `reload_from_disk()` — restores state from persisted files

The `HandStatus` enum is imported from the crate root.

The tests use `tempfile::tempdir()` to guarantee isolation — each test gets a fresh, empty home directory that is cleaned up on drop. No shared state exists between tests.

## When to Extend These Tests

Add a new smoke test here when:

- A new public method is added to `HandRegistry` that participates in the lifecycle state machine
- A cross-method invariant is discovered (or fixed) — e.g., "install after uninstall should work," "activate with invalid config must not leave a zombie instance"
- The disk layout changes — e.g., a new file alongside `HAND.toml` and `SKILL.md`

Do **not** add tests here for:

- Error variant specifics — those belong in unit tests within `registry.rs`
- Parsing validation — covered by TOML/schema tests elsewhere
- Agent execution or LLM behavior — out of scope for registry smoke tests