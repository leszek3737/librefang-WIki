# Shared Types & Configuration — librefang-kernel-src

# Shared Types & Configuration — `librefang-kernel-src`

## Overview

This is the error module for the `librefang-kernel` crate. It defines kernel-specific error types that extend the shared `LibreFangError` from `librefang_types` with additional context relevant to kernel operations such as boot sequences.

## Purpose

The kernel has its own failure modes (e.g., boot failures) that don't belong in the shared type library. This module provides a `KernelError` enum that:

1. **Wraps** the shared `LibreFangError` transparently, so any error originating from shared code propagates without loss of information.
2. **Adds** kernel-specific variants that only make sense in the context of kernel initialization and execution.
3. **Standardizes** the return type across the kernel crate via the `KernelResult<T>` alias.

## Key Components

### `KernelError` Enum

| Variant | Purpose |
|---|---|
| `LibreFang(LibreFangError)` | Transparently wraps any error from the shared `librefang_types` crate. Uses `#[from]` for automatic `From` conversion, so `?` works directly on `LibreFangError` values inside kernel code. |
| `BootFailed(String)` | Indicates the kernel was unable to complete its boot process. Carries a human-readable description of what went wrong. |

The `#[error(transparent)]` attribute on `LibreFang` means the displayed message is delegated to the underlying `LibreFangError` — no wrapping text is added.

### `KernelResult<T>` Type Alias

```rust
pub type KernelResult<T> = Result<T, KernelError>;
```

Use this as the return type for any kernel function that can fail. It keeps signatures clean and consistent across the crate.

## Usage Patterns

### Propagating shared errors

Functions that call into `librefang_types` APIs can use `?` directly, since `KernelError` implements `From<LibreFangError>`:

```rust
use librefang_kernel::error::KernelResult;
use librefang_types::some_shared_function;

fn init_subsystem() -> KernelResult<()> {
    some_shared_function()?; // LibreFangError converts automatically
    Ok(())
}
```

### Reporting boot failures

```rust
use librefang_kernel::error::{KernelError, KernelResult};

fn boot() -> KernelResult<()> {
    if !hardware_ok() {
        return Err(KernelError::BootFailed("RAM initialization failed".into()));
    }
    Ok(())
}
```

## Relationship to the Wider Codebase

```
librefang_types::error::LibreFangError
        │
        │  #[from]
        ▼
librefang_kernel::error::KernelError
        │
        │  via KernelResult<T>
        ▼
   All kernel modules
```

`KernelError` sits between the shared error type and the rest of the kernel crate. Kernel-internal modules depend on `KernelResult<T>` and `KernelError`, but the shared `librefang_types` crate remains unaware of kernel-specific concerns.