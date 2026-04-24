# Other — librefang-llm-driver

# librefang-llm-driver

LLM driver trait and shared types for LibreFang.

## Purpose

This crate defines the abstraction layer through which the rest of the LibreFang application communicates with large language model backends. It provides a common trait (interface) that concrete LLM implementations must satisfy, along with shared types for requests, responses, and errors.

By keeping the driver contract in its own crate, the rest of the codebase depends only on this abstraction—not on any specific LLM provider. This allows swapping or adding backends without touching core game logic.

## Role in the Architecture

```
┌─────────────────────────────────────────────────┐
│               LibreFang Core                     │
│  (depends on librefang-llm-driver trait only)    │
└──────────────┬──────────────────────────────────┘
               │ calls through trait
               ▼
┌──────────────────────────────────┐
│     librefang-llm-driver         │
│  • Driver trait definition       │
│  • Shared request/response types │
│  • Error types                   │
└──────────────┬───────────────────┘
               │ trait is implemented by
               ▼
┌──────────────────────────────────┐
│   Concrete LLM driver crates     │
│  (e.g. OpenAI, local, etc.)      │
└──────────────────────────────────┘
```

## Dependencies and Their Rationale

| Dependency | Why It's Here |
|---|---|
| `librefang-types` | Shares domain types (game-specific definitions) that the driver needs to interpret or produce. |
| `async-trait` | The driver trait is async. `async-trait` provides the procedural macro to make `async fn` work in trait definitions, since the driver will perform network I/O against LLM APIs. |
| `serde` / `serde_json` | Request and response types are serializable. LLM APIs speak JSON, so the driver's shared types need `Serialize`/`Deserialize` bounds. |
| `thiserror` | Defines structured error types for driver failures (API errors, rate limits, malformed responses, etc.). |
| `tokio` | Async runtime. The trait's async methods rely on Tokio for I/O and task execution. |

## What This Crate Provides

### Driver Trait

The central export is an async trait that concrete LLM backends implement. Any crate that needs to call an LLM accepts a boxed or generic instance of this trait, keeping the consumer decoupled from provider specifics.

### Shared Types

Request and response structures that are provider-agnostic. These give the rest of the codebase a stable vocabulary for talking to LLMs regardless of which backend is active.

### Error Types

A common error enum covering the failure modes any LLM driver can hit—network issues, authentication failures, rate limiting, and unexpected response formats. Consumers handle one error type rather than provider-specific ones.

## Integration Guide

**Consuming the trait** from another crate:

1. Add `librefang-llm-driver` as a dependency.
2. Accept the driver trait (typically `Box<dyn DriverTrait>`) in your component's constructor or function signature.
3. Call the trait methods. You never reference a concrete provider type.

**Implementing a new backend**:

1. Create a new crate that depends on `librefang-llm-driver`.
2. Define a struct representing your provider's client.
3. Implement the driver trait for that struct, translating the shared request types into provider-specific HTTP calls and mapping provider-specific responses back into the shared response types.
4. Map provider-specific errors into the shared error enum.

## Design Notes

- This crate intentionally has **no outbound calls**. It defines types and traits only. All I/O happens in concrete implementation crates.
- Because it depends on `librefang-types`, the driver trait can express operations in terms of the game's domain vocabulary rather than raw strings.