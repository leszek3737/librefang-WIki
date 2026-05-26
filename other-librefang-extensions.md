# Other — librefang-extensions

# librefang-extensions

Extension and integration toolkit for LibreFang — credential vault, MCP server management, OAuth2 PKCE flows, shared HTTP client construction, and workspace environment parsing. This crate sits above `librefang-kernel` and provides the cross-cutting infrastructure that agents need but that doesn't belong in the runtime or kernel layers.

## Architecture

```mermaid
graph TD
    subgraph "Higher layers"
        API[librefang-api]
        CLI[librefang-cli]
        Desktop[librefang-desktop]
    end

    subgraph "This crate"
        Vault[vault — CredentialVault]
        Catalog[catalog — MCP metadata]
        OAuth[oauth — PKCE + DCR]
        Installer[installer — MCP lifecycle]
        Creds[credentials — resolve/resolve_all]
        Health[health — liveness probes]
        HTTP[http_client — reqwest builder]
        DotEnv[dotenv — .env parsing]
    end

    Kernel[librefang-kernel]
    Types[librefang-types]

    API --> Vault
    API --> OAuth
    API --> Catalog
    API --> Installer
    CLI --> Vault
    CLI --> Installer
    Desktop --> Health

    Vault --> Types
    Catalog --> Types
    OAuth --> HTTP
    Installer --> HTTP
    Health --> HTTP
    Creds --> Types

    Vault -->|AES-256-GCM| Keyring["OS keyring / file fallback"]
end
```

**Dependency direction**: Nothing in this crate may import from `librefang-api`, `librefang-cli`, or `librefang-desktop`. The crate depends on `librefang-types` and `librefang-kernel` only.

## Module Reference

### `vault` — AES-256-GCM Credential Vault

The `CredentialVault` struct provides encrypted storage for secrets. Encryption is AES-256-GCM. The master encryption key has two storage backends, selected at compile time and runtime:

| Platform | Primary backend | Fallback |
|---|---|---|
| Linux (glibc) | libsecret via OS keyring | File-based encrypted store |
| Windows | Credential Manager via OS keyring | File-based encrypted store |
| macOS | File-based (see `use_os_keyring` plumbing) | — |
| Linux (musl) | File-based (OS keyring not compiled in) | — |
| Android | File-based (OS keyring not compiled in) | — |

The OS keyring dependency (`keyring` crate) is target-gated in `Cargo.toml` to avoid pulling `libdbus-sys` on musl-static and Android targets where there is no usable secret-service backend. The vault transparently falls back to the file-based AES-256-GCM store when no OS keyring backend is compiled in.

Key handling rules (see top-level `CLAUDE.md` for the authoritative cross-cutting rules):
- `LIBREFANG_VAULT_KEY` environment variable can override key source
- Constraints on key format and rotation are defined at the workspace level

### `catalog` — MCP Server Catalog

Manages MCP server catalog metadata stored under `~/.librefang/mcp/catalog/`. This module handles reading, writing, and querying the local catalog of available MCP servers and their configurations.

### `installer` — MCP Server Lifecycle

Handles the install, update, and uninstall flows for MCP servers. This module orchestrates downloading, verifying, registering, and removing MCP server packages. It uses the shared HTTP client from `http_client` for network operations.

### `oauth` — OAuth2 PKCE and Dynamic Client Registration

Implements OAuth2 authorization with PKCE (Proof Key for Code Exchange) tailored for MCP server authentication. Supports RFC 7591 Dynamic Client Registration to programmatically register OAuth2 clients.

**Important boundary**: The `McpOAuthProvider` trait lives in `librefang-runtime`. The concrete implementation lives in `librefang-api`. This module provides the OAuth2 *client* machinery — the flow ownership and callback wiring belong to higher layers. Do not add HTTP routing or callback endpoints here.

Cross-cutting rules for Docker callback URLs and OAuth flow ownership between the daemon and API layer are defined in the top-level `CLAUDE.md`.

### `credentials` — Auth-Source Unification

Provides `resolve` and `resolve_all` functions that unify multiple authentication sources into a consistent credential resolution pipeline. This lets callers request credentials without knowing whether they come from the vault, environment variables, configuration files, or other sources.

### `http_client` — Shared reqwest Client Builder

Exposes two functions for constructing HTTP clients:

- **`client_builder()`** — Returns a configured `reqwest::ClientBuilder` that callers can further customize before building. Use this when the call site needs to add custom middleware, timeouts, or TLS configuration.
- **`new_client()`** — Returns a fully built `reqwest::Client` with sensible defaults. Use this for straightforward HTTP calls.

There is no `shared_client()` function. Use whichever of the above fits the call site.

The builder configures:
- TLS via `rustls` with `webpki-roots` and `rustls-native-certs` for certificate verification
- Standard user-agent and timeout defaults consistent across the project

### `health` — Provider Liveness Probes

Probes MCP providers for liveness and availability. Used by higher layers to determine whether a provider is reachable before attempting operations.

### `dotenv` — Workspace Environment Parsing

Parses `.env` files in agent workspaces, making environment variables available to agent processes. Handles standard `.env` format with variable substitution and quoting rules.

## Dependency Highlights

| Dependency | Purpose |
|---|---|
| `aes-gcm`, `argon2` | Vault encryption (AES-256-GCM) and key derivation (Argon2) |
| `keyring` | OS-native key storage (target-gated, see above) |
| `reqwest`, `rustls`, `webpki-roots` | HTTP client with rustls TLS backend |
| `sha2`, `hmac`, `subtle` | Cryptographic primitives for OAuth PKCE |
| `zeroize` | Secure memory zeroing for credential material |
| `dashmap` | Concurrent map for catalog and credential caches |
| `base64` | Encoding for OAuth and vault operations |

## Boundaries

**Owns**: Vault, MCP catalog, OAuth client logic, shared HTTP client builder, `.env` parsing, plugin installer, provider health probes, credential resolution.

**Does NOT own**: Kernel callback wiring, HTTP routing, channel adapters, the `McpOAuthProvider` trait definition (that's `librefang-runtime`), or its implementation (that's `librefang-api`).

Before adding code that crosses the crate boundary — especially anything touching vault keys, OAuth callback URLs, Docker networking, or the auth middleware allowlist — read the cross-cutting rules in the top-level `CLAUDE.md`. Those rules are authoritative and are not duplicated here to avoid drift.

## Testing

Dev-dependencies include `tempfile` for filesystem-based tests, `serial_test` for tests that share global state (keyring, file-based vault), and `librefang-runtime` for integration scenarios.