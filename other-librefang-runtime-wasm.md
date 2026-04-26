# Other — librefang-runtime-wasm

# librefang-runtime-wasm

WASM skill sandbox for the LibreFang runtime.

## Purpose

This module provides a WebAssembly-based sandbox environment for executing LibreFang skills. By isolating skill logic inside WASM modules, the runtime gains:

- **Security** — skills run in a sandboxed context with controlled access to system resources
- **Portability** — skills can be authored in any language that compiles to WASM
- **Isolation** — a faulty or malicious skill cannot crash the host runtime

## Architecture

```
┌─────────────────────────────────────┐
│         Host Runtime                │
│                                     │
│  ┌───────────────────────────────┐  │
│  │   librefang-runtime-wasm     │  │
│  │                               │  │
│  │  ┌─────────┐    ┌─────────┐  │  │
│  │  │ Wasmtime│───▶│  Skill   │  │  │
│  │  │ Engine  │    │ (WASM)  │  │  │
│  │  └─────────┘    └─────────┘  │  │
│  │         │                     │  │
│  │         ▼                     │  │
│  │  kernel-handle / http         │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

The WASM runtime acts as a bridge between the host system and guest skill code. Skills execute inside wasmtime instances and communicate outward through host-provided APIs.

## Dependencies and Their Roles

| Dependency | Role |
|---|---|
| `wasmtime` | WebAssembly runtime engine — compiles and executes WASM modules |
| `librefang-kernel-handle` | Communication channel back to the LibreFang kernel |
| `librefang-http` | HTTP client/server capabilities exposed to sandboxed skills |
| `librefang-types` | Shared type definitions used across the LibreFang workspace |
| `tokio` | Async runtime for non-blocking WASM instantiation and execution |
| `serde_json` | JSON serialization for structured data exchange with WASM guests |
| `tracing` | Instrumentation for debugging and monitoring skill execution |
| `thiserror` / `anyhow` | Error types for sandbox and runtime failures |

## Key Concepts

### Skill Loading

WASM modules representing skills are loaded into wasmtime `Engine` instances. Each skill gets its own isolated `Store` to prevent cross-skill interference.

### Host-Guest Communication

Skills interact with the outside world exclusively through host functions imported into the WASM module. These imports are backed by `librefang-kernel-handle` and `librefang-http`, giving the runtime fine-grained control over what each skill can access.

Data flows across the boundary as serialized JSON (`serde_json`), keeping the interface language-agnostic for guest modules.

### Sandboxing

The wasmtime runtime provides capability-safe isolation. Skills have no direct access to the filesystem, network, or kernel — all access is mediated through explicitly imported host functions.

## Integration with LibreFang

This module sits between the kernel and skill authors:

- **Inbound**: The kernel (or a higher-level runtime orchestrator) instructs this module to load and execute a specific skill WASM module.
- **Outbound**: This module calls into `librefang-kernel-handle` to relay results or request services, and into `librefang-http` when skills need network access.

The shared `librefang-types` crate ensures type consistency across all layers.