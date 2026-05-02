# Other — librefang-cli-tests

# librefang-cli/tests/build_rs_no_git_mutation

Regression test suite ensuring `build.rs` never mutates the user's Git configuration or invokes side-effecting Git subcommands. Introduced to prevent a recurrence of [issue #3641].

## Background

A previous `build.rs` script silently modified the user's global Git configuration (specifically `core.hooksPath`) during an ordinary `cargo build`. This is a hostile act against developer machines — build scripts must be read-only with respect to the user's environment. These tests statically analyze `build.rs` at test time to guarantee no regression.

## How It Works

Each test reads the source text of `build.rs`, strips line comments to avoid false positives from documentation mentioning the banned tokens, then asserts that forbidden substrings are absent.

```
┌─────────────────────────┐
│   read_build_rs()       │──→ reads build.rs from disk
└──────────┬──────────────┘
           ▼
┌─────────────────────────┐
│   strip_comments()      │──→ removes // line comments
└──────────┬──────────────┘
           ▼
┌──────────────────────────────────┐
│   assert!(!src.contains(bad))    │──→ fails if banned token found
└──────────────────────────────────┘
```

### `read_build_rs()`

Locates `build.rs` relative to `CARGO_MANIFEST_DIR` and reads it into a `String`. Panics if the file cannot be read — the file must exist for any meaningful test run.

### `strip_comments(src: &str) -> String`

Removes `//` line comments (everything from `//` to end-of-line) so that doc comments or inline notes referencing the old bug — for example `// Previously this script ran git config` — do not cause a false-positive test failure.

Stripping is deliberately simple: it looks for the first `//` on each line and truncates. It does not handle string literals or block comments, which is acceptable because `build.rs` is not expected to contain string literals with `//` that collide with the banned tokens.

## Test Cases

### `build_rs_does_not_mutate_git_config`

Checks two things:

| Banned token | Reason |
|---|---|
| `"config"` | Prevents any invocation of `git config` (even read-only). If a future change needs `git config --get`, an explicit carve-out must be added here rather than silently allowing it. |
| `"hooksPath"` | Specifically blocks modification of `core.hooksPath`, the exact mechanism of the original bug. |

The ban on the bare `"config"` token is intentionally broad. The comment in the test explains the rationale: if a legitimate read-only need arises, the test should be updated with a targeted allow-list rather than loosening the blanket prohibition.

### `build_rs_uses_only_read_only_git_subcommands`

Scans for a hardcoded list of side-effecting Git subcommands:

```
"init", "clone", "commit", "push", "pull",
"fetch", "checkout", "reset", "add", "rm"
```

If `build.rs` contains any of these as string literals (e.g., as arguments to `Command::new("git").arg("commit")`), the test fails. The check is substring-based against the post-comment-stripped source, so it catches tokens wherever they appear in the file.

## Extending the Guard

To add new banned operations, append entries to the `forbidden` array in `build_rs_uses_only_read_only_git_subcommands` or add a new `assert!` in `build_rs_does_not_mutate_git_config`.

If a legitimate read-only Git operation is needed in `build.rs` (e.g., `git config --get` to read a value), the test must be updated to:

1. Allow the specific safe pattern (e.g., check that `"config"` only appears alongside `"--get"`).
2. Document why the exception is safe.

Do **not** simply remove the assertion.

## Relationship to the Codebase

This module has no runtime dependencies beyond the standard library. It does not call into any other crate code, and nothing calls into it — it runs solely as part of `cargo test` in `librefang-cli`.

- **Monitors:** `build.rs` at the crate root.
- **Depended on by:** CI and local development workflows (via `cargo test`).
- **Related issue:** #3641.