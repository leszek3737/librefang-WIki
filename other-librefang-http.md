# Other — librefang-http

# librefang-http

Shared HTTP client builder providing proxy support and TLS certificate fallback for the LibreFang project.

## Overview

This crate centralizes the construction of `reqwest::Client` instances so that every component in the LibreFang workspace makes outbound HTTP requests with consistent TLS configuration and proxy handling. Rather than each crate independently configuring its own HTTP client, they all delegate to this single library.

## Why This Exists

Network environments vary wildly. Some systems have well-maintained system certificate stores; others do not. Some networks require an HTTP proxy; others don't. If each crate built its own `reqwest::Client`, TLS and proxy logic would be duplicated and could drift out of sync. `librefang-http` owns that responsibility once.

## TLS Certificate Strategy

The crate uses **rustls** as its TLS backend (not native-tls), with a two-tier certificate trust strategy:

1. **Native system certificates** (`rustls-native-certs`) — Loads the host operating system's trusted certificate store. This is the preferred path because it respects any custom CAs the user or administrator has installed.

2. **Bundled Mozilla certificates** (`webpki-roots`) — Falls back to the Mozilla root program bundled into the binary if native certificate loading fails or the system store is empty/missing.

This fallback ensures LibreFang works on minimal containers (like Alpine or scratch Docker images) that may not have `ca-certificates` installed, while still honoring system trust stores where available.

## Dependencies and Their Roles

| Dependency | Role |
|---|---|
| `reqwest` | HTTP client; configured via its builder API |
| `rustls` | TLS implementation; rustls is chosen over native-tls for cross-platform consistency |
| `webpki-roots` | Bundled Mozilla CA certificates for fallback trust |
| `rustls-native-certs` | Loader for the OS-native certificate store |
| `tracing` | Structured logging for TLS and proxy configuration events |
| `librefang-types` | Shared type definitions used across the workspace |

## Relationship to Other Crates

```
librefang-types
      ▲
      │
librefang-http
      │
      │  used by
      ▼
  (other LibreFang crates that need HTTP)
```

This crate sits between `librefang-types` (which it depends on for shared types) and whichever LibreFang components need to make outbound HTTP calls. It is a leaf dependency in terms of outgoing calls — it does not call into other LibreFang crates at runtime, but it is called *by* them to obtain a configured `reqwest::Client`.

## Design Notes

**No detected internal call graph.** This module is primarily configuration and builder logic. It constructs a `reqwest::Client` and returns it. There are no deep internal call hierarchies, state machines, or async runtimes managed here. The complexity lies in the TLS fallback chain and proxy environment detection, both of which happen at client construction time.

**Logging.** The `tracing` dependency is used to emit diagnostic events — for example, logging whether native certs were found, how many were loaded, whether webpki-roots were used as fallback, and what proxy settings were detected. This is critical for debugging connectivity issues in the field.

## Usage Pattern (Conceptual)

Other crates in the workspace use this library to obtain a client rather than calling `reqwest::Client::builder()` directly:

```rust
// Inside some other LibreFang crate
let client = librefang_http::build_client()?;
let response = client.get("https://example.com").send().await?;
```

The exact API surface (function names, builder options) is defined in this crate's `src/` — consult the Rustdoc for the specific entry points.