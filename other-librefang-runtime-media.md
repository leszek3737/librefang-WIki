# Other — librefang-runtime-media

# librefang-runtime-media

Media generation drivers for [LibreFang](https://github.com/librefang/librefang), providing provider-agnostic abstractions for text-to-speech, image generation, video generation, and music generation.

## Purpose

This crate was extracted from `librefang-runtime` during the #3710 god-crate split. It encapsulates all media generation driver logic behind a unified trait interface, allowing the rest of the codebase to request media output without coupling to any specific provider.

The parent crate `librefang-runtime` re-exports this module at its historical paths (`runtime::media` and `runtime::media_understanding`), gated behind the default-on `media` feature flag. Downstream call sites do not need to change imports.

## Architecture

```mermaid
graph TD
    A[librefang-runtime] -->|"re-exports (media feature)"| B[librefang-runtime-media]
    B --> C[MediaDriver Trait]
    B --> D[MediaDriverCache]
    B --> E[media_understanding]
    C --> F[elevenlabs]
    C --> G[gemini]
    C --> H[google_tts]
    C --> I[minimax]
    C --> J[openai]
    E --> K[speech-to-text]
    E --> L[image/audio analysis]
    B --> M[librefang-types]
    B --> N[librefang-http]
```

## Key Components

### `MediaDriver` Trait

The core abstraction, modeled after `librefang-llm-drivers`. It defines per-modality methods that each provider implements. Modalities a provider does not support are handled gracefully — the default implementation returns a typed `NotSupported` error, so callers can distinguish "this provider can't do video" from "something went wrong generating video."

This design means adding a new provider only requires implementing the methods for modalities that provider actually supports.

### `MediaDriverCache`

A lazy-initializing, thread-safe driver cache backed by `DashMap`. It handles:

- **Lazy init** — drivers are constructed on first access, not at startup.
- **Thread safety** — concurrent requests from the async runtime can safely look up or insert drivers.
- **Per-provider URL overrides** — if a deployment needs to point a provider at a custom endpoint (e.g., a proxy or regional mirror), the cache applies that override during driver construction.

### `media_understanding`

A companion module that routes inbound media analysis requests:

- **Speech-to-text** — transcribing audio input.
- **Image analysis** — describing or extracting information from images.
- **Audio analysis** — processing audio content.

This is the reverse direction from generation: instead of producing media, it consumes media and returns structured data.

## Supported Providers

| Provider | TTS | Image | Video | Music |
|----------|-----|-------|-------|-------|
| `elevenlabs` | ✓ | — | — | — |
| `gemini` | ✓ | ✓ | ✓ | — |
| `google_tts` | ✓ | — | — | — |
| `minimax` | ✓ | ✓ | ✓ | ✓ |
| `openai` | ✓ | ✓ | — | — |

The `elevenlabs` provider includes a voice ID validator that reads the `ELEVENLABS_API_KEY` environment variable. Tests for this validator are serialized (via `serial_test`) to prevent race conditions when mutating that env var concurrently.

## Dependencies

- **`librefang-types`** — shared type definitions (request/response structs, error types).
- **`librefang-http`** — HTTP client construction and shared request plumbing.
- **`tokio`** — async runtime for driver operations.
- **`reqwest`** — HTTP client for provider API calls.
- **`dashmap`** — concurrent map for `MediaDriverCache`.
- **`async-trait`** — trait object support for async methods on `MediaDriver`.
- **`tracing`** — structured logging throughout driver implementations.
- **`base64`** / **`hex`** — encoding utilities for media payload handling.

## Adding a New Provider

1. Create a new module under the crate root named after the provider.
2. Implement the `MediaDriver` trait, filling in methods for supported modalities. Unsupported modalities will automatically return `NotSupported`.
3. Register the provider in `MediaDriverCache` so it can be lazily initialized and looked up.
4. Add any required configuration fields to the relevant types in `librefang-types`.
5. Ensure any env-var–dependent tests use `serial_test` to avoid race conditions.

## Relationship to the Workspace

```
crates/
├── librefang-types/           # Shared types
├── librefang-http/            # HTTP client utilities
├── librefang-runtime/         # Re-exports this crate
└── librefang-runtime-media/   # ← you are here
```

This crate sits between the low-level HTTP/types crates and the main runtime. It is consumed exclusively through `librefang-runtime`'s re-exports, which preserves the existing public API while keeping the implementation modular.