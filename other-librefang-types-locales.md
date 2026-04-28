# Other — librefang-types-locales

# librefang-types-locales

Fluent (`.ftl`) message catalogs providing localized API error strings for the LibreFang platform. This is a pure data module — no executable code, no call graph. Other crates resolve these message IDs at runtime via a Fluent localization library.

## Supported Locales

| Code | Language | Completeness |
|------|----------|-------------|
| `en` | English | **Full** — canonical source of truth |
| `ja` | Japanese | Near-full |
| `de` | German | Partial (core subset) |
| `es` | Spanish | Partial (core subset) |
| `fr` | French | Partial (core subset) |
| `zh-CN` | Simplified Chinese | Partial (core subset + tool errors) |

## File Layout

```
librefang-types/locales/
├── en/errors.ftl      # canonical — all message IDs defined here
├── ja/errors.ftl
├── de/errors.ftl
├── es/errors.ftl
├── fr/errors.ftl
└── zh-CN/errors.ftl
```

Each file contains a single Fluent resource: `errors.ftl`. The locale directory name doubles as the Fluent locale identifier.

## Message ID Convention

Every identifier follows the pattern:

```
api-error-<category>-<detail>
```

Lowercase, hyphen-separated. No dots, no uppercase. This namespace is flat — there is no hierarchical grouping in the identifier itself, only in the source-level comments.

## Error Categories

The English catalog defines messages across these functional areas:

| Category | Prefix | Typical concerns |
|----------|--------|-----------------|
| Agent | `api-error-agent-*` | Spawn, lookup, workspace, execution, cloning, sort validation |
| Message | `api-error-message-*` | Size limits (64 KB), delivery, streaming |
| Template | `api-error-template-*` | Name validation, parsing, manifest requirements |
| Manifest | `api-error-manifest-*` | Size limits (1 MB), format, signature verification |
| Auth | `api-error-auth-*` | API key validation, missing headers, provider config |
| Session | `api-error-session-*` | Load, lookup, cleanup, label resolution |
| Workflow | `api-error-workflow-*` | Step validation, execution, ID format |
| Trigger | `api-error-trigger-*` | Pattern validation, agent binding, registration |
| Budget | `api-error-budget-*` | Amount validation, update failures |
| Config | `api-error-config-*` | Parse/write/save/remove operations |
| Profile | `api-error-profile-*` | Lookup by name |
| Cron | `api-error-cron-*` | Expression validation (5-field), creation |
| Goal | `api-error-goal-*` | CRUD, hierarchy (parent/circular), status enum, field length |
| Memory | `api-error-memory-*` | Enablement check, KV operations, serialization, export/import |
| Network / A2A | `api-error-network-*` | Peer-to-peer, A2A agent discovery, connection/auth failures |
| Plugin | `api-error-plugin-*` | Source validation (registry/local/git), missing fields |
| Channel | `api-error-channel-*` | Agent ID validation, from/to checks |
| Provider | `api-error-provider-*` | Alias management, model registration, secrets, URL validation |
| Skill | `api-error-skill-*` | Name constraints, directory creation, TOML writes, install |
| Hand | `api-error-hand-*` | Definition/instance lookup |
| MCP | `api-error-mcp-*` | Server configuration, transport validation |
| Integration / Extension | `api-error-integration-*`, `api-error-extension-*` | ID-based lookup |
| System | `api-error-system-*` | CLI availability |
| KV | `api-error-kv-*` | Structured memory field/value/path requirements |
| Approval | `api-error-approval-*` | ID format, lookup |
| Webhook | `api-error-webhook-*` | URL validation, event types, reachability, publishing |
| Backup | `api-error-backup-*` | Archive validation, manifest presence, file operations |
| Schedule | `api-error-schedule-*` | CRUD, cron expression validation |
| Job | `api-error-job-*` | Retryable state checks, disappearance |
| Task | `api-error-task-*` | Lookup, disappearance |
| Pairing | `api-error-pairing-*` | Enablement, token validation |
| Binding | `api-error-binding-*` | Index range |
| Command | `api-error-command-*` | Command lookup |
| File / Upload | `api-error-file-*` | Whitelist, size limits, path traversal, content type, depth |
| Tool | `api-error-tool-*` | Allowlist/blocklist, invoke permissions, agent context |
| Validation | `api-error-validation-*` | General field constraints (content, name, title, avatar, color) |
| General | `api-error-not-found`, `api-error-internal`, `api-error-bad-request`, `api-error-rate-limited` | Catch-all HTTP-level errors |

## Interpolation Variables

