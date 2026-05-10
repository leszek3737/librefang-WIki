# Other — librefang-runtime-mcp

# librefang-runtime-mcp

MCP (Model Context Protocol) client for the LibreFang runtime. This crate provides the integration layer that allows the LibreFang runtime to communicate with MCP-compatible tool servers, enabling dynamic tool discovery and invocation.

## Purpose

The Model Context Protocol standardizes how applications expose tools and resources to AI models and other consumers. This crate acts as the **client side** of that protocol within the LibreFang ecosystem — it connects to MCP servers, discovers available tools, and executes tool calls on behalf of the runtime.

## Architecture

```
┌─────────────────────┐
│  LibreFang Runtime   │
│                     │
│  (orchestration,    │
│   agent logic)      │
└────────┬────────────┘
         │ calls into
         ▼
┌─────────────────────┐       ┌─────────────────────┐
│ librefang-runtime-  │  MCP  │   External MCP       │
│       mcp           │◄─────►│   Tool Servers       │
│                     │       │                     │
│  client management, │       │  (file systems, DBs, │
│  tool discovery,    │       │   APIs, custom tools)│
│  invocation         │       └─────────────────────┘
└─────────────────────┘
```

## Key Dependencies

| Dependency | Role |
|---|---|
| `rmcp` | Rust MCP client library — handles the low-level MCP protocol mechanics (transport, message framing, handshake) |
| `librefang-types` | Shared type definitions used across LibreFang crates |
| `librefang-http` | HTTP client infrastructure, used for SSE-based MCP transports |
| `reqwest` | Underlying HTTP client for network communication |
| `arc-swap` | Lock-free atomic swapping of client state, enabling live reconnection or server rotation without blocking |
| `sha2`, `base64` | Cryptographic hashing and encoding, likely used for message integrity or authentication tokens |
| `rand` | Random number generation, likely for request IDs or nonce generation |
| `url` | URL parsing and construction for MCP server endpoints |

## Transport Layer

MCP supports multiple transport mechanisms. Based on the dependency profile, this crate supports:

- **HTTP/SSE (Server-Sent Events)** — the `reqwest` + `http` + `librefang-http` stack handles bidirectional communication over HTTP, with SSE for server-to-client streaming of tool results and notifications.
- **Stdio** — if `rmcp` provides stdio transport, local MCP server processes can be spawned and communicated with via stdin/stdout.

## Concurrency Model

The use of `arc-swap` indicates that MCP client instances are managed behind atomic references. This enables:

- **Hot-swapping** of client connections when a server disconnects or needs to be rotated.
- **Lock-free reads** — tool invocations from multiple concurrent tasks can access the active client without acquiring a mutex.
- **Live reconfiguration** — updating the set of connected MCP servers without stopping the runtime.

All operations are fully async, built on `tokio`.

## Error Handling

Errors are structured through `thiserror`, providing typed error variants for:

- Connection failures and transport errors
- Tool invocation failures (tool not found, execution error)
- Protocol-level errors (handshake failure, unsupported capability)
- Serialization/deserialization errors

Tracing is integrated via the `tracing` crate, emitting structured spans for connection lifecycle events, tool calls, and error conditions.

## Testing

The dev-dependency on `wiremock` enables mock HTTP servers for integration tests, allowing verification of MCP protocol interactions without requiring a real MCP server. Tests use `tokio` with the `macros` and `rt-multi-thread` features for async test functions.

## Relationship to Other Crates

This crate is consumed by the main LibreFang runtime (or an agent runtime module) when it needs to interact with external tools. It sits between the high-level agent orchestration logic and the network layer:

- **Downstream consumers**: The runtime calls into this crate to discover and invoke tools.
- **Upstream dependencies**: This crate relies on `librefang-types` for shared data structures and `librefang-http` for HTTP transport infrastructure, keeping the MCP-specific logic self-contained.