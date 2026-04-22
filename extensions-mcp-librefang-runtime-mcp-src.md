# Extensions & MCP — librefang-runtime-mcp-src

# librefang-runtime-mcp — MCP Client Runtime

This crate implements the Model Context Protocol (MCP) client layer for librefang. It manages connections to external MCP servers, discovers their tools, and executes tool calls on behalf of the LLM agent — all behind a namespaced interface that prevents tool-name collisions and blocks credential exfiltration.

## Architecture Overview

```mermaid
graph TD
    subgraph "McpConnection"
        connect["McpConnection::connect"]
        call["McpConnection::call_tool"]
        taint["scan_mcp_arguments_for_taint"]
    end

    subgraph "Transports"
        stdio["Stdio<br/>(rmcp SDK)"]
        http["Http<br/>(Streamable HTTP / rmcp)"]
        sse["Sse<br/>(JSON-RPC over HTTP POST)"]
        compat["HttpCompat<br/>(plain HTTP/JSON adapter)"]
    end

    subgraph "Security"
        sandbox["Subprocess sandbox<br/>(env whitelist)"]
        ssrf["SSRF checks"]
        oauth["OAuth discovery<br/>(mcp_oauth)"]
    end

    connect --> stdio
    connect --> http
    connect --> sse
    connect --> compat
    call --> taint
    http --> oauth
    http --> ssrf
    sse --> ssrf
    compat --> ssrf
    stdio --> sandbox
```

## Core Entry Points

The crate exposes two public operations on `McpConnection`:

- **`McpConnection::connect(config)`** — Opens a connection to an MCP server, performs the protocol handshake, discovers tools via `tools/list`, and returns a ready-to-use `McpConnection` handle.
- **`McpConnection::call_tool(name, arguments)`** — Invokes a namespaced tool on the connected server. Arguments are taint-scanned before transmission.

Callers outside this crate interact through:

- `execute_tool_raw` in `librefang-runtime::tool_runner` dispatches tool calls here when `is_mcp_tool(name)` returns `true`.
- `resolve_mcp_server_from_known` is called from `tool_runner`, the agent routes (`src/routes/agents.rs`), and the TUI event loop to map a namespaced tool name back to its owning server.

## Configuration

### `McpServerConfig`

```rust
pub struct McpServerConfig {
    pub name: String,           // Display name, used in tool namespacing
    pub transport: McpTransport, // Transport variant (see below)
    pub timeout_secs: u64,      // Default: 60
    pub env: Vec<String>,       // "KEY=VALUE" or "KEY" (legacy lookup)
    pub headers: Vec<String>,   // "Header-Name: value" for HTTP transports
    pub oauth_provider: Option<Arc<dyn McpOAuthProvider>>,
    pub oauth_config: Option<McpOAuthConfig>,
    pub taint_scanning: bool,   // Default: true
    pub roots: Vec<String>,     // Filesystem roots for MCP Roots capability
}
```

`env` entries follow `"KEY=VALUE"` format. The legacy `"KEY"` format (no `=`) looks up the value from the parent process environment. The subprocess does **not** inherit the full parent environment — only the allowlisted system variables in `SAFE_ENV_VARS` plus explicitly declared entries are passed through.

`roots` is populated at runtime by the kernel (home directory + agent workspaces), never from config files. These are advertised via the MCP Roots capability during the `initialize` handshake for local servers.

### `McpTransport`

Four transport variants, selected via `#[serde(tag = "type")]`:

| Variant | Protocol | Use Case |
|---|---|---|
| `Stdio` | MCP over stdin/stdout via rmcp SDK | Local subprocess servers (npx, python) |
| `Http` | Streamable HTTP (MCP 2025-03-26+) | Remote MCP servers with session management |
| `Sse` | JSON-RPC over HTTP POST | Legacy SSE-based servers |
| `HttpCompat` | Plain HTTP/JSON adapter | Non-MCP HTTP backends |

## Transport Implementations

### Stdio (`connect_stdio`)

Spawns a child process and communicates via the `rmcp` SDK's `TokioChildProcess` transport. This is the primary transport for local MCP servers.

**Security checks at spawn time:**

