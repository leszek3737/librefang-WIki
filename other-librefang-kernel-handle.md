# Other — librefang-kernel-handle

# librefang-kernel-handle

Defines the `KernelHandle` trait — the primary abstraction that in-process callers use to interact with the LibreFang kernel.

## Purpose

This crate provides a consistent interface for any component running inside the same process as the LibreFang kernel. Rather than coupling callers directly to a concrete kernel implementation, dependents code against the `KernelHandle` trait. This decouples the "what you can do" from the "how it's implemented," making testing, mocking, and architectural layering straightforward.

The trait is **async** (via `async-trait`) and is designed to be used with the Tokio runtime.

## When to Use This Crate

- You are building an in-process service, plugin, or subsystem that needs to call into the LibreFang kernel.
- You need to mock or stub kernel interactions in tests for a component that lives in the same process.
- You are implementing the kernel itself and need to expose a handle that satisfies this contract.

## Dependencies and What They Signal

| Dependency | Role |
|---|---|
| `librefang-types` | Shared domain types (request/response structs, error types, etc.) that flow through the kernel interface. |
| `async-trait` | Enables the trait to declare async methods that callers can `await`. |
| `serde` / `serde_json` | Trait methods likely accept or return types that are `Serialize`/`Deserialize` — for example, when forwarding structured data through the kernel boundary. |
| `tokio` | The async runtime backing the trait's futures. |
| `tracing` | Instrumentation for kernel handle invocations. |
| `uuid` | Identity types used in requests or correlation tracking. |

## Architecture

```
┌──────────────────────┐
│   In-process Caller   │
│  (service, plugin,    │
│   test harness, etc.) │
└──────────┬───────────┘
           │ depends on trait
           ▼
┌──────────────────────┐
│   KernelHandle trait  │  ← this crate
└──────────┬───────────┘
           │ implemented by
           ▼
┌──────────────────────┐
│   LibreFang Kernel    │
│   (concrete impl)     │
└──────────────────────┘
```

The concrete kernel lives in a separate crate and implements `KernelHandle`. Callers only reference this trait crate, keeping them insulated from kernel internals.

## Integration with the Wider Codebase

- **`librefang-types`** is the shared vocabulary. Any struct or enum that crosses the kernel boundary is defined there, not here. This crate re-uses those types in its trait signatures.
- Concrete implementations of `KernelHandle` live elsewhere (typically in the kernel crate itself or in a dedicated integration layer).
- Downstream crates add `librefang-kernel-handle` as a dependency and accept an `impl KernelHandle` (or `Arc<dyn KernelHandle>`) in their constructors.

## Testing Patterns

Because the trait is the sole contract, unit tests for callers typically:

1. Create a mock/stub that implements `KernelHandle`.
2. Wire the mock into the component under test.
3. Assert that the component invokes the expected trait methods with the expected arguments.

No real kernel is needed for these tests — the trait is the seam.