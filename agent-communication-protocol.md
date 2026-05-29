# Agent Communication Protocol

# Agent Client Protocol (ACP) Adapter

Bridges LibreFang's agent runtime to the [Agent Client Protocol](https://agentclientprotocol.com/), a JSON-RPC 2.0 protocol over a duplex byte stream (typically stdio). Editors that implement ACP—Zed, VS Code, JetBrains—can embed a LibreFang agent natively, providing their own approval modals, file references, prompt streaming, and terminal hosting instead of relying on the LibreFang dashboard.

The crate lives at `librefang-acp/` and is gated behind the `kernel-adapter` feature for the concrete kernel integration.

## Architecture

```mermaid
graph TD
    Editor["Editor (Zed, VS Code, ...node[")"]"]
    Server["server.rs<br/>JSON-RPC handler chain"]
    Sessions["session.rs<br/>ACP ↔ LibreFang ID map"]
    Prompt["prompt.rs<br/>Event pump"]
    Events["events.rs<br/>StreamEvent → SessionUpdate"]
    Perm["permission.rs<br/>Approval bridge"]
    FS["fs.rs<br/>fs/* reverse-RPC"]
    Term["terminal.rs<br/>terminal/* reverse-RPC"]
    Kernel["AcpKernel trait"]
    Adapter["KernelAdapter<br/>(kernel-adapter feature)"]
    LFK["LibreFangKernel"]

    Editor <-->|JSON-RPC / stdio| Server
    Server --> Sessions
    Server --> Prompt
    Server --> Perm
    Prompt --> Events
    Server --> FS
    Server --> Term
    Prompt --> Kernel
    Perm --> Kernel
    FS --> Kernel
    Term --> Kernel
    Kernel -.->|impl| Adapter
    Adapter --> LFK
```

Data flows in both directions:

- **Agent → Editor**: The prompt pump (`prompt.rs`) receives `StreamEvent`s from the kernel, translates them via `events.rs` into ACP `SessionUpdate` notifications, and streams them to the editor.
- **Editor → Agent**: The permission bridge (`permission.rs`) receives kernel `ApprovalEvent`s, sends `session/request_permission` to the editor, and feeds the user's decision back. The `fs/*` and `terminal/*` modules let the runtime route file and command operations through the editor instead of the local filesystem.

## Entry Points

### `run(kernel, agent_id)`

Runs the ACP server on stdio. Used by the `librefang acp` CLI subcommand for in-process execution. Returns when stdin closes or the transport errors.

### `run_with_transport(kernel, agent_id, transport)`

Same logic with an explicit transport. Used by `librefang-api` for daemon-attached UDS connections and by integration tests with `tokio::io::duplex` pipes. On completion—clean or otherwise—drains all remaining sessions and unregisters their kernel-side handles.

## The `AcpKernel` Trait

Defined in `lib.rs`, this is the minimal kernel surface the ACP adapter requires. Extracted as a trait so:

1. The ACP server is testable in isolation—integration tests in `tests/acp_integration.rs` use stub implementations that return canned `StreamEvent` sequences.
2. The crate doesn't depend directly on `librefang-kernel`, avoiding the heavy transitive dependency tree.

Key methods:

| Method | Purpose |
|--------|---------|
| `resolve_agent` | Map a name or UUID string to an `AgentId`. Called once at startup. |
| `send_prompt` | Start a streaming prompt turn; returns an `mpsc::Receiver<StreamEvent>`. |
| `subscribe_approvals` | Subscribe to the kernel's approval broadcast channel. |
| `resolve_approval` | Feed an editor user's permission decision back to the kernel. Must route through `KernelHandle` (not `ApprovalManager` directly) so deferred tool execution fires. |
| `remember_decision` | Persist an "always" choice for the `(agent_id, tool_name)` pair. |
| `set_fs_client` / `set_terminal_client` | Store the reverse-RPC handle after `initialize`. |
| `register_session_fs` / `register_session_terminal` | Bind a handle to a LibreFang session so runtime tools can dispatch through the editor. |
| `fetch_session_history` | Pull persisted messages for `session/load` rehydration. |

## Session Management

`session.rs` maintains a `DashMap` mapping ACP `SessionId` strings to `SessionState` structs containing:

