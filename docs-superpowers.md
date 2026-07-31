# docs — superpowers

# `docs/superpowers` Module

Planning and design documentation for substantial feature work. This module holds two artifact types—**plans** and **specs**—that together define how features are designed, scoped, and implemented before code is written.

---

## Module Structure

```
docs/superpowers/
├── plans/      # Task-by-task implementation roadmaps
└── specs/      # Architecture and design decisions
```

Each document is dated (`YYYY-MM-DD-<slug>.md`) and keyed to a feature branch. The date links the plan to its corresponding spec and vice versa.

---

## Document Types

### Specs (`specs/`)

A spec is the architectural source of truth for a feature. It captures decisions before implementation begins. A typical spec contains:

- **Summary** — one-paragraph description of the feature
- **Goals / Non-Goals** — scoping boundaries
- **Decisions table** — key design choices with rationale (e.g., where logic lives, blocking vs. non-blocking, discovery vs. config)
- **Architecture** — dependency relationships, new/modified files
- **Data structures** — Rust trait definitions, config types, enums
- **Flow diagrams** — initial connect, auth flow, token refresh (often as ASCII art)
- **API contracts** — endpoint signatures, request/response shapes
- **Testing strategy** — unit, integration, manual
- **Open questions** — unresolved items needing verification during implementation

### Plans (`plans/`)

A plan is the executable breakdown of a spec into discrete tasks. Plans are designed to be consumed by agentic workers (subagents or executing-plans workflows) and follow a strict format:

- **Header directive** — tells workers which skill to use for implementation
- **Goal & Architecture summary** — brief recap linking back to the spec
- **File Map** — tables of new and modified files with their responsibilities
- **Tasks** — numbered, each containing:
  - Files affected (with line numbers where applicable)
  - Checkbox-tracked steps (`- [ ]`)
  - Inline code blocks showing exact additions or changes
  - Verification commands (`cargo build`, `cargo test`, `cargo clippy`)
  - Git commit messages

Tasks build on each other sequentially. A task isn't considered done until its verification command passes.

---

## Working With These Documents

### For Implementers

1. Read the spec first to understand *why* decisions were made
2. Follow the plan task-by-task, checking off steps as you go
3. Run the verification command after each task before committing
4. Use the commit message at the end of each task verbatim

Plans reference exact line numbers in existing code. If line numbers have drifted due to other changes, locate the referenced symbol or struct by name instead.

### Delta Tasks

Plans may include a **DELTA task** at the end that supersedes earlier tasks. This happens when implementation reveals that a design assumption was wrong. A delta task explicitly states what it replaces and why.

For example, in the MCP OAuth Discovery plan, **Task 12 (DELTA)** changes the OAuth flow from daemon-initiated (localhost callback listener) to UI-initiated (callback routed through the API server). It supersedes the auth flow logic from Tasks 5–7 because the original design's ephemeral localhost port was unreachable in Docker deployments.

When a delta exists, read it carefully before implementing the tasks it modifies.

---

## Key Conventions

### Plan Header Format

Every plan starts with this directive block:

```markdown
> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task.
> Steps use checkbox (`- [ ]`) syntax for tracking.
```

This signals that the plan is structured for automated execution, not just human reading.

### Commit Messages

Plans specify exact `git commit` commands at the end of each task. These follow conventional commit format scoped to the crate:

```
feat(runtime): add mcp_oauth module with WWW-Authenticate parser and core types
feat(kernel): implement KernelOAuthProvider with vault storage and PKCE flow
feat(api): add MCP OAuth auth endpoints and auth state in server list
```

### Verification Gates

Every task ends with at least one of:

- `cargo build --workspace --lib` — compiles
- `cargo test --lib -p <crate>` — unit tests pass
- `cargo test --workspace` — full suite passes
- `cargo clippy --workspace --all-targets -- -D warnings` — no warnings
- `npm run build` — dashboard builds

The final task in a plan is typically a full verification pass across build, test, and clippy.

---

## Example: MCP OAuth Discovery

The current artifacts in this module (`2026-04-12-mcp-oauth-discovery`) illustrate the full pattern:

| Artifact | Content |
|----------|---------|
| **Spec** | Trait-based injection design for OAuth on MCP Streamable HTTP connections. Decides runtime defines `McpOAuthProvider` trait, kernel implements it. Three-tier metadata discovery (WWW-Authenticate → .well-known → config.toml). UI-driven auth flow with callback through API port 4545. |
| **Plan** | 12 tasks (11 + 1 delta) spanning 4 crates: `librefang-types`, `librefang-runtime`, `librefang-kernel`, `librefang-api`, plus the React dashboard. Starts with config types, builds up through the runtime trait and parser, kernel implementation, API endpoints, and ends with integration tests and full verification. |

The plan demonstrates how a design spec translates into concrete, ordered, verifiable implementation steps.

---

## Cross-Crate Patterns Visible in These Docs

The specs and plans in this module document several architectural patterns used across the codebase:

- **Trait injection** — Runtime crates define traits (`McpOAuthProvider`); kernel crates provide implementations. This avoids circular dependencies while keeping logic testable in isolation.
- **KernelHandle pattern** — Runtime defines interfaces, kernel wires them to concrete infrastructure (vault, HTTP, extensions).
- **Three-tier discovery** — Prefer protocol-native discovery (`WWW-Authenticate`, `.well-known`), fall back to explicit config. Config overrides discovery where both exist.
- **Non-blocking daemon startup** — Long-running operations (browser auth) don't block boot. The daemon marks state and continues; completion is asynchronous.
- **Vault key namespacing** — Secrets stored with prefixed keys (`mcp_oauth:{url}:access_token`) to avoid collisions across features.