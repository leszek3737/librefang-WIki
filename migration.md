# Migration

# Migration Module (`librefang-migrate`)

The migration engine imports agents, memory, sessions, skills, and channel configurations from other agent frameworks into LibreFang's native format.

## Supported Sources

| Source | Status | Config Format |
|--------|--------|---------------|
| OpenClaw | Fully supported | JSON5 (`openclaw.json`) or legacy YAML (`config.yaml`) |
| OpenFang | Fully supported | Same TOML format — copy with schema validation |
| LangChain | Not yet supported | — |
| AutoGPT | Not yet supported | — |

## Public API

The entry point is `run_migration`, which dispatches to the appropriate sub-module based on `MigrateSource`:

```rust
use librefang_migrate::{MigrateSource, MigrateOptions, run_migration};

let report = run_migration(&MigrateOptions {
    source: MigrateSource::OpenClaw,
    source_dir: PathBuf::from("/home/user/.openclaw"),
    target_dir: PathBuf::from("/home/user/.librefang"),
    dry_run: false,
})?;
```

### `MigrateOptions`

| Field | Type | Purpose |
|-------|------|---------|
| `source` | `MigrateSource` | Which framework to import from |
| `source_dir` | `PathBuf` | Path to the source workspace |
| `target_dir` | `PathBuf` | Path to the LibreFang home directory |
| `dry_run` | `bool` | When `true`, report what would happen without writing files |

### `MigrateError`

All errors are consolidated into a single enum with these variants:

- `SourceNotFound(PathBuf)` — source directory does not exist
- `ConfigParse(String)` — config file could not be parsed
- `AgentParse(String)` — agent definition could not be parsed
- `Io(std::io::Error)` — filesystem errors
- `Yaml(serde_yaml::Error)` — YAML parsing errors (legacy format)
- `Json5Parse(String)` — JSON5 parsing errors (modern OpenClaw)
- `TomlSerialize(toml::ser::Error)` — TOML output serialization errors
- `UnsupportedSource(String)` — requested source is not yet implemented

### `MigrationReport`

Returned by `run_migration`, the report contains three lists:

- `imported: Vec<MigrateItem>` — items successfully (or would-be) migrated
- `skipped: Vec<SkippedItem>` — items that cannot be migrated, with reasons
- `warnings: Vec<String>` — non-fatal issues encountered

Each `MigrateItem` has a `kind` (Config, Agent, Channel, Secret, Memory, Session, Skill), a `name`, and a `destination` path. The report's `to_markdown()` method produces a human-readable summary.

## OpenClaw Migration

### Workspace Detection

`detect_openclaw_home()` checks these locations in order:

1. `OPENCLAW_STATE_DIR` environment variable
2. `~/.openclaw`, `~/.clawdbot`, `~/.moldbot`, `~/.moltbot`
3. `~/openclaw`, `~/.config/openclaw`
4. `%APPDATA%/openclaw` and `%LOCALAPPDATA%/openclaw` (Windows)

A directory is considered valid if it contains a recognized config file (`openclaw.json`, `clawdbot.json`, `moldbot.json`, `moltbot.json`, or `config.yaml`) or has `sessions/` or `memory/` subdirectories.

### Workspace Scanning

`scan_openclaw_workspace()` inspects a workspace without migrating and returns a `ScanResult` listing:

- All agents found (name, provider, model, tool count, whether they have memory/sessions/workspaces)
- All channels configured
- All skills installed
- Whether memory data exists

### Two Config Formats

OpenClaw has two generations of config layout. The migration engine auto-detects which one is in use:

**Modern (JSON5)** — Everything in a single `openclaw.json`:

```
~/.openclaw/
├── openclaw.json          # Global config, agents, channels, tools, cron, hooks
├── auth-profiles.json     # Credentials (not migrated)
├── sessions/              # JSONL conversation logs
├── memory/                # Per-agent MEMORY.md files
├── skills/
├── workspaces/
└── memory-search/         # SQLite vector index (not portable)
```

**Legacy (YAML)** — Scattered YAML files:

```
~/.openclaw/
├── config.yaml            # Global model and provider settings
├── agents/
│   └── <agent>/
│       ├── agent.yaml
│       ├── MEMORY.md
│       └── workspace/
├── messaging/
│   ├── telegram.yaml
│   ├── discord.yaml
│   └── ...
└── skills/
    ├── community/
    └── custom/
```

### Migration Pipeline

```
migrate(options)
  ├── find_config_file()       # Detect JSON5 vs YAML
  ├── migrate_from_json5()     # or migrate_from_legacy_yaml()
  │     ├── migrate_config_from_json()
  │     │     └── migrate_channels_from_json()
  │     ├── migrate_agents_from_json()
  │     │     └── convert_agent_from_json()   # per agent
  │     ├── migrate_memory_files()
  │     ├── migrate_workspace_dirs()
  │     ├── migrate_sessions()
  │     └── report_skipped_features()
  └── Write migration_report.md
```

### What Gets Migrated

| Item | Source Location | Target Location |
|------|----------------|-----------------|
| Global config | `openclaw.json` or `config.yaml` | `config.toml` |
| Agent definitions | `agents.list[]` or `agents/<id>/agent.yaml` | `agents/<id>/agent.toml` |
| Channel configs | `channels.{telegram,discord,...}` or `messaging/*.yaml` | `[channels.*]` in `config.toml` |
| Secrets (tokens) | Inline in JSON5 or env var refs | `secrets.env` |
| Memory files | `memory/<agent>/MEMORY.md` | `agents/<agent>/imported_memory.md` |
| Session logs | `sessions/*.jsonl` | `imported_sessions/*.jsonl` |
| Workspace dirs | `workspaces/<agent>/` | `agents/<agent>/workspace/` |

