# LLM Provider Drivers

# LLM Provider Drivers

This module group provides LibreFang's complete LLM integration layer — a trait-based abstraction that decouples the rest of the codebase from any specific LLM provider, plus concrete driver implementations for seven backends.

## Sub-modules

| Sub-module | Role |
|---|---|
| [librefang-llm-driver](librefang-llm-driver-src.md) | Defines the `LlmDriver` trait, request/response types (`CompletionRequest`, `CompletionResponse`, `StreamEvent`), and error classification infrastructure |
| [librefang-llm-drivers](librefang-llm-drivers-src.md) | Ships concrete `LlmDriver` implementations for Anthropic, OpenAI, AWS Bedrock, Google Vertex AI, Aider, Claude Code, and Qwen Code, along with shared credential pooling, rate-limit tracking, and think-tag filtering |

## How they fit together

```
┌─────────────────────────────────────────────────────┐
│  Consumers: agent_loop, compactor, context_engine,  │
│  proactive_memory, routing, media_understanding     │
└──────────────────────┬──────────────────────────────┘
                       │  calls trait methods only
                       ▼
┌──────────────────────────────────────────────────────┐
│  librefang-llm-driver                                │
│  LlmDriver trait · CompletionRequest · StreamEvent   │
│  LlmError · classify_error · extract_retry_delay     │
└──────────────────────┬──────────────────────────────┘
                       │  implements trait
                       ▼
┌──────────────────────────────────────────────────────┐
│  librefang-llm-drivers                               │
│  Provider impls · CredentialPool · RateLimitTracker  │
│  ThinkFilter · jittered_backoff                      │
└──────────────────────────────────────────────────────┘
```

**librefang-llm-driver** owns the contract: the `LlmDriver` trait (`complete()`, `stream()`), the shared request/response types, and the error taxonomy (`LlmError`, `FailoverReason`) with utilities like `classify_error` and `sanitize_raw_excerpt`. No provider-specific logic lives here.

**librefang-llm-drivers** fills in that contract. Each provider (Anthropic, OpenAI, Bedrock, Vertex AI, Aider, Claude Code, Qwen Code) implements `LlmDriver`. The crate also supplies cross-provider infrastructure:

- **CredentialPool** — round-robin credential rotation that skips exhausted keys.
- **RateLimitTracker** — parses rate-limit headers (ISO 8601 and HTTP date formats) and exposes usage-ratio diagnostics used by the CLI.
- **ThinkFilter** — a streaming filter that strips `<think …>…</think)` blocks from model output.
- **jittered_backoff** — retry pacing shared across all provider implementations.

## Key cross-module workflow

A typical completion request flows like this:

1. A consumer (e.g., `agent_loop`) builds a `CompletionRequest` (types from **llm-driver**).
2. It calls `driver.stream(req)` or `driver.complete(req)` against the `LlmDriver` trait.
3. The concrete implementation in **llm-drivers** acquires a credential from `CredentialPool`, applies `jittered_backoff` for retries, and tracks rate limits via `RateLimitTracker`.
4. On error, the driver calls back into **llm-driver**'s `classify_error` (and related helpers like `extract_retry_delay`, `is_transient`) to determine whether to retry, fail over, or surface the error to the caller.
5. Streaming responses pass through `ThinkFilter` before reaching the consumer as `StreamEvent` values.