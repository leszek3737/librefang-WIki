# Agent Runtime — librefang-runtime-src

# Agent Runtime — `librefang-runtime`

The agent runtime is the execution engine that drives every agent turn: receiving a user message, assembling context, calling the LLM, executing tool calls, persisting conversation history, and coordinating with external agents via the A2A protocol.

## Architecture Overview

```mermaid
graph TD
    subgraph "Agent Loop (agent_loop.rs)"
        A[User Message] --> B[Recall Memories]
        B --> C[Assemble Context]
        C --> D[LLM Completion]
        D --> E{Stop Reason?}
        E -->|EndTurn| F[Save Session]
        E -->|ToolUse| G[StagedToolUseTurn]
        G --> H[Execute Tools]
        H --> D
        E -->|MaxTokens| I[Continuation]
        I --> D
    end

    subgraph "Context Loading (agent_context.rs)"
        C -.->|per-turn read| J[context.md]
    end

    subgraph "A2A Protocol (a2a.rs)"
        K[A2aClient] -->|discover| L[/.well-known/agent.json]
        K -->|tasks/send| M[External Agent]
        N[A2aTaskStore] -->|in-memory| O[HashMap]
        N -->|persistent| P[SQLite]
    end

    G -.->|agent_send| K
```

---

## Module: `agent_loop` — Core Agent Execution Loop

The agent loop is the central orchestrator for every agent turn. It manages the full lifecycle: context assembly, LLM completion, tool execution, memory extraction, and session persistence.

### Entry Point

The primary entry point is `run_agent_loop_streaming`, which:

1. Resolves the message-history cap via `resolve_max_history`
2. Builds the context window (system prompt + memories + `context.md` + conversation history)
3. Enters the iteration loop bounded by `MAX_ITERATIONS`
4. Streams events back to the caller through an `mpsc::Sender<StreamEvent>`

### Iteration Loop

Each iteration follows this flow:

1. **Trim history** — `safe_trim_messages` caps the message list to the resolved `max_history`, cutting at conversation-turn boundaries so `ToolUse`/`ToolResult` pairs are never split. Pinned messages (e.g., delegation results from `agent_send`) are rescued and re-inserted after trimming.

2. **Build completion request** — The tool list is resolved through `ResolvedToolsCache`, which avoids re-cloning the full tool catalog on every iteration. When lazy tool loading is active (agent has >30 granted tools and `tool_load` is available), only native tools plus any session-loaded tools are shipped.

3. **Call the LLM** — `stream_with_retry` handles the actual HTTP call, gated by a process-global semaphore (`LLM_CONCURRENCY`, cap of 5 simultaneous calls) to bound memory spikes from concurrent request/response bodies.

4. **Process the response** based on stop reason:
   - **`EndTurn`** — The assistant's text response is final. Save session and return.
   - **`ToolUse`** — Enter the staged tool-use pipeline.
   - **`MaxTokens`** — Continue generation up to `MAX_CONTINUATIONS` (5) rounds.

### Staged Tool-Use Pipeline (`StagedToolUseTurn`)

The `StagedToolUseTurn` struct is the structural fix for orphaned `ToolUse` blocks that caused HTTP 400 errors from strict providers. Instead of eagerly pushing the assistant's `tool_use` message to `session.messages` before tools execute, the entire turn is staged in a local buffer:

- `stage_tool_use_turn` — Extracts tool calls from the LLM response into a `StagedToolUseTurn` without mutating any shared state.
- `append_result` — Per-tool execution adds its `ToolResult` block to the staged buffer.
- `pad_missing_results` — Any tool that didn't execute (mid-turn signal, hard error break) gets a synthetic "tool interrupted" result so every `tool_use_id` is paired.
- `commit` — Atomically pushes both the assistant message and the user `{tool_results}` message to `session.messages` and the LLM working copy.

If the staged turn is dropped without `commit` (via `?` propagation or an early break), `session.messages` is untouched. No orphan `ToolUse` can ever reach persistence.

### Tool Execution (`ToolExecutionContext`)

`ToolExecutionContext` bundles every dependency a tool invocation needs:

