# Extensions & Hands

# Extensions & Hands

This module group provides LibreFang's **external integration layer** — the infrastructure for connecting to outside services, managing credentials, and running autonomous agents.

## Sub-modules

| Crate | Role |
|---|---|
| [LibreFang Extensions](librefang-extensions-src.md) | Infrastructure: environment loading, encrypted vault, MCP server catalog, credential resolution, health monitoring |
| [LibreFang Hands](librefang-hands-src.md) | Autonomous capability packages: hand definitions, marketplace registry, requirement checking, instance lifecycle and state persistence |
| [LibreFang Runtime MCP](librefang-runtime-mcp-src.md) | MCP client: connects to external MCP servers, discovers tools, executes calls with taint-scanning and sandboxed subprocesses |

## How they fit together

```
┌─────────────────────────────────────────────────────────┐
│                    CLI / Daemon startup                  │
│  load_dotenv → load_vault → unlock → resolve_master_key │
│                  (librefang-extensions)                  │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────┼────────────────┐
          ▼            ▼                ▼
   CredentialResolver  McpCatalog   HealthMonitor
   (extensions)        (extensions)  (extensions)
          │            │
          ▼            ▼
   ┌──────────────────────────┐     ┌─────────────────────────┐
   │     HandRegistry         │     │    McpConnection         │
   │  (librefang-hands)       │     │  (librefang-runtime-mcp) │
   │  • install / activate    │────▶│  • tool discovery        │
   │  • check_requirements    │     │  • namespaced execution  │
   │  • persist state         │     │  • taint scanning        │
   └──────────────────────────┘     └─────────────────────────┘
```

Extensions sits at the bottom and owns everything the system needs *before* it can interact with the outside world: decrypting credentials from `vault.enc`, loading `.env` / `secrets.env`, and maintaining the MCP server catalog at `~/.librefang/mcp/catalog/*.toml`.

Hands consume that infrastructure. When a hand is activated from the marketplace, the `HandRegistry` checks its requirements (binaries on `PATH`, env vars, API keys) using credentials resolved by extensions, then spawns autonomous agent instances that persist across daemon restarts.

Runtime MCP is the execution bridge. When a hand (or any agent) needs to call an external tool, the MCP client connects to the relevant server via Stdio or SSE, discovers available tools under the `mcp_{server}_{tool}` namespace, and executes calls within a security envelope.

## Key cross-module workflows

**Startup bootstrap** — The CLI calls `load_dotenv` → `load_vault` → `unlock` → `resolve_master_key` in extensions to decrypt credentials before anything else runs. See [Extensions — Architecture Overview](librefang-extensions-src.md#architecture-overview).

**Hand activation** — A route calls `get_hand` → `readiness` → `check_requirements`, which walks each requirement (e.g., `check_python3_available` → `run_returns_python3`) to confirm the environment supports the hand. See [Hands — Registry](librefang-hands-src.md).

**MCP tool execution** — `tool_runner::execute_tool_raw` resolves the target server via namespace helpers (`is_mcp_tool`, `resolve_mcp_server_from_known`), then delegates to `McpConnection` which handles Stdio/SSE transport and taint-scans outbound arguments. See [Runtime MCP — Architecture](librefang-runtime-mcp-src.md).