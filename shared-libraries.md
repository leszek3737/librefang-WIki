# Shared Libraries

# Shared Libraries

Reusable crates that form the infrastructure layer of the LibreFang Agent OS. Every higher-level crate — the kernel, CLI, API server, and TUI — depends on one or more of these libraries for HTTP connectivity, authentication, cost enforcement, message routing, sandboxing, observability, testing, and migration.

## Sub-module Index

| Crate | Responsibility |
|---|---|
| [`librefang-http`](librefang-http-src.md) | Centralized HTTP client factory — proxy settings, TLS trust anchors, bundled Mozilla CA roots |
| [`librefang-kernel-handle`](librefang-kernel-handle-src.md) | Granular role traits that define the kernel ↔ runtime seam |
| [`librefang-kernel-metering`](librefang-kernel-metering-src.md) | Multi-axis LLM spending quotas (per-agent, per-user, per-provider, global) |
| [`librefang-kernel-router`](librefang-kernel-router-src.md) | Message routing — keyword, metadata, and embedding-based agent/hand selection |
| [`librefang-migrate`](librefang-migrate-src.md) | Import agent configurations from OpenClaw and other external frameworks |
| [`librefang-runtime-mcp`](librefang-runtime-mcp-src.md) | MCP client — multi-transport server connections, tool discovery, outbound taint scanning |
| [`librefang-runtime-oauth`](librefang-runtime-oauth-src.md) | OAuth 2.0 flows for OpenAI ChatGPT and GitHub Copilot, with zeroizing token protection |
| [`librefang-runtime-wasm`](librefang-runtime-wasm-src.md) | Deny-by-default WASM sandbox for untrusted skill/plugin execution |
| [`librefang-telemetry`](librefang-telemetry-src.md) | OpenTelemetry + Prometheus metrics, HTTP request observability |
| [`librefang-testing`](librefang-testing-src.md) | Mock kernel, test app harness, and helpers for integration tests across the workspace |

## How They Fit Together

```
┌─────────────────────────────────────────────────────────────┐
│                    Higher-Level Crates                       │
│          (kernel · API · CLI · TUI · memory)                 │
└──────┬──────────┬──────────┬──────────┬──────────┬──────────┘
       │          │          │          │          │
  ┌────▼────┐ ┌──▼───┐ ┌───▼────┐ ┌───▼───┐ ┌───▼────┐
  │testing  │ │tel-  │ │kernel- │ │kernel-│ │kernel- │
  │         │ │emetry│ │handle  │ │meter- │ │router  │
  │         │ │      │ │(traits)│ │ing    │ │        │
  └─────────┘ └──┬───┘ └───┬────┘ └───┬───┘ └───┬────┘
                │          │          │          │
  ┌─────────────▼──────────▼──────────▼──────────▼────┐
  │                runtime layer                       │
  │  ┌────────┐ ┌─────────┐ ┌──────────────────────┐  │
  │  │runtime-│ │runtime- │ │  runtime-wasm         │  │
  │  │oauth   │ │mcp      │ │  (sandboxed skills)   │  │
  │  └───┬────┘ └────┬────┘ └──────────────────────┘  │
  │      │           │                                 │
  │  ┌───▼───────────▼──┐    ┌──────────────────┐      │
  │  │  librefang-http  │    │  librefang-      │      │
  │  │  (client factory)│    │  migrate         │      │
  │  └──────────────────┘    └──────────────────┘      │
  └────────────────────────────────────────────────────┘
```

### Layer Roles

- **Kernel traits** — [`librefang-kernel-handle`](librefang-kernel-handle-src.md) defines sixteen focused role traits (e.g. `ApprovalGate`, `ChannelIO`) that every runtime module depends on. Callers express narrow bounds rather than requiring the full kernel surface.

- **Shared HTTP** — [`librefang-http`](librefang-http-src.md) is the sole factory for outbound HTTP clients. Both [`librefang-runtime-mcp`](librefang-runtime-mcp-src.md) (SSE and Streamable HTTP transports) and [`librefang-runtime-oauth`](librefang-runtime-oauth-src.md) (token endpoint calls) route through it, ensuring consistent proxy configuration and TLS trust anchors.

- **Runtime modules** — [`librefang-runtime-mcp`](librefang-runtime-mcp-src.md), [`librefang-runtime-oauth`](librefang-runtime-oauth-src.md), and [`librefang-runtime-wasm`](librefang-runtime-wasm-src.md) each encapsulate a distinct execution concern (external tool servers, provider authentication, untrusted plugin sandboxing). They consume kernel traits via `librefang-kernel-handle` and network clients via `librefang-http`.

- **Cost & routing** — [`librefang-kernel-metering`](librefang-kernel-metering-src.md) enforces spending quotas before LLM calls proceed. [`librefang-kernel-router`](librefang-kernel-router-src.md) selects which agent template or hand should handle an incoming message. Both operate between the API layer and the agent runtime.

- **Observability & testing** — [`librefang-telemetry`](librefang-telemetry-src.md) provides metrics primitives consumed application-wide. [`librefang-testing`](librefang-testing-src.md) offers `MockKernelBuilder` and `TestAppState` so that integration tests across the workspace can exercise kernel behavior, API routes, and LLM drivers without network dependencies.

- **Migration** — [`librefang-migrate`](librefang-migrate-src.md) is standalone: it reads external framework configs (OpenClaw JSON5/YAML) and writes LibreFang workspace files, invoking the kernel only during staging and promotion.

## Key Cross-Module Workflows

### MCP OAuth with SSRF Protection

When a user initiates MCP server authentication, the request flows through the API route → [`librefang-runtime-mcp`](librefang-runtime-mcp-src.md) discovers OAuth metadata → [`librefang-http`](librefang-http-src.md) builds a proxied TLS client → SSRF checks (`blocked_v4` / `blocked_v6`) reject internal network addresses before any outbound call is made.

### LLM Call Budget Enforcement

Before an LLM driver issues a completion request, [`librefang-kernel-metering`](librefang-kernel-metering-src.md) checks four independent budget axes (agent, user, provider, global). If any hourly/daily/monthly window is exceeded, the call is rejected before tokens are consumed. Successful calls record actual token usage for future quota calculations.

### Sandbox Execution with Capability Checks

[`librefang-runtime-wasm`](librefang-runtime-wasm-src.md) receives a compiled WASM skill and executes it inside a fresh Wasmtime instance. Every host function call — environment reads, shell execution, key-value writes — passes through a capability gate configured in `SandboxConfig`. Missing capabilities produce compile-time errors in tests (via [`librefang-kernel-handle`](librefang-kernel-handle-src.md) narrow trait bounds) rather than silent runtime failures.