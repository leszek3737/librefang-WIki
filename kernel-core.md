# Kernel Core

# Kernel Core

The central runtime that orchestrates agent execution, enforces policy, and wires together every subsystem the agents depend on. Nothing in the agent loop touches shared state without passing through the kernel first.

## Sub-modules at a glance

| Sub-module | Responsibility |
|---|---|
| [Kernel Core (main)](librefang-kernel-src.md) | Approval gates, workflow DAG engine, RBAC/auth, retry policies, config hot-reload, background agent spawning, trajectory tracking, MCP OAuth, setup wizard |
| [Kernel Handle](librefang-kernel-handle-src.md) | `KernelHandle` async trait — the dependency-inversion boundary that lets the agent runtime call into the kernel without circular imports |
| [Router](librefang-kernel-router-src.md) | Maps inbound user messages to the best **hand** (multi-agent workflow) or **template** (single-agent specialist) via keyword, manifest metadata, and optional embedding similarity |
| [Metering](librefang-kernel-metering-src.md) | Two-phase LLM cost tracking: reserve budget before a call, settle actual usage after — prevents concurrent requests from overspending a quota |

## How they fit together

```
User message
    │
    ▼
┌──────────────────────────┐
│  Router                  │  selects hand or template
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│  KernelHandle            │  agent runtime calls through this trait
│  ┌────────────────────┐  │
│  │  Workflow Engine   │  │  DAG steps, conditionals, pause/resume
│  │  Approval Manager  │  │  gates dangerous tool calls for human review
│  │  Metering Engine   │  │  budget reserve → LLM call → settle
│  │  Auth / RBAC       │  │  role resolution with caching
│  │  Background Spawn  │  │  agent task queue
│  │  Config Reload     │  │  hot-reload planner (log-level → full restart)
│  └────────────────────┘  │
└──────────────────────────┘
```

The **Router** decides *what* runs. **KernelHandle** is the *only way* the agent runtime invokes kernel operations — spawning agents, routing messages, managing memory, enforcing approvals, checking quotas, and scheduling crons all flow through this single interface. The kernel implements the trait and injects it at startup.

## Key cross-cutting workflows

**Inbound message lifecycle.** A user message arrives → [Router](librefang-kernel-router-src.md) scores it against registered hands and templates → the winning hand/template is dispatched via [KernelHandle](librefang-kernel-handle-src.md) → the agent loop runs, with every tool call passing through approval and metering checks.

**Approval-gated tool call.** An agent invokes a tool → [Approval Manager](librefang-kernel-src.md) checks it against policy → if it matches, the call is held (blocking with oneshot receiver, or deferred returning a UUID) until an operator approves, denies, or skips it, or the timeout fallback triggers.

**LLM call with cost enforcement.** Before an LLM dispatch → [Metering](librefang-kernel-metering-src.md) reserves estimated spend against the global quota → the call executes → actual token usage is recorded and the reservation settled. Concurrent requests cannot collectively exceed the cap.

**Workflow DAG execution.** [Workflow Engine](librefang-kernel-src.md) loads step definitions, resolves dependencies into a DAG, and runs steps in parallel where possible — with conditional branching, retry policies via `RetryPolicy`, error modes (skip/retry), output variable propagation, and pause/resume support for long-running human-in-the-loop flows.

**Config hot-reload.** [Config Reload](librefang-kernel-src.md) evaluates what changed and produces a `ReloadPlan` scoped to the kernel's installed `ReloadCapabilities` — e.g., log-level changes apply instantly, while deeper changes require a restart.

**Vault-backed secrets.** TOTP setup and revocation flows route through the kernel's `mcp_oauth_provider` into the vault extension, resolving master keys from the OS keyring before checking or creating credentials.