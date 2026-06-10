# Other — librefang-cli-templates

# librefang-cli-templates

Configuration templates used by the LibreFang CLI (`librefang init`) to generate an initial `librefang.toml` file. This module contains no executable code — it ships two TOML template files that the CLI reads, interpolates, and writes to disk.

## Files

| File | Purpose |
|---|---|
| `init_default_config.toml` | Full reference configuration with every section commented out and documented. Used by `librefang init` in non-interactive mode. |
| `init_wizard_config.toml` | Minimal configuration emitted by the interactive setup wizard (`librefang init --wizard`). Only includes sections the user explicitly configured. |

## Template Placeholders

Both files contain Mustache-style `{{placeholder}}` tokens that the CLI replaces at generation time.

### `init_default_config.toml`

Used in `[default_model]`:

| Placeholder | Example value | Description |
|---|---|---|
| `{{provider}}` | `openai`, `anthropic` | LLM provider identifier |
| `{{model}}` | `gpt-4o`, `claude-sonnet-4-20250514` | Default model name |
| `{{api_key_env}}` | `OPENAI_API_KEY` | Environment variable holding the API key |

### `init_wizard_config.toml`

| Placeholder | Description |
|---|---|
| `{{provider}}` | LLM provider identifier |
| `{{model}}` | Default model name |
| `{{api_key_line}}` | Complete `api_key_env = "..."` line, or empty if the user chose not to configure a key |
| `{{routing_section}}` | Optional agent routing block generated from wizard responses, or empty string |

The wizard template uses coarser placeholders (`{{api_key_line}}`, `{{routing_section}}`) because the wizard may produce multi-line or optional blocks that don't map cleanly to a single value.

## Configuration Sections Reference

The full template covers every configurable subsystem. Active defaults vs. opt-in sections are noted below.

### Always active (written uncommented)

- **Server** — `api_listen`, `log_level`, `mode`, `update_channel`. Default bind is loopback-only (`127.0.0.1:4545`). Binding to a non-loopback address requires authentication to be configured; the daemon will refuse to start otherwise unless `LIBREFANG_ALLOW_NO_AUTH=1` is set.
- **Dashboard Login** — `dashboard_user` / `dashboard_pass`. Defaults to `librefang`/`librefang`; must be changed after first login. Supports vault references (`"vault:dashboard_password"`) and environment variable injection (`LIBREFANG_DASHBOARD_PASS`).
- **Performance** — `prompt_caching`, `stable_prefix_mode`, `usage_footer`.
- **Default Model** — `[default_model]` with provider, model name, and API key env var.
- **Memory** — `[memory]` with `decay_rate`.
- **Proactive Memory** — `[proactive_memory]` with `auto_memorize`, `auto_retrieve`, `max_retrieve` enabled by default.
- **Web Tools** — `[web]` with `search_provider = "auto"` (auto-detects among Tavily, Brave, Jina, Perplexity, DuckDuckGo).
- **Task Queue** — `[queue.concurrency]` with lane limits for main, cron, subagent, and trigger dispatches.
- **Shell Execution Policy** — `[exec_policy]` defaulting to `mode = "deny"`.
- **Config Hot-Reload** — `[reload]` with `mode = "hybrid"` and 500ms debounce.

### Opt-in (shipped commented out)

Each section below is present in the template but prefixed with `#`. Users uncomment to activate.

