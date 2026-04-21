# API Server

# API Server (`librefang-api`)

The HTTP/WebSocket API server for the LibreFang Agent OS daemon. It boots the in-process kernel, wires up route handlers, middleware, and background tasks, then serves the JSON REST API, WebSocket chat endpoints, and the WebChat dashboard SPA.

## Architecture Overview

```mermaid
graph TD
    CLI[CLI / External Clients] -->|HTTP / WebSocket| Axum[Axum Router]
    Axum --> MW[Middleware Stack]
    MW --> V1Routes[v1 API Routes]
    MW --> Dashboard[Dashboard SPA]
    MW --> Webhook[Channel Webhooks]
    V1Routes --> Kernel[LibreFangKernel]
    Webhook --> Bridge[Channel Bridge]
    Bridge --> Kernel
    Kernel --> Agents[Agent Runtimes]
    subgraph Background Tasks
        CAT[Catalog Sync]
        CFG[Config Hot-Reload]
        GC[Cache GC]
        KEYS[Key Validation]
    end
    Kernel --- Background Tasks
```

## Entry Points

### `run_daemon()`

The primary entry point. Blocks until Ctrl+C, SIGTERM, or an API-initiated shutdown:

1. Parses the listen address into a `SocketAddr`
2. Wraps the `LibreFangKernel` in `Arc`
3. Starts background agents via `kernel.start_background_agents()`
4. Calls `build_router()` to construct the Axum app
5. Writes `daemon.json` (PID, listen address, version) for CLI discovery — rejects startup if another daemon is already alive at the same address
6. Spawns background tasks: dashboard sync, provider key validation, approval sweep, config hot-reload, model catalog sync, cache GC
7. Optionally starts the Docker-based observability stack (Prometheus + Grafana)
8. Binds with `SO_REUSEADDR` and runs `axum::serve` with graceful shutdown
9. On shutdown: aborts background tasks, stops channel bridges, cleans up tmux sessions, shuts down the kernel

### `build_router()`

Constructs the full `Router` without starting the server. Used by `run_daemon()` and by embedders (e.g., `librefang-desktop`) that need the router but manage their own lifecycle. Returns `(Router, Arc<AppState>)`.

## Route Structure

All API routes are defined once in `api_v1_routes()` and mounted at both `/api` and `/api/v1` for backward compatibility. Future versions get their own mount point.

```
/                          → WebChat SPA HTML
/dashboard/{*path}         → React SPA assets
/api/v1/*                  → Versioned API (stable)
/api/*                     → Unversioned alias (latest)
/v1/chat/completions       → OpenAI-compatible chat endpoint
/v1/models                 → OpenAI-compatible model list
/mcp                       → MCP HTTP endpoint
/hooks/wake                → Webhook: wake trigger
/hooks/agent               → Webhook: agent trigger
/channels/*                → Channel adapter webhooks (Feishu, Teams, etc.)
```

### Route Domains

Each domain lives in its own submodule under `routes::` and provides a `router()` function merged into the v1 tree:

| Domain | Module | Key Endpoints |
|---|---|---|
| System | `routes::system` | Health, version, status, shutdown |
| Agents | `routes::agents` | CRUD, messaging, sessions, file upload |
| Channels | `routes::channels` | Channel configuration |
| Config | `routes::config` | Get/set/reload config, schema |
| Memory | `routes::memory` | Agent memory management |
| Workflows | `routes::workflows` | Workflow CRUD and execution |
| Skills | `routes::skills` | Skill install, evolve, marketplace |
| Network | `routes::network` | A2A protocol, peers |
| Plugins | `routes::plugins` | Plugin management |
| Providers | `routes::providers` | LLM provider config |
| Budget | `routes::budget` | Token spend tracking |
| Terminal | `routes::terminal` | Terminal session management |
| Auth | (inline in `server.rs`) | Login, logout, change-password, OAuth |

## Middleware Stack

