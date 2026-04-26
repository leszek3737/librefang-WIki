# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry and Prometheus metrics instrumentation for LibreFang.

## Purpose

This crate provides a centralized metrics layer for the LibreFang system. It depends on the `metrics` facade crate, which offers a vendor-neutral API for recording counters, gauges, and histograms. Downstream crates consume this module to emit telemetry without coupling directly to a specific metrics backend.

## Dependencies

| Crate | Role |
|---|---|
| `metrics` | Facade for recording metrics. Actual exporters (Prometheus, OpenTelemetry) are configured at the binary level, not here. |
| `librefang-types` | Shared domain types used in metric labels and dimensions. |

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Binary / entry point               │
│  (installs a metrics exporter: Prometheus, OTel) │
└──────────────┬──────────────────────────────────┘
               │ registers exporter with metrics::recorder
               │
┌──────────────▼──────────────────────────────────┐
│           librefang-telemetry                    │
│  Re-exports metrics macros, defines label        │
│  conventions and helper wrappers                 │
└──────────────┬──────────────────────────────────┘
               │ used by
       ┌───────┴────────┐
       ▼                ▼
  other LibreFang   other LibreFang
  library crates    library crates
```

This module sits between the application's metrics exporter (configured at the binary level) and the library crates that emit measurements. It does not start a metrics server or configure an exporter itself.

## Usage

Library crates should depend on `librefang-telemetry` rather than `metrics` directly. This keeps label naming, dimension conventions, and any helper wrappers consistent across the codebase.

At the binary level, the application installs a concrete `metrics::Recorder` — for example, a Prometheus exporter that scrapes `/metrics` on a designated port. Once installed, all macro calls (`counter!`, `gauge!`, `histogram!`, etc.) flow through that recorder.

## Integration Notes

- **No execution flows or call graph edges** were detected for this module. It is a utility/facade crate consumed statically by other modules rather than participating in runtime call chains.
- The crate is intentionally thin. It exists to own the metrics dependency and any LibreFang-specific telemetry conventions, keeping them out of individual library crates.
- If you need to add new metric names or label dimensions, add them here so all consumers share a single source of truth.