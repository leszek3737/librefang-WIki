# Other — librefang-types-locales

# librefang-types/locales — API Error Message Localizations

## Overview

This module contains the internationalized API error message catalogs for the LibreFang platform. It is a **pure resource module** — no executable Rust code, no functions, no structs — just Fluent (`.ftl`) translation files organized by locale. These files are loaded at runtime by the Fluent localization system and referenced across the HTTP API layer whenever an error response is returned to a client.

## Supported Locales

| Locale | Code | File | Completeness |
|---|---|---|---|
| English | `en` | `en/errors.ftl` | **Full** — reference/master catalog |
| Japanese | `ja` | `ja/errors.ftl` | Full |
| German | `de` | `de/errors.ftl` | Partial |
| French | `fr` | `fr/errors.ftl` | Partial |
| Spanish | `es` | `es/errors.ftl` | Partial |
| Simplified Chinese | `zh-CN` | `zh-CN/errors.ftl` | Partial |

The **English** catalog is the authoritative source. It defines every error key in the system. Other locales translate the core subset of keys. When a key is missing from a non-English locale, the Fluent framework falls back to English (depending on the runtime fallback configuration).

## File Format

All files use [Project Fluent](https://projectfluent.org/) (`.ftl`) syntax. Each entry is a key-value pair with optional interpolation variables:

```ftl
# Simple message — no variables
api-error-agent-not-found = Agent not found

# Interpolated message — variables from runtime context
api-error-message-delivery-failed = Message delivery failed: { $reason }
api-error-template-not-found = Template '{ $name }' not found
```

### Naming Convention

Keys follow a strict prefix pattern:

```
api-error-<domain>-<description>
```

The domain maps to a functional area of the API. The description is a kebab-case phrase identifying the specific error condition.

## Error Domains

The English catalog organizes keys into these domains:

| Domain | Prefix | Example Keys | Purpose |
|---|---|---|---|
| Agent | `api-error-agent-*` | `not-found`, `spawn-failed`, `invalid-id`, `no-workspace`, `execution-failed` | Agent lifecycle, spawning, lookup |
| Message | `api-error-message-*` | `too-large`, `delivery-failed`, `required` | Inter-agent messaging |
| Template | `api-error-template-*` | `invalid-name`, `not-found`, `parse-failed` | Agent template management |
| Manifest | `api-error-manifest-*` | `too-large`, `invalid-format`, `signature-mismatch` | Template manifest validation |
| Auth | `api-error-auth-*` | `invalid-key`, `missing-header`, `missing` | API key authentication |
| Session | `api-error-session-*` | `load-failed`, `not-found`, `invalid-id` | Conversation sessions |
| Workflow | `api-error-workflow-*` | `missing-steps`, `execution-failed`, `not-found` | Multi-step workflows |
| Trigger | `api-error-trigger-*` | `missing-agent-id`, `invalid-pattern`, `not-found` | Event triggers |
| Budget | `api-error-budget-*` | `invalid-amount`, `update-failed`, `provide-at-least-one` | Cost budgets |
| Config | `api-error-config-*` | `parse-failed`, `write-failed`, `save-failed` | Runtime configuration |
| Profile | `api-error-profile-*` | `not-found` | Agent profiles |
| Cron | `api-error-cron-*` | `invalid-id`, `create-failed`, `invalid-expression` | Scheduled jobs |
| Goal | `api-error-goal-*` | `not-found`, `missing-title`, `circular-parent` | Goal tracking and hierarchy |
| Memory | `api-error-memory-*` | `not-enabled`, `not-found`, `export-failed` | Proactive memory / KV store |
| Network | `api-error-network-*` | `not-enabled`, `peer-not-found`, `a2a-not-found` | Peer-to-peer / A2A networking |
| Plugin | `api-error-plugin-*` | `missing-name`, `missing-path`, `invalid-source` | Plugin installation |
| Channel | `api-error-channel-*` | `unknown`, `missing-agent-id`, `invalid-from` | Agent communication channels |
| Provider | `api-error-provider-*` | `missing-alias`, `model-not-found`, `key-not-configured` | LLM provider management |
| Skill | `api-error-skill-*` | `missing-name`, `invalid-name`, `install-failed` | Skill creation and management |
| Hand | `api-error-hand-*` | `not-found`, `definition-not-found` | Hand/gesture definitions |
| MCP | `api-error-mcp-*` | `missing-name`, `invalid-config`, `not-found` | Model Context Protocol servers |
| Integration | `api-error-integration-*` | `not-found`, `missing-id` | Third-party integrations |
| Extension | `api-error-extension-*` | `not-found` | Platform extensions |
| System | `api-error-system-*` | `cli-not-found` | System-level errors |
| KV | `api-error-kv-*` | `missing-fields`, `missing-value`, `array-empty` | Structured key-value data |
| Approval | `api-error-approval-*` | `invalid-id`, `not-found` | Human approval workflows |
| Webhook | `api-error-webhook-*` | `not-enabled`, `missing-url`, `url-unreachable` | Webhook event delivery |
| Backup | `api-error-backup-*` | `not-found`, `missing-manifest`, `finalize-failed` | Backup creation and restore |
| Schedule | `api-error-schedule-*` | `not-found`, `invalid-cron`, `save-failed` | Scheduled task management |
| Job | `api-error-job-*` | `invalid-id`, `not-found`, `not-retryable` | Background job execution |
| Task | `api-error-task-*` | `not-found`, `disappeared` | Task tracking |
| Pairing | `api-error-pairing-*` | `not-enabled`, `invalid-token` | Device/session pairing |
| Binding | `api-error-binding-*` | `out-of-range` | Key/port binding |
| Command | `api-error-command-*` | `not-found` | CLI-style commands |
| File | `api-error-file-*` | `not-found`, `too-large`, `path-traversal` | File upload/download |
| Tool | `api-error-tool-*` | `not-found`, `invoke-disabled`, `invoke-denied` | Tool invocation and access control |
| Validation | `api-error-validation-*` | `content-empty`, `title-required`, `color-invalid` | Input validation |
| General | `api-error-*` (no domain) | `not-found`, `internal`, `bad-request`, `rate-limited`, `generic` | Catch-all and HTTP-level errors |

## The Generic Catch-All Key

The `api-error-generic` key serves a special role. It is used by **41+ HTTP 500 handlers** as a temporary stopgap:

```ftl
api-error-generic = { $error }
```

It interpolates the underlying error string verbatim via `t_args("api-error-generic", …)`. Without this key, any untyped 500 handler would return the literal key name as the response body, breaking the `$error` interpolation entirely. This key exists in all six locales and must not be removed until every route migrates to a typed `MemoryRouteError`-style helper.

## Interpolation Variables

Keys that include runtime context use Fluent variable syntax `{ $variable_name }`. The following variables appear across the catalogs:

| Variable | Used By | Meaning |
|---|---|---|
| `$reason` | `bad-request`, `message-delivery-failed` | Human-readable failure reason |
| `$error` | `*-failed`, `*-parse-failed`, `generic` | Underlying error message from the system |
| `$name` | `template-not-found`, `profile-not-found`, `provider-alias-*` | Resource name or identifier |
| `$id` | `agent-not-found-with-id`, `goal-not-found-with-id`, `hand-not-found` | Resource ID |
| `$step` | `workflow-step-needs-agent` | Workflow step name |
| `$field` | `agent-invalid-sort` | Invalid field name |
| `$valid` | `agent-invalid-sort`, `webhook-unknown-event` | List of valid values |
| `$max` | `file-too-large`, `skill-description-too-long` | Maximum allowed value |
| `$status` | `network-auth-failed` | HTTP status code |
| `$url` | `network-a2a-not-found` | URL string |
| `$alias` | `provider-alias-exists`, `provider-alias-not-found` | Provider alias name |
| `$provider` | `provider-model-exists` | Provider name |
| `$event` | `webhook-unknown-event` | Event type name |

## Adding a New Error Message

1. **Add the key to `en/errors.ftl` first.** This is the master catalog. Follow the naming convention `api-error-<domain>-<description>`.

2. **Add translations to all other locale files.** At minimum, add the key to `de/`, `es/`, `fr/`, `ja/`, and `zh-CN/`. If you cannot provide a translation immediately, copy the English text as a placeholder and mark it with a `TODO` comment.

3. **If the key uses interpolation**, ensure the variable name matches exactly across all locales. Fluent variable names (`$error`, `$name`, etc.) are not translated — they are identifiers.

4. **Reference the key from your Rust code** using the Fluent helper (typically `t("api-error-...")` or `t_args("api-error-...", args)`).

## Adding a New Locale

1. Create a new directory under `locales/` with the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g., `pt-BR/`).

2. Copy `en/errors.ftl` into the new directory and translate all message values. Do **not** translate the keys or variable names.

3. Register the new locale in the Fluent loader configuration (outside this module).

## Locale Completeness

The partial locales (`de`, `es`, `fr`, `zh-CN`) cover a core subset of keys (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, and general errors). They do **not** yet include translations for: goal, memory, network, plugin, channel, provider, skill, hand, MCP, integration, extension, system, KV, approval, webhook, backup, schedule, job, task, pairing, binding, command, file, tool, or validation errors.

When extending these locales, use the English catalog as the reference for which keys exist and what variables they expect.