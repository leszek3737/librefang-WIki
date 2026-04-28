# Other — librefang-kernel

# librefang-kernel

The central orchestration crate for the **LibreFang Agent OS**. This is the top-level kernel that wires together all subsystems — routing, metering, runtime, skills, hands, extensions, LLM drivers, communication channels, and persistence — into a single cohesive agent process.

## Role in the Architecture

`librefang-kernel` is the **composition root**. It does not implement domain logic itself. Instead, it owns initialization, lifecycle management, and the glue that connects sibling crates:

```
                    ┌─────────────────────┐
                    │  librefang-kernel   │
                    │  (this crate)       │
                    └────────┬────────────┘
           ┌─────────────────┼─────────────────┐
           │                 │                  │
     ┌─────▼─────┐    ┌─────▼─────┐     ┌─────▼──────┐
     │  router   │    │ metering  │     │  runtime   │
     └─────┬─────┘    └─────┬─────┘     └─────┬──────┘
           │                │                  │
    ┌──────┼────┐           │           ┌──────┼──────┐
    │      │    │           │           │      │      │
┌───▼──┐┌──▼──┐│     ┌─────▼────┐ ┌────▼──┐┌──▼──┐┌─▼────────┐
│skills││hands││     │  memory  │ │  wire ││ llm ││extensions│
└──────┘└─────┘│     └──────────┘ └───────┘└─────┘└──────────┘
         ┌─────▼───────┐
         │  channels   │
         └─────────────┘
```

Every other crate in the workspace is a leaf dependency that provides a specific capability. The kernel decides **when** and **how** those capabilities are used.

## Key Responsibilities

| Area | Description |
|---|---|
| **Initialization** | Bootstraps the database (via `rusqlite`), loads configuration (TOML/YAML/JSON), and constructs all subsystem instances in dependency order. |
| **Message routing** | Delegates to `librefang-kernel-router` to dispatch incoming messages from channels to the appropriate handler. |
| **Metering** | Delegates to `librefang-kernel-metering` for usage tracking, rate limiting, or quota enforcement. |
| **Agent lifecycle** | Manages the runtime loop — receiving input, invoking the LLM via `librefang-llm-driver`, executing skills via `librefang-skills`, and performing actions via `librefang-hands`. |
| **Scheduled tasks** | Uses the `cron` crate to run periodic or time-based jobs (e.g., cleanup, health checks, sentinel purging). |
| **Authentication** | Integrates `totp-rs` and `subtle` (constant-time comparison) for TOTP-based multi-factor authentication, and `zeroize` to securely clear secrets from memory. |
| **Persistence** | Stores state in SQLite (`rusqlite`) with concurrent access managed through `dashmap` and `arc-swap` for lock-free hot-reload of shared configuration. |
| **Communication** | Sends and receives messages over the wire protocol (`librefang-wire`) through pluggable channel backends (`librefang-channels`). |

## Dependency Breakdown

### Workspace sibling crates

| Crate | Purpose |
|---|---|
| `librefang-types` | Shared data structures, error types, and domain primitives used across all crates. |
| `librefang-memory` | Short-term and long-term memory subsystem for the agent's conversational and factual context. |
| `librefang-kernel-router` | Message routing logic — maps incoming events to the correct handler or skill. |
| `librefang-kernel-metering` | Usage metering, quota tracking, and rate-limiting hooks. |
| `librefang-runtime` | The async runtime layer and task orchestration primitives. |
| `librefang-skills` | Pluggable skill system — discrete capabilities the agent can invoke (e.g., web search, code execution). |
| `librefang-hands` | The action/motor layer — executes side effects the agent decides to perform. |
| `librefang-extensions` | Extension loading and lifecycle for third-party or user-provided plugins. |
| `librefang-llm-driver` | Abstracted LLM backend — sends prompts, receives completions, handles streaming. |
| `librefang-wire` | Wire protocol definitions for inter-service and client-server communication. |
| `librefang-channels` | Channel backends (default features disabled — only the features the kernel selects are compiled in). |

### Notable external libraries

| Library | Why it's here |
|---|---|
| `rusqlite` | Embedded SQLite for persistent state. |
| `dashmap` / `arc-swap` | Lock-free concurrent maps and atomically-swapped shared state. |
| `cron` | Cron-expression parsing and scheduling for periodic tasks. |
| `totp-rs` | TOTP generation and validation for 2FA. |
| `subtle` | Constant-time comparisons to prevent timing attacks on auth tokens. |
| `zeroize` | Securely zeroes secrets from memory when they are dropped. |
| `chrono` / `chrono-tz` | Timezone-aware timestamp handling. |
| `reqwest` | Outbound HTTP client (for extensions, webhooks, or API calls). |
| `regex` / `regex-lite` | Pattern matching in message routing or input validation. |
| `libc` (Unix only) | Low-level Unix system calls (likely for signal handling or process management). |

## Binary: `purge_sentinels`

Located at `bin/purge_sentinels.rs`.

This is a standalone maintenance binary, not part of the main agent process. It is responsible for cleaning up stale sentinel records — likely watchdog entries, lock files, or lease markers that were not gracefully released (e.g., after a crash).

Run it directly:

```bash
# Via cargo
cargo run --bin purge_sentinels

# Or the compiled binary
./purge_sentinels
```

The binary reads the same configuration as the main kernel to locate the database and identify sentinels eligible for removal.

## Configuration Loading

The kernel supports three configuration formats via its Serde-based loading:

- **TOML** (`toml` crate) — the primary format for agent configuration files.
- **YAML** (`serde_yaml` crate) — alternative for complex or human-edited configs.
- **JSON** (`serde_json` crate) — typically machine-generated or for interchange.

The `dirs` crate is used to resolve platform-appropriate configuration paths (`~/.config/librefang/` on Linux, etc.).

## Thread Safety and Concurrency

The kernel is designed to run on the Tokio async runtime (`tokio`) and uses several concurrency patterns:

- **`DashMap`** for concurrent access to hot data structures (e.g., active sessions, in-flight requests) without a global lock.
- **`ArcSwap`** for atomic hot-reload of configuration or shared read-mostly state — readers never block, writers publish a new `Arc` atomically.
- **`futures`** for composing complex async workflows (e.g., fan-out to multiple skills, concurrent channel reads).

## Development

### Running tests

```bash
cargo test -p librefang-kernel
```

Dev dependencies include `tokio-test` (for async test helpers) and `tempfile` (for creating temporary databases and config files in isolation).

### Adding a new subsystem

1. Create a new `librefang-<name>` crate in the workspace.
2. Add it as a dependency in this crate's `Cargo.toml`.
3. Initialize it during kernel bootstrap (respecting dependency order).
4. Wire it into the router if it handles incoming messages.

## Platform Notes

On Unix targets, `libc` is included for low-level operations. The kernel may behave differently on non-Unix platforms in areas like signal handling or process management. The `cfg(unix)` gate ensures `libc` is only compiled where needed.