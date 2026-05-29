# Other — librefang-channels-tests

# librefang-channels-tests

Integration and conformance tests for the `librefang-channels` crate. Two test suites live here:

| File | Purpose |
|------|---------|
| `bridge_integration_test.rs` | End-to-end dispatch pipeline tests exercising `BridgeManager`, `AgentRouter`, and the streaming/approval listener paths |
| `sidecar_protocol_conformance.rs` | Protocol conformance tests asserting that Rust serialization/deserialization matches the shared corpus in `conformance/sidecar/corpus/` |

All tests are fully in-process — no external services, network calls, or files outside the corpus directory are touched.

---

## Test Infrastructure

### `wait_until` — Deadline-bounded condition polling

```rust
async fn wait_until<F>(label: &str, mut cond: F)
```

Replaces fixed `tokio::time::sleep` waits with a 2-second deadline loop that polls every 5 ms. Tests fail fast on regression (stuck dispatch panics at the deadline) rather than flaking on slow CI runners. Every async dispatch test uses this helper.

### Mock Adapters

Three mock `ChannelAdapter` implementations provide injectable message sources and captured output sinks:

| Adapter | Streaming | `send_streaming` | Use case |
|---------|-----------|-------------------|----------|
| `MockAdapter` | No | Uses default (collects → `send`) | Basic dispatch, slash commands, lifecycle |
| `MockStreamingAdapter` | Yes (`supports_streaming() → true`) | Assembles deltas into full text, records in `streamed` | Verifying the streaming path is chosen |
| `MockFailingStreamingAdapter` | Yes | Drains all deltas then returns `Err` | Exercising fallback when transport fails mid-stream |

All adapters share the same construction pattern:

```rust
let (adapter, tx) = MockAdapter::new("name", ChannelType::Telegram);
// `tx` is an mpsc::Sender<ChannelMessage> — inject test messages
// adapter.get_sent() returns Vec<(platform_id, text)> pairs
```

### `NotifyingAdapter` — Approval listener testing

A specialized adapter for approval listener tests that overrides `notification_recipients()` and optionally `account_id()`. Constructed via:

- `NotifyingAdapter::new(name, recipients)` — bare single-bot adapter
- `NotifyingAdapter::with_account(name, account_id, recipients)` — account-qualified multi-bot adapter
- `NotifyingAdapter::with_channel_and_account(name, channel_type, account_id, recipients)` — non-Telegram multi-bot adapter

### Mock Kernel Handles

Four `ChannelBridgeHandle` implementations simulate different kernel behaviors:

| Handle | Key behavior |
|--------|-------------|
| `MockHandle` | Echoes messages via `send_message`, serves agent lists |
| `MockStreamingHandle` | Splits echo response into per-word streaming deltas via `send_message_streaming` |
| `MockProgressHandle` | Emits `🔧 tool_name` progress markers via `send_message_streaming_with_sender_status` |
| `MockKernelOkHandle` | Returns clean streaming success + records `record_delivery` calls for metric contract assertions |
| `MockKernelErrorHandle` | Emits partial deltas then reports kernel error via status oneshot |
| `EventBusHandle` | Exposes a real `tokio::broadcast` channel for injecting `Event` instances |

### Message constructors

```rust
fn make_text_msg(channel: ChannelType, user_id: &str, text: &str) -> ChannelMessage;
fn make_command_msg(channel: ChannelType, user_id: &str, cmd: &str, args: Vec<&str>) -> ChannelMessage;
```

Both create fully populated `ChannelMessage` instances with a `"msg1"` platform message ID and current UTC timestamp.

---

## Bridge Integration Tests

### Dispatch pipeline

```
                    ┌──────────────┐
  tx.send(msg) ──►  │ MockAdapter  │──start()──► Stream<ChannelMessage>
                    └──────┬───────┘
                           │ BridgeManager dispatch loop
                    ┌──────▼───────┐
                    │  AgentRouter │──resolve()──► AgentId
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ MockHandle   │──send_message()──► "Echo: ..."
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
                    │ MockAdapter  │──send()──► captured in sent Vec
                    └──────────────┘
```

**Tests covering basic dispatch:**

| Test | What it verifies |
|------|-----------------|
| `test_bridge_dispatch_text_message` | Text message → routed agent → echo response arrives at adapter |
| `test_bridge_dispatch_agents_command` | `/agents` command lists registered agents (via `ChannelContent::Command`) |
| `test_bridge_dispatch_help_command` | `/help` returns text mentioning `/agents` and `/agent` |
| `test_bridge_dispatch_agent_select_command` | `/agent coder` selects agent, updates router, subsequent `resolve()` confirms binding |
| `test_bridge_dispatch_no_agent_assigned` | Unrouted message returns "No agents available" |
| `test_bridge_dispatch_slash_command_in_text` | `"/agents"` sent as plain text (not `Command` variant) still triggers command handling |
| `test_bridge_dispatch_status_command` | `/status` reports "N agent(s) running" |
| `test_bridge_manager_lifecycle` | Start → 5 sequential messages → all echoed in order → clean stop |
| `test_bridge_multiple_adapters` | Two adapters (Telegram + Discord) dispatch independently through the same `BridgeManager` |

### Streaming dispatch

The bridge chooses between `send()` and `send_streaming()` based on adapter capability and handle support. Three outcomes are tested:

| Test | Streaming path | Expected behavior |
|------|---------------|-------------------|
| `test_bridge_streaming_adapter_uses_send_streaming` | Both support streaming | `send_streaming` called, `send` NOT called |
| `test_bridge_non_streaming_adapter_falls_back_to_send` | Handle streams, adapter doesn't | Falls back to regular `send()` |
| `test_default_send_streaming_collects_and_sends` | Default `send_streaming` impl on non-overriding adapter | Collects all deltas into one `send()` call |

