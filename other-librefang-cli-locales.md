# Other — librefang-cli-locales

# librefang-cli-locales

Internationalization (i18n) resource files for the LibreFang CLI. Contains all user-facing strings in [Fluent](https://projectfluent.org/) (`.ftl`) format, enabling multilingual terminal output without code changes.

## Supported Locales

| Locale | File | Description |
|--------|------|-------------|
| `en` | `locales/en/main.ftl` | English — default / fallback |
| `zh-CN` | `locales/zh-CN/main.ftl` | Simplified Chinese |

## How It Works

The Fluent system is declarative. Each file maps a **message identifier** to a localized string value. Application code references message IDs; the runtime selects the correct file based on the user's locale setting.

### Variable Interpolation

Strings can include placeholders wrapped in curly braces:

```fluent
daemon-error = Daemon error: { $error }
kernel-booted = Kernel booted ({ $provider }/{ $model })
```

The calling code supplies the variables at runtime — for example, passing `error = "port in use"` produces `Daemon error: port in use`.

### Message ID Conventions

Message IDs use a consistent naming scheme:

| Pattern | Purpose | Example |
|---------|---------|---------|
| `section-*` | Section headings in terminal output | `section-daemon-status`, `section-security-status` |
| `label-*` | Field labels in status/config output | `label-uptime`, `label-provider` |
| `hint-*` | Contextual suggestions shown to the user | `hint-stop-daemon`, `hint-check-status` |
| `error-*` | Error messages | `error-boot-config`, `error-connect-refused` |
| `*-fix` | Remediation hint paired with an error | `error-boot-auth-fix`, `error-connect-refused-fix` |
| `warn-*` | Warning labels | `warn-public-bind`, `warn-key-missing` |
| `value-*` | Static descriptive values | `value-audit-trail`, `value-wasm-sandbox` |
| `*-success`, `*-failed` | Operation outcomes | `model-set-success`, `cron-create-failed` |

## String Categories

The locale files cover every user-visible string the CLI emits. Major sections include:

- **Daemon lifecycle** — start, stop, restart, background mode, health checks
- **Error diagnostics** — boot failures, communication errors, config parse errors; each with a paired `*-fix` suggestion
- **Hints** — contextual guidance after commands or errors
- **Interactive setup** — first-run wizard prompts and key verification
- **Agent commands** — spawn, kill, model selection, template management
- **Status display** — daemon status, security status, system info sections
- **Channel setup** — Telegram, Discord, Slack, WhatsApp, Email, Signal, Matrix
- **Config management** — get/set/unset operations, key storage
- **Vault operations** — credential store init, unlock, get, set, remove
- **Cron scheduling** — create, delete, toggle cron jobs
- **Uninstall** — platform-specific cleanup (macOS launch agent, Linux systemd, Windows registry)
- **Doctor** — diagnostic check results and repair guidance

## Adding a New Locale

1. Create the directory: `librefang-cli/locales/<locale-code>/main.ftl`
2. Copy `locales/en/main.ftl` as the starting template.
3. Translate every string value, preserving the message IDs and variable names exactly.
4. Missing messages automatically fall back to the English equivalent.

**Critical rules when translating:**

- **Never rename a message ID.** Code references these IDs directly.
- **Preserve all `{ $variable }` placeholders** with identical variable names. Removing or renaming one will cause a runtime error.
- **Keep paired `*-fix` messages adjacent** to their parent error for maintainability (the comments act as section markers).
- **Test with the CLI** — Fluent's plural rules, selector syntax, and whitespace handling can produce subtle differences across locales.

## Relationship to the Codebase

This module is a leaf dependency. It contains no executable Rust code, no function calls, and no imports. The CLI binary loads these files at startup via a Fluent localization crate and looks up message IDs wherever user-facing text is needed.

When adding a new CLI command or error path, the workflow is:

1. Add the message ID and English string to `locales/en/main.ftl`.
2. Add the translated equivalent to every other locale file (or leave it to fall back to English temporarily).
3. Reference the new message ID from the Rust code via the localization handle.