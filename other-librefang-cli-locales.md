# Other — librefang-cli-locales

# librefang-cli-locales

Localization resources for the LibreFang CLI, providing all user-facing text in English (`en`) and Simplified Chinese (`zh-CN`). The CLI loads these Fluent (`.ftl`) files at runtime via the [Project Fluent](https://projectfluent.org/) framework to resolve every human-readable string — from daemon status messages to error recovery hints — without hardcoding prose in command handlers.

## Directory layout

```
librefang-cli/locales/
├── en/
│   └── main.ftl      # English — the primary / fallback locale
└── zh-CN/
    └── main.ftl      # Simplified Chinese translation
```

Each subdirectory is named by its [Unicode locale identifier](https://unicode.org/reports/tr35/). `main.ftl` is the only file per locale; as the CLI grows, additional files (e.g. `errors.ftl`, `wizard.ftl`) can be added without changing the loading logic.

## Fluent syntax recap

Every entry follows Project Fluent's FTL syntax:

```ftl
# Simple message
daemon-started = Daemon started

# Message with a variable
models-available = { $count } models available

# Multiline message (indented continuation lines)
shutdown-401-fallback-fix = Stop the daemon manually, then start it again:
    kill { $pid }    # or: kill -9 { $pid } if it does not exit
    librefang start
```

- **Comments** begin with `#`. Section headers use `# --- Name ---` for readability but have no semantic meaning.
- **Variables** like `{ $provider }` are injected at runtime by the calling code.
- **Select expressions** (e.g. `{ $count -> [one] … *[other] … }`) are available but not currently used in these files; pluralisation is handled loosely (`{ $count } agent(s) loaded`).

## Message categories

The ~350 messages are grouped into the following functional areas, each prefixed for discoverability:

| Prefix | Domain | Typical caller |
|---|---|---|
| `daemon-*` | Daemon lifecycle (start, stop, restart, background) | `start`, `stop`, `restart` commands |
| `shutdown-*` | Shutdown handshaking including 401 fallback (Issue #4693) | `stop`, `restart` commands |
| `label-*` | Short labels for status tables | `status`, `info` commands |
| `hint-*` | Contextual next-step hints shown after errors or actions | Various commands |
| `guide-*` | Interactive quick-setup wizard prompts | `init` command |
| `init-*` | Initialisation success / cancellation messages | `init` command |
| `error-*` | Error titles and recovery suggestions | Error paths across all commands |
| `detected-*` | Auto-detected LLM provider announcements | `start`, `doctor` commands |
| `desktop-*` | Desktop app launch messages | `desktop` command |
| `dashboard-*` | Dashboard open messages | `dashboard` command |
| `agent-*` | Agent spawn, kill, model-set, template messages | `agent` subcommands |
| `manifest-*` | Agent manifest read/parse errors | `agent` subcommands |
| `section-*` | Section headings for status/info output | `status`, `info`, `doctor` commands |
| `warn-*` | Inline warnings in status output | `status` command |
| `auth-*` / `value-*` | Authentication mode labels and security detail values | `status --verbose`, `security` command |
| `doctor-*` | Diagnostic check result summaries | `doctor` command |
| `audit-*` | Audit trail verification results | Security checks |
| `health-*` | Health check one-liners | Health endpoints |
| `channel-*` | Messaging channel setup flows (Discord, Slack, etc.) | `channel setup` command |
| `vault-*` | Credential vault operations (init, store, rotate) | `vault` subcommands |
| `cron-*` | Scheduled job CRUD messages | `cron` subcommands |
| `approval-*` | Approval workflow responses | `approve`/`reject` commands |
| `memory-*` | Agent memory set/delete messages | `memory` subcommands |
| `device-*` | Device pairing and removal | `device` subcommands |
| `webhook-*` | Webhook CRUD and test messages | `webhook` subcommands |
| `model-*` | Model selection and catalog messages | `model` subcommands |
| `config-*` | Configuration read/write/set-key messages | `config` subcommands |
| `hand-*` | Hand dependency install, pause, resume | `hand` subcommands |
| `uninstall-*` | Uninstall flow progress and cleanup | `uninstall` command |
| `reset-*` | Data reset success/failure | `reset` command |
| `log-*` | Log file tailing messages | `logs` command |

## Adding a new message

1. **Add the key to `en/main.ftl` first** — it is the canonical source and the compile-time fallback.
2. **Copy the key to every other locale** (`zh-CN/main.ftl`, etc.) with a translated value. Leaving a locale file missing a key will fall back to English at runtime, but untranslated keys should be tracked.
3. **Use variables** (`{ $var }`) rather than string concatenation so translators can reorder phrases for their language.
4. **Keep the `-fix` convention** — many error messages have a companion `*-fix` key (e.g. `error-boot-auth` / `error-boot-auth-fix`). The CLI renders them as a title + actionable suggestion pair.

## Adding a new locale

1. Create a new directory under `locales/` named with the locale identifier (e.g. `ja/`).
2. Copy `en/main.ftl` into the new directory as `main.ftl`.
3. Translate every message value, preserving all keys and variable names exactly.
4. Register the locale in the CLI's localization loader (the code that calls `fluent::FluentBundle::add_resource`).

## Key design conventions

- **Prefix-grouped keys** — every key is prefixed by its domain (`daemon-`, `error-`, `hint-`, etc.). This avoids collisions and makes grep/search straightforward across both locale files and Rust source.
- **Paired error + fix messages** — errors that have a user-actionable remedy ship two keys (`error-X` and `error-X-fix`). The CLI's error printer looks for the `-fix` variant and, if present, renders it below the error in a dimmer color.
- **Multiline hints preserve indentation** — Fluent's block syntax (`\n    `) is used for multi-step instructions so they display correctly in a terminal without post-processing.
- **No HTML / no terminal escape codes** — all formatting (colors, bold, etc.) is applied in Rust code; the FTL files contain plain text only.

## The shutdown-401 fallback (Issue #4693)

A notable interaction is documented in-line. When the CLI binary is upgraded via `curl install.sh | sh` without restarting the running daemon, the new CLI's `Authorization` header will not match the old daemon's expected key. The CLI detects the resulting 401 and falls back to a PID-based kill:

```
shutdown-401-detected  →  shutdown-401-explainer
                         shutdown-401-fallback-attempt  →  shutdown-401-fallback-success | shutdown-401-fallback-fail → shutdown-401-fallback-fix
```

All six keys are present in both locales so the full diagnostic + remediation flow works regardless of language setting.

## Relationship to the rest of the CLI

This module is **pure data** — it contains no executable code, no Rust modules, and no build-time code generation. The CLI's localization layer (typically in a `locales` or `i18n` module) reads these files at startup into a `FluentBundle`, then resolves message IDs like `fl("daemon-started")` or `fl("error-boot-auth", provider = "groq")` at each call site. Because the Fluent system handles fallback automatically, a missing translation degrades gracefully to English rather than crashing.