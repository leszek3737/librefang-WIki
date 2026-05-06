# Other — librefang-types-locales

# librefang-types-locales

Fluent (FTL) localization resources for all LibreFang API error messages. This module is a pure data layer — it contains no executable code, only `.ftl` translation files consumed by the Fluent localization system at runtime.

## Overview

Every API-facing error in LibreFang is assigned a stable Fluent message identifier (e.g., `api-error-agent-not-found`). The corresponding human-readable text lives in per-language `.ftl` files under `librefang-types/locales/<lang>/errors.ftl`. Application code references the identifier; the Fluent runtime resolves it to the correct language string.

## Supported Languages

| Locale | Code | Completeness |
|---|---|---|
| English | `en` | Full — canonical source of truth |
| Japanese | `ja` | Full |
| German | `de` | Core subset |
| Spanish | `es` | Core subset |
| French | `fr` | Core subset |
| Simplified Chinese | `zh-CN` | Core subset |

English (`en`) defines the complete set of message identifiers. The other full-coverage locale (`ja`) mirrors it nearly one-to-one. The core-subset locales cover the foundational error domains (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, and general errors).

## File Structure

```
librefang-types/locales/
├── de/errors.ftl
├── en/errors.ftl
├── es/errors.ftl
├── fr/errors.ftl
├── ja/errors.ftl
└── zh-CN/errors.ftl
```

Each file follows identical structure: grouped by error domain, with section headers as comments.

## Error Domains

Messages are prefixed by domain using the naming convention `api-error-<domain>-<detail>`:

| Domain | Prefix | Examples |
|---|---|---|
| Agent | `api-error-agent-*` | `agent-not-found`, `agent-spawn-failed`, `agent-vanished` |
| Message | `api-error-message-*` | `message-too-large`, `message-delivery-failed` |
| Template | `api-error-template-*` | `template-not-found`, `template-parse-failed` |
| Manifest | `api-error-manifest-*` | `manifest-too-large`, `manifest-signature-mismatch` |
| Auth | `api-error-auth-*` | `auth-invalid-key`, `auth-missing-header` |
| Session | `api-error-session-*` | `session-not-found`, `session-invalid-id` |
| Workflow | `api-error-workflow-*` | `workflow-missing-steps`, `workflow-execution-failed` |
| Trigger | `api-error-trigger-*` | `trigger-invalid-pattern`, `trigger-registration-failed` |
| Budget | `api-error-budget-*` | `budget-invalid-amount`, `budget-update-failed` |
| Config | `api-error-config-*` | `config-parse-failed`, `config-write-failed` |
| Profile | `api-error-profile-*` | `profile-not-found` |
| Cron | `api-error-cron-*` | `cron-invalid-expression`, `cron-create-failed` |
| Goal | `api-error-goal-*` | `goal-circular-parent`, `goal-progress-range` |
| Memory | `api-error-memory-*` | `memory-not-enabled`, `memory-missing-kv` |
| Network / A2A | `api-error-network-*` | `network-connection-failed`, `network-a2a-not-found` |
| Plugin | `api-error-plugin-*` | `plugin-missing-name`, `plugin-invalid-source` |
| Channel | `api-error-channel-*` | `channel-invalid-from`, `channel-invalid-to` |
| Provider | `api-error-provider-*` | `provider-alias-exists`, `provider-key-not-configured` |
| Skill | `api-error-skill-*` | `skill-invalid-name`, `skill-install-failed` |
| Hand | `api-error-hand-*` | `hand-not-found`, `hand-instance-not-found` |
| MCP | `api-error-mcp-*` | `mcp-invalid-config`, `mcp-not-found` |
| Integration / Extension | `api-error-integration-*`, `api-error-extension-*` | `integration-not-found` |
| System | `api-error-system-*` | `system-cli-not-found` |
| KV | `api-error-kv-*` | `kv-missing-fields`, `kv-array-empty` |
| Approval | `api-error-approval-*` | `approval-not-found` |
| Webhook | `api-error-webhook-*` | `webhook-url-unreachable`, `webhook-unknown-event` |
| Backup | `api-error-backup-*` | `backup-missing-manifest`, `backup-invalid-archive` |
| Schedule | `api-error-schedule-*` | `schedule-invalid-cron`, `schedule-save-failed` |
| Job | `api-error-job-*` | `job-not-retryable`, `job-disappeared-cancel` |
| Task | `api-error-task-*` | `task-not-found`, `task-disappeared` |
| Pairing | `api-error-pairing-*` | `pairing-not-enabled`, `pairing-invalid-token` |
| Binding | `api-error-binding-*` | `binding-out-of-range` |
| Command | `api-error-command-*` | `command-not-found` |
| File / Upload | `api-error-file-*` | `file-path-traversal`, `file-unsupported-type` |
| Tool | `api-error-tool-*` | `tool-invoke-denied`, `tool-requires-agent` |
| Validation | `api-error-validation-*` | `validation-color-invalid`, `validation-avatar-url-invalid` |
| General | `api-error-*` (no domain) | `not-found`, `internal`, `bad-request`, `rate-limited` |

