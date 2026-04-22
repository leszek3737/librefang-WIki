# Agent Kernel

# Agent Kernel

The Agent Kernel is the central nervous system of LibreFang. It sits between the transport layer (Telegram, Discord, WhatsApp, API) and the agent runtime, governing agent behavior, enforcing security boundaries, controlling costs, and routing messages to the right specialist.

## Sub-modules at a glance

| Sub-module | Role |
|---|---|
| [Kernel Core](librefang-kernel-src.md) | Orchestrator — auth, approval gating, workflow DAG execution, triggers, wizard-based agent planning, transport gateways |
| [Kernel Handle](librefang-kernel-handle-src.md) | Dependency inversion — defines the `KernelHandle` trait so the agent runtime can call into the kernel without circular dependencies |
| [Metering Engine](librefang-kernel-metering-src.md) | Cost control — records every LLM usage event and enforces spending quotas at per-agent, per-provider, and global levels |
| [Router](librefang-kernel-router-src.md) | Message routing — scores incoming messages against agent templates and tool-bearing hands using keyword rules and semantic similarity |

## How they fit together

```
Transport ──► Router ──► Kernel Core ──► Agent Runtime
                │            │                │
                │            │    ◄───────────┤
                │            │   KernelHandle │
                │            │   (trait impl) │
                │            │                │
                │       AuthManager      MeteringEngine
                │       ApprovalManager   (records every LLM call)
                │       Workflow DAG
                │       Triggers
```

1. A message arrives from a transport. The **Router** scores it and selects the best agent template and hand.
2. The **Kernel Core** authorizes the user (`AuthManager`), gates dangerous operations through approval (`ApprovalManager`), and runs the agent loop.
3. The agent runtime communicates back to the kernel through the **Kernel Handle** — spawning sub-agents, reading shared memory, posting tasks — without knowing kernel internals.
4. Every LLM invocation passes through the **Metering Engine**, which records usage and checks quotas in a single SQLite transaction before the call proceeds.

## Key cross-cutting workflows

**Message handling** — Router selects the agent/hand → core enforces auth and approval → metering tracks cost → results flow back through the handle.

**Workflow DAGs** — The core's workflow engine supports conditional steps, loops, parallel branches, and error-mode policies (skip or retry). Steps can inject context from previous steps, and output variables carry forward through the DAG.

**Triggers and cooldowns** — Event-driven triggers can seize and restore agent assignments with cooldown suppression to prevent rapid re-firing.

**Agent creation wizard** — Parses `AgentIntent` into a tiered plan, auto-adding browser or web tools based on the detected intent.

**Secret management** — TOTP setup and revocation flows cross from the kernel's `McpOAuthProvider` into `librefang-extensions`' vault, which resolves master keys via machine fingerprints and OS keyring integration.