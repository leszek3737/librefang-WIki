# Other — librefang-llm-driver

# librefang-llm-driver

Trait definitions and shared types for LLM (Large Language Model) integration within LibreFang.

## Purpose

This crate provides the abstraction layer between LibreFang's core logic and any specific LLM backend. It defines a common driver trait that concrete implementations must satisfy, along with the request/response types and error definitions those implementations share.

By keeping the driver contract in its own crate, the rest of the codebase depends only on this abstraction—not on any particular LLM provider. Concrete drivers (e.g., OpenAI, local models) can then be swapped or added without modifying downstream consumers.

## Architectural Role

```
┌─────────────────────┐       ┌──────────────────────┐       ┌─────────────────────┐
│  LibreFang consumers │──────▶│ librefang-llm-driver │◀──────│  Concrete drivers   │
│  (analysis, chat,    │  uses │  (trait + types)      │ impls │  (OpenAI, local, …) │
│   moderation, etc.)  │  trait│                      │ trait │                     │
└─────────────────────┘       └──────────────────────┘       └─────────────────────┘
```

This is a pure interface module. It contains no business logic and no network calls—only the shapes that other crates agree on.

## Dependencies

| Crate | Reason |
|---|---|
| `librefang-types` | Domain types (game identifiers, player data, etc.) that appear in LLM prompts or responses |
| `async-trait` | Enables async methods in the driver trait, since LLM calls are inherently I/O-bound |
| `serde` / `serde_json` | Serialization of requests to and deserialization of responses from LLM backends |
| `thiserror` | Derive `Error` for the crate's error enum |
| `tokio` | Async runtime support |

## Key Components

### Driver Trait

The central export is an async trait that concrete LLM backends implement. Each method represents a distinct category of LLM interaction used by LibreFang. Because the trait is async, every consuming call site must be within a Tokio runtime context.

Implementors handle all provider-specific concerns—authentication, HTTP clients, retries, rate limiting—internally. Callers see only the trait methods and the shared request/response types.

### Shared Types

Request and response structs that are generic across all LLM providers. These are typically serde-serializable and may include:

- Prompt or message containers
- Configuration knobs (temperature, max tokens, model selection)
- Structured response types that downstream code pattern-matches on

### Error Type

A unified error enum (derived via `thiserror`) representing failure modes common to any LLM backend—network errors, rate limiting, invalid responses, deserialization failures, and similar. Each concrete driver maps its provider-specific errors into this common type so that consumers handle a single error surface.

## Usage from Other Crates

Consumer crates depend on `librefang-llm-driver` and accept the driver trait as a parameter (typically via dependency injection). They never reference a concrete driver directly:

```rust
// In a consumer crate
use librefang_llm_driver::LlmDriver;

async fn analyze_submission(driver: &dyn LlmDriver, context: &GameContext) -> Result<Analysis, LlmError> {
    let request = AnalysisRequest::new(context);
    driver.analyze(request).await
}
```

Concrete driver crates also depend on this module and provide an `impl LlmDriver for ProviderClient`.

## Adding a New LLM Backend

1. Create a new crate (e.g., `librefang-llm-openai`).
2. Depend on `librefang-llm-driver`.
3. Implement the driver trait for your provider's client struct.
4. Map all provider-specific errors into the shared `LlmError` enum.
5. Register the driver in the application's service wiring.

Because the trait and types live here, the new driver crate needs no knowledge of LibreFang's core domain—only of the driver contract defined in this module.