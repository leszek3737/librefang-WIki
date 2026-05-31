# Infrastructure Libraries

# Infrastructure Libraries

Shared foundations that every other LibreFang crate depends on — HTTP transport, telemetry, subprocess bridging, data migration, RL trajectory export, and test harnessing.

## Purpose

This module group collects the cross-cutting concerns that would otherwise be duplicated across the workspace. Each sub-module solves a single infrastructure problem and exposes a small, focused API surface. Higher-level crates (the kernel, API, runtime, channels) compose these building blocks rather than re-implementing them.

## Sub-module Overview

| Sub-module | Role | Depends on other `librefang-*`? |
|---|---|---|
| [librefang-http](librefang-http-src.md) | HTTP client factory with bundled TLS roots and centralised proxy state | No |
| [librefang-subprocess](librefang-subprocess-src.md) | JSON-over-stdio transport for long-lived sidecar processes | No |
| [librefang-telemetry](librefang-telemetry-src.md) | OpenTelemetry metrics + tracing, Prometheus `/metrics` export | No |
| [librefang-import](librefang-import-src.md) | Migrates configs, memory, and sessions from OpenClaw/OpenFang | Uses `librefang-http` |
| [librefang-rl-export](librefang-rl-export-src.md) | Uploads RL rollout trajectories to W&B, Tinker, or Atropos | Uses `librefang-http` |
| [librefang-testing](librefang-testing-src.md) | Mock kernel, fake HTTP clients, test app state — dev-dependency only | No |

## How They Fit Together

```
┌─────────────────────────────────────────────────────────────────┐
│  Consumers (kernel, api, cli, runtime, channels, desktop)       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  librefang-import ──┐                        librefang-testing  │
│                     ▼                           (dev-dep only)   │
│              librefang-http ◄──── librefang-rl-export           │
│                                                                  │
│  librefang-subprocess    librefang-telemetry                    │
│      (leaf)                  (leaf)                              │
└─────────────────────────────────────────────────────────────────┘
```

`librefang-http` and `librefang-subprocess` are the two "wire" crates — every outbound network call or sidecar pipe flows through one of them. `librefang-telemetry` is initialised once at startup and then used implicitly through the `metrics` and `tracing` macros. `librefang-import` and `librefang-rl-export` are higher-level: they encode domain-specific workflows (migration, RL upload) but still delegate their HTTP work to `librefang-http`. `librefang-testing` sits outside the normal dependency graph; it is a `dev-dependency` consumed by test suites across the workspace.

## Key Cross-cutting Workflows

### Authenticated outbound requests

MCP OAuth discovery, CLI self-update, and RL trajectory upload all follow the same path through `librefang-http`:

1. Caller obtains a **proxied client builder** (`proxied_client_builder`) — this reads the global proxy state (`active_proxy`).
2. Builder calls **`build_http_client`**, which seeds `rustls` with Mozilla CA roots (`webpki-roots`) plus any system certs via `tls_config`.
3. The resulting `reqwest::Client` is used for the specific request.

This ensures proxy settings and TLS trust anchors are applied uniformly regardless of which crate initiates the request.

### Migration pipeline

`librefang-import`'s `run_migration()` entry point dispatches to source-specific migrators (OpenClaw, OpenFang). The migrators read legacy config files, convert agents and memory sections, and write results with backup support. A `MigrateOptions` struct controls dry-run mode, idempotency, and user-edit preservation. The migration engine is wired into the `librefang migrate` CLI command, the `/api/config/migrate` HTTP endpoint, and the TUI init wizard.

### RL trajectory egress

`librefang-rl-export` receives an opaque `RlTrajectoryExport` (bytes + metadata) and delivers it to W&B, Tinker, or Atropos. Uploads go through `librefang-http` for consistent proxy/TLS handling and include retry logic for transient 5xx failures.

### Test isolation

`librefang-testing` provides `MockKernelBuilder` and `TestAppState` so that route handlers, kernel subsystems, and agent logic can be tested without a live daemon or network. Every crate listed above uses it in `#[cfg(test)]`.

## When to Reach for Which Crate

- **Making any HTTP request?** → [librefang-http](librefang-http-src.md)
- **Talking to a sidecar process over stdio?** → [librefang-subprocess](librefang-subprocess-src.md)
- **Recording a metric or span?** → [librefang-telemetry](librefang-telemetry-src.md)
- **Importing from another agent framework?** → [librefang-import](librefang-import-src.md)
- **Uploading RL rollouts?** → [librefang-rl-export](librefang-rl-export-src.md)
- **Writing a test?** → [librefang-testing](librefang-testing-src.md)