# Extensions System

# LibreFang Extensions System

The extensions crate provides a one-click integration system for MCP (Model Context Protocol) servers. It handles the complete lifecycle: discovering integrations, resolving credentials, running OAuth flows, installing, health monitoring, and generating the MCP server configs that the kernel consumes.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Installer                            │
│              (install_integration / remove / list)           │
├──────────┬──────────┬──────────────┬────────────────────────┤
│ Registry │Credential│    OAuth     │   Health Monitor       │
│ (templates│Resolver │  (PKCE flow) │ (DashMap + backoff)    │
│  + state) │(4-source│              │                        │
│           │ chain)  │              │                        │
├──────────┴────┬─────┴──────────────┴────────────────────────┤
│     Vault     │          Dotenv / Env Vars                  │
│ (AES-256-GCM) │                                             │
└───────────────┴─────────────────────────────────────────────┘
```

Data flows downward: the installer orchestrates the registry, resolver, and OAuth modules. The resolver pulls secrets from the vault, dotenv files, environment variables, or interactive prompts. The registry outputs `McpServerConfigEntry` objects that the kernel merges into its MCP server list.

## Key Types (lib.rs)

All shared types live in the crate root. The most important ones:

| Type | Purpose |
|------|---------|
| `IntegrationTemplate` | A bundled integration definition loaded from a TOML file. Contains transport config, required env vars, OAuth config, health check settings, and metadata. |
| `InstalledIntegration` | Persisted record of an installed integration (written to `integrations.toml`). |
| `IntegrationStatus` | State enum: `Ready`, `Setup`, `Available`, `Error(String)`, `Disabled`. |
| `IntegrationInfo` | Combined view merging a template with its install state and tool count. |
| `McpTransportTemplate` | How to launch the MCP server: `Stdio { command, args }`, `Sse { url }`, or `Http { url }`. |
| `RequiredEnvVar` | Describes a credential an integration needs — name, label, help text, whether it's secret, and a URL to create the key. |
| `ExtensionError` | Error enum covering all failure modes (`NotFound`, `VaultLocked`, `OAuth`, etc.). |

### File Layout on Disk

```
~/.librefang/
├── integrations/          # Template TOML files (one per integration)
│   ├── github.toml
│   ├── slack.toml
│   └── ...
├── integrations.toml      # Installed integrations state
├── vault.enc              # AES-256-GCM encrypted secrets (OFV1 magic header)
└── .env                   # Optional dotenv fallback
```

---

## Module Reference

### Registry (`registry.rs`)

`IntegrationRegistry` is the central store. It loads integration templates from `~/.librefang/integrations/*.toml` at startup, tracks which are installed via `integrations.toml`, and converts installed integrations to `McpServerConfigEntry` values for the kernel.

```rust
let mut registry = IntegrationRegistry::new(home_dir);
registry.load_templates(home_dir);   // loads *.toml from integrations/
registry.load_installed()?;           // reads integrations.toml

// Query
registry.get_template("github");     // Option<&IntegrationTemplate>
registry.is_installed("github");     // bool
registry.search("git");              // Vec<&IntegrationTemplate>
registry.list_by_category(&IntegrationCategory::DevTools);

// Mutate (persists to integrations.toml)
registry.install(entry)?;
registry.uninstall("github")?;
registry.set_enabled("github", false)?;

// Output for kernel
let configs: Vec<McpServerConfigEntry> = registry.to_mcp_configs();
```

Templates are TOML files matching the `IntegrationTemplate` struct. A minimal example:

```toml
id = "github"
name = "GitHub"
description = "GitHub API via MCP"
category = "devtools"
icon = "🐙"

[transport]
type = "stdio"
command = "npx"
args = ["-y", "@modelcontextprotocol/server-github"]

[[required_env]]
name = "GITHUB_PERSONAL_ACCESS_TOKEN"
label = "Personal Access Token"
help = "Create at https://github.com/settings/tokens"
is_secret = true
get_url = "https://github.com/settings/tokens"

[health_check]
interval_secs = 60
unhealthy_threshold = 3
```

### Credential Vault (`vault.rs`)

AES-256-GCM encrypted storage for secrets, persisted at `~/.librefang/vault.enc`. The file format uses an `OFV1` magic header followed by a JSON payload containing the Argon2id salt, nonce, and ciphertext (all base64-encoded).

**Master key resolution order:**
1. Cached key in memory (avoids repeated keyring/env lookups)
2. OS keyring (file-based fallback at `~/.local/share/librefang/.keyring`, AES-256-GCM wrapped with a machine fingerprint)
3. `LIBREFANG_VAULT_KEY` environment variable (base64-encoded 32-byte key)

**Encryption pipeline:** master key + random salt → Argon2id → derived key → AES-256-GCM encrypt the JSON-serialized secret map.

```rust
let mut vault = CredentialVault::new(vault_path);
vault.init_with_key(master_key)?;           // create vault file
vault.set("API_KEY".into(), Zeroizing::new("secret".into()))?;
let val = vault.get("API_KEY");             // Option<Zeroizing<String>>
vault.remove("API_KEY")?;
```

For production use, call `vault.init()` which generates a random key and stores it in the OS keyring, or `vault.unlock()` which resolves the key from keyring/env.

The `Drop` implementation clears all in-memory entries and the cached key. All secret values use `Zeroizing<String>` to ensure memory is zeroed on deallocation.

### Credential Resolver (`credentials.rs`)

Resolves secrets from four sources in priority order:

1. **Vault** — `~/.librefang/vault.enc` (if unlocked)
2. **Dotenv** — `~/.librefang/.env` (loaded at construction time)
3. **Environment variable** — `std::env::var`
4. **Interactive prompt** — stderr prompt + stdin read (only when `interactive` is enabled)

```rust
let resolver = CredentialResolver::new(Some(vault), Some(dotenv_path))
    .with_interactive(true);

// Single credential
let token = resolver.resolve("GITHUB_TOKEN");  // Option<Zeroizing<String>>

// Batch
let creds = resolver.resolve_all(&["KEY_A", "KEY_B", "KEY_C"]);
let missing = resolver.missing_credentials(&["KEY_A", "KEY_B"]);
```

Dotenv parsing handles `KEY=VALUE`, `KEY="quoted"`, `KEY='single'`, comments (`#`), and blank lines. The dotenv cache is loaded once at construction; call `clear_dotenv_cache` if keys are deleted at runtime.

### OAuth2 PKCE (`oauth.rs`)

Implements the Authorization Code flow with PKCE (Proof Key for Code Exchange) for Google, GitHub, Microsoft, and Slack. No client secret is needed — safe for desktop/CLI apps.

**Flow:**
1. Generate random PKCE verifier → SHA-256 → base64url challenge
2. Bind a localhost TCP listener on a random port
3. Open the user's browser to the authorization URL (with `code_challenge`)
4. Serve a one-shot callback at `/callback` via axum
5. Exchange the authorization code for tokens (POST to `token_url` with `code_verifier`)
6. Return `OAuthTokens` (access_token, refresh_token, expires_in, scope)

```rust
let tokens = run_pkce_flow(&template.oauth.unwrap(), client_id).await?;
// tokens.access_token_zeroizing() -> Zeroizing<String>
```

Client IDs default to placeholder values. Override them via `OAuthConfig` in the app config:

```rust
let client_ids = resolve_client_ids(&config.oauth);
// merges defaults with config.google_client_id, config.github_client_id, etc.
```

The callback server has a 5-minute timeout. State parameter is validated for CSRF protection.

### Health Monitor (`health.rs`)

Thread-safe health tracking for MCP server connections using `DashMap`. Each integration has an `IntegrationHealth` record tracking status, tool count, consecutive failures, reconnect attempts, and timestamps.

```rust
let monitor = HealthMonitor::new(HealthMonitorConfig::default());
monitor.register("github");

// Background task reports
monitor.report_ok("github", 12);              // 12 tools available
monitor.report_error("github", "timeout".into());

// Query
let health = monitor.get_health("github");    // Option<IntegrationHealth>
let all = monitor.all_health();               // Vec<IntegrationHealth>

// Reconnect logic
if monitor.should_reconnect("github") {
    monitor.mark_reconnecting("github");
    let backoff = monitor.backoff_duration(attempt);  // exponential
}
```

**Backoff:** 5s → 10s → 20s → 40s → 80s → ... → capped at 300s (5 min). Maximum 10 reconnect attempts (configurable). After recovery via `mark_ok`, failure counters and reconnect state reset.

Default health check interval is 60 seconds; an integration is marked unhealthy after 3 consecutive failures.

### Installer (`installer.rs`)

Orchestrates the complete install/remove flow:

**`install_integration`** — looks up the template, stores any provided keys in the vault, checks for missing credentials, writes the install record to `integrations.toml`, and returns an `InstallResult` with status `Ready` (all credentials present) or `Setup` (credentials still needed).

**`remove_integration`** — unregisters from the registry and persists the change.

**`list_integrations`** — returns all templates with computed status based on install state and credential availability.

**`search_integrations`** — fuzzy text search across template ID, name, description, and tags.

**`scaffold_integration`** / **`scaffold_skill`** — generate starter TOML templates for custom integrations or skills.

```rust
// Install with a provided API key
let mut keys = HashMap::new();
keys.insert("NOTION_API_KEY".into(), "ntn_xxx".into());
let result = install_integration(&mut registry, &mut resolver, "notion", &keys)?;
// result.status is Ready if all required_env are satisfied
```

### HTTP Client (`http_client.rs`)

Shared `reqwest::Client` builder with TLS root certificates sourced from the native certificate store, falling back to `webpki-roots` if no native certs are found. Uses `rustls` with the `aws_lc_rs` crypto provider.

```rust
let client = new_client();  // reqwest::Client with bundled CA roots
```

---

## Integration with the Kernel

During boot, the kernel calls `vault.unlock()` to decrypt secrets, then the registry converts installed integrations into `McpServerConfigEntry` values via `to_mcp_configs()`. These are merged into the kernel's MCP server list, and the health monitor tracks their status in the background.

The vault is used across the codebase — routes like `list_providers`, `auth_start`, and `wechat_qr_status` all flow through `boot_with_config` → `vault.unlock()` → `resolve_master_key()`, which checks the OS keyring and env var.