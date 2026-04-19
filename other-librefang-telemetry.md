# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry + Prometheus metrics instrumentation for LibreFang.

## Purpose

This crate serves as the centralized metrics and observability layer for the LibreFang project. It provides shared metric definitions, helper functions, and instrumentation utilities that other crates consume to emit telemetry data. By isolating telemetry concerns into a dedicated library, all components in the workspace report metrics through a consistent interface and naming scheme.

## Dependencies

| Dependency | Purpose |
|---|---|
| `metrics` (workspace) | The core `metrics` facade crate — provides the `counter!`, `gauge!`, `histogram!`, and `increment_counter!` macros used throughout the project. This crate is neutral regarding the backend exporter. |
| `librefang-types` | Shared domain types (game state enums, player identifiers, etc.) used as labels or keys in metric instrumentation. |
| `tokio-test` (dev) | Async test utilities used in unit tests within this crate. |

## Design

The crate follows the `metrics` crate's **facade pattern**: it depends only on the `metrics` API, not on any specific exporter. The actual metrics backend (e.g., Prometheus via `metrics-exporter-prometheus`) is configured at the binary/application level, not in this library. This means:

- **No global state is initialized here.** The exporter is wired up by the top-level binary crate.
- **Only definitions and helpers live here.** Metric names, label constants, and instrumentation wrappers that other crates call.

## How It Connects to the Codebase

```
┌──────────────────────────┐
│  Application Binary      │
│  (wires up Prometheus    │
│   exporter, starts server)│
└───────────┬──────────────┘
            │ depends on
            ▼
┌──────────────────────────┐
│  librefang-telemetry     │◄──── other library crates import
│  (metric definitions,    │      helpers to emit metrics
│   label constants,       │
│   instrumentation utils) │
└───────────┬──────────────┘
            │ depends on
            ▼
┌──────────────────────────┐
│  librefang-types         │
│  (shared domain types    │
│   used as metric labels) │
└──────────────────────────┘
```

Library crates such as `librefang-server` or game logic modules depend on `librefang-telemetry` to record events (e.g., connections, game rounds, errors) without coupling to any specific observability backend.

## Adding New Metrics

When instrumenting a new subsystem:

1. **Define metric name constants** in this crate so naming is centralized and discoverable.
2. **Define label key constants** to avoid string duplication across call sites.
3. **Use the `metrics` macros** (`counter!`, `gauge!`, `histogram!`) in helper functions exposed from this crate, or re-export the macros for direct use by consumers.
4. **Reference types from `librefang-types`** for label values when the label represents a domain concept (e.g., game phase, player role).

## Testing

Tests use `tokio-test` for async test scenarios. Because the `metrics` facade is a no-op when no exporter is installed, tests in this crate typically verify that instrumentation helpers compile, accept the correct types, and exercise label construction logic — without needing a live metrics sink.