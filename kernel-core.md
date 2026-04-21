# Kernel Core

# Kernel Core

The central runtime and supporting services for the LibreFang Agent Operating System. The kernel orchestrates agent lifecycles from instantiation through shutdown, while providing the routing, metering, and interface layers that make multi-agent coordination possible.

## Sub-modules at a glance

| Sub-module | Responsibility |
|---|---|
| [Kernel Core (runtime)](librefang-kernel-src.md) | Agent lifecycles, configuration, scheduling, event bus, process supervision, and background execution |
| [Metering](librefang-kernel-metering-src.md) | LLM cost tracking and quota enforcement across per-agent, global, and per-provider budgets |
| [Router](librefang-kernel-router-src.md) | Request dispatch — selects which agent template or hand should handle an incoming message |
| [Handle](librefang-kernel-handle-src.md) | Dependency-inversion trait (`KernelHandle`) that lets agents call back into the kernel without circular imports |

## How they fit together

```
User message
    │
    ▼
┌──────────┐   selects agent    ┌─────────────────┐
│  Router   │ ─────────────────▶│  AgentRegistry   │
└──────────┘                    └────────┬─────────┘
                                         │ spawns / supervises
                                         ▼
                                ┌─────────────────┐
                                │  Agent Instance  │──┐
                                └────────┬─────────┘  │
                                    │    │            │
                          calls via │    │ LLM call   │
                       KernelHandle │    │ completes  │
                                    ▼    ▼            │
                              ┌──────────────┐        │
                              │   Metering    │◀───────┘
                              └──────────────┘  records cost &
                                                  enforces quotas
```

The **Router** is the entry point for every user message. It scores candidate templates and hands using keyword matching, manifest metadata, and optional semantic embeddings, then hands the selected agent to the kernel runtime.

The **Kernel runtime** owns the resulting agent lifecycle. Its `AgentRegistry` tracks live agents, the `AgentScheduler` enforces resource quotas, the `EventBus` handles inter-agent communication via broadcast and per-agent channels, and the `Supervisor` manages shutdown and restart tracking. The `BackgroundExecutor` drives continuous, periodic, and proactive tasks.

The **Handle** breaks what would otherwise be a circular import: the kernel owns the runtime, but agents need to reach back into the kernel to spawn children, read shared memory, post tasks, and request human approval. `KernelHandle` is an `#[async_trait]` that enumerates every such operation; the kernel implements it and injects it at agent startup.

The **Metering engine** sits at the LLM call boundary. When an agent completes an LLM call, `check_all_and_record` validates spending against three independent budget layers—per-agent, global, and per-provider—inside a single SQLite transaction. If any layer is exceeded, the call is rejected with a `QuotaExceeded` error and no usage is recorded.

## Key cross-cutting workflows

- **Request handling:** Router → Kernel (registry + scheduler) → Agent instance → Metering (cost recorded)
- **Agent callback:** Agent → KernelHandle → Kernel (spawn child, shared memory, human approval, etc.)
- **Budget enforcement:** Metering intercepts every LLM call, checks all three budget layers atomically, and blocks over-budget calls before they reach the provider