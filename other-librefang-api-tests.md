# Other — librefang-api-tests

# librefang-api-tests

Integration, lifecycle, load, and spec-generation tests for the LibreFang HTTP API. These tests exercise the full stack—kernel boot, axum router, real HTTP requests via `reqwest`—with no mocks.

## Test Files

| File | Purpose | Runs LLM? |
|---|---|---|
| `api_integration_test.rs` | Full API surface: agents, workflows, triggers, tools, auth, MCP bridge, versioning, config hot-reload, migration, hands | Only `test_send_message_with_llm` (gated by `GROQ_API_KEY`) |
| `daemon_lifecycle_test.rs` | PID file management, `DaemonInfo` serialization, server startup/shutdown sequence | No |
| `load_test.rs` | Concurrent spawns, endpoint latency percentiles, session churn, workflow creation, Prometheus metrics under sustained load | No |
| `openapi_spec_test.rs` | Generates and validates the OpenAPI spec, writes `openapi.json` to the repo root | No |

## Running

```sh
# All API tests (fast, no LLM)
cargo test -p librefang-api --test api_integration_test --test daemon_lifecycle_test --test load_test --test openapi_spec_test -- --nocapture

# Include real LLM calls (requires GROQ_API_KEY)
GROQ_API_KEY=... cargo test -p librefang-api --test api_integration_test -- --nocapture

# Only load tests (verbose metrics on stderr)
cargo test -p librefang-api --test load_test -- --nocapture
```

Two load tests are marked `#[ignore]` due to known race conditions in concurrent agent lifecycle:
- `load_concurrent_agent_spawns`
- `load_spawn_kill_cycle`

Run them explicitly with `-- --ignored` if needed.

## Test Infrastructure

### TestServer Harness

`TestServer` boots a real kernel, builds a minimal axum `Router` with the routes under test, binds to `127.0.0.1:0` (OS-assigned port), and spawns the server in a background tokio task. On drop, it calls `kernel.shutdown()`.

Three constructors provide different configurations:

| Constructor | Provider | Auth | Use case |
|---|---|---|---|
| `start_test_server()` | ollama (no real calls) | none | Default for most tests |
| `start_test_server_with_llm()` | groq | none | `GROQ_API_KEY`-gated LLM round-trip |
| `start_test_server_with_auth(api_key)` | ollama | Bearer token | Auth middleware tests |

Each creates a `tempfile::TempDir` for `home_dir` and `data_dir`, writes a `KernelConfig` to disk, and constructs `AppState` with all required fields (caches, session maps, shutdown notify, etc.). The `_tmp` field is stored so the temp directory lives until the struct is dropped.

### FullRouterHarness

Uses `server::build_router()` instead of manually constructing routes. This gives the full production router—including versioned `/api/v1/` aliases, dashboard static assets, locale files, the `/api/versions` endpoint, and middleware chain. Tests use `app.clone().oneshot()` for in-process requests without a live TCP listener.

### Test Manifests

Three TOML manifests define test agents with varying capability sets:

