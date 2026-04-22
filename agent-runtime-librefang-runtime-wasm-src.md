# Agent Runtime — librefang-runtime-wasm-src

# Agent Runtime — `librefang-runtime-wasm-src`

WASM skill sandbox and host function dispatch for LibreFang. Executes untrusted agent skills/plugins inside Wasmtime with deny-by-default capability enforcement, CPU fuel metering, wall-clock epoch timeouts, and hardened host function implementations.

## Architecture

```mermaid
graph TD
    K[Kernel] -->|wasm_bytes, input, config| S[WasmSandbox]
    S -->|spawn_blocking| X[execute_sync]
    X -->|compile + instantiate| G[Guest Module]
    G -->|librefang::host_call| H[dispatch]
    H --> C{check_capability}
    C -->|denied| ERR[{"error": "denied"}]
    C -->|granted| FN[Host Function]
    FN --> FS[fs_read / fs_write / fs_list]
    FN --> NET[net_fetch]
    FN --> SH[shell_exec]
    FN --> KV[kv_get / kv_set]
    FN --> AG[agent_send / agent_spawn]
    FN --> TM[time_now]
```

The sandbox is split into two tightly coupled files:

| File | Responsibility |
|---|---|
| `sandbox.rs` | Wasmtime engine lifecycle, guest memory management, host function registration, fuel/epoch enforcement, watchdog timer |
| `host_functions.rs` | Per-method capability gates, path traversal protection, SSRF protection, shell env sanitization, kernel delegation |

## Sandbox Lifecycle

`WasmSandbox` is created once per kernel and reused across invocations. The `Engine` is expensive to construct but compiles/instantiates many modules.

```
WasmSandbox::new()
    → Engine with fuel + epoch_interruption enabled

sandbox.execute(wasm_bytes, input, config, kernel, agent_id)
    → spawn_blocking → execute_sync
        → Module::new (compile .wasm or .wat)
        → Store<GuestState> with fuel budget + epoch deadline
        → spawn watchdog thread (park_timeout + AtomicBool RAII guard)
        → Linker with host imports (no WASI)
        → instantiate
        → write input to guest memory via alloc()
        → call guest execute(ptr, len)
        → read output from guest memory
        → signal watchdog, join thread
        → return ExecutionResult { output, fuel_consumed }
```

### Resource Limits

| Mechanism | Config Field | Default | Purpose |
|---|---|---|---|
| Fuel metering | `SandboxConfig::fuel_limit` | 1,000,000 | Deterministic instruction-count cap. Traps `Trap::OutOfFuel`. |
| Epoch interruption | `SandboxConfig::timeout_secs` | 30 | Wall-clock cap. Watchdog thread calls `Engine::increment_epoch` on expiry, guest traps `Trap::Interrupt`. |
| Memory | `SandboxConfig::max_memory_bytes` | 16 MiB | Reserved for future enforcement (Wasmtime linear memory is guest-declared). |

### Watchdog Thread Design

The watchdog avoids the thread-leak and false-interrupt bugs that plagued earlier fire-and-forget implementations:

- Uses `park_timeout(deadline - now)` in a loop rather than a bare `sleep`.
- An `AtomicBool` done flag (Release/Acquire ordering) is set by an RAII `WatchdogGuard` on every exit path (`?`, trap, panic via drop).
- The guard calls `Thread::unpark()` then `JoinHandle::join()`, so the OS thread is reclaimed immediately on the happy path.
- Because `Engine::increment_epoch` is global, the guard ensures it is never called after a clean return — preventing false interrupts on concurrent stores.

## Guest ABI

WASM modules must export three symbols:

| Export | Signature | Purpose |
|---|---|---|
| `memory` | Linear memory | Shared memory for data transfer |
| `alloc` | `(size: i32) -> i32` | Bump allocator; returns pointer to `size` bytes |
| `execute` | `(input_ptr: i32, input_len: i32) -> i64` | Entry point. Receives JSON input bytes, returns packed `(ptr << 32) \| len` pointing to JSON output. |

### Minimal Guest (WAT)

