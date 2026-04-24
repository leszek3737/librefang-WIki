# Other — librefang-migrate

# librefang-migrate

Migration engine for importing agent configurations and data from other agent frameworks into LibreFang format.

## Overview

`librefang-migrate` provides tooling to convert projects, configurations, and agent definitions authored in other frameworks into LibreFang-compatible structures. It handles format detection, parsing, schema mapping, and output generation.

## Purpose

Adopting LibreFang from an existing agent framework requires translating configurations, prompts, tool definitions, and potentially conversation history. This module encapsulates that translation logic so migrations are repeatable and auditable rather than manual one-off efforts.

## Key Dependencies and Their Roles

The dependency selection reveals the module's design:

| Dependency | Role |
|---|---|
| `librefang-types` | Target data structures — all migrated output conforms to LibreFang's type definitions |
| `serde_json`, `serde_yaml`, `json5`, `toml` | Multi-format input parsing — agent frameworks use diverse configuration formats |
| `walkdir` | Recursive directory traversal for scanning source projects |
| `thiserror` | Typed error definitions for migration failures |
| `tracing` | Structured logging of migration progress and warnings |
| `chrono`, `uuid` | Timestamping and identity assignment for migrated entities |
| `dirs` | Resolving standard platform directories for default output paths |

## Architecture

```mermaid
flowchart LR
    A[Source Framework Files] --> B[Directory Scanner]
    B --> C[Format Detection & Parsing]
    C --> D[Schema Mapping]
    D --> E[librefang-types Conversion]
    E --> F[Output Writer]
    
    subgraph Input Formats
        JSON
        YAML
        JSON5
        TOML
    end
    
    subgraph Target
        librefang-types
    end
    
    C -.-> Input Formats
    E -.-> Target
```

The migration pipeline follows a linear flow: discover source files, parse them in their native format, map the source schema to LibreFang's data model, and write normalized output.

## Integration with LibreFang

This module depends on `librefang-types` as its sole internal dependency. All migrated data is expressed through the types defined there, ensuring downstream consumers (the runtime, validators, etc.) can work with imported configurations without any special handling.

The module has no incoming calls from other LibreFang crates, indicating it is a standalone tool or CLI-invoked utility rather than a library consumed at runtime.

## Supported Input Formats

The inclusion of four distinct parsers (`serde_json`, `serde_yaml`, `json5`, `toml`) indicates the module handles source configurations in any of these formats. This breadth accommodates the wide variety of configuration formats used across the agent framework ecosystem.

## Error Handling

Migration errors are defined using `thiserror`, providing structured, typed errors for conditions such as:

- Unreadable or missing source files
- Unparseable configuration in any supported format
- Schema mismatches where source concepts lack LibreFang equivalents
- I/O failures during output writing

## Testing

The `tempfile` dev-dependency indicates tests create temporary directories to exercise file discovery, parsing, and output writing in isolation without touching the real filesystem.