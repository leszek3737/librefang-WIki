# MCP Integration

# MCP Integration

LibreFang's MCP (Model Context Protocol) integration lets the system act as both an **MCP client** — connecting to external tool servers — and an **MCP server** — exposing its own tools to external clients like Claude Desktop or VS Code extensions. The three sub-modules divide this cleanly by role.

## Sub-modules

| Sub-module | Role |
|---|---|
| [librefang-runtime-mcp](librefang-runtime-mcp-src.md) | MCP **client** — connects to external MCP servers, discovers tools, and executes `mcp_*` tool calls on behalf of the LLM runtime. Also contains protocol-level OAuth logic (PKCE, metadata discovery). |
| [librefang-runtime (mcp_server)](librefang-runtime-src.md) | MCP **server** — transport-agnostic JSON-RPC handler that exposes LibreFang's tools to external MCP clients over stdio or HTTP. Also includes `mcp_migrate`, a one-time config migration utility. |
| [librefang-kernel (mcp_oauth_provider)](librefang-kernel-src.md) | OAuth **credential persistence** — stores and refreshes OAuth tokens in the encrypted `CredentialVault` (`~/.librefang/vault.enc`). Stateless; every call unlocks the vault fresh. |

## How they fit together

```
┌─────────────────────────────────────────────────────────────┐
│  External MCP Client (Claude Desktop, VS Code, …)          │
│         │  JSON-RPC (stdio / HTTP)                         │
│         ▼                                                   │
│  ┌──────────────────────────┐                               │
│  │  mcp_server              │  ← librefang-runtime         │
│  │  (tools/list, tools/call)│                               │
│  └──────────────────────────┘                               │
│                                                             │
│  ┌──────────────────┐    tool_runner   ┌─────────────────┐ │
│  │  LLM Runtime     │──dispatch──────▶│ librefang-      │ │
│  │  execute_tool_raw│  mcp_* calls    │ runtime-mcp     │ │
│  └──────────────────┘                 │ (MCP client)    │ │
│                                       └───────┬─────────┘ │
│                                               │             │
│                             OAuth needed?     │             │
│                                   ▼           │             │
│                       ┌───────────────────┐   │             │
│                       │ mcp_oauth_provider│   │             │
│                       │ (vault CRUD)      │   │             │
│                       └───────────────────┘   │             │
│                                       librefang-kernel      │
└─────────────────────────────────────────────────────────────┘
```

## Key cross-module workflows

### Tool execution (client side)

The LLM runtime's `execute_tool_raw` dispatches any tool call prefixed with `mcp_` to **librefang-runtime-mcp**. That crate resolves the server name from the namespaced tool name (`mcp_{server}_{tool}`), connects to the external MCP server over the configured transport (stdio, HTTP, SSE), and returns the result.

### OAuth authentication

When a remote MCP server requires auth, the flow spans all three crates:

1. **API layer** (`src/routes/mcp_auth.rs`) kicks off the browser-facing OAuth redirect.
2. **librefang-runtime-mcp** handles protocol details — PKCE generation, RFC 8414 metadata discovery via `discover_oauth_metadata`, and `WWW-Authenticate` header parsing.
3. **librefang-kernel**'s `McpOAuthProvider` persists tokens and handles refresh, reading/writing the encrypted vault on every call (stateless pattern).

### Serving external clients

**librefang-runtime**'s `mcp_server` module is the inbound side. It receives JSON-RPC requests from external clients, dispatches `tools/list` against LibreFang's registered tools, and routes `tools/call` to the appropriate handler. This is transport-agnostic — the same `handle_mcp_request` function works regardless of whether the client connected over stdio or HTTP.

### Config migration

On first startup after upgrade, **mcp_migrate** (inside librefang-runtime) detects the legacy two-file MCP layout and converts it into the unified `config.toml` format, preserving any existing manual entries.