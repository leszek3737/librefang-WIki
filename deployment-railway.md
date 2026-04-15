# Deployment — railway

# Deployment — Railway

## Overview

This module contains the Railway deployment configuration for the application. Railway uses these files to build a Docker image and manage the deployment lifecycle, including health monitoring and restart behavior.

The module provides the configuration in two formats — `railway.json` and `railway.toml` — both expressing identical settings. Railway supports either format; only one is required at runtime.

## Files

| File | Format | Purpose |
|------|--------|---------|
| `deploy/railway/railway.json` | JSON | Railway configuration conforming to the official `$schema`. Preferred for IDE validation and autocomplete. |
| `deploy/railway/railway.toml` | TOML | Equivalent configuration in TOML. Provided for compatibility or developer preference. |

## Configuration Reference

### Build

```json
{
  "build": {
    "dockerfilePath": "./deploy/Dockerfile"
  }
}
```

- **`dockerfilePath`**: Points to the Dockerfile at `./deploy/Dockerfile` (relative to the project root). Railway uses this to build the container image. The Dockerfile itself lives in the `deploy/` directory alongside this configuration.

### Deploy

```json
{
  "deploy": {
    "healthcheckPath": "/api/health",
    "restartPolicyType": "ON_FAILURE"
  }
}
```

| Setting | Value | Description |
|---------|-------|-------------|
| `healthcheckPath` | `/api/health` | Railway sends periodic HTTP GET requests to this endpoint. A successful response (2xx) confirms the application is ready to receive traffic. |
| `restartPolicyType` | `ON_FAILURE` | The container is restarted only when it exits with a non-zero status code (crash or error). Successful exits do not trigger a restart. |

## Health Check Contract

Railway expects the application to expose `GET /api/health` and return a 2xx status code once it is fully initialized and ready to serve requests. Until this endpoint responds successfully, Railway will not route traffic to the instance.

Make sure the application:
1. Starts listening on the configured port before the health check timeout elapses.
2. Returns a 2xx response from `/api/health` only when all critical dependencies (database, cache, etc.) are connected and ready.

## Relationship to the Rest of the Codebase

This module is a deployment artifact — it contains no executable code, no imports, and no runtime dependencies.

- **Dockerfile** (`deploy/Dockerfile`): Referenced by both config files. Defines the image build steps, including copying source, installing dependencies, and specifying the start command.
- **Application code**: Must implement the `/api/health` endpoint that Railway probes during deployment.

## Usage

### Deploy via Railway CLI

```bash
# Link the project (if not already linked)
railway link

# Deploy using the configuration in this directory
railway up
```

### Deploy via Railway Dashboard

1. Connect your Git repository to a Railway project.
2. Railway auto-detects `railway.json` or `railway.toml` in the `deploy/railway/` directory.
3. Configure the **Root Directory** setting in the Railway service to point to `deploy/railway/` so Railway finds the config file, or ensure the config file is at the repository root.

### Choosing Between Config Formats

Both files are maintained in sync. Use whichever fits your workflow:

- **`railway.json`** — enables schema validation in editors via the `$schema` field.
- **`railway.toml`** — familiar syntax for teams using TOML across other tooling.

Do **not** keep both files in the service root with conflicting values. Remove the one you are not using, or ensure they remain identical.