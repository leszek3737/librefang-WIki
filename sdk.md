# sdk

# SDK

Multi-language client libraries for the LibreFang Agent OS. The SDK module provides typed access to the REST API and a framework for building channel adapters that bridge external messaging platforms to the agent runtime.

## What's Here

| Sub-module | Language | Auto-generated? | Focus |
|---|---|---|---|
| [Go SDK](sdk-go.md) | Go 1.21 | Yes — from `openapi.json` | REST client only |
| [JavaScript SDK](sdk-javascript.md) | JS/TS (Node ≥ 18) | Yes — from `openapi.json` | REST client only |
| [Python SDK](sdk-python.md) | Python (stdlib only) | REST client generated; sidecar hand-written | REST client + agent scripts + sidecar framework |
| [Rust SDK](sdk-rust.md) | Rust (async, tokio) | REST client manual; sidecar hand-written | REST client + sidecar framework + Telegram adapter |

## Two Layers, Shared Contracts

### REST API Clients

The Go, JavaScript, Python (`LibreFang`), and Rust (`librefang`) clients all expose the same resource surface — agents, skills, models, providers, channels, credentials, workflows, plugins, audit, and more — over the LibreFang REST API (default `http://localhost:4545`). Each client supports SSE streaming for real-time agent responses.

The Go and JavaScript clients are fully auto-generated from `openapi.json` by `scripts/codegen-sdks.py`. The Python and Rust REST clients cover the same endpoints with idiomatic, hand-maintained wrappers. Pick whichever matches your runtime; the wire contract is identical.

### Sidecar Framework

Channel adapters are external processes that translate between a messaging platform (Telegram, Bluesky, DingTalk, Feishu, Email, etc.) and the LibreFang daemon. The daemon launches each adapter as a subprocess and communicates over stdin/stdout using newline-delimited JSON.

Both [Python](sdk-python.md) and [Rust](sdk-rust.md) ship a sidecar runtime implementing this protocol:

| Message | Direction | Purpose |
|---|---|---|
| `Ready` | Adapter → Daemon | Announce capabilities and schema |
| `Event` | Adapter → Daemon | Inbound content (message, callback, poll answer) |
| `Send` | Daemon → Adapter | Outbound text, media, interactive content |
| `Command` | Daemon → Adapter | Typing indicators, reactions, streaming lifecycle |

The Python package serves as the reference implementation, with production adapters for Telegram, Bluesky, DingTalk, Feishu, and Email. The Rust `librefang-sidecar` crate provides a `SidecarAdapter` trait and a `librefang-sidecar-telegram` adapter that is explicitly feature-parity with the Python Telegram adapter — same wire shape, same emoji-reaction map, same access-control semantics.

## Cross-Language Parity

The Telegram adapter is the canonical example of cross-language equivalence: the Rust implementation mirrors `sdk/python/librefang/sidecar/adapters/telegram.py` field-for-field. When a new capability lands in the Python reference, the Rust adapter is expected to follow.

## Choosing a Client

| If you need... | Use |
|---|---|
| Quick REST calls from a serverless function | **JavaScript** or **Go** — zero-friction, generated |
| An agent script running inside a sandbox (no dependencies) | **Python** — runs on stdlib alone |
| A high-throughput or type-safe backend service | **Rust** — async, zero-cost |
| A new channel adapter | **Python** (reference adapters to copy) or **Rust** (trait-based framework) |

## Generation Pipeline

```
openapi.json
    │
    ▼
scripts/codegen-sdks.py
    │
    ├──► sdk/go/librefang.go          (overwrite on regen)
    └──► sdk/javascript/index.js      (overwrite on regen)
```

The Python REST client and all sidecar code are maintained manually. Never edit generated files — they will be overwritten on the next regeneration.