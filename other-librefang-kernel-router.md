# Other — librefang-kernel-router

# librefang-kernel-router

Hand/Template routing engine for the LibreFang kernel.

## Purpose

This crate is responsible for **routing** — resolving which hand definition and which template should handle a given request or input event. It acts as the dispatch layer that sits between raw input and the kernel's hand execution system, mapping symbolic names or patterns to concrete hand/template configurations.

## Role in the Architecture

```
┌──────────────┐    ┌─────────────────────┐    ┌────────────────┐
│  Input/Event  │───▶│  kernel-router      │───▶│  librefang-    │
│  Source       │    │  (route resolution) │    │  hands         │
└──────────────┘    └─────────────────────┘    └────────────────┘
                            │
                            ▼
                     ┌────────────────┐
                     │  librefang-    │
                     │  types         │
                     └────────────────┘
```

The router consumes type definitions from `librefang-types` and produces routing decisions that are consumed by `librefang-hands`. It is the glue that determines *which* hand runs, *where* its template lives, and *how* parameters bind.

## Key Dependencies

| Dependency | Role in this crate |
|---|---|
| `librefang-types` | Shared type definitions — route descriptors, hand identifiers, template metadata |
| `librefang-hands` | Hand definitions and execution targets that routes resolve to |
| `regex-lite` | Pattern matching for route rules — supports wildcard/glob-style matching of hand names or template paths |
| `serde_json` | Deserialization of route configuration files (JSON format) |
| `toml` | Deserialization of route configuration files (TOML format) |
| `dirs` | Resolving platform-specific config directories to locate route definition files |
| `tracing` | Structured logging of route resolution, cache hits/misses, and fallback behavior |

## Configuration Loading

The router supports loading routing tables from both **TOML** and **JSON** configuration files. It uses the `dirs` crate to resolve platform-standard configuration directories (e.g., `~/.config/librefang/` on Linux), allowing users to define custom hand-to-template mappings without modifying compiled code.

Typical config lookup order:

1. Path specified by environment variable (if applicable)
2. User config directory (`dirs::config_dir()` / `librefang/`)
3. Bundled/fallback defaults

## Route Resolution

A route maps a **hand identifier** (and optionally a template name or pattern) to a concrete execution target. The `regex-lite` dependency indicates that route rules support pattern-based matching rather than only exact string equality — allowing a single rule to match multiple hands or template variants.

Resolution follows a priority model:

1. **Exact matches** are checked first.
2. **Pattern matches** (regex) are evaluated in definition order.
3. **Fallback/default** route applies if nothing else matches.

All resolution steps are instrumented with `tracing` spans, making it straightforward to debug why a particular hand was (or wasn't) routed.

## Testing

The `librefang-runtime` dev-dependency indicates that integration tests exercise the router within a full or partial runtime environment, verifying that routes resolve correctly end-to-end rather than in isolation. The `tempfile` dev-dependency suggests tests that create temporary configuration files to validate loading and parsing behavior without touching the user's real config.

## Relationship to Other Crates

- **Consumers** call into this crate to resolve which hand/template pair to activate.
- **`librefang-hands`** provides the target definitions; the router does not execute hands itself — it only selects them.
- **`librefang-types`** supplies the shared vocabulary (structs, enums) that both the router and its consumers agree on.
- **`librefang-runtime`** orchestrates the router alongside other kernel subsystems during actual execution.