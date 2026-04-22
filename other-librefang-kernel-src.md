# Other — librefang-kernel-src

# librefang-kernel-src

The core security and governance layer of LibreFang. This module contains two primary subsystems — **approval management** (gating dangerous tool executions behind human decisions) and **RBAC authentication** (mapping platform identities to roles and enforcing permissions) — along with kernel-level integration tests.

## Architecture Overview

```mermaid
graph TD
    subgraph "External callers"
        API["API routes<br/>(system.rs, terminal.rs)"]
        CB["Channel bridge<br/>(channel_bridge.rs)"]
        DASH["Dashboard API<br/>(librefang-api/server.rs)"]
    end

    subgraph "librefang-kernel-src"
        AM["ApprovalManager<br/>(approval.rs)"]
        AU["AuthManager<br/>(auth.rs)"]
    end

    subgraph "Storage"
        DASHDB[("SQLite audit DB")]
        INMEM[("In-memory<br/>DashMap + VecDeque")]
    end

    AM --> DASHDB : "audit_log_write<br/>persist_totp_lockout_*"
    AM --> INMEM : "pending / recent"
    API --> AM : "resolve, submit_request,<br/>list_pending, query_audit"
    CB --> AM : "verify_totp_code,<br/>verify_recovery_code"
    DASH --> AU : "from_str_role"
    DASH --> AM : "verify_totp_code_with_issuer"
    API --> AU : "authorize"
```

---

## ApprovalManager (`approval.rs`)

Gates dangerous tool invocations (shell execution, file writes, etc.) behind human-in-the-loop approval decisions. Supports both blocking and non-blocking (deferred) execution paths, TOTP second-factor authentication, escalation chains, and persistent audit logging.

### Construction

```rust
// In-memory only (no audit DB)
let mgr = ApprovalManager::new(policy);

// With SQLite audit logging and TOTP lockout persistence
let mgr = ApprovalManager::new_with_db(policy, conn);
```

`new_with_db` loads persisted TOTP lockout state from the `totp_lockout` table, discarding entries whose lockout window has already expired so a daemon restart does not extend the original 5-minute window.

### Two Execution Paths

**Blocking path** — `request_approval(req)` — the calling agent coroutine suspends until a human resolves the request, it times out, or the per-agent pending limit is hit. Internally uses a `tokio::sync::oneshot` channel to deliver the decision back to the waiting future.

**Deferred (non-blocking) path** — `submit_request(req, deferred)` — returns the request UUID immediately. The `DeferredToolExecution` payload is stored alongside the pending request and returned atomically on `resolve()`. The kernel's periodic sweep (`expire_pending_requests`) handles timeouts for deferred requests.

Both paths enforce a per-agent limit of `MAX_PENDING_PER_AGENT` (5) and reject overflow immediately.

### Policy Resolution: `requires_approval` vs `requires_approval_with_context`

`requires_approval(tool_name)` checks only the static `require_approval` list, supporting glob patterns (`file_*`, `*_exec`, `*`).

`requires_approval_with_context(tool_name, sender_id, channel)` applies a layered override chain:

1. **Trusted sender bypass** — if `sender_id` appears in `policy.trusted_senders`, returns `false` (no approval needed) regardless of tool or channel.
2. **Channel-specific rules** — if a `channel_rules` entry matches the channel, its `allowed_tools` / `denied_tools` list takes precedence. An explicit allow bypasses approval; an explicit deny forces it.
3. **Default list** — falls back to glob-matching against `require_approval`.

`is_tool_denied_with_context` performs the same checks but returns a boolean for the deny-only case (trusted senders bypass channel deny rules).

### Escalation and Timeout

The timeout behavior depends on `policy.timeout_fallback`:

| `TimeoutFallback` variant | Behavior on timeout |
|---|---|
| `TimedOut` (default) | Resolves as `ApprovalDecision::TimedOut` |
| `Skip` | Resolves as `ApprovalDecision::Skipped` |
| `Escalate { extra_timeout_secs }` | Bumps `escalation_count`, re-inserts with extended timeout. After `MAX_ESCALATIONS` (3) rounds, falls through to `TimedOut`. |

The effective timeout for escalation is `request.timeout_secs + (extra_timeout_secs × escalation_count)`, giving each round more time for the human to respond.

`expire_pending_requests` is called periodically by the kernel. It returns escalated requests (still pending) and expired requests (with their deferred payloads) separately so the kernel can re-notify for escalations and execute fallback decisions for expirations.

### Resolution: `resolve` and Batch Operations

`resolve(request_id, decision, decided_by, totp_verified, user_id)` is the core resolution method:

- Returns `(ApprovalResponse, Option<DeferredToolExecution>)` — the deferred payload is `Some` for the non-blocking path.
- If the request was already resolved, returns an error containing who resolved it (e.g., `"Already denied by admin"`).
- On success, records to the in-memory `recent` deque (capped at `MAX_RECENT_APPROVALS` = 100) and writes to the SQLite `approval_audit` table if configured.

Batch operations:

- **`resolve_batch`** — resolves multiple IDs with the same decision. Does not support TOTP.
- **`resolve_all_for_session`** — resolves every pending request matching a `session_id`. Mirrors Hermes-Agent's `resolve_gateway_approval(session_key, choice, resolve_all=True)`. TOTP-required requests are silently skipped (returned count excludes them).

### Session-Scoped Queries

| Method | Purpose |
|---|---|
| `list_pending_for_session(session_id)` | All pending `ApprovalRequest`s for a session |
| `has_pending_for_session(session_id)` | Quick boolean check for blocking state |
| `resolve_all_for_session(session_id, decision, decided_by)` | Atomically resolve all pending for a session |

