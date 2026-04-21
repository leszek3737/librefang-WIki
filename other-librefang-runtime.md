# Other — librefang-runtime

# librefang-runtime

The core agent execution environment for LibreFang. This crate orchestrates the full lifecycle of an AI agent — from loading a configuration and connecting to an LLM provider, through managing conversation memory and executing tools, to running sandboxed WASM skills and communicating over MCP channels.

## Architecture

`librefang-runtime` is the integration point that pulls together domain-specific subsystems into a coherent agent loop. It does not implement low-level details itself; instead it coordinates the following sibling crates:

```
┌─────────────────────────────────────────────────────────────┐
│                    librefang-runtime                         │
│                  (agent orchestration)                       │
│                                                              │
│  ┌──────────┐ ┌───────────┐ ┌────────────┐ ┌─────────────┐ │
│  │ LLM      │ │ Skill     │ │ Memory     │ │ Channel     │ │
│  │ Drivers  │ │ Execution │ │ Management │ │ I/O         │ │
│  └────┬─────┘ └─────┬─────┘ └─────┬──────┘ └──────┬──────┘ │
│       │             │             │                │        │
│  ┌────┴─────┐ ┌─────┴──────┐ ┌───┴──────┐ ┌──────┴──────┐ │
│  │ -llm-    │ │ -runtime-  │ │ -memory  │ │ -channels   │ │
│  │ driver   │ │ wasm       │ │          │ │              │ │
│  │ -llm-    │ │ -skills    │ │          │ │              │ │
│  │ drivers  │ │ -kernel-   │ │          │ │              │ │
│  │          │ │  handle    │ │          │ │              │ │
│  └──────────┘ └────────────┘ └──────────┘ └─────────────┘ │
│                                                              │
│  ┌──────────────────┐ ┌────────────────────────────────┐    │
│  │ -runtime-mcp     │ │ -runtime-oauth                 │    │
│  │ (Model Context   │ │ (OAuth token management)       │    │
│  │  Protocol)       │ │                                 │    │
│  └──────────────────┘ └────────────────────────────────┘    │
│                                                              │
│  ┌──────────────────┐ ┌────────────────────────────────┐    │
│  │ -types           │ │ -http                           │    │
│  │ (shared types)   │ │ (HTTP primitives)               │    │
│  └──────────────────┘ └────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Dependency roles

| Crate | Role in the runtime |
|---|---|
| `librefang-types` | Shared domain types — agent IDs, message structs, configuration shapes — used across all crates. |
| `librefang-http` | Low-level HTTP primitives for outbound requests to LLM APIs and other services. |
| `librefang-kernel-handle` | Kernel-level interfaces used during sandboxed execution and process management. |
| `librefang-runtime-mcp` | Model Context Protocol client/server implementation for tool discovery and invocation. |
| `librefang-runtime-oauth` | OAuth 2.0 flows for authenticating with LLM providers and third-party APIs. |
| `librefang-llm-driver` | Trait-level abstraction for LLM backends (streaming, chat completions, embeddings). |
| `librefang-llm-drivers` | Concrete driver implementations (OpenAI, Anthropic, local models, etc.). |
| `librefang-runtime-wasm` | WASM-based sandboxed execution engine for running untrusted skill code via Wasmtime. |
| `librefang-channels` | Async message-passing channels between agent components and external systems. |
| `librefang-memory` | Conversation and working-memory persistence (backed by SQLite via `rusqlite`). |
| `librefang-skills` | Skill registry and execution — tools, functions, and capabilities an agent can invoke. |

## Sandbox features

The runtime supports two optional Linux sandboxing mechanisms to constrain untrusted code execution:

### Landlock (`landlock-sandbox` feature)

Enables the `landlock` crate for file-system access control. When active, WASM skill execution and subprocess spawning can be restricted to specific directory trees, preventing arbitrary file access.

```toml
# Cargo.toml
[features]
landlock-sandbox = ["dep:landlock"]
```

Requires Linux kernel ≥ 5.13.

### Seccomp (`seccomp-sandbox` feature)

Enables the `seccompiler` crate to install BPF syscall filters. This provides a hard boundary on which system calls sandboxed code can make.

```toml
# Cargo.toml
[features]
seccomp-sandbox = ["dep:seccompiler"]
```

Both features can be combined for defense in depth. On non-Linux platforms these features compile but are no-ops.

### WASM hooks (`wasm-hooks` feature)

An additional feature flag that extends the WASM runtime with host-side hook callbacks. Enable this when skills need to react to lifecycle events (initialization, pre/post-execution, teardown) within the Wasmtime environment.

## Key subsystems

### Agent execution loop

The runtime drives a repeating cycle:

1. **Receive input** — from a channel, WebSocket (`tokio-tungstenite`), or HTTP endpoint.
2. **Load context** — conversation history and working memory from `librefang-memory`.
3. **Call the LLM** — via `librefang-llm-driver` / `librefang-llm-drivers`, supporting streaming responses.
4. **Process tool calls** — if the LLM requests tool use, dispatch through `librefang-skills` or `librefang-runtime-mcp`.
5. **Execute skills** — optionally sandboxed via `librefang-runtime-wasm` (Wasmtime).
6. **Persist state** — store updated memory, conversation turns, and skill results.
7. **Emit output** — back through the originating channel.

### Memory and persistence

`rusqlite` provides local SQLite storage for conversation history, agent configuration, and session state. The `librefang-memory` crate exposes the interface; the runtime manages lifecycle (opening connections, running migrations).

### Cryptography and identity

- `ed25519-dalek` — Ed25519 signing/verification for agent identity and message authentication.
- `sha2` / `hmac` — HMAC-based integrity checks and hashing.
- `zeroize` — secure memory cleanup for sensitive key material.

### Concurrency model

The runtime is fully async, built on `tokio`. Shared state uses `dashmap` for lock-free concurrent access to registries (active agents, skill caches, session maps). Stream handling uses `tokio-stream` and `futures` utilities.

### Package management

`flate2` and `tar` handle unpacking skill packages and WASM modules distributed as `.tar.gz` archives. `ureq` (blocking HTTP) is used for simple fetch-and-verify operations during package download where an async HTTP client is unnecessary.

## Building

```bash
# Standard build (no sandboxing)
cargo build -p librefang-runtime

# With Landlock sandboxing (Linux only)
cargo build -p librefang-runtime --features landlock-sandbox

# With both sandboxing mechanisms
cargo build -p librefang-runtime --features "landlock-sandbox,seccomp-sandbox"

# With WASM lifecycle hooks
cargo build -p librefang-runtime --features wasm-hooks
```

## Platform notes

- **Unix-specific**: The `libc` dependency (gated behind `cfg(unix)`) is used for POSIX process management — UID/GID switching, `chroot`, signal handling — when running sandboxed subprocesses.
- **Non-Linux**: The `landlock-sandbox` and `seccomp-sandbox` features depend on Linux-specific syscalls. They compile on other platforms but provide no runtime isolation.
- **TLS**: `rustls` with `webpki-roots` and `rustls-native-certs` provides TLS for outbound connections without depending on OpenSSL.