# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry + Prometheus metrics instrumentation for the LibreFang project.

## Overview

`librefang-telemetry` serves as the centralized metrics layer for LibreFang. It wraps the [`metrics`](https://docs.rs/metrics) facade crate and exposes domain-specific metric definitions, helpers, and labels tailored to the server's operational concerns — game sessions, player activity, network throughput, and system health.

By isolating all metric registration and labeling into its own crate, the rest of the codebase can record observations without worrying about cardinality management, naming conventions, or exporter configuration.

## Dependencies

| Dependency | Purpose |
|---|---|
| `metrics` | Vendor-agnostic metrics facade. Provides macros like `counter!`, `gauge!`, `histogram!`, and `increment_counter!`. |
| `librefang-types` | Shared domain types used in metric labels (e.g., game mode identifiers, player states). |
| `tokio-test` (dev) | Async test utilities for validating metric behavior under concurrent scenarios. |

## Role in the Architecture

```
┌──────────────────┐
│  Application     │
│  Binary / Server │
│                  │
│  metrics::*      │  ← macros called throughout business logic
└────────┬─────────┘
         │ depends on
         ▼
┌──────────────────┐       ┌──────────────────┐
│ librefang-       │──────▶│ librefang-types  │
│ telemetry        │       │                  │
│                  │       │ Shared types for │
│ Metric names,    │       │ labels & keys    │
│ label constants, │       └──────────────────┘
│ helper functions │
└──────────────────┘
```

The binary crate wires up a concrete metrics exporter (e.g., `metrics-exporter-prometheus`) at startup. `librefang-telemetry` does **not** select or configure an exporter — it only defines *what* to measure and *how* to label it. This separation keeps the telemetry definitions portable across different observability backends.

## Key Concepts

### Naming Conventions

Metric names follow a dotted namespace to avoid collisions and group related observations:

- `librefang.session.*` — game session lifecycle (created, destroyed, active count)
- `librefang.player.*` — player connection metrics (joins, leaves, concurrent)
- `librefang.network.*` — bytes sent/received, message latency
- `librefang.system.*` — process-level health (CPU, memory, uptime)

All name constants are centralized here so that typos or renaming only need to happen in one place.

### Label Design

Labels are derived from types in `librefang-types`. Keeping label values bounded is critical for Prometheus — unbounded cardinality (e.g., using a raw player ID as a label) would explode metric storage. This crate is responsible for defining which label dimensions are safe to use and providing helpers that map domain types to label sets.

## Integration Points

Other LibreFang crates consume this module by:

1. **Depending on it** in their `Cargo.toml`.
2. **Calling metric helpers** or using the re-exported `metrics` macros with the names/labels defined here.
3. **Never importing `metrics` directly** — going through this crate ensures consistent naming and labeling.

The application binary is responsible for:

1. Initializing a metrics exporter before any observations are recorded.
2. Optionally installing a `metrics::Recorder` that routes to OpenTelemetry, Prometheus, or both.
3. Calling into this crate's setup functions to register any default gauge or counter baseline values.

## Testing

The `tokio-test` dev dependency indicates that tests exercise metric recording under async contexts — validating that counters and histograms behave correctly when multiple tasks record concurrently, and that label resolution from `librefang-types` values produces the expected keys.