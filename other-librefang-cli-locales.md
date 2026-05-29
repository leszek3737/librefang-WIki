# Other — librefang-cli-locales

# librefang-cli-locales

Fluent (FTL) localization files for the LibreFang CLI. Every user-facing string in the CLI is defined here, making the application fully translatable without touching command handlers or UI code.

## Architecture

The module is a pure data layer — it contains no executable Rust code. At runtime, the CLI's Fluent integration (via `fluent-templates` or an equivalent crate) loads the `.ftl` file matching the user's locale and resolves message IDs to translated strings with variable interpolation.

```
locales/
├── en/
│   └── main.ftl      ← primary (English)
└── zh-CN/
    └── main.ftl      ← Simplified Chinese
```

## File Format

Each locale file follows [Project Fluent](https://projectfluent.org/) syntax:

```fluent
# Comments for developer context
message-id = Translated text
message-with-var = Hello { $name }
message-multiline =
    Line one.
    Line two.
```

### Message ID Conventions

| Pattern | Purpose | Example |
|---|---|---|
| `kebab-case` | Top-level message | `daemon-starting` |
| `*-fix` | Remediation hint paired with an error | `error-boot-config-fix` |
| `label-*` | Short UI labels | `label-uptime` |
| `hint-*` | Contextual user guidance | `hint-check-status` |
| `section-*` | Section headings in terminal output | `section-security-status` |
| `warn-*` | Warning indicators | `warn-public-bind` |
| `auth-*` | Authentication mode descriptions | `auth-api-key` |
| `value-*` | Display values for status fields | `value-audit-trail` |
| `detected-*` | Auto-detection result messages | `detected-provider` |
| `guide-*` | Interactive wizard strings | `guide-paste-key-hint` |

The `-fix` convention is significant: when the CLI prints an error, it typically follows with the corresponding `-fix` message to tell the user what to do next. For example:

```
error-connect-refused = Cannot connect to daemon
error-connect-refused-fix = Is the daemon running? Start it with: librefang start
```

## Message Categories

The files are organized into topical sections, each prefixed with a comment header:

### Daemon Lifecycle
Start, stop, restart, background launch, health-wait logic, and the 401 fallback path (Issue #4693 — handles the case where a `curl install.sh | sh` upgrade replaces the binary but the running daemon still holds the old API key).

### Error Hierarchy

```
error-home-dir              ← environment / filesystem
error-create-dir
error-write-config

error-daemon-returned       ← daemon communication
error-request-timeout
error-connect-refused
error-daemon-comm

error-boot-config           ← kernel boot failures
error-boot-db
error-boot-auth
error-boot-generic

error-require-daemon        ← command precondition
```

Each error has a companion `-fix` message except `error-require-daemon` (which uses `-fix` to direct the user to `librefang start`).

### Labels & Display
Static labels (`label-api`, `label-daemon`, `label-uptime`, etc.) used in status tables, info panels, and section headers. These rarely contain variables.

### Hints
Actionable suggestions shown after commands or in interactive prompts. Many reference specific CLI subcommands:

```fluent
hint-doctor-repair = Run `librefang doctor --repair` to attempt auto-fix
hint-run-init = Run `librefang init` to set up the agents directory
```

### Interactive Wizard (`guide-*`)
Strings for the `librefang init` interactive setup flow — provider selection, key pasting, verification feedback, and navigation help.

### Domain-Specific Sections
- **Agents** — spawn, kill, template resolution, field mutation
- **Vault** — init, unlock, store/remove, master key rotation
- **Cron** — create, delete, toggle
- **Approvals** — respond to pending approvals
- **Memory** — per-agent key-value store operations
- **Devices** — QR pairing, removal
- **Webhooks** — CRUD and test
- **Models** — catalog selection and default model
- **Config** — get/set/unset, env file management, parse errors
- **Channels** — setup flows for discord, slack, whatsapp, email, signal, matrix
- **Hands** — dependency install, pause/resume
- **Security** — audit trail verification, sandbox and protocol status display
- **Uninstall/Reset** — cleanup across platforms (Windows registry, macOS launch agent, Linux autostart, systemd)

## Variable Interpolation

Messages use Fluent's `{ $variable }` syntax. Common variables:

| Variable | Context |
|---|---|
| `$error` | Error description from the runtime |
| `$path` | Filesystem path (logs, config, PID file) |
| `$url` | Daemon or dashboard URL |
| `$status` | HTTP status code or process exit code |
| `$command` | CLI subcommand name |
| `$provider` / `$model` | LLM provider and model identifiers |
| `$count` | Numeric count (agents, entries, models) |
| `$id` | Entity identifier (agent, cron job, webhook, device) |
| `$name` | Human-readable name (agent, channel, provider) |
| `$key` / `$value` | Config or vault key-value pair |
| `$pid` | OS process ID |
| `$action` | Past-tense verb for toggle/approve operations |
| `$field` | Agent field name |
| `$display` / `$env_var` | Provider display name and environment variable |

## Adding a New Locale

1. Create `librefang-cli/locales/<locale-code>/main.ftl`.
2. Copy the English file as the starting point.
3. Translate every message value, keeping all message IDs and variable references identical.
4. Register the locale in the CLI's Fluent loader configuration (typically a `locales` array or the `fallback-language` setting in the build script).

To verify completeness, diff the message IDs between the new locale and `en/main.ftl` — every ID present in English must exist in the new locale.

## Adding New Messages

1. Add the message to `en/main.ftl` first — this is the source of truth.
2. Follow the naming conventions above (use the appropriate prefix).
3. If the message is an error, add a companion `-fix` message.
4. Mirror the new entries to all other locale files immediately, even if translations are pending (use English as a placeholder to avoid runtime fallback surprises).

## Notable Design Decisions

**401 shutdown fallback (Issue #4693):** The `shutdown-401-*` messages handle a specific failure mode where an in-place binary upgrade causes authentication mismatch between the new CLI and the old daemon. The messages explain the cause, attempt PID-based termination, and provide manual fallback instructions. This is implemented entirely through message strings rather than special error types — the CLI handler orchestrates the logic and selects the appropriate message IDs.

**Separation of error and fix:** Errors and their remediations are separate messages. This lets the CLI format them differently (e.g., red for the error, dimmed for the fix) and optionally suppress fixes in machine-readable output modes.

**Multi-line messages:** Some messages like `shutdown-401-fallback-fix` use Fluent's block syntax with indented continuation lines. These render as multi-line terminal output with preserved formatting.