1. **Path traversal** — commands containing `..` are rejected.
2. **Shell interpreter blocklist** — `bash`, `sh`, `zsh`, `cmd`, `powershell`, etc. are rejected. MCP servers must use a specific runtime (`npx`, `node`, `python`).
3. **Environment sandboxing** — `cmd.env_clear()` removes the parent environment, then only `SAFE_ENV_VARS` system entries + declared `env` entries are passed through.
4. **Windows `.cmd` adaptation** — on Windows, `npx`/`npm` are `.cmd` wrappers; the code auto-detects and appends `.cmd` when appropriate.

`$VAR` and `${VAR}` references in args are expanded via `expand_env_vars` before spawning, so templates like `$HOME/.config/server` work without needing a shell wrapper.

If `roots` is non-empty, the client uses `RootsClientHandler` (which implements `rmcp::ClientHandler`) to declare the `roots` capability and respond to `roots/list` requests.

### Streamable HTTP (`connect_streamable_http`)

Uses `rmcp`'s `StreamableHttpClientTransport` for the full Streamable HTTP protocol: `Accept` header negotiation, `Mcp-Session-Id` tracking, and SSE stream parsing.

**OAuth handling:** When the server returns 401, the code extracts the `WWW-Authenticate` header via `extract_auth_header_from_error` (which performs a `downcast_ref` through rmcp's type-erased error chain) and attempts three-tier OAuth discovery. If auth is required but no cached token exists, it returns the sentinel error `"OAUTH_NEEDS_AUTH"` to signal the API layer to drive the PKCE flow via the UI.

**Roots scoping:** Filesystem roots are only advertised to local servers (`is_local_url` checks for `127.0.0.0/8`, `localhost`, `::1`). Remote servers like GitHub or Slack don't receive host paths.

### SSE (`connect_sse`)

Legacy JSON-RPC-over-HTTP-POST transport. The `sse_initialize` and `sse_discover_tools` methods perform the MCP handshake and tool discovery manually. SSE is unidirectional (client-initiated only), so the `roots` capability is never declared.

### HttpCompat (`connect_http_compat`)

A built-in adapter for plain HTTP/JSON backends that don't speak MCP. Tools are declared statically in config rather than discovered via `tools/list`.

**Path templating:** Tool paths like `/items/{id}` are rendered from arguments via `render_http_compat_path`, with values URL-percent-encoded by `encode_http_compat_path_value`. Used arguments are stripped from the remaining payload.

**Request modes:** `JsonBody` (POST JSON), `Query` (URL query parameters), or `None`.

**Response modes:** `Text` (raw string) or `Json` (pretty-printed JSON).

**Validation:** `validate_http_compat_config` rejects empty `base_url`, missing tools, headers without `value` or `value_env`, and tools with empty names or paths.

## Security

### Outbound Taint Scanning

Before every `call_tool` invocation (when `taint_scanning` is `true`), `scan_mcp_arguments_for_taint` walks every string leaf in the JSON argument tree:

1. **Content heuristic** — each string value is checked via `check_outbound_text_violation` with `TaintSink::mcp_tool_call`. This catches API keys, tokens, PII patterns.
2. **Key-name blocklist** — object keys matching `authorization`, `api_key`, `secret`, `password`, etc. (see `MCP_SENSITIVE_KEY_NAMES`) with non-empty string values are blocked unconditionally, even when `taint_scanning` is `false`.
3. **Recursion cap** — `MCP_TAINT_SCAN_MAX_DEPTH` (64) prevents pathological nesting from blowing the stack.

The returned violation string contains **only the JSON path** to the offending leaf, never the actual payload. This string flows back to the LLM as an error and is emitted to logs — echoing the secret would defeat the filter.

### SSRF Protection

`check_ssrf` blocks URLs targeting cloud metadata endpoints (`169.254.169.254`, `metadata.google.internal`). `is_local_url` uses proper `url::Url` parsing (not substring matching) to prevent bypasses like `127.0.0.1.evil.com` or `127.0.0.1@attacker.com`.

### Subprocess Environment

The `SAFE_ENV_VARS` allowlist covers essential system paths (`PATH`, `HOME`, `XDG_*`), Windows essentials, and runtime-specific variables (`NODE_PATH`, `PYTHONPATH`, `CARGO_HOME`, etc.). Everything else is stripped.

## Tool Namespacing

All MCP tools are namespaced as `mcp_{server}_{tool}` to prevent collisions across servers:

- `format_mcp_tool_name("github", "create_issue")` → `"mcp_github_create_issue"`
- Server names are normalized via `normalize_name`: lowercase, hyphens → underscores
- `is_mcp_tool(name)` checks the `mcp_` prefix for routing

**Server resolution:** Because normalized server names may contain underscores (e.g., `"my-server"` → `"mcp_my_server_tool"`), simple string splitting is ambiguous. Use `resolve_mcp_server_from_known(tool_name, server_names)` which finds the longest matching prefix among known server names. `extract_mcp_server` is a simpler heuristic for single-word server names only.

The `original_names` map inside `McpConnection` stores the mapping from namespaced name back to the raw tool name the server expects, so `call_tool("mcp_github_create_issue", ...)` correctly sends `"create_issue"` to the server.

## OAuth Authentication (`mcp_oauth` module)

### Three-Tier Metadata Discovery

`discover_oauth_metadata` resolves OAuth endpoints for a server using three fallback tiers:

1. **Tier 1 — WWW-Authenticate header**: Parse the `resource_metadata` URL from the 401 response, validate it (HTTPS, same-origin, non-blocked host), fetch and parse the RFC 8414 metadata.
2. **Tier 2 — `.well-known`**: Construct `{origin}/.well-known/oauth-authorization-server` and fetch metadata.
3. **Tier 3 — Config fallback**: Use `auth_url` and `token_url` from `McpOAuthConfig` if both are provided.

Config values always override discovered values via `merge_metadata_with_config`.

### PKCE Support

`generate_pkce()` produces a `(verifier, challenge)` pair using 32 random bytes base64url-encoded for the verifier and SHA-256 of the verifier base64url-encoded for the challenge. `generate_state()` produces a 16-byte random state parameter.

### `McpOAuthProvider` Trait

```rust
#[async_trait]
pub trait McpOAuthProvider: Send + Sync {
    async fn load_token(&self, server_url: &str) -> Option<String>;
    async fn store_tokens(&self, server_url: &str, tokens: OAuthTokens) -> Result<(), String>;
    async fn clear_tokens(&self, server_url: &str) -> Result<(), String>;
}
```

Implementors handle token persistence (e.g., encrypted vault on disk). The actual OAuth browser flow is driven by the API layer (`src/routes/mcp_auth.rs`), not by the provider.

### `McpAuthState`

Tracks the per-connection authentication lifecycle:

| State | Meaning |
|---|---|
| `NotRequired` | No auth needed or already authenticated with cached token |
| `NeedsAuth` | Server returned 401, awaiting user action |
| `PendingAuth` | OAuth flow in progress, `auth_url` available |
| `Authorized` | Valid token on hand |
| `Expired` | Token has expired |
| `Error` | Auth failed with a message |

### SSRF Hardening for Metadata URLs

`extract_metadata_url` applies three validation layers:

1. **HTTPS only** — `http://` metadata URLs are rejected per RFC 8414.
2. **Same-origin** — metadata URL must share scheme + host + port with `server_url`.
3. **No private/loopback IPs** — `is_ssrf_blocked_host` blocks `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16`, IPv6 loopback, unique-local, and link-local ranges.

## Integration Points

**Incoming calls** (this crate is called by):

- `execute_tool_raw` in `librefang-runtime/src/tool_runner.rs` — calls `is_mcp_tool`, `resolve_mcp_server_from_known`, and `McpConnection::call_tool`
- `get_agent_mcp_servers` in `src/routes/agents.rs` — calls `resolve_mcp_server_from_known`
- `spawn_fetch_agent_mcp_servers` in `src/tui/event.rs` — calls `resolve_mcp_server_from_known`
- `auth_start` in `src/routes/mcp_auth.rs` — calls `discover_oauth_metadata`
- `load_token` in `librefang-kernel/src/mcp_oauth_provider.rs` — calls `store_tokens` on the `McpOAuthProvider` trait

**Outgoing calls** (this crate depends on):

- `librefang_http::proxied_client_builder` — builds HTTP clients with proxy support for SSE and HttpCompat transports
- `librefang_types::taint::check_outbound_text_violation` — the content heuristic used by the taint scanner
- `librefang_types::tool::ToolDefinition` — tool metadata returned to the tool runner
- `librefang_types::config` — configuration types (`HttpCompatToolConfig`, `McpOAuthConfig`, etc.)