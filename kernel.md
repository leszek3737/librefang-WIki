# Kernel

# Kernel

The kernel is the daemon's central coordination layer. It owns the data structures, state machines, and service contracts that every subsystem — the API server, channel adapters, agent spawn loop, and CLI — depends on.

## Sub-modules

| Sub-module | Responsibility |
|---|---|
| [Kernel Core](kernel-core.md) | Agent identity registry, approval gating, workflow engine, orchestration checkpoints, config reload, heartbeat, pairing, skill workshop, trajectory export, triggers, supervisor, and background agent lifecycle. |
| [Kernel Handle](kernel-handle.md) | Role-trait abstraction defining the contract between the agent runtime and kernel services. Refactored from a single god-trait into 20 focused capability traits so the runtime depends only on interfaces, not implementation. |
| [Kernel Router](kernel-router.md) | Message routing engine that selects the best agent template or hand for an inbound user message using keyword matching, manifest metadata, TOML overrides, and optional embedding-based semantic similarity. |
| [Kernel Metering](kernel-metering.md) | LLM cost-tracking and spending-quota enforcement. Every LLM call passes through twice — once to reserve budget before the request, once to settle actual cost afterward. |

## Architecture

```mermaid
flowchart TB
    subgraph External
        API["API Server / CLI / Channels"]
        RT["Agent Runtime"]
    end

    subgraph Kernel
        KC["Kernel Core<br/>identity · approval · workflows<br/>orchestration · pairing · heartbeat"]
        KH["Kernel Handle<br/>role-trait interfaces"]
        KR["Kernel Router<br/>template & hand selection"]
        KM["Kernel Metering<br/>budget reserve & settle"]
    end

    API -->|"create/update workflow,<br/>pair device, patch triggers"| KC
    API -->|"inbound message"| KR
    RT -->|"lifecycle, memory, queues,<br/>channel I/O"| KH
    KH -.->|"implemented by"| KC
    KC -->|"reserve_global_budget()"| KM
    KC -->|"settle cost"| KM
    KC -->|"auto_select_template()<br/>auto_select_hand()"| KR
end
```

## Key cross-cutting workflows

### Inbound message handling

An incoming user message hits the router via `auto_select_template()` or `auto_select_hand()`. The router combines strong/weak keyword aliases, manifest metadata, operator TOML overrides, and optional semantic embeddings to pick the best target. The kernel core then dispatches to the chosen agent through the handle traits.

### LLM call budgeting

Every LLM invocation follows a two-phase metering protocol. The kernel core calls `reserve_global_budget()` on the metering engine, which checks hourly/daily/monthly caps and records a reservation in the `CostReservationLedger`. After the call completes, `check_all_and_record()` settles the actual spend against the reservation and persists it to SQLite. If the provider signals exhaustion, the `ProviderExhaustionStore` tracks that separately.

### Workflow lifecycle

Workflow creation and updates flow through `validate()` → `validate_transform_template()` → `default()` → `new()` in the core workflow engine. Running workflows checkpoint state via `WorkflowCheckpoint` so they can resume after restarts. Output variables and gate conditions (e.g., string-equality checks against raw output) are evaluated at each step.

### Agent identity continuity

When a background agent is spawned (via `start_background_agents` → subprocess spawn with cooldown), the core's `AgentIdentityRegistry` ensures the first UUID issued to a name is canonical for the lifetime of that mapping. A kill retains the entry; only an explicit purge drops it. This means sessions, memories, and cron jobs keyed under that UUID are never silently orphaned by a rename or namespace bump.

### Approval gating

Dangerous tool invocations are routed through the core's `ApprovalManager` before execution. Two modes are supported: **blocking** (the agent loop waits for human resolution) and **deferred** (the tool call is packaged into a `DeferredToolExecution` and returned atomically on `resolve()` for the kernel to execute later).