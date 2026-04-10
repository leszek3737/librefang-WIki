# Build System — rustfmt.toml

# Build System — rustfmt.toml

Configuration file enforcing consistent Rust code formatting across the LibreFang workspace.

## Overview

This file defines the **rustfmt** settings that are applied to all Rust source code in the LibreFang project. The configuration is version-controlled alongside the source code and enforced automatically by CI to prevent formatting inconsistencies from entering the codebase.

## Configuration Options

| Option | Value | Purpose |
|--------|-------|---------|
| `edition` | `"2021"` | Specifies Rust edition 2021 for all crates in the workspace |
| `max_width` | `100` | Maximum line length before rustfmt wraps code |
| `use_field_init_shorthand` | `true` | Allows `Point { x, y }` instead of `Point { x: x, y: y }` |
| `use_try_shorthand` | `true` | Allows `?` operator instead of `try!()` macro |

### edition

```toml
edition = "2021"
```

Sets the Rust language edition to 2021 for the entire workspace. This enables modern syntax features and ensures compatibility with current Rust tooling.

### max_width

```toml
max_width = 100
```

Defines the maximum character width for lines. Code exceeding this limit is automatically wrapped. The 100-character limit is a common balance between readability and preventing excessive line breaks on wide monitors.

### use_field_init_shorthand

```toml
use_field_init_shorthand = true
```

When enabled, struct initialization can use shorthand syntax when the field name matches the variable name:

```rust
// Without shorthand (disabled)
let point = Point { x: x, y: y };

// With shorthand (enabled)
let point = Point { x, y };
```

### use_try_shorthand

```toml
use_try_shorthand = true
```

Allows the `?` operator instead of the older `try!()` macro syntax:

```rust
// Without shorthand (disabled)
let value = try!(some_result);

// With shorthand (enabled)
let value = some_result?;
```

## CI Integration

The configuration file header documents the CI enforcement:

```toml
# Enforced by CI — run `cargo fmt` before pushing.
```

Before pushing changes, developers should run:

```bash
cargo fmt
```

This reformats staged files to match the workspace configuration. CI will reject pull requests containing formatting deviations.

## Developer Workflow

```
┌─────────────────────────────────────────────┐
│            Local Development                 │
├─────────────────────────────────────────────┤
│                                             │
│  1. Edit Rust source files                  │
│  2. Run `cargo fmt` to auto-format          │
│  3. Review changes (formatted code only)    │
│  4. Commit and push                         │
│                                             │
└─────────────────┬───────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────┐
│                 CI Pipeline                  │
├─────────────────────────────────────────────┤
│                                             │
│  cargo fmt --check                          │
│  → Fails if unformatted code detected       │
│  → Passes only if code matches rustfmt.toml │
│                                             │
└─────────────────────────────────────────────┘
```

## Maintenance Notes

- **No external dependencies** — rustfmt ships with Rust, requiring no additional tooling
- **Workspace-level configuration** — Placed at the workspace root, applies to all crates
- **Minimal settings** — Only essential options are specified; rustfmt uses defaults for everything else
- **Single source of truth** — One file controls formatting for the entire codebase