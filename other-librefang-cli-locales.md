# Other — librefang-cli-locales

# LibreFang CLI Locales

## Overview

The `librefang-cli-locales` module provides the internationalization (i18n) layer for the LibreFang CLI. It contains [Project Fluent](https://projectfluent.org/) (`.ftl`) message catalogs that supply all user-facing strings for the command-line interface. These files are pure data — no executable code, no imports, no runtime dependencies. They are consumed at build time or runtime by the CLI's localization machinery.

### Supported Locales

| Locale | File | Language |
|--------|------|----------|
| `en` | `locales/en/main.ftl` | English (default) |
| `zh-CN` | `locales/zh-CN/main.ftl` | Simplified Chinese |

Each locale file is a complete, standalone translation containing every message key used by the CLI. The two files are structurally identical — every key in `en/main.ftl` has a corresponding entry in `zh-CN/main.ftl`.

---

## File Format

Messages use the [FTL syntax](https://projectfluent.org/fluent/guide/):

```fluent
# Simple string
daemon-starting = Starting daemon...

# String with a variable
kernel-booted = Kernel booted ({ $provider }/{ $model })

# Section comment (for human organization, not parsed)
# --- Daemon lifecycle ---
```

Variables are interpolated at runtime via `{ $name }`. The CLI code passes a map of variable values when resolving a message.

---

## Message Catalog Structure

Messages are organized into functional domains by comment headers. The sections below describe each domain and the kinds of messages it contains.

### Daemon Lifecycle

Core messages for the daemon's start, stop, restart, and background operation flows. This is the largest single domain and covers:

- Startup/shutdown confirmations (`daemon-starting`, `daemon-stopped-ok`, `daemon-stopped-forced`)
- Background daemon status (`daemon-started-bg`, `daemon-still-starting`)
- Error states (`daemon-bg-exited`, `daemon-bg-wait-fail`, `daemon-launch-fail`)
- Already-running / not-running conditions (`daemon-already-running`, `daemon-not-running`)
- Kernel boot confirmation with provider/model variables (`kernel-booted`)

### Labels

Short, reusable field labels for display in status tables, configuration output, and diagnostic panels. Examples: `label-api`, `label-pid`, `label-status`, `label-uptime`, `label-version`, `label-pairing-code`. These are typically combined with dynamic values in the CLI's output formatting layer.

### Hints

Actionable user-facing guidance appended after primary output. Hints always suggest a concrete command or action:

```fluent
hint-stop-daemon = Use `librefang stop` to stop the daemon
hint-doctor-repair = Run `librefang doctor --repair` to attempt auto-fix
```

Hints often appear in pairs — a short diagnostic line followed by a hint with the `-fix` suffix pattern used throughout the catalog.

### Error Messages

Errors follow a consistent pattern: a primary message describing the failure, and a companion `-fix` message with remediation steps.

| Primary Key | Fix Key | Example |
|-------------|---------|---------|
| `error-create-dir` | `error-create-dir-fix` | Failed to create path → check permissions |
| `error-boot-config` | `error-boot-config-fix` | Config parse error → check syntax |
| `error-connect-refused` | `error-connect-refused-fix` | Connection refused → start the daemon |

Error domains include:

- **General filesystem errors** — home directory, directory creation, config writes
- **Daemon communication** — timeouts, refused connections, generic comm errors
- **Boot errors** — config parsing, database locks, auth failures, generic kernel boot
- **Daemon requirement** — commands that need a running daemon (`error-require-daemon`)

### Init / Setup Wizard

Messages for the `librefang init` interactive setup flow and the quick-setup guide:

- Setup completion confirmations (`init-quick-success`, `init-interactive-success`)
- Provider selection guide (`guide-free-providers-title`, `guide-get-free-key`)
- API key entry prompts (`guide-paste-key-placeholder`, `guide-paste-key-hint`)
- Key verification feedback (`guide-key-verified`, `guide-test-key-unverified`)
- Navigation help for interactive selectors (`guide-help-select`, `guide-help-paste`)

### Agent Commands

All output for agent lifecycle operations:

- Spawning (`agent-spawned`, `agent-spawned-inprocess`, `agent-spawn-failed`)
- Template discovery (`agent-template-not-found`, `agent-no-templates`)
- Lifecycle control (`agent-killed`, `agent-kill-failed`)
- Configuration (`agent-model-set`, `agent-unknown-field`)
- In-process vs. daemon mode distinction (`agent-note-lost`, `agent-note-persistent`)

### Status Display

Section headers and labels for the `librefang status` command output:

```fluent
section-daemon-status = LibreFang Daemon Status
section-active-agents = Active Agents
label-daemon-not-running = NOT RUNNING
```

Includes status-locked indicators, authentication type labels, and verbose detail sections.

### Doctor (Diagnostics)

Messages for the `librefang doctor` health-check command:

- Title and summary (`doctor-title`, `doctor-all-passed`, `doctor-some-failed`)
- Repair feedback (`doctor-repairs-applied`)
- API key guidance (`doctor-no-api-keys`, `section-getting-api-key`)

### Security

Labels and values for the security status display, describing LibreFang's security mechanisms:

```fluent
label-audit-trail = Audit trail
value-audit-trail = Merkle hash chain (SHA-256)
label-wasm-sandbox = WASM sandbox
value-wasm-sandbox = Dual metering (fuel + epoch)
```

Audit verification results (`audit-verified`, `audit-failed`) are also here.

### Channel Setup

Strings for configuring messaging integrations (`telegram`, `discord`, `slack`, `whatsapp`, `email`, `signal`, `matrix`):

- Per-channel setup section headers (`section-setup-telegram`, etc.)
- Token/credential save confirmations (`channel-token-saved`, `channel-bot-token-saved`)
- Test results (`channel-test-ok`, `channel-test-fail`)

### Vault

Credential vault operations — initialization, storage, retrieval, and removal:

```fluent
vault-stored = Stored '{ $key }' in vault.
vault-key-not-found = Key '{ $key }' not found in vault.
```

### Cron, Approvals, Memory, Webhooks

Short CRUD-style message sets for scheduled jobs, approval workflows, agent memory, and webhook management. Each follows the same pattern:

```fluent
<entity>-created = ... created: { $id }
<entity>-create-failed = Failed to create ...: { $error }
<entity>-deleted = ... { $id } deleted.
<entity>-delete-failed = Failed to delete ...: { $error }
```

### Models and Config

- **Models** — default model setting, catalog display, selection prompts
- **Config** — get/set/unset operations, key management, env-file persistence, parse error reporting with specific guidance on dotted notation and scalar vs. table distinctions

### Uninstall and Reset

Platform-specific cleanup messages for full uninstall (Windows registry, macOS launch agent, Linux autostart/systemd) and data reset operations.

---

## Variable Reference

Messages use the following variable names across the catalog:

| Variable | Used In | Description |
|----------|---------|-------------|
| `$provider` | `kernel-booted`, `detected-provider` | LLM provider name |
| `$model` | `kernel-booted`, `model-set-success` | Model identifier |
| `$count` | `models-available`, `agents-loaded`, `auth-user-keys` | Numeric count |
| `$url` | `daemon-already-running`, `dashboard-opening` | Daemon/dashboard URL |
| `$error` | Various error/failed messages | Error description string |
| `$status` | `daemon-bg-exited`, `error-daemon-returned` | HTTP/process status code |
| `$path` | `daemon-bg-exited-fix`, `error-create-dir`, `log-following` | Filesystem path |
| `$name` | `agent-spawned`, `channel-configured`, `agent-template-not-found` | Entity name |
| `$id` | `agent-killed`, `cron-created`, `webhook-deleted`, `device-removed` | Entity identifier |
| `$value` | `agent-model-set`, `config-set-kv` | Setting value |
| `$key` | `config-key-not-found`, `vault-stored`, `channel-key-saved` | Config/vault key |
| `$command` | `error-require-daemon` | CLI subcommand name |
| `$action` | `cron-toggled`, `approval-responded` | Past-tense action (e.g., "pause", "approve") |
| `$field` | `agent-unknown-field` | Field name |
| `$display` | `detected-provider` | Provider display name |
| `$env_var` | `detected-provider`, `config-saved-key` | Environment variable name |
| `$max` | `model-out-of-range` | Maximum selection number |
| `$agent` | `memory-set`, `memory-deleted` | Agent identifier |

---

## Adding a New Locale

1. Create a new directory under `librefang-cli/locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g., `ja/` for Japanese).
2. Copy `en/main.ftl` into the new directory.
3. Translate every message value while keeping all keys and variable interpolations (`{ $var }`) unchanged.
4. Register the locale in the CLI's localization initialization code (outside this module).

### Conventions

- **Do not rename keys.** The CLI references keys by exact string match.
- **Preserve all variables.** Missing a `{ $error }` interpolation will cause a runtime panic in Fluent.
- **Keep section comments.** The `# --- Section ---` headers are for human maintainers; keep them consistent across locales for easier diffing.
- **Maintain the `-fix` pairing.** Every error message with a `-fix` companion must have the companion translated as well.
- **Backticks in hints.** Command suggestions are wrapped in backticks in English. Follow the target language's convention for inline code.

---

## Relationship to the CLI Codebase

This module is a leaf dependency. It produces no function calls and receives none. The CLI's localization layer reads these `.ftl` files at runtime, resolves the current locale, and looks up messages by key when rendering output. The call graph is entirely external to this module:

```
CLI command handlers
  └─ Localization resolver (fluent-rs or equivalent)
       └─ Reads locales/<lang>/main.ftl
            └─ Returns resolved string with interpolated variables
```

Because there are no code dependencies, changes to message wording (including across locales) are safe to make independently — as long as keys and variable signatures remain consistent with their call sites in the CLI.