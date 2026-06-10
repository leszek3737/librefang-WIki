# Other — librefang-channels-src

# librefang-channels-src: Channel Adapter Allowlist

## Overview

The `channels-allowlist.txt` file enforces LibreFang's **sidecar-first channel policy** at the build and CI level. It is not executable code — it is a policy ratchet consumed by a pre-commit hook and the `cargo xtask channel-policy` CI task.

## Purpose

LibreFang requires that all **new** channel adapters ship as out-of-process sidecar adapters rather than being compiled directly into the main binary. This allowlist grandfathers the in-process adapters that predate the policy and prevents new ones from being silently reintroduced.

The enforcement mechanism scans files under `crates/librefang-channels/src/` for the pattern `ChannelAdapter for`. If it finds that pattern in a file whose basename is **not** listed in `channels-allowlist.txt`, the check fails.

## Current Allowlist

| Basename (no `.rs`) | Status |
|---|---|
| `sidecar` | Active |

The list is kept in sorted order, one entry per line, without the `.rs` extension.

## How Enforcement Works

```mermaid
flowchart LR
    A[Developer commits code] --> B{Pre-commit hook / CI}
    B -->|Scans src/ for "ChannelAdapter for"| C{Basename in allowlist?}
    C -->|Yes| D[Pass]
    C -->|No| E[Fail: remove or migrate]
```

Two gates check this file:

1. **Pre-commit hook** — runs locally on every commit touching `crates/librefang-channels/src/`.
2. **`cargo xtask channel-policy`** — runs in CI on every pull request.

Both perform the same check: find any file matching `{<name>.rs, <name>/*.rs}` under `src/` that contains `ChannelAdapter for`, and verify `<name>` appears in `channels-allowlist.txt`.

## Known Limitations

The detection is text-based, not semantic. Specifically:

- **Macro-generated impls** are not detected. If a macro expands to `ChannelAdapter for SomeType`, the scanner will not catch it.
- **New adapters added inside an already-allowlisted file** are not detected. A developer could add a second adapter to `sidecar.rs` without triggering a failure.

This is intentional — the allowlist is a **policy ratchet**, not a security boundary. It prevents accidental growth of in-process adapters; it does not prevent deliberate circumvention.

## Maintenance Rules

### Shrinking Only

This list **only ever shrinks**. When an in-process adapter is migrated to a sidecar and its source module deleted, its basename must be removed from the allowlist. This ensures the adapter can never be reintroduced in-process without explicit maintainer approval.

### Adding an Entry Back

Re-adding a previously removed name is **not routine** and requires explicit maintainer sign-off. If you believe an adapter must return to in-process, open an issue explaining why the sidecar model is unsuitable.

### Adding a New Sidecar Adapter

New channel adapters should follow the sidecar pattern. See:

- `docs/` in the repository root
- `librefang.sidecar.adapters.*` modules in the SDK
- `examples/sidecar-channel-python/adapter.py` for a generic Python template

New sidecar adapters do **not** need to be added to this allowlist, because they live outside `crates/librefang-channels/src/` and are never scanned.