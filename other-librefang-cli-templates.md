# Other — librefang-cli-templates

# librefang-cli-templates

TOML configuration templates used by the LibreFang CLI during project initialization. These files are **static resources** — not executed directly — and are rendered by the CLI's `init` and `init-wizard` commands to produce a working `librefang.toml`.

## Templates

### `init_default_config.toml`

The full reference configuration. It contains every supported top-level key, section, and sidecar channel adapter, heavily commented with:

- Default values for required fields
- Valid option ranges (e.g. `mode = "stable" | "default" | "dev"`)
- Inline security guidance (loopback binding, auth requirements, SSRF protection)
- Migration notes for channel adapters moved to the sidecar architecture

The CLI copies this file verbatim when a user runs a bare `librefang init` without the interactive wizard. Users are expected to uncomment and edit sections as needed.

### `init_wizard_config.toml`

A minimal skeleton produced by the interactive `librefang init-wizard` flow. Only the sections the user actually configured are emitted. This keeps the resulting file short and legible.

## Template Placeholders

Both files contain Mustache-style `{{…}}` tokens substituted at generation time:

| Placeholder | Used in | Replaced with |
|---|---|---|
| `{{provider}}` | Both | LLM provider name (e.g. `openai`, `anthropic`, `ollama`) |
| `{{model}}` | Both | Default model identifier (e.g. `gpt-4o`, `claude-sonnet-4-20250514`) |
| `{{api_key_env}}` | `init_default_config.toml` | Environment variable name for the API key |
| `{{api_key_line}}` | `init_wizard_config.toml` | Fully rendered `api_key_env = "…"` line (or empty if using a local provider) |
| `{{routing_section}}` | `init_wizard_config.toml` | Agent routing block if the user defined custom agent-to-model mappings |

## Configuration Sections Reference

The full template organizes settings into the following groups:

### Server

```toml
api_listen = "127.0.0.1:4545"
log_level = "info"
mode = "default"
update_channel = "stable"
```

The daemon **refuses to start** if `api_listen` is set to a non-loopback address and no authentication is configured (`api_key`, `dashboard_user`/`dashboard_pass`, or `[[users]]`). Override with `LIBREFANG_ALLOW_NO_AUTH=1` at your own risk.

### Dashboard Authentication

```toml
dashboard_user = "librefang"
dashboard_pass = "librefang"
```

Default credentials must be changed after first login. Production deployments should use one of:
- **Vault**: `librefang vault set dashboard_password` then `dashboard_pass = "vault:dashboard_password"`
- **Environment variable**: `LIBREFANG_DASHBOARD_PASS`

### Default LLM

```toml
[default_model]
provider = "{{provider}}"
model = "{{model}}"
api_key_env = "{{api_key_env}}"
```

`base_url` can be overridden for local proxies or alternative endpoints.

### Memory and Proactive Memory

```toml
[memory]
decay_rate = 0.05

[proactive_memory]
enabled = true
auto_memorize = true
auto_retrieve = true
max_retrieve = 10
```

Controls fact extraction from conversations and automatic recall of relevant memories. Fine-tuning knobs include `extraction_threshold`, `session_ttl_hours`, `duplicate_threshold`, and `max_memories_per_agent`.

### Web Tools

```toml
[web]
search_provider = "auto"  # auto-detects: Tavily → Brave → Jina → Perplexity → DuckDuckGo

[web.fetch]
max_chars = 50000
timeout_secs = 30
readability = true
```

`ssrf_allowed_hosts` is available for self-hosted/K8s deployments that need internal network access. Cloud metadata addresses (`169.254.x.x`, `100.64.x.x`) are always blocked.

### Task Queue Concurrency

```toml
[queue.concurrency]
main_lane = 3
cron_lane = 2
subagent_lane = 3
trigger_lane = 8
default_per_agent = 1
```

Lanes control how many tasks of each type run in parallel. `default_per_agent` can be overridden per-agent via `max_concurrent_invocations` in agent manifests.

### Shell Execution Policy

```toml
[exec_policy]
mode = "deny"  # deny | allowlist | full
timeout_secs = 30
max_output_bytes = 102400
```

`deny` is the safest default — no shell commands execute. Move to `allowlist` or `full` only when explicitly needed.

### Hot-Reload

```toml
[reload]
mode = "hybrid"  # off | restart | hot | hybrid
debounce_ms = 500
```

`hybrid` applies hot-reload to safe settings and requires a restart for structural changes.

### Sidecar Channel Adapters

All messaging platform integrations run as out-of-process Python sidecars communicating via newline-delimited JSON-RPC over stdio. Each adapter follows the same pattern:

```toml
[[sidecar_channels]]
name = "slack"
command = "python3"
args = ["-m", "librefang.sidecar.adapters.slack"]
channel_type = "slack"
[sidecar_channels.env]
SLACK_APP_TOKEN = "xapp-..."
SLACK_BOT_TOKEN = "xoxb-..."
```

**Prerequisite**: `pip install librefang-sdk` into the same Python interpreter that `command = "python3"` resolves to. Verify with:

```bash
python3 -c 'import librefang.sidecar; print(librefang.__file__)'
```

Available adapters: Bluesky, DingTalk, Discord, Email (IMAP+SMTP), Feishu/Lark, Google Chat, Gotify, LINE, Mastodon, Matrix, Mattermost, Nextcloud Talk, ntfy, QQ Bot, Reddit, Rocket.Chat, Signal, Slack, Microsoft Teams, Telegram, Twitch, Webex, Webhook, WeChat, WeCom, WhatsApp, Zulip.

Secrets **must not** be committed in this file. Use `~/.librefang/secrets.env` or a vault.

To inspect an adapter's full environment variable inventory:

```bash
python3 -m librefang.sidecar.adapters.<name> --describe
```

### MCP Servers

```toml
[[mcp_servers]]
name = "filesystem"
timeout_secs = 30
[mcp_servers.transport]
type = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]
```

### Optional Sections (commented out by default)

| Section | Purpose |
|---|---|
| `[budget]` | Per-hour/day/month cost caps, per-provider throttling |
| `[thinking]` | Extended thinking budget for Claude models |
| `[browser]` | Headless browser automation settings |
| `[docker]` | Docker sandbox for isolated code execution |
| `[inbox]` | File-based message ingestion from a watched directory |
| `[network]` | P2P federation via shared secret |
| `[compaction]` | LLM-based history summarization thresholds |
| `[triggers]` | Event trigger cooldowns and recursion limits |
| `[rate_limit]` | GCRA token budget, WebSocket throttling |
| `[registry]` | Agent registry cache TTL |
| `[terminal]` | Remote terminal access and proxy header settings |
| `[[fallback_providers]]` | LLM failover chain |

## Integration with the CLI

These templates have no code or imports — they are pure resource files embedded in the `librefang-cli` crate (or installed alongside it). The initialization commands locate and render them:

```
librefang init          → copies init_default_config.toml → librefang.toml
librefang init-wizard   → processes init_wizard_config.toml → librefang.toml
```

During rendering, the CLI replaces `{{…}}` tokens with values gathered from the user or auto-detected from the environment. The resulting `librefang.toml` is consumed by the LibreFang daemon at startup.