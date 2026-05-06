# Other — librefang-cli-locales

# librefang-cli-locales

Localization data for the LibreFang CLI, providing user-facing strings in multiple languages using [Project Fluent](https://projectfluent.org/) (`.ftl`) message files.

## Purpose

Every piece of text the CLI prints — status messages, error descriptions, hints, section headers, labels — is defined in Fluent message files rather than hardcoded in Rust source. This keeps the CLI codebase free of translatable strings and allows community-driven translations without touching application logic.

## Directory Layout

```
librefang-cli/locales/
├── en/           # English (primary / fallback)
│   └── main.ftl
└── zh-CN/        # Simplified Chinese
    └── main.ftl
```

Each subdirectory is named by its [Unicode locale identifier](https://unicode.org/reports/tr35/). Adding a new language means creating a sibling directory (e.g. `ja/`) with its own `main.ftl`.

## How Fluent Messages Work

A Fluent message has an **identifier** (the key used in Rust code), an optional **value** (the default text), and optional **attributes**. Messages can reference **variables** passed at runtime:

```ftl
daemon-error = Daemon error: { $error }
```

In this example:
- `daemon-error` is the message identifier the CLI looks up.
- `{ $error }` is a variable the caller supplies — e.g. an I/O error string.

### Pluralization & Selectors

Fluent handles plural forms and conditional output natively, though the current messages use simple variable interpolation. If pluralization is needed in the future (e.g. "1 agent" vs "5 agents"), Fluent selectors should be used instead of Rust-side branching.

## Message Categories

Messages are organized into logical sections via comments (`# --- Section ---`). Below is a summary of each group and what it covers.

### Daemon Lifecycle

Keys prefixed with `daemon-` handle every state transition of the background daemon process:

| Message pattern | When it appears |
|---|---|
| `daemon-starting` / `daemon-started` / `daemon-stopped-*` | Normal start/stop flow |
| `daemon-already-running` | Duplicate `start` call |
| `daemon-bg-exited` / `daemon-bg-wait-fail` | Background launch failure |
| `daemon-restart*` / `daemon-no-running-*` | Restart / auto-start logic |

### Labels (`label-*`)

Short noun-phrase strings used in status tables and section headers. Examples: `label-api`, `label-status`, `label-agents`, `label-pairing-code`. These rarely change per locale and are used as column headers or inline tags.

### Hints (`hint-*`)

Actionable suggestions printed after errors or during setup:

```
hint-stop-daemon = Use `librefang stop` to stop the daemon
hint-config-edit = Fix with: librefang config edit
```

Every `hint-*` message pairs with a preceding error or status to tell the user what to do next.

### Setup Guide (`guide-*`)

Messages for the interactive first-run wizard (`librefang init`), including provider selection, key verification, and navigation help:

```ftl
guide-help-select = ↑↓ navigate  Enter select  s/Esc skip
guide-key-verified = ✓ Key verified!
```

### Error Messages (`error-*`)

Grouped into sub-categories:

- **General errors** — `error-home-dir`, `error-create-dir`, `error-write-config`
- **Daemon communication** — `error-daemon-returned`, `error-request-timeout`, `error-connect-refused`, `error-daemon-comm`
- **Boot errors** — `error-boot-config`, `error-boot-db`, `error-boot-auth`, `error-boot-generic`
- **Manifest errors** — `manifest-not-found`, `error-reading-manifest`, `error-parsing-manifest`

Many errors include a companion `-fix` message (e.g. `error-boot-auth-fix`) that suggests remediation.

### Agent Commands (`agent-*`)

All feedback from `librefang agent` subcommands: spawn, kill, model-set, template listing.

### Channel Setup (`channel-*`)

Messages for the `librefang channel setup` flow, covering Telegram, Discord, Slack, WhatsApp, Email, Signal, and Matrix.

### Vault (`vault-*`)

Credential vault operations: init, store, remove, and master key rotation. The rotation messages are notably detailed because the process has multiple failure modes (old key invalid, sentinel check, rewrap failure).

### Other Groups

| Prefix | Domain |
|---|---|
| `cron-*` | Scheduled job CRUD |
| `approval-*` | Approval workflow responses |
| `memory-*` | Agent memory key/value operations |
| `device-*` | Mobile device pairing (QR scan) |
| `webhook-*` | Webhook CRUD and testing |
| `model-*` | Model selection and catalog |
| `config-*` | Configuration file read/write/set |
| `hand-*` | Hand (extension) lifecycle |
| `doctor-*` | Diagnostic check results |
| `desktop-*` | Desktop app launching |
| `uninstall-*` | Full uninstall flow |
| `reset-*` | Data reset operations |
| `log-*` | Log tailing |
| `health-*` | Daemon health checks |
| `auth-*` / `value-*` | Security status display |

## Adding a New Message

1. Add the message identifier and English text to `locales/en/main.ftl` under the appropriate section comment.
2. Add translations to each non-English locale file (e.g. `locales/zh-CN/main.ftl`).
3. In the Rust code, reference the message by its identifier through the Fluent localization harness (typically `fl!("message-id", variable = value)` or equivalent macro).

**Naming conventions:**

- Use `kebab-case` identifiers that match the domain and purpose: `domain-detail-suffix`.
- Error fix messages append `-fix`.
- Section headers use `section-*`.
- Labels use `label-*`.
- Hints use `hint-*`.

## Adding a New Language

1. Create `locales/<locale-id>/main.ftl` (e.g. `locales/fr/main.ftl`).
2. Copy the English file as a starting point.
3. Translate all message values. Do **not** translate message identifiers or variable names.
4. Register the locale in the application's Fluent loader configuration (Rust side).

### Translation Guidelines

- **Keep placeholders intact.** `{ $error }`, `{ $path }`, `{ $url }`, `{ $count }`, etc. must appear exactly as in the English source.
- **Preserve inline code references.** Backtick-wrapped commands like `` `librefang start` `` should remain in Latin characters even in non-Latin locales (the commands themselves are not localized).
- **Match the tone.** The CLI uses a direct, concise style. Avoid adding politeness markers that aren't in the original.
- **Translate section comments if helpful** but they are stripped at runtime — only message values matter.

## Relationship to the CLI Codebase

This module is a **pure data** module — no Rust code, no logic, no call graph edges. It is consumed by the CLI's localization harness, which:

1. Loads the Fluent bundle for the user's locale at startup (falling back to `en`).
2. Resolves message identifiers referenced throughout `librefang-cli` command handlers.
3. Passes runtime variables (error strings, paths, counts) into the Fluent formatter.

The CLI commands never format user-facing strings themselves; they delegate to the Fluent bundle and these `.ftl` files.