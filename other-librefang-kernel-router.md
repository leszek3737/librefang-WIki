# Other — librefang-kernel-router

# librefang-kernel-router

Hand and template routing engine for the LibreFang kernel.

## Purpose

This module is responsible for resolving incoming requests to the correct **Hand** implementation based on configurable routing rules. It acts as the dispatch layer between the LibreFang kernel's input sources and the hand implementations that process them.

Routing decisions can be based on pattern matching (via regular expressions), template resolution, and TOML-based configuration. The router determines *which* hand should handle a given input and *how* that hand should be instantiated or parameterized.

## Role in the Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  Incoming        │────▶│  kernel-router       │────▶│  librefang-hands │
│  Requests/Input  │     │  (match & dispatch)  │     │  (Hand impls)    │
└─────────────────┘     └──────────────────────┘     └──────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  librefang-types  │
                        │  (shared types)   │
                        └──────────────────┘
```

The router sits between the runtime's input layer and the hand implementations. When the kernel receives input, the router evaluates its routing table and dispatches to the matching hand.

## Key Dependencies

| Dependency | Purpose |
|---|---|
| `librefang-types` | Shared type definitions used across the kernel — request types, routing results, error types |
| `librefang-hands` | Hand trait definitions and registry. The router resolves against these hand implementations |
| `regex-lite` | Lightweight regular expression matching for pattern-based route rules |
| `serde_json` | Deserializing route configuration or template parameters from JSON |
| `toml` | Parsing routing table configuration from TOML files |
| `dirs` | Resolving standard system directories to locate configuration files |
| `tracing` | Structured logging of routing decisions for debugging and observability |

## How Routing Works

1. **Configuration Loading** — The router reads a routing table from TOML configuration, typically located via platform-standard directories resolved through the `dirs` crate.

2. **Pattern Matching** — Each route entry contains a pattern (potentially a regular expression). When a request arrives, the router evaluates patterns against the input to find a match.

3. **Template Resolution** — Matched routes may reference templates that parameterize how a hand is invoked. `serde_json` handles any structured data involved in template parameter binding.

4. **Dispatch** — The router returns a reference or descriptor identifying which hand from `librefang-hands` should handle the request, along with any extracted parameters from the match.

## Configuration

Routing rules are defined in TOML. The `dirs` crate locates the configuration root, and `toml` parses the routing table. This allows administrators and developers to modify routing behavior without recompiling the kernel.

## Testing

The dev-dependency on `librefang-runtime` indicates that integration tests exercise the router within the full runtime context, verifying end-to-end dispatch behavior. The `tempfile` crate supports test cases that create temporary configuration files to test different routing scenarios in isolation.

## Integration Points

- **Consumed by**: `librefang-runtime` (and potentially the kernel core) to dispatch incoming requests.
- **Depends on**: `librefang-types` for shared types, `librefang-hands` for hand definitions and registration.
- **Configuration**: External TOML files, located via platform conventions.