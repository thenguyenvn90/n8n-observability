# n8n-observability

A complete, production-ready observability stack for **n8n (queue mode)** using **Traefik, Prometheus, Grafana, cAdvisor, and exporters**.

> **What you get**
>
> - Reverse proxy & HTTPS: **Traefik** (auto TLS via Let’s Encrypt)
> - Data services: **PostgreSQL**, **Redis**
> - Metrics: **Prometheus**, **cAdvisor**, **Postgres/Redis exporters**, n8n + Traefik metrics
> - Dashboards & Alerts: **Grafana** (pre-provisioned datasource & folders)
> - n8n in **queue mode**: `main` + `worker` + **external task runner**

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Folder Layout](#folder-layout)
3. [.env Configuration](#env-configuration)
4. [DNS Records](#dns-records)
5. [Traefik Basic Auth (dashboards)](#traefik-basic-auth-dashboards)
6. [Compose Files & Configs](#compose-files--configs)
7. [Bring the Stack Up](#bring-the-stack-up)
8. [Health Checks & Sanity Testing](#health-checks--sanity-testing)
9. [Accessing Dashboards](#accessing-dashboards)
10. [Dashboards to Import](#dashboards-to-import)
11. [Alerts to Set Up](#alerts-to-set-up)
12. [Maintenance & Upgrades](#maintenance--upgrades)
13. [Backups](#backups)
14. [Security Tips](#security-tips)
15. [Troubleshooting](#troubleshooting)
16. [FAQ](#faq)
---

## Prerequisites

- A Linux server (Ubuntu 22.04+ or similar) with ports **80** and **443** open to the internet.
- **Docker** and **Docker Compose** installed.
  ```bash
  # Docker (Ubuntu example)
  curl -fsSL https://get.docker.com | sh
  sudo usermod -aG docker $USER
  newgrp docker

  # Compose plugin (on recent Docker this is already included)
  docker compose version
  ```
- A domain you control, e.g. `domain.com`.

---

## Folder Layout

Create the folders:

```bash
mkdir -p ~/n8n-observability/monitoring/grafana/provisioning/datasources
mkdir -p ~/n8n-observability/secrets
cd ~/n8n-observability

```

Your tree will look like:

```
n8n-observability/
├─ docker-compose.yml
├─ .env
├─ secrets/
│  └─ htpasswd
└─ monitoring/
   ├─ prometheus.yml
   └─ grafana/
      └─ provisioning/
         └─ datasources/
            └─ datasource.yml
```

---

### Generate strong secrets

```bash
# 16 random bytes, base64 (for STRONG_PASSWORD / RUNNERS_AUTH_TOKEN)
openssl rand -base64 16

# 32 random bytes, base64 (for N8N_ENCRYPTION_KEY)
openssl rand -base64 32
```

---

## DNS Records

Create **A/AAAA** records pointing to your server IP:

- `n8n.${DOMAIN}`
- `grafana.${DOMAIN}`
- `prometheus.${DOMAIN}` (optional; only if you plan to expose Prometheus UI)

---

## Traefik Basic Auth for Dashboards

Create a protected user for Grafana/Prometheus routes:

```bash
# Requires: apt install -y apache2-utils (or use docker httpd)
mkdir -p secrets
htpasswd -nbB admin 'YourSuperSecret' > ./secrets/htpasswd
```

> This file is mounted into Traefik and used by the Basic Auth middleware.

---

## Bring the Stack Up

```bash


docker compose pull          # fetch latest images

# Manual create volume
for v in n8n-data postgres-data redis-data letsencrypt; do docker volume create "$v"; done
docker volume ls | grep -E 'n8n-data|postgres-data|redis-data|letsencrypt'

# Start everything (Traefik, Postgres, Redis, n8n-main, 1 worker)
docker compose up -d

docker compose ps            # check containers are healthy
```

If a container restarts repeatedly, inspect logs:

```bash
docker compose logs -f traefik prometheus grafana postgres-exporter redis-exporter cadvisor n8n-main n8n-worker
```
---

## Health Checks & Sanity Testing

### Quick CLI checks
```bash
# n8n metrics reachable from Prometheus network?
docker compose exec prometheus wget -qO- http://n8n-main:5678/metrics | head

# Traefik metrics reachable?
docker compose exec prometheus wget -qO- http://traefik:8081/metrics | head

# Prometheus targets count (should be >= the jobs you configured)
docker compose exec prometheus wget -qO- http://prometheus:9090/api/v1/targets | jq '.data.activeTargets | length'

```

### Prometheus UI (optional)
Open `https://prometheus.${DOMAIN}` → **Status → Targets**. All targets should be **UP**:
- `n8n-main:5678`, `traefik:8081`, `postgres-exporter:9187`, `redis-exporter:9121`, `cadvisor:8080`

### n8n health
- `n8n-main` has `/healthz` and `/healthz/readiness` internally (Traefik routes your public traffic to the app).

> In **queue mode**, the **worker** does not expose `/metrics`—scrape the **main** service.

---

## Accessing Dashboards

- **Grafana**: `https://grafana.${DOMAIN}`  
  - First prompt: **Traefik basic auth** (user/password from secrets/htpasswd).
  - Then Grafana login: admin / ${GRAFANA_ADMIN_PASSWORD}.
- **Prometheus** (optional): `https://prometheus.${DOMAIN}`  
  - Disabled by default (set EXPOSE_PROMETHEUS=true to expose; still behind Basic Auth)
---

## Dashboards to Import

In Grafana → **Dashboards → Import**, search by ID (from grafana.com) or JSON export files.

Recommended:
- **Traefik v2 dashboard**
- **Postgres exporter dashboard**
- **Redis exporter dashboard**
- **Node.js / Process dashboard** (good fit for n8n’s Node runtime)
- **cAdvisor / Container overview** (any popular container-level dashboard)

> Filter panels by labels like compose_service="main" or compose_service="worker".

---

## Alerts to Set Up

Use **Grafana Alerting** (recommended) or Prometheus rules. Example queries:

- **n8n execution failures (rate over 5m)**
  ```promql
  rate(n8n_execution_failed_total[5m]) > 0
  ```

- **Queue backlog (BullMQ waiting)**
  ```promql
  max(n8n_queue_bull_queue_waiting) > 100
  ```

- **Traefik 5xx error rate > 1%**
  ```promql
  sum(rate(traefik_service_requests_total{code=~"5.."}[5m]))
  / sum(rate(traefik_service_requests_total[5m])) > 0.01
  ```

- **Redis evictions > 0**
  ```promql
  rate(redis_evicted_keys_total[5m]) > 0
  ```

- **Postgres connections > 80% of max** (if exposed)
  ```promql
  sum(pg_stat_activity_count) / max(pg_settings_max_connections) > 0.8
  ```

> Start simple; refine thresholds as you observe normal behavior.

---

## Maintenance & Upgrades

- **Update images**:
  ```bash
  docker compose pull
  docker compose up -d
  ```
- **Rotate secrets**: change in `.env`, then `docker compose up -d`.
- **Scale workers**:
  ```bash
  docker compose up -d --scale n8n-worker=3
  ```

---

## Backups

Prioritize these volumes:
- `n8n-data` (n8n credentials/settings)
- `postgres-data` (workflow executions, etc.)
- `letsencrypt` (TLS certs)
- *(optional)* `grafana-data` (dashboards/users)

Example backup (basic tar of a Docker named volume on Linux):
```bash
# Stop services that write to the volume before backup for consistency
docker run --rm -v postgres-data:/data -v $PWD:/backup busybox \
  tar czf /backup/postgres-data_$(date +%F).tgz /data

docker run --rm -v n8n-data:/data -v $PWD:/backup busybox \
  tar czf /backup/n8n-data_$(date +%F).tgz /data

docker run --rm -v letsencrypt:/data -v $PWD:/backup busybox \
  tar czf /backup/letsencrypt_$(date +%F).tgz /data
```

---

## Security Tips

- Strong, unique passwords; rotate regularly.
- Keep Prometheus internal unless you need the UI (EXPOSE_PROMETHEUS=false).
- Limit access to Grafana/Prometheus with basic auth and IP allowlists if possible.
- Keep **Traefik dashboard disabled** on the public internet (you have `--api.dashboard=false`).
- Keep Docker, images, and the host OS up to date.
- Consider enabling **SSO** for Grafana if available in your environment.

---

## Troubleshooting

**Let’s Encrypt certificate isn’t issued**
- Ensure DNS records point to your server IP.
- Port **80/443** must be reachable.
- Check Traefik logs:
  ```bash
  docker compose logs -f traefik
  ```

**Grafana shows “No data”**
- Check Prometheus targets:
  - `https://prometheus.${DOMAIN}` → *Status → Targets*
- Check that `prometheus.yml` matches service names and ports.

**Redis exporter DOWN**
- Verify password matches `STRONG_PASSWORD` in `.env`.

**Postgres exporter DOWN**
- Ensure `DATA_SOURCE_NAME` matches your DB name/user/password.

**n8n main is up but workers idle**
- Check Redis connectivity (password/host).
- Ensure `EXECUTIONS_MODE=queue` and worker command has proper `--concurrency`.

---
