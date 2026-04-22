# Other — librefang-cli-locales

# LibreFang CLI Locales

Localization resources for the LibreFang command-line interface, providing all user-facing strings in multiple languages using [Project Fluent](https://projectfluent.org/) (`.ftl`) format.

## Structure

```
librefang-cli/locales/
├── en/           # English (default / fallback)
│   └── main.ftl
└── zh-CN/        # Simplified Chinese
    └── main.ftl
```

Each locale directory contains a single `main.ftl` file holding the complete translation catalog for that language. The locale directory names follow [BCP 47](https://tools.ietf.org/html/b4647) conventions.

## Format

All files use Fluent's `.ftl` syntax. A message consists of an identifier, optional attributes, and a value that may contain variables and select expressions:

```ftl
# Simple message
daemon-starting = Starting daemon...

# Message with a variable
daemon-error = Daemon error: { $error }

# Message with a typed variable (pluralization, etc.)
agents-loaded = { $count } agent(s) loaded
```

Variables like `{ $error }`, `{ $path }`, `{ $name }`, `{ $id }`, `{ $provider }`, `{ $model }`, `{ $command }`, `{ $action }`, `{ $status }`, `{ $count }`, `{ $url }`, `{ $display }`, `{ $env_var }`, `{ $key }`, `{ $value }`, `{ $field }`, `{ $agent }`, and `{ $max }` are passed at call sites in the Rust codebase via the `fluent` / `flume` / `icu` integration layer.

## Message Categories

Messages are organized into sections by comment headers within each file:

| Section | Prefix | Purpose |
|---|---|---|
| Daemon lifecycle | `daemon-*` | Start, stop, restart, background launch, health checks |
| Labels | `label-*` | Field labels for status displays and tables |
| Hints | `hint-*` | Contextual guidance shown after commands or errors |
| Setup guide | `guide-*` | Interactive first-run wizard strings |
| Initialization | `init-*` | `librefang init` success/cancel messages |
| Errors | `error-*` | Error messages, each paired with a `*-fix` variant |
| Boot errors | `error-boot-*` | Kernel startup failures |
| Provider detection | `detected-*` | Auto-detected LLM provider announcements |
| Desktop app | `desktop-*` | Desktop launcher status |
| Dashboard | `dashboard-*` | Web dashboard open messages |
| Agent commands | `agent-*` | Spawn, kill, model-set, template operations |
| Manifest errors | `manifest-*`, `error-*-manifest` | Agent manifest file issues |
| Status display | `section-*`, `label-*`, `warn-*` | Status subcommand output sections and warnings |
| Doctor | `doctor-*` | Diagnostic check results |
| Security | `label-*`, `value-*`, `audit-*` | Security status display values |
| Health | `health-*` | Simple health check outcomes |
| Channel setup | `channel-*`, `section-setup-*` | Messaging channel configuration (Telegram, Discord, Slack, WhatsApp, Email, Signal, Matrix) |
| Vault | `vault-*` | Credential vault operations |
| Cron | `cron-*` | Scheduled job management |
| Approvals | `approval-*` | Human-in-the-loop approval responses |
| Memory | `memory-*` | Agent memory key-value operations |
| Devices | `device-*` | Mobile device pairing |
| Webhooks | `webhook-*` | Webhook CRUD and testing |
| Models | `model-*`, `section-select-*` | Model catalog selection |
| Config | `config-*` | Configuration get/set/unset, key management, env file ops |
| Hand commands | `hand-*` | Hand instance lifecycle (install deps, pause, resume) |
| Daemon notify | `daemon-restart-notify` | Post-config-change restart reminder |
| System info | `section-system-info` | System information header |
| Uninstall | `uninstall-*` | Full uninstall flow including per-OS autostart cleanup |
| Reset | `reset-*` | Data directory reset |
| Logs | `log-*` | Log file tailing |

## Error/Fix Pattern

Most error messages follow a paired convention: the error message itself and an adjacent `*-fix` message providing remediation guidance. For example:

```ftl
error-boot-auth = LLM provider authentication failed
error-boot-auth-fix = Run `librefang doctor` to check your API key configuration
```

The calling code retrieves both messages and displays them together, giving users actionable next steps for every failure mode.

## Adding a New Message

1. Add the message identifier and English text to `en/main.ftl` under the appropriate section comment.
2. Mirror the identifier in every other locale's `main.ftl` with the translated value.
3. Use the message ID in Rust code via the Fluent localization helper (typically `fl!("message-id", variable => value)` or the project's wrapper).
4. If the message includes variables, ensure they match the names passed at the call site.

## Adding a New Locale

1. Create a new directory under `locales/` named with the appropriate BCP 47 tag (e.g., `ja` for Japanese, `de` for German).
2. Copy `en/main.ftl` into the new directory.
3. Translate every message value while preserving:
   - All message identifiers (left-hand side of `=`)
   - All variable references (`{ $var }`)
   - All section comments (lines starting with `#`)
4. Register the locale in the application's locale negotiation / configuration layer.

## Naming Conventions

- **Kebab-case** for all message identifiers: `daemon-started-bg`, `config-set-kv`
- **Dashed prefixes** group related messages: `agent-*`, `vault-*`, `cron-*`
- **`-fix` suffix** marks remediation hints paired with an error
- **`section-*` prefix** marks headings used as table/section dividers in output
- **`label-*` prefix** marks short field labels (often used in key-value displays)
- **`hint-*` prefix** marks longer contextual guidance strings