Middleware is applied in reverse order (last `.layer()` runs first). The effective execution order per request:

1. **CORS** (`CorsLayer`) — allows localhost + configured origins
2. **Tracing** (`TraceLayer`) — tower-http structured tracing
3. **Compression** (`CompressionLayer`) — gzip/deflate responses
4. **Request logging** — injects `x-request-id`, logs method/path/status/latency, records Prometheus metrics
5. **Security headers** — `X-Content-Type-Options`, `X-Frame-Options`, CSP, HSTS, `Cache-Control: no-store`
6. **API version headers** — adds `X-API-Version` to responses, negotiates via `Accept: application/vnd.librefang.v1+json`
7. **Rate limiting** (`gcra_rate_limit`) — GCRA-based per-IP rate limiting
8. **OIDC auth middleware** — external OAuth/OIDC token validation
9. **Auth** — bearer token / API key / session cookie validation (see below)
10. **Accept-Language** — parses `Accept-Language`, stores resolved language in request extensions for i18n error responses

## Authentication & Authorization

### Auth Methods (in precedence order)

1. **Loopback bypass** — requests from `127.0.0.1` / `::1` skip all auth (CLI on same machine)
2. **Bearer token** — `Authorization: Bearer <key>` validated against the configured `api_key`
3. **X-API-Key header** — fallback for clients that can't set `Authorization`
4. **Query parameter** — `?token=<key>` for SSE/EventSource clients
5. **Session cookie** — `librefang_session` cookie, only accepted on `/dashboard/*` paths (prevents CSRF on API endpoints)
6. **Per-user API keys** — Argon2id-hashed keys with role-based access control

### Token Resolution (`resolve_dashboard_credential`)

Credentials are resolved from three sources in priority order:

1. Environment variable (e.g., `LIBREFANG_DASHBOARD_USER`)
2. `vault:KEY_NAME` syntax — reads from the encrypted credential vault
3. Literal value from `config.toml`

### Role-Based Access Control

Per-user API keys carry a `UserRole` that gates endpoint access:

| Role | GET | POST | Owner-only writes |
|---|---|---|---|
| **Owner** | ✅ | ✅ | ✅ (config, shutdown, change-password) |
| **Admin** | ✅ | ✅ | ❌ |
| **User** | ✅ | Message/clone/approval only | ❌ |
| **Viewer** | ✅ | ❌ | ❌ |

The `is_owner_only_write()` function explicitly enumerates paths that require `Owner` role for non-GET methods: `/api/config`, `/api/config/set`, `/api/config/reload`, `/api/auth/change-password`, `/api/shutdown`.

### Public Endpoint Classification

Endpoints fall into three visibility tiers:

- **Always public** (no auth regardless of config): `/`, `/api/health`, `/api/versions`, auth flow endpoints, dashboard assets, static files
- **Dashboard reads** (public by default, locked down with `require_auth_for_reads`): `/api/agents`, `/api/status`, `/api/config`, `/api/budget`, `/api/skills`, `/api/workflows`, etc.
- **Always authenticated**: all POST/PUT/DELETE, `/api/health/detail`, `/api/shutdown`

The `require_auth_for_reads` config option controls the dashboard-reads tier. When unset (default), it auto-enables if any authentication is configured (`api_key`, user keys, or dashboard credentials). Explicitly set `false` to keep reads open behind an external auth proxy.

### Session Management

Sessions are randomly generated tokens stored in memory and persisted to `data/sessions.json`:

- Created by `POST /api/auth/dashboard-login` after password + optional TOTP verification
- Validated by the auth middleware with opportunistic expiry pruning
- Persisted across daemon restarts via `save_sessions()` / `load_sessions()`
- Invalidated globally on password change (`clear_sessions_file()`)
- Scoped to `Path=/dashboard` with `HttpOnly; SameSite=Lax` (plus `Secure` when the request is HTTPS)

## `AppState`

Shared application state passed to all handlers via Axum's state extractor:

