# Other — librefang-types-locales

# librefang-types-locales

Fluent (FTL) translation files for API error messages, providing localized user-facing error strings across six languages.

## Overview

This module holds the **authoritative error message catalog** for the LibreFang API layer. Every error returned by API route handlers resolves to a Fluent message key defined here. The files are consumed at runtime by the Fluent localization system (typically via `fluent-templates` or similar Rust/crate bindings) and interpolated with context variables before being sent as HTTP response bodies.

```
librefang-types/locales/
├── de/errors.ftl      # German
├── en/errors.ftl      # English (canonical / most complete)
├── es/errors.ftl      # Spanish
├── fr/errors.ftl      # French
├── ja/errors.ftl      # Japanese
└── zh-CN/errors.ftl   # Simplified Chinese
```

## Key Conventions

### Message ID Format

All keys follow the pattern:

```
api-error-<domain>-<detail>
```

For example, `api-error-agent-not-found` is the agent domain, "not found" detail. This naming makes it straightforward to locate the source of any error in route handler code via grep.

### Interpolation Variables

Messages may include Fluent variables wrapped in `{ $var }`:

| Variable | Used by | Purpose |
|----------|---------|---------|
| `$error` | `*-failed`, `api-error-generic` | Underlying error string from the system |
| `$reason` | `api-error-bad-request`, `api-error-message-delivery-failed` | Human-readable reason for rejection |
| `$name` | Template, provider, webhook keys | Name of the referenced resource |
| `$id` | Agent, goal, hand, provider keys | Identifier of the referenced resource |
| `$step` | `api-error-workflow-step-needs-agent` | Specific workflow step causing the error |
| `$field` | `api-error-agent-invalid-sort` | Invalid field name |
| `$valid` | `api-error-agent-invalid-sort`, `api-error-webhook-unknown-event` | List of valid values |
| `$max` | `api-error-file-too-large`, `api-error-skill-description-too-long` | Maximum allowed value |
| `$status` | `api-error-network-auth-failed` | HTTP status code |
| `$url` | `api-error-network-a2a-not-found` | URL of a remote resource |
| `$provider` | `api-error-provider-model-exists` | Provider name |
| `$alias` | Provider alias keys | Alias string |
| `$event` | `api-error-webhook-unknown-event` | Event type string |

### The `api-error-generic` Catch-All

The `api-error-generic` key is a critical stopgap used by 41+ HTTP 500 handlers. It interpolates the raw `$error` string verbatim:

```fluent
api-error-generic = { $error }          # en
api-error-generic = Fehler: { $error }   # de
api-error-generic = エラー: { $error }    # ja
```

If this key is missing from a locale file, any `t_args("api-error-generic", …)` call will return the literal key name as the response body, and the `$error` interpolation will never run. **Do not remove this key from any locale.**

## Error Domains

The English (`en`) file is canonical and contains the most keys. Below is a summary of all domains and their coverage:

| Domain | Example Key | en Keys | Description |
|--------|-------------|---------|-------------|
| **agent** | `api-error-agent-not-found` | ~16 | Agent lifecycle: spawn, clone, execution, workspace |
| **message** | `api-error-message-too-large` | 5 | Inter-agent messaging, streaming |
| **template** | `api-error-template-not-found` | 6 | Template parsing, manifest validation |
| **manifest** | `api-error-manifest-signature-mismatch` | 5 | Manifest format and signature checks |
| **auth** | `api-error-auth-invalid-key` | 3 | API key validation, header checks |
| **session** | `api-error-session-not-found` | 6 | Session CRUD, cleanup |
| **workflow** | `api-error-workflow-missing-steps` | 5 | Multi-step workflow orchestration |
| **trigger** | `api-error-trigger-missing-pattern` | 7 | Event trigger registration and lookup |
| **budget** | `api-error-budget-invalid-amount` | 3 | Cost limits and budget updates |
| **config** | `api-error-config-parse-failed` | 5 | TOML configuration read/write |
| **profile** | `api-error-profile-not-found` | 1 | Named profile lookup |
| **cron** | `api-error-cron-invalid-expression` | 6 | Scheduled job creation and validation |
| **goal** | `api-error-goal-circular-parent` | 16 | Goal hierarchy, validation, CRUD |
| **memory** | `api-error-memory-not-enabled` | 9 | Proactive memory, KV store, import/export |
| **network** | `api-error-network-a2a-not-found` | 7 | Peer networking, A2A protocol |
| **plugin** | `api-error-plugin-invalid-source` | 5 | Plugin installation sources |
| **channel** | `api-error-channel-unknown` | 4 | Inter-agent channel routing |
| **provider** | `api-error-provider-alias-exists` | 21 | LLM provider config, secrets, models |
| **skill** | `api-error-skill-name-too-long` | 9 | Skill definition and installation |
| **hand** | `api-error-hand-not-found` | 3 | Hand definition and instance lookup |
| **mcp** | `api-error-mcp-missing-transport` | 5 | MCP server configuration |
| **integration / extension** | `api-error-integration-not-found` | 3 | Extension system |
| **system** | `api-error-system-cli-not-found` | 1 | CLI availability |
| **kv** | `api-error-kv-missing-fields` | 4 | Structured key-value memory |
| **approval** | `api-error-approval-not-found` | 2 | Human-in-the-loop approvals |
| **webhook** | `api-error-webhook-url-unreachable` | 10 | Outgoing webhook delivery |
| **backup** | `api-error-backup-missing-manifest` | 10 | Backup creation and restoration |
| **schedule** | `api-error-schedule-invalid-cron` | 9 | Named schedule management |
| **job** | `api-error-job-not-retryable` | 5 | Background job lifecycle |
| **task** | `api-error-task-not-found` | 2 | Task resolution |
| **pairing** | `api-error-pairing-invalid-token` | 2 | Device/client pairing |
| **binding** | `api-error-binding-out-of-range` | 1 | Binding index validation |
| **command** | `api-error-command-not-found` | 1 | CLI command lookup |
| **file** | `api-error-file-path-traversal` | 14 | Upload/download, path security |
| **tool** | `api-error-tool-invoke-denied` | 5 | Direct tool invocation, allowlist |
| **validation** | `api-error-validation-color-invalid` | 5 | Field-level input validation |
| **general** | `api-error-internal` | 4 | HTTP-level catch-alls (404, 500, 429) |

