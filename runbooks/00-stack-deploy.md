# 00 — Deploy the stack (Prometheus, Grafana, Alertmanager)

**Goal:** the three services up with their config mounted from this repo, and each web UI reachable.

## Prereqs

- Docker + Compose on the homelab host.
- A Discord webhook URL if you want notifications now (can come later in [runbook 04](04-alertmanager-routing.md)).

## Steps

1. **Configure:**
   ```bash
   cd compose
   cp .env.example .env
   # edit .env — set GF_ADMIN_PASSWORD
   # (optional) Discord alerting: echo your webhook URL into ../alertmanager/discord.secret (see runbook 04)
   ```

2. **Bring it up:**
   ```bash
   docker compose up -d
   docker compose ps
   ```
   Expected: `prometheus`, `alertmanager`, `grafana`, and `node-exporter` all running.

3. **Reach the UIs:**
   - Prometheus — `http://<host>:9090` (Status → Targets shows scrape health)
   - Alertmanager — `http://<host>:9093`
   - Grafana — `http://<host>:3000` (log in with the admin creds from `.env`)

4. **Confirm config loaded from the repo.** The compose file mounts `prometheus.yml`, `alert.rules.yml`, `alertmanager.yml`, and the Grafana provisioning dirs read-only — so the stack's behaviour is defined by files in git, not by clicking in the UIs.

## Verify

```bash
docker compose ps                                       # all services up
curl -s http://localhost:9090/-/ready                   # "Prometheus is Ready."
curl -s http://localhost:9093/-/ready                   # "OK"
curl -s http://localhost:3000/api/health | grep -q ok && echo "grafana healthy"
```
In Prometheus, **Status → Rule Health** should list the alert rules loaded from `alert.rules.yml`.

## If it breaks

- **Prometheus container restarts.** Almost always a YAML error in `prometheus.yml` or `alert.rules.yml` — check `docker logs prometheus`; it names the line.
- **Grafana has no datasource.** The provisioning mount path is wrong — confirm `../grafana/provisioning` is mounted to `/etc/grafana/provisioning`.
- **node-exporter shows odd/host-less metrics.** It needs `--path.rootfs=/host` and the `/:/host:ro` mount (both in the compose file) to read the host, not the container.

## What I learned

*Filled in after I complete this milestone.*

<!-- Prompts to answer once done:
     - What does mounting config read-only from the repo buy me over configuring in the UI?
     - Which UI do I actually reach for — Prometheus, Grafana, or Alertmanager — and for what?
     - What did Status → Targets tell me on the first run? -->
