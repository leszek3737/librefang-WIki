# Other — librefang-cli-locales

# librefang-cli-locales

Localization resources for the LibreFang CLI, using [Project Fluent](https://projectfluent.org/) (`.ftl`) message files. This module is a **pure data layer** — it contains no executable code, only translation strings consumed at runtime by the CLI's i18n formatter.

## Directory Layout

```
librefang-cli/locales/
├── en/           # English (primary / fallback locale)
│   └── main.ftl
└── zh-CN/        # Simplified Chinese
    └── main.ftl
```

Each locale directory holds a single `main.ftl` file containing every user-facing string the CLI can emit.

## How It Works

The CLI loads Fluent bundles at startup based on the user's system locale (with `en` as the unconditional fallback). Every `print!` or `eprintln!` that faces the user is replaced by a Fluent message lookup. Messages are keyed by a dot-free identifier (e.g. `daemon-starting`, `config-parse-error`) and may include **variables** interpolated at runtime.

Example Fluent definition:

```ftl
daemon-already-running = Daemon already running at { $url }
```

Rendered in code as something like:

```rust
bundle.get_message("daemon-already-running")
    .and_then(|msg| msg.value())
    .map(|pattern| bundle.format_pattern(pattern, &args, &mut errors))
```

## Message Organization

Messages are grouped into logical sections within each `.ftl` file, separated by comments. Both locales follow the **exact same section order and message identifiers**.

| Section | Covers | Typical Variables |
|---|---|---|
| **Daemon lifecycle** | `start`, `stop`, `restart`, background launch, health-wait | `$url`, `$status`, `$error`, `$path`, `$pid` |
| **Labels** | Table/field names in status output | — |
| **Hints** | One-line user guidance after commands | `$url`, `$name`, `$provider` |
| **Init** | First-run setup success/cancel | — |
| **Error messages** | General, daemon-comm, boot failures | `$error`, `$status`, `$path` |
| **Provider detection** | Auto-detected LLM backends | `$display`, `$env_var` |
| **Desktop app** | `librefang desktop` launch messages | `$error` |
| **Dashboard** | Browser-open feedback | `$url` |
| **Agent commands** | Spawn, kill, model-set, templates | `$name`, `$id`, `$error`, `$field`, `$value` |
| **Manifest errors** | Missing/corrupt agent manifests | `$path`, `$error`, `$name` |
| **Status** | `librefang status` section headers and labels | `$count` |
| **Doctor** | Diagnostic pass/fail output | — |
| **Security** | Audit trail, sandbox, wire-protocol status | `$count` |
| **Health** | Simple up/down check | — |
| **Channel setup** | Telegram, Discord, Slack, WhatsApp, Email, Signal, Matrix | `$name`, `$key` |
| **Vault** | Init, unlock, store, remove, rotate master key | `$key`, `$error`, `$count`, `$path` |
| **Cron** | Create, delete, toggle scheduled jobs | `$id`, `$action`, `$error` |
| **Approvals** | Approve/reject pending approvals | `$id`, `$action`, `$error` |
| **Memory** | Per-agent key-value memory | `$agent`, `$key`, `$error` |
| **Devices** | Mobile app pairing via QR | `$id`, `$error` |
| **Webhooks** | Create, delete, test webhook endpoints | `$id`, `$error` |
| **Models** | Default model selection | `$model`, `$error`, `$max` |
| **Config** | Get/set/unset config values, `set-key` | `$key`, `$value`, `$error`, `$env_var`, `$provider` |
| **Hand commands** | Install deps, pause, resume hand instances | `$id` |
| **Uninstall** | Teardown across Windows, macOS, Linux | `$path`, `$error` |
| **Reset** | Remove data directories | `$path`, `$error` |
| **Logs** | `librefang logs --follow` output | `$path` |

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate BCP 47 tag (e.g. `ja` for Japanese).
2. Copy `en/main.ftl` into the new directory.
3. Translate every message value while preserving:
   - The message identifier (left of `=`) exactly.
   - Variable syntax: `{ $variable }`.
   - Multiline formatting (indented continuation lines), especially in messages like `shutdown-401-fallback-fix`.
   - Section comments (lines starting with `#`) are optional but recommended for maintainability.
4. Register the locale in the CLI's Fluent bundle loader (the code that calls `fluent::FluentBundle::add_resource`).

## Adding a New Message

1. Add the identifier and English text to `en/main.ftl` in the appropriate section.
2. Add the same identifier to every other locale's `main.ftl` with a translated value.
3. If the new message is never translated in a locale, the Fluent bundle will fall back to English automatically — but missing identifiers produce no output, so always add the key everywhere.

## Variable Convention

Messages use positional `$variables` that map to `FluentArgs` entries passed from Rust code. The convention is:

| Variable | Meaning |
|---|---|
| `$error` | Error message string from an `anyhow::Error` or similar |
| `$path` | Filesystem path |
| `$url` | HTTP URL |
| `$id` | Entity identifier (agent, cron job, webhook, etc.) |
| `$name` | Human-readable name |
| `$count` | Numeric count |
| `$status` | HTTP status code or exit code |
| `$pid` | OS process ID |
| `$action` | Verb like "pause"/"resume" or "approve"/"reject" |
| `$provider` | LLM provider slug |
| `$model` | Model identifier |
| `$key` | Config key or vault key name |
| `$value` | Config value |
| `$env_var` | Environment variable name |
| `$command` | CLI subcommand name |
| `$display` | Human-readable provider name |

## Notable Patterns

### Error + Fix Pairs

Many error messages come in pairs — the error itself and a companion `*-fix` message that suggests remediation:

```ftl
error-connect-refused = Cannot connect to daemon
error-connect-refused-fix = Is the daemon running? Start it with: librefang start
```

This lets the CLI print the error and its suggestion as separate lines, with different formatting if desired.

### Issue-Linked Comments

Messages tied to specific bug reports or design decisions carry a comment referencing the issue number, so future translators understand the context:

```ftl
# Issue #4693 — after `curl install.sh | sh` upgrades the binary without
# restarting the running daemon ...
shutdown-401-detected = Shutdown request was rejected ...
```

### Multiline Messages

Some messages span multiple lines using Fluent's block syntax. The indentation is part of the output:

```ftl
shutdown-401-fallback-fix = Stop the daemon manually, then start it again:
    kill { $pid }    # or: kill -9 { $pid } if it does not exit
    librefang start
```

## Relationship to the Codebase

```
┌──────────────────────┐
│  librefang-cli       │
│  (Rust binary)       │
│                      │    loads at startup
│  Fluent bundle  ◄────┼──────────────────────────┐
│  formatter            │                          │
└──────────────────────┘                          │
                                                  │
┌─────────────────────────────────────────────────┘
│  librefang-cli/locales/
│    ├── en/main.ftl     ← source of truth
│    └── zh-CN/main.ftl  ← mirrors en/ structure
```

No other modules call into this directory. The CLI's i18n layer reads the `.ftl` files at compile time (via `include_str!`) or runtime, builds `FluentBundle` instances, and the rest of the CLI references messages by their string identifiers. Changes to message IDs require a corresponding update in the Rust code that looks them up.