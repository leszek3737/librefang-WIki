# Build System — rustfmt.toml

# Build System — `rustfmt.toml`

## Purpose

This file defines the workspace-wide code formatting policy for LibreFang. It is consumed automatically by `cargo fmt` (which invokes `rustfmt` under the hood) and is enforced as a CI gate — any PR that does not conform to these rules will fail CI checks.

## When It Runs

- **Locally:** Developers run `cargo fmt` manually (or via an editor save hook) before committing.
- **CI:** The continuous integration pipeline runs `cargo fmt -- --check` to verify that all committed source matches the configured style. A non-zero exit status fails the build.

## Configuration Options

| Option | Value | Effect |
|---|---|---|
| `edition` | `"2021"` | Tells rustfmt to apply formatting rules consistent with Rust Edition 2021 syntax and idioms. |
| `max_width` | `100` | Maximum line width before rustfmt wraps expressions. The default is 100; repeated here for explicitness so future editors don't have to guess the project's intent. |
| `use_field_init_shorthand` | `true` | Rewrites `Foo { x: x }` as `Foo { x }` when field and variable names match, reducing visual noise in struct literals. |
| `use_try_shorthand` | `true` | Rewrites `try!()` macro calls to the `?` operator, keeping the codebase consistent with modern Rust error-propagation style. |

## How It Connects to the Rest of the Codebase

This file lives at the workspace root and applies uniformly to **every crate and module** in the LibreFang workspace. There are no per-crate overrides — a single formatting standard covers the entire project.

No other modules call into or depend on this file at runtime; its influence is purely at the tooling and CI layer.

## Developer Workflow

1. Make code changes.
2. Run `cargo fmt` before staging commits.
3. Verify with `cargo fmt -- --check` if you want to catch drift early.
4. Push — CI will re-validate.