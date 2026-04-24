# Other — librefang-types-locales

# librefang-types/locales

Localization files for LibreFang API error messages using the [Fluent](https://projectfluent.org/) (`.ftl`) translation format.

## Purpose

This directory contains all user-facing API error message translations, organized by locale. Every error returned by the LibreFang API has a stable message identifier (e.g. `api-error-agent-not-found`) with localized string values in each supported language. The application resolves the correct message at runtime based on the client's language preference.

## Supported Locales

| Directory | Language | Coverage |
|-----------|----------|----------|
| `en/` | English | **Complete** — canonical source of truth |
| `de/` | German | Partial (core domains) |
| `es/` | Spanish | Partial (core domains) |
| `fr/` | French | Partial (core domains) |
| `ja/` | Japanese | Full (mirrors English) |
| `zh-CN/` | Simplified Chinese | Partial (core + select extended domains) |

## File Structure

```
locales/
├── de/errors.ftl
├── en/errors.ftl
├── es/errors.ftl
├── fr/errors.ftl
├── ja/errors.ftl
└── zh-CN/errors.ftl
```

Each locale contains a single `errors.ftl` file with all API error messages for that language.

## Fluent Message Format

Messages use [Fluent syntax](https://projectfluent.org/fluent/guide/):

```fluent
# Simple message (no variables)
api-error-agent-not-found = Agent not found

# Message with interpolation variables
api-error-message-delivery-failed = Message delivery failed: { $reason }

# Multiple variables
api-error-cron-invalid-expression-detail = Invalid cron expression: requires 5 fields (minute hour day month weekday)
```

### Variable Patterns

The following interpolation variables appear across the error set:

| Variable | Example Usage |
|----------|--------------|
| `$reason` | Bad request reasons, delivery failure reasons |
| `$error` | Parse failures, I/O errors, operation failures |
| `$name` | Template names, profile names, provider names, command names |
| `$id` | Agent IDs, goal IDs, integration IDs |
| `$step` | Workflow step identifiers |
| `$alias` | Provider alias names |
| `$url` | A2A agent URLs, webhook URLs |
| `$status` | HTTP status codes |
| `$field` | Sort field names |
| `$valid` | Lists of valid values |
| `$max` | Maximum length limits |
| `$provider` | Provider identifiers |
| `$event` | Event type names |

## Error Domain Organization

Messages are grouped by domain within each `.ftl` file. The canonical English locale defines the complete set:

| Domain | Prefix | Example |
|--------|--------|---------|
| Agent | `api-error-agent-*` | `api-error-agent-not-found` |
| Message | `api-error-message-*` | `api-error-message-too-large` |
| Template | `api-error-template-*` | `api-error-template-not-found` |
| Manifest | `api-error-manifest-*` | `api-error-manifest-signature-failed` |
| Auth | `api-error-auth-*` | `api-error-auth-invalid-key` |
| Session | `api-error-session-*` | `api-error-session-not-found` |
| Workflow | `api-error-workflow-*` | `api-error-workflow-missing-steps` |
| Trigger | `api-error-trigger-*` | `api-error-trigger-missing-pattern` |
| Budget | `api-error-budget-*` | `api-error-budget-update-failed` |
| Config | `api-error-config-*` | `api-error-config-parse-failed` |
| Profile | `api-error-profile-*` | `api-error-profile-not-found` |
| Cron | `api-error-cron-*` | `api-error-cron-create-failed` |
| Goal | `api-error-goal-*` | `api-error-goal-circular-parent` |
| Memory | `api-error-memory-*` | `api-error-memory-not-enabled` |
| Network/A2A | `api-error-network-*` | `api-error-network-a2a-not-found` |
| Plugin | `api-error-plugin-*` | `api-error-plugin-invalid-source` |
| Channel | `api-error-channel-*` | `api-error-channel-unknown` |
| Provider | `api-error-provider-*` | `api-error-provider-model-not-found` |
| Skill | `api-error-skill-*` | `api-error-skill-install-failed` |
| Hand | `api-error-hand-*` | `api-error-hand-not-found` |
| MCP | `api-error-mcp-*` | `api-error-mcp-not-found` |
| Integration | `api-error-integration-*` | `api-error-integration-not-found` |
| Extension | `api-error-extension-*` | `api-error-extension-not-found` |
| System | `api-error-system-*` | `api-error-system-cli-not-found` |
| KV/Memory | `api-error-kv-*` | `api-error-kv-missing-fields` |
| Approval | `api-error-approval-*` | `api-error-approval-not-found` |
| Webhook | `api-error-webhook-*` | `api-error-webhook-url-unreachable` |
| Backup | `api-error-backup-*` | `api-error-backup-missing-manifest` |
| Schedule | `api-error-schedule-*` | `api-error-schedule-save-failed` |
| Job | `api-error-job-*` | `api-error-job-not-retryable` |
| Task | `api-error-task-*` | `api-error-task-disappeared` |
| Pairing | `api-error-pairing-*` | `api-error-pairing-invalid-token` |
| Binding | `api-error-binding-*` | `api-error-binding-out-of-range` |
| Command | `api-error-command-*` | `api-error-command-not-found` |
| File/Upload | `api-error-file-*` | `api-error-file-path-traversal` |
| Tool | `api-error-tool-*` | `api-error-tool-invoke-denied` |
| Validation | `api-error-validation-*` | `api-error-validation-color-invalid` |
| General | `api-error-*` (unprefixed) | `api-error-rate-limited` |

## Adding a New Error Message

1. **Add the message to `en/errors.ftl`** first — this is the canonical locale. Choose the correct domain section and follow the naming convention `api-error-{domain}-{description}`.

2. **Add translations to all other locales** in their respective `errors.ftl` files. For locales that don't yet have full coverage, add the translation anyway to prevent fallback gaps.

3. **Preserve variable names exactly** — `$error`, `$name`, `$id`, etc. must match across all locales since they are resolved by the calling code.

Example addition:

```fluent
# In en/errors.ftl
api-error-agent-quota-exceeded = Agent quota exceeded (max { $limit })

# In ja/errors.ftl
api-error-agent-quota-exceeded = エージェントのクォータを超えています（最大 { $limit }）
```

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g. `pt-BR/`, `ko/`).

2. Copy `en/errors.ftl` into the new directory as a starting template.

3. Translate all message values. **Do not** translate the message identifiers (left side of `=`). **Do not** alter variable names inside `{ }` braces.

4. Register the locale in the application's Fluent bundle initialization code.

## Coverage Expectations

The English (`en`) and Japanese (`ja`) locales maintain **full coverage** of all defined error keys. The German (`de`), Spanish (`es`), French (`fr`), and Simplified Chinese (`zh-CN`) locales currently cover the core API domains (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, and general errors). 

When the application encounters a missing key in the requested locale, Fluent falls back through the locale chain and ultimately to the `en` messages. To check for missing keys, compare the identifier set of a partial locale against `en/errors.ftl`.

## Relationship to the Codebase

These `.ftl` files are consumed by the Fluent localization system at runtime. API handler code references error identifiers (as string constants or enum variants) without embedding human-readable text directly. The translation layer resolves the identifier to the localized string based on the active locale context.

```
API Handler → error identifier (e.g. "api-error-agent-not-found")
                    ↓
         Fluent Bundle (loaded from locales/)
                    ↓
         Resolved string → HTTP response body
```

This module has no Rust code, build logic, or dependencies on other modules — it is a pure data/resource module consumed through the Fluent localization framework.