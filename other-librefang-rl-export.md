# Other — librefang-rl-export

# librefang-rl-export

Long-horizon reinforcement learning rollout trajectory exporter, providing integration surfaces for Weights & Biases (W&B), Tinker, and Atropos.

## Purpose

This module is responsible for serializing and transmitting RL rollout trajectory data to external observability and analysis platforms. In a long-horizon RL training loop, rollouts generate sequences of states, actions, and rewards that need to be exported for visualization, experiment tracking, or downstream consumption. `librefang-rl-export` serves as that export layer.

## Position in the Codebase

```
┌─────────────────────────────┐
│   RL Training Pipeline      │
│  (rollout generation)       │
└─────────────┬───────────────┘
              │ trajectory data
              ▼
┌─────────────────────────────┐
│   librefang-rl-export       │
│  (serialize + transmit)     │
└─────────────┬───────────────┘
              │
              ▼
┌─────────────────────────────┐
│   librefang-http            │
│  (shared HTTP client)       │
└─────────────────────────────┘
```

The module depends on `librefang-http` for outbound HTTP communication rather than managing its own HTTP client, ensuring consistent connection pooling, retry logic, and authentication handling across the crate.

## Integration Surfaces

The module targets three external systems:

| Surface | Purpose |
|---------|---------|
| **W&B (Weights & Biases)** | Experiment tracking: metric logging, artifact storage, run visualization |
| **Tinker** | Interactive trajectory inspection and debugging |
| **Atropos** | Rollout data pipeline for distributed RL workloads |

## Key Dependencies and Their Roles

The dependency selections reveal the module's operational concerns:

- **`serde` / `serde_json`** — Trajectory data is serialized to JSON before transmission. Rollout records (states, actions, rewards, metadata) are represented as serializable types.
- **`reqwest`** — Direct HTTP client usage alongside `librefang-http`, likely for endpoints that require specific client configuration separate from the shared client.
- **`base64`** — Encoding binary or structured state data into a text-safe representation for inclusion in JSON payloads or URL parameters.
- **`urlencoding` / `url`** — Constructing and sanitizing URLs for API endpoints, including query parameters for filtering or pagination.
- **`regex`** — Pattern matching on trajectory data or response validation.
- **`chrono`** — Timestamping rollouts and handling time-based queries or windowed exports.
- **`thiserror`** — Typed error definitions for export failures (network errors, serialization issues, API rejections).
- **`tracing`** — Structured logging of export operations, useful for diagnosing transmission failures in long-running training jobs.

## Testing

Dev-dependencies include `wiremock` for HTTP mocking, indicating that tests mock external API endpoints to verify:

- Correct request construction (headers, body format, URL structure)
- Retry behavior on transient failures
- Error propagation when APIs return non-success status codes

Tests use `tokio` with the `macros` and `rt-multi-thread` features, confirming all export operations are fully asynchronous.