# Other — librefang-runtime-src

# librefang-runtime: Agent Loop

## Overview

`librefang-runtime` houses the **agent loop** — the central orchestration engine that drives multi-turn LLM conversations with tool use, streaming, and robust error recovery. The module is responsible for:

- Invoking LLM completions (via the `LlmDriver` trait)
- Dispatching tool calls and injecting results back into the conversation
- Guarding against empty responses, cascade leaks, and hallucinated actions
- Managing session persistence and history folding
- Streaming partial results to callers in real time

## Architecture

```mermaid
graph TD
    A[run_agent_loop / run_agent_loop_streaming] --> B{LLM Complete}
    B --> C[EndTurn]
    B --> D[ToolUse]
    B --> E[MaxTokens]
    C --> F[finalize_end_turn_text]
    F --> G{Empty?}
    G -->|Yes| H[Fallback / Retry]
    G -->|No| I[Return response]
    D --> J[execute_tool]
    J --> K[Record result]
    K --> B
    E --> L{Continuations < MAX?}
    L -->|Yes| M[Append "Please continue."]
    L -->|No| N[Return partial + fallback]
    M --> B
```

## Entry Points

### `run_agent_loop`

The synchronous (non-streaming) entry point. Accepts an `AgentManifest`, user message, `Session`, `MemorySubstrate`, an `LlmDriver` implementation, and a slice of `ToolDefinition`s. Returns `AgentLoopResult` containing the final text, iteration count, directives, owner notices, and provider metadata.

### `run_agent_loop_streaming`

The streaming variant. Accepts an additional `mpsc::Sender<StreamEvent>` and emits `TextDelta`, `OwnerNotice`, and other events as the loop progresses. Shares the same core logic but forwards partial results in real time.

Both functions accept a large set of optional parameters (skill registries, MCP connections, media engines, Docker config, etc.) that are passed through to subsystems.

## Core Loop Mechanics

### Iteration Model

The loop runs up to `MAX_ITERATIONS` (default from `AutonomousConfig::DEFAULT_MAX_ITERATIONS`). The effective cap is resolved as:

1. Per-agent manifest `max_iterations` → takes precedence
2. `LoopOptions.max_iterations` → fallback
3. Library default → final fallback

On each iteration:
1. Build a `CompletionRequest` from session messages + user input
2. Call `driver.complete()` (or `driver.stream()`)
3. Inspect `stop_reason`:
   - **EndTurn** → finalize and return
   - **ToolUse** → execute tools, inject results, continue
   - **MaxTokens** → accumulate partial text, inject "Please continue." prompt, continue (up to `MAX_CONTINUATIONS`)

### Tool Resolution

`resolve_request_tools` decides which tools to include in each LLM request:

- **Eager mode** (default): sends all available tools
- **Lazy mode**: when the tool pool exceeds `LAZY_TOOLS_THRESHOLD` and `tool_load` is present, sends only a subset plus the `tool_load` meta-tool

`ResolvedToolsCache` avoids re-cloning the tool list on every iteration when inputs are stable (keyed by pool contents + session-loaded tools + lazy mode flag).

## Response Finalization

### `finalize_end_turn_text`

Three-way fallback when the LLM produces its final turn:

| Condition | Behavior |
|-----------|----------|
| Final text non-empty | Use final text directly |
| Final text empty, accumulated buffer non-empty | Use accumulated buffer |
| Both empty, tools were executed | Emit "Task completed" guard |
| Both empty, no tools | Emit "empty response" fallback |

### `push_accumulated_text`

Appends text to a running buffer with `"\n\n"` separation. Capped at `ACCUMULATED_TEXT_MAX_BYTES` — once sealed, further pushes are silently dropped. The original prefix is always preserved.

## Guard Systems

### Empty Response Handling

When an LLM returns an empty `EndTurn` after a tool-use cycle (a known edge case), the loop produces a fallback message rather than an empty string. The iteration-0 empty-response path includes a **one-shot retry**: if the first call returns empty `EndTurn`, the loop retries once before emitting the fallback.

