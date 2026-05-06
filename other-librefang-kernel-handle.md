# Other — librefang-kernel-handle

# librefang-kernel-handle

## Purpose

This crate defines the `KernelHandle` trait — the primary interface for in-process callers to interact with the LibreFang kernel. It serves as the abstract boundary between kernel consumers and the kernel's internal implementation, allowing different components to invoke kernel operations without coupling to a specific kernel backend.

## Role in the Architecture

`librefang-kernel-handle` sits at the edge of the kernel layer, acting as a contract that any kernel implementation must satisfy. Downstream crates depend on this trait to make calls into the kernel, while concrete kernel implementations provide the trait's methods.

```
┌──────────────────────┐
│   In-process callers │
│   (plugins, runtime) │
└──────────┬───────────┘
           │ depends on
           ▼
┌──────────────────────┐
│ KernelHandle (trait) │   ← this crate
└──────────┬───────────┘
           │ implemented by
           ▼
┌──────────────────────┐
│  Concrete kernel     │
│  implementation      │
└──────────────────────┘
```

## Key Dependencies

The crate's dependencies reveal the shape of the trait's interface:

| Dependency | Purpose |
|---|---|
| `async-trait` | The `KernelHandle` trait is async — methods return futures. |
| `librefang-types` | Shared domain types used in method signatures (messages, requests, responses). |
| `bytes` | Efficient byte-buffer handling for data passed to/from the kernel. |
| `serde_json` | JSON serialization for structured data exchange across the trait boundary. |
| `uuid` | UUID-based identifiers for entities managed by the kernel. |
| `thiserror` | Defines typed error enums for kernel handle operations. |
| `tracing` | Diagnostic span instrumentation on trait method calls. |

## Error Handling

The crate defines its own error types using `thiserror`. Callers should expect these errors from any `KernelHandle` implementation and handle them appropriately — they represent failures in the kernel operation itself, not in the caller's logic.

## Integration Guide

### Depending on This Crate

Add to your `Cargo.toml`:

```toml
[dependencies]
librefang-kernel-handle = { path = "../librefang-kernel-handle" }
```

### Implementing the Trait

If you are writing a kernel backend, implement `KernelHandle` for your kernel type. The trait is async, so implementations should use `async-trait`-compatible signatures.

### Consuming the Trait

If you are writing a component that calls into the kernel, accept a `dyn KernelHandle` (or a generic `impl KernelHandle`) in your constructors rather than a concrete type. This allows your component to work with any kernel implementation and facilitates testing with mocks.

## Testing

The crate uses `tokio` with `macros` and `rt` features as a dev-dependency, indicating that its tests are asynchronous. When writing tests against `KernelHandle`, use a Tokio runtime:

```rust
#[tokio::test]
async fn my_test() {
    // test code using a KernelHandle implementation
}
```

## Linting

This crate inherits workspace-level lints, ensuring consistency with the broader LibreFang codebase.