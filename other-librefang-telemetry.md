# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry and Prometheus metrics instrumentation for LibreFang.

## Overview

`librefang-telemetry` provides the centralized metrics infrastructure for the LibreFang project. It wraps the [`metrics`](https://docs.rs/metrics) facade crate and exposes a consistent instrumentation layer that other crates in the workspace consume to record operational telemetry — request counts, response latencies, error rates, and any custom counters or histograms relevant to the gateway's behavior.

## Dependencies

| Dependency | Purpose |
|---|---|
| `metrics` (workspace) | Vendor-agnostic metrics facade. Provides macros like `counter!`, `histogram!`, `gauge!` that this crate builds upon. |
| `librefang-types` | Shared type definitions used across the LibreFang workspace. Allows telemetry helpers to accept domain types directly rather than raw primitives. |

## Role in the Architecture

This crate is a **leaf dependency** — it does not call into other LibreFang crates at runtime, nor is it called by them through a traditional function call graph. Instead, other crates reference it at compile time to gain access to metrics definitions, labels, and helper functions. The actual metrics sink (Prometheus exporter, OpenTelemetry OTLP endpoint, etc.) is configured at the application boundary, not inside this library.

```
┌──────────────────────┐
│  Application Binary  │  ← sets up metrics recorder/exporter
└──────────┬───────────┘
           │ depends on
           ▼
┌──────────────────────┐
│ librefang-telemetry  │  ← defines metrics helpers & labels
└──────────┬───────────┘
           │ depends on
           ▼
┌──────────────────────┐
│  librefang-types     │  ← shared domain types
└──────────────────────┘
```

## What This Crate Likely Provides

Based on its dependency footprint and stated purpose, `librefang-telemetry` is responsible for:

- **Metric name constants** — centralized strings or identifiers for counters, histograms, and gauges so that consumers don't duplicate or misspell them.
- **Label helpers** — functions or constructors that turn `librefang-types` values into `metrics` label key-value pairs (e.g., converting a `Method` enum into a `"method"` label).
- **Recording wrappers** — optional convenience functions that combine a metric recording with domain logic, reducing boilerplate in hot paths.

## Usage Pattern

Other crates instrument their code by depending on `librefang-telemetry` and calling into the `metrics` facade:

```rust
use metrics::counter;

// Increment a request counter with labels defined by this crate
counter!("librefang_requests_total", "method" => "GET", "status" => "200").increment(1);
```

The actual exporter (Prometheus, OTLP, etc.) is initialized in the final binary, not in this library. This separation keeps instrumentation independent of the observability backend.

## Adding New Metrics

When adding telemetry for a new feature:

1. Define the metric name as a constant in this crate to avoid duplication.
2. If the metric requires labels derived from domain types, add a helper function here that accepts the relevant `librefang-types` and returns the appropriate label set.
3. Use the `metrics` facade macros in the consuming crate to record values.
4. Document the new metric's name, type (counter / histogram / gauge), labels, and unit in this crate's module-level docs or a dedicated `METRICS.md` if the list grows large.