| Field | Purpose |
|-------|---------|
| `manifest` | Agent configuration (model, capabilities, sandbox settings) |
| `loop_guard` | Circuit-breaker state for runaway loops |
| `memory` | Memory substrate for recall/store operations |
| `session` | Mutable session state (conversation history, agent ID) |
| `kernel` | Optional kernel handle for agent-to-agent delegation |
| `available_tools` | Full `ToolDefinition` list for lazy-load resolution |
| `mcp_connections` | MCP server connections for remote tool calls |
| `web_ctx` | Web search/fetch context |
| `checkpoint_manager` | Checkpoint save/restore for long-running tasks |
| `context_budget` | Token budget tracker for context window management |
| `interrupt` | Per-session interrupt handle for `/stop` signals |

Tools execute with a per-call timeout of `TOOL_TIMEOUT_SECS` (600 seconds).

### Safety Guards

**Consecutive Hard Failure Tracking** — When every tool in an iteration returns a hard error (non-soft, non-parameter error), a counter increments. After `MAX_CONSECUTIVE_ALL_FAILED` (3) consecutive all-failed iterations, the loop exits with `RepeatedToolFailures`. Soft errors (sandbox rejections, approval denials, parameter errors) reset the counter.

**Loop Guard** — `LoopGuard` monitors iteration count, token spend, wall-clock time, and tool-call patterns. It applies configurable circuit-breaker thresholds and can force-exit the loop to prevent runaway agents.

**Context Overflow Recovery** — When the LLM returns a context-length error, `recover_from_overflow` applies a multi-stage strategy (trim history → compress → switch model) to salvage the turn.

**Accumulated Text Cap** — Intermediate text emitted alongside `tool_use` blocks is accumulated in a fallback buffer capped at `ACCUMULATED_TEXT_MAX_BYTES` (64 KiB). This prevents unbounded heap growth across many iterations in autonomous mode.

**Progress-Text Leak Detection** — `is_progress_text_leak` catches short ellipsis-terminated preambles (e.g., `"Waiting for the script to complete..."`) that the model emits before a tool call that never materializes. These are suppressed rather than delivered as the agent's final reply.

**Silent Response Detection** — `is_no_reply` delegates to `silent_response::is_silent_response` for case-insensitive detection of no-reply sentinels (`NO_REPLY`, `[no reply needed]`), preventing unnecessary messages on channels.

### Message History Management

```rust
pub const DEFAULT_MAX_HISTORY_MESSAGES: usize = 40;
const MIN_HISTORY_MESSAGES: usize = 4;
```

The history cap is resolved in priority order:

1. `manifest.max_history_messages` — per-agent override
2. `opts.max_history_messages` — kernel/operator config
3. `DEFAULT_MAX_HISTORY_MESSAGES` — compiled-in fallback (40)

Values below `MIN_HISTORY_MESSAGES` (4) are clamped up because the safe-trim heuristic requires at least one full tool round-trip to function correctly.

`safe_trim_messages` cuts at turn boundaries (never splitting `ToolUse`/`ToolResult` pairs), rescues pinned messages, re-validates via `session_repair::validate_and_repair`, and synthesizes a minimal user message if post-trim repair leaves fewer than 2 messages.

### Session Repair

`repair_session_before_save` is called on all non-normal exit paths (circuit breaker, max iterations, timeout). It runs `validate_and_repair_with_stats` to fix orphaned tool results, empty messages, misordered results, and duplicate entries — ensuring the persisted session is always well-formed and can be loaded without errors on the next turn.

### Lazy Tool Loading

Agents with more than `LAZY_TOOLS_THRESHOLD` (30) granted tools enter lazy mode. Instead of shipping the full tool catalog on every LLM call:

- Only native tools (always available) plus any tools explicitly loaded via `tool_load` this session are included.
- `tool_load(name)` and `tool_search(query)` remain available so the LLM can discover and pull in additional tools on demand.
- `ResolvedToolsCache` avoids re-cloning the tool list on every iteration, rebuilding only when the loaded-tool set grows.

### Image Data Stripping

`strip_processed_image_data` replaces base64 image blocks in historical messages with lightweight text placeholders after the LLM has processed them. Each image block consumes ~56K tokens; stripping prevents context window exhaustion in long conversations with images.

