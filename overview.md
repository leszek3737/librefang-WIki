# librefang — Wiki

# LibreFang

**Libre Agent Operating System — Free as in Freedom**

LibreFang is an open-source Agent OS built entirely in Rust. It provides a complete platform for creating, running, and managing autonomous AI agents — from interactive chatbots to background workers that operate on schedules and triggers. The workspace contains 14 crates, passes 2,100+ tests, and ships zero clippy warnings.

New developers should start here. This page explains what the system does, how it's structured, and where to go next.

---

## What LibreFang Does

At its core, LibreFang boots a **kernel** that manages the full lifecycle of AI agents. Each agent is configured with a model provider, system prompt, memory scope, tool access, and resource limits. The kernel coordinates agent execution, enforces security policies, handles inter-agent messaging, and exposes everything through a REST/WebSocket API.

Users interact with agents through several surfaces:

- The **CLI** for terminal-based management and single-shot commands
- The **Dashboard**, a React SPA served by the API server
- The **Desktop Application**, a Tauri 2.0 native client
- **Channel adapters** that bridge 40+ external messaging platforms (Telegram, Discord, Slack, WhatsApp, and more)

Agents can be extended with **Skills** (pluggable tool bundles) and connected to external services through the **Extensions System** (one-click MCP server integration with OAuth support). For longer-running autonomous tasks, pre-built **Agent Hands** provide domain-complete background configurations.

---

## Architecture

```mermaid
graph TD
    CLI["CLI"]
    DASH["Dashboard"]
    DESK["Desktop App"]
    API["API Server"]
    KERNEL["Kernel Core"]
    RUNTIME["Runtime Engine"]
    MEMORY["Memory Management"]
    CHANNELS["Channels"]
    SKILLS["Skills System"]
    TYPES["Type Definitions"]

    CLI --> API
    DASH --> API
    DESK --> API
    API --> KERNEL
    API --> RUNTIME
    KERNEL --> RUNTIME
    RUNTIME --> MEMORY
    RUNTIME --> SKILLS
    KERNEL --> CHANNELS
    CHANNELS --> API

    style TYPES fill:#e8e8e8,stroke:#999
    linkStyle 10 stroke:#999,stroke-dasharray:5 5
```

The diagram above shows the primary request flow. Every user-facing surface communicates through the [API Server](librefang-api.md), which delegates to the [Kernel Core](librefang-kernel.md) for agent lifecycle management and the [Runtime Engine](librefang-runtime.md) for execution. The runtime pulls context from [Memory Management](librefang-memory.md) and invokes tools through the [Skills System](librefang-skills.md). [Channels](librefang-channels.md) route external platform messages into the kernel and deliver responses back.

[Type Definitions](librefang-types.md) underpin the entire workspace — every crate depends on it for shared data structures, and it contains no business logic.

---

## Key End-to-End Flows

### Spawning and Talking to an Agent

1. A user runs `librefang agent spawn examples/custom-agent/agent.toml` via the [CLI](librefang-cli.md), or sends a POST through the [API Server](librefang-api.md).
2. The [Kernel Core](librefang-kernel.md) parses the agent config, registers it, and makes it available for sessions.
3. When a chat message arrives (from the Dashboard, CLI, or a [Channel](librefang-channels.md) adapter), the [Runtime Engine](librefang-runtime.md) enters the agent loop: it recalls relevant memories, assembles the prompt, calls the LLM driver, and processes the response.
4. If the LLM requests a tool call, the runtime dispatches it through the [Skills System](librefang-skills.md) or the [Extensions System](librefang-extensions.md) for MCP tools, then feeds the result back into the loop.
5. The final response streams back through the API to whichever surface originated the request.

### Installing an MCP Extension

1. The user runs `librefang integration install <name>` or uses the [Extensions System](librefang-extensions.md) API endpoint.
2. The extension installer discovers the integration, resolves credentials (including OAuth flows via `librefang-runtime-oauth`), generates the MCP server config, and registers it with the kernel.
3. On the next agent turn, the [Runtime Engine](librefang-runtime.md) includes the MCP server's tools in the LLM's tool list, and the agent can use them.

### Receiving a Message from an External Platform

1. A platform adapter in [Channels](librefang-channels.md) (e.g., Telegram, Discord) receives an incoming message and converts it into a unified `ChannelMessage`.
2. The message is routed to the appropriate agent through the [Kernel Core](librefang-kernel.md).
3. The [Runtime Engine](librefang-runtime.md) executes the agent loop and produces a response.
4. The response is delivered back through the same channel adapter, handling platform-specific formatting and message length limits.

---

## The Crate Map

The workspace is organized into focused crates with clear responsibilities:

| Layer | Crates | Purpose |
|-------|--------|---------|
| **Core** | `librefang-kernel`, `librefang-runtime`, `librefang-types` | Kernel lifecycle, agent execution loop, shared types |
| **Memory** | `librefang-memory` | Structured, semantic (vector), and knowledge graph storage |
| **LLM** | `librefang-llm-driver`, `librefang-llm-drivers` | Driver abstraction and provider implementations |
| **Interface** | `librefang-api`, `librefang-http`, `librefang-wire` | REST/WebSocket API, HTTP utilities, inter-kernel wire protocol |
| **Sandboxing** | `librefang-runtime-wasm` | WASM sandbox for untrusted skill/plugin code |
| **Protocols** | `librefang-runtime-mcp`, `librefang-runtime-oauth` | MCP client and OAuth credential flows |
| **Introspection** | `librefang-kernel-handle`, `librefang-kernel-ro` | Kernel handle abstractions for read/write and read-only access |

---

## Getting Started

### Prerequisites

- **Rust** 1.80+ (install via [rustup](https://rustup.rs))
- A supported LLM provider API key (OpenAI, Anthropic, Groq, etc.)

### Build

```bash
git clone https://github.com/librefang/librefang.git
cd librefang
cargo build --workspace
```

### Configure

On first run, initialize the config file with an interactive wizard:

```bash
librefang init
```

This creates `~/.librefang/config.toml` and walks you through provider selection and API key setup. The full configuration schema is documented in the [Configuration](configuration.md) page.

### Run

Start the daemon:

```bash
librefang start
```

This boots the kernel, starts the API server (default `http://localhost:8080`), and opens the Dashboard. From there you can spawn agents, install skills, and connect channels.

For single-shot usage without a daemon, most CLI commands boot an in-process kernel, execute, and exit.

### Test

```bash
cargo test --workspace
```

The [Testing Utilities](librefang-testing.md) crate provides `MockKernelBuilder` for writing integration tests against API routes without starting a full daemon or hitting real LLM providers.

---

## Where to Go Next

- **[Kernel Core](librefang-kernel.md)** — understand agent lifecycle, the registry, event bus, and security policies
- **[Runtime Engine](librefang-runtime.md)** — the agent loop, LLM driver abstraction, and tool execution pipeline
- **[Memory Management](librefang-memory.md)** — structured state, semantic search, and knowledge graph storage
- **[API Server](librefang-api.md)** — HTTP/WebSocket endpoints, middleware, and the Dashboard SPA
- **[Skills System](librefang-skills.md)** — how to build and register pluggable agent capabilities
- **[Extensions System](librefang-extensions.md)** — MCP integration lifecycle and OAuth flows
- **[Channels](librefang-channels.md)** — the `ChannelAdapter` trait and platform bridge architecture
- **[Examples](examples.md)** — ready-to-use templates for custom agents, skills, and channel adapters
- **[CLI](librefang-cli.md)** — daemon management, argument parsing, and TUI screens