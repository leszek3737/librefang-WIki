# Other — librefang-extensions

# librefang-extensions

The "everything-side-of-an-agent" toolkit — extension and integration utilities that don't belong in the kernel or runtime layers. Provides MCP server catalog management, encrypted credential storage, OAuth2 PKCE flows, provider health probing, plugin installation, shared HTTP client construction, and `.env` file parsing.

## Architecture

```mermaid
graph TD
    A[librefang-api / CLI / desktop] -->|depends on| EXT[librefang-extensions]
    EXT -->|depends on| T[librefang-types]
    K[librefang-kernel] -.->|used by| EXT
    EXT -.->|McpOAuthProvider trait defined in| R[librefang-runtime]

    subgraph EXT
      C[catalog] V[vault] O[oauth]
      CR[credentials] H[health] I[installer]
      HC[http_client] DE[dotenv]
    end
```

**Dependency boundary:** This crate sits above `librefang-kernel` and depends only on `librefang-types`. Nothing in this crate may import from `librefang-api`, the CLI, or the desktop layer.

## Module Map

| Module | Purpose |
|---|---|
| `catalog` | MCP server catalog metadata stored under `~/.librefang/mcp/catalog/` |
| `credentials` | Auth-source unification — `resolve()` and `resolve_all()` to gather credentials from multiple backends |
| `dotenv` | `.env` file parsing for agent workspaces |
| `health` | Provider liveness probes — check whether an MCP server or external service is reachable |
| `http_client` | Shared `reqwest` client builder — `client_builder()` and `new_client()` |
| `installer` | MCP server install, update, and uninstall flows |
| `oauth` | OAuth2 PKCE client with Dynamic Client Registration (RFC 7591) for MCP servers |
| `vault` | AES-256-GCM encrypted credential vault (`CredentialVault`) |

## Key Components

### Credential Vault (`vault`)

`CredentialVault` stores secrets using AES-256-GCM encryption. The master key has two storage strategies depending on the compile target and OS:

- **Linux (glibc) / Windows:** OS keyring via `libsecret` / Credential Manager (the `keyring` crate). macOS also supports this path but see note below.
- **musl-static / Android / fallback:** File-based AES-256-GCM store. The vault transparently falls back when no OS keyring backend is compiled in.
- **macOS:** File fallback is the default due to `use_os_keyring` gating — see the `use_os_keyring` plumbing in `vault.rs`.

The `keyring` dependency is target-gated in `Cargo.toml` to avoid pulling `libdbus-sys` on musl and Android targets where no usable secret-service backend exists.

> **Important:** The `LIBREFANG_VAULT_KEY` environment variable constraints and vault key handling rules are defined in the top-level `CLAUDE.md`. Do not duplicate them here.

### OAuth2 PKCE (`oauth`)

Implements the full OAuth2 PKCE flow plus Dynamic Client Registration (RFC 7591) for MCP servers that require authorization. This module owns the OAuth *client* logic, but the `McpOAuthProvider` trait that wires it into the kernel lives in `librefang-runtime`, and its concrete implementation lives in `librefang-api`. Cross-cutting rules about Docker callback URLs and OAuth flow ownership between daemon and API are in the top-level `CLAUDE.md`.

### Shared HTTP Client (`http_client`)

Provides two entry points:

- **`client_builder()`** — returns a pre-configured `reqwest::ClientBuilder` with shared TLS settings (rustls + native cert roots), user-agent, and timeout defaults. Callers can add additional configuration before building.
- **`new_client()`** — builds a client immediately with the default shared configuration.

> **Note:** There is no `shared_client()` function. Previous documentation references to one were incorrect. Use `client_builder()` when you need to customize, `new_client()` when defaults suffice.

### MCP Catalog (`catalog`)

Manages MCP server metadata entries stored as files under `~/.librefang/mcp/catalog/`. Each entry describes a registered MCP server's connection details, capabilities, and configuration.

### Credential Resolution (`credentials`)

`resolve()` and `resolve_all()` unify auth sources — they look up credentials from the vault, environment variables, catalog metadata, or other configured backends, returning a normalized credential set for the caller.

### Plugin Installer (`installer`)

Handles the lifecycle of MCP server plugins: download, install, update, and uninstall. Orchestrates catalog updates alongside filesystem operations.

### Provider Health (`health`)

Liveness probes for external providers and MCP servers. Used to detect connectivity issues before attempting operations.

### Dotenv (`dotenv`)

Parses `.env` files in agent workspaces to inject environment variables into the agent's execution context.

## Compile-Time Feature Gates

The `keyring` crate (OS keyring integration) is conditionally compiled:

```toml
[target.'cfg(any(all(target_os = "linux", not(target_env = "musl")), target_os = "macos", target_os = "windows"))'.dependencies]
keyring = { workspace = true }
```

- **Included:** glibc Linux, macOS, Windows
- **Excluded:** musl-static Linux, Android, and any target without a native secret-service backend

When `keyring` is not compiled in, the vault automatically falls back to file-based encryption. No feature flags or runtime toggles are needed — the `os_keyring` function in `vault.rs` handles this transparently.

## Dependency Highlights

| Dependency | Role |
|---|---|
| `aes-gcm` / `argon2` | Symmetric encryption and key derivation for the vault |
| `rustls` / `webpki-roots` / `rustls-native-certs` | TLS for the shared HTTP client (no OpenSSL dependency) |
| `reqwest` | HTTP client construction |
| `axum` | Used for OAuth callback handling |
| `zeroize` | Secure memory zeroing for key material |
| `rand` / `sha2` / `hmac` / `subtle` | Cryptographic primitives for PKCE and vault operations |
| `dashmap` | Concurrent map for caching (catalog entries, health state) |
| `keyring` | OS-native keyring access (target-gated) |

## Adding Code to This Crate

Before adding code that crosses the crate boundary (especially OAuth flows, vault key handling, or auth middleware), read the cross-cutting rules in the top-level `CLAUDE.md`. Key constraints:

1. **Do not depend on `librefang-api`, CLI, or desktop crates.** This crate is a leaf above the kernel.
2. **OAuth flow ownership** is split: the trait lives in `librefang-runtime`, the implementation in `librefang-api`. This crate provides the client primitives.
3. **Vault key rules** (environment variable constraints, rotation) are centralized. Reference them, don't restate them.
4. **Auth middleware allowlist** rules live in the top-level project documentation.
5. **When in doubt about API names**, check the actual `src/` files — the agent notes in this crate have drifted before.