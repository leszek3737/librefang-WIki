# Other — librefang-import

# librefang-import

Import engine for migrating agent configurations from other frameworks into LibreFang.

## Purpose

This module provides the tooling needed to locate, parse, and convert configuration files authored for other agent frameworks into LibreFang-native types. It handles format detection, directory traversal, and data transformation so that operators can point the import engine at an existing deployment and produce a working LibreFang configuration without manual translation.

## Architecture

```mermaid
flowchart LR
    A[Source Config Files] --> B[Directory Walker]
    B --> C[Format Detection]
    C --> D[Parser: JSON / YAML / JSON5 / TOML]
    D --> E[Intermediate Representation]
    E --> F[librefang-types]
```

The import pipeline follows a straightforward flow: discover files on disk, detect their format, parse into a generic intermediate representation, and then convert into the strongly-typed structures defined in `librefang-types`.

## Supported Input Formats

The engine can ingest configuration files in four formats, chosen based on file extension and content:

| Format | Crate | Typical Extension |
|--------|-------|-------------------|
| JSON | `serde_json` | `.json` |
| YAML | `serde_yaml` | `.yaml`, `.yml` |
| JSON5 | `json5` | `.json5` |
| TOML | `toml` | `.toml` |

All four are deserialized through `serde`, so any type implementing `Deserialize` can be populated from any supported format.

## Key Dependencies

| Dependency | Role |
|------------|------|
| `librefang-types` | Shared type definitions that the importer produces as output |
| `walkdir` | Recursive directory traversal to discover configuration files in nested layouts |
| `dirs` | Resolves standard platform directories (home, config, data) when searching for default agent installation paths |
| `chrono` | Timestamp parsing and conversion for log files, cron schedules, and time-based config values |
| `thiserror` | Derives structured error types for parse failures, missing files, and conversion errors |
| `tracing` | Structured logging throughout the import pipeline for diagnostics and progress reporting |

## Error Handling

All fallible operations return `Result<T, E>` where `E` is derived via `thiserror`. Import errors generally fall into these categories:

- **IO errors** — files or directories not found, permission denied
- **Parse errors** — malformed JSON/YAML/TOML, unexpected structure
- **Conversion errors** — valid syntax but semantically incompatible with LibreFang types (e.g., unrecognized enum variant, out-of-range value)

Errors are annotated with file paths and line/context information where possible, and propagated with `tracing` spans so operators can identify exactly which source file caused a failure.

## Testing

The dev-dependency on `tempfile` indicates that tests create isolated temporary directory trees mimicking foreign agent installations, then assert that the import engine discovers and converts them correctly. This avoids depending on fixture files checked into the repository and keeps tests self-contained.

## Relationship to Other Modules

This module depends only on `librefang-types` within the workspace. It has no incoming calls from other modules, meaning it is typically invoked as a standalone utility or CLI subcommand rather than being called during normal LibreFang runtime. The output of an import run is a set of `librefang-types` structures that can be serialized to disk and consumed by the rest of the system.