# Other — librefang-kernel-handle

# librefang-kernel-handle

## Purpose

`librefang-kernel-handle` defines the `KernelHandle` trait — the primary interface that in-process callers use to interact with the LibreFang kernel. It provides a consistent, async-friendly contract for issuing requests and receiving responses without requiring callers to understand the kernel's internal architecture.

This is a **trait-only crate**. It contains no runtime logic of its own; it exists to decouple "what the kernel can do" from "how the kernel does it," enabling dependency injection, testing with mocks, and separation of concerns across the workspace.

## Role in the Architecture

```
┌──────────────────────────────────────────────────────┐
│                  In-Process Caller                    │
│         (holds a dyn KernelHandle)                    │
└──────────────┬───────────────────────────────────────┘
               │ async method calls
               ▼
┌──────────────────────────────────────────────────────┐
│           librefang-kernel-handle                     │
│         KernelHandle trait definition                 │
└──────────────┬───────────────────────────────────────┘
               │ implemented by
               ▼
┌──────────────────────────────────────────────────────┐
│         Concrete kernel implementation                │
│    (e.g., librefang-kernel or test doubles)           │
└──────────────────────────────────────────────────────┘
```

Any crate that needs to talk to the kernel depends on `librefang-kernel-handle` rather than the kernel implementation directly. This keeps the dependency graph clean and makes unit testing straightforward — test suites can provide a lightweight stub or mock implementing the same trait.

## Dependencies

| Crate | Why it's needed |
|---|---|
| `librefang-types` | Shared domain types (requests, responses, errors) that flow through the trait methods |
| `async-trait` | Enables `async` methods in the trait definition via the `#[async_trait]` macro |
| `serde_json` | JSON serialization for message payloads passed across the handle boundary |
| `tracing` | Structured logging/spans within trait default implementations or instrumented wrappers |
| `uuid` | Correlation and request identifiers used in handle operations |

## Usage

Add the crate as a dependency in `Cargo.toml`:

```toml
[dependencies]
librefang-kernel-handle = { path = "../librefang-kernel-handle" }
```

Then accept a handle as a trait object where the kernel is needed:

```rust
use librefang_kernel_handle::KernelHandle;

async fn do_work(kernel: &dyn KernelHandle) {
    // Call trait methods on `kernel`...
}
```

## Testing

Because the crate is a pure trait definition with no execution flows, testing against it means providing your own implementation. A typical pattern:

```rust
struct MockKernelHandle { /* fields */ }

#[async_trait]
impl KernelHandle for MockKernelHandle {
    // Return canned responses for unit tests
}
```

This avoids pulling in the real kernel or any I/O layer into test binaries.

## Design Notes

- **No `impl` blocks with real logic.** The crate intentionally avoids concrete implementations. All behavior lives in downstream crates.
- **Async-first.** Every trait method is async, reflecting the inherently asynchronous nature of kernel communication.
- **Trait object safe.** The trait is designed to be usable as `dyn KernelHandle`, enabling runtime polymorphism across the workspace.
- **Minimal dependency surface.** The crate depends only on types and utilities — never on other implementation crates — to prevent circular dependencies.