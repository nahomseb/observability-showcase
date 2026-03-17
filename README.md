# Observability Stack

Production-tested monitoring, logging, and security configurations for containerized infrastructure. Runs Grafana, Prometheus, Loki, Promtail, and OpenTelemetry Collector with security hardening for nginx.

## Architecture

```
                    ┌─────────────────────────────────────────┐
                    │              Grafana :3000               │
                    │    dashboards, alerting, log exploration │
                    └────────┬──────────────────┬─────────────┘
                             │                  │
                    ┌────────┴────────┐ ┌───────┴────────┐
                    │  Prometheus     │ │     Loki       │
                    │  :9090         │ │     :3100      │
                    │  metrics store │ │   log store    │
                    └────────┬───────┘ └───────┬────────┘
                             │                  │
              ┌──────────────┼──────────┐       │
              │              │          │       │
        ┌─────┴─────┐ ┌─────┴────┐ ┌───┴───┐ ┌┴────────┐
        │ node-exp  │ │ cAdvisor │ │ OTel  │ │Promtail │
        │ host CPU  │ │container │ │traces │ │log ship │
        │ mem, disk │ │ metrics  │ │       │ │         │
        └───────────┘ └──────────┘ └───────┘ └─────────┘
```

## Quick Start

```bash
cp .env.example .env
# Edit .env — set GRAFANA_PASSWORD

docker compose up -d

# Grafana: http://localhost:3000
# Prometheus: http://localhost:9090
```

## What's Included

### Core Stack

| Component | Config | Purpose |
|-----------|--------|---------|
| **Prometheus** | `prometheus/prometheus.yml` | Metrics scraping and time-series storage |
| **Loki** | `loki/loki.yml` | Log aggregation with 7-day retention |
| **Promtail** | `loki/promtail.yml` | Log shipping from containers + system logs |
| **Grafana** | `grafana/` | Dashboards, alerting, datasource provisioning |

### Optional Components

| Component | Config | Purpose |
|-----------|--------|---------|
| **OpenTelemetry Collector** | `otel/collector.yml` | Distributed tracing (OTLP gRPC/HTTP) |
| **Node Exporter** | (in docker-compose) | Host-level CPU, memory, disk, network metrics |
| **cAdvisor** | (in docker-compose) | Per-container resource usage tracking |

### Security Hardening

| Config | Purpose |
|--------|---------|
| `nginx/default-security.conf` | Catch-all server block — drops unmatched requests with 444 |
| `nginx/security-locations.conf` | Blocks .env/.git probes, WordPress scanners, SQL injection attempts, phpMyAdmin crawlers |
| `fail2ban/nginx-jail.conf` | Auto-bans repeated bot scans (24h) and auth failures (1h) |

### Tools

| Script | Purpose |
|--------|---------|
| `scripts/add-loki-to-service.py` | Automatically add Loki logging driver to any docker-compose.yml |

### Dashboards

| Dashboard | Purpose |
|-----------|---------|
| `grafana/dashboards/security-command-center.json` | Security overview — probe counts, auth failures, bot activity, ban status |

## Adding Your Services

### 1. Add Prometheus scrape targets

Edit `prometheus/prometheus.yml` — uncomment and adapt the example targets:

```yaml
scrape_configs:
  - job_name: "my-api"
    static_configs:
      - targets: ["my-api:8000"]
    metrics_path: "/metrics"
```

### 2. Ship logs to Loki

**Option A** — Docker logging driver (per-service):
```yaml
# In your service's docker-compose.yml
services:
  my-api:
    logging:
      driver: loki
      options:
        loki-url: "http://localhost:3100/loki/api/v1/push"
        loki-external-labels: "service=my-api,environment=prod"
```

**Option B** — Use the automation script:
```bash
python3 scripts/add-loki-to-service.py docker-compose.yml my-api:production:api
```

**Option C** — Promtail file scraping (add to `loki/promtail.yml`):
```yaml
scrape_configs:
  - job_name: my-app
    static_configs:
      - targets: [localhost]
        labels:
          job: my-app
          __path__: /var/log/my-app/*.log
```

### 3. Import dashboards

Drop JSON files into `grafana/dashboards/` — Grafana auto-loads them on restart.

## Structured Logging Standard

All services should emit JSON logs with these fields:

```json
{
  "level": "info",
  "service": "my-api",
  "message": "Request processed",
  "timestamp": "2026-03-03T12:00:00Z",
  "duration_ms": 42
}
```

Required: `level`, `service`, `message`, `timestamp`

## Configuration Notes

- **All ports bind to 127.0.0.1** — not exposed to the internet. Use a reverse proxy for external access.
- **Loki retention**: 7 days (configurable in `loki/loki.yml` → `retention_period`)
- **Prometheus retention**: 15 days (configurable in docker-compose → `--storage.tsdb.retention.time`)
- **Grafana passwords**: Set via environment variables, never hardcoded
- **Fail2ban bans**: 24h for bot scanners, 1h for auth failures

## Project Structure

```
docker-compose.yml                    # Full stack (Prometheus + Loki + Grafana + optional exporters)
.env.example                          # Environment template
prometheus/
  prometheus.yml                      # Scrape config with commented examples
loki/
  loki.yml                            # Loki server config (retention, limits, compaction)
  promtail.yml                        # Log shipping (Docker, syslog, nginx)
grafana/
  provisioning/
    prometheus.yml                    # Auto-provision Prometheus datasource
    loki.yml                          # Auto-provision Loki datasource
  dashboards/
    security-command-center.json      # Security monitoring dashboard
otel/
  collector.yml                       # OpenTelemetry Collector (gRPC + HTTP → Prometheus)
nginx/
  default-security.conf               # Catch-all server block (drop unknown hosts)
  security-locations.conf             # Block attack paths (.env, wp-*, SQL injection)
fail2ban/
  nginx-jail.conf                     # Auto-ban bot scanners and auth failures
scripts/
  add-loki-to-service.py              # Add Loki logging driver to any docker-compose.yml
```

## Tech Stack

Grafana, Prometheus, Loki, Promtail, OpenTelemetry, Fail2ban, Nginx, Docker Compose

---

Built by [D. Michael Piscitelli](https://github.com/herakles-dev) | [herakles.dev](https://herakles.dev)
