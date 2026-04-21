# Skills & Extensions — librefang-extensions-src

# LibreFang Extensions (`librefang-extensions`)

MCP server catalog, encrypted credential vault, OAuth2 PKCE flows, health monitoring, and installation plumbing.

## Architecture Overview

```mermaid
graph TD
    subgraph "Disk"
        A["~/.librefang/mcp/catalog/*.toml"]
        B["~/.librefang/vault.enc"]
        C["~/.librefang/.env"]
        D["~/.librefang/config.toml"]
    end

    subgraph "librefang-extensions"
        MC["McpCatalog<br/>(read-only templates)"]
        CR["CredentialResolver<br/>(4-source chain)"]
        CV["CredentialVault<br/>(AES-256-GCM)"]
        DE["dotenv loader<br/>(process env injection)"]
        HM["HealthMonitor<br/>(DashMap + backoff)"]
        OA["OAuth PKCE<br/>(localhost callback)"]
        IN["Installer<br/>(catalog → config entry)"]
    end

    A --> MC
    B --> CV
    C --> DE
    MC --> IN
    CR --> IN
    CV --> CR
    DE --> CR
    IN -->|writes [[mcp_servers]]| D
```

## Module Layout

| Module | Visibility | Purpose |
|---|---|---|
| `catalog` | pub | In-memory view of MCP server templates from `~/.librefang/mcp/catalog/` |
| `credentials` | pub | Multi-source credential resolution chain |
| `dotenv` | pub | Load vault + `.env` files into process environment at startup |
| `health` | pub | Track MCP server health with auto-reconnect backoff |
| `http_client` | pub(crate) | Shared `reqwest` client with bundled CA roots |
| `installer` | pub | Pure transform: catalog entry → `McpServerConfigEntry` |
| `oauth` | pub | OAuth2 PKCE localhost callback flow |
| `vault` | pub | AES-256-GCM encrypted credential storage |

---

## Core Types

The crate root (`src/lib.rs`) defines the shared data model used across all submodules.

### Error Handling

All fallible operations return `ExtensionResult<T>` (`Result<T, ExtensionError>`). The error enum covers:

- `NotFound(id)` — catalog entry doesn't exist
- `AlreadyInstalled(id)` / `NotInstalled(id)` — duplicate or missing server
- `CredentialNotFound(key)` — key absent from all sources
- `Vault(msg)` / `VaultLocked` — vault I/O or locked state
- `OAuth(msg)` — PKCE flow failures
- `TomlParse(msg)` / `Io` / `Http(msg)` — infrastructure errors
- `HealthCheck(msg)` — monitor failures

### Catalog Model

**`McpCatalogEntry`** — a TOML-deserializable template describing how to configure an MCP server:

```rust
pub struct McpCatalogEntry {
    pub id: String,               // e.g. "github"
    pub name: String,             // e.g. "GitHub"
    pub description: String,
    pub category: McpCategory,    // DevTools, Productivity, Communication, Data, Cloud, AI
    pub icon: String,             // emoji
    pub transport: McpCatalogTransport,  // Stdio, Sse, or Http
    pub required_env: Vec<McpCatalogRequiredEnv>,
    pub oauth: Option<OAuthTemplate>,
    pub tags: Vec<String>,
    pub setup_instructions: String,
    pub health_check: HealthCheckConfig,
}
```

