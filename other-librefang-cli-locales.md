# Other — librefang-cli-locales

# librefang-cli-locales

Localization resources for the LibreFang CLI. Contains user-facing strings in [Project Fluent](https://projectfluent.org/) (`.ftl`) format, organized by locale and functional domain.

## Directory Layout

```
librefang-cli/locales/
├── en/           # English — primary locale, source of truth
│   └── main.ftl
└── zh-CN/        # Simplified Chinese
    └── main.ftl
```

Each locale directory mirrors the same message IDs. The `en` locale is the canonical source; all other locales must provide translations for every message key defined there.

## Role in the Codebase

This module is a **pure resource module** — no Rust code, no logic. The CLI binary loads these `.ftl` files at runtime via a Fluent integration layer (typically `fluent-bundle` or a thin wrapper). CLI command handlers request localized strings by message ID, optionally passing variables, and the framework resolves the correct locale file based on the user's language preference.

Because it contains no executable code, it has no call graph edges of its own.

## Message Organization

Messages are grouped into named sections via `# ---` comments. The section boundaries are not structural to Fluent itself — they exist for human readability. The major sections are:

| Section | Covers |
|---|---|
| Daemon lifecycle | Start, stop, restart, background launch, health polling |
| Labels | Short nouns for table headers and status displays |
| Hints | Contextual guidance shown after commands or errors |
| Init | First-run setup wizard outcomes |
| Error messages | Directory/config failures |
| Daemon communication errors | Timeouts, refused connections, generic comm failures |
| Boot errors | Config parse, database lock, auth failures during kernel boot |
| Provider detection | Auto-detected LLM provider announcements |
| Desktop app | Native desktop app launch messages |
| Dashboard | Browser-based dashboard opening |
| Agent commands | Spawn, kill, template listing, field mutation |
| Manifest errors | Missing or corrupt agent manifests |
| Status | Daemon status display, security info, auth descriptions |
| Doctor | Diagnostic check results and repair outcomes |
| Security | Audit trail, taint tracking, sandbox, wire protocol labels |
| Channel setup | Discord, Slack, WhatsApp, Email, Signal, Matrix configuration |
| Vault | Credential vault init, store, remove, master key rotation |
| Cron / Approvals / Memory / Devices / Webhooks | CRUD feedback for respective subsystems |
| Models | Model catalog selection and default model changes |
| Config | Get/set/unset operations, key management, env persistence |
| Hand commands | Dependency installation, pause/resume |
| Uninstall | Teardown across platforms (Windows, macOS, Linux) |
| Reset | Data removal feedback |
| Logs | File tailing hints |

## Naming Conventions

Messages follow a consistent naming scheme:

- **Domain prefix + description**: `daemon-starting`, `vault-stored`, `config-set-success`
- **Error + fix pairs**: Many errors have a companion message suffixed `-fix` that provides remediation steps:

```ftl
error-boot-db = Database error (file may be locked)
error-boot-db-fix = Check if another LibreFang process is running: librefang status
```

CLI handlers reference both IDs — the error message and its fix — so the user sees the problem and an actionable next step together.

- **Section headings** use the `section-` prefix: `section-daemon-status`, `section-channel-setup`
- **Labels** use the `label-` prefix: `label-uptime`, `label-pid`
- **Hints** use the `hint-` prefix: `hint-stop-daemon`, `hint-check-status`
- **Warnings** use the `warn-` prefix: `warn-public-bind`, `warn-key-missing`

## Variable Interpolation

Fluent uses `{ $name }` for runtime variable substitution. Variables used across this module include:

| Variable | Typical source |
|---|---|
| `$error` | Error message from the underlying subsystem |
| `$status` | HTTP status code from the daemon |
| `$path` | Filesystem path (log file, PID file, config, etc.) |
| `$url` | Daemon or dashboard URL |
| `$name` | Agent or channel name |
| `$id` | Agent, cron job, webhook, or device identifier |
| `$count` | Numeric count (agents, keys, entries) |
| `$provider` / `$model` | LLM provider and model identifiers |
| `$command` | CLI subcommand name |
| `$key` / `$value` / `$env_var` | Config key-value pairs and environment variable names |
| `$pid` | OS process ID |
| `$action` | Past-tense verb for approval/cron toggles (e.g., "pause", "resume") |

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g., `ja` for Japanese, `pt-BR` for Brazilian Portuguese).
2. Copy `en/main.ftl` into the new directory.
3. Translate every message value while preserving:
   - The message ID (left of `=`) exactly as-is.
   - All `{ $variable }` interpolations.
   - Multiline formatting (indented continuation lines), especially in messages like `shutdown-401-fallback-fix`.
4. Register the new locale in the CLI's Fluent bundle initialization code (outside this module).

## Adding a New Message

1. Add the message to `en/main.ftl` under the appropriate section comment.
2. If the message is an error, add a companion `-fix` message with remediation guidance.
3. Add the same message ID with a translated value to every other locale's `main.ftl`.
4. Reference the new message ID from the CLI handler code via the Fluent bundle.

## Notable Design Decisions

**Issue #4693 — 401 shutdown fallback**: The `shutdown-401-*` message cluster handles a specific upgrade scenario where the CLI binary is replaced (via `curl install.sh | sh`) without restarting the running daemon. The new CLI's auth credentials don't match the old daemon's, producing a 401 on `/api/shutdown`. The strings walk the user through understanding the cause, attempting a PID-based fallback, and providing manual `kill` instructions if the fallback also fails. This is a rare example of a localization key being tied to a specific issue number in comments.

**Non-interactive detection**: Messages like `hint-non-interactive` and `hint-non-interactive-wizard` support headless/CI environments where the CLI auto-selects a quick mode and directs users to a terminal for the full interactive wizard.