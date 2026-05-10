# Other — librefang-telemetry

# librefang-telemetry

OpenTelemetry and Prometheus metrics instrumentation for LibreFang.

## Purpose

This crate centralizes all metrics definitions and telemetry helpers used across the LibreFang project. By isolating telemetry into its own crate, metric names, labels, and instrumentation patterns stay consistent and discoverable — any crate that needs to emit or observe metrics depends on `librefang-telemetry` rather than importing the raw `metrics` crate directly.

## Dependencies

| Crate | Why |
|---|---|
| `metrics` (workspace) | Core metrics façade — provides macros like `counter!`, `gauge!`, `histogram!`. This crate wraps or re-exports them with LibreFang-specific conventions. |
| `librefang-types` | Shared domain types. Metrics often need to derive label values from types like player IDs, game states, or room identifiers. |

## Architectural Role

`librefang-telemetry` is a **leaf dependency** — it does not call into any other LibreFang crates at runtime. Other crates throughout the workspace import it to emit metrics. This keeps the telemetry surface area small and prevents circular dependencies.

```
┌──────────────────┐
│ librefang-server │ ──┐
├──────────────────┤   │
│ librefang-game   │ ──┤    depends on
├──────────────────┤   ├───────────────────►  librefang-telemetry
│ librefang-irc    │ ──┤                         │
├──────────────────┤   │                         │ uses
│      ...        │ ──┘                         ▼
└──────────────────┘                      metrics (facade)
```

## Usage Patterns

Other crates interact with this module by calling into its definitions at instrumentation points — typically around network events, game actions, or resource lifecycle changes. The actual metric exporters (Prometheus scrape endpoint, OpenTelemetry push) are configured at the application boundary, not in this library.

### Registering metrics

Metrics are expected to be described or initialized through this crate so that label namespaces and naming conventions (e.g., `librefang_players_connected_total`) remain uniform. This avoids ad-hoc `counter!("my_counter")` calls scattered across the codebase.

### Consuming types for labels

Because `librefang-types` is a dependency, metric helpers can accept domain types directly and extract the relevant label values internally, keeping call sites clean.

## Workspace Integration

This crate is part of the LibreFang workspace. It shares workspace-level lint configuration and version numbering via `workspace = true` inheritance in `Cargo.toml`. To add it as a dependency in another workspace crate:

```toml
[dependencies]
librefang-telemetry = { path = "../librefang-telemetry" }
```

## Adding New Metrics

When adding a new metric:

1. **Define it here** — add the metric name constant and any label helpers to this crate.
2. **Use domain types** — accept `librefang-types` values where possible so label extraction stays centralized.
3. **Follow naming conventions** — use the `librefang_` prefix and snake_case to keep Prometheus output consistent.
4. **Document units** — note whether a metric is a count, a gauge in bytes/milliseconds, etc., so downstream dashboard authors know what to expect.