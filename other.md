# Other

# Other Module

Supporting infrastructure, integrations, and tooling for the LibreFang Agent OS. This module group encompasses everything that isn't a core domain crate — from the HTTP API and dashboard UI to the CLI, desktop app, LLM drivers, messaging channels, and test infrastructure.

## Architecture

```mermaid
graph TD
    subgraph Clients
        CLI[librefang-cli]
        Desktop[librefang-desktop]
        Dashboard[librefang-api-dashboard]
        External[External Clients]
    end

    subgraph API Layer
        API[librefang-api]
        OpenAI[OpenAI Compat Layer]
        Wire[librefang-wire]
    end

    subgraph Kernel
        Kernel[librefang-kernel]
        Router[kernel-router]
        Metering[kernel-metering]
        Handle[kernel-handle]
    end

    subgraph Runtime
        Runtime[librefang-runtime]
        WASM[runtime-wasm]
        MCP[runtime-mcp]
        OAuth[runtime-oauth]
    end

    subgraph Providers
        LLMDriver[librefang-llm-driver]
        LLMDrivers[librefang-llm-drivers]
    end

    subgraph Data & Capabilities
        Memory[librefang-memory]
        Skills[librefang-skills]
        Hands[librefang-hands]
        Channels[librefang-channels]
        Extensions[librefang-extensions]
    end

    subgraph Foundation
        Types[librefang-types]
        HTTP[librefang-http]
        Telemetry[librefang-telemetry]
    end

    CLI --> API
    Desktop --> API
    Dashboard --> API
    External --> OpenAI
    External --> API

    API --> Kernel
    API --> Runtime
    API --> Channels
    OpenAI --> API

    Kernel --> Router
    Kernel --> Metering
    Kernel --> Handle
    Kernel --> Runtime
    Kernel --> Memory
    Kernel --> Skills
    Kernel --> Hands

    Runtime --> WASM
    Runtime --> MCP
    Runtime --> OAuth
    Runtime --> LLMDriver
    Runtime --> Memory

    LLMDriver --> LLMDrivers
    LLMDrivers --> HTTP

    Extensions --> HTTP
    Wire --> Types
    Channels --> Kernel

    Types --> Foundation
    HTTP --> Foundation
    Telemetry --> Foundation
```

## Sub-module Groups

### Core Types

[librefang-types](librefang-types.md) is the foundational crate every other module depends on. It defines shared domain types, error types, configuration models, and trait interfaces. Supporting it are [librefang-types-src](librefang-types-src.md) (model catalog schemas), [librefang-types-locales](librefang-types-locales.md) (localized API error messages in Fluent format), and [librefang-types-tests](librefang-types-tests.md) (contract tests ensuring the dashboard's TOML serializer and kernel's deserializer stay in sync).

### Kernel & Orchestration

[librefang-kernel](librefang-kernel.md) is the central orchestrator that wires together routing, metering, memory, skills, hands, extensions, and LLM drivers. It delegates to:

- [librefang-kernel-router](librefang-kernel-router.md) — matches incoming messages to registered hands via template patterns
- [librefang-kernel-metering](librefang-kernel-metering.md) — tracks resource consumption and enforces quotas
- [librefang-kernel-handle](librefang-kernel-handle.md) — trait-based abstraction for in-process kernel communication

Testing is covered by [librefang-kernel-src](librefang-kernel-src.md) (unit tests) and [librefang-kernel-tests](librefang-kernel-tests.md) (integration tests including WASM execution and workflow pipelines).

### Agent Runtime

[librefang-runtime](librefang-runtime.md) orchestrates the full agent lifecycle — LLM interaction, tool invocation, memory management, and sandboxing. It pulls in specialized subsystems:

- [librefang-runtime-wasm](librefang-runtime-wasm.md) — wasmtime-based sandbox for executing untrusted skills
- [librefang-runtime-mcp](librefang-runtime-mcp.md) — Model Context Protocol client for dynamic tool discovery
- [librefang-runtime-oauth](librefang-runtime-oauth.md) — OAuth 2.0 PKCE flows for ChatGPT and GitHub Copilot authentication
- [librefang-runtime-tests](librefang-runtime-tests.md) — MCP OAuth integration tests

### LLM Integration

[librefang-llm-driver](librefang-llm-driver.md) defines the trait that all LLM backends implement. [librefang-llm-drivers](librefang-llm-drivers.md) provides concrete implementations for Anthropic, OpenAI, Google Gemini, and others, using the shared HTTP client from [librefang-http](librefang-http.md).

