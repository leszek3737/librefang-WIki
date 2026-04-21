# Other — librefang-types-locales

# librefang-types-locales

Fluent (FTL) locale files providing localized API error messages for the LibreFang platform. This is a pure data module — no executable code — consumed at runtime by the Fluent localization system to resolve user-facing error strings by language.

## Directory Layout

```
librefang-types/locales/
├── de/errors.ftl       # German (partial)
├── en/errors.ftl       # English (canonical — most complete)
├── es/errors.ftl       # Spanish (partial)
├── fr/errors.ftl       # French (partial)
├── ja/errors.ftl       # Japanese (near-complete)
└── zh-CN/errors.ftl    # Simplified Chinese (partial)
```

## Message Format

All files use [Project Fluent](https://projectfluent.org/) syntax. Each message has an identifier and a string value, optionally with interpolated variables:

```ftl
# Simple message
api-error-agent-not-found = Agent not found

# Parameterized message
api-error-message-delivery-failed = Message delivery failed: { $reason }
```

## Naming Convention

Identifiers follow the pattern `api-error-{domain}-{description}`:

| Segment | Meaning |
|---------|---------|
| `api-error` | Fixed prefix — all messages are API error responses |
| `{domain}` | Feature area: `agent`, `message`, `template`, `manifest`, `auth`, `session`, `workflow`, `trigger`, `budget`, `config`, `profile`, `cron`, `goal`, `memory`, `network`, `plugin`, `channel`, `provider`, `skill`, `hand`, `mcp`, `integration`, `extension`, `system`, `kv`, `approval`, `webhook`, `backup`, `schedule`, `job`, `task`, `pairing`, `binding`, `command`, `file`, `tool`, `validation` |
| `{description}` | kebab-case summary of the specific error |

## Coverage by Domain

**English (`en`)** is the canonical locale and defines every key. Other locales translate a subset. The major domains are:

| Domain | Example Keys | Key Count (en) |
|--------|-------------|----------------|
| agent | `agent-not-found`, `agent-spawn-failed`, `agent-execution-failed` | 17 |
| provider | `provider-missing-alias`, `provider-alias-exists`, `provider-secret-write-failed` | 21 |
| goal | `goal-not-found`, `goal-circular-parent`, `goal-save-failed` | 19 |
| file | `file-not-found`, `file-path-traversal`, `file-unsupported-type` | 16 |
| webhook | `webhook-not-enabled`, `webhook-url-unreachable` | 11 |
| skill | `skill-missing-name`, `skill-invalid-name`, `skill-install-failed` | 10 |
| backup | `backup-invalid-archive`, `backup-missing-manifest` | 11 |
| schedule | `schedule-not-found`, `schedule-invalid-cron` | 10 |
| memory | `memory-not-enabled`, `memory-missing-kv` | 9 |
| cron | `cron-invalid-expression`, `cron-missing-field` | 6 |
| session | `session-load-failed`, `session-cleanup-expired-failed` | 7 |
| network | `network-a2a-not-found`, `network-auth-failed` | 7 |
| mcp | `mcp-missing-name`, `mcp-invalid-config` | 5 |
| Others | (template, manifest, auth, workflow, trigger, budget, config, profile, plugin, channel, integration, extension, system, kv, approval, job, task, pairing, binding, command, tool, validation, general) | varies |

## Locale Completeness

English and Japanese are the most complete. German, Spanish, French, and Simplified Chinese cover the core set of older domains (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, general) but lack newer domains added later (goal, memory, network, plugin, channel, provider, skill, hand, mcp, webhook, backup, schedule, job, task, pairing, binding, command, file, tool, validation, integration, extension, system, kv, approval).

**When adding a new error message**, always add it to `en/errors.ftl` first, then add translations to `ja/errors.ftl`. The other locales can be translated incrementally — the Fluent system will fall back to English for missing keys.

## Interpolated Variables

Messages use Fluent variables wrapped in `{ $name }`. The following variable names appear across the locales:

| Variable | Used In | Example |
|----------|---------|---------|
| `$reason` | `bad-request`, `message-delivery-failed` | `Bad request: { $reason }` |
| `$error` | `*-failed`, `*-parse-failed` keys | `Failed to parse template: { $error }` |
| `$name` | `template-not-found`, `profile-not-found`, `command-not-found` | `Template '{ $name }' not found` |
| `$id` | `agent-not-found-with-id`, `goal-not-found-with-id`, `hand-not-found` | `Agent not found: { $id }` |
| `$step` | `workflow-step-needs-agent` | `Step '{ $step }' needs 'agent_id'` |
| `$alias` | `provider-alias-exists`, `provider-alias-not-found` | `Alias '{ $alias }' already exists` |
| `$url` | `network-a2a-not-found`, `network-missing-url` | `A2A agent '{ $url }' not found` |
| `$status` | `network-auth-failed` | `Authentication failed (HTTP { $status })` |
| `$field` | `agent-invalid-sort` | `Invalid sort field '{ $field }'` |
| `$valid` | `agent-invalid-sort`, `webhook-unknown-event` | `Valid fields: { $valid }` |
| `$max` | `skill-description-too-long`, `file-too-large` | `max { $max }` |
| `$event` | `webhook-unknown-event` | `Unknown event type '{ $event }'` |
| `$provider` | `provider-model-exists` | `already exists in provider '{ $provider }'` |

## How It Connects to the Codebase

The consuming code loads these FTL files via a Fluent bundle (typically `fluent-templates` or `fluent-bundle` in Rust). At runtime:

1. The application initializes a Fluent localization bundle for the user's locale
2. When the API needs to return an error, it looks up the message identifier (e.g., `api-error-agent-not-found`)
3. The bundle resolves the message in the current locale, interpolating any variables
4. If the current locale lacks a key, Fluent falls back through the locale chain to English

The error identifiers in these `.ftl` files are the single source of truth for error message text. Backend error-handling code references these identifiers by name; the actual human-readable strings live exclusively in these locale files.

## Adding a New Error Message

1. Add the key to `locales/en/errors.ftl` under the appropriate domain comment section
2. Follow the naming pattern: `api-error-{domain}-{description}`
3. Use `{ $variable }` interpolation for any dynamic content — do not embed runtime values directly
4. Add the same key to `locales/ja/errors.ftl` with a Japanese translation
5. Optionally add translations to `de`, `es`, `fr`, `zh-CN`

## Adding a New Locale

1. Create `locales/{locale-code}/errors.ftl`
2. Copy the full contents of `en/errors.ftl` as a starting point
3. Translate all message values — do **not** change the identifiers or variable names
4. Register the new locale in the application's Fluent bundle configuration