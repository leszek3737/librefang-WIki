# Other — librefang-types-locales

# librefang-types — Locales (API Error Messages)

## Overview

This directory contains localized API error messages for LibreFang, stored as [Project Fluent](https://projectfluent.org/) (`.ftl`) resource files. Every user-facing error returned by the LibreFang API resolves its display string through these locale files, keeping error text separate from application logic.

```
librefang-types/locales/
├── de/errors.ftl      # German
├── en/errors.ftl      # English  (canonical / most complete)
├── es/errors.ftl      # Spanish
├── fr/errors.ftl      # French
├── ja/errors.ftl      # Japanese
└── zh-CN/errors.ftl   # Simplified Chinese
```

There is no executable code in this module. The `.ftl` files are pure translation resources consumed at runtime by the Fluent localization framework bundled into the LibreFang type definitions.

---

## Fluent Message Format

Each message follows the Fluent syntax:

```fluent
message-key = Translated text
message-with-variable = Operation failed: { $reason }
```

- **Keys** use kebab-case prefixed with `api-error-` and grouped by domain (e.g. `api-error-agent-not-found`).
- **Variables** are interpolated with `{ $variable_name }` — these are supplied by the calling code at runtime.

---

## Message Key Convention

All keys follow the pattern:

```
api-error-<domain>-<detail>
```

| Segment | Description |
|---|---|
| `api-error-` | Fixed prefix — distinguishes these from UI or CLI messages |
| `<domain>` | Resource or subsystem: `agent`, `session`, `provider`, `webhook`, etc. |
| `<detail>` | Specific error condition: `not-found`, `invalid-id`, `missing-pattern`, etc. |

### Variable interpolation patterns

Most variables pass through context from the API layer:

| Variable | Typical source |
|---|---|
| `{ $reason }` | A human-readable explanation of why a request was rejected |
| `{ $error }` | An underlying error string from a subsystem |
| `{ $name }` | A resource identifier (template name, profile name, etc.) |
| `{ $id }` | A UUID or resource ID |
| `{ $step }` | A workflow step identifier |
| `{ $alias }` | A provider alias |
| `{ $provider }` | A provider name |
| `{ $url }` | A URL argument |
| `{ $max }` | A size limit value |
| `{ $field }` / `{ $valid }` | Sort field validation details |
| `{ $status }` | HTTP status code |
| `{ $event }` | Event type name |

---

## Coverage by Domain

The **English** locale is the canonical source and contains the full message set. The following table summarizes the error domains and their coverage across locales.

| Domain | en | ja | de | es | fr | zh-CN |
|---|:---:|:---:|:---:|:---:|:---:|:---:|
| Agent | ✅ 17 | ✅ 13 | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 |
| Message | ✅ 5 | ✅ 5 | ✅ 2 | ✅ 2 | ✅ 2 | ✅ 2 |
| Template | ✅ 6 | ✅ 6 | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 |
| Manifest | ✅ 5 | ✅ 5 | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 |
| Auth | ✅ 3 | ✅ 3 | ✅ 3 | ✅ 3 | ✅ 3 | ✅ 3 |
| Session | ✅ 7 | ✅ 6 | ✅ 2 | ✅ 2 | ✅ 2 | ✅ 2 |
| Workflow | ✅ 5 | ✅ 5 | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 |
| Trigger | ✅ 7 | ✅ 7 | ✅ 7 | ✅ 7 | ✅ 7 | ✅ 7 |
| Budget | ✅ 3 | ✅ 3 | ✅ 2 | ✅ 2 | ✅ 2 | ✅ 2 |
| Config | ✅ 5 | ✅ 5 | ✅ 2 | ✅ 2 | ✅ 2 | ✅ 2 |
| Profile | ✅ 1 | ✅ 1 | ✅ 1 | ✅ 1 | ✅ 1 | ✅ 1 |
| Cron | ✅ 6 | ✅ 6 | ✅ 3 | ✅ 3 | ✅ 3 | ✅ 3 |
| Goal | ✅ 16 | ✅ 16 | — | — | — | — |
| Memory | ✅ 9 | ✅ 9 | — | — | — | — |
| Network/A2A | ✅ 7 | ✅ 7 | — | — | — | — |
| Plugin | ✅ 5 | ✅ 5 | — | — | — | — |
| Channel | ✅ 4 | ✅ 4 | — | — | — | — |
| Provider | ✅ 20 | ✅ 20 | — | — | — | — |
| Skill | ✅ 9 | ✅ 9 | — | — | — | — |
| Hand | ✅ 3 | ✅ 3 | — | — | — | — |
| MCP | ✅ 5 | ✅ 5 | — | — | — | — |
| Integration/Extension | ✅ 3 | ✅ 3 | — | — | — | — |
| System | ✅ 1 | ✅ 1 | — | — | — | — |
| KV | ✅ 4 | ✅ 4 | — | — | — | — |
| Approval | ✅ 2 | ✅ 2 | — | — | — | — |
| Webhook | ✅ 13 | ✅ 10 | — | — | — | — |
| Backup | ✅ 11 | ✅ 11 | — | — | — | — |
| Schedule | ✅ 9 | ✅ 9 | — | — | — | — |
| Job | ✅ 5 | ✅ 5 | — | — | — | — |
| Task | ✅ 2 | ✅ 2 | — | — | — | — |
| Pairing | ✅ 2 | ✅ 2 | — | — | — | — |
| Binding | ✅ 1 | ✅ 1 | — | — | — | — |
| Command | ✅ 1 | ✅ 1 | — | — | — | — |
| File/Upload | ✅ 14 | ✅ 14 | — | — | — | — |
| Tool | ✅ 2 | ✅ 2 | — | — | — | — |
| Validation | ✅ 5 | ✅ 5 | — | — | — | — |
| General | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 | ✅ 4 |

### Locale tiers

- **Tier 1 (complete)** — `en`, `ja`: All domains represented. Japanese tracks closely with English.
- **Tier 2 (base)** — `de`, `es`, `fr`, `zh-CN`: Core domains only (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, general).

---

## Adding a New Error Message

1. **Add the key to `en/errors.ftl`** first. English is the canonical locale and acts as the fallback when a key is missing in another language.

   ```fluent
   api-error-<domain>-<detail> = English description here
   ```

2. **Add translations** to every other locale file that covers that domain. If a locale does not include the domain at all (e.g., German has no `goal` messages), the Fluent runtime will fall back to English for that key.

3. **Follow naming rules**:
   - Prefix: `api-error-`
   - Use kebab-case throughout.
   - Group by domain in the file, matching existing section comments (e.g. `# Agent errors`).

4. **If the message needs runtime data**, use a Fluent variable:
   ```fluent
   api-error-foo-bar-failed = Foo bar failed: { $reason }
   ```
   Ensure the calling code passes `reason` into the Fluent message resolver.

---

## Adding a New Locale

1. Create a new directory under `locales/` using an [BCP 47](https://tools.ietf.org/html/bcp47) language tag (e.g. `pt-BR/`).
2. Create `errors.ftl` inside it.
3. Translate messages from `en/errors.ftl`. At minimum, include the Tier 2 core domains to provide a functional experience.
4. Register the new locale in the application's Fluent bundle initialization (outside this module).

---

## Fallback Behavior

When a requested message key is absent from the active locale, the Fluent resolver falls back through the locale chain and ultimately resolves against the English bundle. This means:

- A German user will see English text for any domain not present in `de/errors.ftl` (e.g., goals, memory, webhooks).
- **Missing keys never produce runtime crashes** — they surface as the English string or the raw key name if even the English bundle lacks it.

---

## Relationship to the Rest of the Codebase

```
┌──────────────────────┐
│   API route handlers  │
│  (construct error     │
│   response with key)  │
└─────────┬────────────┘
          │ looks up key + passes args
          ▼
┌──────────────────────┐
│   Fluent bundle       │
│  (loaded from         │
│   locales/<lang>/     │
│   errors.ftl)         │
└─────────┬────────────┘
          │ reads .ftl at startup
          ▼
┌──────────────────────┐
│  librefang-types/     │
│  locales/             │
│    en/errors.ftl      │
│    de/errors.ftl      │
│    ...                │
└──────────────────────┘
```

The `librefang-types` crate defines the shared type system and, through this locales directory, the canonical error message catalog. API handlers reference keys by name (e.g., `api-error-agent-not-found`) and delegate formatting to the Fluent runtime, which loads the appropriate `.ftl` file based on the user's language preference. This separation ensures that error messages can be corrected or translated without modifying business logic.