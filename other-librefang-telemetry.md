# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry + Prometheus metrics instrumentation for LibreFang.

## Purpose

`librefang-telemetry` provides the centralized metrics layer for the LibreFang project. It wraps the `metrics` facade crate and pairs it with LibreFang-specific domain types from `librefang-types`, giving other workspace crates a single, consistent dependency for emitting telemetry data.

Rather than each binary or library importing and configuring metrics independently, this crate acts as the one place that defines what gets measured and how those measurements are labeled.

## Role in the Workspace

```
┌──────────────────┐
│  librefang-types │
└────────┬─────────┘
         │  depends on
┌────────▼─────────┐
│librefang-telemetry│
└────────┬─────────┘
         │  used by
    ┌────┴────┐
    ▼         ▼
 binaries  libraries
```

Downstream crates depend on `librefang-telemetry` to:

- **Record metrics** without coupling to a specific metrics backend.
- **Use domain-relevant label types** (from `librefang-types`) rather than raw strings, keeping telemetry consistent across the codebase.

The crate itself has no incoming or outgoing runtime call edges—it is a pure utility layer. Other modules call into it at their own discretion.

## Dependencies

| Dependency | Source | Purpose |
|---|---|---|
| `metrics` | Workspace | Generic metrics facade (counters, gauges, histograms). The actual exporter (e.g., Prometheus endpoint) is configured at the binary level, not here. |
| `librefang-types` | Workspace path `../librefang-types` | Shared domain types used as metric labels or keys, ensuring telemetry labels stay type-safe and consistent. |

## Design Decisions

**Backend-agnostic facade.** By depending on the `metrics` crate rather than a concrete exporter, this module remains decoupled from any specific observability stack. Binaries wire up the exporter they need (Prometheus, OpenTelemetry OTLP, etc.) at startup; library code only calls into this crate.

**No runtime graph edges.** The call graph shows no internal, outgoing, or incoming calls because this module exposes static helpers—metric definitions, label constructors, and thin wrappers. There is no stateful runtime or background thread owned by this crate.

## Integration Guide

To use telemetry from another workspace crate, add the dependency:

```toml
[dependencies]
librefang-telemetry = { path = "../librefang-telemetry" }
```

Then import and use the provided helpers wherever you need to emit metrics. The actual metric recording delegates to the `metrics` facade, so you must install an exporter in your binary's `main` for the data to go anywhere.

## Linting

This crate inherits workspace-level lints via the `[lints] workspace = true` configuration, ensuring it follows the same code quality standards as the rest of the project.