# Other — librefang-cli-locales

# librefang-cli-locales

Localization module for the LibreFang CLI, providing all user-facing strings in Fluent (`.ftl`) format. This module contains no executable code — it is a pure resource directory consumed at runtime by the CLI's i18n layer (typically via the `fluent` / `fluent-templates` Rust crates).

## Structure

```
librefang-cli/locales/
├── en/
│   └── main.ftl      # English (default / fallback)
└── zh-CN/
    └── main.ftl      # Simplified Chinese
```

Each locale directory contains a single `main.ftl` file. The directory name is the BCP 47 language tag used for locale negotiation at runtime.

## Message Format

All messages use [Project Fluent](https://projectfluent.org/) syntax. Key patterns:

**Simple strings:**
```
daemon-starting = Starting daemon...
```

**Interpolated variables:**
```
models-available = { $count } models available
agent-killed = Agent { $id } killed.
```

**Multiline messages (used for fix instructions):**
```
shutdown-401-fallback-fix = Stop the daemon manually, then start it again:
    kill { $pid }    # or: kill -9 { $pid } if it does not exit
    librefang start
```

Every message identifier follows a `category-description` convention that doubles as a namespacing mechanism — the Fluent bundle is flat, so prefixes are the only organizational tool.

## Message Categories

The ~300 messages in each locale are organized into these sections:

| Prefix | Purpose | Typical consumers |
|---|---|---|
| `daemon-*` | Daemon lifecycle (start, stop, restart, background launch) | `start`, `stop`, `restart` subcommands |
| `shutdown-*` | Shutdown-specific flows, including the 401 fallback path (Issue #4693) | `stop`, `restart` subcommands |
| `label-*` | Short field labels for status tables and info displays | `status`, `info`, `doctor` subcommands |
| `hint-*` | User guidance printed after operations or in error states | Various subcommands |
| `guide-*` | Interactive setup wizard strings | `init` subcommand |
| `init-*` | Initialization success/cancellation messages | `init` subcommand |
| `error-*` | Error messages, often paired with a `*-fix` variant | Error handling paths throughout the CLI |
| `detected-*` | Provider auto-detection announcements | Boot sequence, `doctor` |
| `desktop-*` | Desktop app launch messages | `desktop` subcommand |
| `dashboard-*` | Dashboard open messages | `dashboard` subcommand |
| `agent-*` | Agent spawn, kill, model set, template errors | `agent` subcommand group |
| `manifest-*` | Agent manifest file errors | Agent template loading |
| `section-*` | Section headers for formatted output | `status`, `info`, `doctor`, `security` |
| `vault-*` | Credential vault operations including full key rotation flow | `vault` subcommand |
| `cron-*` | Cron job CRUD messages | `cron` subcommand |
| `approval-*` | Approval workflow responses | `approve` / `reject` subcommands |
| `memory-*` | Agent memory set/delete operations | `memory` subcommand |
| `device-*` | Device pairing and removal | `device` subcommand |
| `webhook-*` | Webhook CRUD and test messages | `webhook` subcommand |
| `model-*` | Model selection and catalog messages | `model` subcommand |
| `config-*` | Configuration get/set/unset errors and confirmations | `config` subcommand |
| `hand-*` | Hand (tool) dependency install, pause, resume | `hand` subcommand |
| `channel-*` | Channel setup flows (Discord, Slack, WhatsApp, etc.) | `channel setup` subcommand |
| `health-*` | Health check responses | `status`, health endpoints |
| `audit-*` | Audit trail integrity verification results | `security` subcommand |
| `value-*` | Security mechanism values displayed in status | `security` subcommand |
| `auth-*` | Authentication mode labels | `status` subcommand |
| `warn-*` | Config warning labels (public bind, missing key) | `status` subcommand |
| `uninstall-*` | Uninstall flow messages across platforms | `uninstall` subcommand |
| `reset-*` | Reset/remove data messages | `reset` subcommand |
| `log-*` | Log tailing messages | `logs` subcommand |

## Error / Fix Pairing Convention

Errors that have actionable next steps are split into two messages: the error itself and a `*-fix` companion. This lets the CLI print them together or separately depending on context (e.g., machine-readable JSON output might omit the fix).

```
error-boot-config = Failed to parse configuration
error-boot-config-fix = Check your config.toml syntax: librefang config show
```

The fix message is always printed on a separate line below the error, typically styled differently (dimmed, indented, or with a `→` prefix, depending on the CLI's output formatter).

## Adding a New Locale

1. Create `librefang-cli/locales/<lang-tag>/main.ftl` (e.g., `ja/main.ftl`).
2. Copy `en/main.ftl` as the starting point.
3. Translate every message value. **Do not change message identifiers.**
4. Preserve all `{ $variable }` interpolations exactly — they are filled by the Rust code.
5. Preserve multiline indentation (4 spaces for continuation lines in Fluent).
6. Keep inline comments starting with `#` as translation aids for context.

To verify completeness, diff the message identifiers between the new locale and `en/main.ftl`. Missing identifiers will silently fall back to English at runtime, but complete coverage is preferred.

## Adding a New Message

1. Add the identifier and English text to `en/main.ftl` under the appropriate section comment.
2. Add the same identifier with a translated value to every other locale's `main.ftl`.
3. Reference the message from Rust code using the CLI's fluent helper (typically something like `fl!("message-id", variable = value)`).

When adding an error with a suggested fix, create both `error-foo` and `error-foo-fix` in the same section.

## Variable Reference

Variables are not declared in Fluent — they are passed from the calling Rust code. The following variables appear across this module:

| Variable | Used in | Meaning |
|---|---|---|
| `$count` | `models-available`, `agents-loaded`, `auth-user-keys`, `vault-rotate-success` | Numeric count |
| `$provider` | `kernel-booted`, `detected-provider` | LLM provider name |
| `$model` | `kernel-booted`, `model-set-success` | Model identifier |
| `$name` | `agent-spawned`, `channel-configured`, `channel-unknown` | Entity name |
| `$id` | `agent-killed`, `cron-*`, `webhook-*`, `device-*`, `approval-*`, `hand-*` | Entity identifier |
| `$url` | `daemon-already-running`, `hint-dashboard-url`, `dashboard-opening` | URL |
| `$error` | `daemon-error`, `*-failed` messages | Error description |
| `$status` | `daemon-bg-exited`, `shutdown-request-fail`, `error-daemon-returned` | HTTP status or exit code |
| `$path` | `daemon-bg-exited-fix`, `error-create-dir`, `log-*`, `uninstall-*`, `reset-*` | Filesystem path |
| `$pid` | `shutdown-401-*` | Process ID |
| `$field` | `agent-unknown-field` | Config field name |
| `$value` | `agent-model-set`, `config-set-kv` | Scalar value |
| `$key` | `vault-*`, `config-*`, `channel-key-saved` | Key name |
| `$agent` | `memory-*` | Agent name |
| `$command` | `error-require-daemon` | CLI subcommand name |
| `$action` | `cron-toggled`, `approval-*` | Past-tense action verb |
| `$display` | `detected-provider` | Provider display name |
| `$env_var` | `detected-provider`, `config-saved-key`, `config-removed-env`, `config-env-not-set` | Environment variable name |
| `$max` | `model-out-of-range` | Upper bound of selection range |

## Notes on the Issue #4693 Block

The `shutdown-401-*` messages handle a specific upgrade edge case: when the CLI binary is upgraded in-place (e.g., `curl install.sh | sh`) without restarting the daemon, the new CLI cannot authenticate against the old daemon because the API key may have changed or the vault holding it cannot be unlocked. The messages explain the situation and guide the user through a PID-based fallback. The Chinese translation includes a supplementary comment explaining the root cause for translators who may not be familiar with the bug context.