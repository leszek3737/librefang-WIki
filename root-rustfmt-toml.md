# Root — rustfmt.toml

# Root — `rustfmt.toml`

Workspace-level formatting configuration for the LibreFang Rust project. This file has no runtime behavior; it governs how `cargo fmt` and `rustfmt` rewrite source across **every** crate in the workspace.

## Purpose

Consistent formatting is enforced through CI, and this configuration is the single source of truth for those rules. Every contribution is expected to pass `cargo fmt --check` before merge, and the settings below define exactly what "passing" means.

The file lives at the workspace root so it applies automatically to all member crates without per-crate duplication.

## Settings

| Setting | Value | Effect |
|---|---|---|
| `edition` | `2021` | Formats code assuming Rust 2021 edition semantics (e.g., prelude changes, closure capture rules). |
| `max_width` | `100` | Lines are wrapped to 100 characters. This is the primary knob for visual layout. |
| `use_field_init_shorthand` | `true` | Rewrites `Foo { x: x }` as `Foo { x }`. |
| `use_try_shorthand` | `true` | Rewrites `match { Ok(v) => v, Err(e) => return Err(e) }` idioms as `?`. |

Anything **not** listed here falls back to `rustfmt` defaults.

## How to Use

Before pushing changes, run:

```sh
cargo fmt
```

This rewrites files in place across all workspace members to conform to the rules above. To verify formatting without modifying files — mirroring what CI checks — run:

```sh
cargo fmt --check
```

If the check exits non-zero, CI will fail on the pull request.

## CI Enforcement

The header comment states this is "Enforced by CI." Concretely:

- CI runs `cargo fmt --check` (or equivalent) as a gate.
- Any formatting drift blocks the build until `cargo fmt` is applied and the changes committed.
- Because the config is at workspace root, there is no ambiguity about which rules apply to which crate.

## Interaction with the Rest of the Workspace

This file is purely a formatting contract. It does not:

- Affect compilation, type-checking, or runtime behavior.
- Introduce dependencies or feature flags.
- Get consumed by any code path (the call graph has no incoming, outgoing, or internal edges).

It is, however, coupled to the toolchain: the CI runner's `rustfmt` version determines how unrecognized options or version-specific formatting behave. Contributors should ensure their local nightly/stable `rustfmt` matches what CI uses, since formatting output can differ slightly between toolchain versions.

## When to Change This File

Update this file when you want to change a workspace-wide style convention (e.g., adjusting `max_width` or enabling additional options like `imports_granularity`). Because it applies to every crate simultaneously, such changes typically produce a large diff across the repository and should be coordinated as a dedicated formatting commit rather than mixed into feature work.