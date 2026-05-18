# Other — librefang-kernel-metering

# librefang-kernel-metering

Cost metering and quota enforcement for the LibreFang kernel.

## Purpose

This module tracks resource consumption and enforces spending limits across LLM operations. It acts as the kernel's financial control layer — every token processed through the system is accounted for, and quotas are checked before operations are dispatched.

## Role in the Architecture

Metering sits between the runtime and the LLM driver layer. When the runtime prepares to execute an LLM call, metering checks whether the operation is within budget. After the call completes, metering records the actual cost against the relevant account or session.

```
┌──────────────┐
│   Runtime    │
└──────┬───────┘
       │ pre-check / post-record
┌──────▼───────┐
│   Metering   │
└──────┬───────┘
       │
┌──────▼───────┐
│  LLM Driver  │
└──────────────┘
```

## Dependencies

| Dependency | Reason |
|---|---|
| `librefang-types` | Shared cost and quota types (price tables, usage counters, budget descriptors) |
| `librefang-memory` | Persistent storage for usage records and quota state |
| `librefang-runtime` | Hooks into the kernel execution loop for pre/post operation checks |
| `librefang-llm-driver` | Model pricing metadata and token count reporting from LLM responses |
| `serde` | Serialization of metering records for persistence and auditing |
| `tracing` | Observability — logs quota breaches, spending milestones, and enforcement decisions |

## Key Concepts

### Cost Metering

Each LLM response carries token usage data. Metering extracts the input and output token counts, applies the model's per-token price, and accumulates the cost. This produces a running total tied to whatever granularity the kernel requires — per-session, per-user, per-agent, or global.

### Quota Enforcement

Quotas are checked before an LLM call is issued. If the remaining budget cannot cover the estimated cost of the operation, the call is rejected before any tokens are sent to the provider. This prevents overshoot and ensures hard spending caps are respected.

Enforcement is optimistic for variable-cost operations — output tokens cannot be known in advance, so the check is made against a worst-case estimate or a configured max-output-tokens parameter.

### State Persistence

Metering state must survive kernel restarts. Usage counters and quota balances are persisted through `librefang-memory`, which provides the storage backend. Serialized metering snapshots are written on every update or at configured intervals.

## Integration Points

**Upstream consumers** — the runtime calls into metering before and after each LLM invocation.

**Downstream data** — metering reads pricing information from `librefang-llm-driver` (model-specific cost per token) and reads/writes quota state through `librefang-memory`.

**Observability** — all enforcement decisions and cost accumulations are emitted as `tracing` events, allowing external systems to monitor spending in real time.