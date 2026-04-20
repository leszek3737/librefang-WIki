# Other — librefang-types-locales

# librefang-types-locales

Localized API error message strings for the LibreFang platform, using the [Fluent](https://projectfluent.org/) (`.ftl`) translation format.

## Purpose

This module holds all user-facing error messages returned by the LibreFang API. Each language gets its own directory under `locales/`, containing an `errors.ftl` file with translated strings. The Fluent format supports positional variables (`{ $reason }`, `{ $error }`, etc.) so that dynamic context is preserved across translations.

The rest of the codebase references these message keys (e.g. `api-error-agent-not-found`) via a Fluent-compatible localization library. At runtime, the caller selects the active locale and resolves the key to produce a localized error string.

## Directory Layout

```
locales/
├── de/errors.ftl       # German
├── en/errors.ftl       # English  (canonical / most complete)
├── es/errors.ftl       # Spanish
├── fr/errors.ftl       # French
├── ja/errors.ftl       # Japanese
└── zh-CN/errors.ftl    # Simplified Chinese
```

## Locale Coverage

| Locale  | Language         | Completeness |
|---------|-----------------|--------------|
| `en`    | English          | Full (all keys) |
| `ja`    | Japanese         | Full (all keys) |
| `de`    | German           | Partial (core keys only) |
| `es`    | Spanish          | Partial (core keys only) |
| `fr`    | French           | Partial (core keys only) |
| `zh-CN` | Simplified Chinese | Partial (core keys only) |

The English file is the authoritative source of truth. All other locales should eventually mirror its full key set.

## Key Naming Convention

Every key follows the pattern:

```
api-error-<domain>-<specific>
```

- **`api-error-`** — fixed prefix identifying these as API error messages.
- **`<domain>`** — the subsystem or resource type (e.g. `agent`, `session`, `webhook`).
- **`<specific>`** — a short description of the error condition.

### Domains Covered

| Domain | Examples |
|--------|---------|
| `agent` | `not-found`, `spawn-failed`, `already-exists`, `execution-failed` |
| `message` | `too-large`, `delivery-failed`, `required` |
| `template` | `invalid-name`, `not-found`, `parse-failed` |
| `manifest` | `too-large`, `invalid-format`, `signature-mismatch` |
| `auth` | `invalid-key`, `missing-header`, `missing` |
| `session` | `load-failed`, `not-found`, `invalid-id` |
| `workflow` | `missing-steps`, `execution-failed`, `not-found` |
| `trigger` | `missing-agent-id`, `invalid-pattern`, `registration-failed` |
| `budget` | `invalid-amount`, `update-failed` |
| `config` | `parse-failed`, `write-failed`, `save-failed` |
| `profile` | `not-found` |
| `cron` | `invalid-id`, `not-found`, `create-failed`, `invalid-expression` |
| `goal` | `not-found`, `missing-title`, `circular-parent`, `save-failed` |
| `memory` | `not-enabled`, `not-found`, `operation-failed`, `export-failed` |
| `network` | `not-enabled`, `peer-not-found`, `connection-failed`, `auth-failed` |
| `plugin` | `missing-name`, `missing-path`, `invalid-source` |
| `channel` | `unknown`, `missing-agent-id`, `invalid-from` |
| `provider` | `missing-alias`, `alias-exists`, `model-not-found`, `key-not-configured` |
| `skill` | `missing-name`, `invalid-name`, `install-failed` |
| `hand` | `not-found`, `definition-not-found` |
| `mcp` | `missing-name`, `missing-transport`, `invalid-config` |
| `integration` / `extension` | `not-found`, `missing-id` |
| `system` | `cli-not-found` |
| `kv` | `missing-fields`, `missing-value`, `array-empty` |
| `approval` | `invalid-id`, `not-found` |
| `webhook` | `not-enabled`, `invalid-id`, `missing-url`, `url-unreachable` |
| `backup` | `not-found`, `invalid-filename`, `missing-manifest` |
| `schedule` | `not-found`, `invalid-cron`, `save-failed` |
| `job` | `invalid-id`, `not-found`, `not-retryable` |
| `task` | `not-found`, `disappeared` |
| `pairing` | `not-enabled`, `invalid-token` |
| `binding` | `out-of-range` |
| `command` | `not-found` |
| `file` | `not-found`, `too-large`, `path-traversal`, `unsupported-type` |
| `tool` | `provide-allowlist` |
| `validation` | `content-empty`, `name-empty`, `title-required` |
| *(general)* | `not-found`, `internal`, `bad-request`, `rate-limited` |

## Fluent Variable Interpolation

Many messages include runtime parameters using Fluent's `{ $variable }` syntax:

```ftl
api-error-message-delivery-failed = Message delivery failed: { $reason }
api-error-template-not-found = Template '{ $name }' not found
api-error-agent-not-found-with-id = Agent not found: { $id }
api-error-network-auth-failed = Authentication failed (HTTP { $status })
api-error-cron-create-failed = Failed to create cron job: { $error }
```

When adding a new key that accepts parameters, ensure the same parameter names are used across all locale files for that key.

## Adding a New Error Key

1. **Add the key to `en/errors.ftl`** under the appropriate domain section. If no section exists, create one with a `# <Domain> errors` comment header.

2. **Copy the key to every other locale's `errors.ftl`**, translating the message text. Keep the same variable names.

3. **For incomplete locales** (de, es, fr, zh-CN), you may add only the English message as a placeholder if no translator is available — the Fluent runtime will fall back gracefully depending on configuration, but ideally every locale should have a translation.

Example — adding a new `snapshot` domain:

```ftl
# --- In en/errors.ftl ---

# Snapshot errors
api-error-snapshot-not-found = Snapshot not found
api-error-snapshot-create-failed = Failed to create snapshot: { $error }
```

```ftl
# --- In ja/errors.ftl ---

# スナップショットエラー
api-error-snapshot-not-found = スナップショットが見つかりません
api-error-snapshot-create-failed = スナップショットの作成に失敗しました: { $error }
```

## Adding a New Locale

1. Create a new directory under `locales/` using the appropriate [BCP 47](https://tools.ietf.org/html/bcp47) tag (e.g. `pt-BR`, `ko`).

2. Copy `en/errors.ftl` into the new directory as a starting point.

3. Translate every message value, preserving keys and variable names exactly.

4. Register the new locale in whatever Fluent runtime configuration the application uses (outside this module).

## Integration Notes

This module is **data-only** — it contains no executable code. Other crates in the workspace depend on it by:

- Bundling the `locales/` directory at compile time (e.g. via `include_str!` or a build script), or
- Loading `.ftl` files at runtime from a known path relative to the binary.

The consuming code is responsible for locale selection, fallback chains, and passing variables when resolving messages.