## Fluent Variable Interpolation

Many messages include Fluent variables that are populated at runtime. The available variables are:

| Variable | Used in | Description |
|---|---|---|
| `{ $reason }` | `message-delivery-failed`, `bad-request` | Reason text for failure |
| `{ $error }` | `template-parse-failed`, `config-parse-failed`, `config-write-failed`, `cron-create-failed`, `manifest-invalid`, `agent-execution-failed`, `agent-clone-spawn-failed`, `agent-error`, `goal-save-failed`, `network-connection-failed`, `network-task-post-failed`, `provider-secrets-write-failed`, `skill-dir-create-failed`, `mcp-invalid-config`, `backup-*`, `schedule-*` | Error detail from underlying operation |
| `{ $name }` | `template-not-found`, `profile-not-found`, `tool-not-found`, `command-not-found`, `mcp-not-found`, `provider-not-found`, `provider-unknown` | Name of the requested resource |
| `{ $id }` | `agent-not-found-with-id`, `goal-not-found-with-id`, `hand-not-found`, `integration-not-found`, `extension-not-found` | Identifier of the requested resource |
| `{ $step }` | `workflow-step-needs-agent` | Step name in a workflow |
| `{ $alias }` | `provider-alias-exists`, `provider-alias-not-found` | Provider alias |
| `{ $url }` | `network-a2a-not-found`, `network-missing-url` | URL parameter |
| `{ $status }` | `network-auth-failed` | HTTP status code |
| `{ $max }` | `skill-description-too-long`, `file-too-large` | Maximum allowed value |
| `{ $event }` | `webhook-unknown-event` | Event type string |
| `{ $valid }` | `agent-invalid-sort`, `webhook-unknown-event` | List of valid values |
| `{ $field }` | `agent-invalid-sort` | Field name that was invalid |
| `{ $provider }` | `provider-model-exists` | Provider name |

## Adding a New Error Message

1. Add the message to `en/errors.ftl` first — this is the canonical source. Place it under the appropriate domain comment heading.
2. Use the naming convention `api-error-<domain>-<detail>` with kebab-case.
3. If the message needs runtime data, use Fluent variables: `{ $variable-name }`.
4. Mirror the message to `ja/errors.ftl` for full coverage.
5. Add the message to any core-subset locales (`de`, `es`, `fr`, `zh-CN`) if it falls within their covered domains, or leave it for a future translation pass. Untranslated messages fall back to English via Fluent's built-in fallback mechanism.

Example entry in `en/errors.ftl`:

```ftl
api-error-example-domain-new-error = Description of the new error with { $detail }
```

## Adding a New Locale

1. Create the directory `librefang-types/locales/<code>/`.
2. Create `errors.ftl` with the same message identifiers as `en/errors.ftl`.
3. Translate the message values. Do **not** translate the identifiers (the left-hand side of `=`).
4. Preserve all Fluent variables exactly — `{ $reason }` must remain `{ $reason }`.
5. Keep the same section comment structure for maintainability.

## Integration with Application Code

This module is a passive data dependency. The consuming application:

1. Loads the Fluent bundle for the negotiated locale at startup.
2. On error, calls the Fluent bundle with the message ID (e.g., `api-error-agent-not-found`) and any required variables.
3. If the message ID is missing from the active locale, Fluent falls back through the locale chain, ultimately reaching English.

Because these are `.ftl` files bundled at compile time or loaded at runtime, no Rust code in this crate performs any logic — the crate exists solely to co-locate and version the translation resources alongside the type definitions in `librefang-types`.