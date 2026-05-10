# Kernel Core

# Kernel Core

The kernel core is the foundation of the agent runtime, providing identity management, cost controls, message routing, the runtime–kernel interface, and shared HTTP infrastructure. Every other layer in the system ultimately depends on one or more of these sub-modules.

## Sub-modules at a glance

| Sub-module | Responsibility |
|---|---|
| [Kernel — Identity & Approvals](librefang-kernel-src.md) | Agent identity registry (atomic TOML persistence) and approval manager (SQLite-backed pending approvals, TOTP gates, audit trail, remembered decisions) |
| [Kernel — Metering](librefang-kernel-metering-src.md) | LLM cost tracking and quota enforcement at four levels (per-agent, global, per-provider, per-user) via a reservation ledger backed by a SQLite `UsageStore` |
| [Kernel — Router](librefang-kernel-router-src.md) | Message-to-agent routing using keyword rules, manifest metadata, and optional embedding similarity to select specialist hands or templates |
| [Kernel — Handle](librefang-kernel-handle-src.md) | 18 granular role traits (`AgentControl`, `MemoryAccess`, `ApprovalGate`, etc.) composing the `KernelHandle` supertrait — the seam between runtime and kernel |
| [Kernel — HTTP](librefang-http-src.md) | Centralized HTTP client factory with uniform proxy support, resilient TLS (webpki-roots + system certs), and bearer-token redaction |

## How they fit together

```
┌─────────────────────────────────────────────────────────┐
│                    Agent Runtime                         │
│          (depends on KernelHandle role traits)           │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────┼────────────────┐
          ▼            ▼                ▼
   ┌────────────┐ ┌──────────┐  ┌──────────────┐
   │  Router     │ │ Metering │  │  Identity &  │
   │  (who?)     │ │ (budget) │  │  Approvals   │
   └──────┬─────┘ └────┬─────┘  └──────┬───────┘
          │            │               │
          └────────────┼───────────────┘
                       ▼
              ┌─────────────────┐
              │  HTTP Client    │
              │  (all outbound) │
              └─────────────────┘
```

## Key cross-cutting workflows

**Incoming message processing.** A user message arrives and the [Router](librefang-kernel-router-src.md) scores candidate hands and templates to select the best specialist. The chosen agent's identity is loaded from the [Identity Registry](librefang-kernel-src.md). If the agent invokes a gated tool, the [Approval Manager](librefang-kernel-src.md) checks remembered decisions or queues a pending approval (with optional TOTP). All LLM calls during this flow pass through the [Metering engine](librefang-kernel-metering-src.md), which reserves estimated budget before dispatch and records actual cost atomically on completion.

**Provider health probing.** When the API lists available providers, the call flows through the [HTTP client](librefang-http-src.md) — `proxied_client_builder` → `build_http_client` → `tls_config` — ensuring consistent proxy routing and certificate validation even on minimal hosts (Docker, Termux, musl). Metering tracks the cost of these probe calls at the per-provider level.

**Skill workshop & agent learning.** The system captures repeated tool patterns and user corrections, persists pending skill candidates, and routes them through the Approval Manager for review. Approved skills are created under an agent identity from the registry, with the entire HTTP surface protected by the shared TLS/proxy configuration.

**Narrow dependency via role traits.** Rather than importing the full kernel surface, callers express minimal bounds — `T: ApprovalGate` for approval workflows, `T: MemoryAccess` for context retrieval — through the role traits defined in [Handle](librefang-kernel-handle-src.md). This keeps individual components testable and decoupled while the `KernelHandle` supertrait remains available for the main runtime entry point.