# Other — librefang-hands

# librefang-hands

Curated autonomous capability packages for LibreFang.

## Overview

In LibreFang, a **Hand** is a discrete, self-contained capability package that defines something the system can autonomously do. Hands are the primary extension mechanism — each hand encapsulates a specific skill, behavior, or set of actions that can be loaded, managed, and invoked by the runtime.

This crate provides the data structures, loading logic, and registry for defining and managing hands throughout their lifecycle.

## Purpose

The hands system serves as the bridge between declarative capability definitions and executable behavior. Rather than baking all functionality into the core runtime, LibreFang delegates to hands, which are:

- **Curated** — each hand has a manifest describing its identity, dependencies, and capabilities
- **Autonomous** — hands operate independently once invoked, managing their own state
- **Composable** — hands can be combined and orchestrated by the runtime

## Key Dependencies

The crate's dependency profile reveals its responsibilities:

| Dependency | Role |
|---|---|
| `librefang-types` | Shared type definitions (hand IDs, capability descriptors, etc.) |
| `serde`, `serde_json`, `toml` | Serialization of hand manifests and configuration |
| `thiserror` | Typed error definitions for load and validation failures |
| `tracing` | Structured logging during hand registration and lifecycle events |
| `uuid` | Unique hand instance identification |
| `chrono` | Timestamp tracking for hand registration and execution metadata |
| `dashmap` | Concurrent hand registry — supports thread-safe registration and lookup |

## Architecture

```
┌─────────────────────────────────────────┐
│           Hand Manifest (TOML)          │
│  id, version, capabilities, config      │
└──────────────────┬──────────────────────┘
                   │ load / parse
                   ▼
┌─────────────────────────────────────────┐
│           Hand Registry                 │
│  (dashmap-backed concurrent store)      │
│  register · lookup · enumerate · revoke │
└──────────────────┬──────────────────────┘
                   │ query
                   ▼
┌─────────────────────────────────────────┐
│          librefang-runtime              │
│  (consumer — test dependency only)      │
└─────────────────────────────────────────┘
```

The registry is the central component. It maintains a thread-safe map of loaded hands, keyed by their identifiers, and provides operations for the hand lifecycle.

## Hand Manifests

Hands are defined declaratively via TOML manifests. The manifest describes:

- **Identity** — a unique ID and version for the hand
- **Capabilities** — what the hand can do, expressed as typed descriptors from `librefang-types`
- **Configuration schema** — accepted parameters and their defaults

The `toml` and `serde` dependencies handle parsing these manifests into typed Rust structures, with `serde_json` available for any JSON-formatted configuration embedded within.

## Error Handling

Errors are defined using `thiserror` and cover the hand lifecycle:

- **Parse errors** — malformed or missing manifest fields
- **Validation errors** — invalid capability declarations or version conflicts
- **Registry errors** — duplicate registration or missing hand lookup

## Testing

The crate uses `serial_test` to coordinate tests that share the global registry state, and `tempfile` to create isolated filesystem environments for manifest loading tests. `librefang-runtime` is included as a dev-dependency for integration tests that verify hand invocation through the full runtime stack.

## Relationship to Other Crates

- **`librefang-types`** — provides the shared vocabulary (hand IDs, capability types) that this crate consumes
- **`librefang-runtime`** — consumes this crate at runtime to discover and invoke hands; only a dev-dependency here to avoid circular coupling