# Dashboard API Client

# Dashboard API Client

The Dashboard API Client is the data-access layer for the LibreFang dashboard frontend. It provides typed, authenticated HTTP functions for every backend endpoint and wraps them in React Query hooks so page components never call `fetch` directly.

## Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  Page Components                                                 │
│  (ChatPage, AgentsPage, WorkflowsPage, ...)                     │
└──────────┬───────────────────────────────┬───────────────────────┘
           │ queries/                      │ mutations/
           ▼                               ▼
┌─────────────────┐            ┌─────────────────────────────────┐
│ useQuery hooks   │            │ useMutation hooks               │
│ (read paths)     │            │ (write paths)                   │
└────────┬────────┘            └──────────────┬──────────────────┘
         │                                     │
         │      lib/http/client.ts             │
         │  (curated re-export whitelist)      │
         └──────────────┬──────────────────────┘
                        │
                        ▼
               ┌─────────────────┐
               │    api.ts        │
               │  (raw functions  │
               │   + interfaces)  │
               └────────┬────────┘
                        │
              ┌─────────┴──────────┐
              │  HTTP primitives   │
              │  get / post / put  │
              │  patch / del       │
              └─────────┬──────────┘
                        │
                  buildHeaders()
                  authHeader()
                        │
                        ▼
                   Backend REST API