### Progress markers (V2 pipeline)

`test_bridge_non_streaming_adapter_sees_progress_markers` verifies that non-streaming adapters (Discord/Slack/Matrix) receive `🔧 tool_name` progress markers in their consolidated response via the `send_message_streaming_with_sender_status` → `send_response` pipeline.

### Error and fallback paths

| Test | Failure mode | Expected behavior |
|------|-------------|-------------------|
| `test_bridge_streaming_adapter_kernel_and_transport_both_fail` | `send_streaming` returns Err AND kernel reports error via status oneshot | Fallback `send()` delivers buffered partial text with progress markers preserved |
| `test_bridge_streaming_adapter_kernel_ok_transport_fail_records_clean_success` | `send_streaming` returns Err but kernel succeeds | `record_delivery(success=true, err=None)` — transport error must not leak into the metrics `err` field |

---

## Approval Listener Tests

The approval listener (`BridgeManager::start_approval_listener`) subscribes to the kernel's event bus and delivers `ApprovalRequested` events to adapter notification recipients. A series of PRs harden this path against cross-agent leaks:

### Delivery scoping evolution

```
#4875  Approval listener existed but was dead code — no callers
#4985  Fixed broadcast leak: every approval went to every adapter regardless of agent
#4994  Fixed account-qualified key fallback: multi-bot adapters leaked via bare-key fallback
#5002  Fixed binding-only adapters: adapters with no channel_default but with AgentBindings were silently dropped
```

### Scoping mechanism

The listener resolves the target adapter(s) for an `ApprovalRequested` event using this logic:

1. **Channel default path**: If `router.channel_default(<channel_key>)` matches the requesting agent → deliver to that adapter's recipients
2. **Binding fallback path** (#5002): If no channel default matches → query `router.bound_recipients_for_agent(channel_type, account_id, agent_id)` → deliver to each binding's `peer_id`
3. **Drop path**: If neither path resolves → silently skip (do not broadcast)

The channel key is constructed as:
- `<channel_type>` when `account_id()` is `None` (single-bot)
- `<channel_type>:<account_id>` when `account_id()` returns `Some(...)` (multi-bot) — no fallback to bare key is allowed

### Test coverage

| Test | PR | Scenario |
|------|-----|----------|
| `test_approval_listener_delivers_to_configured_recipients` | #4875 | Single adapter with recipients → notification arrives |
| `test_approval_listener_skips_adapter_without_recipients` | #4875 | Empty recipient list → no crash, no `send()` calls |
| `test_approval_listener_scopes_delivery_to_requesting_agent_adapter` | #4985 | Two account-qualified adapters (bot-a, bot-b) → approval for agent A only reaches bot-a |
| `test_approval_listener_skips_unbound_adapter` | #4985 | Adapter with no router binding → notification dropped |
| `test_approval_listener_drops_malformed_agent_id` | #4994 | Non-UUID agent_id on event → dropped, not broadcast |
| `test_approval_listener_does_not_fall_back_from_qualified_to_bare_key` | #4994 | Mixed single-bot + multi-bot config → multi-bot does not receive via bare-key fallback |
| `test_approval_listener_scopes_to_non_telegram_multibot_adapter` | #4994 | Discord adapters with account IDs → scoping is channel-type-agnostic |
| `test_approval_listener_falls_back_to_agent_binding_when_default_unset` | #5002 | No `channel_default`, but `AgentBinding` maps chat → approval delivered |
| `test_approval_listener_binding_fallback_does_not_leak_cross_agent` | #5002 | Binding for agent X, approval for agent Y → nothing delivered |
| `test_approval_listener_fans_out_to_all_bound_chats` | #5002 | Agent bound to two chats → both receive notification |
| `test_approval_listener_skips_binding_with_no_peer_id` | #5002 | Catch-all binding with no `peer_id` → skipped |
| `test_approval_listener_binding_respects_account_id_scope` | #5002 | Binding scoped to bot-a → bot-b does not receive |

---

## Sidecar Protocol Conformance Tests

Located in `sidecar_protocol_conformance.rs`. These tests assert structural JSON equality between the Rust `SidecarCommand`/`SidecarEvent` types and the shared corpus files under `conformance/sidecar/corpus/`.

### Directionality

- **Events** (corpus `events/` directory): Rust is the *consumer* — tests deserialize every corpus event file and verify it maps to the expected `SidecarEvent` variant by matching the `method` field.
- **Commands** (corpus `commands/` directory): Rust is the *producer* — tests construct each `SidecarCommand` variant, serialize it, and assert the output equals the corpus JSON value.

### Tests

| Test | What it verifies |
|------|-----------------|
| `corpus_files_are_well_formed` | Every `.json` under `events/` and `commands/` is a JSON object with a string `method` field |
| `events_deserialize_into_expected_variant` | Every corpus event deserializes into the `SidecarEvent` variant matching its `method` string |
| `ready_full_and_minimal_both_parse` | `ready_full.json` parses with full capabilities/account_id/protocol_version; `ready_minimal.json` parses with empty capabilities and no protocol_version (backward compat) |
| `commands_serialize_to_corpus` | Every `SidecarCommand` variant serializes to structurally equal JSON as the corresponding corpus file; also asserts the list of tested commands exactly matches the corpus file list (no orphan fixtures) |

### Adding new protocol messages

1. Add a corpus fixture to `conformance/sidecar/corpus/events/` or `commands/`
2. For events: the deserialization test will pick it up automatically — just ensure it parses into the correct variant
3. For commands: add a case to the `cases` vector in `commands_serialize_to_corpus` — the test asserts the vector covers every corpus file, so a missing entry will fail