# API Server

# API Server

The API Server module group provides the HTTP and WebSocket interface for the LibreFang Agent OS daemon, including interactive documentation and a formal API specification.

## Components

| Module | Purpose |
|--------|---------|
| [librefang-api](librefang-api.md) | HTTP and WebSocket API server built on Axum |
| [api-docs](api-docs.md) | Interactive Swagger UI documentation portal |
| [openapi.json](openapi.json.md) | OpenAPI 3.1.0 specification with 130+ endpoints |

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     API Server                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────┐    serves    ┌─────────────────────┐  │
│  │  api-docs   │──────────────│   Swagger UI        │  │
│  └─────────────┘              └─────────────────────┘  │
│         │                              ▲               │
│         │ loads                        │ fetches        │
│         ▼                              │               │
│  ┌─────────────┐                       │               │
│  │ openapi.json│───────────────────────────────────────┘
│  └─────────────┘
│         ▲
│         │ defines
│         │
│  ┌─────────────┐    implements     ┌─────────────────┐
│  │ librefang-api│─────────────────│  REST + WebSocket │
│  └─────────────┘                   │   Endpoints      │
│                                    └─────────────────┘
└─────────────────────────────────────────────────────────┘
```

The three modules form a layered stack:

1. **[openapi.json](openapi.json.md)** — The canonical contract defining all endpoints, request/response schemas, and authentication requirements
2. **[librefang-api](librefang-api.md)** — The runtime implementation that fulfills the contract, handling HTTP/WebSocket traffic and business logic
3. **[api-docs](api-docs.md)** — The developer-facing interface that renders the specification as interactive documentation

## Key Cross-Module Workflows

### Authentication Flow (Terminal WebSocket)

Terminal sessions demonstrate how the server coordinates authentication across multiple systems:

```
terminal_ws → validate_api_tokens → dashboard_session_token 
→ resolve_dashboard_credential → vault.unlock 
→ vault.resolve_master_key → vault.load_keyring_key
```

This chain verifies API tokens, resolves session credentials, and unlocks secure storage for the terminal session.

### Webhook Delivery

Webhook registration and delivery flows through the server:

```
create_webhook → validate_url → persist_webhook 
→ trigger_event → deliver_to_webhook_endpoint
```

Webhooks are validated for security (rejecting private IPs), stored with configurable event filters, and dispatched asynchronously.

## API Surface

The combined API provides:

- **Agent lifecycle management** — create, configure, start, stop agents
- **Chat interaction** — message exchange with streaming responses
- **Session management** — persistent conversation contexts
- **Memory operations** — knowledge storage and retrieval
- **Tool/skill registration** — extend agent capabilities
- **OAuth integration** — third-party authentication
- **OpenAI-compatible endpoints** — drop-in replacement support
- **Webhook system** — event-driven integrations
- **Dashboard SPA** — browser-based control interface

## Relationship to Other Modules

The API server acts as the gateway for all external clients. Internally, it coordinates with:

- **[librefang-extensions/vault](librefang-extensions.md#vault)** — secure credential and key storage
- **Kernel modules** — agent execution, memory, and tool registry
- **Authentication providers** — OAuth flows and API key management

See individual sub-module documentation for implementation details.