# Other — librefang-runtime-wasm

# librefang-runtime-wasm

WASM skill sandbox for the LibreFang runtime. This crate provides a WebAssembly-based execution environment that isolates and runs user-authored "skills" within the LibreFang game system.

## Purpose

LibreFang skills are user-defined behaviors that extend game functionality. Running them inside a WASM sandbox provides:

- **Security**: Skills cannot access host memory or system resources directly
- **Portability**: Skills compiled to WASM run on any platform LibreFang supports
- **Control**: The host determines exactly which capabilities are exposed to each skill

## Architecture

```mermaid
graph LR
    subgraph Host
        K[librefang-kernel-handle]
        H[librefang-http]
        T[librefang-types]
    end

    subgraph "librefang-runtime-wasm"
        E[Wasmtime Engine]
        S[Sandbox]
    end

    subgraph "Skill WASM Module"
        SK[User Code]
    end

    E -->|loads & instantiates| SK
    SK -->|imports (host functions)| S
    S -->|delegates to| K
    S -->|delegates to| H
    T -.->|shared types| S
```

The runtime loads compiled `.wasm` modules using **Wasmtime**, injects host-provided functions as WASM imports, and orchestrates skill execution. Skills interact with the game kernel and HTTP layer exclusively through these imported functions—never directly.

## Key Dependencies

| Dependency | Role |
|---|---|
| `wasmtime` | Compiles and executes WASM modules; provides the sandboxing boundary |
| `librefang-kernel-handle` | Exposes game kernel operations that skills can invoke |
| `librefang-http` | Provides HTTP capabilities to skills when permitted |
| `librefang-types` | Shared type definitions passed between host and guest |
| `tokio` | Async runtime for non-blocking skill execution |
| `serde_json` | Serializing/deserializing data exchanged across the WASM boundary |
| `tracing` | Structured logging of sandbox lifecycle events |

## Integration Points

This crate sits between the game kernel and user-authored code. It consumes `librefang-kernel-handle` to proxy kernel operations into the WASM guest and uses `librefang-types` to ensure type consistency across the boundary.

Other LibreFang components depend on this crate to instantiate and manage the skill sandbox rather than interacting with WASM directly.