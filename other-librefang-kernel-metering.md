# Other — librefang-kernel-metering

# librefang-kernel-metering

## Purpose

Cost metering and quota enforcement for the LibreFang kernel. This module tracks resource consumption—primarily LLM token usage and associated costs—and enforces quotas to prevent runaway spending or resource exhaustion.

## Dependencies & What They Imply

```toml
librefang-types    # Shared type definitions used across the kernel
librefang-memory   # Memory management — likely for storing metering state
librefang-runtime  # Runtime infrastructure — lifecycle, async runtime
librefang-llm-driver  # LLM driver — the primary source of billable operations
serde              # Serialization — metering data is persisted or transmitted
tracing            # Structured logging — audit trails for cost events
```

The dependency on `librefang-llm-driver` is the key signal: this module wraps or intercepts LLM operations to measure them. The `serde` dependency suggests metering state is serialized, likely for persistence across sessions or for reporting. The `tracing` dependency indicates that cost events are logged for observability.

## Integration Points

Based on the dependency graph, this module sits between the LLM driver and the rest of the kernel:

```
┌─────────────────┐
│ librefang-llm-  │
│     driver      │
└────────┬────────┘
         │ LLM calls produce
         │ billable events
         ▼
┌─────────────────┐
│ librefang-      │
│ kernel-metering │◄── quota configuration
└────────┬────────┘
         │ enforce / block
         ▼
┌─────────────────┐
│ librefang-      │
│    runtime      │
└─────────────────┘
```

The module consumes telemetry or hook points from the LLM driver, accumulates usage, checks against configured limits, and signals back when quotas are exceeded.

## Expected Responsibilities

Given its stated purpose and dependencies, this module is responsible for:

- **Token counting**: Tracking prompt and completion tokens per request.
- **Cost accumulation**: Converting token counts to monetary cost using model-specific pricing.
- **Quota enforcement**: Rejecting or throttling requests when a budget or token limit is reached.
- **State persistence**: Serializing metering data so it survives restarts (via `serde` and `librefang-memory`).
- **Audit logging**: Emitting structured trace events for every metered operation.

## Status

No execution flows or call edges were detected in the static analysis. This indicates the module is either newly scaffolded, contains only type definitions and configuration structs without active call paths, or its interactions are dispatched indirectly (e.g., through trait objects or dynamic dispatch) that the analysis did not resolve.

When adding to or consuming this module, inspect the actual source files under `librefang-kernel-metering/src/` for the concrete API surface.