```rust
pub struct AppState {
    pub kernel: Arc<LibreFangKernel>,
    pub started_at: Instant,
    pub peer_registry: Option<Arc<PeerRegistry>>,
    pub bridge_manager: Mutex<ChannelBridge>,
    pub channels_config: RwLock<ChannelsConfig>,
    pub shutdown_notify: Arc<Notify>,
    pub clawhub_cache: DashMap<...>,      // 120s TTL
    pub skillhub_cache: DashMap<...>,     // 120s TTL
    pub provider_probe_cache: ProbeCache,
    pub provider_test_cache: DashMap<...>,
    pub webhook_store: WebhookStore,
    pub active_sessions: RwLock<HashMap<String, SessionToken>>,
    pub api_key_lock: RwLock<String>,     // hot-reloadable via change_password
    pub media_drivers: MediaDriverCache,
    pub webhook_router: RwLock<Arc<Router>>,  // dynamic for hot-reload
    pub config_write_lock: Mutex<()>,     // serializes config writes
}
```

## Background Tasks

Spawned during `run_daemon()`, tracked for clean shutdown:

| Task | Interval | Purpose |
|---|---|---|
| Dashboard sync | Once at boot | Downloads/updates SPA assets from release |
| Provider key validation | Once at boot | Validates API keys so dashboard shows status |
| Approval sweep | Every 10s | Expires pending approval requests |
| Config hot-reload | Polls every 30s | Detects `config.toml` mtime changes, reloads kernel + channel bridges |
| Model catalog sync | Every 24h | Syncs community model catalog to local cache |
| Cache GC | Every 5m | Evicts expired clawhub/skillhub entries and session tokens |

## Daemon Discovery

`run_daemon()` writes a `daemon.json` file (default `~/.librefang/daemon.json`) containing:

```json
{
  "pid": 12345,
  "listen_addr": "127.0.0.1:4280",
  "started_at": "2024-01-15T10:30:00Z",
  "version": "0.8.0",
  "platform": "linux"
}
```

The CLI reads this via `read_daemon_info()` to find the running daemon. On startup, if the file exists, the server checks whether the recorded PID is still alive and the listen address is responding before refusing to start.

## API Versioning

The current version is `"v1"`, defined in `versioning::CURRENT_VERSION`. The `api_version_headers` middleware:

- Reads version from the path prefix (`/api/v1/...`) or `Accept: application/vnd.librefang.<version>+json`
- Returns `406 Not Acceptable` for unknown versions
- Adds `X-API-Version` to every response

To add a new version: create `api_v2_routes()`, nest it at `/api/v2`, and update `API_VERSION_LATEST`.

## Graceful Shutdown

Triggered by SIGINT, SIGTERM, Ctrl+C, or the API shutdown endpoint. The shutdown sequence:

1. Stop accepting new connections
2. Abort tracked background tasks and await their termination
3. Remove `daemon.json`
4. Stop channel bridges
5. Stop observability stack (Docker compose down)
6. Kill tmux sessions (if terminal tmux is enabled)
7. Shut down the kernel

## Feature Flags

- **`telemetry`** — enables `crate::telemetry` module with Prometheus metrics recorder and OpenTelemetry OTLP tracing initialization

## Security Considerations

- File permissions on `daemon.json` are restricted to `0600` (owner-only) on Unix
- Password hashing uses Argon2id with transparent migration from legacy plaintext
- All token comparisons use constant-time equality (`subtle::ConstantTimeEq`)
- Session cookies are scoped to `/dashboard` to prevent CSRF on API endpoints
- The `Secure` cookie flag is set automatically when the request arrives over HTTPS (detected via `X-Forwarded-Proto`)
- Dashboard username is never echoed in unauthenticated responses (prevents credential half-leak)
- Webhook URLs are validated against private IP ranges to prevent SSRF
- Request body size is limited by `max_request_body_bytes` (upload routes are exempt and enforce their own limit)