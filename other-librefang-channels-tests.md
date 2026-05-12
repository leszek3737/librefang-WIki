# Other — librefang-channels-tests

# librefang-channels Integration Tests

## Purpose

Integration tests that exercise the **full `BridgeManager` dispatch pipeline** end-to-end. Every test wires real production components (`BridgeManager`, `AgentRouter`) against mock implementations of `ChannelAdapter` and `ChannelBridgeHandle`, sending messages through tokio channels and asserting on the results. No external services are contacted.

## Architecture

```mermaid
graph LR
    T[Test injects message via mpsc::Sender] --> A[MockAdapter]
    A -->|start() returns Stream| BM[BridgeManager]
    BM -->|dispatch_message| H[MockHandle / MockStreamingHandle]
    H -->|echo response| BM
    BM -->|send() / send_streaming()| A
    A -->|captured in Arc<Mutex<Vec>>| assertions

    subgraph Production Components
        BM
        R[AgentRouter]
    end

    subgraph Test Doubles
        A
        H
    end

    BM --- R
```

## Key Utility: `wait_until`

```rust
async fn wait_until<F>(label: &str, mut cond: F)
where
    F: FnMut() -> bool,
```

Replaces fixed `tokio::time::sleep` with a **2-second deadline-bounded poll** (5ms tick interval). This gives the async dispatch pipeline exactly as much time as it needs while failing fast on regressions. The 2-second budget is generous for in-process communication but catches stuck dispatches quickly.

Use this in every test instead of arbitrary sleeps. The `label` parameter appears in the panic message on timeout, making flake diagnosis trivial.

## Mock Implementations

### MockAdapter

Basic `ChannelAdapter` that does **not** support streaming. Used for Discord/Slack/WebChat-style adapters.

| Method | Behavior |
|--------|----------|
| `start()` | Returns a `ReceiverStream` from an `mpsc::Receiver<ChannelMessage>` |
| `send()` | Captures `(platform_id, text)` pairs into `Arc<Mutex<Vec<...>>>`. Flattens `Interactive` button labels into text. |
| `stop()` | Sends `true` on a `watch` channel |
| `supports_streaming()` | Returns `false` (default) |

Created via `MockAdapter::new(name, channel_type)` which returns `(Arc<Self>, mpsc::Sender<ChannelMessage>)`. Inject test messages through the sender; read captured responses via `get_sent()`.

### MockStreamingAdapter

`ChannelAdapter` that **does** support streaming. Tracks both `send()` and `send_streaming()` calls separately so tests can assert which path was taken.

| Method | Behavior |
|--------|----------|
| `send()` | Captures into `self.sent` |
| `supports_streaming()` | Returns `true` |
| `send_streaming()` | Collects all deltas into `full_text`, stores in `self.streamed` |

Use `get_streamed()` to inspect streaming output and `get_sent()` to verify the non-streaming path was **not** invoked.

### MockFailingStreamingAdapter