### API & Dashboard

[librefang-api](librefang-api.md) exposes the primary HTTP/WebSocket interface and embeds the dashboard UI. It integrates with nearly every other subsystem. The frontend is a React SPA ([librefang-api-dashboard](librefang-api-dashboard.md)) built with TanStack Router and Query. [librefang-api-src](librefang-api-src.md) adds an OpenAI-compatible API surface (`/v1/chat/completions`, `/v1/models`) and a zero-dependency login page. [librefang-api-static](librefang-api-static.md) provides i18n locale files (English, Japanese), and [librefang-api-tests](librefang-api-tests.md) contains integration and load tests.

### Messaging Channels

[librefang-channels](librefang-channels.md) provides pluggable messaging integrations for 43+ platforms (Telegram, Discord, Slack, etc.), each behind a Cargo feature flag. Performance-critical paths are benchmarked in [librefang-channels-benches](librefang-channels-benches.md), and the full dispatch pipeline is tested in [librefang-channels-tests](librefang-channels-tests.md).

### CLI & Desktop

[librefang-cli](librefang-cli.md) produces the `librefang` binary — the primary terminal entry point with an interactive TUI and shell completions. It uses [librefang-cli-locales](librefang-cli-locales.md) for i18n (English, Simplified Chinese) and [librefang-cli-templates](librefang-cli-templates.md) for `init` project scaffolding.

[librefang-desktop](librefang-desktop.md) packages the runtime as a native desktop app via Tauri 2.0, with system tray integration and auto-updates. Security is managed through [librefang-desktop-capabilities](librefang-desktop-capabilities.md) and auto-generated [librefang-desktop-gen](librefang-desktop-gen.md) artifacts.

### Capabilities & Data

- [librefang-hands](librefang-hands.md) — curated capability packages assignable to agents
- [librefang-skills](librefang-skills.md) — skill registry, filesystem loader, marketplace client, and OpenClaw compatibility
- [librefang-memory](librefang-memory.md) — persistence layer for conversation history and agent state, tested via [librefang-memory-tests](librefang-memory-tests.md)
- [librefang-extensions](librefang-extensions.md) — MCP server bootstrap, AES-256-GCM credential vault, and OAuth2 PKCE flow, with a shared HTTP client in [librefang-extensions-src](librefang-extensions-src.md)

### Cross-cutting Concerns

- [librefang-wire](librefang-wire.md) — agent-to-agent networking with HMAC-SHA256 authentication and JSON framing
- [librefang-telemetry](librefang-telemetry.md) — OpenTelemetry and Prometheus metrics instrumentation
- [librefang-migrate](librefang-migrate.md) — imports configurations from other agent frameworks (JSON, YAML, TOML, JSON5)
- [librefang-testing](librefang-testing.md) — shared mock kernel, mock LLM driver, and route-level test utilities used across integration tests

## Key Cross-Module Workflows

**Dashboard → API → Kernel → Runtime → LLM:** A user interacts with the React dashboard ([librefang-api-dashboard](librefang-api-dashboard.md)), which calls the API via TanStack Query mutations. The API server ([librefang-api](librefang-api.md)) routes requests through the kernel ([librefang-kernel](librefang-kernel.md)), which dispatches to the runtime ([librefang-runtime](librefang-runtime.md)). The runtime invokes an LLM via a concrete driver from [librefang-llm-drivers](librefang-llm-drivers.md), using the trait from [librefang-llm-driver](librefang-llm-driver.md).

**Channel Message → Agent Response:** An inbound message from Telegram or Discord arrives through [librefang-channels](librefang-channels.md). The `BridgeManager` dispatches it to the kernel, which uses [librefang-kernel-router](librefang-kernel-router.md) to match it to a hand ([librefang-hands](librefang-hands.md)). The runtime executes the agent loop, invoking skills ([librefang-skills](librefang-skills.md)) or WASM modules ([librefang-runtime-wasm](librefang-runtime-wasm.md)) as needed, and the response flows back through the channel adapter.

**OpenAI-Compatible Access:** External tools hit the `/v1/chat/completions` endpoint in [librefang-api-src](librefang-api-src.md), which translates the OpenAI protocol into a LibreFang agent interaction — enabling any OpenAI client library to drive an agent session.