- **`librefang_session_id`**: Derived deterministically from the ACP session id via UUID v5 (`ACP_SESSION_NS` namespace). Same ACP id always produces the same kernel-side session, so a reconnecting editor's `session/load` rejoins the existing persisted history.
- **`cwd`**: The working directory the editor declared at `session/new`.
- **`cancel`**: A `CancellationToken` triggered by `session/cancel` to interrupt the active prompt pump.

The store supports reverse lookup via `find_by_librefang_id`, used by the permission bridge to translate kernel approval events back to the correct ACP session. On connection teardown, `drain_active` collects all remaining sessions for cleanup.

## Event Translation

`events.rs` houses `EventTranslator`, a stateful translator instantiated once per prompt turn. It maintains an `in_flight_by_name` map (tool name → FIFO of `ToolCallId`s) to correlate `ToolExecutionResult` events back to the correct tool call when multiple same-named tools run concurrently.

### Mapping Table

| `StreamEvent` variant | ACP output |
|---|---|
| `TextDelta` | `SessionUpdate::AgentMessageChunk` |
| `ThinkingDelta` | `SessionUpdate::AgentThoughtChunk` |
| `OwnerNotice` | `SessionUpdate::AgentMessageChunk` (Phase 1; may get dedicated treatment later) |
| `ToolUseStart` | `SessionUpdate::ToolCall` (status `Pending`) |
| `ToolInputDelta` | Suppressed — too granular for the wire |
| `ToolUseEnd` | `SessionUpdate::ToolCallUpdate` with raw input, status `InProgress` |
| `ToolExecutionResult` | `SessionUpdate::ToolCallUpdate` with result content, status `Completed`/`Failed` |
| `ContentComplete` / `PhaseChange` | No wire output — consumed by the pump for `StopReason` |

### Parallel Tool Call Disambiguation

`ToolExecutionResult` carries only `name`, not the originating `tool_use_id`. When multiple same-named calls are in flight, the FIFO pop attributes the result to the oldest pending call—a best-effort guess. When ≥2 calls are pending, the wire payload is prepended with a disambiguation note so the editor user knows the attribution may be wrong. Emptied queues are reaped from the map to prevent unbounded growth.

### Tool Kind Inference

`infer_tool_kind` maps tool names to ACP `ToolKind` variants by prefix/pattern matching (e.g., `read*` → `Read`, `bash` → `Execute`, `*search*` → `Search`). Unknown tools default to `Other`.

## Prompt Handling

`prompt.rs` drives a single `session/prompt` turn:

1. Look up the ACP session, resolve the LibreFang `SessionId`.
2. Concatenate prompt content blocks via `concat_text_blocks`. Text blocks are joined; non-text blocks (image, audio, resource links) degrade to bracketed placeholders with a warning notification. True multimodal support is a separate follow-up.
3. Call `AcpKernel::send_prompt` to start the streaming agent turn.
4. Race the event channel against the session's cancel token in a `tokio::select!`. Each `StreamEvent` is translated and sent as a `session/update` notification.
5. On channel close or cancel, map the last `StopReason` and return a `PromptResponse`.

Stop reason mapping: `EndTurn` → `EndTurn`, `MaxTokens` → `MaxTokens`, `ToolUse`/`StopSequence` → `EndTurn`, `ContentFiltered` → `Refusal`.

## Permission Bridge

`permission.rs` runs as a background task, subscribing to kernel `ApprovalEvent::Created` broadcasts:

1. Filter by session id—skip approvals from non-ACP surfaces.
2. Map the LibreFang session id back to its ACP counterpart via `SessionStore::find_by_librefang_id`.
3. Build a `RequestPermissionRequest` with the tool call id (from the LLM-assigned `tool_use_id` or a synthetic `approval-{req_id}` fallback), title, and permission options.
4. Send to the editor, race the response against a 60-second timeout.
5. On response (or timeout), translate the outcome via `decision_from_outcome` and call `resolve_approval`.

### High-Risk Tool Suppression

`Allow always` is hidden for high-risk tools (`shell_exec`, `file_write`, `file_delete`, `apply_patch`, `skill_evolve_*`) because the in-memory approval cache keys only on `(agent_id, tool_name)` without binding to arguments. One-click blanket allow on `shell_exec` would grant permanent shell access regardless of command. `Deny always` is always available since it's safe.

