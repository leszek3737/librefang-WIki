# Other — librefang-kernel-router

# librefang-kernel-router

Hand and template routing engine for the LibreFang kernel. This crate is responsible for resolving which hand definition or template should handle a given request, loading route configurations from disk, and performing pattern-based matching.

## Purpose

LibreFang's kernel delegates request handling to modular components called **hands**. The router sits between incoming requests and those hands, determining the correct dispatch target based on configurable routing rules. It also supports **template routing**, allowing static or dynamically rendered content to be served from named templates matched against request paths.

## Role in the Kernel

```
┌─────────────┐     ┌───────────────────┐     ┌─────────────────┐
│  Incoming    │────▶│  kernel-router    │────▶│  Target Hand /  │
│  Request     │     │  (this crate)     │     │  Template       │
└─────────────┘     └───────────────────┘     └─────────────────┘
```

The router is a pure decision-making layer. It does not execute hands or render templates itself — it resolves a match and returns the result to the caller (typically the kernel's request loop).

## Key Dependencies

| Dependency | Role in this crate |
|---|---|
| `librefang-types` | Shared type definitions (request representations, route entries, match results) |
| `librefang-hands` | Hand registry and metadata used during resolution |
| `serde_json` | Deserializing route configuration files in JSON format |
| `toml` | Deserializing route configuration files in TOML format |
| `regex-lite` | Pattern matching for path-based route rules |
| `dirs` | Locating platform-appropriate configuration directories |
| `tracing` | Structured logging of route resolution decisions |

## Configuration

Route definitions are loaded from configuration files on disk. The router searches standard configuration directories (resolved via the `dirs` crate) and supports both TOML and JSON formats. This allows route tables to be managed externally without recompiling the kernel.

## Pattern Matching

Routes may specify literal paths or regex patterns (using `regex-lite`). When a request arrives, the router evaluates registered patterns in priority order and returns the first matching hand or template. The use of `regex-lite` keeps the dependency footprint small while supporting the subset of regex syntax needed for path matching.

## Testing

The dev-dependency on `librefang-runtime` and `tempfile` indicates that tests spin up temporary directory structures with mock configuration files, then verify that the router correctly resolves routes against those configurations. When writing tests for new route rules, create a temporary config directory using `tempfile`, write route definitions into it, and assert the router's resolution output.