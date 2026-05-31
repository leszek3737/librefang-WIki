# Kernel Core

# Kernel Core

The kernel core module group forms the central runtime infrastructure for the agent platform. It manages agent identity, execution safety, cost control, message routing, and defines the interface contract between the agent runtime and the kernel backend.

## Sub-module Overview

| Sub-module | Responsibility |
|---|---|
| [Kernel Core (`librefang-kernel-src`)](librefang-kernel-src.md) | Agent identity persistence across respawns; approval gating for dangerous tool operations |
| [Kernel Handle (`librefang-kernel-handle-src`)](librefang-kernel-handle-src.md) | Role traits that define the seam between agent runtime and kernel backend — 19 focused capability interfaces |
| [Metering Engine (`librefang-kernel-metering`)](librefang-kernel-metering-src.md) | LLM cost tracking and quota enforcement across global, agent, provider, and user scopes |
| [Kernel Router (`librefang-kernel-router`)](librefang-kernel-router-src.md) | Dispatches inbound messages to the best-matching specialist agent template or hand |

## How They Fit Together

```
Inbound message
      │
      ▼
┌─────────────┐    selects agent
│   Router     │─────────────────┐
└─────────────┘                  │
                                 ▼
                      ┌──────────────────┐
                      │ Identity Registry │  reuses canonical AgentId
                      └──────────────────┘
                                 │
                                 ▼
                       Agent runtime loop
                      ┌────────┴─────────┐
                      │                  │
               ┌──────▼──────┐   ┌───────▼───────┐
               │  Metering   │   │   Approval    │
               │  Engine     │   │   Manager     │
               └─────────────┘   └───────────────┘
                      │                  │
               reserve → settle    block or defer
```

All of these subsystems are accessed exclusively through the **role traits** defined in Kernel Handle. Agent code never touches concrete kernel types — it calls methods on bounded trait objects like `dyn MeteringHandle` or `dyn ApprovalHandle`, keeping each subsystem independently testable and replaceable.

## Key Workflows Spanning Sub-modules

### Message dispatch and agent spawn

An inbound user message hits the **Router**, which scores candidates via keyword rules, manifest metadata, and optional embedding similarity. The selected template name feeds into the spawn path, which consults the **Identity Registry** to reuse a canonical `AgentId` if one was previously recorded for that agent name. First UUID wins; subsequent respawns honor it even across renames or namespace changes.

### LLM call with cost control and safety gates

During execution, every LLM dispatch flows through the **Metering Engine**, which reserves estimated budget against four scopes (global, agent, provider, user) before the call proceeds. If the tool is flagged as dangerous, the **Approval Manager** intercepts it — either blocking the agent loop until a human resolves the request, or deferring the execution into a `DeferredToolExecution` package. After the LLM returns, metering settles the actual cost against the reservation.

### Agent restart and identity preservation

When an agent panics or is killed, the **Supervisor** orchestrates restart. The identity registry entry survives a normal kill (preserving sessions, memories, and cron keyed under the UUID). Only an explicit purge (`?purge_identity=true`) removes the entry, allowing a clean identity on next spawn. Disk persistence uses atomic rename writes so concurrent `register` calls never corrupt the TOML file.

## Design Principles

- **Trait-seamed architecture.** Every kernel capability is behind a role trait. New code narrows to only the bounds it needs; legacy code continues using the blanket `KernelHandle` supertrait alias.
- **In-memory authoritative, disk best-effort.** Both the identity registry and metering ledger treat their in-memory state as the source of truth. Disk I/O failures log warnings but never halt kernel operation.
- **Reserve-then-settle costing.** The metering engine reserves estimated budget upfront, preventing concurrent requests from collectively overshooting caps before their actual costs are known.