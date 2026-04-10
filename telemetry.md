# Telemetry

# Telemetry Module

The `librefang-telemetry` crate provides centralized telemetry instrumentation for the LibreFang Agent OS, exposing metrics and tracing signals that enable monitoring across all 14 workspace crates.

## Purpose

This module bridges the gap between application code and observability infrastructure by:

1. **Standardizing metric recording** — All HTTP request instrumentation flows through a single interface
2. **Reducing cardinality** — Path normalization prevents metric explosion from dynamic identifiers (UUIDs, hashes)
3. **Abstraction** — Code doesn't couple to a specific metrics backend; it uses the `metrics` crate which supports Prometheus, Datadog, or custom exporters

## Architecture

The telemetry crate acts as a thin instrumentation layer that delegates to the underlying `metrics` crate. The actual Prometheus exporter is installed in `librefang-api`, and this crate provides the recording utilities that feed into it.

```mermaid
graph LR
    A[HTTP Request] --> B[API Middleware]
    B --> C[record_http_request]
    C --> D[normalize_path]
    D --> E[is_dynamic_segment]
    E --> F[metrics crate]
    F --> G[Prometheus Handle]
    G --> H[/api/metrics endpoint]
```

## Module Structure

```
librefang-telemetry/
├── lib.rs      # Public re-exports
├── config.rs   # TelemetryConfig re-export
└── metrics.rs  # Core instrumentation functions
```

## Key Components

### Path Normalization

The `normalize_path` function collapses high-cardinality path segments into static labels:

| Input Path | Normalized Output |
|------------|-------------------|
| `/api/agents/550e8400-e29b-41d4-a716-446655440000/message` | `/api/agents/{id}/message` |
| `/api/agents/deadbeef01234567/status` | `/api/agents/{id}/status` |
| `/.well-known/agent.json` | `/.well-known/agent.json` |
| `/api/my-agent/status` | `/api/my-agent/status` |

Dynamic segment detection identifies:
- **UUIDs** — Standard 8-4-4-4-12 hex format (e.g., `550e8400-e29b-41d4-a716-446655440000`)
- **Hex strings** — 8–64 character pure hexadecimal strings without hyphens

Segments like `well-known`, `my-agent`, or `a2a` are preserved because they don't match these patterns.

### Recording HTTP Metrics

```rust
pub fn record_http_request(path: &str, method: &str, status: u16, duration: Duration)
```

This function is called by the request-logging middleware in `librefang-api`. It records two metrics:

1. **Counter** — `librefang_http_requests_total` with labels `method`, `path`, `status`
2. **Histogram** — `librefang_http_request_duration_seconds` with labels `method`, `path`

The normalized path prevents cardinality explosion. Without normalization, each unique UUID in a path would create a separate time series.

### Configuration

The `config` module re-exports `TelemetryConfig` from `librefang-types`:

```rust
pub use librefang_types::config::TelemetryConfig;
```

This keeps configuration centralized in the types crate while allowing imports from either location.

## Usage

### From Another Crate

```rust
use librefang_telemetry::{record_http_request, normalize_path};
use std::time::Duration;

// Normalize a path for internal use
let normalized = normalize_path("/api/agents/abc123/status");

// Record metrics (typically called by middleware, not directly)
record_http_request("/api/health", "GET", 200, Duration::from_millis(5));
```

### Middleware Integration

The primary consumer is the request-logging middleware in `librefang-api/src/middleware.rs`. The flow is:

1. Middleware intercepts HTTP request
2. Records start time
3. After response, calls `record_http_request(path, method, status_code, elapsed)`
4. Metrics are aggregated by the global recorder

## Metric Reference

| Metric Name | Type | Labels | Description |
|-------------|------|--------|-------------|
| `librefang_http_requests_total` | Counter | `method`, `path`, `status` | Total HTTP requests |
| `librefang_http_request_duration_seconds` | Histogram | `method`, `path` | Request latency |

## Design Notes

- **No global state in this crate** — The `metrics` crate handles the global recorder registration
- **Backward compatibility** — `get_http_metrics_summary()` exists for API compatibility but delegates to the Prometheus handle in `librefang-api`
- **No external dependencies** — Only depends on `metrics` and `librefang-types`, keeping the crate lightweight