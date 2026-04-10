# Deployment — prometheus

# Prometheus Deployment Module

## Overview

This module provides the Prometheus configuration for monitoring the librefang service. It defines a scrape job that periodically collects metrics from librefang's `/api/metrics` endpoint.

## Configuration Structure

```yaml
global:
  scrape_interval: 15s      # How often to scrape targets
  evaluation_interval: 15s   # How often to evaluate rules

scrape_configs:
  - job_name: "librefang"    # Unique identifier for this scrape job
    metrics_path: /api/metrics
    static_configs:
      - targets: ["host.docker.internal:4545"]
        labels:
          instance: "librefang-local"
```

## Configuration Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| `scrape_interval` | 15s | Frequency at which Prometheus collects metrics from targets |
| `evaluation_interval` | 15s | Frequency of rule/alert evaluation cycles |
| `job_name` | "librefang" | Logical name for the scraped targets |
| `metrics_path` | `/api/metrics` | HTTP endpoint on the target service exposing Prometheus-formatted metrics |
| `targets` | `host.docker.internal:4545` | Target address for metrics collection |
| `instance` label | "librefang-local" | Distinguishes this local deployment from other environments |

## Target Resolution

The target `host.docker.internal:4545` uses Docker's special DNS name to reach the host machine from within a container. This allows the Prometheus container to scrape a service running directly on the host machine (librefang) rather than within Docker's network.

**Prerequisites:**
- Prometheus must run inside a Docker container
- The host machine must have a service listening on port 4545
- Docker must support `host.docker.internal` (available in Docker Desktop and Docker Engine 20.03+)

## Integration with Librefang

The librefang service must expose a `/api/metrics` endpoint that returns metrics in Prometheus text format:

```
# HELP <metric_name> <description>
# TYPE <metric_name> <type>
<metric_name>{<labels>} <value>
```

Typical metrics collected include:
- HTTP request counts and latencies
- Database query statistics
- Cache hit/miss ratios
- Business logic counters

## Usage

This configuration file is typically mounted into the Prometheus container:

```bash
docker run -d \
  --name prometheus \
  -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

Alternatively, include it in a Docker Compose `volumes` section:

```yaml
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./deploy/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
```

## Extending the Configuration

To monitor additional services, add new entries under `scrape_configs`. For example:

```yaml
scrape_configs:
  # ... existing librefang config ...

  - job_name: "additional-service"
    metrics_path: /metrics
    static_configs:
      - targets: ["additional-service:8080"]
```