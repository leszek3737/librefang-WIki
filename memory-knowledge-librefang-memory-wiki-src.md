# Memory & Knowledge — librefang-memory-wiki-src

# librefang-memory-wiki

Durable markdown knowledge vault for the LibreFang Agent OS.

This crate provides a file-backed knowledge base where each topic is a markdown page with structured YAML provenance metadata. Pages live on disk as plain `.md` files, making them simultaneously accessible to agents (via the `WikiVault` API) and to humans (via Obsidian, VS Code, or any text editor).

The vault is a companion to `librefang-memory` (the SQLite + vector substrate). Where the memory substrate handles "find me the K nearest snippets," the wiki handles "give me a navigable, auditable knowledge base I can also open in Obsidian."

## Architecture

```mermaid
graph TD
    A[wiki_write caller] -->|"\"[[topic]]\" placeholders"| B[WikiVault::write]
    B --> C{Drift detected?}
    C -->|Yes, force=false| D[HandEditConflict error]
    C -->|Yes, force=true| E[Preserve external body, append provenance]
    C -->|No| F[Rewrite links via RenderMode]
    E --> G[frontmatter::render]
    F --> G
    G --> H[atomic_write to disk]
    H --> I[Rebuild index.md + backlinks.json]
    H --> J[Update compile-state.json]
```

## On-Disk Layout

```
<vault_path>/
├── <topic>.md              # one page per topic
├── index.md                # auto-generated alphabetical index
└── _meta/
    ├── compile-state.json  # mtime + sha256 per page (last compiler run)
    └── backlinks.json      # { target → [source, ...] }
```

Each `.md` page has this structure:

```markdown
---
topic: project-conventions
created: 2026-05-06T10:30:00Z
updated: 2026-05-06T11:00:00Z
content_sha256: 6a4f...
provenance:
  - agent: agent_xyz
    session: sess_abc
    channel: cli
    turn: 4
    at: 2026-05-06T10:30:00Z
---

body markdown ...
```

The `content_sha256` field is the SHA-256 of the body (with trailing newline stripped). The compiler uses it to detect external edits between runs.

## Configuration

The vault is **off by default**. Operators opt in via `config.toml`:

```toml
[memory_wiki]
enabled = true
mode = "isolated"          # isolated | bridge | unsafe_local (only isolated works in v1)
vault_path = "~/.librefang/wiki/main"
render_mode = "native"     # native | obsidian
ingest_filter = "tagged"   # tagged | all (all has no effect in v1)
```

`vault_path` is optional. When unset, the vault root is derived from the kernel's `home_dir` (not from the `LIBREFANG_HOME` environment variable), preventing embedded profiles and tests from mixing data with a developer's personal vault.

## Key Types

| Type | Module | Purpose |
|---|---|---|
| `WikiVault` | `vault` | Main entry point — read, write, search, backlinks |
| `WikiPage` | `vault` | A loaded page: topic, frontmatter, body |
| `WikiWriteOutcome` | `vault` | Result of a write — path, hash, merge status |
| `SearchHit` | `vault` | A search result — topic, snippet, score |
| `BacklinkEntry` | `vault` | A directed edge: source → target |
| `Frontmatter` | `frontmatter` | Typed YAML header with provenance |
| `ProvenanceEntry` | `frontmatter` | Audit record: agent, session, channel, turn, timestamp |
| `RenderMode` | `render` | Link flavor: `Native` or `Obsidian` |
| `WikiError` | `error` | All error variants for the crate |

## Core Operations

### Writing Pages — `WikiVault::write`

```rust
pub fn write(
    &self,
    topic: &str,
    body_with_placeholders: &str,
    provenance: ProvenanceEntry,
    force: bool,
) -> WikiResult<WikiWriteOutcome>
```

Callers always author bodies using the `[[topic]]` placeholder syntax for cross-references. The vault rewrites these into the active render flavor at flush time:

- **Native** → `[topic](topic.md)`
- **Obsidian** → `[[topic]]` (identity — already in the right form)

This means the same body text is portable across render modes without re-authoring.

**Provenance is monotonic** — each write appends to the existing provenance list. The vault never drops history.

**Hand-edit safety**: On every write, the vault compares the on-disk mtime and body SHA-256 against the last compiler state. If either has drifted (a human edited the file in Obsidian between runs), the write is rejected with `WikiError::HandEditConflict` unless `force = true`. When forced, the external body is preserved — only the provenance list is augmented and links are re-rendered.

The write path holds a `Mutex` lock for the duration, serializing concurrent writes to any topic within the same vault instance.

**Limits**: Body size is capped at 1 MiB (`MAX_BODY_BYTES`). Topic names must match `[a-zA-Z0-9_-]+`, be at most 100 characters, and cannot be `index` or start with `_`.

### Reading Pages — `WikiVault::get`

```rust
pub fn get(&self, topic: &str) -> WikiResult<WikiPage>
```

Reads the page from disk, parses the frontmatter, and returns the body as-is. CRLF line endings are normalized to LF on read, so files saved on Windows or via `git autocrlf` parse correctly.

If the YAML frontmatter is malformed (a hand edit broke it), `get` does **not** fail — it synthesizes a default `Frontmatter` and logs a warning. The body remains accessible, and the next successful `wiki_write` will re-render the page with a clean header.

### Searching — `WikiVault::search`

```rust
pub fn search(&self, query: &str, limit: usize) -> WikiResult<Vec<SearchHit>>
```

