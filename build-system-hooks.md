# Build System — hooks


# Build System — hooks

A collection of Git hooks that enforce code quality standards at commit time. Currently provides automated Rust code formatting checks.

## Purpose

This module ensures consistent Rust code formatting across the codebase without requiring developers to manually remember to run formatting commands. The pre-commit hook catches formatting issues before they enter the version control history, preventing style debates in code reviews and keeping diffs focused on logic changes rather than whitespace or style differences.

## How It Works

### Pre-commit Hook

Located at `scripts/hooks/pre-commit`, this hook executes automatically before each `git commit` completes.

**Execution flow:**

1. Git triggers the hook before finalizing a commit
2. The hook runs `cargo fmt --check` to compare current code against Rust's default formatting rules
3. If the check passes (exit code 0), the commit proceeds normally
4. If the check fails (non-zero exit code):
   - The hook runs `cargo fmt` to auto-format the code
   - Prints instructions asking the developer to re-stage the formatted files
   - Exits with code 1 to abort the current commit

The hook intentionally aborts the commit rather than committing poorly formatted code. Developers must stage the auto-formatted changes and commit again—this reinforces the habit of reviewing formatting changes before committing.

## Installation

The repository uses Git's `core.hooksPath` configuration to point to this directory:

```bash
git config core.hooksPath scripts/hooks
```

This is a repository-level setting, so the hooks apply to all developers cloning the repository without any additional setup. The `core.hooksPath` approach is preferred over `.git/hooks/` symlinks because it works reliably in CI/CD environments and doesn't require per-developer configuration.

## Integration with the Build System

The hooks module integrates with the broader build system through Cargo commands:

| Command | Purpose |
|---------|---------|
| `cargo fmt --check` | Reads `rustfmt.toml` and applies Rust's default formatting rules to detect deviations |
| `cargo fmt` | Auto-corrects formatting deviations according to the same rules |

Both commands operate on the entire Rust codebase. The formatting configuration lives in `rustfmt.toml` at the repository root—if present, it customizes the formatting rules; if absent, Rust's defaults apply.

## Extending the Hooks

To add additional checks (such as linting, tests, or other quality gates), modify `scripts/hooks/pre-commit` to run those checks before the final exit. Chain checks so that a failure in any check aborts the commit:

```sh
#!/bin/sh
# Example extension pattern
cargo fmt --check 2>/dev/null || { cargo fmt; echo "Formatted. Re-stage and commit."; exit 1; }

# Add more checks here
# npm run lint || { echo "Lint failed"; exit 1; }
# cargo test || { echo "Tests failed"; exit 1; }
```

Each check should exit with non-zero to block the commit on failure.