Messages use Fluent placeable syntax for dynamic values. The variables in use across the catalogs are:

| Variable | Used in | Example message |
|----------|---------|----------------|
| `{ $reason }` | `message-delivery-failed`, `bad-request` | Reason for failure or rejection |
| `{ $error }` | `template-parse-failed`, `config-parse-failed`, `agent-execution-failed`, `cron-create-failed`, `goal-save-failed`, `network-connection-failed`, `skill-dir-create-failed`, `mcp-invalid-config`, `webhook-url-unreachable`, `backup-*-failed`, `schedule-*-failed`, `provider-secrets-*-failed`, `provider-token-save-failed` | Underlying error detail |
| `{ $name }` | `template-not-found`, `profile-not-found`, `provider-alias-not-found`, `provider-not-found`, `provider-unknown`, `mcp-not-found`, `command-not-found`, `tool-not-found`, `tool-invoke-denied` | Entity name |
| `{ $id }` | `agent-not-found-with-id`, `goal-not-found-with-id`, `goal-parent-not-found`, `hand-not-found`, `integration-not-found`, `extension-not-found` | Entity identifier |
| `{ $step }` | `workflow-step-needs-agent` | Step name in a workflow |
| `{ $url }` | `network-a2a-not-found`, `network-missing-url` | URL parameter |
| `{ $status }` | `network-auth-failed` | HTTP status code |
| `{ $alias }` | `provider-alias-exists`, `provider-alias-not-found` | Provider alias |
| `{ $provider }` | `provider-model-exists` | Provider name |
| `{ $max }` | `skill-description-too-long`, `file-too-large` | Maximum allowed value |
| `{ $event }` | `webhook-unknown-event` | Event type string |
| `{ $valid }` | `agent-invalid-sort`, `webhook-unknown-event` | List of valid options |
| `{ $field }` | `agent-invalid-sort` | Field name |

## Adding a New Error Message

1. **Add the identifier to `en/errors.ftl` first.** The English catalog is the canonical source. Choose the correct category prefix and a descriptive suffix.

2. **Add translations to other locale files.** At minimum, add the same identifier to all locale files. If a translation is not yet ready, you may omit it — Fluent will fall back to English for missing keys — but this should be tracked as a translation gap.

3. **Use variables, not string concatenation.** Reference dynamic values as `{ $variableName }` in the message body. Pass them as a Fluent args map at the call site.

Example addition to `en/errors.ftl`:

```fluent
# Widget errors
api-error-widget-not-found = Widget '{ $id }' not found
api-error-widget-creation-failed = Failed to create widget: { $error }
```

Corresponding entry in `ja/errors.ftl`:

```fluent
# ウィジェットエラー
api-error-widget-not-found = ウィジェット '{ $id }' が見つかりません
api-error-widget-creation-failed = ウィジェットの作成に失敗しました: { $error }
```

## Adding a New Locale

1. Create a new directory under `locales/` with the appropriate BCP 47 tag (e.g., `pt-BR/`).
2. Create `errors.ftl` inside it.
3. Translate entries from the English catalog. Keep the identifiers **identical** — only the right-hand-side message text changes.
4. Register the new locale in the application's localization bundle initialization.

## Locale Completeness

The non-English locales fall into two tiers based on coverage:

- **Near-complete** (`ja`): Covers all categories including goal, memory, network, plugin, channel, provider, skill, hand, MCP, integration/extension, system, KV, approval, webhook, backup, schedule, job, task, pairing, binding, command, file, tool, and validation errors.
- **Core subset** (`de`, `es`, `fr`, `zh-CN`): Covers agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, and general errors. Notably missing are goal, memory, network, plugin, channel, provider, skill, hand, MCP, webhook, backup, schedule, job, file, tool, and validation categories. `zh-CN` additionally includes tool errors.

When consuming messages at runtime, ensure the fallback chain terminates at `en` so that missing translations in partial locales degrade gracefully.

## Integration with the Codebase

This module is consumed by the `librefang-types` crate, which exposes typed error enums or response structures that reference these Fluent message IDs. The typical usage pattern:

1. Application code raises a typed error (e.g., `AgentNotFound`).
2. The error handler maps it to the Fluent message ID `api-error-agent-not-found`.
3. The localization layer looks up the message in the current locale's `errors.ftl`, interpolating any variables.
4. The rendered string is returned to the client in the API response.

Because this module is pure static data, it has no dependencies and no runtime behavior. It is included in the build as a resource file (via `include_str!` or a build-time copy step, depending on the consuming crate's approach).