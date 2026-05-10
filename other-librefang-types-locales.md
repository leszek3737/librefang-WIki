# Other — librefang-types-locales

# librefang-types-locales

Localized API error message definitions for the LibreFang platform, using [Project Fluent](https://projectfluent.org/) (`.ftl`) format. This module is a static resource bundle consumed at runtime by the Fluent localization system to render human-readable error messages in the user's preferred language.

## Directory Layout

```
librefang-types/locales/
├── de/errors.ftl      # German
├── en/errors.ftl      # English  (canonical / most complete)
├── es/errors.ftl      # Spanish
├── fr/errors.ftl      # French
├── ja/errors.ftl      # Japanese
└── zh-CN/errors.ftl   # Simplified Chinese
```

Each file contains the same set of message identifiers. The **English** locale (`en`) is the authoritative source — other locales may contain a subset of keys and will fall back to English for any missing entry.

## Message Identifiers

All identifiers follow the naming convention:

```
api-error-{domain}-{specific-error}
```

Domains and their associated identifiers:

| Domain | Example Identifiers | Purpose |
|---|---|---|
| Agent | `api-error-agent-not-found`, `api-error-agent-spawn-failed` | Agent lifecycle, lookup, execution |
| Message | `api-error-message-too-large`, `api-error-message-delivery-failed` | Inter-agent messaging |
| Template | `api-error-template-not-found`, `api-error-template-parse-failed` | Agent template management |
| Manifest | `api-error-manifest-too-large`, `api-error-manifest-signature-mismatch` | Template manifest validation |
| Auth | `api-error-auth-invalid-key`, `api-error-auth-missing-header` | API key authentication |
| Session | `api-error-session-not-found`, `api-error-session-load-failed` | Session management |
| Workflow | `api-error-workflow-missing-steps`, `api-error-workflow-execution-failed` | Workflow orchestration |
| Trigger | `api-error-trigger-missing-pattern`, `api-error-trigger-not-found` | Event trigger registration |
| Budget | `api-error-budget-invalid-amount`, `api-error-budget-update-failed` | Cost budget enforcement |
| Config | `api-error-config-parse-failed`, `api-error-config-write-failed` | Runtime configuration |
| Profile | `api-error-profile-not-found` | Provider profiles |
| Cron | `api-error-cron-create-failed`, `api-error-cron-invalid-expression` | Scheduled tasks |
| Goal | `api-error-goal-not-found`, `api-error-goal-circular-parent` | Goal tracking with hierarchy |
| Memory | `api-error-memory-not-found`, `api-error-memory-missing-kv` | Agent persistent memory |
| Network | `api-error-network-connection-failed`, `api-error-network-a2a-not-found` | Peer networking / A2A protocol |
| Plugin | `api-error-plugin-missing-name`, `api-error-plugin-invalid-source` | Plugin installation |
| Channel | `api-error-channel-unknown`, `api-error-channel-invalid-from` | Agent communication channels |
| Provider | `api-error-provider-alias-exists`, `api-error-provider-model-not-found` | LLM provider configuration |
| Skill | `api-error-skill-missing-name`, `api-error-skill-install-failed` | Custom skill management |
| Hand | `api-error-hand-not-found`, `api-error-hand-instance-not-found` | Hand/tool definitions |
| MCP | `api-error-mcp-missing-name`, `api-error-mcp-invalid-config` | Model Context Protocol servers |
| Integration | `api-error-integration-not-found` | Third-party integrations |
| Extension | `api-error-extension-not-found` | Platform extensions |
| System | `api-error-system-cli-not-found` | System-level operations |
| KV | `api-error-kv-missing-fields`, `api-error-kv-missing-path` | Key-value / structured memory |
| Approval | `api-error-approval-not-found` | Human-approval workflows |
| Webhook | `api-error-webhook-not-enabled`, `api-error-webhook-url-unreachable` | Outbound webhook triggers |
| Backup | `api-error-backup-not-found`, `api-error-backup-missing-manifest` | System backup/restore |
| Schedule | `api-error-schedule-invalid-cron`, `api-error-schedule-save-failed` | Recurring schedules |
| Job | `api-error-job-not-found`, `api-error-job-not-retryable` | Background job processing |
| Task | `api-error-task-not-found`, `api-error-task-disappeared` | Task lifecycle |
| Pairing | `api-error-pairing-not-enabled` | Device/token pairing |
| Binding | `api-error-binding-out-of-range` | Input binding indices |
| Command | `api-error-command-not-found` | CLI-style commands |
| File | `api-error-file-path-traversal`, `api-error-file-unsupported-type` | File upload/download |
| Tool | `api-error-tool-not-found`, `api-error-tool-invoke-disabled` | Direct tool invocation |
| Validation | `api-error-validation-title-required`, `api-error-validation-color-invalid` | Input field validation |
| General | `api-error-not-found`, `api-error-internal`, `api-error-rate-limited` | Catch-all HTTP errors |

## Fluent Variable Interpolation

Many messages accept parameters using Fluent's `{$variable}` syntax. The runtime caller is responsible for passing these values when resolving a message.

Common variables:

| Variable | Used By | Meaning |
|---|---|---|
| `$reason` | `bad-request`, `message-delivery-failed` | Human-readable reason string |
| `$error` | `*-parse-failed`, `*-create-failed`, many others | Underlying error detail |
| `$name` | `template-not-found`, `profile-not-found`, `command-not-found` | Resource name |
| `$id` | `agent-not-found-with-id`, `goal-not-found-with-id` | Resource identifier |
| `$step` | `workflow-step-needs-agent` | Workflow step label |
| `$alias` | `provider-alias-exists`, `provider-alias-not-found` | Provider alias |
| `$url` | `network-a2a-not-found`, `network-missing-url` | Endpoint URL |
| `$status` | `network-auth-failed` | HTTP status code |
| `$max` | `skill-description-too-long`, `file-too-large` | Size limit |
| `$field` | `agent-invalid-sort` | Field name |
| `$valid` | `agent-invalid-sort`, `webhook-unknown-event` | Valid options list |
| `$provider` | `provider-model-exists` | Provider name |
| `$event` | `webhook-unknown-event` | Event type name |

## Locale Completeness

Not all locales have full coverage. The matrix below summarizes which domains are translated beyond the core set (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, general):

| Domain | `en` | `ja` | `de` | `es` | `fr` | `zh-CN` |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Goal | ✅ | ✅ | — | — | — | — |
| Memory | ✅ | ✅ | — | — | — | — |
| Network / A2A | ✅ | ✅ | — | — | — | — |
| Plugin | ✅ | ✅ | — | — | — | — |
| Channel | ✅ | ✅ | — | — | — | — |
| Provider | ✅ | ✅ | — | — | — | — |
| Skill | ✅ | ✅ | — | — | — | — |
| Hand | ✅ | ✅ | — | — | — | — |
| MCP | ✅ | ✅ | — | — | — | — |
| Integration / Extension | ✅ | ✅ | — | — | — | — |
| System | ✅ | ✅ | — | — | — | — |
| KV | ✅ | ✅ | — | — | — | — |
| Approval | ✅ | ✅ | — | — | — | — |
| Webhook | ✅ | ✅ | — | — | — | — |
| Backup | ✅ | ✅ | — | — | — | — |
| Schedule | ✅ | ✅ | — | — | — | — |
| Job / Task | ✅ | ✅ | — | — | — | — |
| Pairing | ✅ | ✅ | — | — | — | — |
| Binding | ✅ | ✅ | — | — | — | — |
| Command | ✅ | ✅ | — | — | — | — |
| File / Upload | ✅ | ✅ | — | — | — | — |
| Tool | ✅ | ✅ | — | — | — | ✅ |
| Validation | ✅ | ✅ | — | — | — | — |

## Adding a New Error Message

1. **Add the identifier to `en/errors.ftl` first.** This is the canonical locale. Follow the `api-error-{domain}-{description}` naming convention. Place it in the appropriate domain section.

2. **Add translations to all other locale files.** If a locale is not yet translated, the Fluent runtime will fall back to English, but adding a stub entry is preferred for discoverability.

3. **Use variables for dynamic content.** Never hardcode dynamic values into the message string:
   ```
   # Good
   api-error-template-not-found = Template '{ $name }' not found

   # Bad — do not do this
   api-error-template-not-found = Template not found
   ```

4. **Pass the variables at the call site.** The code that resolves this message must supply all referenced variables (`$name`, `$error`, etc.).

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g., `pt-BR/`).
2. Copy `en/errors.ftl` into the new directory as a starting point.
3. Translate all message values. Do **not** translate the identifiers (the left-hand side of `=`).
4. Register the locale in the application's Fluent bundle configuration so the runtime knows to load it.