### Cascade Leak Detection (`is_cascade_leak`)

Detects when the LLM leaks internal structural markers into its output — for example, WhatsApp envelope headers (`[Group message from ...]`) and turn frames (`User asked: ... / I responded: ...`). Requires **two or more** structural markers to trip; thematic headers alone (e.g., `## Calendar`) are legitimate content and do not trigger the guard.

When triggered, the turn is silently dropped: `result.silent = true` and `result.response = ""`.

The guard operates in both streaming and non-streaming paths. In streaming, an incremental check can abort mid-stream even if the final `stop_reason` is `ToolUse`.

### Hallucinated Action Detection (`looks_like_hallucinated_action`)

Catches LLMs claiming to have performed actions (file creation, message sending, payment booking) without actually calling tools. Matches patterns in English ("I've created...", "Successfully modified...") and Italian ("Ho registrato...", "Messaggio inviato."). Neutral text like questions or conditional statements does not trigger the guard.

### Silent Response (`is_no_reply`)

Recognizes sentinel tokens like `NO_REPLY`, `[no reply needed]`, and `no reply needed` (exact match). Silent results are constructed via `build_silent_agent_loop_result`, which enforces `response == ""`.

### Progress Text Leak (`is_progress_text_leak`)

Catches short (<120 char) ellipsis-terminated preamble text with no tool use (e.g., "Let me check that..."). These are LLM thinking-out-loud fragments that should not reach the user.

## Text-to-Tool-Call Recovery (`recover_text_tool_calls`)

Some models (notably Groq/Llama, Qwen3) emit tool calls as plain text instead of structured `tool_calls` fields. The recovery system parses **nine patterns**:

| # | Pattern | Example |
|---|---------|---------|
| 1 | `<function=NAME>{JSON}</function>` | `<function=web_search>{"query":"rust"}</function>` |
| 2 | `<function>NAME{JSON}</function>` | `<function>web_search{"url":"..."}</function>` |
| 3 | `<tool>NAME{JSON}</tool>` | `<tool>exec{"command":"ls"}</tool>` |
| 4 | Markdown code block: ` ``` NAME {JSON} ``` ` | With or without language tag |
| 5 | Backtick-wrapped: `` `NAME {JSON}` `` | `` `exec {"command":"pwd"}` `` |
| 6 | `[TOOL_CALL]...[/TOOL_CALL]` | JSON or arrow syntax (`{tool => "name", args => {...}}`) |
| 7 | `ditorJSONsya` (Qwen3 XML-style delimiters) | Various field names: `name`/`function`, `arguments`/`parameters` |
| 8 | Bare JSON tool call object | `{"name": "...", "arguments": {...}}` (fallback) |
| 9 | `<function name="..." parameters="..." />` | XML attribute style with HTML entities |

All recovered calls are validated against the available `ToolDefinition` list. Unknown tools, invalid JSON, and unclosed tags are silently skipped. Deduplication prevents the same call from appearing twice across different patterns.

Helper functions:
- `parse_dash_dash_args` — converts `--key "value"` arrow syntax into JSON
- `parse_json_tool_call_object` — extracts `(name, args)` from a JSON object with flexible field names

## Message Sanitization (`sanitize_for_memory`)

Strips known envelope prefixes from user messages before persisting to memory:

- `[Group message from ...]`
- `[In risposta a: "..."]` / `[Replying to: "..."]`
- `[Stranger from ...]` / `[Stranger]`
- `[Forwarded]`
- `[User]`

Only standalone line-leading envelopes are stripped; inline brackets (e.g., `[meet at 5pm]`) are preserved. Input that collapses to empty after stripping returns `None`.

Invariant: every envelope prefix stripped by the sanitizer is also detected by `is_cascade_leak`, so legacy memory rows don't keep tripping the leak guard.

## History Folding (`maybe_fold_stale_tool_results`)

Compresses old tool-result messages into compact `[history-fold]` stubs to reduce token usage. Configuration via `ToolResultsConfig`:

