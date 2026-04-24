# Other — librefang-cli-locales

# librefang-cli-locales

Localization resource files for the LibreFang CLI. This module contains all user-facing strings used by the command-line interface, organized as Fluent (`.ftl`) translation catalogs.

## Purpose

The CLI never hardcodes user-visible text. Every label, error message, hint, and status line lives in these Fluent message files, allowing the interface to be presented in the user's preferred language without changing application logic.

## Supported Locales

| Locale | File | Language |
|--------|------|----------|
| `en` | `locales/en/main.ftl` | English (default) |
| `zh-CN` | `locales/zh-CN/main.ftl` | Simplified Chinese |

## File Format

All files use [Project Fluent](https://projectfluent.org/) syntax. A message consists of an identifier, an optional `=` sign, and the translated value:

```fluent
daemon-starting = Starting daemon...
```

Messages can reference variables passed at runtime using `{ $variable }`:

```fluent
kernel-booted = Kernel booted ({ $provider }/{ $model })
```

Some messages have companion `-fix` variants that provide remediation hints displayed alongside errors:

```fluent
error-daemon-comm = Daemon communication error: { $error }
error-daemon-comm-fix = Check `librefang status` or restart: librefang start
```

## Message Categories

The files are internally organized by section comments. Below is the complete category map shared across all locales.

### Daemon Lifecycle

Covers start, stop, restart, background launch, health polling, and error states. Key identifiers:

- `daemon-starting`, `daemon-started`, `daemon-stopped`, `daemon-stopped-ok`
- `daemon-started-bg`, `daemon-still-starting`
- `daemon-bg-exited`, `daemon-bg-exited-fix`, `daemon-bg-wait-fail`, `daemon-bg-wait-fail-fix`
- `daemon-already-running`, `daemon-not-running`
- `daemon-restarting`, `daemon-no-running-starting`, `daemon-no-running-auto`

### Labels

Short nouns used as table headers, field names, and status indicators. These are typically used in structured output (status tables, info sections).

- `label-api`, `label-dashboard`, `label-provider`, `label-model`, `label-pid`, `label-log`
- `label-status`, `label-agents`, `label-data-dir`, `label-uptime`, `label-version`, `label-daemon`
- `label-id`, `label-active-agents`, `label-pairing-code`, `label-expires`
- `label-home`, `label-platform`, `label-sessions`, `label-memory`, `label-started`
- `label-response`, `label-checks`, `label-auth`, `label-mcp`, `label-peers`
- `label-channels`, `label-skills`, `label-hands`, `label-config-warnings`

### Hints

Actionable guidance displayed after commands or in interactive flows. These often include inline command examples.

- `hint-open-dashboard`, `hint-stop-daemon`, `hint-check-status`
- `hint-start-daemon`, `hint-start-daemon-cmd`, `hint-or-chat`
- `hint-non-interactive`, `hint-non-interactive-wizard`
- `hint-no-api-keys`, `hint-groq-free`, `hint-ollama-local`, `hint-gemini-free`, `hint-deepseek-free`
- `hint-doctor-repair`, `hint-run-init`, `hint-run-start`, `hint-config-edit`
- `hint-set-key`, `hint-set-key-provider`

### Interactive Guide

Strings for the first-run setup wizard (`librefang init`). Covers provider selection, API key pasting, and verification feedback.

- `guide-title`, `guide-free-providers-title`, `guide-get-free-key`
- `guide-paste-key-placeholder`, `guide-paste-key-hint`
- `guide-setting-up`, `guide-testing-key`, `guide-key-verified`, `guide-test-key-unverified`
- `guide-help-select`, `guide-help-paste`, `guide-help-wait`

### Init

Completion messages for the initialization flow.

- `init-quick-success`, `init-interactive-success`, `init-cancelled`
- `init-next-start`, `init-next-chat`

### Error Messages

General filesystem and I/O errors:

- `error-home-dir`, `error-create-dir`, `error-create-dir-fix`
- `error-write-config`, `error-config-created`, `error-config-exists`

Daemon communication errors:

- `error-daemon-returned`, `error-daemon-returned-fix`
- `error-request-timeout`, `error-request-timeout-fix`
- `error-connect-refused`, `error-connect-refused-fix`
- `error-daemon-comm`, `error-daemon-comm-fix`

Boot errors:

- `error-boot-config`, `error-boot-config-fix`
- `error-boot-db`, `error-boot-db-fix`
- `error-boot-auth`, `error-boot-auth-fix`
- `error-boot-generic`, `error-boot-generic-fix`

Daemon requirement guard:

- `error-require-daemon`, `error-require-daemon-fix`

### Provider Detection

Auto-detection feedback shown at startup.

- `detected-provider`, `detected-gemini`, `detected-ollama`

### Desktop App

Messages for launching the optional desktop GUI.

- `desktop-launching`, `desktop-started`, `desktop-launch-fail`, `desktop-not-found`

### Agent Commands

All agent lifecycle messages: spawning, killing, model assignment, and template handling.

- `agent-spawned`, `agent-spawned-inprocess`, `agent-spawn-failed`, `agent-spawn-agent-failed`
- `agent-killed`, `agent-kill-failed`, `agent-invalid-id`
- `agent-model-set`, `agent-set-model-failed`, `agent-unknown-field`
- `agent-no-agents`, `agent-no-daemon-for-set`
- `agent-template-not-found`, `agent-template-not-found-fix`
- `agent-no-templates`, `agent-no-templates-fix`
- `agent-template-parse-fail`, `agent-template-parse-fail-fix`
- `section-agent-templates`

### Manifest Errors

- `manifest-not-found`, `manifest-not-found-fix`
- `error-reading-manifest`, `error-parsing-manifest`

### Status Display

Section headers and field values for `librefang status` output.

- `section-daemon-status`, `section-status-inprocess`
- `section-active-agents`, `section-persisted-agents`
- `label-daemon-not-running`, `section-status-locked`
- `section-recent-errors`, `section-verbose`
- `auth-none`, `auth-api-key`, `auth-dashboard-login`, `auth-user-keys`
- `warn-public-bind`, `warn-key-missing`

### Doctor

Diagnostic and repair output.

- `doctor-title`, `doctor-all-passed`, `doctor-repairs-applied`, `doctor-some-failed`
- `doctor-no-api-keys`, `section-getting-api-key`

### Security

Security posture display (audit trail, sandboxing, wire protocol).

- `section-security-status`
- `label-audit-trail`, `label-taint-tracking`, `label-wasm-sandbox`, `label-wire-protocol`
- `value-audit-trail`, `value-taint-tracking`, `value-wasm-sandbox`, `value-wire-protocol`
- `value-api-keys`, `value-manifests`
- `audit-verified`, `audit-failed`

### Health

- `health-ok`, `health-not-running`

### Channel Setup

Messaging platform configuration flows (Telegram, Discord, Slack, WhatsApp, Email, Signal, Matrix).

- `section-channel-setup`, `channel-configured`, `channel-no-token`, `channel-no-email`
- `channel-token-saved`, `channel-app-token-saved`, `channel-bot-token-saved`
- `channel-password-saved`, `channel-phone-saved`, `channel-key-saved`
- `channel-unknown`, `channel-unknown-fix`
- `channel-test-ok`, `channel-test-fail`
- `section-setup-telegram`, `section-setup-discord`, `section-setup-slack`, `section-setup-whatsapp`
- `section-setup-email`, `section-setup-signal`, `section-setup-matrix`

### Vault

Encrypted credential storage messages.

- `vault-initialized`, `vault-not-initialized`, `vault-not-init-run`
- `vault-unlock-failed`, `vault-empty-value`
- `vault-stored`, `vault-store-failed`, `vault-removed`, `vault-key-not-found`, `vault-remove-failed`

### Cron

Scheduled job management.

- `cron-created`, `cron-create-failed`
- `cron-deleted`, `cron-delete-failed`
- `cron-toggled`, `cron-toggle-failed`

### Approvals

Human-in-the-loop approval workflow.

- `approval-responded`, `approval-failed`

### Memory

Agent persistent memory key-value operations.

- `memory-set`, `memory-set-failed`, `memory-deleted`, `memory-delete-failed`

### Devices

Mobile device pairing via QR code.

- `section-device-pairing`, `device-scan-qr`
- `device-removed`, `device-remove-failed`

### Webhooks

Webhook CRUD and test operations.

- `webhook-created`, `webhook-create-failed`
- `webhook-deleted`, `webhook-delete-failed`
- `webhook-test-ok`, `webhook-test-failed`

### Models

Model selection and catalog.

- `model-set-success`, `model-set-failed`, `model-no-catalog`
- `section-select-model`, `model-out-of-range`

### Config

Configuration file operations: get, set, unset, set-key.

- `config-set-success`, `config-unset-success`
- `config-no-file`, `config-no-file-fix`
- `config-read-failed`, `config-parse-error`, `config-parse-fix`, `config-parse-fix-alt`
- `config-key-not-found`, `config-key-path-not-found`, `config-empty-key`
- `config-section-not-scalar`, `config-section-not-scalar-fix`
- `config-parent-not-table`, `config-serialize-failed`, `config-write-failed`
- `config-set-kv`, `config-removed-key`, `config-no-key`
- `config-saved-key`, `config-save-key-failed`
- `config-removed-env`, `config-remove-key-failed`, `config-env-not-set`
- `config-set-key-hint`, `config-update-key-hint`

### Hands

Tool/hand instance lifecycle.

- `hand-install-deps-success`, `hand-paused`, `hand-resumed`

### Uninstall

Full uninstall flow covering daemon stop, file removal, and platform-specific autostart cleanup (Windows registry, macOS launch agent, Linux systemd/autostart).

- `uninstall-goodbye`, `uninstall-cancelled`, `uninstall-stopping-daemon`
- `uninstall-removed`, `uninstall-remove-failed`, `uninstall-removed-data-kept`
- `uninstall-removed-autostart-win`, `uninstall-removed-launch-agent`, `uninstall-remove-launch-fail`
- `uninstall-removed-autostart-linux`, `uninstall-remove-autostart-fail`
- `uninstall-removed-systemd`, `uninstall-remove-systemd-fail`
- `uninstall-cleaned-path`, `uninstall-cleaned-path-win`

### Reset

Selective data removal.

- `reset-success`, `reset-fail`

### Logs

Log tailing output.

- `log-following`, `log-path-hint`

## How the CLI Consumes These Messages

The CLI loads Fluent bundles at startup based on the user's locale (typically derived from environment variables like `LANG`, `LC_ALL`, or an explicit `--locale` flag). Code references messages by their Fluent identifier:

```
// Conceptual usage (actual API varies by Fluent bindings)
bundle.getMessage("daemon-starting")        // → "Starting daemon..."
bundle.getMessage("kernel-booted", { provider: "groq", model: "llama3" })
```

The `-fix` suffix convention pairs every remediatable error with an actionable suggestion. The CLI renders them together:

```
Daemon communication error: connection refused
Check `librefang status` or restart: librefang start
```

## Adding a New Locale

1. Create a directory under `locales/` with the appropriate locale code (e.g. `locales/ja/`).
2. Copy `locales/en/main.ftl` into the new directory.
3. Translate all right-hand-side values. **Do not** change message identifiers or variable names.
4. Keep the section comment structure intact for maintainability.
5. Register the new locale in the CLI's Fluent bundle loader.

## Adding a New Message

1. Add the message to **every** locale file, not just `en/main.ftl`. If a translation is not yet available, use the English text as a placeholder — missing identifiers cause runtime fallback errors.
2. Place it under the correct section comment.
3. Follow existing naming conventions:
   - `kebab-case` identifiers
   - `-fix` suffix for remediation hints
   - `section-` prefix for UI section headers
   - `label-` prefix for short field labels
   - `hint-` prefix for guidance text
   - `error-` prefix for error messages
   - `warn-` prefix for warnings
   - `value-` prefix for descriptive values paired with labels

## Conventions

| Pattern | Meaning | Example |
|---------|---------|---------|
| `{ $var }` | Runtime variable interpolation | `daemon-error = Error: { $error }` |
| `-fix` suffix | Remediation hint paired with an error | `error-connect-refused-fix` |
| `section-` prefix | Section heading in TUI output | `section-daemon-status` |
| `label-` prefix | Short noun for tables/lists | `label-uptime` |
| `hint-` prefix | Contextual guidance | `hint-stop-daemon` |
| `error-` prefix | Error condition | `error-boot-auth` |
| `warn-` prefix | Warning condition | `warn-public-bind` |
| `value-` prefix | Descriptive text paired with a label | `value-wasm-sandbox` |
| `guide-` prefix | First-run wizard content | `guide-paste-key-hint` |

All files within a single locale use `main.ftl` as the sole entry point. There is no message splitting across multiple files per locale.