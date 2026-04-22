# Other — librefang-kernel-metering

# librefang-kernel-metering

Cost metering and quota enforcement for the LibreFang kernel.

## Purpose

This module is responsible for tracking resource consumption during kernel operations and enforcing quota limits. It provides the accounting layer that ensures callers cannot exceed their allocated resource budgets—whether those resources are CPU time, memory allocations, or other finite kernel resources.

Metering sits between the execution of an operation and its completion: every significant resource-consuming action is recorded, and quotas are checked before expensive operations proceed.

## Dependencies

| Dependency | Role in This Module |
|---|---|
| `librefang-types` | Shared types for representing costs, quotas, and metering identifiers |
| `librefang-memory` | Memory allocation tracking and accounting |
| `librefang-runtime` | Access to runtime context for correlating metering data with executing tasks |
| `serde` | Serialization of metering records for persistence, logging, or network transmission |

## Architectural Role

```
┌─────────────────────┐
│   Calling Module    │
└─────────┬───────────┘
          │ check quota / record cost
          ▼
┌─────────────────────────────┐
│  librefang-kernel-metering  │
│                             │
│  • Quota enforcement        │
│  • Cost accumulation        │
│  • Usage reporting          │
└─────────┬───────────────────┘
          │ reads / writes
          ▼
┌─────────────────────────────┐
│  librefang-memory           │
│  (allocation accounting)    │
└─────────────────────────────┘
```

The metering module is a cross-cutting concern. Other kernel components query it before performing expensive operations, and report costs back to it after operations complete. It does not drive execution itself—it is consulted by callers.

## Key Concepts

### Cost

A quantified measure of resource consumption. Costs are associated with specific operations or task contexts and accumulate over the lifetime of a task or session.

### Quota

A predefined upper bound on allowed resource consumption. Quotas are checked before operations proceed. When a quota would be exceeded, the operation is rejected rather than allowed to fail partway through.

### Metering Context

The runtime context against which costs are accumulated. Typically tied to a task, session, or tenant identity so that enforcement is scoped correctly.

## Integration Notes

When adding a new resource-consuming operation to the kernel:

1. **Before the operation**: Query the metering module to check whether the projected cost fits within the active quota. Abort early if it does not.
2. **After the operation**: Report the actual cost incurred. This keeps the metering ledger accurate even when actual costs differ from projections.
3. **On task/session teardown**: Metering data is available for final reporting or serialization via `serde`.

When defining a new resource type, extend the cost types in `librefang-types` so that metering can represent and track the new dimension.

## Caution

Because metering is invoked on hot paths, operations within this module should remain lightweight. Avoid allocations within metering logic itself—use pre-sized buffers or borrow from the caller where possible. Quota checks in particular must be fast, as they gate every metered operation.