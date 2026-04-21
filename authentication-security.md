# Authentication & Security

# Authentication & Security

LibreFang implements a **layered, defense-in-depth security model** spanning five crates — from shared type definitions through API gateway enforcement to runtime data protection.

## Sub-module Map

| Sub-module | Layer | Role |
|---|---|---|
| [librefang-types](librefang-types-src.md) | **Shared vocabulary** | Data structures and validation rules for capabilities, approval gates, taint tracking, and tool policies |
| [librefang-api](librefang-api-src.md) | **API gateway** | GCRA rate limiting, OAuth2/OIDC federation, Argon2id dashboard password hashing |
| [librefang-kernel](librefang-kernel-src.md) | **Core authorization** | RBAC identity, per-agent capability grants, human-in-the-loop approval for dangerous tools |
| [librefang-runtime](librefang-runtime-src.md) | **Runtime protection** | Tamper-evident audit logging, PII redaction/pseudonymisation, provider circuit breaker with auth-profile rotation |
| [librefang-runtime-oauth](librefang-runtime-oauth-src.md) | **Provider OAuth** | ChatGPT (browser + device flow) and GitHub Copilot device-flow authentication |

## How the Layers Fit Together

```
Request ──► Rate Limiter (api)
               │
               ▼
          API Key / OIDC Auth (api)
               │
               ▼
          RBAC + Capability Check (kernel ← types)
               │
               ▼
          Approval Gate? (kernel ← types)
               │
               ▼
          PII Filter on outbound (runtime ← types taint)
               │
               ▼
          Audit Record (runtime)
               │
               ▼
          Provider Call (runtime → runtime-oauth for token refresh)
```

**Types are the contract.** `librefang-types` defines capability sets, approval request/response shapes, taint markers, and tool-policy glob rules with built-in validation. Every other crate consumes these types so enforcement is consistent across the API boundary, kernel, and runtime.

**The API is the gatekeeper.** Every inbound request passes through the GCRA rate limiter, then API-key or OIDC middleware (`oauth.rs`), and finally Argon2id-based dashboard session derivation (`password_hash.rs`) before reaching kernel authorization.

**The kernel is the policy engine.** `AuthManager` performs RBAC lookups, `CapabilitySet` checks grant or deny specific agent actions, and the approval subsystem gates dangerous tool executions behind human confirmation — all using type definitions from `librefang-types`.

**The runtime is the safety net.** Before any message reaches an LLM provider, `PiiFilter` redacts or pseudonymises sensitive data. `ProviderCooldown` implements a circuit breaker that tracks auth failures per provider and rotates auth profiles when needed. `AuditLog` records every security-relevant event with a tamper-evident hash chain anchored to SQLite.

**Runtime-oauth bridges external providers.** When the circuit breaker triggers a profile rotation, or when a user first connects a provider, `librefang-runtime-oauth` handles the ChatGPT or Copilot OAuth exchange and returns fresh tokens.

## Key Cross-module Workflows

**Dashboard login → terminal action.** A dashboard password is verified with Argon2id (`api/password_hash.rs`), producing a session token. Subsequent terminal operations (rename, delete window) pass through `valid_api_tokens` → `dashboard_session_token` → `authorize_terminal_request` in the kernel, then are audit-logged by the runtime.

**Dangerous tool execution.** The kernel's approval subsystem (defined in `types/approval.rs`, enforced in `kernel/approval.rs`) pauses execution and surfaces a confirmation request. Approval policies use glob-pattern tool rules with deny-wins semantics. Only after human response does execution resume — and the decision is recorded in the audit log.

**Outbound data protection.** Taint tracking types (`types/taint.rs`) classify data provenance. The runtime's `PiiFilter` applies these rules before LLM calls, blocking shell injection patterns, credit card numbers, phone numbers, and authorization-shaped JSON in outbound payloads. Simultaneously, `check_outbound_text_violation` prevents MCP tool calls from leaking sensitive context.

**Provider auth resilience.** The runtime's circuit breaker (`auth_cooldown.rs`) independently tracks each provider. On repeated auth failures, it opens the circuit; a probe interval allows periodic retry. Successful probes close the circuit. Profile rotation triggers a fresh OAuth flow via `librefang-runtime-oauth`, which handles the token exchange and redirect handling for both ChatGPT and Copilot.

---

Consult each sub-module's page for implementation details, configuration options, and internal architecture diagrams.