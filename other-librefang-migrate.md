# Other — librefang-migrate

# librefang-migrate

Migration engine for importing configurations and data from other agent frameworks into LibreFang.

## Overview

`librefang-migrate` provides tooling to convert projects, agent definitions, and configurations authored in other agent frameworks so they can be used within the LibreFang ecosystem. It handles format detection, parsing, schema translation, and output of LibreFang-native types.

## Supported Input Formats

The crate pulls in parsers for four configuration formats, allowing it to ingest definitions from a wide range of existing tools:

| Format | Dependency | Common Use |
|---|---|---|
| JSON | `serde_json` | Most agent frameworks |
| YAML | `serde_yaml` | Common in declarative agent configs |
| JSON5 | `json5` | Human-friendly JSON with comments |
| TOML | `toml` | Rust-ecosystem configurations |

## Key Dependencies and Their Roles

- **`librefang-types`** — Target types. Migrated data is ultimately expressed using the shared type definitions from this crate.
- **`walkdir`** — Recursive directory traversal for discovering configuration files in a source project.
- **`uuid`** / **`chrono`** — Generating new identifiers and timestamps during migration, ensuring imported entities receive fresh, non-colliding metadata.
- **`thiserror`** — Typed error definitions for migration failures (parse errors, schema mismatches, I/O errors).
- **`tracing`** — Structured logging of migration progress and warnings.
- **`dirs`** — Resolving standard system directories when locating default input or output paths.

## Architecture

```mermaid
flowchart LR
    A[Source Project Files] --> B[Directory Walker]
    B --> C[Format Detection]
    C --> D[Parser]
    D --> E[Schema Translation]
    E --> F[librefang-types Output]
```

The migration pipeline discovers files via `walkdir`, detects the appropriate parser based on file extension or content, deserializes into intermediate representations, translates those into `librefang-types`, and emits the result.

## Error Handling

All migration errors are defined using `thiserror` derive macros. Expect error variants covering:

- File I/O failures during directory traversal
- Parse failures across any of the four supported formats
- Schema translation failures when source structures don't map cleanly to LibreFang types

Callers should handle these via `Result<T, MigrationError>` (or the crate's concrete error type).

## Testing

The `tempfile` dev-dependency indicates that tests create temporary directories to exercise the directory-walking and file-discovery logic in isolation. When contributing new importers, use `tempfile` to scaffold a fake source project structure rather than relying on fixture files in the repository.

## Integration with LibreFang

This crate is a utility tool. Other LibreFang crates do not depend on it at runtime. Instead, it is invoked as a build-time or one-time CLI step to bootstrap a LibreFang project from an existing agent codebase. The only coupling is through `librefang-types`, which defines the output contract.