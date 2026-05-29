# Other — librefang-rl-export

# librefang-rl-export

Long-horizon reinforcement learning rollout trajectory exporter. Provides the integration surface for exporting RL rollout data to external tracking and evaluation systems — W&B (Weights & Biases), Tinker, and Atropos.

## Purpose

This module handles the export pipeline for trajectory data generated during long-horizon RL rollouts. It sits at the boundary between the core RL engine and external services, responsible for serializing, encoding, and transmitting rollout trajectories over HTTP.

## Architecture

The module depends on `librefang-http` as its internal HTTP transport layer and builds export-specific logic on top of it. It does not consume trajectory data directly from the RL engine — it exposes an integration surface that other modules push data into for export.

```
┌─────────────────┐      ┌──────────────────────┐      ┌───────────────────┐
│  RL Engine      │─────▶│  librefang-rl-export │─────▶│  External Services│
│  (rollout data) │      │  (serialize + send)  │      │  W&B / Tinker /   │
└─────────────────┘      └──────────────────────┘      │  Atropos          │
                                │                      └───────────────────┘
                                ▼
                         ┌──────────────┐
                         │ librefang-   │
                         │ http         │
                         └──────────────┘
```

## Key Dependencies and Their Roles

| Dependency | Role |
|---|---|
| `librefang-http` | Internal HTTP client abstraction; handles request construction and response handling |
| `reqwest` | Underlying HTTP client used for outbound requests to external services |
| `serde` / `serde_json` | Serialization of trajectory data structures to JSON |
| `base64` | Encoding binary or large trajectory payloads for transport |
| `urlencoding` | Encoding parameters embedded in URL query strings |
| `url` | Constructing and manipulating export endpoint URLs |
| `regex` | Pattern matching for payload validation or URL template expansion |
| `chrono` | Timestamp generation for exported trajectory records |
| `thiserror` | Typed error definitions for export failures |
| `tracing` | Structured logging of export operations and failures |

## Integration Surface

The module targets three categories of external systems:

- **W&B (Weights & Biases)**: Experiment tracking. Trajectory metrics and summaries are logged as W&B runs or artifacts.
- **Tinker**: Visualization or analysis tooling. Rollout trajectories are exported in a format Tinker can consume.
- **Atropos**: RL infrastructure component. Trajectory data is shipped for further evaluation or distributed processing.

Each integration may require different payload formats, authentication schemes, and endpoint configurations. The `url`, `urlencoding`, and `base64` dependencies suggest that endpoint construction and payload encoding vary per target.

## Error Handling

Export failures (network errors, serialization failures, rejected payloads) use `thiserror`-derived error types, allowing callers to distinguish between transient transport errors and permanent data errors. All export operations should be instrumented with `tracing` spans for observability.

## Testing

Tests use `wiremock` to mock external HTTP endpoints, allowing verification of request shape, headers, authentication, and payload structure without hitting real services. Test configuration requires the `tokio` multi-thread runtime with macro support (specified in `[dev-dependencies]`).

When adding tests for a new integration target:

1. Set up a `wiremock::MockServer` in your test.
2. Define the expected request matcher (method, path, headers, body pattern).
3. Call the export function against the mock server's URI.
4. Assert the mock received the expected request.

## Relationship to the Wider Codebase

This module is a leaf dependency — it depends on `librefang-http` but nothing else in the workspace depends on it. Modules producing rollout data import `librefang-rl-export` directly when they need to ship trajectories externally. The module has no incoming calls detected at the static analysis level, meaning it is called at runtime by consumer modules rather than being wired through a central dependency graph.