# Extensions & MCP

# Extensions & MCP

The Extensions & MCP module group provides everything needed to discover, install, authenticate, and run Model Context Protocol (MCP) integrations. It splits cleanly into two cooperating layers: **lifecycle management** and **client runtime**.

## Sub-Modules

| Sub-module | Role |
|---|---|
| [librefang-extensions](librefang-extensions-src.md) | Catalog browsing, credential vault, OAuth PKCE flows, health monitoring, and installation into `config.toml` |
| [librefang-runtime-mcp](librefang-runtime-mcp-src.md) | MCP client connections, tool discovery, namespaced tool execution, transport handling, and argument taint scanning |

## How They Fit Together

```mermaid
flow LR
    subgraph Management [librefang-extensions]
        CAT[McpCatalog] --> INST[install_integration]
        INST --> CV[CredentialVault]
        OA[OAuth PKCE] --> CV
        CV --> HM[HealthMonitor]
    end

    subgraph Runtime [librefang-runtime-mcp]
        CONN[McpConnection] --> DISC[Tool Discovery]
        DISC --> EXEC[Namespaced call_tool]
        EXEC --> TAINT[Taint Scanner]
    end

    CV -. "provides credentials" .-> CONN
    OA -. "provides tokens" .-> CONN
    HM -. "watches liveness" .-> CONN
    INST -. "writes config.toml" .-> CONN
```

**librefang-extensions** handles the setup phase: operators browse the `McpCatalog`, choose integrations, and `install_integration` scaffolds config entries while secrets are stored in the AES-256-GCM `CredentialVault`. OAuth PKCE flows obtain tokens for servers that require them.

**librefang-runtime-mcp** consumes that config at runtime. `McpConnection` opens stdio/HTTP/SSE transports, discovers available tools, and exposes them through a namespaced interface that prevents tool-name collisions. Every outbound tool call passes through `scan_mcp_arguments_for_taint` to block credential exfiltration.

## Key Cross-Module Workflows

- **Installation → Connection**: `install_integration` writes integration config (including any vault-stored credentials). On startup, the runtime reads that config, unlocks credentials via `CredentialVault`, and establishes `McpConnection` sessions.
- **OAuth token acquisition**: The PKCE flow in librefang-extensions drives browser-based auth; the resulting tokens are consumed by the runtime's connection layer when authenticating to protected MCP servers.
- **Credential resolution for system operations**: Flows like TOTP setup and terminal management (delete/rename window) route through `has_dashboard_credentials` → `resolve_dashboard_credential` → `CredentialVault::unlock`, showing how the vault serves as a central secrets provider beyond just MCP tooling.