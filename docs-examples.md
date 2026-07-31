# docs — examples

# Examples — Reference Sidecars

The `docs/examples` directory contains dependency-free Python 3 reference implementations of the two sidecar extension points LibreFang exposes: the **context engine** and the **proactive memory extractor**. Each script is a long-lived stdio process that speaks a newline-delimited JSON protocol with the daemon. They are intentionally minimal — their purpose is to demonstrate the wire format, the configuration wiring, and two well-known stdio pitfalls, not to provide production logic.

## Files

| File | Replaces | Config section |
|---|---|---|
| `context_engine_sidecar.py` | The built-in context windowing / recall engine | `[context_engine.sidecar]` |
| `memory_extractor_sidecar.py` | The built-in proactive memory extractor | `[proactive_memory.extractor_sidecar]` |

## The Wire Protocol

Both sidecars share the same framing:

```
daemon  ──>{"id": N, "method": "...", "params": {...}}\n ──>  sidecar
sidecar ──>{"id": N, "ok": {...}}\n                      ──>  daemon
         ──>{"id": N, "error": "<msg>"}\n                ──>  daemon
```

- **One JSON object per line**, terminated by `\n`.
- Every reply echoes the request's `id` so the daemon can match asynchronous responses.
- On success, the payload goes under the `ok` key. On failure, the human-readable message goes under `error`.
- The sidecar must never exit on a single bad request — both scripts wrap handler calls in `try/except` and emit an `error` reply instead of crashing the read loop.

### Two stdio pitfalls the examples guard against

Both docstrings call these out because they are easy to get wrong:

1. **Read with `sys.stdin.readline()`, not `for line in sys.stdin`.** The iterator form read-ahead-buffers and will block until EOF, meaning no line is ever delivered while the process is alive. `readline()` returns as soon as one newline is available.
2. **Call `sys.stdout.flush()` after every reply.** When stdout is a pipe (which it is here), it is block-buffered by default. Without an explicit flush, the daemon would never receive the reply.

Both scripts share the same `main()` loop shape:

```python
def main():
    while True:
        line = sys.stdin.readline()
        if not line:
            break               # EOF — daemon closed stdin
        line = line.strip()
        if not line:
            continue
        try:
            req = json.loads(line)
        except ValueError:
            continue            # skip malformed lines
        ...build reply...
        sys.stdout.write(json.dumps(reply) + "\n")
        sys.stdout.flush()
```

## context_engine_sidecar.py

Replaces the in-process context engine. The daemon dispatches one of four methods per turn:

| Method | When called | Reference behavior |
|---|---|---|
| `bootstrap` | Sidecar startup | No-op, returns `{}` |
| `ingest` | After a memory is written | Returns `{"recalled_memories": []}` — no recall in the reference |
| `assemble` | Before building the LLM request | Trims to the most recent `KEEP_RECENT` (40) messages |
| `after_turn` | After the assistant reply is committed | No-op, returns `{}` |

Handlers are registered in a dispatch table:

```python
HANDLERS = {
    "ingest": ingest,
    "assemble": assemble,
    "after_turn": after_turn,
    "bootstrap": bootstrap,
}
```

### Recovery signaling

`assemble` is the one handler that does real work. When it trims messages from the window it reports the count back to the daemon via the `recovery` field so the daemon knows history was dropped:

```python
recovery = "None" if removed == 0 else {"AutoCompaction": {"removed": removed}}
```

The reply shape is `{"messages": [...], "recovery": recovery}`. Returning `"None"` (the string) signals "nothing was removed"; the `AutoCompaction` object signals the daemon that compaction occurred.

### Configuration

```toml
[context_engine]
engine = "sidecar"

[context_engine.sidecar]
command = "python3"
args = ["/path/to/context_engine_sidecar.py"]
```

Set `engine = "sidecar"` to activate the external process; otherwise the built-in context engine is used.

## memory_extractor_sidecar.py

Replaces the built-in proactive memory extractor. The daemon sends:

```json
{"id": 7, "method": "extract_memories",
 "params": {"messages": [...], "categories": ["preference", ...]}}
```

The sidecar returns a list of memory shards in the simple shape the daemon expects:

```json
{"id": 7, "ok": {"memories": [{"content": "...", "category": "preference"}],
                  "has_content": true}}
```

The daemon is responsible for assigning `id` (UUID), `created_at`, and `source` — the sidecar only produces `{content, category?, level?, metadata?}`. Extracted memories should be restricted to the categories supplied in `params.categories`.

### The reference heuristic

The toy `extract()` function scans only the most recent user message for a fixed set of trigger phrases:

```python
TRIGGERS = ("i prefer ", "remember that ", "my name is ", "i like ", "i work ")
```

A real extractor would replace this body with a call to a model (local or remote), an embedding pipeline, or any other strategy. The loop inspects messages in reverse, stops at the first user turn, and returns `has_content: false` when nothing matched — which tells the daemon there is nothing to persist.

### Configuration

```toml
[proactive_memory.extractor_sidecar]
command = "python3"
args = ["/path/to/memory_extractor_sidecar.py"]
```

## Lifecycle and Data Flow

```mermaid
sequenceDiagram
    participant D as LibreFang daemon
    participant S as Sidecar (stdio)
    D->>S: spawn process, keep stdin open
    D->>S: {"id":1,"method":"bootstrap","params":{}}
    S-->>D: {"id":1,"ok":{}}
    Note over D,S: Per-turn cycle
    D->>S: {"id":2,"method":"assemble","params":{messages}}
    S-->>D: {"id":2,"ok":{messages,recovery}}
    D->>S: {"id":3,"method":"after_turn","params":{}}
    S-->>D: {"id":3,"ok":{}}
    D->>S: close stdin (EOF)
    S->>S: process exits
```

## Extending the Examples

The intended customization points are:

- **`context_engine_sidecar.py`** — replace `ingest` (to return `MemoryFragment` objects for system-prompt injection), `assemble` (to implement your own windowing/retrieval policy), and `after_turn` (to persist state to a vector store or database).
- **`memory_extractor_sidecar.py`** — replace `extract` with a model call, keeping the same return shape and respecting the `categories` allow-list.

Because the protocol is language-agnostic and dependency-free, you can rewrite either sidecar in any language; only the newline-delimited JSON framing matters.