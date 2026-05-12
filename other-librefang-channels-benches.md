# Other — librefang-channels-benches

# librefang-channels Benchmarks — Dispatch Hot Paths

## Overview

The `dispatch.rs` benchmark suite measures the performance of three critical hot paths in the `librefang-channels` crate using [Criterion](https://bheisler.github.io/criterion.rs/):

| Category | What it measures | Benches affected |
|---|---|---|
| **Serialization** | JSON round-trip cost of `ChannelMessage` | 3 benchmarks |
| **Routing** | Agent resolution latency across dispatch strategies | 4 benchmarks |
| **Formatting** | Markdown-to-channel format conversion, message splitting, emoji lookup | 8 benchmarks |

These benchmarks cover the per-message overhead incurred on every incoming and outgoing channel event. Regression in any of these paths directly impacts message throughput.

## Running

```bash
# All benchmarks
cargo bench -p librefang-channels

# Single group
cargo bench -p librefang-channels -- serialization
cargo bench -p librefang-channels -- routing
cargo bench -p librefang-channels -- formatting

# Single benchmark by name
cargo bench -p librefang-channels -- router_resolve_direct
```

Criterion groups are registered via `criterion_main!` at the bottom of the file:

```
criterion_main!(serialization, routing, formatting);
```

## Benchmark Groups

### Serialization (`serialization` group)

Measures `serde_json` overhead on `ChannelMessage`, a struct containing strings, an enum (`ChannelContent`), an optional `AgentId`, a `DateTime<Utc>`, and a `HashMap<String, String>` for metadata.

| Benchmark | Operation |
|---|---|
| `message_serialize` | `serde_json::to_string(&msg)` |
| `message_deserialize` | `serde_json::from_str::<ChannelMessage>(&json)` |
| `message_roundtrip` | Serialize then deserialize in one iteration |

The sample message is constructed by `make_sample_message()` with a Telegram channel, a text content payload, and a populated sender. All three benches call this helper once in setup; only the iteration body is measured.

**What to watch for:** Adding fields to `ChannelMessage` or changing `serde` attributes (e.g., `#[serde(flatten)]`) can shift both serialize and deserialize timings.

### Routing (`routing` group)

Measures `AgentRouter::resolve` and `resolve_with_context` under four scenarios of increasing complexity:

#### `router_resolve_direct`

The fastest path. A direct route is registered via `set_direct_route("telegram", "user-42", agent)`, then `resolve` is called with a matching channel + peer. This hits a direct map lookup.

#### `router_resolve_default_fallback`

No bindings match. The router falls through to the default agent set via `set_default`. Measures the cost of a cache miss on direct routes followed by the default fallback.

#### `router_resolve_binding_match`

Bindings are loaded from an `AgentBinding` config (match on `channel` + `peer_id`). Tests the cost of evaluating match rules against incoming channel + user. The agent is first registered with `register_agent("support", agent)`, then `load_bindings` is called.

#### `router_resolve_with_context`

The richest resolution path. Uses `resolve_with_context` with a `BindingContext` that includes `guild_id` and `roles`. The binding rule matches on `channel`, `guild_id`, and `roles`. This exercises the full context-aware matching logic including role set intersection.

**What to watch for:** The number of loaded bindings and the complexity of `BindingMatchRule` fields (especially `roles` as a vec) scale the per-resolve cost. If binding count grows, this is where you'll see it first.

### Formatting (`formatting` group)

Exercises `format_for_channel` (from `librefang_channels::formatter`) across all `OutputFormat` variants, plus `split_message` and `default_phase_emoji`.

#### Format conversion benchmarks

Two input sizes are used:
- `SAMPLE_MARKDOWN` — a multi-paragraph string (~300 chars) with bold, italic, inline code, and links.
- `SHORT_TEXT` — `"Hello world!"` (12 chars).

| Benchmark | Input | OutputFormat |
|---|---|---|
| `format_markdown_passthrough` | `SAMPLE_MARKDOWN` | `Markdown` |
| `format_telegram_html` | `SAMPLE_MARKDOWN` | `TelegramHtml` |
| `format_slack_mrkdwn` | `SAMPLE_MARKDOWN` | `SlackMrkdwn` |
| `format_plain_text` | `SAMPLE_MARKDOWN` | `PlainText` |
| `format_telegram_html_short` | `SHORT_TEXT` | `TelegramHtml` |

The short-text benchmark isolates the per-call overhead (function call, format dispatch) from the per-character parsing cost.

#### Message splitting benchmarks

| Benchmark | Input | Chunk limit |
|---|---|---|
| `split_message_short` | `"Hello!"` | 4096 |
| `split_message_long` | 500 lines (~9 KB) | 4096 |

`split_message` is called with a 4096-character Telegram limit. The long variant exercises the line-boundary splitting algorithm under realistic multi-chunk conditions.

#### Emoji lookup benchmark

`default_phase_emoji_all` iterates over all six `AgentPhase` variants (`Queued`, `Thinking`, `tool_use("web_fetch")`, `Streaming`, `Done`, `Error`) in a single iteration. This is a micro-benchmark to catch regressions if emoji resolution becomes non-trivial (e.g., loading from a map instead of a match expression).

## Shared Test Fixtures

### `make_sample_message()`

Constructs a `ChannelMessage` with:
- `channel`: `ChannelType::Telegram`
- `platform_message_id`: `"msg-12345"`
- `sender`: `ChannelUser` with `platform_id: "user-42"`, `display_name: "Alice"`, no linked user
- `content`: `ChannelContent::Text("Hello, how can you help me today?")`
- `target_agent`: `None`
- `timestamp`: `Utc::now()`
- `is_group`: `false`
- `metadata`: empty `HashMap`

This is representative of a typical inbound message. If the `ChannelMessage` schema evolves, this fixture should be updated to match.

## External Dependencies

The benchmarks exercise code from these `librefang-channels` modules:

```
dispatch.rs
├── types.rs       → ChannelMessage, ChannelUser, ChannelContent, ChannelType,
│                    split_message, default_phase_emoji, AgentPhase
├── router.rs      → AgentRouter, BindingContext
│                    methods: new, set_default, set_direct_route,
│                    register_agent, load_bindings, resolve, resolve_with_context
├── formatter.rs   → format_for_channel
└── librefang_types::config → AgentBinding, BindingMatchRule, OutputFormat
    librefang_types::agent  → AgentId
```

## Interpreting Results

Criterion outputs median and mean iteration times along with a change estimate against the previous run. Key thresholds to care about:

- **Serialization**: Should remain under a few microseconds for a single message. A jump likely indicates a `serde` attribute change or a new heavy field type.
- **Routing**: `router_resolve_direct` should be near-instant (hash lookup). `router_resolve_with_context` is the ceiling — if it exceeds a few hundred nanoseconds, review the binding matching algorithm.
- **Formatting**: The `format_telegram_html` benchmark is the most complex parser path. Compare against `format_markdown_passthrough` to see how much overhead the actual format conversion adds beyond identity return.