# Other — librefang-runtime

# librefang-runtime

Agent execution engine and tooling layer for LibreFang. This crate owns the turn-by-turn agent loop, tool dispatch, context window management, audit trail, OAuth flows, sandbox environments, and the Agent-to-Agent peer protocol.

## Architecture

```mermaid
graph TD
    API["librefang-api"] -->|calls| RT["librefang-runtime"]
    KERNEL["librefang-kernel"] -->|spawns agents via| RT
    RT -.->|KernelHandle trait| KH["librefang-kernel-handle"]
    KH -.->|implemented by| KERNEL
    RT --> TYPES["librefang-types"]
    RT --> HTTP["librefang-http"]
    RT --> LLM["librefang-llm-drivers"]
    RT --> SKILLS["librefang-skills"]
    RT --> MEMORY["librefang-memory"]
    subgraph Leaf Crates (re-exported)
        AUDIT["librefang-runtime-audit"]
        MEDIA["librefang-runtime-media"]
        DOCKER["librefang-runtime-sandbox-docker"]
        MCP["librefang-runtime-mcp"]
    end
    RT --> AUDIT
    RT -->|feature: media| MEDIA
    RT -->|feature: docker-sandbox| DOCKER
    RT --> MCP
```

The runtime sits between the kernel (which orchestrates agents) and the API layer (which routes HTTP). It never imports `librefang-kernel` or `librefang-api` directly. All kernel communication flows through the `KernelHandle` trait defined in `librefang-kernel-handle`, which the kernel implements and the runtime consumes. This breaks what would otherwise be a circular dependency.

## Public API Entry Points

| Module | Role |
|---|---|
| `agent_loop` | Turn-by-turn agent execution. The primary orchestrator. |
| `tool_runner` | Tool dispatch and execution. |
| `apply_patch` | Patch-application tool. |
| `a2a` | Agent-to-Agent peer protocol. |
| `compactor` | Context compaction for long conversations. |
| `context_budget` | Token budget tracking and allocation. |
| `context_compressor` | Lossy context compression. |
| `context_overflow` | Overflow handling when context exceeds limits. |
| `prompt_builder` | Assembles the final prompt sent to the LLM. |
| `model_catalog` | The `ModelCatalog` type — registry of 130+ models across 28 providers. |
| `mcp` | MCP client re-exported from `librefang-runtime-mcp`. |
| `chatgpt_oauth` / `copilot_oauth` | OAuth flows for ChatGPT and Copilot integrations. |
| `host_functions` / `sandbox` | In-tree WASM host functions and sandbox environment. |
| `channel_registry` | Channel adapter registry. |
| `browser` | Browser sandbox (feature-gated). |
| `subprocess_sandbox` | Subprocess-level sandboxing. |
| `audit` | Audit trail re-exported from `librefang-runtime-audit`. |
| `auth_cooldown` | Rate-limiting for auth attempts. |
| `aux_client` | Auxiliary HTTP client helper. |
| `catalog_sync` | Model catalog synchronization. |
| `checkpoint_manager` | Agent state checkpointing. |
| `dangerous_command` | Dangerous command detection / gating. |
| `media` / `media_understanding` | Media processing re-exported from `librefang-runtime-media`. |
| `docker_sandbox` | Docker sandbox re-exported from `librefang-runtime-sandbox-docker`. |

Constant: `USER_AGENT` — mandatory on every outbound HTTP request.

## Key Subsystems

### Agent Loop (`agent_loop`)

~10k lines. The god module that drives turn-by-turn execution: receives a message, builds a prompt, calls the LLM, processes the response (which may include tool calls), dispatches tools, feeds results back, and repeats until the agent produces a final response or hits a stop condition.

**Do not grow this module.** Issue #3710 tracks its extraction into smaller, focused units. Any new functionality belongs in a sibling file or a new submodule.

### Tool Runner (`tool_runner`)

~9.7k lines. The tool execution path. When the LLM response contains tool calls, `tool_runner` dispatches each call to the appropriate handler, captures output, and returns it to the agent loop.

**Same constraint as `agent_loop`:** don't grow `tool_runner/mod.rs`. New tool kinds get their own sibling file inside `tool_runner/`.

### Model Catalog (`model_catalog::ModelCatalog`)

