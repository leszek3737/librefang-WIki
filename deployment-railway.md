# Deployment — railway

# Railway Deployment Module

## Overview

This module provides Railway deployment configuration for the application. Railway is a modern deployment platform that supports Docker-based deployments with built-in health checks and restart policies.

The module contains two equivalent configuration files:
- `railway.json` — JSON Schema format
- `railway.toml` — TOML format

Both files express the same configuration; Railway accepts either format.

## Configuration

### Build Configuration

```json
"build": {
  "dockerfilePath": "./deploy/Dockerfile"
}
```

| Field | Value | Description |
|-------|-------|-------------|
| `dockerfilePath` | `./deploy/Dockerfile` | Relative path from project root to the Dockerfile used for building the container image |

### Deploy Configuration

```json
"deploy": {
  "healthcheckPath": "/api/health",
  "restartPolicyType": "ON_FAILURE"
}
```

| Field | Value | Description |
|-------|-------|-------------|
| `healthcheckPath` | `/api/health` | HTTP endpoint Railway uses to verify the container is healthy. The application must respond with a 2xx status code. |
| `restartPolicyType` | `ON_FAILURE` | Container restarts only when the process exits with a non-zero status code. Does not restart on successful completion. |

## Deployment Pipeline

Railway reads this configuration and executes the following workflow:

1. **Build Phase** — Railway invokes Docker to build the image using the specified Dockerfile
2. **Health Check** — After container starts, Railway polls `/api/health` to confirm readiness
3. **Runtime** — Container runs continuously, restarting automatically on failure per the restart policy

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐
│   Build     │────▶│   Start     │────▶│  Health Check Loop   │
│  (Docker)   │     │  Container  │     │  GET /api/health     │
└─────────────┘     └─────────────┘     └─────────────────────┘
                                                    │
                    ┌───────────────────────────────┘
                    │  Container exits with code ≠ 0
                    ▼
              ┌───────────┐
              │  Restart  │
              │  Container│
              └───────────┘
```

## Prerequisites

For this deployment to succeed, the application must expose:

- **`/api/health`** — A `GET` endpoint that returns HTTP 200 when the service is ready to accept traffic

The Dockerfile at `./deploy/Dockerfile` must exist and produce a working container image.

## Relationship to Other Modules

This module does not call or depend on application code at runtime. It is purely a platform configuration consumed by Railway's deployment system. The Dockerfile it references will typically bundle the compiled application and define the startup command.

## Usage

Connect your GitHub repository to a Railway project. Railway automatically detects the `railway.json` or `railway.toml` file and applies the configuration during deployment.