# Other — librefang-cli-templates

# librefang-cli-templates

Configuration templates used by the `librefang-cli init` command to bootstrap a new LibreFang Agent OS installation.

## Overview

This module contains two TOML template files that serve as the source for generated configuration files. They are static resources — no executable code, no imports, no runtime logic. The CLI reads these templates during `librefang init` and substitutes Mustache-style placeholders with user-provided or auto-detected values before writing the final `config.toml` to disk.

## Files

| File | Purpose |
|---|---|
| `init_default_config.toml` | Full reference config with every section and inline documentation |
| `init_wizard_config.toml` | Minimal config produced by the interactive setup wizard |

## Template Variables

Both files use `{{variable}}` syntax that the CLI replaces at generation time:

| Placeholder | Used In | Description |
|---|---|---|
| `{{provider}}` | Both | LLM provider name (e.g. `openai`, `anthropic`, `ollama`) |
| `{{model}}` | Both | Default model identifier (e.g. `gpt-4o`, `claude-sonnet-4-20250514`) |
| `{{api_key_env}}` | `init_default_config.toml` | Environment variable name holding the API key |
| `{{api_key_line}}` | `init_wizard_config.toml` | Pre-formatted `api_key_env = "..."` line (may be empty for providers like Ollama) |
| `{{routing_section}}` | `init_wizard_config.toml` | Optional agent routing block inserted when the wizard configures multiple agents |

## When Each Template Is Used

```mermaid
flowchart TD
    A["librefang init"] --> B{Interactive wizard?}
    B -- Yes --> C["init_wizard_config.toml"]
    B -- No / --full --> D["init_default_config.toml"]
    C --> E["Replace placeholders"]
    D --> E
    E --> F["Write config.toml"]
```

- **`init_wizard_config.toml`** — Selected when the user runs through the interactive setup wizard. Produces a lean config with only the sections the user explicitly configured (server, default model, memory, optional routing).
- **`init_default_config.toml`** — Selected with `librefang init --full` or equivalent non-interactive flag. Produces the complete reference config with every available section present and commented out, serving as both a working config and on-device documentation.

## Configuration Sections

The full template covers these top-level sections. Sections marked with `*` are active by default; all others are commented out and must be manually enabled.

### Core Server

| Key | Default | Notes |
|---|---|---|
| `api_listen` | `127.0.0.1:4545` | Local-only. Use `0.0.0.0:4545` for LAN/remote access |
| `log_level` | `info` | trace, debug, info, warn, error |
| `mode` | `default` | stable, default, dev |
| `update_channel` | `stable` | stable, beta, rc |

### Authentication

`dashboard_user` / `dashboard_pass` — Default credentials for the web dashboard. The template includes inline guidance for upgrading to Vault-backed or environment variable storage:

```toml
# Vault:   dashboard_pass = "vault:dashboard_password"
# Env var: LIBREFANG_DASHBOARD_PASS=your-secret
```

### Terminal Access Control (commented out)

Three keys controlling remote shell access: `allow_remote`, `allow_unauthenticated_remote`, and `require_proxy_headers`. All default to safe values (disabled). The `allow_unauthenticated_remote` flag is explicitly called out as a foot-gun guard.

### Performance

`prompt_caching` and `stable_prefix_mode` reduce LLM costs by improving cache hit rates. `usage_footer` controls what token/cost information is displayed.

### Default Model (`[default_model]`*)

The primary LLM configuration. Template variables fill in `provider`, `model`, and `api_key_env`. An optional `base_url` override is available for local proxies.

### Memory (`[memory]`*)

`decay_rate` controls how quickly memory confidence degrades over cycles. `embedding_model` defaults to `all-MiniLM-L6-v2` when uncommented.

### Proactive Memory (`[proactive_memory]`*)

Enables automatic fact extraction from conversations and context-aware retrieval. Key tuning parameters:

- `max_retrieve` — Maximum memories pulled per retrieval cycle (default 10)
- `extraction_threshold` — Minimum confidence to persist a memory (default 0.7)
- `duplicate_threshold` — Similarity cutoff for deduplication (default 0.5)

### Web Tools (`[web]`*)

Search provider auto-detection cascades through Tavily → Brave → Jina → Perplexity → DuckDuckGo based on available API keys. The `[web.fetch]` subsection controls page extraction limits and SSRF protection. Cloud metadata IPs (169.254.x.x, 100.64.x.x) are always blocked regardless of `ssrf_allowed_hosts`.

### Task Queue (`[queue.concurrency]`*)

Four concurrency lanes with independent limits:

| Lane | Default | Purpose |
|---|---|---|
| `main_lane` | 3 | User messages |
| `cron_lane` | 2 | Scheduled jobs |
| `subagent_lane` | 3 | Child agents |
| `trigger_lane` | 8 | In-flight trigger dispatches |
| `default_per_agent` | 1 | Per-agent fallback (legacy serial) |

### Shell Execution (`[exec_policy]`*)

Security-critical section. Default `mode = "deny"` blocks all shell commands. Promote to `allowlist` or `full` as needed.

### Hot-Reload (`[reload]`*)

`mode = "hybrid"` combines hot-reload for compatible settings with restart for structural changes. `debounce_ms` prevents rapid successive reloads during file saves.

### Optional Sections (all commented out)

| Section | Purpose |
|---|---|
| `[provider_regions]` | Multi-region endpoint selection (Qwen intl, MiniMax China) |
| `[provider_urls]` | Override API endpoints (Ollama, vLLM local servers) |
| `[[fallback_providers]]` | LLM failover chain |
| `[rate_limit]` | GCRA-based API rate limiting, WebSocket throttling |
| `[registry]` | Agent registry cache TTL |
| `[compaction]` | LLM-based session history summarization |
| `[triggers]` | Event trigger cooldown, recursion depth, workflow timeout |
| `[budget]` | Hourly/daily/monthly cost caps with per-provider overrides |
| `[thinking]` | Extended thinking budget for Claude models |
| `[channels.*]` | Telegram, Discord, Slack, WeChat bot integrations |
| `[[mcp_servers]]` | External MCP tool servers (stdio transport) |
| `[browser]` | Headless browser automation settings |
| `[docker]` | Docker sandbox for code execution |
| `[inbox]` | File-based async message ingestion |
| `[network]` | P2P federation with shared secret auth |

## Relationship to the Rest of the Codebase

These templates are consumed exclusively by the CLI initialization flow:

1. **`librefang-cli`** reads the appropriate template file at `librefang init` time, performs placeholder substitution, and writes the result to the user's config directory.
2. The generated `config.toml` is then read by the **LibreFang Agent OS server** at startup. Every section in these templates maps to a corresponding config struct in the server's configuration parser.
3. The inline comments in `init_default_config.toml` serve as the primary on-disk reference documentation — they should be kept in sync with the actual server behavior when config keys change.

## Contributing

When adding a new configuration section to the server:

1. Add the section to `init_default_config.toml` as a commented-out block with descriptive comments following the established style (`# ── Section Name ───────────`).
2. If the section is commonly configured during initial setup, consider adding it to `init_wizard_config.toml` with appropriate template variables.
3. Keep comments factual and concise — they are the first thing users read when editing their config.