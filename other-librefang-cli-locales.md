# Other — librefang-cli-locales

# librefang-cli-locales

## Purpose

This module provides the internationalization (i18n) layer for the LibreFang CLI. It contains all user-facing strings — status messages, error reports, hints, labels, and section headers — as [Project Fluent](https://projectfluent.org/) (`.ftl`) translation files. Every piece of text the CLI prints is defined here, decoupled from application logic, so the CLI can be localized into any language without touching Rust code.

## Locale Structure

```
locales/
├── en/           # English (primary/default)
│   └── main.ftl
└── zh-CN/        # Simplified Chinese
    └── main.ftl
```

`en/main.ftl` is the canonical source of truth. All other locales must define the same set of message IDs. Missing IDs in a non-English locale fall back to English at runtime.

## Fluent Format Conventions

Each message follows the Fluent syntax:

```fluent
# Simple message
daemon-starting = Starting daemon...

# Message with a variable
daemon-error = Daemon error: { $error }

# Message with multiple variables
kernel-booted = Kernel booted ({ $provider }/{ $model })
```

- **Message IDs** use `kebab-case` and are grouped under comment headers (e.g., `# --- Daemon lifecycle ---`).
- **Variables** are interpolated with `{ $name }` syntax. The calling Rust code passes these as key-value pairs to the Fluent bundle.
- Many messages come in pairs: the error/status message itself, and a companion `-fix` message suggesting remediation (e.g., `daemon-start-fail` + `daemon-start-fail-fix`).

## Message Categories

The locale files are organized into these topical sections:

| Section | Prefixes | Content |
|---|---|---|
| **Daemon lifecycle** | `daemon-*`, `kernel-*` | Startup, shutdown, background launch, health checks |
| **Labels** | `label-*` | Table/field headers for status output |
| **Hints** | `hint-*`, `guide-*` | Contextual guidance, setup wizard prompts |
| **Init** | `init-*` | First-run setup messages |
| **Error messages** | `error-*` | General errors, daemon communication errors, boot errors |
| **Provider detection** | `detected-*` | Auto-detected LLM provider announcements |
| **Desktop app** | `desktop-*` | Desktop application launch messages |
| **Dashboard** | `dashboard-*` | Web dashboard open messages |
| **Agent commands** | `agent-*`, `section-agent-*` | Agent spawn, kill, template listing |
| **Manifest errors** | `manifest-*`, `error-reading-manifest`, `error-parsing-manifest` | Template manifest issues |
| **Status** | `section-*`, `warn-*`, `auth-*`, `value-*` | Status display sections, security status values |
| **Doctor** | `doctor-*` | Diagnostic command output |
| **Channel setup** | `channel-*`, `section-setup-*` | Telegram, Discord, Slack, WhatsApp, Email, Signal, Matrix |
| **Vault** | `vault-*` | Credential vault operations |
| **Cron** | `cron-*` | Scheduled job management |
| **Approvals** | `approval-*` | Approval workflow messages |
| **Memory** | `memory-*` | Agent memory key-value operations |
| **Devices** | `device-*` | Mobile device pairing |
| **Webhooks** | `webhook-*` | Webhook CRUD and testing |
| **Models** | `model-*` | Model catalog and selection |
| **Config** | `config-*` | Configuration file operations |
| **Hand commands** | `hand-*` | Hand dependency and lifecycle messages |
| **Uninstall** | `uninstall-*` | Full uninstall flow (daemon stop, autostart removal, PATH cleanup) |
| **Reset** | `reset-*` | Data directory reset |
| **Logs** | `log-*` | Log file tailing |

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g., `ja/` for Japanese, `fr/` for French).
2. Copy `en/main.ftl` into the new directory as `main.ftl`.
3. Translate every message value to the target language. **Do not modify message IDs or variable names.**
4. Register the locale in the CLI's Fluent bundle initialization (typically in the Rust localization loader).

Example for French (`fr/main.ftl`):

```fluent
daemon-starting = Démarrage du démon...
daemon-stopped = Le démon LibreFang s'est arrêté.
daemon-error = Erreur du démon : { $error }
```

## Adding New Messages

When adding new CLI features that produce user-facing output:

1. Add the message ID and English text to `en/main.ftl` under the appropriate section comment.
2. Follow the naming convention: `{domain}-{brief-description}` for simple messages, or `{domain}-{brief-description}-{suffix}` for paired messages.
3. If the message has a suggested fix, add a companion `*-fix` message.
4. Update all other locale files with the new message ID and a translated value. If a translation isn't ready, the English fallback will be used.

### Naming Conventions

- **Errors:** `error-{context}` (e.g., `error-boot-auth`)
- **Error fixes:** `error-{context}-fix`
- **Status labels:** `label-{name}`
- **Hints:** `hint-{description}`
- **Section headers:** `section-{name}`
- **Warnings:** `warn-{description}`
- **Command success:** `{domain}-{verb}-success` or `{domain}-{verb}ed`

## Variable Reference

These are the commonly used variables across messages:

| Variable | Type | Used in |
|---|---|---|
| `$error` | string | Error messages throughout |
| `$path` | string | File paths in errors, logs, uninstall |
| `$url` | string | Dashboard URLs, daemon addresses |
| `$status` | string/number | HTTP or exit status codes |
| `$count` | number | Agent counts, model counts, key counts |
| `$name` | string | Agent names, channel names, template names |
| `$id` | string | Agent/CRON/webhook/device identifiers |
| `$provider` | string | LLM provider name |
| `$model` | string | Model identifier |
| `$command` | string | CLI subcommand name |
| `$key` | string | Config key, vault key, env var name |
| `$value` | string | Config value being set |
| `$action` | string | Past-tense verb for toggle/respond actions |
| `$field` | string | Agent field name |
| `$display` | string | Provider display name |
| `$env_var` | string | Environment variable name |
| `$max` | number | Upper bound for selection ranges |

## Design Patterns

### Error + Fix Pairs

Most error messages come with a companion fix suggestion. The CLI displays both, giving users an actionable next step:

```fluent
error-boot-auth = LLM provider authentication failed
error-boot-auth-fix = Run `librefang doctor` to check your API key configuration
```

### Section Headers

Messages prefixed with `section-` are used as display headers in formatted output (e.g., status tables). They are standalone labels, not sentences:

```fluent
section-daemon-status = LibreFang Daemon Status
section-active-agents = Active Agents
```

### Detection Messages

Provider detection messages inform users which LLM backend was auto-discovered. They reference specific environment variables so users know what triggered the detection:

```fluent
detected-provider = Detected { $display } ({ $env_var })
detected-ollama = Detected Ollama running locally (no API key needed)
```