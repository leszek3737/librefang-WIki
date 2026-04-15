# Deployment

# Deployment

Everything needed to run and monitor LibreFang in production. This module provides multiple deployment targets, a complete observability stack, a one-click deployment portal, and a WhatsApp integration bridge.

## Deployment Targets

LibreFang supports several deployment paths, all building from the same multi-stage Dockerfile defined in [deploy/](deploy.md):

| Target | Module | Best for |
|--------|--------|----------|
| Docker Compose | [deploy/](deploy.md) | Self-hosted, full control |
| Fly.io | [fly/](fly.md) | Edge deployment, one-command setup |
| GCP Free Tier | [gcp/](gcp.md) | Zero-cost `e2-micro` instance via Terraform |
| Railway | [railway/](railway.md) | Managed platform with health checks |
| Render | [render.yaml](render-yaml.md) | Managed free-tier container hosting |
| Bare metal | [deploy/](deploy.md) | systemd unit (`librefang.service`) |

The [Cloudflare Worker](worker.md) at `deploy.librefang.ai` provides a browser-based deployment portal that links to all platform options and automates Fly.io provisioning via its `POST /api/deploy` endpoint.

## Observability Stack

Prometheus and Grafana work together to provide full-stack monitoring:

1. **[Prometheus](prometheus.md)** scrapes the LibreFang application and Ollama backend every 15 seconds.
2. **[Grafana](grafana.md)** auto-provisions four linked dashboards — system overview, LLM/token usage, HTTP/API metrics, and cost/budget tracking — all querying the Prometheus datasource.

Both are included in the `docker-compose.yml` stack from [deploy/](deploy.md).

## Integrations

The [WhatsApp Gateway](whatsapp-gateway.md) is a standalone Node.js process that bridges WhatsApp conversations to LibreFang agents. It uses the Baileys library for WhatsApp connectivity, SQLite for message persistence, and communicates with the agent platform via HTTP POST and SSE streaming. Deploy it alongside any of the deployment targets above.

## Quick Navigation

- **Deploy locally with Docker:** See [deploy/](deploy.md) for the compose stack.
- **One-click cloud deploy:** Visit the [Worker](worker.md) portal or run [fly/deploy.sh](fly.md) directly.
- **Free-tier cloud:** Use [gcp/](gcp.md) (Terraform) or [render.yaml](render-yaml.md).
- **Set up monitoring:** Deploy [Prometheus](prometheus.md) and [Grafana](grafana.md) together.
- **Connect WhatsApp:** Configure the [WhatsApp Gateway](whatsapp-gateway.md) with your LibreFang instance.