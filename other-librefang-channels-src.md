# Other — librefang-channels-src

# librefang-channels-src — Channel Adapter Allowlist

## Purpose

LibreFang enforces a **sidecar-first** architecture for channel adapters. All new channel adapters must ship as out-of-process sidecar adapters, not as in-process implementations compiled into the main binary.

This module contains `channels-allowlist.txt`, a policy ratchet that grandfathers the in-process adapters that predate the sidecar mandate. It exists to prevent new in-process adapters from being silently reintroduced.

## How It Works

The allowlist is a plain-text file listing Rust source basenames (without `.rs`) that are permitted to contain `ChannelAdapter for` implementations. The current entries:

```
sidecar
```

The enforcement tooling scans every file under `crates/librefang-channels/src/{<name>.rs, <name>/*.rs}` and checks whether it contains the token sequence `ChannelAdapter for`. If it does, the file's basename **must** appear in the allowlist. If it doesn't, the check fails.

## Enforcement

Two mechanisms enforce the allowlist:

| Mechanism | When it runs | How to invoke manually |
|---|---|---|
| Pre-commit hook | On every `git commit` | Configured in `.git/hooks/` |
| `cargo xtask channel-policy` | On every PR in CI | Local: `cargo xtask channel-policy` |

If either detects a `ChannelAdapter for` impl in a file not listed in the allowlist, the operation is rejected.

## Policy Rules

### The list only ever shrinks

When an adapter is migrated to a sidecar and its source module deleted, its basename is removed from the allowlist. This ensures it can never be reintroduced as an in-process adapter.

### Adding a name back requires explicit maintainer approval

Reverting a removal is not routine. Any addition to this file is a deliberate architectural decision that must be reviewed and approved.

## Known Limitations

The enforcement is a **grep-based heuristic**, not a semantic analysis:

- **Macro-generated impls** are not detected. If a `ChannelAdapter for` implementation is produced by a macro expansion, the scanner won't catch it.
- **New adapters inside allowlisted files** are not detected. Adding a second `ChannelAdapter for` impl within an already-allowlisted file passes the check without complaint.

This is intentional. The file is a **policy ratchet**, not a security boundary. It prevents accidental growth of in-process adapters, not malicious circumvention.

## File Format

- One basename per line.
- No `.rs` extension.
- Keep entries sorted alphabetically.
- Lines beginning with `#` are comments.

## Integration Points

- **Sidecar adapter template**: `librefang.sidecar.adapters.*` modules in the SDK and `examples/sidecar-channel-python/adapter.py` provide reference implementations for out-of-process adapters.
- **XTask**: `cargo xtask channel-policy` contains the scanning logic that reads this file and validates source tree compliance.