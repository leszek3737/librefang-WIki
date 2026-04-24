# LLM Drivers — librefang-llm-drivers-src

# librefang-llm-drivers

Multi-provider LLM driver layer that abstracts completion and streaming requests behind a unified trait, with built-in credential pooling, jittered retry, rate-limit observability, and streaming content filtering.

## Architecture

```mermaid
graph TD
    subgraph "Consumer Layer"
        RT[librefang-runtime]
        AG[librefang-types / agent.rs]
    end

    subgraph "Core Trait"
        LD[LlmDriver trait<br>complete() / stream()]
    end

    subgraph "Drivers"
        AN[AnthropicDriver]
        OAI[OpenAIDriver]
        CGPT[ChatGptDriver]
        AID[AiderDriver]
        QC[QwenCodeDriver]
        CC[ClaudeCodeDriver]
        VA[VertexAIDriver]
        TR[TokenRotationDriver]
    end

    subgraph "Infrastructure"
        BO[backoff::jittered_backoff]
        CP[CredentialPool]
        RLT[RateLimitTracker]
        TF[ThinkFilter]
    end

    RT --> LD
    AG --> LD
    LD --> AN
    LD --> OAI
    LD --> CGPT
    LD --> AID
    LD --> QC
    LD --> CC
    LD --> VA
    LD --> TR
    AN --> BO
    AN --> RLT
    OAI --> BO
    OAI --> RLT
    TR --> CP
    RT --> TF
```

## Core Abstraction: `LlmDriver`

All providers implement the `LlmDriver` async trait defined in `llm_driver.rs`. The trait exposes two methods:

- **`complete(request: CompletionRequest) → Result<CompletionResponse, LlmError>`** — single-shot request/response.
- **`stream(request: CompletionRequest, tx: Sender<StreamEvent>) → Result<CompletionResponse, LlmError>`** — server-sent events streamed incrementally over `tx`, with a final `CompletionResponse` returned when the stream ends.

`CompletionRequest` carries the model ID, message history, system prompt, tools, temperature, max tokens, thinking configuration, prompt caching flag, response format, and an optional timeout override. `CompletionResponse` normalises content blocks (text, tool use, thinking), stop reason, parsed tool calls, and token usage across all providers.

`LlmError` distinguishes `Http`, `Api { status, message }`, `RateLimited`, `Overloaded`, `Parse`, `MissingApiKey`, and `AuthenticationFailed` — callers can pattern-match to decide whether to retry, rotate credentials, or surface an error to the user.

---

## Drivers

### AnthropicDriver (`drivers/anthropic.rs`)

Calls the Anthropic Messages API (`/v1/messages`). Key features:

- **Prompt caching** — when `request.prompt_caching` is true, `cache_control: {"type": "ephemeral"}` markers are stamped on the system prompt, the last tool definition, and the last content block of the last user message. This causes Anthropic to cache the full prefix (system + tools + conversation history), yielding cost savings on multi-turn conversations.
- **Extended thinking** — when `request.thinking` has a `budget_tokens ≥ 1024`, the driver enables Anthropic's thinking mode and automatically raises `max_tokens` to exceed the budget.
- **Tool use** — full round-trip support. Tool inputs are normalised to JSON objects via `ensure_object()`, which wraps non-object values in `{"raw_input": ...}` so malformed model outputs are preserved rather than silently dropped.
- **Streaming** — SSE events (`content_block_start`, `content_block_delta`, `content_block_stop`, `message_delta`) are parsed incrementally. Tool call arguments are accumulated across deltas and parsed at `content_block_stop`.
- **Retry** — 429 (rate-limited) and 529 (overloaded) responses trigger up to 3 retries with `standard_retry_delay()`, honouring any `Retry-After` header as a floor.
- **Response format** — Anthropic has no native `response_format` field; the driver appends JSON/schema instructions to the system prompt instead.
- **Rate-limit headers** — parsed via `RateLimitSnapshot::from_headers()` and logged at warn/debug level.

`build_anthropic_request()` is the shared function that constructs the `ApiRequest` for both `complete()` and `stream()`, ensuring identical wire format.

### ChatGptDriver (`drivers/chatgpt.rs`)

Calls the ChatGPT Responses API (`/codex/responses`) using OAuth session tokens rather than API keys. This is distinct from the OpenAI `/v1/chat/completions` endpoint — OAuth tokens with `api.connectors` scopes only work with the Responses API.

**Token lifecycle:**
1. Session token is provided at construction (from `librefang auth chatgpt`).
2. The token is cached in `ChatGptTokenCache` with a 7-day TTL and 1-hour refresh buffer.
3. If the API responds with 401/403, `refresh_token()` uses the `CHATGPT_REFRESH_TOKEN` env var to obtain a new access token via `librefang_runtime_oauth`.
4. Post-refresh 401/403 is raised as `AuthenticationFailed` with a re-authentication hint.

**Streaming:** The driver always sends `stream: true`. `stream_sse()` handles Responses API events including `response.output_text.delta`, `response.reasoning.delta`, `response.output_item.added` (function calls), `response.function_call_arguments.delta/done`, and `response.completed`. If streaming deltas were missed, `backfill_from_completed_output()` extracts content from the final `response.completed` payload.

### AiderDriver (`drivers/aider.rs`)

Spawns the `aider` CLI as a non-interactive subprocess. Aider handles its own LLM provider authentication via standard environment variables (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.).

