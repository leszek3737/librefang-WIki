# Other — librefang-cli-locales

# librefang-cli-locales

## Overview

This module provides the **internationalization (i18n) layer** for the LibreFang CLI. It contains all user-facing strings—status messages, error reports, hints, labels, and interactive prompts—in [Project Fluent](https://projectfluent.org/) (`.ftl`) format. Two locales are currently maintained:

| Locale | Path |
|---|---|
| English (default) | `librefang-cli/locales/en/main.ftl` |
| Simplified Chinese | `librefang-cli/locales/zh-CN/main.ftl` |

The module is **pure data** — no executable code, no imports, no side effects. CLI subcommands reference message keys at runtime through the Fluent localization API, and the active locale is resolved from the user's environment (typically `LANG` / `LC_ALL`).

---

## File Format

Every line follows the Fluent grammar. The patterns used in this module are:

```
# comment              → section divider / developer note
key = value            → simple message
key = text { $var }    → message with a variable
```

There are **no plurals, selectors, or message attributes** in the current files — all messages are flat string patterns with optional positional variables.

### Variable Convention

Variables are passed from the calling code at runtime. Commonly used variables across both locales:

| Variable | Example usage |
|---|---|
| `$error` | Error descriptions passed from `anyhow` / `thiserror` |
| `$path` | Filesystem paths (config, logs, data) |
| `$url` | Daemon bind address, dashboard URL |
| `$status` | HTTP-style status codes or exit codes |
| `$name` | Agent names, channel names, template names |
| `$id` | Agent IDs, cron job IDs, webhook IDs, device IDs |
| `$provider` / `$model` | LLM provider and model identifiers |
| `$count` | Numeric counts (agents, keys) |
| `$command` | CLI subcommand name (`chat`, `status`, etc.) |
| `$action` | Verb for toggle/approval operations (`pause`, `approve`) |
| `$field` | Config field names |
| `$key` / `$value` | Config key-value pairs, vault keys, env var names |
| `$env_var` / `$display` | Provider detection display strings |

---

## Message Organization

Messages are grouped by functional domain using comment headers. The grouping is identical in both locales. Below is the full category map with the prefix convention each group uses:

### Daemon Lifecycle
Keys: `daemon-*`, `kernel-*`, `models-*`, `agents-*`

Covers the entire daemon lifecycle: starting, background launch, health-wait, stop (graceful and forced), restart, and all failure modes with suggested fixes.

### Labels
Keys: `label-*`

Short noun phrases used as table headers, field labels, and status indicators (e.g. `label-api`, `label-uptime`, `label-active-agents`).

### Hints
Keys: `hint-*`, `guide-*`

Actionable suggestions shown after commands or errors. The `guide-*` subset powers the interactive first-run setup wizard (provider selection, key pasting, verification feedback, navigation help).

### Init
Keys: `init-*`

Messages for `librefang init` — success confirmations, cancellation, and next-step instructions.

### Error Messages
Keys: `error-*`

Subdivided into:
- **General errors** — home directory, file creation, config write failures
- **Daemon communication errors** — HTTP errors, timeouts, connection refused, with `-fix` companion messages
- **Boot errors** — config parse, database lock, authentication, generic kernel boot failure
- **Require daemon** — commands that need a running daemon

Each error has a companion `-fix` key providing the suggested remediation command.

### Provider Detection
Keys: `detected-*`

Auto-detection messages for LLM providers (Gemini, Ollama, generic).

### Desktop / Dashboard
Keys: `desktop-*`, `dashboard-*`

Launch status for the desktop app and dashboard URL opening.

### Agent Commands
Keys: `agent-*`, `section-agent-*`

Spawn, kill, model-set, template listing — the full agent management surface. Includes `manifest-*` keys for template manifest errors.

### Status
Keys: `section-*-status`, `label-*` (status-specific), `warn-*`, `auth-*`

Structured status display: daemon health, agents, security posture, system info. Includes `section-status-locked` for restricted-status scenarios.

### Doctor
Keys: `doctor-*`, `section-getting-*`

Diagnostic command output — pass/fail summaries, repair confirmations, API key guidance.

### Security
Keys: `section-security-*`, `label-*` (security-specific), `value-*`, `audit-*`

Security posture display values (audit trail, taint tracking, WASM sandbox, wire protocol) and audit integrity check results.

### Channel Setup
Keys: `channel-*`, `section-setup-*`

Interactive setup flows for seven messaging channels: `telegram`, `discord`, `slack`, `whatsapp`, `email`, `signal`, `matrix`.

### Vault
Keys: `vault-*`

Credential vault operations — init, unlock, store, remove, and associated error states.

### Cron
Keys: `cron-*`

Scheduled job CRUD and toggle (pause/resume) operations.

### Approvals
Keys: `approval-*`

Human-in-the-loop approval responses.

### Memory
Keys: `memory-*`

Per-agent key-value memory store operations.

### Devices
Keys: `section-device-*`, `device-*`

QR-code-based device pairing and removal.

### Webhooks
Keys: `webhook-*`

Webhook CRUD and test dispatch.

### Models
Keys: `model-*`, `section-select-model`

Model catalog listing and default-model selection.

### Config
Keys: `config-*`

Config read/write/edit, key-value set/unset, env-file management (`set-key` / `remove-key`), and parse error recovery.

### Hands
Keys: `hand-*`

Dependency installation, pause, and resume for Hand instances.

### Uninstall
Keys: `uninstall-*`

Full uninstall flow — daemon stop, file removal, platform-specific autostart cleanup (Windows registry, macOS launch agent, Linux systemd/desktop), and PATH cleanup.

### Reset / Logs
Keys: `reset-*`, `log-*`

Data directory reset and log tailing.

---

## Usage from CLI Code

The CLI module loads these Fluent resources at startup using a Fluent bundler. A typical call site looks like:

```rust
// Pseudocode — actual API depends on the Fluent integration crate used
let msg = bundle.get_message("daemon-started-bg");
println!("{}", msg);
```

For parameterized messages:

```rust
let mut args = FluentArgs::new();
args.set("error", error.to_string());
let msg = bundle.get_message("daemon-start-fail");
// render with args → "Could not start daemon: connection refused"
```

### Adding a New String

1. Add the key to `locales/en/main.ftl` under the appropriate section comment.
2. Add the translated key to `locales/zh-CN/main.ftl` in the same section position.
3. Reference the key from the CLI command handler using the Fluent bundle.

### Adding a New Locale

1. Create `locales/<locale-code>/main.ftl`.
2. Copy the English file as a template.
3. Translate every value (leave keys and variable names unchanged).
4. Register the locale in the CLI's locale resolution logic.

---

## Naming Conventions

| Pattern | Purpose | Example |
|---|---|---|
| `noun-verb` | Action result | `daemon-started`, `vault-stored` |
| `noun-verb-fail` / `noun-verb-failed` | Action failure | `daemon-start-fail`, `agent-spawn-failed` |
| `noun-verb-fix` | Suggested fix | `error-boot-auth-fix` |
| `section-*` | UI section heading | `section-active-agents` |
| `label-*` | Field/table label | `label-provider` |
| `hint-*` | Guidance message | `hint-stop-daemon` |
| `warn-*` | Warning indicator | `warn-public-bind` |
| `value-*` | Descriptive value | `value-audit-trail` |
| `auth-*` | Auth method label | `auth-api-key` |

Every error-path message is paired with a `-fix` companion key so the CLI can render both the problem and the remediation step together.