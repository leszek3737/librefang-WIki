# Other — librefang-types

# librefang-types

Core type definitions and traits shared across the LibreFang Agent OS.

## Purpose

This crate is the foundational type library for LibreFang. It contains no business logic or runtime behavior — only data structures, error types, trait definitions, and serialization helpers that other crates in the workspace depend on. Every other module in LibreFang references this crate for shared vocabulary.

Because it has no incoming or outgoing call edges and no execution flows, it is a pure leaf dependency in the compile graph. It can be reviewed, tested, and versioned independently of any agent runtime.

## Dependency Rationale

The dependencies reveal the domains this crate covers:

| Dependency | Purpose |
|---|---|
| `serde`, `serde_json`, `toml` | Serialization formats for messages, config, and persistence |
| `chrono`, `uuid` | Timestamps and unique identifiers for events and agents |
| `thiserror` | Ergonomic error type derivation |
| `ed25519-dalek`, `sha2`, `hex`, `zeroize` | Cryptographic types — Ed25519 keys, SHA digests, secure memory handling |
| `fluent`, `unic-langid` | Internationalization types and language identifiers |
| `regex-lite` | Pattern types for validation rules |
| `schemars` | JSON Schema generation from Rust types (enables `chrono` and `uuid1` features so generated schemas understand those types) |
| `async-trait` | Async trait definitions for plugin interfaces or transport abstractions |
| `dirs` | Standard directory path constants |
| `tracing` | Span/event types for structured logging |

Dev dependencies (`rmp-serde`, `tempfile`) support unit tests that verify serialization round-trips through MessagePack and temporary file I/O.

## Type Domains

Based on the dependency set, this crate defines types across these domains:

### Identity and Cryptography

Types for Ed25519 key pairs, public keys, signatures, and SHA-256 digests. The `zeroize` dependency ensures secret key material can be securely cleared from memory. These types wrap `ed25519-dalek` primitives with serde support so they can be transmitted and stored.

### Messaging and Events

Request/response envelopes, event payloads, and message headers using `uuid` for correlation IDs and `chrono` for timestamps. All message types derive `Serialize` and `Deserialize`.

### Configuration

Structured config types parsed from TOML files. The `dirs` crate provides standard paths (config dir, data dir) as constants or helper functions.

### Error Handling

A hierarchy of error types using `thiserror` — covering serialization failures, crypto errors, validation errors, and I/O errors. These are designed to be composable so consuming crates can use `From` conversions.

### Localization

Types for language identifiers (`unic-langid`) and Fluent localization resources. These allow the agent OS to produce localized status messages and error descriptions.

### Schema Generation

Types annotated with `schemars` derive macros so JSON Schemas can be auto-generated. This is useful for API documentation, validation, and interop with external tooling.

## Traits

The `async-trait` dependency indicates async trait definitions. These likely define interfaces that agent plugins or transport backends must implement, such as:

- Message send/receive contracts
- Authentication or key exchange protocols
- Plugin lifecycle hooks

These traits live in the types crate so that implementations and consumers both depend on the same abstraction without creating circular dependencies.

## Usage in the Workspace

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  agent-bin   │────▶│  librefang-  │────▶│  librefang-  │
│              │     │  runtime     │     │  types       │
└──────────────┘     └──────────────┘     └──────────────┘
┌──────────────┐     ┌──────────────┐            ▲
│  librefang-  │────▶│  librefang-  │────────────┘
│  crypto      │     │  transport   │
└──────────────┘     └──────────────┘
```

Every other crate in the workspace depends on `librefang-types` but `librefang-types` depends on nothing within the workspace. This unidirectional dependency graph keeps the type layer stable and avoids circular imports.

## Conventions

- All public types derive `Serialize`, `Deserialize`, `Debug`, and `Clone` unless there is a specific reason not to (e.g., secret key types omit `Debug`).
- Error types implement `std::error::Error` and `Display` via `thiserror`.
- Types that are part of a public API surface derive `JsonSchema` from `schemars`.
- Secret-bearing structures use `zeroize::Zeroize` on drop.

## Testing

Tests focus on serialization round-trips (JSON, TOML, and MessagePack via `rmp-serde`) and schema generation correctness. The `tempfile` dev dependency supports tests that verify config file parsing from disk.