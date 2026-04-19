# Other — librefang-kernel-handle

# librefang-kernel-handle

## Purpose

This crate defines the `KernelHandle` trait — the primary in-process interface for interacting with the LibreFang kernel. It exists as a standalone crate to break circular dependencies: consumers that need to call into the kernel depend on this lightweight trait crate rather than the full kernel implementation.

The trait follows the classic Rust "interface crate" pattern, providing a stable contract between the kernel and its in-process callers.

## Role in the Architecture

```
┌─────────────────────┐
│  In-process callers │   (plugins, services, tests)
└────────┬────────────┘
         │ depends on
         ▼
┌─────────────────────────┐
│ librefang-kernel-handle │   ← this crate (trait only)
└────────┬────────────────┘
         │ depends on
         ▼
┌─────────────────────┐
│  librefang-types    │
└─────────────────────┘
```

The concrete kernel implementation depends on this crate and provides a type that satisfies `KernelHandle`. Callers depend only on this crate and `librefang-types`, remaining decoupled from kernel internals.

## Dependencies

| Dependency | Role |
|---|---|
| `librefang-types` | Shared domain types used in trait method signatures |
| `async-trait` | Enables async methods in the trait definition |
| `serde` / `serde_json` | Serialization of request/response payloads |
| `tokio` | Async runtime primitives |
| `tracing` | Instrumentation for trait method implementations |
| `uuid` | Identifier generation within handle operations |

## Key Component: `KernelHandle` Trait

The central export of this crate is the `KernelHandle` trait, annotated with `#[async_trait]` to support async method calls. It defines the operations available to in-process callers — typically mirroring the operations the kernel exposes over its external transport, but without serialization overhead.

### Implementing the Trait

A concrete implementation lives in the kernel crate itself. The implementor wraps internal kernel state (channels, state machines, etc.) and translates each trait method call into the appropriate internal operation.

### Consuming the Trait

Callers receive a `dyn KernelHandle` (or a generic `H: KernelHandle`) and invoke methods without knowing whether they are talking to the real kernel, a test mock, or a shim.

## Usage in Tests

Because this crate contains no logic — only a trait definition and shared types — it is ideal for mock-based testing. Test suites depend on `librefang-kernel-handle`, define a stub implementation, and exercise caller code in isolation.

## Extension Guidelines

- **Adding methods:** Add new async methods to the trait with default implementations where possible to avoid breaking existing implementors. If a default body isn't feasible, all implementors (kernel, mocks, shims) must be updated in the same change.
- **Changing signatures:** Treat the trait as a public API. Deprecate old signatures before removing them.
- **New dependencies:** Keep this crate minimal. Only add dependencies that are required by the trait signatures themselves. Implementation-heavy dependencies belong in the kernel crate.