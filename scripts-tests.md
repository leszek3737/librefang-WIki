# scripts — tests

# scripts/tests

Regression, smoke, and corpus tests for hooks, installers, and code-generation scripts that live outside the Rust crate graph. These tests guard behavior that Cargo's test runner cannot reach: shell hooks executed by Git, an installer sourced via `curl|sh`, and Python utility scripts invoked from CI or developer machines.

## What lives here

| File | Target under test | Language |
|---|---|---|
| `channel_progress_smoke.sh` | Daemon end-to-end tool-execution progress markers | Bash |
| `commit-msg-attribution.sh` | `scripts/hooks/commit-msg` — AI attribution guard | Bash |
| `install_sh_test.sh` | `web/public/install.sh` — installer logic | POSIX sh |
| `pre-commit-sha-fallback.sh` | `scripts/hooks/pre-commit` — openapi SHA baseline sync | Bash |
| `pre-commit-spaces.sh` | `scripts/hooks/pre-commit` — rustfmt path quoting | Bash |
| `test_check_changelog_attribution.py` | `scripts/check-changelog-attribution.py` | Python |
| `test_codegen_sdks.py` | `scripts/codegen-sdks.py` | Python |

## Hook regression tests

### commit-msg-attribution.sh

A corpus-driven test for the AI-attribution guard in the commit-msg hook. It exercises two independent checks:

1. **Message-body regex** — rejects commits whose message contains attribution strings (`🤖 Claude Code`, `Co-Authored-By: …`, `Generated with Claude`). The corpus includes zero-space (`ClaudeCode`), single-space, double-space, and mixed-case variants to lock in the `[[:space:]]*` quantifier and `-i` flag that a prior bug weakened.

2. **Author-identity check** — rejects commits authored under a known AI identity even when the message body is clean. Covers family-then-version (`Claude 3 Opus`) and version-then-family (`Claude 3.5 Sonnet`, `claude-3-5-sonnet`) naming, plus the `noreply@anthropic.com` mailbox. Accepts humans who happen to have an `anthropic.com` address or names that merely start with the same letters (`Claudia`, `Claudio`).

Two helper functions drive the corpus:

- `run <message>` — writes the message to a temp file, forces a human author identity, and invokes the hook. Return code indicates accept/reject.
- `run_as <name> <email>` — writes a clean message and varies the author identity, isolating the author check from the message regex.

### pre-commit-sha-fallback.sh

Regression test for issue #5664: the pre-commit hook's openapi SHA baseline sync silently skipped on Linux dev boxes where only `sha256sum` exists (no `shasum`). The test builds a throwaway git repo with the `openapi.json` + `xtask/baselines/openapi.sha256` layout, stages a modified spec, masks `shasum` out of PATH via a shim that exits 127, then asserts:

- The hook exits 0 under a `sha256sum`-only PATH.
- The baseline file contains the correct digest of the staged spec.
- The refreshed baseline was auto-staged by the hook.

Copies the repo's `.gitleaks.toml` into the throwaway repo so the secret-scan step doesn't fail on a missing config.

### pre-commit-spaces.sh

Regression test for issue #5664: the pre-commit rustfmt pipeline word-split unquoted `$STAGED_RS`, so a file named `with space.rs` was treated as two separate paths. The test stages a deliberately mis-formatted `with space.rs`, runs the hook, and asserts:

- The hook exits non-zero (rejects the commit).
- The rejection came through the rustfmt path (`grep -q "not rustfmt-clean"`).

Skips cleanly when `rustfmt` is not installed.

## Installer test suite

### install_sh_test.sh

A POSIX-sh test suite for `web/public/install.sh`, sourced with `LIBREFANG_INSTALLER_SOURCE_ONLY=1` so the installer's functions are available without executing the install. Covers:

**Shell RC detection**
- `shell_rc_from_shell` mappings for zsh, bash, fish.
- `choose_shell_rc` falls back to `$SHELL` when `detect_user_shell` returns empty (the `curl|sh` scenario).
- File-existence fallback order: `.zshrc` → `.bashrc` → fish config.
- The "already installed" check matches `\.librefang/bin` precisely, not any line mentioning "librefang" (which would false-positive for a user named `librefang`).

**Flag parsing and session state**
- `is_enabled` accepts `1/true/yes/on` (case-insensitive) and rejects `0/false/no/off/""`.
- `SESSION_NEEDS_PATH_REFRESH` correctly detects whether the install dir is in PATH.
- `RESTART_SHELL` prefers `$SHELL`, falls back to `USER_SHELL`.

**Parent-shell detection**
Mocks `ps` via a stateful fake binary that returns `sh` on the first `comm` query and `zsh` on the second, verifying `detect_user_shell` walks the process tree to find the real parent shell.

**Distro detection (`detect_distro`)**
Fixture-driven through `OS_RELEASE_FILE` and `NIXOS_MARKER_FILE` globals:

| Fixture | Asserts |
|---|---|
| `os-release-nixos` | `DISTRO=nixos`, no `DISTRO_LIKE`, not debian family |
| `os-release-deepin` (`ID=Deepin`) | Lowercased to `deepin`, `DISTRO_LIKE=debian`, debian family |
| `os-release-deepin-bare` | `deepin` matched by own ID when `ID_LIKE` absent |
| `os-release-ubuntu` | `DISTRO=ubuntu`, `DISTRO_LIKE=debian` |
| `os-release-derivative` (`ID_LIKE="ubuntu debian"`) | Multi-entry `ID_LIKE` unquoted and matched |
| `os-release-inherited-debian` + NixOS marker | NixOS marker outranks inherited `ID=debian` |
| Missing os-release | Degrades to `unknown`, never fails |

