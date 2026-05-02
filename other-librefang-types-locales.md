# Other — librefang-types-locales

# librefang-types-locales

Localized API error messages for the LibreFang platform, stored in [Project Fluent](https://projectfluent.org/) (`.ftl`) format. This module is a **static resource** — it contains no executable code. Other crates load these files at runtime through a Fluent-compatible localization system to produce human-readable error strings in the user's preferred language.

## Supported Locales

| Directory | Language | Completeness |
|-----------|----------|-------------|
| `en/` | English (reference) | Full — all error keys |
| `ja/` | Japanese | Full — matches English |
| `de/` | German | Partial — core errors only |
| `es/` | Spanish | Partial — core errors only |
| `fr/` | French | Partial — core errors only |
| `zh-CN/` | Simplified Chinese | Partial — core + tool errors |

English (`en`) is the canonical source of truth. When other locales are missing a key, the Fluent runtime falls back to English automatically.

## File Structure

Each locale contains a single `errors.ftl` file:

```
locales/
├── de/errors.ftl
├── en/errors.ftl
├── es/errors.ftl
├── fr/errors.ftl
├── ja/errors.ftl
└── zh-CN/errors.ftl
```

## Message Format

Messages follow Fluent syntax. Each line defines a message identifier and its localized text:

```fluent
api-error-agent-not-found = Agent not found
```

### Variables

Many messages accept interpolated variables using `{$name}` syntax:

```fluent
api-error-message-delivery-failed = Message delivery failed: { $reason }
api-error-template-not-found = Template '{ $name }' not found
api-error-agent-execution-failed = Agent execution failed: { $error }
api-error-agent-not-found-with-id = Agent not found: { $id }
```

When calling code resolves a message, it passes a map of variable values. The Fluent runtime substitutes them into the output string.

## Error Key Naming Convention

All keys follow the pattern:

```
api-error-<domain>-<specific-error>
```

Domains in the current catalog:

| Domain | Example Key | Typical Variables |
|--------|------------|-------------------|
| `agent` | `api-error-agent-not-found` | `$id`, `$error`, `$field`, `$valid` |
| `message` | `api-error-message-too-large` | `$reason` |
| `template` | `api-error-template-not-found` | `$name`, `$error` |
| `manifest` | `api-error-manifest-signature-mismatch` | — |
| `auth` | `api-error-auth-invalid-key` | — |
| `session` | `api-error-session-not-found` | `$error` |
| `workflow` | `api-error-workflow-missing-steps` | `$step` |
| `trigger` | `api-error-trigger-not-found` | — |
| `budget` | `api-error-budget-update-failed` | — |
| `config` | `api-error-config-parse-failed` | `$error` |
| `profile` | `api-error-profile-not-found` | `$name` |
| `cron` | `api-error-cron-create-failed` | `$error` |
| `goal` | `api-error-goal-circular-parent` | `$id`, `$error` |
| `memory` | `api-error-memory-not-found` | — |
| `network` | `api-error-network-connection-failed` | `$url`, `$error`, `$status` |
| `plugin` | `api-error-plugin-missing-name` | — |
| `channel` | `api-error-channel-unknown` | — |
| `provider` | `api-error-provider-not-found` | `$name`, `$alias`, `$id`, `$provider`, `$error` |
| `skill` | `api-error-skill-missing-name` | `$error`, `$max` |
| `hand` | `api-error-hand-not-found` | `$id` |
| `mcp` | `api-error-mcp-not-found` | `$name`, `$error` |
| `integration` | `api-error-integration-not-found` | `$id` |
| `extension` | `api-error-extension-not-found` | `$id` |
| `system` | `api-error-system-cli-not-found` | — |
| `kv` | `api-error-kv-missing-fields` | — |
| `approval` | `api-error-approval-not-found` | — |
| `webhook` | `api-error-webhook-not-found` | `$error`, `$event`, `$valid` |
| `backup` | `api-error-backup-not-found` | `$error` |
| `schedule` | `api-error-schedule-not-found` | `$error` |
| `job` | `api-error-job-not-found` | — |
| `task` | `api-error-task-not-found` | — |
| `pairing` | `api-error-pairing-not-enabled` | — |
| `binding` | `api-error-binding-out-of-range` | — |
| `command` | `api-error-command-not-found` | `$name` |
| `file` | `api-error-file-not-found` | `$max` |
| `tool` | `api-error-tool-not-found` | `$name` |
| `validation` | `api-error-validation-title-required` | — |
| *(general)* | `api-error-not-found` | `$reason` |

General errors (`api-error-not-found`, `api-error-internal`, `api-error-bad-request`, `api-error-rate-limited`) have no domain prefix segment beyond `api-error-`.

## How the Codebase Consumes These Files

This module produces no function calls and receives none. It is a **pure data** dependency. The typical integration path is:

1. At build time or startup, the consuming crate locates the `locales/` directory (via `CARGO_MANIFEST_DIR`, a configured path, or `include_str!`).
2. A Fluent `Bundle` is constructed per locale, loading `errors.ftl` from each directory.
3. When an API handler needs to return an error, it calls into a localization function that:
   - Selects the user's preferred locale (from `Accept-Language`, configuration, or session data).
   - Looks up the message by its Fluent identifier (e.g., `api-error-agent-not-found`).
   - Passes any required variables (e.g., `{ "id": "abc-123" }`).
   - Returns the resolved string for use in the HTTP response body.

## Adding a New Error Message

1. **Add the key to `en/errors.ftl`** — this is the reference locale. Place it under the correct domain comment heading, or add a new heading if it's a new domain.

   ```fluent
   api-error-agent-new-example = New example error: { $detail }
   ```

2. **Add translations to all other locales** that should cover this error. At minimum, add the key to every locale directory that exists. If a locale is still a partial translation, adding the key is still recommended to make future completion easier.

3. **Use the key in application code** — reference the identifier when constructing API error responses so the Fluent resolver can look it up at runtime.

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g., `pt-BR/`).
2. Copy `en/errors.ftl` into the new directory as a starting template.
3. Translate every message value (the text after `=`) while keeping:
   - All message identifiers unchanged.
   - All variable references (`{ $name }`) unchanged.
   - All comments (optional — translate or remove as needed).
4. Register the new locale in whatever locale negotiation or bundle construction logic the consuming crate uses.