# Other — librefang-hands

# librefang-hands

Curated autonomous capability packages for LibreFang.

## Purpose

A **Hand** in LibreFang is a self-contained, curated capability package — a declarative unit of autonomous functionality that can be loaded, validated, and managed by the runtime. This crate defines the Hand data model, handles parsing and serialization of hand definitions (TOML/JSON), and provides a thread-safe registry for tracking available hands.

Think of a hand as a blueprint: it describes *what* a capability does, *what* it requires, and *how* it should be configured, without embedding the execution logic itself. Execution is handled by the runtime layer.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                librefang-hands                   │
│                                                  │
│  ┌──────────┐  ┌────────────┐  ┌─────────────┐  │
│  │ Hand     │  │ Hand       │  │ Hand        │  │
│  │ Defini-  │  │ Registry   │  │ Loading &   │  │
│  │ tion     │  │ (DashMap)  │  │ Validation  │  │
│  └──────────┘  └────────────┘  └─────────────┘  │
│                                                  │
│  Error types · Serialization helpers             │
└──────────────────────┬──────────────────────────┘
                       │ depends on
              ┌────────┴─────────┐
              │ librefang-types  │
              └──────────────────┘
```

## Key Concepts

### Hand Definition

A hand is defined declaratively, typically authored in TOML for human readability and serialized to JSON for machine consumption. Each hand carries:

- **Identity** — A unique identifier (UUID) and a human-readable name.
- **Metadata** — Version, description, authorship, and timestamps (`chrono`).
- **Capability specification** — What this hand provides to the autonomous system.
- **Requirements** — Dependencies or conditions the hand expects to be satisfied.

### Hand Registry

The registry is backed by `DashMap`, a concurrent hashmap, allowing multiple threads to query and update the set of available hands without external locking. This is important because LibreFang's runtime may load or reload hands while other components are reading from the registry.

### Serialization Formats

| Format | Purpose |
|--------|---------|
| TOML   | Authoring — hand definition files written by developers |
| JSON   | Transport — machine-readable interchange between crates |

Both are supported via `serde` with derived `Serialize`/`Deserialize` implementations on hand data structures.

## Dependencies

| Dependency | Role in this crate |
|------------|-------------------|
| `librefang-types` | Shared type definitions used across all LibreFang crates |
| `serde` / `serde_json` / `toml` | Declarative parsing and serialization of hand definitions |
| `thiserror` | Typed error definitions for loading and validation failures |
| `tracing` | Structured logging of hand lifecycle events |
| `uuid` | Unique identification of hand instances |
| `chrono` | Timestamps for creation and modification tracking |
| `dashmap` | Lock-free concurrent registry storage |

## Testing

Tests use `tempfile` for creating isolated hand definition files, `librefang-runtime` for integration scenarios, and `serial_test` to serialize tests that share global state. Tests validate parsing, validation rules, registry operations, and serialization round-trips.

## Relationship to Other Crates

- **`librefang-types`** — Foundational types that hands reference (enums, structs shared across the system).
- **`librefang-runtime`** — Consumes hands from the registry and executes their declared capabilities. Listed as a dev-dependency here for integration testing only; at runtime, the dependency flows the other direction.