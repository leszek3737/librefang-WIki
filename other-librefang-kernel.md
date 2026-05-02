# Other — librefang-kernel

# librefang-kernel

The central orchestration crate for the LibreFang Agent OS. It assembles the various subsystems into a coherent kernel that drives the agent's lifecycle — from LLM interaction and skill dispatch to extension management, metering, and inter-process communication.

## Architecture

`librefang-kernel` sits at the top of the dependency graph. It does not implement subsystem logic itself; instead, it wires together the following specialized crates:

```
                    ┌─────────────────────┐
                    │  librefang-kernel   │
                    └─────────┬───────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   ┌──────────────┐   ┌──────────────┐   ┌───────────────┐
   │ kernel-router│   │kernel-metering│   │   runtime     │
   └──────────────┘   └──────────────┘   └───────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
   ┌──────────────┐   ┌──────────────┐   ┌───────────────┐
   │    skills    │   │  llm-driver  │   │   channels    │
   └──────────────┘   └──────────────┘   └───────────────┘
          │                   │                   │
          ▼                   ▼                   ▼
   ┌──────────────┐   ┌──────────────┐   ┌───────────────┐
   │    hands     │   │  extensions  │   │     wire      │
   └──────────────┘   └──────────────┘   └───────────────┘
          │                                       │
          ▼                                       ▼
   ┌──────────────┐                       ┌───────────────┐
   │   memory     │                       │    types      │
   └──────────────┘                       └───────────────┘
```

## Subsystem Responsibilities

| Crate | Role within the kernel |
|---|---|
| `librefang-kernel-router` | Routes incoming requests to the appropriate handler |
| `librefang-kernel-metering` | Tracks resource consumption and usage metrics |
| `librefang-runtime` | Manages the async runtime and agent execution lifecycle |
| `librefang-skills` | Registers and dispatches agent skills |
| `librefang-hands` | Executes concrete actions (tool use, I/O) |
| `librefang-extensions` | Loads and manages extensions that augment agent behavior |
| `librefang-llm-driver` | Abstracts communication with language model providers |
| `librefang-wire` | Defines the wire protocol for inter-service messaging |
| `librefang-channels` | Provides communication channels (feature-gated, default features disabled) |
| `librefang-memory` | Manages conversation and persistent memory storage |
| `librefang-types` | Shared type definitions used across all crates |

## Key External Dependencies

The kernel pulls in several categories of functionality through workspace dependencies:

**Persistence & storage**
- `rusqlite` — SQLite for local structured storage
- `dashmap` — Concurrent hash maps for in-memory state
- `arc-swap` — Atomic swapping of shared references (hot-reloadable configuration)

**Serialization & configuration**
- `serde`, `serde_json`, `toml`, `serde_yaml` — Multi-format config and data handling

**Security & cryptography**
- `sha2` — SHA-256 hashing
- `subtle` — Constant-time comparisons (credential verification)
- `totp-rs` — Time-based one-time password generation
- `zeroize` — Secure memory clearing for sensitive data
- `hex`, `rand` — Hex encoding and cryptographic randomness

**Scheduling & time**
- `cron` — Cron-expression based task scheduling
- `chrono`, `chrono-tz` — Timezone-aware timestamp handling

**Networking**
- `reqwest` — HTTP client for outbound API calls
- `regex`, `regex-lite` — Pattern matching for routing and input validation

**Platform-specific**
- `libc` (Unix only) — Low-level system calls on Unix platforms

## Binary: `purge_sentinels`

A dedicated CLI tool (`bin/purge_sentinels.rs`) for cleaning up sentinel artifacts — likely lock files, marker files, or temporary state left by the kernel during operation. Run independently of the main agent process for maintenance.

## Testing

The crate uses several testing utilities:
- `tokio-test` — Async test helpers
- `tempfile` — Temporary directory/file fixtures
- `serial_test` — Serialized test execution (for tests sharing stateful resources like SQLite)
- `librefang-testing` — Project-specific test harnesses and fixtures
- `librefang-kernel-handle` — Provides kernel handles for integration testing without a full agent bootstrap

Tests that touch SQLite or the filesystem should be annotated with `#[serial_test::serial]` to avoid concurrency issues.

## Relationship to the Rest of the Codebase

`librefang-kernel` is a **consumer** crate — it depends on every other `librefang-*` module but nothing depends on it in return. It acts as the composition root where all subsystems are instantiated, configured, and connected. To add a new subsystem:

1. Implement the logic in its own `librefang-*` crate.
2. Add the crate as a dependency in this crate's `Cargo.toml`.
3. Wire it into the kernel's initialization and runtime loop.