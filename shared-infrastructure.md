# Shared Infrastructure

# Shared Infrastructure

Foundational crates that every other layer of LibreFang depends on. These modules handle cross-cutting concerns — HTTP transport, telemetry, cost enforcement, message routing, sandboxed skill execution, kernel-runtime decoupling, test mocking, and framework migration — so that higher-level crates don't duplicate plumbing.

## Sub-modules

| Crate | Responsibility |
|-------|---------------|
| [librefang-http](librefang-http-src.md) | Uniform `reqwest::Client` construction with portable TLS roots and proxy support |
| [librefang-telemetry](librefang-telemetry-src.md) | HTTP metrics instrumentation (`metrics` facade) exported to Prometheus |
| [librefang-kernel-handle](librefang-kernel-handle-src.md) | Async trait that breaks the kernel ↔ runtime circular dependency |
| [librefang-kernel-metering](librefang-kernel-metering-src.md) | Per-agent, per-provider, and global LLM spending quotas |
| [librefang-kernel-router](librefang-kernel-router-src.md) | Keyword + embedding-based message-to-agent routing |
| [librefang-runtime-wasm](librefang-runtime-wasm-src.md) | Wasmtime sandbox for untrusted WASM skills with fuel metering and capability gating |
| [librefang-migrate](librefang-migrate-src.md) | Imports agents and config from OpenClaw and OpenFang into LibreFang's TOML format |
| [librefang-testing](librefang-testing-src.md) | `MockKernelBuilder` and test helpers used workspace-wide |

## How they fit together

```
┌──────────────────────────────────────────────────────────┐
│                    Application Layer                      │
│         CLI  ·  HTTP API  ·  TUI                         │
└──────┬──────────┬──────────────┬────────────────────────┘
       │          │              │
       ▼          ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐
│ migrate  │ │ telemetry│ │   testing    │
│          │ │          │ │(MockKernel   │
│          │ │          │ │ Builder)     │
└──────────┘ └────┬─────┘ └──────┬───────┘
                  │              │
       ┌──────────┼──────────────┘
       ▼          ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐  ┌─────────┐
│   http   │ │ kernel-  │ │ kernel-      │  │ kernel- │
│ (client  │ │ handle   │ │ metering     │  │ router  │
│  builder)│ │ (trait)  │ │ (quotas)     │  │ (route) │
└────┬─────┘ └────┬─────┘ └──────┬───────┘  └─────────┘
     │            │              │
     └────────────┼──────────────┘
                  ▼
        ┌──────────────────┐
        │  runtime-wasm    │
        │  (WASM sandbox)  │
        └──────────────────┘
```

Three key dependency chains define the architecture:

1. **Every outbound HTTP call flows through [librefang-http](librefang-http-src.md).** The API routes, WASM host functions (`host_net_fetch`), catalog sync, CLI daemon health checks, and the migration scanner all use the same client builder, ensuring consistent proxy routing and TLS trust regardless of the host environment.

2. **The runtime talks to the kernel through [librefang-kernel-handle](librefang-kernel-handle-src.md).** The WASM sandbox, the agent loop, and tool execution all invoke methods on the `KernelHandle` trait rather than depending on the kernel crate directly. The kernel injects its concrete implementation at startup. This keeps the runtime compilable and testable in isolation.

3. **[librefang-testing](librefang-testing-src.md) underpins the entire test suite.** Every crate from the CLI to the runtime modules (OAuth, MCP, provider health, skill registries) uses `MockKernelBuilder` to spin up a lightweight kernel without external services. The mock kernel itself wires in [librefang-http](librefang-http-src.md) for realistic HTTP behaviour during tests.

## Key cross-cutting workflows

### LLM request lifecycle

```
User message
  → kernel-router selects agent/template
  → agent loop runs
  → LLM dispatch calls kernel-metering to check quotas
  → metering records cost and enforces limits
  → telemetry increments counters and histograms
  → any outbound fetch uses http client builder
```

### WASM skill execution

```
kernel.spawn_agent / execute_skill
  → runtime-wasm WasmSandbox.execute
  → host functions dispatched (fs_read, net_fetch, agent_spawn, …)
  → each host call checks capabilities, then invokes kernel-handle
  → kernel-handle implementation delegates to the real kernel
  → metering tracks any LLM calls triggered by the skill
```

### Migration import

```
CLI / API / TUI invokes migrate
  → scans source framework layout (OpenClaw JSON5/YAML or OpenFang)
  → converts to LibreFang TOML workspace structure
  → uses http client for any URL resolution during scan
  → produces a Markdown report via migrate::report
```

## Design principles

- **One client, one TLS story.** Building your own `reqwest::Client` anywhere else is a lint error in practice. Use [librefang-http](librefang-http-src.md).
- **Trait-based decoupling.** The kernel and runtime never import each other directly; they agree on [librefang-kernel-handle](librefang-kernel-handle-src.md).
- **Deny-by-default sandboxing.** [librefang-runtime-wasm](librefang-runtime-wasm-src.md) grants no capabilities unless explicitly requested; fuel and epoch timeouts prevent runaway computation.
- **Observability from day one.** [librefang-telemetry](librefang-telemetry-src.md) instruments every HTTP request automatically via middleware — no manual metric calls needed in route handlers.