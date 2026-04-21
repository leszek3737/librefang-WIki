# LLM Providers

# LLM Providers

The LLM Providers module group supplies LibreFang's complete LLM integration layer — from abstract driver interface to concrete provider implementations to runtime orchestration including model selection, health probing, and retry logic.

## Sub-Modules

| Module | Responsibility |
|---|---|
| [librefang-llm-driver](librefang-llm-driver-src.md) | Core `LlmDriver` trait, request/response types, streaming event protocol, and error classification |
| [librefang-llm-drivers](librefang-llm-drivers-src.md) | Static provider registry, driver factory (`create_driver`), thread-safe `DriverCache`, and 40+ concrete driver implementations |
| [librefang-runtime](librefang-runtime-src.md) | Model catalog (TOML-backed), authentication detection, provider health probing, complexity-based routing, and retry utility |

## How They Fit Together

```
┌─────────────────── librefang-runtime ───────────────────┐
│  ModelCatalog ──► ModelRouter ──► TaskComplexity scoring │
│  ProviderHealth / ProbeCache                             │
│  RetryConfig / RetryOutcome                              │
└──────────────────────┬──────────────────────────────────┘
                       │ DriverConfig
                       ▼
┌─────────────────── librefang-llm-drivers ───────────────┐
│  create_driver(DriverConfig)                            │
│    → PROVIDER_REGISTRY lookup                           │
│    → concrete driver instantiation                      │
│  DriverCache (thread-safe reuse)                        │
└──────────────────────┬──────────────────────────────────┘
                       │ implements
                       ▼
┌─────────────────── librefang-llm-driver ────────────────┐
│  LlmDriver trait (.complete / .stream)                  │
│  CompletionRequest → CompletionResponse / StreamEvent   │
│  LlmError → classify_error → ClassifiedError            │
└─────────────────────────────────────────────────────────┘
```

## Key Workflows

**Model selection and driver creation.** The runtime's `ModelCatalog` loads provider definitions from bundled TOML files, user aliases, and discovered custom models. `ModelRouter` scores candidates based on `TaskComplexity` and availability. The resulting `DriverConfig` is passed to `create_driver`, which looks up the provider in the static `PROVIDER_REGISTRY` and returns the appropriate concrete driver (e.g., `OpenAIDriver`, `AnthropicDriver`, `GeminiDriver`, or a CLI subprocess driver). Drivers are cached in a thread-safe `DriverCache` for reuse.

**Request execution.** Agent loops call `LlmDriver::complete` or `LlmDriver::stream` with a `CompletionRequest`. The concrete driver translates this into the provider's wire format and returns a `CompletionResponse` or a stream of `StreamEvent`s. Streaming responses pass through the think filter to sanitize `<think)` blocks.

**Error handling and retries.** Any failure produces an `LlmError`, which `classify_error` maps to a `ClassifiedError` indicating retryability and extracted backoff hints. The runtime's retry utility uses this classification alongside `RetryConfig` to drive exponential backoff with jitter, ultimately producing a `RetryOutcome` that either delivers a successful response or surfaces a final error to the user.

**Health and auth detection.** `provider_health` probes API key validity and endpoint reachability, caching results in `ProbeCache`. The model catalog's `detect_auth` integrates probe results to filter models to only those the user can actually access, ensuring the router never selects an unavailable provider.