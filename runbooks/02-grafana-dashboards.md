# 02 — Grafana dashboards as code

**Goal:** dashboards and the datasource provisioned from files in this repo, so a fresh Grafana comes up already configured.

## Steps

1. **The datasource is already provisioned.** [`grafana/provisioning/datasources/prometheus.yml`](../grafana/provisioning/datasources/prometheus.yml) points Grafana at `http://prometheus:9090` at startup. Confirm in Grafana → *Connections → Data sources* — Prometheus is there and `editable: false`.

2. **The dashboard provider is set up.** [`grafana/provisioning/dashboards/dashboards.yml`](../grafana/provisioning/dashboards/dashboards.yml) tells Grafana to load every JSON in `/var/lib/grafana/dashboards` into a "Homelab" folder. The repo ships [`node-overview.json`](../grafana/dashboards/node-overview.json).

3. **Open the dashboard.** Grafana → *Dashboards → Homelab → Node Overview*. It has an `instance` variable (multi-select) and panels for instances-up, CPU busy %, memory used %, root FS free %, and network receive.

4. **Edit as code, not just in the UI.** You *can* tweak a panel in the UI, but the durable workflow is: edit the JSON in the repo → it reloads within `updateIntervalSeconds`. To capture a UI change, use *Dashboard settings → JSON Model*, copy it back into the repo file, and commit — so the repo stays the source of truth.

5. **Add more dashboards** by dropping another JSON in `grafana/dashboards/` (import a community dashboard, save its JSON here). No clicking required on the next fresh deploy.

## Verify

- The "Node Overview" dashboard renders live data for each instance.
- Delete the Grafana container and recreate it (`docker compose up -d --force-recreate grafana`); the datasource and dashboard come back automatically — proof they're provisioned, not hand-made.

## If it breaks

- **Dashboard folder empty.** The dashboards mount or `path` is wrong — confirm `grafana/dashboards` is mounted to `/var/lib/grafana/dashboards`.
- **Panels say "datasource not found".** The dashboard references a datasource UID that doesn't exist. Note a `${DS_PROMETHEUS}`-style placeholder is only substituted by Grafana's **import wizard** — provisioned dashboards are loaded from disk as-is, with no import step, so the placeholder is never resolved. Give the provisioned datasource a fixed `uid` (here `uid: prometheus` in `provisioning/datasources/prometheus.yml`) and reference that same uid literally in the dashboard JSON.
- **Edits vanish on restart.** That's provisioning working as designed — the file is truth. Save UI edits back to the JSON.