---

## Module: `agent_context` — Per-Turn Context Loading

Loads external `context.md` files that agents depend on for live data (market feeds, project state, etc.) that changes between turns.

### Resolution Path

```
{workspace}/.identity/context.md   ← preferred (new layout)
{workspace}/context.md             ← fallback (legacy)
```

`resolve_context_path` checks the `.identity` subdirectory first. The first candidate that exists on disk wins, even if it's empty or unreadable — so failures are reported against the canonical location rather than silently falling back to a stale legacy file.

### Caching Behavior

| `cache_context` | Behavior |
|-----------------|----------|
| `false` (default) | Fresh disk read every turn. On read failure, falls back to the last successfully cached content. |
| `true` | First successful read is cached and returned verbatim for the entire session. External updates are invisible. |

The in-memory cache (`OnceLock<Mutex<HashMap<PathBuf, String>>>`) is process-global and keyed by resolved path. It serves a dual purpose: freeze-on-first-read for `cache_context = true` agents, and graceful degradation for `cache_context = false` agents when the external writer is mid-rewrite.

### Security Constraints

- **Symlink rejection** — `read_capped` uses `symlink_metadata` and refuses to follow symlinks. This prevents a prompt-injection exfiltration vector where an attacker could point `context.md` at `/etc/passwd` and have its contents injected into the LLM prompt.
- **Size cap** — Files are read up to `MAX_CONTEXT_BYTES` (32 KiB). Oversized files are truncated to the last valid UTF-8 boundary using `str_utils::safe_truncate_str`.
- **UTF-8 validation** — Files containing no valid UTF-8 prefix are treated as I/O errors so the caller falls back to cached content rather than injecting garbage into the prompt.

### Integration Point

Called once per agent turn during context assembly in `agent_loop`, before the system prompt is built. The loaded content is injected as a "Live Context" section in the agent's prompt.

---

## Module: `a2a` — Agent-to-Agent Protocol

Implements Google's A2A protocol for cross-framework agent interoperability. LibreFang agents can both *expose* their capabilities to external systems and *consume* capabilities from external A2A agents.

### Agent Cards

`AgentCard` is a JSON capability manifest served at `/.well-known/agent.json` per the A2A specification. `build_agent_card` converts an `AgentManifest` into an `AgentCard`, mapping LibreFang tool names to A2A skill descriptors.

```rust
pub fn build_agent_card(manifest: &AgentManifest, base_url: &str) -> AgentCard
```

The card includes:
- Agent name, description, and endpoint URL
- Capabilities: streaming, push notifications, state transition history
- Skills: one per tool, with tags for discovery
- Input/output modes: `["text"]` by default

### A2A Tasks

`A2aTask` is the unit of work exchanged between agents. Task status follows this lifecycle:

```
Submitted → Working → Completed
                    → Failed
                    → Cancelled
                    → InputRequired → (caller provides input) → Working
```

`A2aTaskStatusWrapper` handles the two serialization forms found in the wild: bare string (`"completed"`) and object form (`{"state": "completed", "message": null}`). The `state()` method extracts the underlying status regardless of encoding.

Tasks carry:
- `messages` — conversation between agents (user/agent turns with `A2aPart` content: text, file, or structured data)
- `artifacts` — outputs produced by the task (with optional name, description, metadata, and streaming chunk markers)
- `agent_id` — the local agent that processed the task
- `caller_a2a_agent_id` — the external caller's identity (from `X-A2A-Agent-ID` header), stored for audit/ACL

### A2aTaskStore — Task Lifecycle Tracking

`A2aTaskStore` tracks in-flight and completed A2A tasks with bounded memory:

| Constructor | Behavior |
|-------------|----------|
| `new(max_tasks)` | In-memory only, default 24-hour TTL |
| `with_ttl(max_tasks, ttl)` | In-memory only, custom TTL |
| `with_persistence(max_tasks, db_path)` | SQLite-backed with automatic pruning and reload |

**Eviction policy** (applied lazily on each `insert`):

