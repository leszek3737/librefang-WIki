# Other — librefang-runtime-wasm

# librefang-runtime-wasm

WASM skill sandbox for the LibreFang runtime.

## Purpose

This crate provides a sandboxed WebAssembly execution environment for LibreFang skills. It uses [wasmtime](https://wasmtime.dev/) to compile and run WASM modules in a controlled, isolated runtime, allowing user-defined or third-party skill logic to execute without compromising the host process.

## Architecture

```
┌─────────────────────────────────────────────┐
│           librefang-runtime-wasm            │
│                                             │
│  ┌─────────────┐    ┌───────────────────┐   │
│  │ WASM Module  │◄───│ Sandbox Runtime   │   │
│  │ (skill.wasm) │    │ (wasmtime Engine) │   │
│  └──────┬───────┘    └────────┬──────────┘   │
│         │                     │              │
│         │   ┌─────────────────┘              │
│         │   │ Host Functions                 │
│         ▼   ▼                                │
│  ┌──────────────────────────────────────┐    │
│  │        Kernel Handle Bridge          │    │
│  └──────────────────────────────────────┘    │
└───────────┬─────────────────────────┬────────┘
            │                         │
            ▼                         ▼
┌───────────────────────┐  ┌──────────────────┐
│  librefang-kernel-    │  │ librefang-http   │
│  handle               │  │                  │
└───────────────────────┘  └──────────────────┘
```

## Key Design Decisions

### Sandboxed Execution

Skills run inside a wasmtime instance, which provides:

- **Memory isolation** — WASM linear memory is bounded and cannot access host memory directly
- **Capability control** — host functions are explicitly imported, giving the runtime fine-grained control over what a skill can do
- **Resource limits** — wasmtime supports fuel-based execution, enabling CPU time caps on untrusted code

### Host-Side Integration

The crate bridges WASM guest code to LibreFang kernel services through the `librefang-kernel-handle` dependency. This allows sandboxed skills to interact with game state, events, and other runtime facilities via controlled host function imports.

## Dependencies

| Crate | Role in this module |
|---|---|
| `wasmtime` | WebAssembly runtime — compiles and executes WASM modules |
| `librefang-kernel-handle` | Exposes kernel operations as host functions callable from WASM |
| `librefang-http` | HTTP client capability for skills that need network access |
| `librefang-types` | Shared type definitions exchanged across crate boundaries |
| `tokio` | Async runtime backing WASM instantiation and host calls |
| `serde_json` | Serialization of structured data between host and guest |
| `tracing` | Instrumentation for sandbox lifecycle and execution events |
| `thiserror` / `anyhow` | Error types for sandbox setup and execution failures |

## Integration Points

This crate sits between the game engine and user-supplied skill code. The broader flow is:

1. The engine loads a skill's `.wasm` binary (compiled from Rust, C, or any WASM-targeting language)
2. This crate creates a wasmtime `Engine` and `Store`, configures allowed imports, and instantiates the module
3. The skill's entry point is invoked, with host functions available for kernel interaction and HTTP requests
4. Results are returned to the engine via the kernel handle layer

No other crates in the workspace currently call into this module at compile time — it is expected to be consumed through runtime dependency injection or feature-gated initialization.