Naive case-insensitive substring search across all page bodies and topics. Scoring:

- Topic name contains the query → +10.0
- Body matches → `ln(1 + match_count)` (sub-linear to prevent long pages from drowning short topic matches)

Results are sorted by score descending, then topic name ascending. Snippets are extracted around the first body match with `…` truncation.

This is intentionally simple for v1. Vector/FTS5 ranking is tracked as a follow-up.

### Backlinks — `WikiVault::backlinks`

```rust
pub fn backlinks(&self) -> WikiResult<Vec<BacklinkEntry>>
```

Returns all cross-reference edges from the `_meta/backlinks.json` file. The backlink index is rebuilt on every write and recognizes links in both forms:

- `[[topic]]` — the Obsidian / authoring placeholder form
- `[topic](topic.md)` — the native render form (only when display text matches the target stem, to avoid false positives from generic markdown links)

This invariance means the backlink index survives render-mode flips.

## Render Modes

The `RenderMode` enum controls how `[[topic]]` placeholders are rendered to disk:

| Mode | Disk form | Use case |
|---|---|---|
| `Native` (default) | `[topic](topic.md)` | Standard markdown tooling, GitHub preview |
| `Obsidian` | `[[topic]]` | Obsidian / Logseq wiki-link syntax |

Both modes produce valid CommonMark in their respective ecosystems. A vault can be re-rendered in the other mode by changing the config and rewriting pages — no data is lost.

Key methods on `RenderMode`:

- `link(topic)` — render a single cross-reference
- `rewrite_links(body)` — substitute all `[[topic]]` placeholders in a body
- `extract_links(body)` — find all wiki-links in either form (used for backlink indexing)

## Frontmatter Handling

The `frontmatter` module provides four operations:

- **`split(raw)`** — separate a raw page into `(Option<yaml>, body)`. Tolerates pages without frontmatter — returns `(None, raw)` so hand-authored pages load cleanly.
- **`parse(yaml, topic)`** — deserialize YAML into a `Frontmatter`. Returns `WikiError::Frontmatter` on invalid YAML.
- **`render(frontmatter, body)`** — serialize a `Frontmatter` + body into the on-disk `---`-delimited format.
- **`Frontmatter::hash_body(body)`** — compute the canonical SHA-256 of the body (trailing newline stripped for normalization).

The render→split round-trip is tested to be exact: `split(render(fm, body))` recovers the original frontmatter and body.

## Compile State

The `_meta/compile-state.json` file stores a `PageState` per topic:

```json
{
  "pages": {
    "project-conventions": {
      "mtime_ns": "1746534600000000000",
      "sha256": "6a4f..."
    }
  }
}
```

The `mtime_ns` is the filesystem modification time in nanoseconds since UNIX epoch (stored as a string for JSON portability). The `sha256` is the body hash from the last successful compile. Both must match for a write to proceed without `force = true`. The dual check protects against filesystems with coarse mtime granularity (HFS+ has 1-second precision) — the SHA-256 catches edits that mtime alone would miss.

## Atomic Writes

All file mutations use `atomic_write`: write to a `.tmp.write` file in the same directory, then `fs::rename`. This prevents readers from seeing half-written pages. The rename is atomic on POSIX systems when source and destination are on the same filesystem.

## Error Handling

All errors are captured in `WikiError`:

| Variant | When |
|---|---|
| `Disabled` | `enabled = false` in config |
| `ModeNotImplemented` | `bridge` or `unsafe_local` mode requested |
| `InvalidTopic` | Bad topic name (empty, too long, reserved, bad chars) |
| `BodyTooLarge` | Body exceeds 1 MiB |
| `NotFound` | No `.md` file for the topic |
| `HandEditConflict` | External edit detected, `force = false` |
| `Frontmatter` | YAML parse or serialize failure |
| `Io` | Filesystem error (includes path context) |

The `WikiError::io(path, source)` helper constructor wraps `std::io::Error` with the path that caused the failure.

## Integration Points

**Consumed by**: The runtime's tool surface exposes three builtin tools (`wiki_get`, `wiki_search`, `wiki_write`) that delegate to `WikiVault` methods. The kernel constructs the vault via `WikiVault::new(&config, home_dir)` during startup if `enabled = true`.

**Re-exports from `lib.rs`**: The crate re-exports `WikiVault`, `WikiPage`, `WikiWriteOutcome`, `SearchHit`, `BacklinkEntry`, `Frontmatter`, `ProvenanceEntry`, `RenderMode`, `WikiError`, `WikiResult`, `MemoryWikiConfig`, `MemoryWikiMode`, `MemoryWikiIngestFilter`, and `MemoryWikiRenderMode` — downstream crates only need to depend on `librefang-memory-wiki`.

## v1 Scope and Limitations

**In scope**:
- `isolated` mode with its own vault directory
- Explicit `wiki_write` calls as the only ingestion path
- `native` and `obsidian` render modes
- Hand-edit safety with mtime + SHA-256 conflict detection

**Out of scope (tracked as follow-ups)**:
- `bridge` mode — reading shared artifacts from the memory substrate
- `unsafe_local` mode — same-machine escape hatch for an existing Obsidian vault
- Memory-event subscription (`ingest_filter = "all"`) — v1 ignores this setting and warns
- LLM-assisted topic extraction — v1 requires explicit topic tags
- `memory_search` cross-corpus integration — lives in `librefang-runtime`