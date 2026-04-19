# Agent Kernel

# Agent Kernel

The Agent Kernel is the runtime core of the LibreFang Agent Operating System. It orchestrates agent lifecycles, enforces budgets, routes messages, manages workflows, and provides the trait boundary that lets the agent runtime call into kernel operations without creating circular dependencies.

## Sub-modules

| Module | Responsibility |
|---|---|
| [**Core**](librefang-kernel-src.md) | Top-level kernel re-exports (`LibreFangKernel`, `DeliveryTracker`), plus internal subsystems: scheduler, supervisor, approval, auth/RBAC, event bus, registry, workflow engine, triggers, and wizard |
| [**Handle**](librefang-kernel-handle-src.md) | Defines the `KernelHandle` trait — the abstract interface the agent runtime uses to spawn agents, send messages, and manage tasks without depending on kernel internals |
| [**Metering**](librefang-kernel-metering-src.md) | Cost accounting and budget enforcement for every LLM call, with per-agent, per-provider, and global limits across hourly/daily/monthly windows |
| [**Router**](librefang-kernel-router-src.md) | Routes incoming user messages to the best-matching agent template or hand using layered keyword matching and optional semantic similarity |

## How they connect

```
User Message
     │
     ▼
  Router ──▶ selects agent template
     │
     ▼
 LibreFangKernel (core)
  ├── Supervisor spawns agent
  ├── Scheduler queues work
  ├── Auth/RBAC checks permissions
  └── Approval gates sensitive actions
     │
     ▼
 Agent Runtime (librefang-runtime)
  ├── calls through KernelHandle trait
  ├── Metering intercepts every LLM call
  │     ├── estimates cost
  │     ├── checks budgets (agent / provider / global)
  │     └── records usage to SQLite via UsageStore
  └── EventBus carries inter-agent messages
```

**Key integration points:**

1. **Decoupling boundary.** The kernel implements the `KernelHandle` trait from [Handle](librefang-kernel-handle-src.md) and injects it into the agent loop at startup. The runtime never sees concrete kernel types — it only calls through the trait object.

2. **Message routing → lifecycle.** When a message arrives, the [Router](librefang-kernel-router-src.md) scores it against registered templates and hands. The core's `LibreFangKernel` then uses the scheduler and supervisor to spin up the chosen agent, subject to auth and approval checks.

3. **LLM cost gating.** Every LLM invocation passes through the [Metering](librefang-kernel-metering-src.md) engine before execution. If any budget axis (per-agent, per-provider, or global) is exceeded for the current window, the call is rejected. This protects against runaway spend regardless of which agent or workflow triggered the call.

4. **Workflow orchestration.** The core's workflow engine ([in workflow.rs](librefang-kernel-src.md)) manages DAG-based multi-step runs — instantiating plans, expanding variables, executing steps sequentially or in parallel, and persisting state. Workflows can fan out across multiple agents, each of which is metered and permission-checked independently.

5. **Triggers and automation.** The core's trigger system ([in triggers.rs](librefang-kernel-src.md)) evaluates lifecycle and external events against registered patterns, firing actions automatically. Triggers are budgeted per-event to prevent cascading agent spawns from consuming unbounded resources.

For details on any individual subsystem, follow the links above.