`McpCatalogTransport` mirrors `librefang_types::config::McpTransportEntry` but excludes `HttpCompat` (a power-user-only transport that doesn't ship as a template). The installer converts between the two.

`McpCatalogRequiredEnv` describes each credential the server needs — name, human label, help text, whether it's secret, and a URL where the user can create the key. Defaults `is_secret` to `true`.

### Server Status

**`McpStatus`** tracks the lifecycle:

| Variant | Meaning |
|---|---|
| `Available` | In catalog only, not yet configured |
| `Setup` | Configured but credentials missing |
| `Ready` | Running with tools available |
| `Error(String)` | Server errored |
| `Disabled` | User-disabled |

---

## Catalog (`catalog`)

**`McpCatalog`** holds an in-memory `HashMap<String, McpCatalogEntry>` loaded from `~/.librefang/mcp/catalog/`.

### File Layout

Two layouts are supported:

- **Flat:** `<id>.toml` — id derived from filename minus extension
- **Directory:** `<id>/MCP.toml` — for multi-file MCP packages, id from directory name

This mirrors `web/scripts/fetch-registry.ts` so catalog loading, the live API, and the UI all agree.

### Loading

```rust
let mut catalog = McpCatalog::new(&home_dir);
let count = catalog.load(&home_dir);
```

`load` performs a full reload — it clears the internal map first so entries deleted on disk don't linger. Returns the count of successfully parsed entries. Malformed files are skipped with a `tracing::warn`.

### Querying

- `get(id)` — lookup by exact id
- `list()` — all entries sorted by id
- `list_by_category(&McpCategory)` — filtered list
- `search(query)` — case-insensitive search across id, name, description, and tags

Catalog entries are refreshed from the upstream registry by `librefang_runtime::registry_sync`. The catalog module itself is purely read-only.

---

## Credential Vault (`vault`)

AES-256-GCM encrypted storage at `~/.librefang/vault.enc`. The vault uses Argon2id for key derivation and `Zeroizing<String>` for all secret material in memory.

### Master Key Resolution

The master key is resolved in this order:

1. **Cached key** — from a previous `unlock()` or `init()` call on this instance
2. **OS keyring** — file-based fallback at `$LOCAL_DATA/librefang/.keyring`, AES-256-GCM wrapped with a machine fingerprint (Argon2id-derived from username + hostname)
3. **Environment variable** — `LIBREFANG_VAULT_KEY` (base64-encoded 32-byte key)

In test builds, the OS keyring path is disabled; tests use `init_with_key` / `unlock_with_key` with explicit keys.

### On-Disk Format

The vault file starts with a 4-byte magic header `OFV1`, followed by a JSON `VaultFile`:

```json
{
  "version": 1,
  "salt": "<base64 Argon2 salt>",
  "nonce": "<base64 AES-GCM nonce>",
  "ciphertext": "<base64 encrypted VaultEntries>"
}
```

Legacy vault files without the `OFV1` header (starting with `{`) are still accepted for backward compatibility. Unrecognized formats produce an `"Unrecognized vault file format"` error.

### Key Lifecycle

```
init() / init_with_key()  →  generates vault.enc with empty entries
unlock() / unlock_with_key()  →  decrypts and loads entries into memory
set(key, value)  →  inserts + re-encrypts + writes to disk
get(key)  →  returns Zeroizing<String> clone
remove(key)  →  removes + re-encrypts if changed
```

Every `set` and `remove` immediately re-encrypts and writes the entire vault to disk. The `Drop` implementation clears the in-memory entries and cached key.

### OS Keyring Fallback

When the native OS keyring is unavailable, the crate falls back to a file at `$LOCAL_DATA/librefang/.keyring`. This file wraps the master key with AES-256-GCM using an Argon2id-derived key from the machine fingerprint. Legacy v1 files (XOR-obfuscated) are automatically migrated to v2 on first load.

---

## Credential Resolution (`credentials`)

**`CredentialResolver`** tries multiple sources in priority order:

```
1. Vault (~/.librefang/vault.enc)
2. Dotenv file (~/.librefang/.env)
3. Process environment variable
4. Interactive prompt (CLI only, opt-in)
```

### Usage

```rust
let vault = CredentialVault::new(vault_path);
let resolver = CredentialResolver::new(Some(vault), Some(dotenv_path))
    .with_interactive(true);

// Single key
if let Some(val) = resolver.resolve("GITHUB_TOKEN") {
    // val is Zeroizing<String>
}

// Batch check
let missing = resolver.missing_credentials(&["KEY_A", "KEY_B", "KEY_C"]);

// Batch resolve
let all = resolver.resolve_all(&["KEY_A", "KEY_B"]);

// Store back to vault
resolver.store_in_vault("KEY_A", Zeroizing::new("secret".into()))?;
```

The dotenv file is loaded once at construction. If a key is deleted via the dashboard at runtime, call `clear_dotenv_cache(&key)` to prevent stale values from being returned.

The internal `load_dotenv` parser handles `#` comments, `KEY=VALUE`, and surrounding quotes (double or single).

---

## Dotenv Loader (`dotenv`)

**Not to be confused with `credentials::load_dotenv`.** This module provides `dotenv::load_dotenv()` — a `Once`-gated function called from the synchronous `main()` of every binary *before* spawning any tokio runtime. This is required because `std::env::set_var` is undefined behavior in Rust 1.80+ once other threads exist.

### Priority Order (highest first)

1. System environment variables (already present) — **never overridden**
2. Credential vault (`vault.enc`)
3. `~/.librefang/.env`
4. `~/.librefang/secrets.env`

The `Once` guard ensures repeated calls are no-ops.

### Managed .env Editing

The module also exposes functions for programmatic `.env` management:

- `save_env_key(key, value)` — upsert into `.env` with 0600 permissions (Unix), also sets in process env
- `remove_env_key(key)` — remove from `.env` and process env
- `list_env_keys()` — key names only (no values)
- `env_file_exists()` — existence check

The home directory respects the `LIBREFANG_HOME` env var, falling back to `~/.librefang`.

---

## Health Monitor (`health`)

**`HealthMonitor`** tracks the status of all configured MCP servers using a `DashMap<String, McpHealth>` for lock-free concurrent access from background tokio tasks.

### McpHealth Record

```rust
pub struct McpHealth {
    pub id: String,
    pub status: McpStatus,
    pub tool_count: usize,
    pub last_ok: Option<DateTime<Utc>>,
    pub last_error: Option<String>,
    pub consecutive_failures: u32,
    pub reconnecting: bool,
    pub reconnect_attempts: u32,
    pub connected_since: Option<DateTime<Utc>>,
}
```

### Auto-Reconnect with Exponential Backoff

Backoff is calculated as `5s × 2^attempt`, capped at `max_backoff_secs` (default 300s = 5 min):

| Attempt | Delay |
|---|---|
| 0 | 5s |
| 1 | 10s |
| 2 | 20s |
| 3 | 40s |
| ... | ... |
| 6+ | 300s (capped) |

After `max_reconnect_attempts` (default 10), the monitor stops trying. A successful health check resets all failure counters.

### Configuration

```rust
pub struct HealthMonitorConfig {
    pub auto_reconnect: bool,           // default: true
    pub max_reconnect_attempts: u32,    // default: 10
    pub max_backoff_secs: u64,          // default: 300
    pub check_interval_secs: u64,       // default: 60
}
```

### Integration Pattern

The kernel registers servers, then a background task periodically pings them:

```rust
let monitor = HealthMonitor::new(HealthMonitorConfig::default());
monitor.register("github");

// Background task calls:
monitor.report_ok("github", 12);  // 12 tools available
// or:
monitor.report_error("github", "Connection refused".into());

// Check if reconnect warranted:
if monitor.should_reconnect("github") {
    monitor.mark_reconnecting("github");
    let delay = monitor.backoff_duration(attempt);
    // ... reconnect logic ...
}
```

`health_map()` returns the `Arc<DashMap>` for direct use by background tasks.

---

## OAuth2 PKCE (`oauth`)

Implements the full Authorization Code Flow with PKCE for Google, GitHub, Microsoft, and Slack.

### Flow

1. Generate random PKCE verifier (32 bytes, base64url) and S256 challenge
2. Bind a random port on `127.0.0.1`
3. Build authorization URL with `code_challenge` and `state` parameters
4. Open browser (falls back to printing the URL)
5. Listen for callback on `/callback` with a 5-minute timeout
6. Exchange authorization code for tokens at the provider's token endpoint
7. Return `OAuthTokens` (access_token, refresh_token, expires_in, scope)

### Client IDs

Default placeholder client IDs are provided. Override via `OAuthConfig`:

```rust
let config = OAuthConfig {
    google_client_id: Some("real-google-id".into()),
    ..Default::default()
};
let ids = resolve_client_ids(&config);
```

Client IDs are safe to embed because PKCE doesn't require a client_secret.

### Token Handling

`OAuthTokens` provides `access_token_zeroizing()` and `refresh_token_zeroizing()` to obtain `Zeroizing<String>` wrappers for vault storage.

---

## Installer (`installer`)

Pure transforms — no side effects. Converts a catalog template into a `McpServerConfigEntry` that the caller persists to `config.toml`.

### `install_integration`

```rust
pub fn install_integration(
    catalog: &McpCatalog,
    resolver: &mut CredentialResolver,
    id: &str,
    provided_keys: &HashMap<String, String>,
) -> ExtensionResult<InstallResult>
```

Steps:

1. Look up the template by `id` in the catalog
2. Store any `provided_keys` in the vault (best-effort; warns on failure)
3. Check which `required_env` keys are still missing across all sources
4. Build an `McpServerConfigEntry` with `template_id` set to the catalog id
5. Return `InstallResult` with status (`Ready` if all creds present, `Setup` otherwise)

The returned `InstallResult.server` should be written to `config.toml` under `[[mcp_servers]]` by the caller, who then triggers a kernel reload.

### `catalog_entry_to_mcp_server`

Maps transport types:

| `McpCatalogTransport` | `McpTransportEntry` |
|---|---|
| `Stdio { command, args }` | `Stdio { command, args }` |
| `Sse { url }` | `Sse { url }` |
| `Http { url }` | `Http { url }` |

Also maps `OAuthTemplate` → `McpOAuthConfig`, collecting required env var names into the `env` field.

### Scaffolding

- `scaffold_integration(dir)` — creates a `mcp.toml` template for a custom MCP server
- `scaffold_skill(dir)` — creates `skill.toml` + `SKILL.md` for a new skill

---

## HTTP Client (`http_client`)

Internal shared `reqwest` client builder. Loads native CA certificates first; if none are found, falls back to `webpki_roots`. Uses `rustls` with `aws_lc_rs` crypto provider.

```rust
// Internal use only
let client = new_client();  // reqwest::Client
```

---

## Integration with the Rest of the Codebase

### Startup Sequence

```
main (CLI/desktop/kernel)
  → dotenv::load_dotenv()          // injects vault + .env into process env (sync, Once-gated)
  → kernel boots
  → catalog.load(&home_dir)        // reads template files
  → health monitor starts          // background tokio task
```

### Cross-Crate Callers

- **`librefang-kernel`** — calls `vault::unlock()` during `vault_get` for OAuth token retrieval and dashboard credential resolution
- **`librefang-skills`** — heavily uses `Path::exists()` checks via the extensions crate for skill file operations
- **`librefang-api`** — resolves dashboard credentials by unlocking the vault

### Data Flow: Installation

```
User selects "GitHub" in TUI/API
  → catalog.get("github")
  → installer::install_integration(catalog, resolver, "github", provided_keys)
  → resolver.store_in_vault() for each provided key
  → returns InstallResult { server, status, missing_credentials }
  → caller writes server to config.toml [[mcp_servers]]
  → kernel reloads config, connects to MCP server
  → health monitor reports status
```