# Other — librefang-types

# librefang-types

Core type definitions and traits shared across the LibreFang Agent OS ecosystem.

## Purpose

This crate serves as the foundational type layer for the entire LibreFang project. It defines the canonical data structures, error types, configuration schemas, and trait contracts that all other crates consume. By centralizing these definitions, it ensures type consistency across the agent runtime, communication protocols, and tooling.

This is a **pure definition crate** — it contains no business logic, no I/O, and no side effects. Other crates depend on it; it depends on nothing within the workspace.

## Role in the Architecture

```
┌──────────────────────────────────────────────┐
│           Application / Agent Crate           │
├──────────────────────────────────────────────┤
│         Other workspace crates                │
├──────────────────────────────────────────────┤
│              librefang-types                  │
│   (types, traits, errors, config schemas)     │
├──────────────────────────────────────────────┤
│         External crate dependencies           │
└──────────────────────────────────────────────┘
```

Every other crate in the workspace imports from `librefang-types` to share a common vocabulary. This eliminates duplicate definitions and guarantees that, for example, an agent ID is always the same `uuid::Uuid`-backed type everywhere it appears.

## Key Areas

The dependency set reveals several distinct domains this crate covers:

### Identity and Cryptography

Backed by `ed25519-dalek`, `sha2`, `hex`, and `rand`. This crate likely defines types for agent identity keys, message signing, signature verification, and hashing. Agents in the LibreFang system authenticate and verify each other using Ed25519 keypairs.

### Serialization

Backed by `serde` and `serde_json`. All core types derive `Serialize` and `Deserialize`, making them wire-ready for JSON transport, persistent storage, or inter-process communication.

### Identifiers and Timestamps

Backed by `uuid` and `chrono`. Agent IDs, session tokens, task identifiers, and timestamped events all rely on these types. Using `chrono` with `serde` integration ensures consistent datetime serialization across the system.

### Error Handling

Backed by `thiserror`. The crate defines the canonical error enums for the system — agent registration failures, validation errors, protocol mismatches, and so on. Other crates reference these error types rather than defining their own overlapping variants.

### Configuration

Backed by `toml` and `dirs`. The agent configuration file schema lives here, including agent settings, path resolution, and deployment profiles.

### Localization

Backed by `fluent` and `unic-langid`. User-facing messages and log output support multiple languages. The localization types and the fluent bundle loading contracts are defined here so that any crate can produce localized output consistently.

### Validation

Backed by `regex-lite`. Field validation rules — agent name patterns, allowed characters, format constraints — are expressed as types or validation functions that other crates call into.

### Async Trait Contracts

Backed by `async-trait`. Trait definitions for async-capable interfaces — message handlers, transport layers, storage backends — are declared here so that implementations across different crates remain interchangeable.

## Usage

Add to your crate's `Cargo.toml`:

```toml
[dependencies]
librefang-types = { path = "../librefang-types" }
```

Then import the types you need:

```rust
use librefang_types::AgentId;
use librefang_types::error::AgentError;
use librefang_types::config::AgentConfig;
```

## Testing

The dev-dependency on `rmp-serde` indicates that some types are tested for MessagePack serialization round-trips, even though MessagePack is not a runtime dependency. This verifies that the type definitions remain compatible with binary serialization formats used elsewhere in the system.