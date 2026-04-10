# Deployment — render.yaml

# Deployment Configuration — render.yaml

This file defines the deployment configuration for Librefang on [Render.com](https://render.com), a cloud platform that hosts the application's Docker container.

## Overview

The configuration deploys Librefang as a single Docker-based web service with the following characteristics:

| Property | Value |
|----------|-------|
| Runtime | Docker |
| Plan | Free tier |
| Health Check | `GET /api/health` |
| Dockerfile | `deploy/Dockerfile` |

## Service Configuration

```yaml
services:
  - type: web
    runtime: docker
    name: librefang
    dockerfilePath: deploy/Dockerfile
    plan: free
    healthCheckPath: /api/health
```

### Key Properties

- **`type: web`** — Marks this as an HTTP web service, making it publicly accessible via HTTPS.
- **`runtime: docker`** — Instructs Render to build and run a Docker container rather than using a platform-supported language runtime.
- **`dockerfilePath`** — Points to `deploy/Dockerfile` which contains the instructions for building the application image.
- **`healthCheckPath`** — Render periodically sends `GET /api/health` requests. The service must return a 2xx response to pass health checks. This endpoint is expected to be implemented in the application.
- **`plan: free`** — Runs on Render's free tier with [ephemeral disk storage](#storage-considerations).

## Environment Variables

```yaml
envVars:
  - key: GROQ_API_KEY
    sync: false
  - key: OPENAI_API_KEY
    sync: false
  - key: ANTHROPIC_API_KEY
    sync: false
```

All three API keys follow the same pattern:

| Property | Purpose |
|----------|---------|
| `key` | The environment variable name consumed by the application |
| `sync: false` | Prevents Render from syncing a value from the dashboard on each deploy. The actual key values are managed via the Render dashboard or `render-cli` and are not stored in this file. |

### Required Keys

| Variable | Purpose |
|----------|---------|
| `GROQ_API_KEY` | API key for Groq LLM inference |
| `OPENAI_API_KEY` | API key for OpenAI LLM inference |
| `ANTHROPIC_API_KEY` | API key for Anthropic LLM inference |

These keys must be set in the Render dashboard before the service will function correctly. The application expects these variables to be present at runtime.

## Storage Considerations

The free tier uses **ephemeral disk storage**, meaning:

- All files written to disk are lost when the container restarts or is redeployed
- Any local database files, cached data, or configuration written to disk will be reset
- The `/data` directory (if used) will not persist between deployments

### Enabling Persistent Storage

To use persistent storage, upgrade to a paid Render plan and add a persistent disk:

```yaml
services:
  - type: web
    # ... existing config ...
    disk:
      name: librefang-data
      mountPath: /data
      sizeGB: 1
    envVars:
      - key: LIBREFANG_HOME
        value: /data
```

This mounts a 1 GB persistent disk at `/data` and sets `LIBREFANG_HOME` so the application writes persistent data there instead of ephemeral storage.

## Relationship to Codebase

```
librefang/
├── deploy/
│   └── Dockerfile      ← Referenced by dockerfilePath
└── render.yaml         ← This file
```

The `render.yaml` references `deploy/Dockerfile`, which builds the application container. The health check endpoint `/api/health` must be implemented in the application's API routes to pass Render's verification.

## Deployment Process

1. Render detects changes to `render.yaml` or receives a manual deploy trigger
2. Render builds the Docker image using `deploy/Dockerfile`
3. The container starts and reads required environment variables
4. Render verifies the service responds to `GET /api/health`
5. The service becomes available at `https://librefang.onrender.com`