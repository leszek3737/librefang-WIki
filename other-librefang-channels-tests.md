# Other — librefang-channels-tests

# librefang-channels — Bridge Integration Tests

## Overview

`bridge_integration_test.rs` provides end-to-end integration tests for the `BridgeManager` dispatch pipeline. Tests exercise the full message flow—**inbound message → router resolution → kernel dispatch → response delivery**—using in-process tokio channels and mock implementations of `ChannelAdapter` and `ChannelBridgeHandle`. No external services are contacted.

## Architecture

```mermaid
graph LR
    TX[Test injects message] --> Adapter[MockAdapter]
    Adapter -->|start() stream| BM[BridgeManager]
    BM -->|resolve agent| Router[AgentRouter]
    BM -->|send_message / streaming| Handle[MockHandle]
    Handle -->|response| BM
    BM -->|send / send_streaming| Adapter
    Adapter -->|captured in sent vec| Assert[Test assertions]
```

Every test follows the same skeleton:

1. Create a mock handle with known agents.
2. Create an `AgentRouter` (optionally pre-routing users).
3. Create a mock adapter wired to an `mpsc::Sender` for message injection.
4. Build a `BridgeManager`, call `start_adapter()`.
5. Inject messages via the sender.
6. Sleep briefly to let async dispatch settle.
7. Assert on captured responses in the adapter's `sent`/`streamed` vectors.
8. Call `manager.stop()`.

## Mock Implementations

### MockAdapter

A basic `ChannelAdapter` that does **not** support streaming.

| Method | Behavior |
|---|---|
| `start()` | Returns a `ReceiverStream` from an injectable `mpsc::Receiver` |
| `send()` | Captures `(platform_id, text)` pairs in a `Mutex<Vec>`; flattens `Interactive` button labels into text |
| `stop()` | Signals shutdown via `watch` channel |
| `supports_streaming()` | Returns `false` (default) |

Created via `MockAdapter::new(name, channel_type)` which returns `(Arc<Self>, mpsc::Sender<ChannelMessage>)`. Inject messages through the sender; inspect output via `get_sent()`.

### MockStreamingAdapter

Extends `MockAdapter` with `supports_streaming() → true` and a `send_streaming()` implementation that assembles all deltas into a single string, captured separately in `streamed`. Provides both `get_streamed()` and `get_sent()` for assertions.

### MockFailingStreamingAdapter

Reports `supports_streaming() → true` but `send_streaming()` **always returns `Err`**. Used to exercise fallback paths where the transport layer fails but the kernel may or may not have succeeded. Drains the delta channel so the bridge's tee task can populate `buffered_text`.

### MockHandle / MockStreamingHandle / MockProgressHandle / MockKernelErrorHandle / MockKernelOkHandle

All implement `ChannelBridgeHandle` with different response strategies:

| Handle | `send_message` | `send_message_streaming` | `send_message_streaming_with_sender_status` | Special |
|---|---|---|---|---|
| `MockHandle` | Returns `"Echo: {msg}"` | — | — | Records received messages |
| `MockStreamingHandle` | Returns `"Echo: {msg}"` | Emits word-by-word deltas | — | Records received messages |
| `MockProgressHandle` | Returns echo | — | Emits `🔧 tool_name` progress + prose | Mirrors real tool-use flow |
| `MockKernelErrorHandle` | Returns echo | — | Emits progress + partial text, then `Err("rate limit hit")` via status oneshot | Exercises dual-failure path |
| `MockKernelOkHandle` | Returns echo | — | Emits clean text, then `Ok(())` via status oneshot | Implements `record_delivery`; captures `(success, err)` tuples for metric assertions |

## Message Constructors

Two helper functions reduce boilerplate:

- **`make_text_msg(channel, user_id, text)`** — builds a `ChannelMessage` with `ChannelContent::Text`.
- **`make_command_msg(channel, user_id, cmd, args)`** — builds a `ChannelMessage` with `ChannelContent::Command`.

Both set `is_group: false`, empty metadata, no `thread_id`, and a fixed `platform_message_id` of `"msg1"`.

## Test Catalog

### Basic Dispatch

| Test | What it verifies |
|---|---|
| `test_bridge_dispatch_text_message` | A text message from a pre-routed user reaches the correct agent; the echo response is delivered back through the adapter. |
| `test_bridge_dispatch_no_agent_assigned` | An unrouted user receives a "No agents available" error message instead of a silent drop. |
| `test_bridge_dispatch_slash_command_in_text` | Plain text starting with `/` (e.g. `"/agents"`) is recognized and handled as a command even when sent as `ChannelContent::Text`. |