- `history_fold_after_turns` — turns older than this are candidates for folding
- `fold_min_batch_size` — minimum stale messages before a fold runs

The fold operates on both the working message list (sent to the LLM) **and** `session.messages` (the durable record). `messages_generation` is bumped via `mark_messages_mutated()` so `save_session_async` detects the change. On subsequent turns, the `is_already_folded` short-circuit prevents redundant re-folding.

A `FoldSummaryDriver` (aux LLM) generates deterministic summaries during tests; in production, the aux client chain handles summarization.

## Reply Directives

The loop parses structural directives from LLM output:

- `[[reply:msg_ID]]` → sets `directives.reply_to`
- `[[@current]]` → sets `directives.current_thread`
- These are stripped from the visible response text

Directives survive through MaxTokens continuations and empty-response fallbacks.

## Session Persistence

On MaxTokens overflow, the session is saved with the partial conversation. Session persistence respects ephemeral flags:

| Option | Persists? |
|--------|-----------|
| Default | Yes |
| `incognito: true` | No |
| `is_fork: true` | No |

## Test Infrastructure

### Mock Drivers

All integration tests use mock `LlmDriver` implementations:

| Driver | Behavior |
|--------|----------|
| `NormalDriver` | Returns fixed text with `EndTurn` |
| `EmptyAfterToolUseDriver` | First call: tool use; second call: empty `EndTurn` |
| `EmptyMaxTokensDriver` | Always returns empty `MaxTokens` |
| `FailThenTextDriver` | First call: tool use (fails); second: text recovery |
| `AlwaysFailingToolDriver` | Always emits tool call for nonexistent tool |
| `DirectiveDriver` | Returns configurable text + stop reason |
| `NotifyOwnerThenMaxTokensDriver` | Tool call then `MaxTokens` with `actual_provider` |
| `CascadeLeakTimedOutDriver` | Streams text then returns `LlmError::TimedOut` |
| `MultiToolCycleDriver` | N tool-use rounds then `EndTurn`; records all requests |
| `FoldSummaryDriver` | Deterministic fold summaries for aux client |
| `EmptyThenNormalDriver` | Empty first call, normal retry |
| `AlwaysEmptyDriver` | Always empty `EndTurn` |
| `TextToolCallDriver` | Emits tool calls as text (simulates Groq/Llama) |

### Helper Functions

- `test_manifest()` — minimal `AgentManifest` for testing
- `fresh_session()` — empty in-memory session
- `fake_tool(name)` — minimal `ToolDefinition`
- `session_texts()` — extracts text content from session messages
- `assert_saved_max_tokens_session()` — validates persisted session state
- `run_streaming_for_test()` — thin wrapper around `run_agent_loop_streaming`

### Grep Guard

The `silent_response_single_source_of_truth` test runs `grep -rln NO_REPLY` across the crate and asserts the literal only appears in an explicit allow-list. This prevents the sentinel token from leaking into new files without deliberate approval.

## Key Constants

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_ITERATIONS` | From `AutonomousConfig` | Loop iteration cap |
| `MAX_CONTINUATIONS` | Hardcoded | Max consecutive `MaxTokens` rounds |
| `MAX_RETRIES` | 3 | LLM call retry attempts |
| `BASE_RETRY_DELAY_MS` | 1000 | Exponential backoff base |
| `DEFAULT_MAX_HISTORY_MESSAGES` | 60 | History message limit |
| `ACCUMULATED_TEXT_MAX_BYTES` | Hardcoded | Text buffer size cap |
| `LAZY_TOOLS_THRESHOLD` | Hardcoded | Pool size triggering lazy resolution |

## Dependencies

- `librefang-types` — message types, tool definitions, agent config, directives
- `librefang-memory` — `Session`, `MemorySubstrate`, `SessionStore`
- `crate::llm_driver` — `LlmDriver` trait, `CompletionRequest`, `CompletionResponse`, `LlmError`
- `crate::silent_response` — cascade leak markers, envelope prefixes
- `crate::aux_client` — `AuxClient` for history fold summarization
- `crate::reply_directives` — `DirectiveSet` parsing