## Locale Coverage Matrix

Not all locales have full parity with English. The matrix below shows which domains are translated:

| Domain | en | ja | de | es | fr | zh-CN |
|--------|:--:|:--:|:--:|:--:|:--:|:-----:|
| agent | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| message | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| template | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| manifest | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| auth | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| session | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| workflow | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| trigger | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| budget | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| config | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| profile | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| cron | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| goal | ✅ | ✅ | — | — | — | — |
| memory | ✅ | ✅ | — | — | — | — |
| network | ✅ | ✅ | — | — | — | — |
| plugin | ✅ | ✅ | — | — | — | — |
| channel | ✅ | ✅ | — | — | — | — |
| provider | ✅ | ✅ | — | — | — | — |
| skill | ✅ | ✅ | — | — | — | — |
| hand | ✅ | ✅ | — | — | — | — |
| mcp | ✅ | ✅ | — | — | — | — |
| integration/extension | ✅ | ✅ | — | — | — | — |
| system | ✅ | ✅ | — | — | — | — |
| kv | ✅ | ✅ | — | — | — | — |
| approval | ✅ | ✅ | — | — | — | — |
| webhook | ✅ | ✅ | — | — | — | ✅* |
| backup | ✅ | ✅ | — | — | — | — |
| schedule | ✅ | ✅ | — | — | — | — |
| job | ✅ | ✅ | — | — | — | — |
| task | ✅ | ✅ | — | — | — | — |
| pairing | ✅ | ✅ | — | — | — | — |
| binding | ✅ | ✅ | — | — | — | — |
| command | ✅ | ✅ | — | — | — | — |
| file | ✅ | ✅ | — | — | — | — |
| tool | ✅ | ✅ | — | — | — | ✅* |
| validation | ✅ | ✅ | — | — | — | — |

✅\* = Partial coverage (subset of keys).

When a key is missing from a non-English locale, the Fluent fallback chain resolves to the English version. This means **all routes function correctly regardless of locale completeness**, but users see English for untranslated domains.

## Adding a New Error Message

1. **Add the key to `en/errors.ftl` first** — this is the canonical source.
2. **Copy the key to all other locale files** with translated text, or leave it absent (English fallback applies).
3. **Follow the naming convention**: `api-error-<domain>-<detail>`.
4. **Use interpolation variables** for dynamic content rather than string concatenation in code:

```fluent
# Good — uses interpolation
api-error-template-not-found = Template '{ $name }' not found

# Avoid — no context for translators
api-error-template-not-found = Template not found
```

5. **If the error is used by an HTTP 500 handler**, consider whether `api-error-generic` suffices temporarily, but plan to create a typed key.

## Adding a New Locale

1. Create `librefang-types/locales/<locale-code>/errors.ftl`.
2. At minimum, translate the **general** domain keys (`api-error-not-found`, `api-error-internal`, `api-error-bad-request`, `api-error-rate-limited`, `api-error-generic`). These are the most user-visible.
3. Register the locale in whatever Fluent loader configuration the application uses.
4. Translate remaining domains incrementally — the fallback mechanism ensures nothing breaks.

## Relationship to Route Handlers

Route handlers reference these keys by name using the Fluent binding's lookup function (e.g., `t_args("api-error-agent-not-found")` or `t_args("api-error-generic", error = ...)`)`). The module itself has no executable code — it is pure data consumed by the i18n layer at runtime. The long-term migration path, noted in the source comments, is to replace bare `api-error-generic` calls with typed, domain-specific error keys backed by a `MemoryRouteError`-style enum so that every error is self-documenting and locale-independent in the type system.