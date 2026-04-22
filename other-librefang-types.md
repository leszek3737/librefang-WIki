# Other — librefang-types

# librefang-types

Shared type definitions, traits, and utility constants for the LibreFang Agent OS ecosystem.

## Purpose

This crate acts as the canonical type layer that all other LibreFang crates depend on. It defines the data structures exchanged between agents, configuration formats, cryptographic material representations, error types, and localization infrastructure. Because it has no runtime behavior of its own—no I/O, no network calls, no state—it can be freely imported by any other crate without introducing circular dependencies or pulling in heavyweight subsystems.

## Dependency Rationale

Each dependency was chosen to support a specific category of shared types:

| Dependency | Role |
|---|---|
| `serde`, `serde_json`, `toml` | Serialization of messages, config files, and on-wire payloads |
| `chrono` | Timestamps and time-related types |
| `uuid` | Unique identifiers for agents, tasks, sessions |
| `thiserror` | Ergonomic error type derivation |
| `ed25519-dalek`, `sha2`, `hex`, `rand` | Public-key identity types, hashing, and key generation |
| `fluent`, `unic-langid` | Localized user-facing strings |
| `regex-lite` | Pattern validation on structured data |
| `async-trait` | Async trait definitions consumed by agent runtimes |
| `dirs` | Standard path constants for config and data directories |

## How It Connects to the Codebase

Every other LibreFang crate that needs to agree on the shape of data—message payloads, agent descriptors, error codes, configuration—depends on `librefang-types`. Because the crate is pure data definitions with no side effects, it sits at the bottom of the dependency graph:

```
┌─────────────────────────────────────────────┐
│  Higher-level crates (agent runtime, CLI,   │
│  networking, task scheduler, ...)            │
└──────────────┬──────────────────────────────┘
               │ depends on
       ┌───────▼───────┐
       │ librefang-    │
       │ types         │
       └───────────────┘
```

No incoming or outgoing runtime calls exist because this module is not executable—it is consumed at compile time by its dependents.

## Building and Testing

```bash
# Build the crate
cargo build -p librefang-types

# Run unit tests
cargo test -p librefang-types
```

The `rmp-serde` dev-dependency is available for tests that verify MessagePack round-trip behavior of serializable types.

## Contributing

When adding a new type:

1. **Keep it free of side effects.** No file I/O, no network, no global state. If a type needs to _do_ something, it belongs in a higher crate.
2. **Derive `Serialize` and `Deserialize`** on any struct or enum that crosses a crate boundary or is persisted to disk.
3. **Use `thiserror`** for error enums so downstream crates get idiomatic `std::error::Error` implementations.
4. **Avoid adding dependencies** unless the type genuinely requires them for correctness. Heavy dependencies should live closer to where they are used.
5. **Write round-trip tests** for any type that is serialized—to JSON, TOML, or binary formats—to guard against breaking wire-format changes.