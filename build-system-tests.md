# Build System — tests

# install_sh_test.sh — Installer Script Tests

Tests the `install.sh` installer script by sourcing it in isolated environments and verifying behavior of its core functions.

## Overview

This test suite validates the installer script's ability to:
- Map shell names to their appropriate RC file paths
- Select the correct RC file when multiple shells are present
- Parse boolean configuration flags
- Detect parent shells even when invoked via `curl | sh`

## Test Environment Setup

Each test runs against a temporary home directory to avoid polluting the developer's actual shell configuration:

```sh
TMP_HOME=$(mktemp -d)
HOME="$TMP_HOME" LIBREFANG_INSTALLER_SOURCE_ONLY=1 . "$INSTALLER_PATH"
```

- `LIBREFANG_INSTALLER_SOURCE_ONLY=1` loads the script's functions without triggering interactive prompts
- The script is sourced (`.`) rather than executed, making functions directly callable

## Test Cases

### `shell_rc_from_shell` Mappings

Verifies that each supported shell maps to its correct RC file location:

| Shell | Expected RC File |
|-------|------------------|
| `zsh` | `$HOME/.zshrc` |
| `/bin/bash` | `$HOME/.bashrc` |
| `fish` | `$HOME/.config/fish/config.fish` |

```sh
[ "$(shell_rc_from_shell zsh)" = "$TMP_HOME/.zshrc" ] || fail "zsh rc mapping"
```

### `choose_shell_rc` Fallback Order

Tests the priority order when multiple RC files exist. The function should prefer shells in this order: **bash → zsh → fish**.

```sh
# All three present → bashrc wins
[ "$(choose_shell_rc "")" = "$TMP_HOME/.bashrc" ]

# bashrc missing → zshrc wins
[ "$(choose_shell_rc "")" = "$TMP_HOME/.zshrc" ]

# bashrc and zshrc missing → fish wins
[ "$(choose_shell_rc "")" = "$TMP_HOME/.config/fish/config.fish" ]
```

### `is_enabled` Flag Parser

Validates the function that parses `LIBREFANG_AUTO_START` and similar boolean flags.

**Accept as true:**
```
1, true, TRUE, yes, YES, on, ON
```

**Reject (must return false):**
```
0, false, FALSE, no, NO, off, OFF, "" (empty string)
```

```sh
for truthy in 1 true TRUE yes YES on ON; do
    is_enabled "$truthy" || fail "is_enabled should accept $truthy"
done
for falsy in 0 false FALSE no NO off OFF ""; do
    if is_enabled "$falsy"; then
        fail "is_enabled should reject $falsy"
    fi
done
```

### `detect_user_shell` Parent Shell Detection

Tests an edge case where `detect_user_shell` must look past an intermediate `sh` shell to find the actual parent (typically zsh when using `curl | sh`).

The test uses a fake `ps` binary that simulates this sequence:

1. First call to `ps -o comm=` returns `"sh"` (the pipe shell)
2. Query for parent PID returns `"222"`
3. Second call to `ps -o comm= -p 222` returns `"zsh"` (the actual shell)

```sh
# Fake ps outputs based on call count:
# Call 1: "sh"
# Call 2 (ppid query): "222"  
# Call 3: "zsh"
```

The detector must traverse this chain and return `"zsh"`:

```sh
DETECTED=$(HOME="$TMP_HOME" ... sh -c '. "$INSTALLER_PATH"; detect_user_shell')
[ "$DETECTED" = "zsh" ] || fail "detect_user_shell expected zsh, got: $DETECTED"
```

## Running the Tests

```sh
./scripts/tests/install_sh_test.sh
```

All tests must pass for the output to be:

```
PASS: shell_rc_from_shell mappings
PASS: choose_shell_rc fallback order
PASS: LIBREFANG_AUTO_START flag parser
PASS: detect_user_shell handles curl|sh parent shell
All install.sh tests passed.
```

## Failure Handling

The test script uses two helper functions:

```sh
fail() {
    echo "FAIL: $*" >&2
    exit 1
}

pass() {
    echo "PASS: $*"
}
```

Any assertion failure exits immediately with status 1, making the script suitable for CI integration.

## Relationship to Installer

This test file validates the non-interactive portions of `web/public/install.sh`. The tested functions handle:

- **Shell detection** — determining which shell the user runs
- **RC file selection** — choosing where to write initialization commands
- **Configuration parsing** — interpreting boolean settings
- **Parent shell traversal** — safely detecting the real shell across process boundaries

These are the building blocks that allow `install.sh` to add itself to the user's shell configuration regardless of which shell they use.