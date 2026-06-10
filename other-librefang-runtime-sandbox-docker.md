# Other — librefang-runtime-sandbox-docker

# librefang-runtime-sandbox-docker

Docker container sandbox for LibreFang tool execution. Provides OS-level isolation for agent code execution by spawning commands inside Docker containers with strict resource limits, network isolation, and capability dropping.

## Purpose

This crate was extracted from `librefang-runtime` as part of the **#3710 god-crate split** (Phase 1). Its sole responsibility is to take a command that an agent wants to run and execute it inside a hardened Docker container, preventing untrusted or semi-trusted code from affecting the host system.

## Security Model

Containers are spawned with three layers of hardening:

- **Resource limits** — CPU and memory constraints prevent runaway processes from starving the host.
- **Network isolation** — containers are cut off from the host network unless explicitly granted access.
- **Capability dropping** — Linux capabilities are stripped to the minimum required set.

User-supplied commands pass through a shell metacharacter inspection pipeline (the **helpers module**) before execution. This denylist is parity-tested against the parent crate's own denylist to ensure no drift — see `crates/librefang-runtime/tests/docker_sandbox_helpers_parity.rs`.

## Dependencies

| Crate | Role |
|---|---|
| `librefang-types` | Shared types used across the workspace |
| `tokio` | Async runtime for container lifecycle management |
| `tracing` | Structured logging of sandbox events |
| `dashmap` | Concurrent map for tracking active containers |
| `chrono` | Timestamps for container metadata and timeouts |
| `sha2` | Hashing, likely for image or container identity |

## Integration with the Workspace

`librefang-runtime` re-exports this crate at its historical path:

```rust
// Downstream code continues to work unchanged:
use librefang_runtime::docker_sandbox;
```

The re-export is gated behind the parent crate's **`docker-sandbox` feature**, which is **enabled by default**. Downstream call sites do not need to switch imports.

```
librefang-runtime
  └── (feature: docker-sandbox, default on)
        └── re-exports as runtime::docker_sandbox
              └── librefang-runtime-sandbox-docker
```

## Testing

The critical invariant for this crate is **denylist parity** with the parent `librefang-runtime` crate. The test at `crates/librefang-runtime/tests/docker_sandbox_helpers_parity.rs` asserts that both crates reject the same set of shell metacharacters, preventing a situation where the extracted module is more or less permissive than the original.