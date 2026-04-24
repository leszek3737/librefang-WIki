# Other — librefang-types

# librefang-types

Core type definitions and shared traits for the LibreFang Agent OS.

## Purpose

This crate serves as the foundational type library that all other LibreFang components depend on. It defines the common data structures, error types, configuration schemas, cryptographic primitives, and async traits that flow through the system. By isolating types into their own crate, the broader workspace avoids circular dependency issues and gains a single source of truth for shared contracts.

This is a **leaf crate** — it has no internal dependencies on other LibreFang crates and makes no outbound calls. Every other crate in the workspace that needs to speak a common language imports from here.

## Dependency Rationale

The external dependencies reveal the categories of types this crate provides:

| Dependency | Role in the crate |
|---|---|
| `serde`, `serde_json` | Derive `Serialize`/`Deserialize` on all wire-format types |
| `chrono` | Timestamps on events, logs, and signed payloads |
| `uuid` | Unique identifiers for agents, sessions, tasks |
| `thiserror` | Ergonomic error enum definitions |
| `ed25519-dalek` | Ed25519 keypairs, signatures, and verification types |
| `sha2` | SHA-256 hashing for integrity checks |
| `hex` | Encoding/decoding of byte data in hex representation |
| `rand` | Cryptographically secure random number generation for key material |
| `dirs` | Platform-aware path resolution for config and data directories |
| `toml` | Deserialization of TOML-based configuration files |
| `async-trait` | Async trait definitions for agent behavior and transport abstractions |
| `fluent`, `unic-langid` | Internationalization (i18n) message types and locale identifiers |
| `regex-lite` | Lightweight regex pattern definitions for validation rules |

The dev-dependency on `rmp-serde` (MessagePack) indicates that serialization round-trip tests cover multiple formats beyond JSON.

## How It Fits the Architecture

```
┌─────────────────────────────────────────────┐
│           LibreFang Agent OS                 │
│                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐    │
│  │  Agent    │ │ Transport│ │  Task     │   │
│  │  Runtime  │ │  Layer   │ │  Runner   │   │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘    │
│       │             │            │          │
│       └─────────────┼────────────┘          │
│                     ▼                       │
│            ┌─────────────────┐              │
│            │ librefang-types │              │
│            └─────────────────┘              │
└─────────────────────────────────────────────┘
```

Every crate that produces, consumes, or routes messages depends on `librefang-types` for the shared vocabulary. This includes agent identifiers, signed envelopes, task definitions, configuration structs, and error types.

## Likely Type Categories

Based on the dependency set, the crate organizes into these conceptual groups:

### Identity and Cryptography

Types wrapping `ed25519-dalek` and `sha2` for agent keypairs, payload signing, signature verification, and hash-based integrity checks. These types are serialized with `serde` for transmission over the wire and use `hex` for human-readable representations of byte data.

### Configuration

Structs deserialized from TOML that define agent behavior — connection endpoints, polling intervals, feature flags. The `dirs` crate resolves platform-specific configuration paths.

### Task and Message Envelopes

Message types that carry task payloads, results, and control signals between agents and the control plane. Each carries a `uuid` identifier and `chrono` timestamp.

### Error Types

Error enums derived with `thiserror` that provide structured, typed errors across the system rather than raw string propagation.

### Async Traits

Trait definitions annotated with `async-trait` that define contracts for transport backends, agent lifecycle hooks, and task execution — without coupling consumers to a specific implementation.

### Internationalization

Types leveraging `fluent` and `unic-langid` to support localized log messages and user-facing strings, keyed by locale identifiers.

### Validation

Regex patterns built with `regex-lite` for input validation rules on identifiers, network addresses, or configuration values.

## Testing

The dev-dependency on `rmp-serde` confirms that type definitions are tested for serialization round-trips across multiple formats (at minimum JSON and MessagePack). This ensures that any type annotated with `Serialize` and `Deserialize` maintains fidelity when encoded and decoded.

## Contributing Guidelines

When adding types to this crate:

- **Every public struct or enum must derive `Serialize` and `Deserialize`** unless there is a compelling reason not to. These types cross process and network boundaries.
- **Avoid pulling in logic dependencies.** This crate should remain free of side effects, I/O, and runtime behavior. If a type needs to *do* something, the logic belongs in a higher crate; only the shape of the data belongs here.
- **Prefer `thiserror` for error types** to maintain consistency with existing error definitions.
- **Write serialization round-trip tests** for any new type, covering at least JSON. Use the existing `rmp-serde` dev-dependency if MessagePack is a relevant wire format for the new type.
- **Keep the dependency list lean.** Any new dependency here affects every crate in the workspace. Evaluate whether the type truly belongs in this shared crate or in a more specialized one.