### Command Handling

| Test | What it verifies |
|---|---|
| `test_bridge_dispatch_agents_command` | `/agents` lists all running agents by name in the response. |
| `test_bridge_dispatch_help_command` | `/help` returns text mentioning `/agents` and `/agent`. |
| `test_bridge_dispatch_agent_select_command` | `/agent coder` selects that agent for the user; the router is updated and subsequent `resolve()` returns the correct `AgentId`. |
| `test_bridge_dispatch_status_command` | `/status` reports the number of running agents (e.g. `"2 agent(s) running"`). |

### Lifecycle and Multi-Adapter

| Test | What it verifies |
|---|---|
| `test_bridge_manager_lifecycle` | Start → send 5 messages → verify 5 echo responses in order → `stop()` completes without hanging. |
| `test_bridge_multiple_adapters` | Two adapters (Telegram + Discord) run simultaneously in the same `BridgeManager`; each receives only responses for its own channel. |

### Streaming

| Test | What it verifies |
|---|---|
| `test_bridge_streaming_adapter_uses_send_streaming` | When both the adapter and handle support streaming, `send_streaming` is called (not `send`); streamed text contains the echo. |
| `test_bridge_non_streaming_adapter_falls_back_to_send` | When the adapter doesn't support streaming (even though the handle does), the bridge falls back to `send()` with the full assembled response. |
| `test_default_send_streaming_collects_and_sends` | The default `send_streaming` implementation on `ChannelAdapter` collects all deltas and calls `self.send()` with the concatenated result. |

### Progress Markers (V2 Contract)

| Test | What it verifies |
|---|---|
| `test_bridge_non_streaming_adapter_sees_progress_markers` | A non-streaming adapter (Discord) receives progress markers (`🔧 web_search`) in the consolidated response via the `send_message_streaming_with_sender_status` pipeline. Progress is not Telegram-only. |

### Error Paths and Regression Guards

| Test | What it verifies |
|---|---|
| `test_bridge_streaming_adapter_kernel_and_transport_both_fail` | When both `send_streaming` (transport) and the kernel status report errors, the fallback `send()` delivers the partial buffered text including progress markers. |
| `test_bridge_streaming_adapter_kernel_ok_transport_fail_records_clean_success` | **Bug 1 regression guard.** When the kernel succeeds but `send_streaming` fails, the fallback delivers buffered text and `record_delivery` is called with `(success=true, err=None)`. Previously the transport error leaked into the `err` field, producing contradictory metrics (`success=true` + `err=Some(...)`). |

## Key Interactions with Production Code

- **`BridgeManager::new(handle, router)`** — production constructor; tests pass mock trait objects.
- **`BridgeManager::start_adapter(adapter)`** — spawns the dispatch loop for one adapter.
- **`BridgeManager::stop()`** — graceful shutdown of all adapter tasks.
- **`AgentRouter::set_user_default(user, agent_id)`** — pre-routes a user to an agent before message dispatch.
- **`AgentRouter::resolve(channel, user, target)`** — looked up internally by the bridge to determine which agent receives a message.
- **`ChannelBridgeHandle::send_message_streaming_with_sender_status`** — the V2 streaming-with-progress path; returns both a delta channel and a status oneshot.

## Timing Considerations

All tests use `tokio::time::sleep(100–300ms)` to allow the async dispatch pipeline to settle. This is a deliberate tradeoff: deterministic synchronization (e.g. oneshot channels per message) would require modifying production code, so the tests accept brief non-determinism in exchange for testing unmodified production paths. If tests flake, increasing sleep durations or adding retry loops is the recommended fix.

## Adding New Tests

To add a new integration test:

1. Create the appropriate mock handle (or reuse an existing one) with the agents you need.
2. Create an `AgentRouter` and optionally pre-route users.
3. Choose or create a mock adapter. For streaming scenarios, use `MockStreamingAdapter`; for transport failure scenarios, use `MockFailingStreamingAdapter`.
4. Instantiate `BridgeManager`, call `start_adapter()`, inject messages, sleep, assert, and call `stop()`.
5. If testing a new `ChannelBridgeHandle` method, create a new mock handle struct implementing the trait with the desired behavior.