**Platform fallback policy (NixOS)**
NixOS has no `/lib64/ld-linux-x86-64.so.2`, so the glibc (`gnu`) fallback must be suppressed before download, not discovered after a failed `--version` check.

- `effective_platform_fallback` returns empty on NixOS, keeps `gnu` on Ubuntu.
- `apply_distro_platform_policy` emits a message naming NixOS and the missing interpreter; stays silent off NixOS.
- `should_try_platform_fallback` covers all three retry states: worth trying, already held, no usable fallback.
- **Order independence**: running `detect_distro` → `detect_platform` → `apply_distro_platform_policy` vs. `detect_distro` → `apply_distro_platform_policy` → `detect_platform` yields the same fallback decision. The NixOS rule must not depend on call order.

**Version resolution (`resolve_installable_version`)**
Uses a mock `curl` driven by `MOCK_TAGS`, `MOCK_GOOD_TAGS`, and `MOCK_BAD_PLATFORM`:

- Skips a stuck newest release (no assets) and falls back to an older one.
- Falls back across platform variants (musl → gnu) within a release.
- Refuses the gnu build on NixOS.
- Honors `LIBREFANG_VERSION` as a hard pin (no asset probe).
- Treats `LIBREFANG_PREFERRED_VERSION` as a soft hint (falls back if stuck).
- Fails when no release ships an installable package.

The mock curl enforces that tarball probes use a 1-byte range request (`-r 0-0`), failing loudly if the optimization regresses.

**Binary rollback (`install_binary_with_rollback`)**
- Working upgrade: installs new binary, removes backup.
- Broken upgrade: rolls back to previous binary, cleans up backup.
- Broken fresh install (no existing binary): removes the unrunnable binary entirely.

**Desktop hint (`print_debian_desktop_hint`)**
Mocks `pkg-config` and `apt-cache` to verify the hint never prescribes a package the repositories don't carry, names the correct webkit series when probed, and stays silent on headless or non-Debian hosts.

**Source install hint (`print_source_install_hint`)**
NixOS gets the flake URL (`nix profile install`, `nix run`, `services.librefang.enable`); all other distros get the `cargo install` fallback.

## Python script tests

### test_check_changelog_attribution.py

Tests `scripts/check-changelog-attribution.py` by importing it via `importlib` (no package install required). Two test groups:

**`bullet_block_has_attribution`** — verifies that `(@user)` attribution is recognized anywhere in a bullet's continuation block, not just on the `- ` marker line. This matches the one-sentence-per-line prose wrapping where a long bullet carries its attribution on the final continuation line. Also tests:

- Attribution does not leak across blank lines or adjacent bullets.
- `# pragma: no-attribution` on a continuation line exempts the bullet.

**`fragment_tests`** — exercises `check_fragment` and `classify_fragment_path` against `changelog.d/` entries:

- Well-formed fragments in recognized sections (`fixed/`, `added/`, etc.) pass.
- Missing attribution is flagged at line 1 with `MISSING_ATTRIBUTION`.
- Attribution stranded after a blank line does not count.
- Empty fragments are rejected.
- Unrecognized section directories (`fix/` instead of `fixed/`) are rejected even with attribution, because assembly has no heading for them.
- Infrastructure files (`README.md`, `.gitkeep`, `.txt`) are never scanned.

### test_codegen_sdks.py

Smoke test for `scripts/codegen-sdks.py` against the real `openapi.json`. Asserts invariants that have historically regressed:

- **Operation loading** — `invoke_tool` has `agent_id` in `query_params` and `has_body=True`; `list_agents` has the full query param set; `send_message_stream` is detected as a stream operation.
- **SDK signatures** — `invoke_tool` signatures include the query/agent_id parameter across all four SDKs (Python, JS, Go, Rust).
- **Stream correctness** — Go uses `bufio.NewReaderSize` (not bare `strings.Split`); Rust uses `Vec<u8>` (not `from_utf8_lossy` on chunks); both surface HTTP status in error events.
- **SSE line-size cap** — `MAX_SSE_LINE` / `maxSSELine` present in Rust/Go output.
- **Reserved-word escaping** — `_py_safe("class")` → `"class_"`, `_rust_safe("type")` → `"type_"`.

## Smoke test

### channel_progress_smoke.sh

A live end-to-end integration test that verifies tool-execution progress markers (`🔧 Web Search`) surface through the daemon. Unlike the other tests, this one requires external prerequisites and does not run in CI:

- An LLM API key in env (`GROQ_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, or `MINIMAX_API_KEY`).
- At least one channel adapter configured in `~/.librefang/config.toml`.
- A release binary at `target/release/librefang`.

The script starts a daemon, reuses an existing enabled agent or spawns an ephemeral one (with `web_search` tool capability) via the API, sends a message likely to trigger a tool call, and checks the session log for `tool_use` events. Full channel-delivery verification requires a configured webhook receiver and is documented externally.

Cleanup runs on EXIT via `trap`, stopping the daemon and deleting any temporary agent that was spawned.

## Running the tests

```sh
# Shell tests — run from repo root
bash scripts/tests/commit-msg-attribution.sh
bash scripts/tests/pre-commit-sha-fallback.sh
bash scripts/tests/pre-commit-spaces.sh
sh scripts/tests/install_sh_test.sh

# Python tests — run directly
python3 scripts/tests/test_check_changelog_attribution.py
python3 scripts/tests/test_codegen_sdks.py

# Smoke test (requires live LLM + daemon)
bash scripts/tests/channel_progress_smoke.sh
```

The hook regression tests skip cleanly when their dependency (`sha256sum`, `rustfmt`) is missing, exiting 0 with a `SKIP:` message. The installer suite and Python tests have no optional-skip paths — they run to completion or fail.