1. **TTL sweep** — All tasks whose `updated_at` exceeds the TTL are removed regardless of state. This prevents `Working`/`InputRequired` tasks from accumulating indefinitely.
2. **Capacity eviction** — If still at capacity after the TTL sweep, evict the oldest terminal-state task (`Completed`/`Failed`/`Cancelled`). If no terminal tasks exist, evict the oldest task overall.

**Persistence** (`with_persistence`):

- Schema: `a2a_tasks_v2` table stores the full `messages` and `artifacts` arrays as JSON, plus `session_id`, `agent_id`, and `caller_a2a_agent_id`.
- On startup: prunes rows older than 7 days, then loads the most recent `max_tasks` rows into memory. Older rows remain queryable through `get()`'s SQLite fallback.
- `get()` fast-paths the in-memory map; on miss, queries the SQLite store directly so evicted tasks remain retrievable.
- Persistence is best-effort: SQLite write failures log a warning but don't block the caller. A full disk degrades silently to in-memory-only.

### A2aClient — External Agent Interaction

`A2aClient` discovers and communicates with external A2A agents via JSON-RPC:

| Method | JSON-RPC | Description |
|--------|----------|-------------|
| `discover(url)` | `GET /.well-known/agent.json` | Fetch and parse an agent's capability card |
| `send_task(url, message, session_id)` | `tasks/send` | Submit a task to an external agent |
| `get_task(url, task_id)` | `tasks/get` | Poll a task's status from an external agent |

### Security Architecture

The A2A client implements multiple defense layers against SSRF, DNS rebinding, and resource-exhaustion attacks:

**SSRF Protection** — Every outbound request calls `build_client_for_url`, which:
1. Runs `web_fetch::check_ssrf` to resolve DNS, validate IP addresses, and obtain a `SsrfResolution`.
2. Pins DNS to the validated addresses via `ClientBuilder::resolve`, closing the DNS-rebinding TOCTOU window.
3. Rejects private IPs and cloud metadata ranges unconditionally (even when allowlisted).

**Redirect Blocking** — `Policy::none()` prevents all redirect following. A custom policy that re-validates each hop is insufficient because DNS for the redirect target would be re-resolved by reqwest's connector, and the pinned-DNS protection only covers the original hostname.

**Body Size Caps** — `read_capped_body` streams the response in chunks rather than using `reqwest::Response::json()` or `bytes()`, which are unbounded:
- Agent Cards: `MAX_AGENT_CARD_BYTES` = 256 KiB
- Task RPC responses: `MAX_A2A_TASK_BYTES` = 1 MiB
- Checks `Content-Length` upfront and aborts mid-stream if the running total exceeds the cap.
- Transport-layer decompression is disabled (`no_gzip`, `no_brotli`, `no_deflate`) so `Content-Length` reflects the actual wire bytes and decompression bombs can't bypass the upfront check.

### URL Canonicalization

`canonicalize_a2a_url` normalizes peer URLs for trust-list comparison:

- Strips fragments and empty query strings
- Lowercases scheme and host
- Drops default ports (80 for HTTP, 443 for HTTPS)
- Preserves trailing slash normalization

Both trust insertion (`approve`, static seeding via config) and the gate at `/api/a2a/send` must run input through the same canonicalizer to prevent cosmetic variations (trailing slash, default port, host casing) from bypassing or falsely denying legitimate calls.

### Discovery at Boot

`discover_external_agents` is called during kernel boot to fetch Agent Cards from all configured external agents. Results are keyed by canonicalized URL so the trust gate in `/api/a2a/send` can match on the same key callers pass.

---

## Cross-Module Interactions

```
agent_loop
  ├── agent_context::load_context_md()      — per-turn context.md loading
  ├── tool_runner::execute_single_tool_call() — tool dispatch
  │     └── a2a::A2aClient::send_task()     — agent_send delegates to external agents
  ├── session_repair                         — validate_and_repair on trim/exit
  ├── context_engine                         — memory recall + context assembly
  ├── loop_guard                             — circuit breaker thresholds
  ├── context_overflow                       — recovery from context-length errors
  └── checkpoint_manager                     — save/restore for long tasks
```

The `agent_loop` module is the integration point: it orchestrates context loading (`agent_context`), A2A communication (`a2a`), memory operations, tool execution, and session persistence into a single coherent execution loop.