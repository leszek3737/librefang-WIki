# Other — librefang-migrate

# librefang-migrate

Migration engine for importing configurations, agents, and related data from other C2 agent frameworks into LibreFang's native format.

## Purpose

When transitioning to LibreFang from another command-and-control agent framework, operators typically need to port existing agent configurations, listener definitions, credential stores, or other operational data. `librefang-migrate` provides the tooling to parse foreign framework export formats and convert them into LibreFang-compatible types defined in `librefang-types`.

## Supported Input Formats

The module depends on multiple deserialization libraries, indicating support for reading migration sources in:

| Format | Crate | Typical Use |
|---|---|---|
| JSON | `serde_json` | Standard API exports, structured dumps |
| YAML | `serde_yaml` | Configuration files, human-edited exports |
| JSON5 | `json5` | Relaxed JSON with comments, some framework configs |
| TOML | `toml` | Rust-ecosystem and some tool configurations |

All deserialization is driven through `serde`, so any format can be used as long as the source data maps to the expected schema.

## Key Dependencies

- **`librefang-types`** — The target type system. All migration output is expressed as types from this crate (agent configs, listener definitions, etc.).
- **`walkdir`** — Recursive directory traversal for discovering migration source files when importing from a directory tree rather than a single file.
- **`uuid`** / **`chrono`** — Generating new identifiers and timestamps during migration, since source frameworks use incompatible ID schemes.
- **`dirs`** — Resolving standard platform directories (e.g., locating a source framework's default config path).
- **`thiserror`** — Typed error definitions for migration failures.
- **`tracing`** — Structured logging of migration progress and warnings.

## Architecture

Migration generally follows a three-stage pipeline:

```mermaid
flowchart LR
    A[Discover Source Files] --> B[Parse Foreign Format]
    B --> C[Convert to librefang-types]
    C --> D[Emit / Write Output]
```

1. **Discovery** — Locate source files on disk using `walkdir` or accept an explicit path. When importing from an installed framework, `dirs` can resolve default installation paths.

2. **Parsing** — Deserialize the source file into an intermediate representation using the appropriate format crate. Invalid or partial data is reported through `thiserror`-based error types and logged via `tracing`.

3. **Conversion** — Map the intermediate representation to `librefang-types` structures. This is where framework-specific field mapping, ID regeneration (`uuid`), and timestamp normalization (`chrono`) occur.

4. **Output** — The caller receives native LibreFang types ready for serialization or direct use in the rest of the application.

## Error Handling

Migration failures are represented as typed errors using `thiserror`. Common failure modes include:

- **Unrecognized format** — The source file cannot be parsed as any supported format.
- **Schema mismatch** — The file parses but does not match the expected structure for the source framework.
- **Validation failure** — The converted LibreFang type fails validation constraints defined in `librefang-types`.
- **I/O errors** — Filesystem issues during discovery or reading.

All errors implement `std::error::Error` and carry context about which file and field caused the failure.

## Testing

The `tempfile` dev-dependency is used to construct isolated directory trees in tests. Migration tests typically:

1. Create a temporary directory populated with synthetic source files in the foreign framework's format.
2. Run the migration pipeline against the temp directory.
3. Assert that the output converts to the expected `librefang-types` structures.
4. Assert that malformed inputs produce the expected error variant.

## Relationship to Other Crates

`librefang-migrate` is a pure consumer of `librefang-types`. It produces those types but does not depend on any other LibreFang crate — no runtime, no database, no networking. This keeps the migration engine testable in isolation and suitable for inclusion in either a CLI tool or a larger application.

No other crate in the workspace currently calls into `librefang-migrate` at compile time; it is expected to be invoked explicitly (e.g., from a binary or a plugin) rather than linked automatically.