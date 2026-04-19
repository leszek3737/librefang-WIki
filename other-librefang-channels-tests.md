# Other — librefang-channels-tests

# Bridge Integration Tests

## Overview

This module provides integration tests for the `BridgeManager` dispatch pipeline in `librefang-channels`. It exercises the **full message routing path** — from a channel adapter receiving a user message, through router resolution and kernel dispatch, back to the adapter sending a response — without contacting any external services.

All communication is in-process via real tokio channels and tasks.

## Architecture

Each test follows the same wiring pattern:

```
MockAdapter ──start()──▶ Stream<ChannelMessage>
                              │
                              ▼
                      BridgeManager (dispatch loop)
                              │
                    ┌─────────┼─────────┐
                    ▼                   ▼
             Command handling     AgentRouter → resolve agent
                                      │
                                      ▼
                             MockHandle / MockStreamingHandle
                              (sends to "agent")
                                      │
                                      ▼
                           response ──▶ adapter.send() / send_streaming()
```

The components under test are the **real** `BridgeManager` and `AgentRouter`. Only the leaf interfaces are mocked: `ChannelAdapter` (the messaging platform) and `ChannelBridgeHandle` (the kernel/agent runtime).

## Mock Components

### MockAdapter

Implements `ChannelAdapter` for non-streaming platforms.

| Capability | Behavior |
|---|---|
| **Message injection** | `new()` returns an `mpsc::Sender<ChannelMessage>`. Send messages into it to simulate incoming platform events. |
| **Response capture** | Every call to `send()` records `(platform_id, text)` in an internal `Vec`. Retrieve with `get_sent()`. |
| **Interactive content** | Button labels are flattened into the response text for easy assertion (e.g., `"Choose:\nYes, No"`). |
| **Streaming** | Does **not** override `supports_streaming()` or `send_streaming()` — relies on the default `ChannelAdapter` implementation which collects all deltas and calls `send()`. |

### MockStreamingAdapter

Implements `ChannelAdapter` for streaming-capable platforms. Overrides `supports_streaming() → true` and `send_streaming()`.

| Capability | Behavior |
|---|---|
| **Stream capture** | `send_streaming()` collects all deltas from the `mpsc::Receiver<String>` into a single string, stored per `(platform_id)`. Retrieve with `get_streamed()`. |
| **Non-streaming capture** | The `send()` path still records to a separate `sent` vector, retrievable with `get_sent()`. Tests can assert which path was taken. |

### MockHandle / MockStreamingHandle

Both implement `ChannelBridgeHandle`. They share the same agent lookup and listing behavior but differ in how they respond:

| Implementation | `send_message()` | `send_message_streaming()` |
|---|---|---|
| **MockHandle** | Returns `Ok("Echo: {message}")` | — (not implemented) |
| **MockStreamingHandle** | Returns `Ok("Echo: {message}")` | Returns an `mpsc::Receiver<String>` that emits the echo response as individual word-by-word deltas |

Both record all received messages in `received: Arc<Mutex<Vec<(AgentId, String)>>>` for assertion.

`spawn_agent_by_name()` is intentionally unimplemented (`Err("mock: spawn not implemented")`) — no test currently exercises agent spawning.

## Helper Functions

### `make_text_msg(channel, user_id, text) → ChannelMessage`

Constructs a plain text `ChannelMessage` with sensible defaults (timestamp = now, `is_group = false`, empty metadata, no `target_agent` or `thread_id`).

### `make_command_msg(channel, user_id, cmd, args) → ChannelMessage`

Constructs a `ChannelMessage` with `ChannelContent::Command { name, args }`. Same defaults as `make_text_msg`.

## Test Coverage

### Message Dispatch

| Test | What it verifies |
|---|---|
| `test_bridge_dispatch_text_message` | A text message from a user with a pre-routed default agent is forwarded to the correct agent, and the kernel's response is sent back to the adapter. |
| `test_bridge_dispatch_no_agent_assigned` | A text message from an unrouted user produces a "No agents available" response instead of being forwarded. |

### Slash Commands

All command tests send messages through the adapter's `mpsc::Sender`, simulating real platform input.

| Test | Command | Verifies |
|---|---|---|
| `test_bridge_dispatch_agents_command` | `/agents` | Response lists all registered agent names (e.g., "coder", "researcher"). |
| `test_bridge_dispatch_help_command` | `/help` | Response mentions `/agents` and `/agent`. |
| `test_bridge_dispatch_agent_select_command` | `/agent coder` | Confirmation message says "Now talking to agent: coder", and `AgentRouter::resolve()` subsequently returns the correct `AgentId` for that user. |
| `test_bridge_dispatch_status_command` | `/status` | Response contains "{N} agent(s) running". |
| `test_bridge_dispatch_slash_command_in_text` | `/agents` sent as plain `ChannelContent::Text` | The bridge parses slash commands from text content and handles them as commands rather than forwarding to an agent. |

### Lifecycle and Concurrency

| Test | What it verifies |
|---|---|
| `test_bridge_manager_lifecycle` | Start adapter → send 5 sequential messages → all 5 responses are received in order → `manager.stop()` completes cleanly. |
| `test_bridge_multiple_adapters` | Two adapters (Telegram + Discord) run concurrently on the same `BridgeManager`. Each receives responses only for messages sent through its own channel. |

### Streaming

| Test | What it verifies |
|---|---|
| `test_bridge_streaming_adapter_uses_send_streaming` | When both the adapter (`MockStreamingAdapter`) and handle (`MockStreamingHandle`) support streaming, `send_streaming()` is called on the adapter and `send()` is **not** called. |
| `test_bridge_non_streaming_adapter_falls_back_to_send` | When the adapter does **not** support streaming but the handle does, the bridge falls back to calling `send()` with the full assembled response. |
| `test_default_send_streaming_collects_and_sends` | Directly calls the default `ChannelAdapter::send_streaming()` implementation on `MockAdapter`. Verifies it collects all string deltas from the channel and calls `send()` with the concatenated result (`"Hello world!"`). |

## Running

```bash
# All bridge integration tests
cargo test -p librefang-channels --test bridge_integration_test

# A single test
cargo test -p librefang-channels --test bridge_integration_test test_bridge_dispatch_text_message

# Streaming tests only
cargo test -p librefang-channels --test bridge_integration_test streaming
```

All tests use `#[tokio::test]` and include small `tokio::time::sleep` pauses (100–200ms) to allow the async dispatch loop to process messages. No external services or network calls are involved.