```

## Module Structure

| File | Role |
|---|---|
| `api.ts` | All endpoint functions, TypeScript interfaces, and low-level HTTP primitives. This is the only file that calls `fetch`. |
| `lib/http/errors.ts` | `ApiError` class — thrown on non-2xx responses. |
| `lib/http/client.ts` | Explicit re-export whitelist. Query and mutation hooks import from here, never directly from `api.ts`. |
| `lib/mutations/*.ts` | React Query `useMutation` wrappers that call client functions and invalidate the relevant query caches on success. |

## Authentication

Authentication is token-based. The API key is stored in `localStorage` under the key `librefang-api-key`.

### Token lifecycle

- **`setApiKey(key)`** — persists the token and resets the 401 guard so future unauthorized responses can re-trigger the login dialog.
- **`clearApiKey()`** — removes the token from storage.
- **`getStoredApiKey()`** — reads the raw token string.
- **`hasApiKey()`** — returns `true` when a non-empty token exists.

### How tokens reach the backend

Every request flows through `buildHeaders()` → `authHeader()`. The resulting `Authorization: Bearer <token>` header is merged into every HTTP primitive (`get`, `post`, `put`, `patch`, `del`, `getText`).

`authHeader()` also injects an `Accept-Language` header sourced from `localStorage("i18nextLng")` with a fallback to `navigator.language`.

### 401 handling

When the backend returns a 401, `parseError()` fires a one-shot global callback (`_onUnauthorized`), clears the stored token, and redirects the user to the login screen. The `_unauthorizedFired` guard prevents infinite loops when multiple in-flight requests fail simultaneously.

Call `setOnUnauthorized(fn)` at app startup (typically in `App.tsx`) to register this callback.

### Session login

- **`dashboardLogin(username, password, totpCode?)`** — posts credentials to `/api/auth/dashboard-login`. On success the returned token is automatically persisted via `setApiKey`.
- **`dashboardLogout()`** — posts to `/api/auth/logout` (best-effort, network failures don't block), then clears the local token.
- **`verifyStoredAuth()`** — probes `/api/security` with the stored token to check if it is still accepted by the backend. Retries up to 3 times with 1-second backoff.

### Auth mode detection

`checkDashboardAuthMode()` queries `/api/auth/dashboard-check` and returns one of four modes: `"credentials"`, `"api_key"`, `"hybrid"`, or `"none"`.

## HTTP Primitives

Five generic helpers handle all communication. Each accepts a path, merges auth headers, and throws `ApiError` on failure.

| Function | Method | Timeout |
|---|---|---|
| `get<T>(path)` | GET | Browser default |
| `post<T>(path, body, timeout?)` | POST | 60s default, 300s for long-running ops |
| `put<T>(path, body)` | PUT | Browser default |
| `patch<T>(path, body)` | PATCH | Browser default |
| `del<T>(path)` | DELETE | Browser default |
| `getText(path)` | GET | Returns raw string instead of JSON |

`post` uses an `AbortController` to enforce the timeout. If the request aborts, a human-readable timeout error is thrown that notes the operation may still be running server-side.

Two timeout constants are defined:

- `DEFAULT_POST_TIMEOUT_MS` = 60 000 (standard writes)
- `LONG_RUNNING_TIMEOUT_MS` = 300 000 (agent messages, workflow runs, skill installs)

## WebSocket URLs

`buildAuthenticatedWebSocketUrl(path)` constructs a `ws://` or `wss://` URL with the current API key appended as a `?token=` query parameter. Used by streaming and real-time features.

## Error Handling

All non-2xx responses are parsed into an `ApiError` instance:

```typescript
class ApiError extends Error {
  readonly status: number;   // HTTP status code
  readonly code: string;     // error code string (e.g. "HTTP_401")
  readonly message: string;  // human-readable message
}
```

The parser prefers the `detail` field from the response body (human-readable), falling back to the `error` field (machine code). Consumers can narrow with `err instanceof ApiError` to access structured error information.

## The Client Re-export Layer

`lib/http/client.ts` is a deliberate abstraction barrier between `api.ts` and the React Query layer. It re-exports specific functions and types — nothing is glob-exported.

**Why this matters:**

- Renaming or removing a function in `api.ts` breaks at `client.ts` first, not in dozens of hook files.
- The whitelist is a reviewable surface — reviewers can see exactly what the data layer exposes.
- `ApiError` is re-exported alongside the functions so hooks can `import { ApiError } from "../http/client"` without reaching into the errors module directly.

The file is organized into three sections: **query functions** (reads), **mutation functions** (writes), and **type re-exports**.

## React Query Mutation Hooks

Each `lib/mutations/*.ts` file exports `useMutation` hooks that:

1. Call the corresponding `client.ts` function.
2. Invalidate the affected query caches on success using `queryClient.invalidateQueries()`.

### Cache invalidation patterns

Mutations use query key factories (imported from `lib/queries/keys`) to target precise cache slices:

```typescript
// Only invalidate one agent's detail
qc.invalidateQueries({ queryKey: agentKeys.detail(variables.agentId) });

// Invalidate the entire agents namespace
qc.invalidateQueries({ queryKey: agentKeys.all });

// Invalidate snapshot + agents when an agent is created
qc.invalidateQueries({ queryKey: agentKeys.all });
qc.invalidateQueries({ queryKey: overviewKeys.snapshot() });
```

Some mutations intentionally skip invalidation. For example, `useStopAgent` does not invalidate queries because stopping an agent's run does not change agent list state — the UI reconciles streaming state separately.

### Domain-specific mutation files

| File | Domain | Key Hooks |
|---|---|---|
| `agents.ts` | Agent CRUD, sessions, prompt versions, experiments | `useSpawnAgent`, `usePatchAgent`, `usePatchAgentConfig`, `useSwitchAgentSession`, `useCreatePromptVersion` |
| `analytics.ts` | Budget | `useUpdateBudget` |
| `approvals.ts` | Approval decisions, TOTP | `useApproveApproval`, `useRejectApproval`, `useBatchResolveApprovals`, `useTotpSetup` |
| `autoDream.ts` | Memory consolidation | `useTriggerAutoDream`, `useAbortAutoDream`, `useSetAutoDreamEnabled` |
| `channels.ts` | Channel config and testing | `useConfigureChannel`, `useTestChannel`, `useReloadChannels` |
| `config.ts` | Configuration and batch writes | `useSetConfigValue`, `useBatchSetConfigValues`, `useReloadConfig` |
| `goals.ts` | Goal CRUD | `useCreateGoal`, `useUpdateGoal`, `useDeleteGoal` |
| `hands.ts` | Hand lifecycle and messaging | `useActivateHand`, `useDeactivateHand`, `useSendHandMessage` |
| `mcp.ts` | MCP server management | `useAddMcpServer`, `useUpdateMcpServer`, `useReconnectMcpServer` |

## Endpoint Coverage by Domain

### Agents (`/api/agents/*`)

Listing, CRUD, session management, prompt versioning, and A/B experiments. Two distinct patch endpoints exist:

- `patchAgent(id, body)` — manifest-level updates (name, system prompt, MCP servers, model).
- `patchAgentConfig(id, config)` — runtime model tuning only (max tokens, temperature, web search augmentation).

### Workflows (`/api/workflows/*`)

Full lifecycle including a `dryRunWorkflow` endpoint that validates variable expansion without making LLM calls, and `saveWorkflowAsTemplate` to promote a workflow into a reusable template.

### Skills (`/api/skills/*`, hubs)

Three skill registries are surfaced:

- **Local skills** — `listSkills`, `installSkill`, evolution APIs (`evolveUpdateSkill`, `evolvePatchSkill`, `evolveRollbackSkill`).
- **ClawHub** — community marketplace (`clawhubBrowse`, `clawhubSearch`, `clawhubInstall`).
- **SkillHub** — alternative registry (`skillhubBrowse`, `skillhubSearch`, `skillhubInstall`).
- **FangHub** — official LibreFang registry (`fanghubListSkills`).

### Hands (`/api/hands/*`)

Hands are persistent agent orchestrations. The API supports activation, pausing, settings management, secret injection, session messaging, and per-instance stats. Built-in hands cannot be uninstalled; `uninstallHand` works only for user-installed custom hands.

### Memory (`/api/memory/*`)

CRUD for memory items, semantic search, stats, config (embedding provider/model, decay rate, proactive memory toggles), and manual `decayMemories`/`cleanupMemories` operations.

### MCP Servers (`/api/mcp/*`)

Unified API for both raw-configured and catalog-installed servers. Includes health monitoring, OAuth flow (`startMcpAuth`, `revokeMcpAuth`), and a browseable catalog.

### Media (`/api/media/*`)

Image generation, speech synthesis, audio transcription (accepts a `Blob` directly with appropriate content type), video generation with polling, and music generation.

### Configuration (`/api/config/*`)

Read full config, get schema (with field types, options, and validation constraints), set individual values, export raw TOML, and reload. The reload response includes `restart_required` and `restart_reasons` so the UI can prompt the user.

### Other domains

- **Channels** — messaging channel setup (WeChat/WhatsApp QR flows), testing, reload.
- **Comms** — inter-agent topology, events, message passing, task posting.
- **Security** — security status, audit trail, approval queue with TOTP second factor.
- **Budget/Usage** — cost tracking by agent/model/day, budget limits.
- **Terminal** — tmux window management (list, create, rename, delete).
- **Auto-Dream** — background memory consolidation status, triggering, and abort.
- **Network/Peers/A2A** — p2p networking and agent-to-agent discovery.
- **Plugins** — install, uninstall, scaffold, dependency installation, registry listing.

## Adding a New Endpoint

1. **Define the response interface** in `api.ts` if the backend returns a structured shape.
2. **Implement the function** in `api.ts` using the appropriate HTTP primitive. Use `encodeURIComponent` on all path parameters. Use `LONG_RUNNING_TIMEOUT_MS` for operations that invoke LLM calls.
3. **Re-export the function** in `lib/http/client.ts` in the correct section (query vs. mutation).
4. **Re-export any new types** in the type section of `client.ts`.
5. **Create a mutation hook** in the appropriate `lib/mutations/*.ts` file if the endpoint is a write operation. Add cache invalidation using the relevant query key factory.
6. **Create a query hook** in `lib/queries/*.ts` if the endpoint is a read operation (not shown in this module but follows the same pattern through `client.ts`).

## Conventions

- All path parameters are encoded with `encodeURIComponent`.
- Optional query parameters are built with `URLSearchParams` and conditionally appended.
- Response arrays default to `[]` when the backend returns `undefined` (e.g., `data.providers ?? []`).
- Integer query parameters are clamped to sane bounds (e.g., `listAuditRecent` limits to 1–1000).
- The `Accept-Language` header is attached to every request for i18n-aware error messages from the backend.