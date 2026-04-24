# Agent Runtime — librefang-runtime-wasm-src

# Agent Runtime — `librefang-runtime-wasm`

WASM sandbox for executing untrusted agent skills and plugins with deny-by-default capability-based access control.

## Overview

This crate provides a Wasmtime-based sandbox that runs agent skills compiled to WASM. Every operation a guest module attempts — filesystem access, network requests, shell execution, inter-agent messaging — is gated behind an explicit capability check. Nothing is allowed unless the caller grants it.

The sandbox defends against both accidental misuse and deliberate exploitation through layered security controls:

- **Capability gating** — every host function checks `check_capability` before acting
- **Path traversal protection** — `..` components are rejected outright; symlinks are canonicalized
- **SSRF protection** — DNS is resolved once, validated against private IP ranges (including IPv4-mapped IPv6), then pinned to prevent rebinding attacks
- **Shell environment sanitization** — child processes inherit only a hardcoded allowlist of safe env vars, never API keys or vault tokens
- **Fuel metering** — deterministic CPU instruction budget per invocation
- **Epoch-based wall-clock timeouts** — with an RAII-guarded watchdog thread that won't leak or cause cross-store false interrupts

## Architecture

```mermaid
graph TD
    Kernel["Kernel / Runtime"] -->|"wasm_bytes + config + input"| Sandbox["WasmSandbox"]
    Sandbox -->|"spawn_blocking"| ExecSync["execute_sync"]
    ExecSync -->|"compile + instantiate"| Wasmtime["Wasmtime Engine"]
    Wasmtime -->|"import 'librefang'"| HostCall["host_call"]
    Wasmtime -->|"import 'librefang'"| HostLog["host_log"]
    HostCall --> Dispatch["dispatch"]
    Dispatch --> CapCheck["check_capability"]
    CapCheck -->|granted| Handlers["fs_read / net_fetch / shell_exec / kv_* / ..."]
    CapCheck -->|denied| ErrorJSON["{\"error\": \"Capability denied: ...\"}"]
    Handlers --> KernelHandle["KernelHandle (optional)"]
```

## Guest ABI

WASM modules must export three symbols:

| Export | Signature | Purpose |
|--------|-----------|---------|
| `memory` | `(memory 1+)` | Linear memory for data exchange |
| `alloc` | `(func (param i32) (result i32))` | Bump allocator — guest receives a byte count, returns a pointer |
| `execute` | `(func (param i32 i32) (result i64))` | Main entry point — receives `(input_ptr, input_len)`, returns packed `(result_ptr << 32 \| result_len)` |

The `execute` function receives its input as JSON bytes written by the host into guest memory. It returns its output the same way: a packed `i64` where the high 32 bits are a pointer into guest memory and the low 32 bits are the byte length of the JSON result.

## Host ABI

The host injects two functions into the `"librefang"` import module:

### `host_call(request_ptr: i32, request_len: i32) -> i64`

Single RPC dispatch point for all capability-checked operations. The request is JSON:

```json
{"method": "fs_read", "params": {"path": "/data/config.toml"}}
```

The response is a packed pointer to JSON — either `{"ok": ...}` or `{"error": "..."}`.

### `host_log(level: i32, msg_ptr: i32, msg_len: i32)`

Lightweight logging with no capability check. Levels: 0=trace, 1=debug, 2=info, 3=warn, 4+=error.

## Sandbox Lifecycle

`WasmSandbox::execute` is `async` but offloads all CPU-bound work to `spawn_blocking`:

1. **Compile** — Wasmtime compiles the `.wasm` or `.wat` bytes into a `Module`
2. **Create store** — a Wasmtime `Store<GuestState>` is created with the guest's capabilities, kernel handle, agent ID, and tokio handle
3. **Set budgets** — fuel is set for instruction-level metering; epoch deadline is set to 1 for wall-clock timeout
4. **Start watchdog** — a dedicated OS thread sleeps until the timeout, then calls `Engine::increment_epoch`. An RAII `WatchdogGuard` signals completion and joins the thread on every exit path (success, error, panic)
5. **Instantiate** — the module is linked against host function imports (no WASI)
6. **Validate exports** — `memory`, `alloc`, and `execute` must all be present
7. **Write input** — input JSON is serialized, `alloc` is called in the guest, bytes are copied into guest memory with overflow-checked bounds
8. **Execute** — the guest's `execute` function runs. Fuel exhaustion → `SandboxError::FuelExhausted`. Epoch interrupt → timeout error
9. **Read output** — the packed return value is unpacked, output JSON is read from guest memory with overflow-checked bounds
10. **Report** — returns `ExecutionResult { output, fuel_consumed }`

### Watchdog thread design

The watchdog thread is not fire-and-forget. It uses `park_timeout` to sleep until the deadline, and the `WatchdogGuard::drop` implementation:

1. Stores `done = true` with `Release` ordering
2. Calls `thread::unpark()` to wake the watchdog early
3. Joins the thread to reclaim the OS resource