### What Is Not Migrated

These features are reported as skipped items with explanations:

- **Cron jobs** — LibreFang uses `ScheduleMode::Periodic` instead
- **Webhook hooks** — LibreFang uses its own event system
- **Auth profiles** — Security-sensitive; users must set environment variables manually
- **Memory vector index** (`memory-search/index.db`) — SQLite binary index is not portable; LibreFang rebuilds embeddings
- **Memory backend config** — LibreFang uses its own SQLite + vector store
- **Session scope config** — LibreFang uses per-agent sessions
- **Skills** — Must be reinstalled via `librefang skill install`
- **iMessage** — macOS-only, requires manual setup
- **BlueBubbles** — No LibreFang adapter exists

### Channel Support

Thirteen channel types are recognized. Tokens extracted from inline values are written to `secrets.env` with restricted permissions (0600 on Unix).

| Channel | Migrated | Notes |
|---------|----------|-------|
| Telegram | Yes | `bot_token` → `secrets.env` |
| Discord | Yes | `token` → `secrets.env` |
| Slack | Yes | `bot_token` + `app_token` → `secrets.env`; `allow_from` cannot map (Slack uses channel IDs) |
| WhatsApp | Yes | Baileys credential directory is copied; re-authentication may be needed |
| Signal | Yes | Constructs API URL from `http_host` + `http_port` |
| Matrix | Yes | `access_token` → `secrets.env`; `allow_from` cannot map |
| Google Chat | Yes | Service account JSON file is copied |
| Teams | Yes | `app_password` → `secrets.env`; `allow_from` cannot map (Teams uses tenant IDs) |
| IRC | Yes | Server, port, nick, channels, TLS all mapped |
| Mattermost | Yes | `bot_token` → `secrets.env` |
| Feishu | Yes | `domain` is mapped to `region` ("intl" for Lark, "cn" for Feishu) |
| iMessage | Skipped | macOS-only, manual setup required |
| BlueBubbles | Skipped | No adapter available |

### Policy Mapping

OpenClaw policies are translated to LibreFang equivalents:

**DM Policy:**

| OpenClaw | LibreFang |
|----------|-----------|
| `open` | `respond` |
| `allowlist` / `allow_list` | `allowed_only` |
| `pairing` / `disabled` | `ignore` |

**Group Policy:**

| OpenClaw | LibreFang |
|----------|-----------|
| `open` / `all` | `all` |
| `mention` / `mention_only` | `mention_only` |
| `commands` / `commands_only` / `slash_only` | `commands_only` |
| `disabled` / `ignore` | `ignore` |

### Model Reference Handling

OpenClaw uses `"provider/model"` strings (e.g. `"anthropic/claude-sonnet-4-20250514"`). The `split_model_ref()` function separates these, falling back to provider `"anthropic"` if no slash is present. The `map_provider()` function normalizes provider names (e.g., `"claude"` → `"anthropic"`, `"gemini"` → `"google"`).

Agent entries can also specify fallback models via `OpenClawAgentModel::Detailed`, which are emitted as `[[fallback_models]]` TOML arrays in the agent manifest.

### Tool Mapping and Capabilities

Tool names from OpenClaw are mapped to LibreFang equivalents via `librefang_types::tool_compat::{is_known_librefang_tool, map_tool_name}`. Unrecognized tools are reported as warnings and omitted.

Tool profiles (`minimal`, `coding`, `research`, `messaging`, `automation`, `full`) are resolved through `librefang_types::agent::ToolProfile::tools()`.

Capabilities are derived heuristically from the tool list:

- `shell_exec` → `capabilities.shell = ["*"]`
- `web_fetch`, `web_search`, `browser_navigate` → `capabilities.network = ["*"]`
- `agent_send`, `agent_list` → `capabilities.agent_message = ["*"]` and `agent_spawn = true`
- `"*"` (wildcard) → all of the above

### Agent Identity / System Prompt

The `extract_identity_prompt()` function handles multiple identity formats:

- **Plain string** — used directly
- **Structured object** — searches for keys in priority order: `systemPrompt`, `system_prompt`, `prompt`, `instructions`, `instruction`, `content`, `text`, `value`, `persona`, `identity`, `description`
- **Array** — parts are joined with double newlines
- **Nested objects** — recursed until a string value is found

If no identity is configured, a default prompt is generated using the agent's display name.

## OpenFang Migration

OpenFang uses the same TOML format as LibreFang (it is a community fork). The migration copies config, agents, and related files with validation and schema drift warnings via `librefang_types::config::validation`.

## Report Module (`report`)

`MigrationReport` is serializable and includes:

- `to_markdown()` — generates a markdown summary for display or file output
- `print_summary()` — prints a formatted summary to stdout (used by the CLI)

## Integration Points

The migration module is called from three places in the larger codebase:

- **CLI** (`librefang-cli/src/main.rs`) — `cmd_migrate` calls `run_migration()` and prints the summary
- **HTTP API** (`src/routes/config.rs`) — `run_migrate`, `migrate_detect`, and `migrate_scan` endpoints
- **TUI init wizard** (`tui/screens/init_wizard.rs`) — `run` detects OpenClaw, `handle_migration_key` executes the migration

## Security Considerations

- Tokens and secrets extracted from inline JSON5 values are written to `secrets.env`, not to `config.toml`. The config file references them via `*_env` field names (e.g., `bot_token_env = "TELEGRAM_BOT_TOKEN"`).
- `write_secret_env()` restricts file permissions to `0600` on Unix systems.
- Auth profiles and credential files are explicitly excluded from migration and reported as skipped with instructions to set environment variables manually.