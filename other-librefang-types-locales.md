# Other — librefang-types-locales

# librefang-types-locales

Localization resources for LibreFang API error messages, using the [Fluent](https://projectfluent.org/) (`.ftl`) message format. This is a pure data module — no executable code, no build step. Consumed at runtime by the application's i18n layer via `t_args()` calls.

## Supported Languages

| Locale | Code | Coverage |
|--------|------|----------|
| English | `en` | Full — canonical source |
| German | `de` | Partial (core errors only) |
| Spanish | `es` | Partial (core errors only) |
| French | `fr` | Partial (core errors only) |
| Japanese | `ja` | Near-full |
| Simplified Chinese | `zh-CN` | Partial (core errors + tools) |

English (`en`) is the authoritative locale. All other locales should mirror its key structure; missing keys fall back to English at runtime.

## File Layout

```
librefang-types/locales/
├── de/errors.ftl
├── en/errors.ftl
├── es/errors.ftl
├── fr/errors.ftl
├── ja/errors.ftl
└── zh-CN/errors.ftl
```

Each file contains a single Fluent message group: API error strings.

## Key Naming Convention

All keys follow the pattern:

```
api-error-{domain}-{description}
```

Where **domain** maps to an API subsystem and **description** is a kebab-case summary of the failure mode.

## Error Domains

| Domain | Prefix | Typical HTTP Status | Example Keys |
|--------|--------|-------------------|--------------|
| Agent | `api-error-agent-*` | 404, 500 | `agent-not-found`, `agent-spawn-failed`, `agent-vanished` |
| Message | `api-error-message-*` | 400, 500 | `message-too-large`, `message-delivery-failed` |
| Template | `api-error-template-*` | 400, 404 | `template-not-found`, `template-parse-failed` |
| Manifest | `api-error-manifest-*` | 400, 500 | `manifest-signature-mismatch` |
| Auth | `api-error-auth-*` | 401, 403 | `auth-invalid-key`, `auth-missing-header` |
| Session | `api-error-session-*` | 400, 404 | `session-load-failed`, `session-not-found` |
| Workflow | `api-error-workflow-*` | 400, 500 | `workflow-missing-steps`, `workflow-execution-failed` |
| Trigger | `api-error-trigger-*` | 400, 404 | `trigger-missing-pattern`, `trigger-not-found` |
| Budget | `api-error-budget-*` | 400, 500 | `budget-invalid-amount` |
| Config | `api-error-config-*` | 400, 500 | `config-parse-failed`, `config-write-failed` |
| Profile | `api-error-profile-*` | 404 | `profile-not-found` |
| Cron | `api-error-cron-*` | 400, 404 | `cron-invalid-expression`, `cron-not-found` |
| Goal | `api-error-goal-*` | 400, 404 | `goal-circular-parent`, `goal-save-failed` |
| Memory | `api-error-memory-*` | 400, 404 | `memory-not-enabled`, `memory-serialization-error` |
| Network / A2A | `api-error-network-*` | 400, 500 | `network-connection-failed`, `network-auth-failed` |
| Plugin | `api-error-plugin-*` | 400 | `plugin-invalid-source`, `plugin-missing-name` |
| Channel | `api-error-channel-*` | 400 | `channel-missing-agent-id` |
| Provider | `api-error-provider-*` | 400, 404 | `provider-alias-exists`, `provider-not-found` |
| Skill | `api-error-skill-*` | 400, 500 | `skill-name-too-long`, `skill-install-failed` |
| Hand | `api-error-hand-*` | 404 | `hand-not-found` |
| MCP | `api-error-mcp-*` | 400, 404 | `mcp-invalid-config`, `mcp-not-found` |
| Integration | `api-error-integration-*` | 404 | `integration-not-found` |
| Extension | `api-error-extension-*` | 404 | `extension-not-found` |
| System | `api-error-system-*` | 500 | `system-cli-not-found` |
| KV | `api-error-kv-*` | 400 | `kv-missing-fields`, `kv-array-empty` |
| Approval | `api-error-approval-*` | 400, 404 | `approval-invalid-id` |
| Webhook | `api-error-webhook-*` | 400, 404 | `webhook-url-unreachable` |
| Backup | `api-error-backup-*` | 400, 404 | `backup-missing-manifest` |
| Schedule | `api-error-schedule-*` | 400, 404 | `schedule-save-failed` |
| Job | `api-error-job-*` | 400, 404 | `job-not-retryable` |
| Task | `api-error-task-*` | 404 | `task-disappeared` |
| Pairing | `api-error-pairing-*` | 403 | `pairing-not-enabled` |
| Binding | `api-error-binding-*` | 400 | `binding-out-of-range` |
| Command | `api-error-command-*` | 404 | `command-not-found` |
| File | `api-error-file-*` | 400, 404 | `file-path-traversal`, `file-too-large` |
| Tool | `api-error-tool-*` | 403, 404 | `tool-invoke-denied`, `tool-not-found` |
| Validation | `api-error-validation-*` | 400 | `validation-color-invalid` |
| General | `api-error-*` (no domain) | Various | `not-found`, `internal`, `rate-limited` |

## Variable Interpolation

Messages support Fluent variable substitution using `{ $variable }` syntax. The runtime call `t_args("api-error-generic", error = "disk full")` resolves `$error` inside the template.

**Common variables:**

| Variable | Used By | Meaning |
|----------|---------|---------|
| `$error` | Generic catch-all, parse/write failures | Underlying error string |
| `$reason` | `bad-request`, `message-delivery-failed` | Human-readable failure reason |
| `$name` | Template, profile, tool lookups | Resource identifier |
| `$id` | Agent, goal, hand lookups | Entity ID |
| `$step` | Workflow step validation | Step identifier |
| `$alias` | Provider alias operations | Provider alias string |
| `$provider` | Provider model conflict | Provider name |
| `$url` | Network, webhook, MCP | Endpoint URL |
| `$status` | Network auth failure | HTTP status code |
| `$max` | Skill description length | Character limit |
| `$field` | Sort validation | Field name |
| `$valid` | Sort, webhook event validation | List of valid values |
| `$event` | Webhook event errors | Event type name |

## The Generic Catch-All Key

```
api-error-generic = { $error }          # en
api-error-generic = Fehler: { $error }  # de
api-error-generic = Erreur : { $error } # fr
```

This key is a deliberate stopgap. Over 41 HTTP 500 handlers call `t_args("api-error-generic", error = ...)` to pass raw error strings through to the client. It exists in every locale specifically because omitting it would cause the Fluent resolver to return the literal key name as the response body, and `$error` interpolation would never execute.

New routes should prefer domain-specific typed errors (e.g., `MemoryRouteError`) over this catch-all. Once all routes are migrated, `api-error-generic` can be deprecated.

## How to Add a New Error Key

1. **Add to `en/errors.ftl` first.** Choose the correct domain prefix and write the message with any needed interpolation variables:
   ```fluent
   api-error-agent-suspended = Agent { $id } is suspended and cannot accept requests
   ```

2. **Add to all other locale files.** At minimum, add the key to `de`, `es`, `fr`, `ja`, and `zh-CN`. If a translation isn't ready, copy the English text as a placeholder — it's better to return English than to return a raw key.

3. **Reference from code** using the application's translation helper:
   ```rust
   t_args("api-error-agent-suspended", id = agent_id)
   ```

## Locale Coverage Gap

The partial locales (`de`, `es`, `fr`, `zh-CN`) only cover core domains (agent, message, template, manifest, auth, session, workflow, trigger, budget, config, profile, cron, and general). They are missing keys for: goal, memory, network, plugin, channel, provider, skill, hand, MCP, integration, extension, system, KV, approval, webhook, backup, schedule, job, task, pairing, binding, command, file, tool, and validation.

When adding functionality that targets international users, ensure the corresponding locale files are updated. The Fluent resolver will fall back to English for missing keys, so the application won't break — but users will see mixed-language error output.