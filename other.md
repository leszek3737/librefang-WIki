# Other

# Other — Supporting Infrastructure & Cross-Cutting Modules

This module group contains the bulk of the LibreFang Agent OS: everything from shared types and the kernel core through to client-facing applications, specialized subsystems, and their test suites.

## Architecture Overview

```mermaid
graph TD
    subgraph "Clients"
        CLI["librefang-cli"]
        DESK["librefang-desktop"]
        ACP["librefang-acp"]
    end

    subgraph "API Surface"
        API["librefang-api"]
        DASH["librefang-api-dashboard"]
        STATIC["librefang-api-static"]
    end

    subgraph "Kernel & Runtime"
        KERNEL["librefang-kernel"]
        KH["librefang-kernel-handle"]
        KR["librefang-kernel-router"]
        KM["librefang-kernel-metering"]
        RT["librefang-runtime"]
        AUDIT["librefang-runtime-audit"]
        MCP["librefang-runtime-mcp"]
        MEDIA["librefang-runtime-media"]
        SANDBOX["librefang-runtime-sandbox-docker"]
    end

    subgraph "LLM"
        DRIVER["librefang-llm-driver"]
        DRIVERS["librefang-llm-drivers"]
    end

    subgraph "Memory & Knowledge"
        MEM["librefang-memory"]
        WIKI["librefang-memory-wiki"]
    end

    subgraph "Capabilities"
        HANDS["librefang-hands"]
        SKILLS["librefang-skills"]
        CHANS["librefang-channels"]
        EXT["librefang-extensions"]
    end

    subgraph "Networking & Transport"
        WIRE["librefang-wire"]
        SUBPROC["librefang-subprocess"]
    end

    subgraph "Foundation"
        TYPES["librefang-types"]
        HTTP["librefang-http"]
        TELE["librefang-telemetry"]
        TEST["librefang-testing"]
    end

    CLI --> API
    DESK --> API
    ACP --> KH
    API --> KERNEL
    KERNEL --> RT
    RT --> DRIVER
    DRIVERS --> DRIVER
    RT --> MEM
    RT --> MCP
    RT --> SANDBOX
    RT --> AUDIT
    RT --> MEDIA
    KERNEL --> MEM
    KERNEL --> HANDS
    KERNEL --> SKILLS
    KERNEL --> CHANS
    KERNEL --> EXT
    CHANS --> SUBPROC
    KERNEL --> WIRE
    KERNEL --> KR
    KERNEL --> KM
    DRIVER --> TYPES
    KERNEL --> TYPES
    MEM --> TYPES
    HTTP --> TYPES
    DRIVER --> HTTP
    DRIVERS --> HTTP
end
```

## Layer Breakdown

### Foundation

[librefang-types](librefang-types.md) is the bottom of the dependency DAG — pure shared data structures with no business logic. Every other workspace crate depends on it, and it depends on nothing else in the workspace.

Three cross-cutting utilities sit alongside it: [librefang-http](librefang-http.md) centralizes HTTP client construction (TLS, proxy, certificate configuration) so every crate uses a consistent `reqwest` client; [librefang-telemetry](librefang-telemetry.md) provides a single OpenTelemetry/Prometheus setup for the whole project; and [librefang-testing](librefang-testing.md) supplies shared mock implementations (`MockKernel`, `MockLlmDriver`, test helpers) used across integration suites.

### Kernel & Orchestration

[librefang-kernel](librefang-kernel.md) is the core orchestration crate — agent lifecycles, scheduling, permissions, inter-agent communication, and message dispatch. It does not own the agent loop body or tool dispatch (those live in the runtime).

The kernel's in-process interface is defined by the `KernelHandle` trait in [librefang-kernel-handle](librefang-kernel-handle.md), which decouples consumers from the kernel's concrete implementation. Three specialized kernel subsystems handle routing ([librefang-kernel-router](librefang-kernel-router.md)), cost tracking and quota enforcement ([librefang-kernel-metering](librefang-kernel-metering.md)), and long-horizon RL trajectory export ([librefang-rl-export](librefang-rl-export.md)).

### Runtime & Execution

[librefang-runtime](librefang-runtime.md) owns the agent loop: prompt assembly, LLM invocation, tool dispatch, streaming, error recovery, and session persistence. Several extracted sub-crates handle specific concerns:

- [librefang-runtime-audit](librefang-runtime-audit.md) — tamper-evident SHA-256 hash-chain audit log, persisted to SQLite
- [librefang-runtime-mcp](librefang-runtime-mcp.md) — Model Context Protocol client for dynamic tool discovery and invocation
- [librefang-runtime-media](librefang-runtime-media.md) — provider-agnostic drivers for TTS, image, video, and music generation
- [librefang-runtime-sandbox-docker](librefang-runtime-sandbox-docker.md) — Docker-based OS-level sandboxing for tool execution

### LLM Drivers

