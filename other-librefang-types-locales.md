# Other — librefang-types-locales

# librefang-types-locales

Fluent (`.ftl`) locale files that define every user-facing API error string for the LibreFang platform. These files are pure data — no executable code, no function calls — consumed at runtime by the Fluent localization runtime (typically via `fluent-templates` or similar) to produce localized error responses.

## Purpose

Every HTTP error the LibreFang API returns to a client is keyed from this catalogue. The English file (`en/errors.ftl`) is the canonical source of truth; the other locale files provide translations. Keeping all error text in Fluent files rather than hardcoded in route handlers achieves two things:

1. **Separation of concerns** — route handlers reference a stable message identifier (e.g. `"api-error-agent-not-found"`) and never embed prose.
2. **Extensible localization** — adding a new language is a matter of adding another `errors.ftl`, with zero changes to application code.

## File Layout

```
librefang-types/locales/
├── de/errors.ftl       # German
├── en/errors.ftl       # English  ← canonical / most complete
├── es/errors.ftl       # Spanish
├── fr/errors.ftl       # French
├── ja/errors.ftl       # Japanese
└── zh-CN/errors.ftl    # Simplified Chinese
```

## Fluent Syntax Quick Reference

Each entry follows the Fluent pattern:

```fluent
# Simple key
api-error-agent-not-found = Agent not found

# Key with interpolation — variables are wrapped in { $name }
api-error-message-delivery-failed = Message delivery failed: { $reason }

# Key with multiple variables
api-error-agent-not-found-with-id = Agent not found: { $id }
```

At runtime the application calls something equivalent to:

```rust
// Pseudocode — actual call site uses whatever fluent helper the project provides
t_args("api-error-message-delivery-failed", {"reason": "timeout"})
// → "Message delivery failed: timeout"
```

## Coverage by Domain

Error keys are prefixed by the subsystem they belong to. The English file is the most exhaustive reference. Below is a summary of every domain and a representative key.

| Domain | Prefix | Example key | Interpolated vars |
|---|---|---|---|
| Agent | `api-error-agent-*` | `api-error-agent-not-found-with-id` | `$id`, `$error`, `$field`, `$valid` |
| Message | `api-error-message-*` | `api-error-message-delivery-failed` | `$reason` |
| Template | `api-error-template-*` | `api-error-template-not-found` | `$name`, `$error` |
| Manifest | `api-error-manifest-*` | `api-error-manifest-signature-mismatch` | — |
| Auth | `api-error-auth-*` | `api-error-auth-missing-header` | — |
| Session | `api-error-session-*` | `api-error-session-cleanup-expired-failed` | `$error` |
| Workflow | `api-error-workflow-*` | `api-error-workflow-step-needs-agent` | `$step` |
| Trigger | `api-error-trigger-*` | `api-error-trigger-registration-failed` | — |
| Budget | `api-error-budget-*` | `api-error-budget-provide-at-least-one` | — |
| Config | `api-error-config-*` | `api-error-config-parse-failed` | `$error` |
| Profile | `api-error-profile-*` | `api-error-profile-not-found` | `$name` |
| Cron | `api-error-cron-*` | `api-error-cron-create-failed` | `$error` |
| Goal | `api-error-goal-*` | `api-error-goal-parent-not-found` | `$id`, `$error` |
| Memory | `api-error-memory-*` | `api-error-memory-missing-kv` | — |
| Network / A2A | `api-error-network-*` | `api-error-network-auth-failed` | `$url`, `$error`, `$status` |
| Plugin | `api-error-plugin-*` | `api-error-plugin-invalid-source` | — |
| Channel | `api-error-channel-*` | `api-error-channel-invalid-from` | — |
| Provider | `api-error-provider-*` | `api-error-provider-model-exists` | `$id`, `$provider`, `$alias`, `$name`, `$error` |
| Skill | `api-error-skill-*` | `api-error-skill-description-too-long` | `$max`, `$error` |
| Hand | `api-error-hand-*` | `api-error-hand-not-found` | `$id` |
| MCP | `api-error-mcp-*` | `api-error-mcp-not-found` | `$name`, `$error` |
| Integration | `api-error-integration-*` | `api-error-integration-not-found` | `$id` |
| Extension | `api-error-extension-*` | `api-error-extension-not-found` | `$id` |
| System | `api-error-system-*` | `api-error-system-cli-not-found` | — |
| KV | `api-error-kv-*` | `api-error-kv-missing-path` | — |
| Approval | `api-error-approval-*` | `api-error-approval-not-found` | — |
| Webhook | `api-error-webhook-*` | `api-error-webhook-url-unreachable` | `$error`, `$event`, `$valid` |
| Backup | `api-error-backup-*` | `api-error-backup-invalid-archive` | `$error` |
| Schedule | `api-error-schedule-*` | `api-error-schedule-save-failed` | `$error` |
| Job | `api-error-job-*` | `api-error-job-not-retryable` | — |
| Task | `api-error-task-*` | `api-error-task-disappeared` | — |
| Pairing | `api-error-pairing-*` | `api-error-pairing-invalid-token` | — |
| Binding | `api-error-binding-*` | `api-error-binding-out-of-range` | — |
| Command | `api-error-command-*` | `api-error-command-not-found` | `$name` |
| File / Upload | `api-error-file-*` | `api-error-file-too-large` | `$max` |
| Tool | `api-error-tool-*` | `api-error-tool-not-found` | `$name` |
| Validation | `api-error-validation-*` | `api-error-validation-avatar-url-invalid` | — |
| General | `api-error-*` (unprefixed) | `api-error-bad-request` | `$reason` |
| Catch-all | `api-error-generic` | `api-error-generic` | `$error` |