These enable dashboard and channel UIs to scope approval views and actions to a single conversation.

### TOTP Second-Factor Authentication

When `policy.second_factor` is `SecondFactor::Totp`, approval of TOTP-gated tools requires a verified code.

**Per-tool gating** — `policy.totp_tools` optionally scopes TOTP to specific tools. When empty, all tools require TOTP. `policy.tool_requires_totp(tool_name)` encodes this logic.

**Grace period** — after a successful TOTP verification, the user enters a grace window (`totp_grace_period_secs`). Subsequent approvals by the same `user_id` skip TOTP until the window expires. Set to 0 to always require TOTP.

**Lockout** — after `TOTP_MAX_FAILURES` (5) consecutive failures, the user is locked out for `TOTP_LOCKOUT_SECS` (300 seconds). Lockout state is persisted to SQLite (`totp_lockout` table) so it survives daemon restarts.

**Resolution gate** — `resolve()` checks `totp_verified` when the decision is `Approved` and the tool requires TOTP. If the user is not within grace and `totp_verified` is false, resolution is rejected with a TOTP-required error. Denial never requires TOTP.

**Static TOTP utilities** (no state, can be called without an `ApprovalManager` instance):

| Method | Description |
|---|---|
| `verify_totp_code(secret, code)` | Verify a 6-digit code against a base32 secret (SHA-1, 30s step, ±1 window) |
| `verify_totp_code_with_issuer(secret, code, issuer)` | Same, with custom issuer label |
| `generate_totp_secret(issuer, account)` | Returns `(base32_secret, otpauth_uri, qr_base64_png)` |
| `generate_recovery_codes()` | 8 codes in `DDDD-DDDD` format |
| `verify_recovery_code(stored_json, code)` | Consumes a matching code, returns updated JSON |
| `is_recovery_code_format(code)` | Validates `DDDD-DDDD` format |

### Audit Logging

When constructed with `new_with_db`, every resolution is written to the `approval_audit` table via `push_recent → audit_log_write`. The `query_audit` and `audit_count` methods support paginated queries with optional `agent_id` and `tool_name` filters.

### Risk Classification

`classify_risk(tool_name)` is a static mapping:

| Tool | Risk Level |
|---|---|
| `shell_exec` | Critical |
| `file_write`, `file_delete`, `apply_patch` | High |
| `web_fetch`, `browser_navigate` | Medium |
| Everything else | Low |

### Policy Hot-Reload

`update_policy(policy)` replaces the active policy behind an `RwLock`. The new policy takes effect immediately for subsequent checks. `policy()` returns a clone of the current policy.

### Key Constants

| Constant | Value | Purpose |
|---|---|---|
| `MAX_PENDING_PER_AGENT` | 5 | Back-pressure limit per agent |
| `MAX_RECENT_APPROVALS` | 100 | In-memory history ring buffer |
| `MAX_ESCALATIONS` | 3 | Escalation rounds before forced timeout |
| `TOTP_MAX_FAILURES` | 5 | Consecutive failures before lockout |
| `TOTP_LOCKOUT_SECS` | 300 | Lockout duration in seconds |

---

## AuthManager (`auth.rs`)

Maps platform user identities (Telegram ID, Discord ID, etc.) to LibreFang users with hierarchical roles, then enforces RBAC permission checks.

### Role Hierarchy

```
Owner (3) > Admin (2) > User (1) > Viewer (0)
```

Roles implement `Ord` — any role at or above the required level is authorized.

### Channel Binding

Users are registered with `channel_bindings`: a map of `channel_type → platform_id`. During construction, `AuthManager` builds a reverse index (`"telegram:123456" → UserId`) so that incoming channel messages can be mapped to LibreFang identities via `identify(channel_type, platform_id)`.

A single user can bind to multiple channels (e.g., Telegram and Discord), and `identify` returns the same `UserId` for all of them.

### Authorization Model

Each `Action` declares a minimum `UserRole`:

| Action | Minimum Role |
|---|---|
| `ChatWithAgent` | User |
| `ViewConfig` | User |
| `SpawnAgent` | Admin |
| `KillAgent` | Admin |
| `InstallSkill` | Admin |
| `ViewUsage` | Admin |
| `ModifyConfig` | Owner |
| `ManageUsers` | Owner |

`authorize(user_id, action)` looks up the user's role and compares it against the action's requirement. Returns `Ok(())` on success or `LibreFangError::AuthDenied` with a descriptive message on failure.

### Integration Points

- **Terminal routes** (`src/routes/terminal.rs`) — `authorize_terminal_request` calls `AuthManager::authorize` before allowing WebSocket connections, window creation, and deletes.
- **Dashboard API** (`librefang-api/src/server.rs`) — `configured_user_api_keys` uses `UserRole::from_str_role` to map API key configurations to roles.
- **System routes** (`src/routes/system.rs`) — the approval router reads `policy()` from `ApprovalManager` before processing approval requests.

---

## Kernel Integration Tests (`kernel/tests.rs`)

Integration tests exercising the full kernel boot sequence with temporary directories. Key test scenarios:

- **API key rotation** — `collect_rotation_key_specs` deduplicates profiles sharing the primary key, skips profiles with missing environment variables, and prepends the distinct primary driver key.
- **Escalation notification routing** — `notify_escalated_approval` prefers per-request `route_to` targets over policy-level routing rules, agent notification rules, and global approval channels.
- **Agent registry** — name resolution, tag-based filtering, and manifest-to-capability mapping including profile-based tool expansion.
- **Default model override** — verifies that `spawn_agent_inner` applies local default model configuration to newly spawned agents.

The `RecordingChannelAdapter` test double captures sent messages for assertion without requiring a live channel connection.