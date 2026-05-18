# Other — librefang-kernel-handle

# librefang-kernel-handle

A trait abstraction that defines the interface for in-process callers into the LibreFang kernel.

## Purpose

This crate provides the `KernelHandle` trait — a boundary contract between the LibreFang kernel and any in-process component that needs to interact with it. By defining this as a trait rather than a concrete type, the kernel interface can be mocked, decorated, or replaced in tests and alternative configurations without touching caller logic.

## Role in the Architecture

```
┌─────────────────────────┐
│   In-process caller     │
│  (plugin, service, etc) │
└────────────┬────────────┘
             │ depends on
             ▼
┌─────────────────────────┐
│ librefang-kernel-handle │   ← this crate
│   KernelHandle trait    │
└────────────┬────────────┘
             │ implemented by
             ▼
┌─────────────────────────┐
│   LibreFang Kernel      │
│   (concrete impl)       │
└─────────────────────────┘
```

The trait lives in its own crate to avoid circular dependencies. Callers depend only on this lightweight interface crate, while the kernel provides the concrete implementation. Neither side needs to depend on the other's full crate.

## Dependencies

| Dependency | Role |
|---|---|
| `librefang-types` | Shared domain types exchanged across kernel calls (messages, errors, identifiers) |
| `async-trait` | Enables `async` methods in the trait definition |
| `bytes` | Efficient byte buffer handling for payload data |
| `serde_json` | JSON serialization for structured kernel messages |
| `thiserror` | Derive macro for error types surfaced from kernel operations |
| `uuid` | Unique identifiers for sessions, requests, or correlation tokens |
| `tracing` | Structured logging and diagnostics within trait implementations |

## Testing

The dev-dependency on `tokio` (with `macros` and `rt` features) indicates that tests in this crate use the Tokio runtime, consistent with the async nature of the trait. Test authors should annotate test functions with `#[tokio::test]`.

## Relationship to Other Crates

- **Consumers**: Any crate that needs to call into the kernel (plugins, adapters, middleware) depends on this crate and accepts a `dyn KernelHandle` or generic `H: KernelHandle`.
- **Implementors**: The kernel crate itself implements this trait on its concrete handle type, satisfying the contract defined here.
- **Shared types**: All types passed across the trait boundary are defined in `librefang-types`, keeping this crate free of domain logic.