# Other — librefang-kernel-router

# librefang-kernel-router

Hand and template routing engine for the LibreFang kernel.

## Purpose

This module is responsible for **routing** — resolving which **Hand** definition and which **template** apply to a given input. It acts as the decision layer between receiving raw input and dispatching it to the appropriate processing logic. Think of it as the kernel's traffic director: given a piece of content, it determines where that content should go.

## Architecture

The routing engine depends on several sibling crates, each serving a distinct role in the pipeline:

```
┌──────────────────────────┐
│   librefang-kernel-router│
│                          │
│  ┌────────────────────┐  │
│  │  Route Definitions │◄─────── toml (config files)
│  └───────┬────────────┘  │
│          │               │
│          ▼               │
│  ┌────────────────────┐  │
│  │  Pattern Matching  │◄─────── regex-lite
│  └───────┬────────────┘  │
│          │               │
│          ▼               │
│  ┌────────────────────┐  │
│  │    Route Result    │──────► librefang-hands
│  └────────────────────┘  │     librefang-types
└──────────────────────────┘
```

### Key dependencies and what they imply

| Dependency | Role in this module |
|---|---|
| `librefang-types` | Shared type definitions used in route matching results — likely route IDs, match metadata, or resolved targets |
| `librefang-hands` | The target of routing decisions. Routes resolve to a specific Hand that will handle the input |
| `regex-lite` | Lightweight regex engine used for pattern-based route matching. Route rules likely use regex patterns to test against incoming content |
| `toml` | Route definitions are loaded from TOML configuration files, allowing declarative route setup |
| `dirs` | Standard platform directories — used to locate configuration files (e.g., user-level or system-level route definitions) |
| `serde` / `serde_json` | Deserialization of route configuration and serialization of match results |
| `tracing` | Structured logging of routing decisions for debugging and observability |

## How routing works

Based on the dependency profile, the routing flow follows this pattern:

1. **Load route definitions** — TOML configuration files are discovered (via `dirs`) and deserialized (via `toml` + `serde`) into internal route rule structures.

2. **Evaluate patterns** — When input arrives, each route's pattern (expressed as a regex via `regex-lite`) is tested against the input. Routes are evaluated in a defined order, likely respecting priority or specificity.

3. **Resolve to a Hand** — A successful match produces a result referencing a Hand from `librefang-hands`, along with any captured groups or metadata typed via `librefang-types`.

4. **Emit tracing events** — Routing decisions are instrumented with `tracing` spans, making it possible to trace why a particular route was selected or why no route matched.

## Configuration

Route definitions are expected to live in TOML files. The use of `dirs` suggests the router looks in standard configuration locations:

- A user-level config directory (e.g., `~/.config/librefang/`)
- Potentially a system-level fallback

Route entries likely include:

- A **pattern** field containing a regex expression
- A **target** field referencing a Hand identifier
- Optional **priority** or **order** for disambiguation when multiple routes match

## Testing

The dev-dependency on `tempfile` and `librefang-runtime` indicates that tests:

- Create temporary directories and TOML files to test configuration loading in isolation
- May use `librefang-runtime` to exercise the router within a realistic execution context, verifying end-to-end routing behavior

## Relationship to the kernel

This module is one component of the LibreFang kernel. It sits between input ingestion and hand execution:

```
Input → [router resolves Hand] → [Hand processes input] → Output
```

The router itself is stateless with respect to input — it makes a routing decision per request based on the current configuration. Configuration changes (loaded from TOML) would affect subsequent routing decisions.

## Integration points

When consuming this module from elsewhere in the kernel:

- Initialize the router by providing a configuration path or relying on default discovery via `dirs`
- Call into the router with the relevant input or content to match against
- Receive a resolved reference to a Hand (from `librefang-hands`) to dispatch to
- Handle the case where no route matches, which should be logged via `tracing`