## Reverse-RPC: Filesystem

`fs.rs` wraps the ACP `fs/read_text_file` and `fs/write_text_file` agent→client requests. The editor is the file source, not the agent's local filesystem.

`FsClientHandle` holds an `Arc` over the protocol connection and a snapshot of `FsCapabilities` (editor-declared at `initialize`). All requests carry a 60-second timeout (`FS_RPC_TIMEOUT`). The handle implements `AcpFsClient` from `librefang-kernel-handle` so the kernel can route runtime tool calls without depending on the ACP schema crate.

Since the kernel keys its registry by LibreFang `SessionId` rather than ACP `SessionId`, the bridge uses a dummy (empty) session id on the wire—editors echo it back without inspecting it.

## Reverse-RPC: Terminal

`terminal.rs` wraps the five-method ACP terminal state machine:

1. `terminal/create` — spin up a PTY in the editor. 60s timeout.
2. `terminal/wait_for_exit` — block until the command finishes. 10-minute timeout (`TERMINAL_WAIT_TIMEOUT`).
3. `terminal/output` — snapshot captured stdout/stderr. 60s timeout.
4. `terminal/kill` — kill the process, keep the terminal alive for subsequent output reads.
5. `terminal/release` — drop the terminal entirely. 60s timeout.

`TerminalClientHandle` implements `AcpTerminalClient` with a `run_command` convenience method that chains create → wait_for_exit → output → release, always releasing even on intermediate failure.

## Kernel Adapter

`kernel_adapter.rs` (behind `kernel-adapter` feature) provides the concrete `AcpKernel` implementation over `Arc<LibreFangKernel>`:

- Holds the kernel as both the concrete type (for `send_message_streaming_with_routing_and_session_override`) and as a `KernelHandle` trait object (for `resolve_tool_approval`, which must go through the trait to fire deferred tool execution).
- Stores `FsClientHandle` and `TerminalClientHandle` in `Arc<RwLock<Option<...>>>` slots, populated at `initialize`.
- `resolve_agent` accepts both UUID strings and human-readable names, resolved through the agent registry.
- `resolve_approval` routes through `KernelHandle::resolve_tool_approval` rather than calling `ApprovalManager::resolve` directly, ensuring deferred tool execution spawns correctly.
- `fetch_session_history` pulls persisted messages from the memory substrate, filtering out system prompts.

## Server Handler Chain

`server.rs` assembles the full JSON-RPC handler chain:

| Handler | Method | Behaviour |
|---------|--------|-----------|
| `initialize` | `initialize` | Sets fs/terminal handles, declares capabilities (no multimodal in Phase 1). |
| `session/new` | `session/new` | Mints a UUID v4 ACP session id, derives the LibreFang session, registers fs/terminal. |
| `session/load` | `session/load` | Creates-or-replaces the mapping, replays up to `MAX_REPLAY_TURNS` (50) recent history turns as `session/update` notifications. |
| `session/resume` | `session/resume` | Same as load in Phase 1, including history replay. |
| `session/list` | `session/list` | Returns active sessions, optionally filtered by cwd. |
| `session/close` | `session/close` | Removes the session, unregisters fs/terminal clients. |
| `session/prompt` | `session/prompt` | Delegates to `prompt::handle`. |
| `session/cancel` | `session/cancel` | Fires the session's cancel token. |
| Catch-all | * | Returns `method_not_found` (-32601) for unimplemented methods so editors silently skip optional features. |

A background task runs `permission::run_bridge` to forward approval events. On connection teardown (clean or abrupt), a drop guard drains all remaining sessions and unregisters their kernel-side handles to prevent stale references.

## Error Handling

`error.rs` defines `AcpError` with variants for unknown sessions, missing agents, kernel errors, transport errors, in-flight prompt conflicts, and generic internal failures. The `into_acp_error` method converts to the protocol crate's error type for JSON-RPC responses—`UnknownSession` and `AgentNotFound` map to `invalid_params`, everything else to `internal_error`.

`AcpResult<T>` is the standard result alias used throughout the crate.