Registry of 130+ models across 28 providers. The kernel wraps this in `arc_swap::ArcSwap` (issue #3384) for lock-free reads. All mutations go through the kernel's `model_catalog_update(|cat| ...)` callback — the runtime never mutates the catalog directly.

### Context Management

Four cooperating modules:

- **`context_budget`** — tracks token budgets per conversation turn.
- **`context_compressor`** — applies lossy compression when the budget is tight.
- **`context_overflow`** — handles cases where context exceeds hard limits.
- **`compactor`** — runs context compaction strategies for long-running conversations.

### MCP Client (`mcp`)

Re-exported from `librefang-runtime-mcp`. OAuth state lives in `mcp_auth_states`. The `McpOAuthProvider` trait is defined on the kernel side; the kernel implements it and passes it through `KernelHandle`.

### A2A Protocol (`a2a`)

Agent-to-Agent peer protocol for inter-agent communication. Allows one agent to invoke another as a tool-like peer.

### Sandboxes

Three sandbox backends:

- **`subprocess_sandbox`** — process-level isolation (always available).
- **`docker_sandbox`** — Docker-based isolation (feature `docker-sandbox`, re-exported from `librefang-runtime-sandbox-docker`).
- **`browser`** — browser sandbox for web-based tool execution (feature `browser`).

Additional sandbox hardening behind optional features: `landlock-sandbox` (Linux Landlock), `seccomp-sandbox` (seccomp-bpf filters).

Remote execution backends: `ssh-backend` (feature-gated, uses `russh`), `daytona-backend` (feature-gated, uses existing reqwest stack).

## Feature Gates

| Feature | Default | Effect when disabled |
|---|---|---|
| `media` | on | `media` / `media_understanding` modules become no-op stubs |
| `browser` | on | `browser` module becomes a no-op stub |
| `docker-sandbox` | on | `docker_sandbox` module becomes a no-op stub |
| `landlock-sandbox` | off | Enables Landlock-based sandbox hardening (Linux only) |
| `seccomp-sandbox` | off | Enables seccomp-bpf sandbox hardening |
| `ssh-backend` | off | Enables SSH remote-exec tool backend |
| `daytona-backend` | off | Enables Daytona managed-sandbox backend |

Build without defaults to produce a minimal runtime:

```bash
cargo build -p librefang-runtime --no-default-features
```

## Cross-Cutting Invariants

### Deterministic Prompt Ordering (#3298)

Tool definitions, MCP server summaries, and capability lists **must** be sorted before stringifying. Use `BTreeMap` / `BTreeSet`, never `HashMap`. Non-deterministic ordering causes flaky tests and prompt diff noise.

### Identity Files

Identity files live at `{workspace}/.identity/`, not at the workspace root. `read_identity_file()` falls back to the root for pre-migration workspaces. `migrate_identity_files()` runs on every agent spawn to handle legacy layouts.

### USER_AGENT on All Outbound HTTP

Every outbound HTTP request must include:

```rust
req.header("User-Agent", librefang_runtime::USER_AGENT)
```

An audit hook flags requests missing this header.

## Async Safety Rules

### `ErrorTranslator` is `!Send`

`ErrorTranslator` (from `RequestLanguage`) cannot be sent across threads. Any `.await` point must come **after** `drop(translator)`, or you get a cryptic axum `Handler<_, _>` trait-bound error at compile time.

### No Blocking I/O in Async Context

Do not use `std::fs` or `std::sync::RwLock` inside async handlers. Use `tokio::fs`, `arc_swap::ArcSwap`, or `parking_lot` primitives instead (issue #3579). Never call `tokio::runtime::Handle::block_on`.

## Dependency Boundaries

### Allowed

- `librefang-types`
- `librefang-http`
- `librefang-kernel-handle` (the trait crate — NOT `librefang-kernel`)
- `librefang-runtime-audit`, `librefang-runtime-mcp`, `librefang-runtime-media`, `librefang-runtime-sandbox-docker` (leaf crates)
- `librefang-llm-drivers`, `librefang-llm-driver`
- `librefang-channels`, `librefang-memory`, `librefang-skills`

### Forbidden

- `librefang-kernel` — would create a circular dependency. Use `KernelHandle`.
- `librefang-api` — API consumes runtime, never the reverse.

### KernelHandle Trait

Defined in `librefang-kernel-handle`. The kernel implements it; the runtime and API consume it. Any time runtime code needs a kernel callback (persisting state, updating the model catalog, fetching configuration), it goes through `KernelHandle`.

## Testing

This crate historically had zero integration tests (issue #3696). New runtime work **should** include at least one `#[tokio::test]` exercising the new code path.

Run tests:

```bash
cargo test -p librefang-runtime
```

When mocking the kernel in tests, use `librefang-testing::MockKernelBuilder`. Do not fake `KernelHandle` inline — that bypasses the trait contract and hides real integration issues.

## Taboos

- **No `librefang-kernel` import.** Use `KernelHandle`.
- **No `librefang-api` import.** The dependency arrow points one way.
- **No growing `agent_loop/` or `tool_runner/` `mod.rs`.** Issue #3710 constrains them to their current shape. New tool kinds go in new files under `tool_runner/`.
- **No `unwrap()` or `panic!()` on wire data.** Values from LLM responses, tool outputs, and HTTP bodies must be handled with `?`, `map_err`, or explicit fallbacks.
- **No mocking `KernelHandle` inline.** Use `librefang-testing::MockKernelBuilder`.
- **No raw `cargo build`.** Use `cargo check --workspace --lib`. Full builds run in CI.