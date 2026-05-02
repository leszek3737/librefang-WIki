# Other — librefang-migrate

# librefang-migrate

Migration engine for importing configurations and data from other agent frameworks into LibreFang.

## Purpose

`librefang-migrate` provides tooling to bring existing agent framework installations—their configurations, agent definitions, conversation histories, and related artifacts—into the LibreFang ecosystem. It handles format detection, parsing, transformation, and output of LibreFang-compatible structures.

## Design

The crate is structured around a multi-format parsing pipeline. Its dependency footprint reveals its approach:

| Dependency | Role |
|---|---|
| `serde`, `serde_json`, `serde_yaml`, `json5`, `toml` | Deserialize source configurations in their native formats |
| `walkdir` | Recursively discover files in a source framework's directory tree |
| `librefang-types` | Target types for the converted output |
| `thiserror` | Typed error handling for migration failures |
| `tracing` | Structured logging of migration progress and warnings |
| `chrono` | Timestamp normalization during import |
| `dirs` | Locate standard platform directories (e.g., auto-discovering existing framework data) |

### Migration Pipeline

```mermaid
flowchart LR
    A[Source Directory] --> B[Discovery walkdir]
    B --> C[Format Detection]
    C --> D[Parse serde / json5 / toml / yaml]
    D --> E[Transform to librefang-types]
    E --> F[Write Output]
```

The process follows four stages:

1. **Discovery** — Walk a directory tree using `walkdir` to locate candidate files for migration.
2. **Detection** — Identify the source framework and file format (JSON, YAML, TOML, JSON5) based on file extensions and content structure.
3. **Parsing & Transformation** — Deserialize the source data and map it onto `librefang-types` structures.
4. **Output** — Emit the converted data in LibreFang's native representation.

## Relationship to Other Crates

This crate sits at the edge of the workspace. It depends on `librefang-types` for its output structures but is not depended upon by any other workspace crate. It is intended to be used as a standalone binary or invoked as a one-time import step, not as a runtime dependency of the core agent system.

## Error Handling

Errors are defined using `thiserror` and cover common migration failure modes: unsupported source formats, malformed configuration files, missing required fields during transformation, and filesystem I/O errors during discovery or output.

## Testing

The `tempfile` dev-dependency is used to construct ephemeral directory trees in tests, simulating source framework layouts without touching the real filesystem.