- `detect()` checks whether `aider --version` succeeds.
- `build_args()` constructs CLI flags: `--message`, `--yes-always`, `--no-auto-commits`, `--no-git`, and optionally `--model` (the `aider/` prefix is stripped from model IDs).
- The multi-turn message history is flattened into a single `[System]\n…\n[User]\n…` text prompt.
- Token usage is reported as zero (Aider doesn't expose it).

### Other Drivers (referenced in call graph)

| Driver | Transport | Auth |
|--------|-----------|------|
| `OpenAIDriver` | HTTP `/v1/chat/completions` | API key |
| `VertexAIDriver` | HTTP Vertex AI endpoint | Google service account JWT (self-signed RSA-SHA256, no external JWT library) |
| `GeminiDriver` | Used by Vertex AI for tool conversion | — |
| `QwenCodeDriver` | CLI subprocess (`qwen-code`) | Credentials file from `~/.qwen/` |
| `ClaudeCodeDriver` | CLI subprocess (`claude`) | Credentials file check |
| `TokenRotationDriver` | Wraps another driver | Delegates credential selection to `CredentialPool` |

---

## Infrastructure Modules

### Backoff (`backoff.rs`)

Jittered exponential backoff for retry loops.

**Formula:** `delay = max(exp_delay, floor) + jitter`

Where `exp_delay = min(base × 2^(attempt−1), max_delay)` and `jitter ∈ [0, jitter_ratio × base_for_jitter]`.

The random seed combines `SystemTime::now().subsec_nanos()` XOR'd with a process-global monotonic Weyl-sequence counter (`JITTER_COUNTER`), ensuring seed diversity even when the OS clock has coarse granularity or multiple threads retry simultaneously.

All exponential computation is done in `f64` space to avoid `Duration::mul_f64` panicking on overflow at high attempt numbers. The result is clamped to `max_delay` before constructing a `Duration`.

| Function | Base | Cap | Jitter | Floor |
|----------|------|-----|--------|-------|
| `standard_retry_delay(attempt, floor)` | 2 s | 60 s | 50% | Caller-supplied (e.g. `Retry-After`), capped at 300 s |
| `tool_use_retry_delay(attempt)` | 1.5 s | 60 s | 50% | None |

### Credential Pool (`credential_pool.rs`)

Thread-safe pool of API keys for a single provider, designed for failover when one key hits rate limits or quota exhaustion.

**Strategies:**

| Strategy | Behaviour |
|----------|-----------|
| `FillFirst` | Always pick the highest-priority available key; fall back only when exhausted. |
| `RoundRobin` (default) | Cycle through available keys in priority order, distributing load. |
| `Random` | Pick a random available key using a time-seeded LCG (avoids `rand` dependency). |
| `LeastUsed` | Pick the key with the lowest `request_count`. |

**Lifecycle:**
1. `acquire()` — returns a cloned API key string, or `None` if all keys are exhausted.
2. `mark_exhausted(key)` — places the key in cooldown for `exhausted_ttl` (default 1 hour).
3. `mark_success(key)` — increments `request_count` and clears any exhaustion marker (early recovery).

The inner state (`credentials` + `round_robin_idx`) is protected by a single `Mutex`, so the index read and credential selection are atomic — no TOCTOU between concurrent `acquire()` calls.

`CredentialSnapshot` provides a redacted view (`****abcd`) for diagnostics and dashboards. `ArcCredentialPool` is the standard shared handle type.

### Rate Limit Tracker (`rate_limit_tracker.rs`)

Parses HTTP rate-limit headers into a structured `RateLimitSnapshot`. Supports:
- Token-bucket headers (`x-ratelimit-limit-*`, `x-ratelimit-remaining-*`)
- Reset timestamps (ISO 8601, HTTP-date, or seconds-from-now)
- `has_warning()` / `display()` for log output

The `display()` formatter produces ASCII progress bars and usage ratios, consumed throughout the codebase (migrations, tool runners, plugin runtime, etc.).

### Think Filter (`think_filter.rs`)

Stateful streaming filter that strips `<think …>…</think)` blocks from LLM output. Designed for models that emit reasoning traces in custom XML-like tags. The filter operates on partial text deltas and correctly handles tags split across multiple `process()` calls. Its `flush()` method is called from the wire layer and process manager when output ends.

---

## Error Handling and Retry Pattern

All HTTP-based drivers follow the same retry pattern:

1. Attempt the request (up to `max_retries + 1` total attempts).
2. On 429 or provider-specific overload (e.g. Anthropic 529), extract `Retry-After` header if present.
3. Sleep for `standard_retry_delay(attempt, retry_after_floor)`.
4. On final failure, return `LlmError::RateLimited` or `LlmError::Overloaded`.
5. Non-retryable errors (4xx except 429) return immediately as `LlmError::Api`.

For credential exhaustion (402, 429 with quota depletion), `TokenRotationDriver` uses `CredentialPool::mark_exhausted()` to rotate to the next key.

---

## Integration Points

- **`librefang-runtime`** — consumes `LlmDriver` implementations to execute agent turns. Uses `ThinkFilter` to clean streaming output and `RateLimitTracker` for observability.
- **`librefang-types`** — defines `CompletionRequest`, `CompletionResponse`, `Message`, `ContentBlock`, `TokenUsage`, `ToolDefinition`, `ToolCall`, and `ResponseFormat` shared across all drivers.
- **`librefang-http`** — provides `proxied_client()` and `proxied_client_with_override()` for HTTP clients that respect system proxy settings.
- **`librefang-runtime-oauth`** — provides `chatgpt_oauth::refresh_access_token()` for ChatGPT token renewal.
- **`librefang-extensions/vault`** — credential file existence checks for CLI-based drivers (`claude_credentials_exist`, `qwen_credentials_exist`).