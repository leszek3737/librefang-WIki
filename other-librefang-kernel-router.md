# Other — librefang-kernel-router

# librefang-kernel-router

Hand and template routing engine for the LibreFang kernel. This module is responsible for resolving user input events to the appropriate hand definitions and template layouts, acting as the central dispatch layer between raw input and the rendering pipeline.

## Role in the Architecture

`librefang-kernel-router` sits between the input layer and the layout engine. When a key event or gesture is detected, the router determines which hand configuration and which template should handle it. This decouples input detection from the actual finger positioning and glyph rendering logic.

```
┌─────────────┐     ┌──────────────────────┐     ┌──────────────┐
│ Input Layer │────▶│ librefang-kernel-    │────▶│ Layout/      │
│             │     │ router               │     │ Rendering    │
└─────────────┘     └──────────────────────┘     └──────────────┘
                           │
                    ┌──────┴──────┐
                    ▼             ▼
             librefang-     Config /
             hands          Template
                            Files
```

## Dependencies and What They Enable

| Dependency | Purpose in This Module |
|---|---|
| `librefang-types` | Shared type definitions—key codes, hand identifiers, routing result types |
| `librefang-hands` | Hand definitions and finger mapping data used during routing |
| `serde` / `serde_json` | Deserialization of route configurations and template files |
| `toml` | Parsing of user-facing TOML configuration files for custom routes |
| `regex-lite` | Pattern matching against input identifiers or template selectors |
| `dirs` | Resolving platform-specific config directories for user route overrides |
| `tracing` | Instrumentation of routing decisions for debugging and diagnostics |

## Key Concepts

### Hands

A "hand" represents a physical hand layout—a mapping of fingers to positional slots. The router queries `librefang-hands` to look up hand definitions when resolving a route. Hands are typically identified by a string or enum variant (e.g., left hand vs. right hand, or a named custom layout).

### Templates

Templates define the structural arrangement of glyphs or actions within a given context. The router matches an input event to a template, which then dictates how downstream modules render or interpret the input.

### Routing

Routing is the process of accepting an input event (key press, gesture, or mode change) and producing a resolved target: which hand to use and which template to apply. Routes can be built-in defaults or user-defined overrides loaded from configuration.

## Configuration Loading

The module uses `dirs` to locate platform-appropriate configuration directories and `toml` to parse route definitions. This allows users to define custom routing rules without modifying compiled code. A typical config lookup sequence:

1. Check the user's local config directory (via `dirs::config_dir()`).
2. Look for a LibreFang-specific subdirectory.
3. Parse any `.toml` route override files found.
4. Fall back to compiled-in default routes if no overrides exist.

## Testing

The dev-dependency on `librefang-runtime` indicates that integration tests spin up a minimal runtime to exercise routing end-to-end, while `tempfile` is used to create isolated config directories for deterministic test runs.

## Integration Points

Other kernel modules call into the router when they need to resolve an input context. The router itself is primarily a lookup and dispatch engine—it does not own state beyond its loaded configuration. It delegates to:

- **`librefang-hands`** for hand structure queries.
- **`librefang-types`** for shared data structures passed back to callers.

Consumers of this module should expect a stateless or lazily-initialized service: load the configuration once, then query routes repeatedly with no side effects.