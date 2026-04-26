# Infrastructure & Utilities

# Infrastructure & Utilities

Shared foundations that every other LibreFang crate depends on — HTTP transport, kernel abstractions, cost enforcement, routing, observability, migration tooling, and test infrastructure.

## What This Module Group Provides

These crates have no dependency on each other (except [`librefang-testing`](librefang-testing-src.md), which mocks several of them). They are grouped here because they are horizontal concerns: cross-cutting capabilities that the kernel, runtime, API, and CLI layers all consume but that don't implement business logic themselves.

| Sub-module | One-line role |
|---|---|
| [`librefang-http`](librefang-http-src.md) | Centralized HTTP client factory — proxy config, TLS fallback, uniform request handling |
| [`librefang-kernel-handle`](librefang-kernel-handle-src.md) | Trait that lets runtime tools call back into the kernel without creating a circular dependency |
| [`librefang-kernel-metering`](librefang-kernel-metering-src.md) | Token/cost tracking and quota enforcement across agent, global, provider, and user budgets |
| [`librefang-kernel-router`](librefang-kernel-router-src.md) | Routes incoming messages to the best hand or template using keywords, metadata, and embeddings |
| [`librefang-migrate`](librefang-migrate-src.md) | Imports agents, channels, sessions, and config from external frameworks (OpenClaw) into LibreFang format |
| [`librefang-telemetry`](librefang-telemetry-src.md) | OpenTelemetry/Prometheus recording helpers and HTTP path normalization |
| [`librefang-testing`](librefang-testing-src.md) | In-memory mock kernel, mock LLM drivers, and `TestAppState` builder for integration tests |

## How They Fit Together

```
┌─────────────────────────────────────────────────────┐
│                   Consumer layers                    │
│          (API · CLI · Kernel · Runtime)              │
└──────┬──────────┬──────────┬──────────┬─────────────┘
       │          │          │          │
       ▼          ▼          ▼          ▼
  ┌─────────┐ ┌────────┐ ┌───────┐ ┌──────────┐
  │  HTTP   │ │Metering│ │Router │ │Telemetry │
  │ client  │ │ quotas │ │routing│ │ metrics  │
  └─────────┘ └────────┘ └───────┘ └──────────┘
       ▲          ▲          ▲
       │          │          │
       │      ┌───┴──────────┴───┐
       │      │  KernelHandle    │
       │      │  (trait bridge)  │
       │      └──────────────────┘
       │
  ┌────┴─────────────────────────────────────────┐
  │              librefang-testing                │
  │  (mocks for all of the above, in-memory DB)  │
  └──────────────────────────────────────────────┘
```

**Request lifecycle across crates:** When a user message arrives, [`librefang-kernel-router`](librefang-kernel-router-src.md) selects the appropriate hand or template. Before the LLM call is dispatched, [`librefang-kernel-metering`](librefang-kernel-metering-src.md) checks quota across all budget dimensions. The runtime processes the response using [`librefang-kernel-handle`](librefang-kernel-handle-src.md) callbacks to reach back into the kernel (spawning sub-agents, reading shared memory, requesting approval). Every outbound HTTP request — whether to an LLM provider or an external API — flows through [`librefang-http`](librefang-http-src.md). Throughout, [`librefang-telemetry`](librefang-telemetry-src.md) records request metrics.

**Migration path:** [`librefang-migrate`](librefang-migrate-src.md) operates independently at import time, converting external framework data (OpenClaw JSON5/YAML layouts, channel definitions, secrets) into LibreFang's TOML-based workspace format.

**Testing foundation:** Every crate's integration tests rely on [`librefang-testing`](librefang-testing-src.md), which provides `TestAppState`, `MockKernelBuilder`, configurable mock LLM drivers, and an in-memory SQLite database — enabling full route and kernel testing without external services.