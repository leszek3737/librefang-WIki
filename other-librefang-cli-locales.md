# Other — librefang-cli-locales

# LibreFang CLI Locales

Localization resources for the `librefang-cli` crate, written in [Project Fluent](https://projectfluent.org/) (`.ftl`) format. This module is a static translation layer — no executable logic — consumed at runtime by the CLI to display user-facing messages in the user's preferred language.

## Locale Inventory

| Locale | Directory | Status |
|--------|-----------|--------|
| English (`en`) | `locales/en/main.ftl` | Source of truth |
| Simplified Chinese (`zh-CN`) | `locales/zh-CN/main.ftl` | Complete translation |

All locales are single-file (`main.ftl`). Additional locales follow the same `locales/<BCP47-tag>/main.ftl` convention.

## Message Format

Messages use Fluent syntax with variable interpolation:

```fluent
# Simple string
daemon-stopped = LibreFang daemon stopped.

# String with positional variable
kernel-booted = Kernel booted ({ $provider }/{ $model })

# Pluralization-ready count
agents-loaded = { $count } agent(s) loaded
```

**Naming convention:** `category-description` or `category-description-fix`.

- `*-fix` variants are remediation hints shown alongside the primary error (e.g., `error-boot-config` + `error-boot-config-fix`).
- `section-*` messages are UI section headers.
- `hint-*` messages are contextual guidance shown after an action.
- `label-*` messages are short field labels for status tables.
- `warn-*` messages flag configuration problems.
- `value-*` messages provide human-readable descriptions for security features.

## Message Categories

The ~250 messages in each locale file are organized into the following groups:

### Daemon Lifecycle
Startup, shutdown, background launch, health waits, and restart flows. Key messages: `daemon-starting`, `daemon-started-bg`, `daemon-bg-exited`, `daemon-restarting`.

### Error Hierarchy
Three tiers of errors, each with an accompanying `*-fix` message:

| Tier | Prefix | Example |
|------|--------|---------|
| General | `error-*` | `error-write-config` |
| Daemon communication | `error-daemon-*`, `error-request-*`, `error-connect-*` | `error-daemon-returned` |
| Boot failures | `error-boot-*` | `error-boot-auth` |

A `*-fix` companion always follows its parent error, providing an actionable command or diagnostic step.

### Agent Management
Spawn, kill, model assignment, and template resolution: `agent-spawned`, `agent-template-not-found`, `agent-killed`.

### Configuration
Read/write validation, key management, and `.env` operations: `config-set-success`, `config-parse-error`, `config-saved-key`.

### Channel Setup
Provider-specific onboarding for Telegram, Discord, Slack, WhatsApp, Email, Signal, and Matrix: `section-setup-telegram`, `channel-configured`, `channel-test-ok`.

### Infrastructure Features
- **Vault** — credential storage: `vault-initialized`, `vault-stored`
- **Cron** — scheduled jobs: `cron-created`, `cron-toggled`
- **Webhooks** — outbound integrations: `webhook-created`, `webhook-test-ok`
- **Memory** — per-agent key/value store: `memory-set`, `memory-deleted`
- **Devices** — mobile pairing: `device-scan-qr`, `device-removed`
- **Approvals** — human-in-the-loop gating: `approval-responded`

### Security & Diagnostics
- **Security status** — audit trail, taint tracking, WASM sandbox, wire protocol: `section-security-status`, `audit-verified`
- **Doctor** — health checks and auto-repair: `doctor-title`, `doctor-all-passed`
- **Uninstall** — platform-aware cleanup (macOS launch agent, Windows registry, Linux systemd): `uninstall-goodbye`, `uninstall-removed-systemd`

## Adding a New Locale

1. Create `locales/<tag>/main.ftl` (e.g., `locales/ja/main.ftl`).
2. Copy `locales/en/main.ftl` as the starting template.
3. Translate every message value. **Do not** change message identifiers or variable names (`$provider`, `$error`, etc.).
4. Keep `*-fix` messages paired with their parent errors and preserve the suggested commands (CLI flags and subcommands are not translated).

Example — translating a single entry:

```fluent
# English (source)
daemon-started = Daemon started

# Japanese (new locale)
daemon-started = デーモンが起動しました
```

## Adding New Messages

When adding CLI features that produce user-facing output:

1. Add the message to `locales/en/main.ftl` under the appropriate section comment.
2. Immediately add the same identifier to all other locale files (untranslated is acceptable short-term; untranslated keys fall back to English at runtime).
3. Follow the naming patterns: `error-*-fix` for error remediation, `section-*` for headings, `hint-*` for guidance.
4. Use Fluent variables (`{ $name }`) for any dynamic content — never hard-code interpolated values into the string.

## Relationship to the CLI

This module is a resource consumed by `librefang-cli` via a Fluent localization bundle. The CLI initializes the bundle with the user's locale (defaulting to `en`) and looks up messages by their Fluent identifier. Because the `.ftl` files contain no logic, the module has no call graph — it is read at runtime through the Fluent API and rendered into terminal output, status tables, and error reports.