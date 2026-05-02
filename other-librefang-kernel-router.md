# Other — librefang-kernel-router

# librefang-kernel-router

Hand and template routing engine for the LibreFang kernel. This module is responsible for resolving incoming requests to the correct hand implementation based on configurable routing rules and template matching.

## Overview

LibreFang uses a **hand** abstraction — discrete processing units that handle specific categories of input. The router sits at the front of the kernel's request pipeline, examining incoming data and determining which hand (or chain of hands) should process it.

Routing decisions are driven by **templates**: pattern-based rules that match against request attributes. Templates can use literal matching, regex patterns, or structured criteria to classify input.

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Incoming Request                    │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│            Router                                │
│  ┌─────────────┐  ┌──────────────────────────┐  │
│  │ Template    │─▶│ Hand Resolution           │  │
│  │ Matching    │  │ (via librefang-hands)     │  │
│  └─────────────┘  └──────────────────────────┘  │
│  ┌─────────────────────────────────────────────┐│
│  │ Route Configuration (TOML)                  ││
│  └─────────────────────────────────────────────┘│
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────┐
│          Target Hand Execution                   │
└─────────────────────────────────────────────────┘
```

## Dependencies

| Crate | Purpose |
|-------|---------|
| `librefang-types` | Shared type definitions for requests, routes, and routing metadata |
| `librefang-hands` | Hand registry and hand trait implementations |
| `serde_json` | Deserialization of route definitions stored in JSON format |
| `regex-lite` | Pattern matching within template rules |
| `toml` | Parsing route configuration from TOML files |
| `dirs` | Resolving platform-specific configuration directories |
| `tracing` | Structured logging of routing decisions and diagnostics |

## Configuration

Route definitions are loaded from TOML configuration files. The router uses the `dirs` crate to locate platform-appropriate configuration paths (e.g., `~/.config/librefang/` on Linux).

Configuration defines the mapping between templates and hands. Each route entry specifies:

- A **template pattern** to match against incoming data
- The **hand identifier** to dispatch to upon match
- Optional **priority** or **ordering** hints for ambiguous matches

## Routing Process

1. **Load configuration** — Parse TOML route definitions at startup.
2. **Receive request** — Accept an incoming request from the kernel pipeline.
3. **Template evaluation** — Match the request against loaded templates using literal and regex-based comparison (`regex-lite`).
4. **Hand resolution** — Use `librefang-hands` to look up the hand implementation associated with the matched template.
5. **Dispatch** — Forward the request to the resolved hand for execution.

All routing decisions are instrumented with `tracing` spans, enabling runtime diagnostics of which templates matched and why.

## Integration with LibreFang

This module is a kernel-level component. It consumes types from `librefang-types` and delegates actual request processing to hands provided by `librefang-hands`. The runtime (`librefang-runtime`) is used only in dev-dependencies for integration tests that verify end-to-end routing behavior.

## Testing

Tests use `tempfile` to create isolated configuration files, ensuring route loading and matching logic is verified without mutating system config paths.