# Other — librefang-kernel-handle

# librefang-kernel-handle

## Purpose

This crate defines the **`KernelHandle`** trait — the primary interface that in-process callers use to interact with the LibreFang kernel. It serves as the contract between consumer code (plugins, services, or internal subsystems) and the kernel's core operations.

Rather than exposing kernel internals directly, the `KernelHandle` trait provides an abstracted, async-capable boundary. Any in-process component that needs to invoke kernel functionality does so through this trait, keeping the coupling loose and testable.

## Role in the Architecture

```
┌──────────────────────────────┐
│   In-process callers         │
│   (plugins, services, etc.)  │
│                              │
│         KernelHandle trait   │
│              ▲               │
│              │               │
│   ┌──────────┴──────────┐    │
│   │  Concrete kernel     │    │
│   │  implementation      │    │
│   └─────────────────────┘    │
└──────────────────────────────┘
```

The trait lives in its own crate so that consumers depend only on this lightweight interface — not on the full kernel implementation. This separation means:

- **Consumers** add `librefang-kernel-handle` as a dependency and program against the trait.
- **The kernel** provides a concrete implementation of `KernelHandle`.
- **Tests** can substitute a mock or stub implementation without pulling in kernel internals.

## Dependencies and What They Indicate

| Dependency | Reason |
|---|---|
| `librefang-types` | Shared domain types used in trait method signatures and return values. |
| `async-trait` | The `KernelHandle` trait is async — methods return futures that callers `.await`. |
| `serde` / `serde_json` | Some trait methods accept or return serialized data, likely for message-passing or persistence boundaries. |
| `tokio` | Async runtime primitives (channels, spawning) used within the trait's support code or default method bodies. |
| `tracing` | Instrumentation spans for observability across kernel calls. |
| `uuid` | Operations are identified or keyed by UUID — sessions, requests, or entity references. |

## Usage Pattern

A typical consumer receives a `KernelHandle` implementation (often via dependency injection or construction at startup) and calls trait methods:

```rust
use librefang_kernel_handle::KernelHandle;

async fn do_work(kernel: &dyn KernelHandle) {
    // Call methods defined on the trait — the concrete kernel
    // handles dispatch, validation, and side effects.
}
```

Because the trait is defined with `async-trait`, every method returns a `Pin<Box<dyn Future>>` that the caller simply awaits. Implementors provide the concrete async logic.

## Connection to the Rest of the Codebase

- **`librefang-types`**: The shared vocabulary. Every struct, enum, or newtype that crosses the `KernelHandle` boundary is defined there, ensuring both the kernel and its consumers agree on the data shapes.
- **Kernel implementation crate**: Provides the concrete struct that `impl KernelHandle`. That crate depends on `librefang-kernel-handle` (for the trait) plus all the subsystem crates needed to actually fulfill requests.
- **Consumer crates**: Any in-process component that needs kernel access depends on `librefang-kernel-handle` and accepts `dyn KernelHandle` (or a generic parameter constrained by it).

## Implementing `KernelHandle`

When adding a new method to the trait:

1. **Define it in this crate** with an async signature via `async-trait`.
2. **Update the kernel implementation** to provide the concrete logic.
3. **Update tests and mocks** to cover the new method.
4. **Keep `librefang-types` in sync** — any new parameter or return type belongs there, not here.

This crate should never contain business logic. Its sole responsibility is the trait definition and any supporting types that are *specific* to the handle abstraction itself (e.g., error types wrapping kernel-level errors for consumption by callers).