| Section | Key | Purpose |
|---|---|---|
| Terminal Access Control | `[terminal]` | Remote shell access, proxy header trust |
| Provider Regions | `[provider_regions]` | Region-specific endpoints (e.g., Qwen intl, MiniMax China) |
| Provider URL Overrides | `[provider_urls]` | Custom endpoints for self-hosted providers (Ollama, vLLM) |
| Fallback Providers | `[[fallback_providers]]` | LLM failover chain |
| Rate Limiting | `[rate_limit]` | GCRA-based API rate limits, WebSocket throttling |
| Registry Sync | `[registry]` | Agent registry cache TTL |
| Session Compaction | `[compaction]` | LLM-based history summarization thresholds |
| Event Triggers | `[triggers]` | Cooldown, recursion depth, workflow timeout |
| Budget & Cost Control | `[budget]` | Hourly/daily/monthly caps, per-provider limits |
| Extended Thinking | `[thinking]` | Claude extended thinking budget and streaming |
| Browser Automation | `[browser]` | Headless browser sessions |
| Docker Sandbox | `[docker]` | Containerized code execution |
| File Inbox | `[inbox]` | Directory-based async message ingestion |
| Network (P2P) | `[network]` | Federation shared secret |
| MCP Servers | `[[mcp_servers]]` | External tool integration via Model Context Protocol |

## Sidecar Channel Adapters

All messaging channel integrations run as out-of-process Python sidecars communicating over newline-delimited JSON-RPC via stdio. The template ships commented-out `[[sidecar_channels]]` blocks for each supported adapter.

### Prerequisites

Sidecars require the LibreFang Python SDK:

```bash
pip install librefang-sdk
# Or from source:
pip install -e sdk/python/
```

Verify the SDK is importable from the same `python3` the daemon will use:

```bash
python3 -c 'import librefang.sidecar; print(librefang.__file__)'
```

### Adapter inventory

Each adapter block follows the same structure:

```toml
[[sidecar_channels]]
name = "<adapter>"
command = "python3"
args = ["-m", "librefang.sidecar.adapters.<adapter>"]
channel_type = "<adapter>"
[sidecar_channels.env]
# Adapter-specific env vars — secrets come from ~/.librefang/secrets.env, not here
```

Available adapters (each annotated with its migration PR in the template):

Bluesky, DingTalk, Discord, Email (IMAP+SMTP), Feishu/Lark, Google Chat, Gotify, LINE, Mastodon, Matrix, Mattermost, Nextcloud Talk, ntfy, QQ Bot, Reddit, Rocket.Chat, Signal, Slack, Microsoft Teams, Telegram, Twitch, Webex, Webhook (generic HMAC), WeChat, WeCom, WhatsApp (Cloud API or Baileys), Zulip.

### Adapter schema discovery

Each adapter publishes a SCHEMA describing its environment variables:

```bash
python3 -m librefang.sidecar.adapters.<name> --describe
```

### Regenerating adapter listings

LibreFang maintainers can regenerate the adapter blocks from source:

```bash
cd sdk/python && python3 ../../scripts/gen_sidecar_samples.py
```

## Security Notes

The template embeds several safety mechanisms:

1. **Loopback-only default** — `api_listen = "127.0.0.1:4545"`. The daemon refuses non-loopback binds without authentication configured.
2. **Shell deny-by-default** — `[exec_policy] mode = "deny"`.
3. **SSRF protection** — Cloud metadata addresses (`169.254.x.x`, `100.64.x.x`) are always blocked in web fetch; internal hosts require explicit `ssrf_allowed_hosts` entries.
4. **Terminal guard** — `allow_unauthenticated_remote` is a separate hard guard that must be explicitly enabled even when `allow_remote = true`.
5. **Secrets not in config** — Sidecar adapter secrets belong in `~/.librefang/secrets.env`, never inline in `librefang.toml`.

## Integration with the CLI

The CLI's init command loads these templates at compile time (or runtime from the crate's template directory) and performs string substitution on the `{{...}}` tokens. The flow is:

1. **Non-interactive** (`librefang init`): Reads `init_default_config.toml`, substitutes provider/model/key from CLI flags or defaults, writes the result.
2. **Interactive** (`librefang init --wizard`): Guides the user through provider selection, model choice, and optional agent routing, then renders `init_wizard_config.toml` with the collected answers.

The generated file is written to the project's working directory (or a path specified via `--config`). The daemon (`librefang start`) then reads this file at startup, applying `[reload]` settings for hot-reload behavior.