# Other — librefang-http

# librefang-http

Shared HTTP client builder providing proxy support and TLS certificate fallback for the LibreFang project.

## Purpose

This crate centralizes HTTP client construction so that every component in LibreFang that needs to make outbound requests uses a consistent, correctly configured `reqwest` client. It handles two concerns that are easy to get wrong in isolation:

- **TLS root certificates** — Attempts to load native system certificates first via `rustls-native-certs`; if that fails or produces an empty set, it falls back to the Mozilla bundle bundled by `webpki-roots`.
- **Proxy configuration** — Builds clients that respect environment proxy settings transparently.

By encapsulating this logic once, the rest of the workspace avoids duplicating brittle TLS setup code.

## Dependencies

| Dependency | Role |
|---|---|
| `reqwest` | Underlying HTTP client, compiled with `rustls` TLS backend |
| `rustls` | TLS implementation (no OpenSSL dependency) |
| `rustls-native-certs` | Loads CA certificates from the operating system's trust store |
| `webpki-roots` | Mozilla's curated CA certificate bundle, used as fallback |
| `tracing` | Diagnostic logging for certificate loading outcomes and errors |
| `librefang-types` | Shared type definitions across the LibreFang workspace |

## How It Works

### TLS Certificate Loading Strategy

The builder follows a two-stage approach:

1. **Primary path** — Call into `rustls-native-certs` to load certificates from the platform's native store (`/etc/ssl/certs` on Linux, the Windows certificate store, or the Keychain on macOS).
2. **Fallback path** — If native cert loading returns an error or an empty set, `webpki-roots` provides the Mozilla CA bundle as a reliable default.

Both outcomes are logged via `tracing` so operators can diagnose TLS handshake failures by inspecting logs.

### Client Construction

The crate exposes a builder (or builder function) that returns a configured `reqwest::Client`. Consumers should call into this crate rather than constructing their own `reqwest::Client` directly, ensuring uniform proxy behavior, TLS configuration, and timeout defaults across all LibreFang HTTP traffic.

## Integration with LibreFang

`librefang-http` sits between the shared types crate and any component that performs outbound HTTP calls:

```
librefang-types  ←  librefang-http  ←  (consumers: scanners, updaters, etc.)
```

Any workspace member that needs to make HTTP requests should depend on this crate and use its client builder. This guarantees that:

- Proxy settings are honored everywhere or nowhere consistently.
- TLS trust roots are identical across all components.
- Changes to the HTTP stack (timeouts, headers, user-agents) can be made in one place.