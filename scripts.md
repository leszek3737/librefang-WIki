# scripts

# scripts/

Repository automation: Git hooks, CI lint gates, release tooling, SDK code generation, and install-serving infrastructure. Every script is designed to run identically on a contributor's laptop and in CI.

## Sub-modules at a glance

| Sub-module | Scope |
|---|---|
| [scripts](scripts.md) | Top-level Python and shell scripts — changelog attribution checks, architectural invariant enforcement, SDK codegen, release article scaffolding |
| [docker](docker.md) | Dockerfile that smoke-tests the user-facing installer in a clean container |
| [hooks](hooks.md) | Git hooks (`pre-commit`, `pre-push`, `commit-msg`) wired via `core.hooksPath` |
| [tests](tests.md) | Bash/POSIX regression tests for hooks, installer logic, and daemon progress markers |
| [workers](workers.md) | Cloudflare Pages Functions that redirect `/install.sh` and `/install.ps1` to the latest GitHub release asset |

## How they fit together

### Commit lifecycle

The [hooks](hooks.md) module gates every commit and push locally. `pre-commit` stays fast (sub-2 s) by running only staged-diff-scoped checks; heavier validation deferred to `pre-push` or CI. The `commit-msg` hook enforces AI-attribution rules, and `check-changelog-attribution.py` (in the [root scripts](scripts.md)) validates `(@user)` attribution on CHANGELOG bullets and `changelog.d/` fragments — it runs both inside the hook path (on staged fragments) and in CI (on the full file).

The [tests](tests.md) module closes the loop: `commit-msg-attribution.sh` exercises the `commit-msg` hook, and `pre-commit-sha-fallback.sh` validates the `pre-commit` hook's SHA-baseline sync logic. These tests exist because Git-invoked shell scripts are invisible to Cargo's test runner.

### Installer and release pipeline

Three sub-modules cooperate to keep the installer trustworthy end-to-end:

1. **[workers](workers.md)** serves friendly URLs (`/install.sh`, `/install.ps1`) by fetching the latest release metadata from GitHub and redirecting to the matching platform asset.
2. **[docker](docker.md)** smoke-tests `web/public/install.sh` itself — syntax validation and platform detection by default, optional full binary install — acting as the CI gate on installer changes.
3. **[tests](tests.md)** provides `install_sh_test.sh`, a POSIX sh suite that exercises installer logic without Docker.

On the release side, `changelog-to-article.sh` (root) scaffolds a release article from a CHANGELOG section, complementing the attribution checks that already validated the changelog content.

### Code generation

`codegen-sdks.py` (root) generates JavaScript, Python, and Rust SDK bindings from the API kernel. It is driven by `_tag_pascal` / `_tag_attr` helpers for consistent naming across all three targets. Changes to generated output are checked for drift in `pre-push` and CI.

## Shared conventions

- **No source-code reading required at runtime** — hooks and checks operate on Git diffs, staged content, or API metadata, keeping them portable across environments.
- **Attribution is mandatory** — every CHANGELOG bullet, changelog fragment, and AI-assisted commit must carry `(@user)` attribution, enforced at hook-time and CI-time by the same underlying logic (`has_attribution`, `bullet_block_has_attribution`).
- **Tests live next to what they test** — when behavior can't be reached by Cargo, a shell test in [tests](tests.md) covers it instead.