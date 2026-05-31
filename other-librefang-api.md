# Other — librefang-api

# librefang-api

HTTP and WebSocket API server for the LibreFang Agent OS daemon. This crate is the primary network interface between the agent kernel and external clients (CLI, desktop, mobile, or any HTTP/WebSocket consumer). The kernel runs in-process; all clients connect over this surface.

## Architecture

```mermaid
graph TD
    subgraph "Clients"
        CLI[CLI]
        DESK[Desktop App]
        MOB[Mobile App]
        CUST[Custom HTTP/WS Clients]
    end

    subgraph "librefang-api"
        MW[Middleware: Auth, Rate Limit, Telemetry]
        REST[REST Route Handlers]
        WS[WebSocket Handlers]
        DASH[Dashboard SPA]
    end

    subgraph "In-Process Crates"
        KERN[librefang-kernel]
        MEM[librefang-memory]
        LLM[librefang-llm-drivers]
        SKILLS[librefang-skills]
        HANDS[librefang-hands]
        EXT[librefang-extensions]
        CH[librefang-channels]
    end

    CLI --> MW
    DESK --> MW
    MOB --> MW
    CUST --> MW
    MW --> REST
    MW --> WS
    REST --> KERN
    WS --> KERN    KERN --> MEM
    KERN --> LLM
    KERN --> SKILLS
    KERN --> HANDS
    KERN --> EXT
    DASH -.-> |"static/react"| REST
```

## Public API Entry Points

### `server::build_router(kernel, addr)`

The primary assembly point. Takes a kernel handle and listen address, constructs the shared `AppState`, registers all route handlers, mounts middleware layers, and returns a fully-wired axum `Router` ready to serve.

### `routes::*`

Endpoint handlers organized by domain. Based on the crate's dependencies and README, the route modules cover:

- **Agent management** — lifecycle, configuration
- **Sessions** — conversation sessions with agents
- **Channels** — channel adapter management (sidecar orchestration)
- **Approvals** — human-in-the-loop approval workflows
- **MCP** — Model Context Protocol endpoints
- **Peer/A2A networking** — agent-to-agent communication
- **Import** — data import endpoints
- **Health/telemetry** — diagnostics and metrics

### `middleware`

Three-tier route visibility for authentication:

| Constant | Meaning |
|---|---|
| `PUBLIC_ROUTES_ALWAYS` | No auth required for any method |
| `PUBLIC_ROUTES_GET_ONLY` | Auth bypassed for GET, required for mutations |
| `PUBLIC_ROUTES_DASHBOARD_READS` | Dashboard read endpoints accessible without auth |

Additional middleware layers handle rate limiting (via `governor`), OpenTelemetry tracing, and request logging.

### `ws`

WebSocket authentication and streaming handlers. These manage long-lived connections for real-time session updates, approval notifications, and other push-based interactions.

## Dashboard SPA

The React dashboard (TypeScript, TanStack Query) lives under `dashboard/` and is built separately via `cargo xtask build-web`. The compiled assets are embedded into the binary using `include_dir!` from the `static/react` directory.

At runtime, if the embedded directory is empty (fresh build without the web assets), the server falls back to serving assets from `~/.librefang/dashboard/`.

## Feature Flags

### `telemetry` (default)

Enables OpenTelemetry trace export and Prometheus metrics. Pulls in `opentelemetry`, `opentelemetry-otlp`, `tracing-opentelemetry`, `metrics`, and `metrics-exporter-prometheus`.

### Removed channel feature flags

The historical `core-channels`, `all-channels`, `channel-*`, `mini`, and `all-channels-no-email` features have been removed. All channel adapters now run as out-of-process sidecars. See `SIDECAR_CATALOG` in `src/routes/channels.rs` and the `librefang.sidecar.adapters.*` SDK entries.

## Build Script (`build.rs`)

The build script performs three tasks:

1. **Ensures `static/react` exists** — Creates the directory if absent so `include_dir!` never fails on fresh clones.

2. **Captures build metadata** — Sets `GIT_SHA`, `BUILD_DATE`, and `RUSTC_VERSION` as compile-time environment variables accessible via `env!()` in the binary.

3. **Git SHA resolution order**:
   - `GITHUB_SHA` environment variable (GitHub Actions) → truncated to 7 characters
   - `CI_COMMIT_SHA` environment variable (GitLab CI, generic) → truncated to 7 characters
   - `git rev-parse --short HEAD`, with `git` located via `which::which("git")`
   - Falls back to `"unknown"`

   `BUILD_DATE` uses `chrono::Utc::now()` rather than shelling out to the platform-specific `date` command.

The script declares `rerun-if-env-changed` directives for the CI variables so cargo invalidates appropriately.

## Platform-Specific Dependencies

### Unix (`cfg(unix)`)

- **`rustix`** (with `process` feature) — low-level Unix API access
- **`libc`** — FFI bindings for system calls

### Windows (`cfg(windows)`)

- **`windows-sys`** — used for `ConvertStringSecurityDescriptorToSecurityDescriptorW` to set the named pipe DACL to the daemon's owner SID only (refs #3313). This restricts the ACP named-pipe listener so other local users cannot connect. Features: `Win32_Foundation`, `Win32_Security`, `Win32_Security_Authorization`.

## Authentication and Security

The crate integrates several authentication and security mechanisms:

- **JWT validation** — `jsonwebtoken` for token verification with caching (`validate_jwt_cached`)
- **Password hashing** — `argon2` for credential storage
- **HMAC** — `hmac` + `sha2` for message authentication
- **Constant-time comparison** — `subtle` for timing-safe checks
- **TOTP support** — tested via `totp-rs` in dev-dependencies
- **Rate limiting** — `governor` middleware
- **OIDC integration** — dev-dependencies include `rsa` for generating test keypairs and signing JWTs in-process to test `sub` claim enforcement end-to-end without a live identity provider (#5128)

## Key External Dependencies

| Crate | Purpose |
|---|---|
| `axum`, `tower`, `tower-http` | HTTP framework and middleware |
| `utoipa` + `schemars` | OpenAPI schema generation |
| `tokio` | Async runtime |
| `dashmap`, `arc-swap` | Concurrent state management |
| `governor` | Request rate limiting |
| `reqwest` | Outbound HTTP (proxying, webhook delivery) |
| `flate2`, `tar`, `zip` | Archive handling for imports/exports |
| `portable-pty` | PTY management for terminal sessions |
| `include_dir` | Compile-time embedding of dashboard assets |

## OpenAPI

The committed `openapi.json` at the workspace root is regenerated by:

```
cargo xtask codegen --openapi
```

Drift is verified in CI against hash baselines in `xtask/baselines/`. Do not edit `openapi.json` by hand; regenerate it after any route or schema changes.

## Testing

Dev-dependencies include `librefang-testing` for shared test utilities and `librefang-runtime` for tool-exec backend integration tests (#3332). The OIDC integration test spins up a local axum listener serving a JWKS endpoint and drives `validate_jwt_cached` end-to-end.