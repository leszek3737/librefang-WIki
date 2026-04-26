# Other — librefang-types-locales

# librefang-types-locales

Fluent (FTL) localization resources for LibreFang API error messages. This module provides translated user-facing error strings consumed by the `fluent` / `fluent-bundle` localization runtime at the API layer.

## Purpose

When the API returns an error, the message identifier (e.g. `api-error-agent-not-found`) is resolved against the active locale to produce a human-readable string in the caller's preferred language. This module is the **single source of truth** for those translations — no error text is hardcoded elsewhere.

## Directory Layout

```
librefang-types/locales/
├── de/errors.ftl       # German
├── en/errors.ftl       # English  (canonical / most complete)
├── es/errors.ftl       # Spanish
├── fr/errors.ftl       # French
├── ja/errors.ftl       # Japanese
└── zh-CN/errors.ftl    # Simplified Chinese
```

Each file follows the same naming convention: `<language-tag>/errors.ftl`.

## FTL Format Quick Reference

Every entry is a key-value pair in [Fluent syntax](https://projectfluent.org/):

```ftl
# Simple message
api-error-agent-not-found = Agent not found

# Message with interpolation variables
api-error-message-delivery-failed = Message delivery failed: { $reason }

# Variables are referenced as { $name }
api-error-template-not-found = Template '{ $name }' not found
```

### Supported Variables

| Variable | Typical Source | Example Key |
|---|---|---|
| `$reason` | API request validation | `api-error-bad-request` |
| `$error` | Internal error detail | `api-error-template-parse-failed` |
| `$name` | Resource identifier | `api-error-template-not-found` |
| `$id` | Entity ID | `api-error-agent-not-found-with-id` |
| `$step` | Workflow step label | `api-error-workflow-step-needs-agent` |
| `$alias` | Provider alias | `api-error-provider-alias-exists` |
| `$provider` | Provider name | `api-error-provider-model-exists` |
| `$url` | Endpoint URL | `api-error-network-a2a-not-found` |
| `$status` | HTTP status code | `api-error-network-auth-failed` |
| `$max` | Size limit value | `api-error-skill-description-too-long` |
| `$field` | Field name | `api-error-agent-invalid-sort` |
| `$valid` | List of valid values | `api-error-agent-invalid-sort` |
| `$event` | Event type string | `api-error-webhook-unknown-event` |

## Error Categories

Messages are grouped by domain within each FTL file. The English (`en`) locale defines the full canonical set; other locales may contain a subset. The categories and their key prefixes are:

| Prefix | Domain | Example |
|---|---|---|
| `api-error-agent-*` | Agent lifecycle | `api-error-agent-spawn-failed` |
| `api-error-message-*` | Messaging | `api-error-message-too-large` |
| `api-error-template-*` | Template management | `api-error-template-parse-failed` |
| `api-error-manifest-*` | Manifest parsing / signing | `api-error-manifest-signature-mismatch` |
| `api-error-auth-*` | Authentication / API keys | `api-error-auth-invalid-key` |
| `api-error-session-*` | Session management | `api-error-session-load-failed` |
| `api-error-workflow-*` | Workflow orchestration | `api-error-workflow-missing-steps` |
| `api-error-trigger-*` | Trigger registration | `api-error-trigger-missing-pattern` |
| `api-error-budget-*` | Budget enforcement | `api-error-budget-update-failed` |
| `api-error-config-*` | Configuration I/O | `api-error-config-write-failed` |
| `api-error-profile-*` | Profiles | `api-error-profile-not-found` |
| `api-error-cron-*` | Cron / scheduled tasks | `api-error-cron-invalid-expression` |
| `api-error-goal-*` | Goal tracking | `api-error-goal-circular-parent` |
| `api-error-memory-*` | Proactive memory / KV store | `api-error-memory-not-enabled` |
| `api-error-network-*` | Peer network / A2A | `api-error-network-connection-failed` |
| `api-error-plugin-*` | Plugin installation | `api-error-plugin-invalid-source` |
| `api-error-channel-*` | Inter-agent channels | `api-error-channel-invalid-from` |
| `api-error-provider-*` | LLM provider config | `api-error-provider-alias-exists` |
| `api-error-skill-*` | Skill management | `api-error-skill-install-failed` |
| `api-error-hand-*` | Hand instances | `api-error-hand-not-found` |
| `api-error-mcp-*` | MCP server config | `api-error-mcp-missing-transport` |
| `api-error-integration-*` / `api-error-extension-*` | Integrations & extensions | `api-error-integration-not-found` |
| `api-error-system-*` | System-level checks | `api-error-system-cli-not-found` |
| `api-error-kv-*` | Structured KV memory | `api-error-kv-missing-fields` |
| `api-error-approval-*` | Approval workflow | `api-error-approval-not-found` |
| `api-error-webhook-*` | Webhook triggers | `api-error-webhook-url-unreachable` |
| `api-error-backup-*` | Backup create / restore | `api-error-backup-missing-manifest` |
| `api-error-schedule-*` | Scheduled jobs | `api-error-schedule-invalid-cron` |
| `api-error-job-*` | Background jobs | `api-error-job-not-retryable` |
| `api-error-task-*` | Task tracking | `api-error-task-disappeared` |
| `api-error-pairing-*` | Device pairing | `api-error-pairing-not-enabled` |
| `api-error-binding-*` | Binding indices | `api-error-binding-out-of-range` |
| `api-error-command-*` | CLI commands | `api-error-command-not-found` |
| `api-error-file-*` | File upload / workspace | `api-error-file-path-traversal` |
| `api-error-tool-*` | Tool invocation | `api-error-tool-invoke-disabled` |
| `api-error-validation-*` | Input validation | `api-error-validation-color-invalid` |
| `api-error-*` (unprefixed) | General fallbacks | `api-error-not-found`, `api-error-internal` |

## Locale Completeness

The English locale is the canonical reference. Other locales vary in coverage:

| Locale | Approximate Coverage | Notes |
|---|---|---|
| `en` | Full (200+ keys) | Canonical; all domains covered |
| `ja` | Near-full | Missing only a few keys like `api-error-webhook-invalid-url`, `api-error-agent-invalid-sort` |
| `de` / `es` / `fr` / `zh-CN` | Core subset (~40–50 keys) | Covers agents, auth, sessions, triggers, workflows, cron, config; newer domains (goals, memory, webhooks, backup, schedule, etc.) are untranslated |

When a key is missing from a non-English locale, the Fluent runtime falls back according to its configured fallback chain (typically to `en`).

## Adding a New Error Message

1. **Add the key to `en/errors.ftl` first** — this is the source of truth.
2. Provide translations in all other locale files that cover the relevant domain. If you don't have a translation, omit the key; the English fallback will be used.
3. Follow the naming convention: `api-error-<domain>-<short-description>`.
4. Use Fluent variables (`{ $variable }`) for any dynamic content — never concatenate strings in application code.

Example — adding a new agent error:

```ftl
# en/errors.ftl
api-error-agent-paused = Agent is paused and cannot accept messages

# ja/errors.ftl
api-error-agent-paused = エージェントは一時停止中でメッセージを受信できません
```

## Adding a New Locale

1. Create the directory `librefang-types/locales/<language-tag>/`.
2. Add an `errors.ftl` file with translations for all keys in `en/errors.ftl`.
3. Register the new locale in the Fluent bundle initialization code so the runtime knows to load it.

## Integration Points

This module is purely a static resource. At runtime:

- The API server loads the FTL files into a `fluent-bundle` per locale.
- Error handling code references messages by identifier (e.g. `api-error-agent-not-found`), passing any required variables as a hash/map.
- The Fluent runtime interpolates variables and returns the localized string.
- Locale selection is typically based on the `Accept-Language` header or an explicit `lang` query parameter.

Because there are no executable calls in or out of this module, it introduces no runtime dependencies beyond the Fluent library itself.