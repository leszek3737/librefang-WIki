# Other — librefang-cli-templates

# librefang-cli-templates

Configuration templates shipped with the LibreFang CLI. These TOML files serve as the source material for two initialization paths: a fully-commented reference config (`init_default_config.toml`) and a wizard-generated minimal config (`init_wizard_config.toml`). Both contain Mustache-style placeholders that are rendered at runtime by the CLI's init machinery.

## Templates

### `init_default_config.toml`

A comprehensive, heavily annotated configuration file. Every section the daemon recognizes is present — most commented out — so a user can uncomment and edit rather than write from scratch. The inline comments include security warnings, valid value enumerations, and cross-references to documentation.

**Placeholders:**

| Placeholder | Replaced with |
|---|---|
| `{{provider}}` | LLM provider name (e.g. `openai`, `anthropic`, `ollama`) |
| `{{model}}` | Default model identifier (e.g. `gpt-4o`, `claude-sonnet-4-20250514`) |
| `{{api_key_env}}` | Environment variable name holding the API key |

### `init_wizard_config.toml`

A minimal configuration emitted by the interactive `librefang init` wizard. Only the sections the user actually configured during the Q&A are included, keeping the file short and uncluttered.

**Placeholders:**

| Placeholder | Replaced with |
|---|---|
| `{{provider}}` | LLM provider name |
| `{{model}}` | Default model identifier |
| `{{api_key_line}}` | A rendered `api_key_env = "..."` line (or empty string if not applicable) |
| `{{routing_section}}` | Optional agent routing block the wizard generated |

## Configuration Sections Reference

Below is a summary of every section defined across the templates, grouped by domain.

### Server Core

| Key | Default | Values |
|---|---|---|
| `api_listen` | `127.0.0.1:4545` | `host:port` — loopback-only until auth is configured |
| `log_level` | `info` | `trace`, `debug`, `info`, `warn`, `error` |
| `mode` | `default` | `stable`, `default`, `dev` |
| `update_channel` | `stable` | `stable`, `beta`, `rc` |

**Security guard:** The daemon refuses to bind a non-loopback address unless authentication is configured (`api_key`, dashboard credentials, or `[[users]]`). Override with `LIBREFANG_ALLOW_NO_AUTH=1` at your own risk.

### Dashboard & Authentication

```toml
dashboard_user = "librefang"
dashboard_pass = "librefang"   # change immediately after first login
```

Passwords can reference a vault (`"vault:dashboard_password"`) or the `LIBREFANG_DASHBOARD_PASS` environment variable.

Terminal access is controlled by two separate gates under `[terminal]`:

- `allow_remote` — permits remote terminal connections when auth is present
- `allow_unauthenticated_remote` — must be explicitly `true` to expose an unauthenticated shell (foot-gun guard)

### Default LLM (`[default_model]`)

```toml
[default_model]
provider = "openai"
model = "gpt-4o"
api_key_env = "OPENAI_API_KEY"
# base_url = ""   # optional proxy override
```

Failover chains are defined via `[[fallback_providers]]` array-of-tables entries.

### Memory & Proactive Memory

```toml
[memory]
decay_rate = 0.05           # confidence decay per cycle

[proactive_memory]
enabled = true
auto_memorize = true        # extract facts from conversations
auto_retrieve = true        # recall relevant memories
max_retrieve = 10
```

Additional tunables (`extraction_threshold`, `session_ttl_hours`, `duplicate_threshold`, `max_memories_per_agent`) are commented out in the default template.

### Task Queue (`[queue.concurrency]`)

Controls parallelism across four lanes:

| Lane | Default | Purpose |
|---|---|---|
| `main_lane` | 3 | Concurrent user messages |
| `cron_lane` | 2 | Scheduled jobs |
| `subagent_lane` | 3 | Child agent invocations |
| `trigger_lane` | 8 | Global in-flight trigger dispatches |
| `default_per_agent` | 1 | Per-agent fallback (serial behavior) |

### Shell Execution Policy (`[exec_policy]`)

```toml
[exec_policy]
mode = "deny"             # deny | allowlist | full
timeout_secs = 30
max_output_bytes = 102400  # 100 KB
```

### Hot-Reload (`[reload]`)

```toml
[reload]
mode = "hybrid"    # off | restart | hot | hybrid
debounce_ms = 500
```

- **`hot`** — picks up changes without restarting.
- **`restart`** — schedules a graceful restart.
- **`hybrid`** — applies what it can hot, restarts for the rest.

### Optional Integrations

Each of these is fully commented out in the default template, ready to uncomment:

| Section | Purpose |
|---|---|
| `[web]` / `[web.fetch]` | Web search (auto-detects among Tavily, Brave, Jina, Perplexity, DuckDuckGo) and page fetching with SSRF protection |
| `[compaction]` | LLM-based session history summarization |
| `[triggers]` | Event-driven trigger system with cooldown and recursion limits |
| `[budget]` | Cost caps (hourly/daily/monthly) with per-provider overrides |
| `[thinking]` | Extended thinking budget for Claude models |
| `[channels.telegram]` / `[channels.discord]` / `[channels.slack]` / `[channels.wechat]` | Messaging channel adapters |
| `[[mcp_servers]]` | Model Context Protocol external tool servers |
| `[browser]` | Headless browser automation |
| `[docker]` | Docker sandbox for code execution |
| `[inbox]` | File-based async command intake |
| `[network]` | P2P federation between LibreFang nodes |
| `[rate_limit]` | GCRA-based API and WebSocket rate limiting |
| `[registry]` | Agent registry cache TTL |

### Editor Integration (ACP)

No configuration keys are required. The daemon exposes an Agent Client Protocol (ACP) server at `~/.librefang/acp.sock` (Unix). Editors spawn `librefang acp` as a child process; the command auto-selects in-process mode or daemon-attached mode based on whether `librefang start` is running.

## Adding or Modifying Sections

When adding a new configuration section to the daemon:

1. **Add it to `init_default_config.toml`** — commented out, with inline documentation covering purpose, valid values, and security implications.
2. **Optionally wire it into the wizard** — if the wizard should prompt for it, add the corresponding placeholder to `init_wizard_config.toml` and update the wizard rendering logic.
3. **Keep comments factual** — these files are the first thing users read. Note deprecations (e.g., `require_proxy_headers` supersedes `trust_proxy_headers`).
4. **Preserve the section ordering convention** — server → auth → LLM → memory → tools → integrations → advanced.