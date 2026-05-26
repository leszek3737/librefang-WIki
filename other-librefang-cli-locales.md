# Other — librefang-cli-locales

# librefang-cli-locales

Fluent (FTL) message catalogs for the LibreFang CLI. Provides all user-facing strings—status messages, error reports, hints, and interactive prompts—in a structured, localizable format. Currently ships English (`en`) as the primary locale and Simplified Chinese (`zh-CN`) as a complete translation.

## Architecture

```mermaid
graph LR
    CLI["librefang-cli<br/>(Rust binary)"] -->|"fluent::FluentBundle<br/>load(\"en/main.ftl\")"| EN["locales/<b>en</b>/main.ftl"]
    CLI -->|"load(\"zh-CN/main.ftl\")"| ZH["locales/<b>zh-CN</b>/main.ftl"]
    EN -->|fallback| CLI
    ZH -->|override| CLI
```

The CLI uses the [Fluent](https://projectfluent.org/) system (Rust crate `fluent`) to resolve message IDs to localized strings at runtime. The English catalog is the canonical source; all other locales translate against it one-to-one.

## File Layout

```
librefang-cli/locales/
├── en/main.ftl       # Primary (canonical) catalog
└── zh-CN/main.ftl    # Simplified Chinese translation
```

Each file is a single self-contained Fluent resource with no imports.

## Message ID Conventions

### Prefixes group messages by subsystem

| Prefix | Domain | Example |
|--------|--------|---------|
| `daemon-*` | Daemon lifecycle (start, stop, restart, background) | `daemon-started-bg`, `daemon-stopped-forced` |
| `label-*` | Short field labels for status/info tables | `label-uptime`, `label-pid` |
| `hint-*` | Non-critical guidance shown after output | `hint-stop-daemon`, `hint-check-status` |
| `error-*` | Error titles and diagnostics | `error-boot-auth`, `error-connect-refused` |
| `agent-*` | Agent spawning, killing, model management | `agent-spawned`, `agent-kill-failed` |
| `vault-*` | Credential vault operations | `vault-stored`, `vault-rotate-success` |
| `cron-*` | Scheduled job CRUD | `cron-created`, `cron-deleted` |
| `config-*` | Config file reads/writes/edits | `config-set-kv`, `config-parse-error` |
| `channel-*` | Messaging channel setup (Discord, Slack, etc.) | `channel-test-ok`, `channel-unknown` |
| `section-*` | Table/section headers for status output | `section-daemon-status`, `section-active-agents` |
| `webhook-*` | Webhook management | `webhook-created`, `webhook-test-ok` |
| `memory-*` | Agent memory key-value operations | `memory-set`, `memory-deleted` |
| `uninstall-*` | Teardown and cleanup | `uninstall-goodbye`, `uninstall-removed` |
| `doctor-*` | Diagnostic checks | `doctor-all-passed`, `doctor-some-failed` |
| `model-*` | Model catalog selection | `model-set-success`, `model-no-catalog` |
| `approval-*` | Human-in-the-loop approvals | `approval-responded`, `approval-failed` |
| `device-*` | Mobile device pairing | `device-scan-qr`, `device-removed` |
| `guide-*` | Interactive setup wizard | `guide-title`, `guide-testing-key` |
| `shutdown-*` | Shutdown-specific edge cases | `shutdown-401-detected`, `shutdown-401-fallback-attempt` |
| `reset-*` / `log-*` | Reset and log tailing | `reset-success`, `log-following` |

### Error + fix pattern

Many error messages have a companion `*-fix` variant that provides remediation guidance. Consumers typically print the error message followed by the fix hint:

```ftl
error-boot-auth = LLM provider authentication failed
error-boot-auth-fix = Run `librefang doctor` to check your API key configuration
```

In code, these are resolved as a pair:

```rust
println!("{}: {}", ftl("error-boot-auth"), ftl("error-boot-auth-fix"));
```

### Value labels

Static display values for the security status table use a `value-*` prefix to pair with their `label-*` counterpart:

```ftl
label-audit-trail = Audit trail
value-audit-trail = Merkle hash chain (SHA-256)
```

## Variable Interpolation

Messages use Fluent's `$variable` syntax. All variables are positional and untyped (rendered as strings by the CLI before passing to Fluent). Common variables:

| Variable | Used in | Meaning |
|----------|---------|---------|
| `$error` | `daemon-error`, `vault-unlock-failed`, etc. | Underlying error string |
| `$status` | `error-daemon-returned`, `daemon-bg-exited` | HTTP or exit status code |
| `$path` | `log-path-hint`, `vault-rotate-no-vault` | Filesystem path |
| `$url` | `daemon-already-running`, `dashboard-opening` | Daemon/dashboard URL |
| `$count` | `models-available`, `agents-loaded`, `vault-rotate-success` | Numeric count |
| `$pid` | `shutdown-401-fallback-attempt`, `shutdown-401-fallback-success` | OS process ID |
| `$id` | `agent-killed`, `cron-deleted`, `webhook-created` | Entity identifier |
| `$name` | `agent-spawned`, `channel-configured` | Human-readable name |
| `$key` | `vault-stored`, `config-key-not-found` | Config or vault key |
| `$value` | `agent-model-set`, `config-set-kv` | Assigned value |
| `$command` | `error-require-daemon` | CLI subcommand name |
| `$provider` / `$model` | `kernel-booted`, `model-set-success` | LLM provider or model ID |
| `$display` / `$env_var` | `detected-provider`, `config-saved-key` | Provider display name, env var name |
| `$action` | `cron-toggled`, `approval-responded` | Past-tense action (paused/resumed, approved/rejected) |
| `$field` | `agent-unknown-field` | Field name in set operations |
| `$max` | `model-out-of-range` | Upper bound of selection range |

## Multiline Messages

Some remediation hints use Fluent's block syntax (indented continuation lines). For example, `shutdown-401-fallback-fix`:

```ftl
shutdown-401-fallback-fix = Stop the daemon manually, then start it again:
    kill { $pid }    # or: kill -9 { $pid } if it does not exit
    librefang start
```

The indented lines are part of the same message value. Consumers should preserve line breaks when rendering.

## Adding a New Locale

1. Create `librefang-cli/locales/<locale>/main.ftl` (e.g., `ja/main.ftl`).
2. Copy `en/main.ftl` as the starting template.
3. Translate every message value. **Do not change message IDs or variable names.**
4. Keep the same section comment structure for diff-ability.
5. Register the locale in the CLI's Fluent bundle loader (see the i18n initialization code in `librefang-cli`).

### Completeness check

Every message ID present in `en/main.ftl` must exist in the new locale file. Missing IDs will silently fall back to English if the bundle is configured with an `en` fallback chain, but this should be treated as a bug.

## Adding New Messages

1. Add the message to `en/main.ftl` under the appropriate section comment.
2. Follow the naming convention (`subsystem-variant` or `subsystem-variant-fix`).
3. Use `$variable` placeholders for any dynamic content—never hardcode formatted strings.
4. Immediately add the same message ID to all other locale files with a translation. Leaving other locales out of date is acceptable short-term only if fallback is configured.
5. If the message is an error, add a companion `*-fix` variant.

## Notable Design Decisions

**Issue #4693 shutdown flow.** The `shutdown-401-*` family of messages handles a specific upgrade scenario: `curl install.sh | sh` replaces the CLI binary while the old daemon keeps running with a different `api_key`. The messages explain the cause, attempt a PID-based fallback, and guide manual recovery if that also fails. The entire flow is driven through these seven message IDs rather than hardcoded English strings.

**Quick mode vs. interactive mode.** The `hint-non-interactive` and `hint-non-interactive-wizard` messages distinguish between piped/CI terminals and interactive ones. The `init-quick-success` / `init-interactive-success` pair reflects the same split in the init command.

**In-process vs. daemon agents.** Several message pairs distinguish ephemeral in-process agents from daemon-managed persistent ones:
- `agent-spawned` vs. `agent-spawned-inprocess`
- `section-daemon-status` vs. `section-status-inprocess`
- `hint-agent-lost-on-exit` and `hint-persistent-agents`