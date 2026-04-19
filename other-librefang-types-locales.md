# Other — librefang-types-locales

# librefang-types-locales

Localization data for LibreFang API error messages, stored as [Project Fluent](https://projectfluent.org/) (`.ftl`) message catalogs.

## Purpose

This module holds the human-readable error strings returned by the LibreFang API. Each locale file maps a stable message identifier to a translated string, allowing the API surface to remain locale-aware without hardcoding user-facing text in business logic.

## Directory Layout

```
librefang-types/locales/
├── de/errors.ftl      # German
├── en/errors.ftl      # English (canonical / most complete)
├── es/errors.ftl      # Spanish
├── fr/errors.ftl      # French
├── ja/errors.ftl      # Japanese
└── zh-CN/errors.ftl   # Simplified Chinese
```

Every locale uses the same file name (`errors.ftl`) and the same message identifiers. The Fluent runtime resolves which file to load based on the requested locale at runtime.

## Message Identifier Convention

All identifiers follow the pattern:

```
api-error-<domain>-<detail>
```

| Segment | Meaning |
|---------|---------|
| `api-error-` | Constant prefix — scopes these keys to API-level errors |
| `<domain>` | The subsystem or resource type (e.g. `agent`, `session`, `webhook`) |
| `<detail>` | Specific error condition (e.g. `not-found`, `missing-url`, `too-large`) |

Examples:

```ftl
api-error-agent-not-found = Agent not found
api-error-webhook-url-unreachable = Webhook URL is unreachable: { $error }
api-error-goal-circular-parent = Circular parent reference detected
```

## Interpolated Variables

Some messages include Fluent variables enclosed in `{ $name }` syntax. The calling code must supply these values at lookup time.

Common variables:

| Variable | Used by | Example |
|----------|---------|---------|
| `$reason` | `bad-request`, `message-delivery-failed` | `"Bad request: { $reason }"` |
| `$error` | `*-failed`, `*-parse-failed` messages | `"Failed to parse configuration: { $error }"` |
| `$name` | `template-not-found`, `profile-not-found`, `provider-unknown` | `"Unknown provider '{ $name }'"` |
| `$id` | `agent-not-found-with-id`, `goal-not-found-with-id` | `"Agent not found: { $id }"` |
| `$step` | `workflow-step-needs-agent` | `"Step '{ $step }' needs 'agent_id' or 'agent_name'"` |
| `$alias` | `provider-alias-exists`, `provider-alias-not-found` | `"Alias '{ $alias }' already exists"` |
| `$url` | `network-a2a-not-found`, `network-missing-url` | `"A2A agent '{ $url }' not found"` |
| `$status` | `network-auth-failed` | `"Authentication failed (HTTP { $status })"` |
| `$event` | `webhook-unknown-event` | `"Unknown event type '{ $event }'..."` |
| `$valid` | `agent-invalid-sort`, `webhook-unknown-event` | `"Valid fields: { $valid }"` |
| `$field` | `agent-invalid-sort` | `"Invalid sort field '{ $field }'..."` |
| `$max` | `skill-description-too-long`, `file-too-large` | `"File too large (max { $max })"` |
| `$provider` | `provider-model-exists` | `"...in provider '{ $provider }'"` |

## Domain Coverage

Messages are organized into these domains. English (`en/errors.ftl`) is the authoritative source and defines the full set:

| Domain | Example messages | EN count |
|--------|-----------------|----------|
| Agent | `not-found`, `spawn-failed`, `invalid-id`, `execution-failed`, `clone-spawn-failed` | 17 |
| Message | `too-large`, `delivery-failed`, `required`, `streaming-failed` | 5 |
| Template | `invalid-name`, `not-found`, `parse-failed`, `required` | 6 |
| Manifest | `too-large`, `invalid-format`, `signature-mismatch`, `signature-failed` | 5 |
| Auth | `invalid-key`, `missing-header`, `missing` | 3 |
| Session | `load-failed`, `not-found`, `cleanup-*-failed` | 7 |
| Workflow | `missing-steps`, `step-needs-agent`, `execution-failed` | 5 |
| Trigger | `missing-agent-id`, `invalid-pattern`, `registration-failed` | 7 |
| Budget | `invalid-amount`, `update-failed`, `provide-at-least-one` | 3 |
| Config | `parse-failed`, `write-failed`, `save-failed`, `remove-failed` | 5 |
| Profile | `not-found` | 1 |
| Cron | `invalid-id`, `not-found`, `invalid-expression-detail` | 6 |
| Goal | `not-found`, `circular-parent`, `progress-range`, `save-failed` | 16 |
| Memory | `not-enabled`, `export-failed`, `missing-kv`, `serialization-error` | 9 |
| Network / A2A | `not-enabled`, `connection-failed`, `auth-failed` | 7 |
| Plugin | `missing-name`, `missing-path`, `invalid-source` | 5 |
| Channel | `unknown`, `invalid-from`, `invalid-to` | 4 |
| Provider | `missing-alias`, `alias-exists`, `secrets-write-failed` | 22 |
| Skill | `invalid-name`, `only-prompt`, `install-failed` | 10 |
| Hand | `not-found`, `definition-not-found`, `instance-not-found` | 3 |
| MCP | `missing-name`, `missing-transport`, `invalid-config` | 5 |
| Integration / Extension | `not-found`, `missing-id` | 3 |
| System | `cli-not-found` | 1 |
| KV | `missing-fields`, `missing-value`, `missing-path` | 4 |
| Approval | `invalid-id`, `not-found` | 2 |
| Webhook | `not-enabled`, `url-unreachable`, `unknown-event` | 14 |
| Backup | `not-found`, `missing-manifest`, `dir-create-failed` | 11 |
| Schedule | `not-found`, `missing-cron`, `invalid-cron-detail` | 10 |
| Job | `invalid-id`, `not-retryable`, `disappeared-cancel` | 5 |
| Task | `not-found`, `disappeared` | 2 |
| Pairing | `not-enabled`, `invalid-token` | 2 |
| Binding | `out-of-range` | 1 |
| Command | `not-found` | 1 |
| File / Upload | `not-found`, `too-large`, `path-traversal` | 14 |
| Tool | `provide-allowlist`, `not-found` | 2 |
| Validation | `content-empty`, `color-invalid`, `avatar-url-invalid` | 5 |
| General | `not-found`, `internal`, `bad-request`, `rate-limited` | 4 |

## Locale Completeness

The English catalog is the most comprehensive. Other locales fall into two tiers:

**Tier 1 — Near-complete** (follows `en` closely, includes newer domains like goals, memory, network, webhooks, backups, schedules, file uploads):
- `ja` (Japanese)

**Tier 2 — Core subset** (covers foundational domains: agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, plus general errors):
- `de` (German)
- `es` (Spanish)
- `fr` (French)
- `zh-CN` (Simplified Chinese)

When adding new error messages, update `en/errors.ftl` first, then propagate to other locales.

## Adding a New Error Message

1. **Define the key in English** (`locales/en/errors.ftl`):
   ```ftl
   api-error-<domain>-<detail> = Human-readable message with optional { $variable }
   ```

2. **Add translations** to each locale file under the same identifier. If a translation is not yet available, the Fluent runtime will fall back to the English string (depending on the configured fallback chain).

3. **Use the message in code** by referencing the identifier string exactly. For example, in Rust with the `fluent` crate:
   ```rust
   let msg = bundle.get_message("api-error-agent-not-found");
   ```

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g. `pt-BR/`).
2. Copy `en/errors.ftl` into the new directory as a starting template.
3. Translate all message values, preserving:
   - The message identifier (left-hand side) exactly as-is.
   - Variable syntax (`{ $name }`) exactly as-is.
   - Any literal formatting such as `Bearer <api_key>` or size limits like `(max 64KB)` — verify whether these should be localized or kept as-is for your locale.

## Integration with the Codebase

This module is a **pure data module** — it contains no executable code, no function calls, and no type definitions. Other crates in the workspace consume these `.ftl` files at build time or runtime via a Fluent localization bundle. The `librefang-types` crate likely includes these files as `include_str!` resources or bundles them into the binary for the API server to look up error messages by identifier when constructing error responses.