- **`TEST_MANIFEST`** — ollama provider, `tools = ["file_read"]`. No real LLM calls. Used for CRUD, pagination, search, and error-path tests.
- **`LLM_MANIFEST`** — groq provider, `llama-3.3-70b-versatile`. Requires `GROQ_API_KEY`. Only used by `test_send_message_with_llm`.
- **`MCP_TEST_MANIFEST`** — ollama provider, `tools = ["cron_list", "cron_create", "cron_cancel"]`. Used by `/mcp` bridge tests to exercise caller-context rehydration (issue #2699).

```mermaid
graph TD
    A[Test Function] --> B{Harness type?}
    B -->|HTTP on random port| C[TestServer]
    B -->|In-process oneshot| D[FullRouterHarness]
    C --> E[start_test_server / start_test_server_with_auth]
    E --> F[LibreFangKernel::boot_with_config]
    F --> G[AppState + axum Router]
    G --> H[TcpListener bind 0]
    H --> I[tokio::spawn axum::serve]
    D --> J[server::build_router]
    J --> F
    J --> K[app.clone().oneshot]
    A --> L[reqwest::Client]
    L --> I
    A --> K
```

## Coverage by Domain

### Agent Lifecycle

`test_spawn_list_kill_agent`, `test_multiple_agents_lifecycle`, `test_kill_nonexistent_agent_returns_404`, `test_invalid_agent_id_returns_400`, `test_spawn_invalid_manifest_returns_400`, `test_agent_session_empty`, `test_agent_monitoring_endpoints`

These cover the full CRUD cycle: spawn from TOML manifest, list with pagination/sort/search, fetch session state, read metrics and filtered logs, and kill. Error paths validate 400 for malformed UUIDs, 404 for unknown agents, and 400 for invalid manifest TOML.

### Agent List Filtering & Pagination

`test_agent_list_paginated_response_format`, `test_agent_list_pagination`, `test_agent_list_text_search`, `test_agent_list_invalid_sort_returns_400`, `test_agent_list_valid_sort_fields`, `test_agent_list_limit_clamped_to_max`

Validates the paginated envelope (`items`, `total`, `offset`, `limit`), `q=` text search across name and description, `sort=` by `name`/`created_at`/`last_active`/`state`, and limit clamping to 100.

### Workflows and Triggers

`test_workflow_crud`, `test_trigger_crud`

Workflow tests create a named workflow with steps referencing an agent, then list to verify. Trigger tests exercise the full lifecycle: create with a lifecycle pattern and `max_fires`, list unfiltered and filtered by `agent_id`, delete, then verify empty.

### Tools

`test_list_tools`, `test_get_tool_found`, `test_get_tool_not_found`

Lists all registered tools, fetches a specific tool by name (verifying `name`, `description`, `input_schema`), and returns 404 for unknown tools.

### Authentication

`test_auth_health_is_public`, `test_auth_rejects_no_token`, `test_auth_rejects_wrong_token`, `test_auth_accepts_correct_token`, `test_auth_disabled_when_no_key`

Uses `start_test_server_with_auth("secret-key-123")` to enable Bearer-token auth via the `middleware::auth` layer. Validates that `/api/health` remains public, protected endpoints return 401 with descriptive error messages ("Missing" vs "Invalid"), correct tokens are accepted, and an empty API key disables auth entirely.

### MCP Bridge (Issue #2699)

`test_mcp_http_rehydrates_caller_context_from_agent_header`, `test_mcp_http_invalid_agent_header_falls_back_to_unauthenticated`, `test_mcp_http_unrestricted_agent_can_call_any_tool`, `test_mcp_http_enforces_agent_tool_allowlist`

Regression guards for the `/mcp` HTTP bridge. Tests verify:
- `X-LibreFang-Agent-Id` header rehydrates caller context so tools like `cron_list` succeed
- Missing or invalid headers degrade gracefully to the unauthenticated error path (no 500)
- Agents with no `[capabilities].tools` (unrestricted) can call any tool through the bridge
- Agents with a specific tool allowlist are denied tools not in their manifest (no privilege escalation)

### API Versioning

`test_build_router_exposes_versioned_api_aliases`, `test_build_router_path_version_beats_unknown_accept_header`, `test_build_router_unauthorized_responses_include_api_version_header`

Uses `FullRouterHarness` to verify `/api/v1/` aliases work identically to `/api/` routes, path-based versioning wins over conflicting `Accept` headers, the `x-api-version: v1` header is set on all responses (including 401s), and `/api/versions` returns the current and supported version list.

### Dashboard & Providers

`test_build_router_serves_dashboard_locales`, `test_build_router_providers_marks_local_providers`

Verifies locale JSON files (`en.json`, `zh-CN.json`, `ja.json`) are served with correct content and content-type, and the `/api/providers` endpoint marks local providers (ollama) with `is_local: true`.

### Config Hot-Reload

`test_config_reload_hot_reloads_proxy_changes`

Writes a modified `config.toml` with a proxy setting, hits `/api/config/reload`, and asserts `restart_required: false` with `ReloadProxy` in `hot_actions_applied`.

### Migration

`test_run_migrate_uses_daemon_home_when_target_dir_is_empty`

Creates an OpenClaw-format source directory, posts to `/api/migrate` with an empty `target_dir` (should default to daemon home), and verifies the resulting `config.toml`, `agent.toml`, and `migration_report.md` are written to the kernel's home directory.

### Hands (Active)

`list_active_hands_includes_definition_metadata`

Installs a hand definition, activates it, assigns agent IDs to roles via `set_agents`, and hits `/api/hands/active` to verify the response includes `hand_name`, `hand_icon`, `coordinator_role`, and `agent_ids` alongside legacy fields.

### Daemon Lifecycle

`test_daemon_info_serde_roundtrip`, `test_read_daemon_info_from_file`, `test_read_daemon_info_missing_file`, `test_read_daemon_info_corrupt_json`, `test_full_daemon_lifecycle`, `test_stale_daemon_info_detection`, `test_server_immediate_responsiveness`

Tests `DaemonInfo` JSON serialization, `read_daemon_info()` for valid/missing/corrupt files, the full startup→health-check→shutdown lifecycle with PID file management, and that the health endpoint responds in under 1 second.

### Load & Performance

`load_endpoint_latency`, `load_concurrent_reads`, `load_session_management`, `load_workflow_operations`, `load_metrics_sustained` (plus two `#[ignore]` tests for concurrent spawns and spawn/kill cycles).

Latency tests warm up each endpoint (10 requests) then measure p50/p95/p99 over 100 iterations, gating on p95 < 1 second. Concurrent read tests hammer 50 simultaneous requests across health/agents/status/metrics. Session tests create 10 sessions, list them, and rapidly switch between them. Workflow tests create 15 workflows concurrently. Metrics tests fire 200 sequential requests at `/api/metrics` and verify the `librefang_agents_active` metric is present.

All load tests print throughput and latency metrics to stderr via `eprintln!`.

### OpenAPI Spec

`generate_openapi_json`

Uses `utoipa::OpenApi::openapi()` on `ApiDoc` to generate the spec, validates it has 100+ paths, and writes it to `<repo_root>/openapi.json` for SDK codegen and CI consumption.

## Conventions

- **All tests use `#[tokio::test(flavor = "multi_thread")]`** because the kernel spawns background tasks that require a multi-threaded runtime.
- **Temp directories** are held alive by the `_tmp` field on harness structs, ensuring config files and data directories persist for the test duration.
- **Kernel shutdown** is deterministic: both `TestServer` and `FullRouterHarness` implement `Drop` to call `kernel.shutdown()`.
- **No external dependencies** except optional `GROQ_API_KEY`. All non-LLM tests use ollama as a provider with no actual API calls.
- **Request IDs**: `test_request_id_header_is_uuid` validates the `x-request-id` middleware header is a valid UUID on every response.