```wat
(module
    (memory (export "memory") 1)
    (global $bump (mut i32) (i32.const 1024))

    (func (export "alloc") (param $size i32) (result i32)
        (local $ptr i32)
        (local.set $ptr (global.get $bump))
        (global.set $bump (i32.add (global.get $bump) (local.get $size)))
        (local.get $ptr)
    )

    (func (export "execute") (param $ptr i32) (param $len i32) (result i64)
        (i64.or
            (i64.shl (i64.extend_i32_u (local.get $ptr)) (i64.const 32))
            (i64.extend_i32_u (local.get $len))))
)
```

## Host ABI

The sandbox provides two imports under the `"librefang"` module:

### `host_call(request_ptr: i32, request_len: i32) -> i64`

Single RPC dispatch for all capability-checked operations.

**Request** (JSON in guest memory):
```json
{"method": "fs_read", "params": {"path": "/data/input.txt"}}
```

**Response** (packed pointer to JSON in guest memory):
```json
{"ok": "file contents here"}
```
or
```json
{"error": "Capability denied: FileRead(\"/data/input.txt\")"}
```

The host allocates response memory by calling back into the guest's `alloc` export.

### `host_log(level: i32, msg_ptr: i32, msg_len: i32)`

Lightweight logging with no capability check. Levels: 0=trace, 1=debug, 2=info, 3=warn, ≥4=error. Messages are tagged with `[wasm]` and the agent ID.

## Capability Enforcement

Every host function (except `time_now`) calls `check_capability` before executing. Capabilities are matched via `librefang_types::capability::capability_matches`, which supports wildcard patterns (e.g., `FileRead("*")`).

| Host Method | Required Capability | Parameters |
|---|---|---|
| `time_now` | *(none — always allowed)* | — |
| `fs_read` | `FileRead(path)` | `{"path": "..."}` |
| `fs_write` | `FileWrite(path)` | `{"path": "...", "content": "..."}` |
| `fs_list` | `FileRead(path)` | `{"path": "..."}` |
| `net_fetch` | `NetConnect(host:port)` | `{"url": "...", "method": "GET", "body": ""}` |
| `shell_exec` | `ShellExec(command)` | `{"command": "...", "args": [...]}` |
| `env_read` | `EnvRead(name)` | `{"name": "..."}` |
| `kv_get` | `MemoryRead(key)` | `{"key": "..."}` |
| `kv_set` | `MemoryWrite(key)` | `{"key": "...", "value": ...}` |
| `agent_send` | `AgentMessage(target)` | `{"target": "...", "message": "..."}` |
| `agent_spawn` | `AgentSpawn` | `{"manifest": "..."}` |

### Capability Inheritance

`agent_spawn` calls `kernel.spawn_agent_checked(manifest, parent_id, &parent_capabilities)`. The kernel enforces that the child's capabilities are a subset of the parent's — a spawned agent can never exceed its spawner's permissions.

## Security Hardening

### Path Traversal Protection

Two functions guard all filesystem access:

- **`safe_resolve_path`** — for reads/lists where the target must already exist. Rejects any `..` component, then `canonicalize`s to resolve symlinks.
- **`safe_resolve_parent`** — for writes where the file may not exist yet. Canonicalizes the parent directory, validates the filename has no `..`, and joins them.

Both use `checked_add` on pointer arithmetic to prevent overflow on 32-bit hosts.

### SSRF Protection

`host_net_fetch` validates every URL through `is_ssrf_target`:

1. **Scheme allowlist** — only `http://` and `https://` (blocks `file://`, `gopher://`, `ftp://`).
2. **Hostname blocklist** — `localhost`, `metadata.google.internal`, `metadata.aws.internal`, `instance-data`, `169.254.169.254`.
3. **DNS resolution** — resolves the hostname and checks every returned IP.
4. **IP canonicalization** — `canonical_ip` unwraps IPv4-mapped IPv6 (`::ffff:10.0.0.1` → `10.0.0.1`) before checking. Blocks loopback, unspecified, RFC 1918 (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), link-local (`169.254.0.0/16`), and IPv6 unique local / link-local.
5. **DNS pinning** — the resolved addresses are pinned into the HTTP client via `librefang_http::proxied_client_builder().resolve()`, preventing DNS-rebinding TOCTOU attacks.

### Shell Environment Sanitization

`host_shell_exec` does **not** use a shell (`Command::new` passes args directly to the process — no shell injection). The child environment is stripped via `sanitize_shell_env`:

- Calls `env_clear()` on the child `Command`.
- Re-adds only `PATH`, `HOME`, `TMPDIR`, `TMP`, `TEMP`, `LANG`, `LC_ALL`, `TERM`.
- On Windows: also `USERPROFILE`, `SYSTEMROOT`, `APPDATA`, `LOCALAPPDATA`, `COMSPEC`, `WINDIR`, `PATHEXT`.

This prevents exfiltration of LLM provider API keys, vault tokens, and cloud metadata credentials that may exist in the daemon's environment.

### Memory Safety

All guest-to-host and host-to-guest memory copies use `checked_add` on `ptr + len` to prevent integer overflow, followed by a bounds check against the actual memory size. This is defensive on 64-bit hosts but reachable on 32-bit.

## Key Types

### `WasmSandbox`

```rust
pub struct WasmSandbox {
    engine: Engine,  // expensive, reuse across invocations
}
```

Created once. Holds the Wasmtime `Engine` with fuel and epoch interruption configured. Call `execute()` per skill invocation.

### `SandboxConfig`

```rust
pub struct SandboxConfig {
    pub fuel_limit: u64,           // default: 1_000_000
    pub max_memory_bytes: usize,   // default: 16 MiB (reserved)
    pub capabilities: Vec<Capability>,
    pub timeout_secs: Option<u64>, // default: 30s
}
```

Passed per invocation. Different skills can receive different capability sets and budgets.

### `GuestState`

```rust
pub struct GuestState {
    pub capabilities: Vec<Capability>,
    pub kernel: Option<Arc<dyn KernelHandle>>,
    pub agent_id: String,
    pub tokio_handle: tokio::runtime::Handle,
}
```

Stored inside the Wasmtime `Store`. Accessible to host functions via `Caller<GuestState>`. The `kernel` handle is required for `kv_get`, `kv_set`, `agent_send`, and `agent_spawn` — these return `{"error": "No kernel handle available"}` when absent.

### `ExecutionResult`

```rust
pub struct ExecutionResult {
    pub output: serde_json::Value,
    pub fuel_consumed: u64,
}
```

### `SandboxError`

| Variant | Meaning |
|---|---|
| `Compilation(String)` | Module compilation failed |
| `Instantiation(String)` | Linking/instantiation failed |
| `Execution(String)` | Runtime error (includes epoch timeout message) |
| `FuelExhausted` | Guest exceeded `fuel_limit` instructions |
| `AbiError(String)` | Missing exports, invalid JSON, bounds violations |

## Dispatch Table

`host_functions::dispatch` is the central router. Given a method string and JSON params, it delegates to the appropriate handler:

```
dispatch(state, method, params)
  ├─ "time_now"       → host_time_now()                    // no capability check
  ├─ "fs_read"        → host_fs_read(state, params)        // FileRead
  ├─ "fs_write"       → host_fs_write(state, params)       // FileWrite
  ├─ "fs_list"        → host_fs_list(state, params)        // FileRead
  ├─ "net_fetch"      → host_net_fetch(state, params)      // NetConnect + SSRF
  ├─ "shell_exec"     → host_shell_exec(state, params)     // ShellExec + env sanitize
  ├─ "env_read"       → host_env_read(state, params)       // EnvRead
  ├─ "kv_get"         → host_kv_get(state, params)         // MemoryRead + kernel
  ├─ "kv_set"         → host_kv_set(state, params)         // MemoryWrite + kernel
  ├─ "agent_send"     → host_agent_send(state, params)     // AgentMessage + kernel
  ├─ "agent_spawn"    → host_agent_spawn(state, params)    // AgentSpawn + kernel
  └─ _                → {"error": "Unknown host method: ..."}
```

## Dependencies on Other Crates

| Crate | Usage |
|---|---|
| `librefang_types` | `Capability` enum, `capability_matches` |
| `librefang_kernel_handle` | `KernelHandle` trait — `memory_recall`, `memory_store`, `send_to_agent`, `spawn_agent_checked` |
| `librefang_http` | `proxied_client_builder` for DNS-pinned HTTP client |

## Adding a New Host Function

1. Define the capability variant in `librefang_types::capability`.
2. Add a handler function in `host_functions.rs` following the existing pattern: extract params → `check_capability` → execute → return `json!({"ok": ...})` or `json!({"error": ...})`.
3. Add a match arm in `dispatch`.
4. Write tests covering: denied (no capability), granted (with capability), missing parameters, edge cases.