Streaming-capable adapter whose `send_streaming()` **always returns `Err`**. Drains the delta stream first (so the bridge's internal `buffered_text` accumulates), then fails. Used to exercise fallback-to-`send()` error paths.

### MockHandle

Basic `ChannelBridgeHandle` that echoes messages and serves a static agent list.

| Method | Behavior |
|--------|----------|
| `send_message()` | Records `(agent_id, message)` in `received`, returns `"Echo: {message}"` |
| `find_agent_by_name()` | Looks up the static agent list |
| `list_agents()` | Returns the full agent list |
| `spawn_agent_by_name()` | Always returns `Err("mock: spawn not implemented")` |

### MockStreamingHandle

`ChannelBridgeHandle` that provides `send_message_streaming()`. Splits the echo response into **word-level deltas** emitted from a spawned task, returning an `mpsc::Receiver<String>`.

### MockProgressHandle

`ChannelBridgeHandle` implementing `send_message_streaming_with_sender_status()`. Synthesizes a progress marker line (`🔧 tool_name`) followed by prose text, then reports success via the status oneshot. Mirrors production `start_stream_text_bridge_with_status` behavior.

### MockKernelErrorHandle / MockKernelOkHandle

Handle variants that control the **status oneshot** outcome:

- `MockKernelErrorHandle` — emits progress + text deltas, then reports `Err("rate limit hit")` on the status channel. Exercises the `send_streaming Err + kernel Err` outcome.
- `MockKernelOkHandle` — emits clean text, reports `Ok(())` on status. Also implements `record_delivery()` to capture `(success, error)` metric pairs. Exercises the `send_streaming Err + kernel Ok` outcome (Bug 1 regression fix).

### EventBusHandle

`ChannelBridgeHandle` that exposes a real `tokio::sync::broadcast` channel via `subscribe_events()`. Used for approval listener tests. Returns `(handle, broadcast::Sender)` so tests can inject `Event` instances.

### NotifyingAdapter

`ChannelAdapter` that overrides `notification_recipients()` to return a configured operator list. Returns an immediately-closed `mpsc` stream from `start()` (no inbound messages needed). Captures outbound notifications in `sent`.

## Message Constructors

```rust
fn make_text_msg(channel: ChannelType, user_id: &str, text: &str) -> ChannelMessage
fn make_command_msg(channel: ChannelType, user_id: &str, cmd: &str, args: Vec<&str>) -> ChannelMessage
```

Both set `platform_message_id` to `"msg1"`, `display_name` to `"TestUser"`, `is_group` to `false`, `thread_id` to `None`, and empty metadata. `make_command_msg` wraps the content as `ChannelContent::Command`.

## Test Coverage Matrix

### Basic Dispatch

| Test | What it verifies |
|------|-----------------|
| `test_bridge_dispatch_text_message` | Text message routes to the correct agent via `AgentRouter`; response echoes back through the adapter |
| `test_bridge_dispatch_no_agent_assigned` | Unrouted user gets `"No agents available"` error message |
| `test_bridge_dispatch_slash_command_in_text` | Plain text `"/agents"` is parsed and handled as a command |
| `test_bridge_manager_lifecycle` | Start → 5 sequential messages → stop completes without hanging |
| `test_bridge_multiple_adapters` | Two adapters (Telegram + Discord) dispatch independently in the same `BridgeManager` |

### Command Handling

| Test | Command | Expected behavior |
|------|---------|-------------------|
| `test_bridge_dispatch_agents_command` | `/agents` | Lists all registered agent names |
| `test_bridge_dispatch_help_command` | `/help` | Returns help text mentioning `/agents` and `/agent` |
| `test_bridge_dispatch_agent_select_command` | `/agent coder` | Confirms selection; updates `AgentRouter` so subsequent `resolve()` returns the chosen agent |
| `test_bridge_dispatch_status_command` | `/status` | Returns `"N agent(s) running"` |

### Streaming Dispatch

| Test | What it verifies |
|------|-----------------|
| `test_bridge_streaming_adapter_uses_send_streaming` | Streaming-capable adapter receives deltas via `send_streaming`, not `send` |
| `test_bridge_non_streaming_adapter_falls_back_to_send` | Non-streaming adapter uses `send()` even when the handle supports streaming |
| `test_default_send_streaming_collects_and_sends` | Default `ChannelAdapter::send_streaming` impl collects all deltas and calls `send()` with assembled text |

### Progress Markers

| Test | What it verifies |
|------|-----------------|
| `test_bridge_non_streaming_adapter_sees_progress_markers` | Non-streaming adapter (Discord) receives progress markers (🔧) in the consolidated response — the V2 contract that progress is surfaced on every channel |

### Error Recovery

| Test | What it verifies |
|------|-----------------|
| `test_bridge_streaming_adapter_kernel_and_transport_both_fail` | Both `send_streaming` and kernel status report errors; fallback `send()` delivers partial buffered text with progress markers preserved |
| `test_bridge_streaming_adapter_kernel_ok_transport_fail_records_clean_success` | **Bug 1 regression test**: kernel succeeds but transport fails → `record_delivery` is called with `(success=true, err=None)`. Success=true + err=Some is a contradictory metric that must not occur. |

### Approval Listener

| Test | What it verifies |
|------|-----------------|
| `test_approval_listener_delivers_to_configured_recipients` | `ApprovalRequested` event on the kernel's event bus reaches all configured notification recipients with formatted text (includes approval ID prefix, tool name, `/approve`/`/reject` hints) |
| `test_approval_listener_skips_adapter_without_recipients` | Adapter with no notification recipients produces no `send()` calls — no crash, no spurious delivery |

## Adding a New Test

1. **Create your mock(s)** using the existing patterns. Most tests need `MockHandle::new(agents)` and `MockAdapter::new(name, channel_type)`.
2. **Set up routing** — call `router.set_user_default(user_id, agent_id)` if the test expects a routed agent.
3. **Wire the bridge**:
   ```rust
   let mut manager = BridgeManager::new(handle, router);
   manager.start_adapter(adapter.clone()).await.unwrap();
   ```
4. **Inject messages** through the `mpsc::Sender` returned by the mock's constructor.
5. **Assert asynchronously** using `wait_until("descriptive label", || condition)`.
6. **Clean up** with `manager.stop().await`.

### When you need a new mock

- For **adapter behavior variants**, implement `ChannelAdapter` on a new struct. Override `supports_streaming()` and `send_streaming()` only if testing streaming paths.
- For **kernel behavior variants**, implement `ChannelBridgeHandle`. Override `send_message_streaming_with_sender_status()` for progress/error path testing, or `subscribe_events()` for event-driven features.
- Use `Arc<Mutex<Vec<...>>>` for capturing side effects — this is `Send + Sync + 'static` safe for spawned tasks.

## Key Production Interfaces Exercised

| Interface | Production module | Role in tests |
|-----------|------------------|---------------|
| `ChannelAdapter` | `librefang_channels::types` | Mocked; provides inbound stream and outbound delivery |
| `ChannelBridgeHandle` | `librefang_channels::bridge` | Mocked; acts as the kernel/agent backend |
| `BridgeManager` | `librefang_channels::bridge` | Real; orchestrates dispatch |
| `AgentRouter` | `librefang_channels::router` | Real; maps users to agents |
| `ChannelMessage`, `ChannelContent`, `ChannelUser`, `ChannelType` | `librefang_channels::types` | Constructed directly |