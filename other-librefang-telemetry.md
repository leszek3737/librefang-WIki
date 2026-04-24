# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry + Prometheus metrics instrumentation for LibreFang.

## Purpose

This crate serves as the central observability layer for the LibreFang system. It provides metrics definitions, labeling conventions, and instrumentation helpers so that other crates in the workspace can emit structured telemetry without directly depending on a specific metrics backend.

The crate abstracts the choice of metrics backend behind the [`metrics`](https://docs.rs/metrics) facade, meaning all recording goes through the `metrics` crate's macros and functions. The actual exporter (e.g., Prometheus via `metrics-exporter-prometheus`) is wired up at the application boundary, not in this library.

## Dependencies

| Dependency | Role |
|---|---|
| `metrics` | Facade crate providing counters, gauges, histograms, and the `increment_counter!` / `histogram!` / `gauge!` macros used throughout the workspace. |
| `librefang-types` | Shared domain types (game identifiers, player states, etc.) used as metric label values. |

## How It Fits in the Architecture

```
┌──────────────────────────────────────────────────┐
│                Application Binary                 │
│  (wires up metrics-exporter-prometheus, etc.)     │
└──────────────┬───────────────────────────────────┘
               │ uses
┌──────────────▼───────────────────────────────────┐
│           librefang-telemetry                     │
│  - Metric name constants                         │
│  - Label helpers built from librefang-types      │
│  - Convenience wrappers around metrics facade     │
└──────────────┬───────────────────────────────────┘
               │ depends on
┌──────────────▼───────────────────────────────────┐
│           librefang-types                         │
└──────────────────────────────────────────────────┘
```

Other crates in the workspace (game server, lobby service, etc.) depend on `librefang-telemetry` and call into it to record events. Because this crate has no incoming or outgoing call edges in the static analysis, its functions are called directly by application code rather than being part of a callback or plugin chain.

## Integration Guide

### Recording metrics from another crate

Add the dependency:

```toml
[dependencies]
librefang-telemetry = { path = "../librefang-telemetry" }
```

Call the provided metric helpers from game logic, connection handlers, or wherever instrumentation is needed. All calls ultimately delegate to the `metrics` facade, so no async runtime or exporter configuration is required at this layer.

### Exporting metrics at the application level

The binary crate that assembles the final server executable is responsible for installing a metrics recorder/exporter. This is not handled inside `librefang-telemetry` itself, keeping the library agnostic to the export destination.

## Testing

The `tokio-test` dev-dependency supports async unit tests. Because this crate primarily defines metric helpers and constants, tests verify label formatting, metric name correctness, and that recording calls don't panic—without requiring a live metrics sink.