This avoids thread leaks on the happy path (a 5 ms call doesn't burn a sleeping thread for 30 seconds) and prevents cross-store false interrupts that occurred when the old design called `increment_epoch` globally after every timeout regardless of which store was actually running.

## Host Functions Reference

All functions are dispatched through `dispatch(state, method, params)` in `host_functions.rs`.

### Always allowed (no capability check)

| Method | Parameters | Returns |
|--------|-----------|---------|
| `time_now` | — | `{"ok": <unix_timestamp_secs>}` |

### Filesystem — requires `FileRead` / `FileWrite`

| Method | Parameters | Capability | Notes |
|--------|-----------|------------|-------|
| `fs_read` | `path` | `FileRead(path)` | Path traversal checked after capability gate |
| `fs_write` | `path`, `content` | `FileWrite(path)` | Uses `safe_resolve_parent` for new files |
| `fs_list` | `path` | `FileRead(path)` | Returns array of filenames |

### Network — requires `NetConnect`

| Method | Parameters | Capability | Notes |
|--------|-----------|------------|-------|
| `net_fetch` | `url`, `method?`, `body?` | `NetConnect(host:port)` | SSRF-validated, DNS-pinned HTTP client |

### Shell — requires `ShellExec`

| Method | Parameters | Capability | Notes |
|--------|-----------|------------|-------|
| `shell_exec` | `command`, `args?` | `ShellExec(command)` | No shell invocation; args passed directly. Env is stripped to safe allowlist. |

### Environment — requires `EnvRead`

| Method | Parameters | Capability | Notes |
|--------|-----------|------------|-------|
| `env_read` | `name` | `EnvRead(name)` | Returns `null` if variable is unset |

### Memory KV — requires `MemoryRead` / `MemoryWrite`

| Method | Parameters | Capability | Kernel Required | Notes |
|--------|-----------|------------|-----------------|-------|
| `kv_get` | `key` | `MemoryRead(key)` | Yes | Delegates to `kernel.memory_recall` |
| `kv_set` | `key`, `value` | `MemoryWrite(key)` | Yes | Delegates to `kernel.memory_store` |

### Agent interaction — requires `AgentMessage` / `AgentSpawn`

| Method | Parameters | Capability | Kernel Required | Notes |
|--------|-----------|------------|-----------------|-------|
| `agent_send` | `target`, `message` | `AgentMessage(target)` | Yes | Async via `kernel.send_to_agent` |
| `agent_spawn` | `manifest` | `AgentSpawn` | Yes | Capability inheritance enforced: child ≤ parent |

## Security Deep Dives

### Capability checking

Every host function (except `time_now`) calls `check_capability` before executing. The function iterates the guest's granted capabilities and uses `capability_matches` from `librefang_types` to support wildcards (e.g., `FileRead("*")` grants read access to any path).

If no match is found, the operation is rejected immediately with `{"error": "Capability denied: ..."}`.

### Path traversal protection

Two functions protect filesystem operations:

- **`safe_resolve_path`** — for reads where the file must exist. Rejects any `..` component, then `canonicalize`s to resolve symlinks
- **`safe_resolve_parent`** — for writes where the file may not yet exist. Canonicalizes the parent directory and validates the filename. Double-checks that the filename itself doesn't contain `..`

Both use `checked_add` on pointer arithmetic to prevent overflow on 32-bit hosts.

### SSRF protection

`is_ssrf_target` performs three-layer validation before any HTTP request:

1. **Scheme allowlist** — only `http://` and `https://` are permitted
2. **Hostname blocklist** — blocks `localhost`, `metadata.google.internal`, `metadata.aws.internal`, `instance-data`, `169.254.169.254`
3. **IP validation** — DNS is resolved, every returned address is checked against loopback, unspecified, and private ranges

IPv4-mapped IPv6 addresses (`::ffff:X.X.X.X`) are unwrapped via `canonical_ip` before the private-range checks, closing the bypass where mapped addresses appeared "public" to the V6 check branch.

The resolved addresses are pinned into the HTTP client via `resolve()` to prevent DNS-rebinding TOCTOU attacks — the actual HTTP connection goes to the same IPs that were validated.

### Shell environment sanitization

`sanitize_shell_env` clears the child process environment and re-adds only:

```
PATH, HOME, TMPDIR, TMP, TEMP, LANG, LC_ALL, TERM
```

On Windows, additional safe variables are preserved: `USERPROFILE`, `SYSTEMROOT`, `APPDATA`, `LOCALAPPDATA`, `COMSPEC`, `WINDIR`, `PATHEXT`.

This prevents a WASM guest with `ShellExec` capability from exfiltrating daemon secrets (LLM provider keys, vault tokens, cloud metadata) through the child process environment.

## Configuration

`SandboxConfig` controls each execution:

| Field | Default | Description |
|-------|---------|-------------|
| `fuel_limit` | `1_000_000` | Max WASM instructions. 0 = unlimited |
| `max_memory_bytes` | `16 MiB` | Reserved for future memory limit enforcement |
| `capabilities` | `[]` | Granted capabilities — deny-by-default |
| `timeout_secs` | `30` | Wall-clock timeout via epoch interruption |

## Error Handling

`SandboxError` covers the full lifecycle:

| Variant | When |
|---------|------|
| `Compilation` | Wasmtime fails to compile the module |
| `Instantiation` | Linker can't resolve imports or instantiate |
| `Execution` | Generic runtime error, epoch timeout, or spawn_blocking failure |
| `FuelExhausted` | Guest consumed all fuel before returning |
| `AbiError` | Missing exports, invalid JSON output, pointer overflow, or bounds violation |

Host functions always return JSON — either `{"ok": ...}` on success or `{"error": "..."}` on any failure. The `host_call` trapper in `sandbox.rs` catches panics/errors from the dispatch path and returns a JSON error to the guest rather than crashing the sandbox.

## External Dependencies

| Crate | Usage |
|-------|-------|
| `wasmtime` | WASM compilation, instantiation, fuel metering, epoch interruption |
| `librefang_types` | `Capability` enum and `capability_matches` |
| `librefang_kernel_handle` | `KernelHandle` trait for inter-agent and KV operations |
| `librefang_http` | `proxied_client_builder` for DNS-pinned HTTP client |
| `serde_json` | All input/output serialization |
| `tokio` | Async runtime; `block_on` used in sync host functions for kernel calls |