The LLM layer is split into trait and implementation: [librefang-llm-driver](librefang-llm-driver.md) defines the `LlmDriver` trait with no concrete providers, while [librefang-llm-drivers](librefang-llm-drivers.md) implements it for Anthropic, OpenAI, Gemini, Groq, and Ollama. This separation keeps test crates from pulling in transitive HTTP/TLS dependencies.

### Memory & Knowledge

[librefang-memory](librefang-memory.md) provides the unified `Memory` trait backed by three stores: structured state (SQLite), semantic search (FTS5/Qdrant), and a knowledge graph. [librefang-memory-wiki](librefang-memory-wiki.md) adds a durable Markdown knowledge vault with YAML frontmatter and Obsidian-compatible export.

### Capabilities

Four crates define what agents can do:

- [librefang-hands](librefang-hands.md) — registry and lifecycle management for *hands*, composable autonomous capability packages
- [librefang-skills](librefang-skills.md) — skill discovery, loading (TOML/YAML/JSON/ZIP), marketplace client, and OpenClaw compatibility
- [librefang-channels](librefang-channels.md) — bridge layer connecting the kernel to out-of-process messaging adapters (WhatsApp, Discord, etc.) via [librefang-subprocess](librefang-subprocess.md), the shared JSON-over-stdio transport
- [librefang-extensions](librefang-extensions.md) — credential vault, MCP server catalog, OAuth2 PKCE flows, plugin installer, and `.env` parsing

### Networking

[librefang-wire](librefang-wire.md) handles agent-to-agent communication over the LibreFang Protocol (OFP): cryptographic handshake, framed message transport, session management, and message authentication. [librefang-subprocess](librefang-subprocess.md) provides the persistent JSON-over-stdio transport used by sidecar bridges.

### API Surface

[librefang-api](librefang-api.md) is the primary HTTP/WebSocket interface between the kernel and external clients. It serves a React 19 SPA dashboard ([librefang-api-dashboard](librefang-api-dashboard.md)), static assets and i18n locale files ([librefang-api-static](librefang-api-static.md)), and a self-contained login page ([librefang-api-src](librefang-api-src.md)).

### Client Applications

Three client interfaces connect users to the system:

- [librefang-cli](librefang-cli.md) — the `librefang` binary; runs as a daemon or single-shot, with localization in [librefang-cli-locales](librefang-cli-locales.md) and project templates in [librefang-cli-templates](librefang-cli-templates.md)
- [librefang-desktop](librefang-desktop.md) — Tauri 2.0 native app (full runtime on desktop, thin remote client on mobile), with capability configs in [librefang-desktop-capabilities](librefang-desktop-capabilities.md) and generated build artifacts in [librefang-desktop-gen](librefang-desktop-gen.md)
- [librefang-acp](librefang-acp.md) — Agent Client Protocol adapter enabling embedding in editors (Zed, VS Code, JetBrains) via stdio JSON-RPC

### Import & Migration

[librefang-import](librefang-import.md) handles migrating configurations and agent definitions from external frameworks into LibreFang's native types.

## Test & Benchmark Suites

Each major crate has a dedicated integration test module that validates cross-cutting contracts against mock kernels and drivers — hermetic, no external services required:

| Test Module | Validates |
|---|---|
| [librefang-acp-tests](librefang-acp-tests.md) | ACP JSON-RPC wire behavior with `MockKernel` |
| [librefang-api-tests](librefang-api-tests.md) | Production router, auth, and middleware |
| [librefang-channels-tests](librefang-channels-tests.md) | `BridgeManager` dispatch pipeline end-to-end |
| [librefang-channels-benches](librefang-channels-benches.md) | Dispatch hot-path Criterion benchmarks |
| [librefang-cli-tests](librefang-cli-tests.md) | Build-script policy and vault key-rotation |
| [librefang-extensions-tests](librefang-extensions-tests.md) | CredentialVault encrypt/decrypt round-trips |
| [librefang-hands-tests](librefang-hands-tests.md) | `HandRegistry` lifecycle invariants |
| [librefang-import-tests](librefang-import-tests.md) | Migration idempotency and crash recovery |
| [librefang-kernel-handle-tests](librefang-kernel-handle-tests.md) | Trait default method contracts |
| [librefang-kernel-src](librefang-kernel-src.md) | Internal kernel integration tests |
| [librefang-kernel-tests](librefang-kernel-tests.md) | Public kernel API boundary |
| [librefang-llm-drivers-tests](librefang-llm-drivers-tests.md) | Provider wire contracts against mock servers |
| [librefang-memory-tests](librefang-memory-tests.md) | Chat-scoped privacy and session isolation |
| [librefang-memory-wiki-tests](librefang-memory-wiki-tests.md) | Vault acceptance and JSON shape stability |
| [librefang-runtime-tests](librefang-runtime-tests.md) | Agent loop, tool dispatch, MCP integration |
| [librefang-runtime-mcp-tests](librefang-runtime-mcp-tests.md) | `HttpCompat` MCP transport |
| [librefang-testing-src](librefang-testing-src.md) | Example tests demonstrating test infrastructure |
| [librefang-types-tests](librefang-types-tests.md) | TOML drift, `Default`/`serde` divergence, schemars |