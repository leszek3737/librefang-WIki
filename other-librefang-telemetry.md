# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry + Prometheus metrics instrumentation for LibreFang.

## Overview

`librefang-telemetry` is a utility crate that provides observability primitives for the LibreFang project. It builds on the [`metrics`](https://docs.rs/metrics) facade to define domain-specific counters, gauges, and histograms that other crates in the workspace record during runtime.

## Dependencies

| Dependency | Purpose |
|---|---|
| `metrics` (workspace) | The `metrics` facade crate — provides the `counter!`, `gauge!`, `histogram!`, and `describe_*!` macros used to define and emit metrics. |
| `librefang-types` | Shared domain types. Telemetry labels and metric dimensions reference types defined here (e.g., game identifiers, player states). |

## Role in the Workspace

This crate sits alongside `librefang-types` as a leaf dependency: other workspace crates depend on it, but it does not depend on any application or server crate. The dependency direction is:

```
librefang-server  ──►  librefang-telemetry  ──►  librefang-types
librefang-game    ──►  librefang-telemetry  ──►  librefang-types
       ...                (records metrics)        (shared types)
```

Other crates call into `librefang-telemetry` at key points in their execution (connection handling, game events, errors) to record measurements. This crate itself makes no outgoing calls to other workspace modules.

## Usage Pattern

Consumers import this crate and call its metric-registration functions at startup (to describe and initialize metrics), then use the `metrics` macros or helper functions throughout their code paths to emit data points:

```rust
// At application startup — registers and describes metrics
librefang_telemetry::install();

// During runtime — record a measurement
counter!("librefang_connections_total", "protocol" => "tcp").increment(1);
```

## Notes

- This crate does **not** configure an exporter. The choice of Prometheus endpoint, OpenTelemetry push/pull, or console output is left to the final binary crate, which pulls in the appropriate `metrics-exporter-*` implementation.
- Because the `metrics` crate uses a global receiver, calls made before an exporter is installed are no-ops — tests and benchmarks can run without any telemetry backend.
