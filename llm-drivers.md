# LLM Drivers

# LLM Drivers

The LLM Drivers module group provides LibreFang's unified LLM provider abstraction — a trait-based interface that decouples the rest of the platform from any single model vendor.

## Sub-modules

| Crate | Role |
|---|---|
| [**librefang-llm-driver**](librefang-llm-driver-src.md) | Core `LlmDriver` trait, shared types (`CompletionRequest`, `CompletionResponse`, `LlmError`), exhaustion tracking, and error sanitisation infrastructure |
| [**librefang-llm-drivers**](librefang-llm-drivers-src.md) | Concrete driver implementations, retry/backoff logic, rate-limit guards, credential pooling, prompt caching, and stream decoding |

## How they fit together

```
┌─────────────────────────────────────────────────┐
│  Consumers: kernel, FallbackChain, agent_loop   │
│              metering, ws / API layer            │
└──────────────┬──────────────────────────────────┘
               │ depends on trait + types
┌──────────────▼──────────────────────────────────┐
│         librefang-llm-driver (core)              │
│  LlmDriver trait · ProviderExhaustionStore       │
│  LlmError · FailoverReason · CompletionRequest   │
└──────────────┬──────────────────────────────────┘
               │ implements trait
┌──────────────▼──────────────────────────────────┐
│        librefang-llm-drivers (implementations)   │
│  Anthropic · OpenAI · Gemini · Bedrock · Ollama  │
│  Claude Code · Codex CLI · Aider · Qwen · …     │
│  + backoff · rate_limit_tracker · utf8_stream    │
└─────────────────────────────────────────────────┘
```

`librefang-llm-driver` owns the interface contract. Every concrete driver in `librefang-llm-drivers` implements the `LlmDriver` trait's `complete` and `stream` methods against a specific provider API. This split keeps downstream crates (kernel, FallbackChain, agent_loop) depending only on the thin trait crate, while the heavier HTTP/gRPC/CLI integrations live behind an optional dependency boundary.

## Key cross-cutting workflows

- **Failover / exhaustion tracking** — `FallbackChain` queries `ProviderExhaustionStore` (core) to decide which driver to call next; individual drivers record failures back into the same shared `Arc<DashMap>`.
- **Rate limiting** — `RateLimitBucket` and `RateLimitSnapshot` (drivers) are consumed by `shared_rate_guard`'s cooldown-picker logic, enabling per-provider RPM/RPH throttling before requests hit the wire.
- **Retry with jittered backoff** — failed calls flow through `jittered_backoff` (drivers), which wraps exponential delay with configurable jitter before the caller retries against the same or a fallback driver.
- **Error sanitisation** — raw provider error bodies pass through `sanitize_raw_excerpt` → `cap_message` (core) so logs and API responses never leak full prompt text or credentials.
- **Credential bootstrapping** — CLI-based drivers (Claude Code, Codex CLI, etc.) probe the filesystem at startup (`home_dir`, `claude_credentials_exist`, `codex_cli_credentials_exist`) and cache tokens; this chain is invoked from the API route layer during provider availability checks.