## The `api-error-generic` Catch-All

This key deserves special attention:

```fluent
api-error-generic = { $error }
# or in non-English locales, e.g. German:
api-error-generic = Fehler: { $error }
```

It is used by **41+ HTTP 500 handlers** as a temporary bridge. Any route that hasn't yet been migrated to a typed `MemoryRouteError`-style helper falls back to `t_args("api-error-generic", …)`. If this key were missing, every such handler would return the literal string `"api-error-generic"` as the response body — the `$error` interpolation would never run.

**When adding new error keys**, prefer creating a specific typed key in the relevant domain (e.g. `api-error-agent-clone-spawn-failed`) rather than relying on `api-error-generic`.

## Locale Completeness

The English file defines the full set of keys. Other locales vary in coverage:

| Locale | Approximate key count | Notes |
|---|---|---|
| **en** | ~170+ | Complete — canonical source |
| **ja** | ~170+ | Near-complete parity with English |
| **de** | ~35 | Core subset only |
| **es** | ~35 | Core subset only |
| **fr** | ~35 | Core subset only |
| **zh-CN** | ~40 | Core subset + tool errors |

When a key is missing from a non-English locale, the Fluent runtime falls back to English. This means **all keys must exist in the English file** for fallback to work correctly.

## Adding a New Error Key

1. **Add the key to `en/errors.ftl`** in the correct domain section. Use the naming convention `api-error-<domain>-<description>`.
2. If the message needs dynamic data, declare a Fluent variable: `{ $variableName }`.
3. **Add translations** to every other locale file. If a full translation isn't ready, you can omit the key — the English fallback will be used.
4. **Reference the key** from the route handler via the `t_args` helper (or equivalent).

Example:

```fluent
# en/errors.ftl
api-error-agent-paused = Agent is paused and cannot accept messages
```

```fluent
# ja/errors.ftl
api-error-agent-paused = エージェントは一時停止中でメッセージを受信できません
```

## Adding a New Locale

1. Create `librefang-types/locales/<locale-code>/errors.ftl`.
2. Copy the structure from `en/errors.ftl` and translate each value.
3. Register the locale with whatever Fluent loader the application uses (typically a build.rs or runtime scan of the `locales/` directory).

## Conventions

- **Naming**: always `api-error-<domain>-<description>`, lowercase, hyphen-separated.
- **Interpolation variables**: use `$camelCase` names that describe the data (e.g. `$reason`, `$error`, `$name`, `$id`, `$step`, `$max`).
- **Comments**: each domain section is preceded by a `# --- <Domain> errors ---` comment. Maintain this for readability.
- **No trailing whitespace or blank keys** — Fluent is sensitive to these.
- **Domain ordering** follows the same sequence across all locale files to make diffing practical.