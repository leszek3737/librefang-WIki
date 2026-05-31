# LLM Drivers

# LLM Drivers

Provider abstraction layer for LibreFang. This module group defines a uniform interface for multiple LLM providers and implements the fallback routing, credential management, and retry logic that keep completions flowing even when individual providers or keys degrade.

## Sub-modules

| Crate | Role |
|-------|------|
| [**librefang-llm-driver**](librefang-llm-driver-src.md) | Core `LlmDriver` trait, `CompletionRequest`/response types, `LlmError` classification engine, `ProviderExhaustionStore`, and `FallbackChain` orchestration |
| [**librefang-llm-drivers**](librefang-llm-drivers-src.md) | Concrete provider drivers (Anthropic, OpenAI/ChatGPT, Bedrock, CLI tools), `CredentialPool`, backoff/retry logic, and rate-limit guards |

## How they fit together

`librefang-llm-driver` owns the *contract*: the trait every provider implements, the request/response types that flow through the system, and the error classifier that decides whether a failure is retryable, should trigger a provider switch, or requires a key rotation. It knows nothing about HTTP or subprocess calls.

`librefang-llm-drivers` fills in the *implementations*: each driver translates a `CompletionRequest` into provider-native HTTP or CLI subprocess calls, relies on `CredentialPool` to select an unexhausted API key, and applies jittered backoff on transient failures. Drivers return `LlmError` variants that the core crate classifies into `FailoverReason` values.

## Key cross-module workflow

```
Agent Loop
  │
  ▼
FallbackChain (core)
  │  checks ProviderExhaustionStore
  │  selects next provider + credential
  ▼
Concrete Driver (drivers)
  │  CredentialPool selects unexhausted key
  │  jittered backoff on transient errors
  │  HTTP / SSE / subprocess call
  ▼
Response ──or── LlmError
                │
                ▼
           classify_error (core)
                │
                ▼
           FailoverReason
           ├─ Retryable    → backoff, same provider
           ├─ KeyExhausted → rotate key via CredentialPool
           └─ ProviderDown → mark exhausted, next provider
```

1. **Dispatch** — `FallbackChain` checks `ProviderExhaustionStore` for live providers, then hands a `CompletionRequest` to the chosen driver.
2. **Credential selection** — The driver calls `CredentialPool::fill` to get an available key, skipping any previously marked exhausted.
3. **Request** — The driver makes the provider-native call (HTTP for API providers, stdin/subprocess for CLI drivers like Claude Code, Gemini CLI, and Qwen Code).
4. **Error classification** — On failure, `classify_error` inspects the raw error (HTML pages, JSON messages, timeouts, rate-limit headers) and produces a `FailoverReason`.
5. **Feedback loop** — `ProviderExhaustionStore` marks providers or keys as exhausted; `CredentialPool::mark_success` clears exhaustion on recovery, incrementing request counts for diagnostics.

## Provider landscape

The drivers crate supports three families of providers:

- **API drivers** — Anthropic, OpenAI/ChatGPT (including Responses API with JSON schema formats), and AWS Bedrock. These speak HTTP/SSE and use `CredentialPool` for key rotation.
- **CLI drivers** — Claude Code, Gemini CLI, Qwen Code. These invoke local CLI tools via subprocess, discovering credentials from the user's home directory.
- **Availability checks** — Each CLI driver exposes an `*_available` function that probes for the binary and its credentials, used by provider-test routes to report what's reachable.

For implementation details, see the individual sub-module pages linked above.