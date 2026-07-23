# 01 — node_exporter targets

**Goal:** node_exporter running on each host you want to watch, and Prometheus scraping all of them with `up == 1`.

## Steps

1. **The local exporter is already up** (it's in the compose stack). Confirm Prometheus sees it: `http://<host>:9090` → **Status → Targets** → the `node` job's `node-exporter:9100` target is **UP**.

2. **Deploy node_exporter to another host** (e.g. DC01's Linux counterpart, the Ubuntu box, a Pi). The clean way on a systemd host:
   ```bash
   # On the target host:
   useradd -rs /bin/false node_exporter 2>/dev/null || true
   # download the release binary to /usr/local/bin/node_exporter, then:
   cat </dev/null | sudo tee /etc/systemd/system/node_exporter.service >/dev/null
   sudo tee /etc/systemd/system/node_exporter.service >/dev/null <<'UNIT'
   [Unit]
   Description=Node Exporter
   After=network-online.target
   [Service]
   User=node_exporter
   ExecStart=/usr/local/bin/node_exporter
   [Install]
   WantedBy=multi-user.target
   UNIT
   sudo systemctl daemon-reload && sudo systemctl enable --now node_exporter
   ```
   (For a Windows host like DC01, use **windows_exporter** instead — same idea, exposes `:9182`.)

3. **Add the target** to [`prometheus/prometheus.yml`](../prometheus/prometheus.yml) under the `node` job's `targets:` list (e.g. `'10.0.10.20:9100'`), then reload Prometheus:
   ```bash
   curl -X POST http://localhost:9090/-/reload    # requires --web.enable-lifecycle, or just restart the container
   ```

4. **Open firewall** on the target for the exporter's port from the Prometheus host only (don't expose 9100 to the world).

## Verify

- **Status → Targets** shows every target **UP**.
- A quick PromQL check in the Prometheus UI: `up{job="node"}` returns `1` for each instance; `count(up{job="node"} == 1)` equals your host count.

## If it breaks

- **Target shows DOWN, "connection refused".** The exporter isn't running or the port is firewalled — `curl http://<target>:9100/metrics` from the Prometheus host.
- **Target DOWN, "context deadline exceeded".** Network/routing between Prometheus and the target — check the address and that they're on reachable networks.
- **Metrics look like the container, not the host.** That's the *local* exporter missing the rootfs mount — fixed in the compose file; remote binary exporters read their own host natively.

## What I learned

*Filled in after I complete this milestone.*

<!-- Prompts to answer once done:
     - Why is a DOWN target visible at all — where does the `up` metric come from?
     - windows_exporter vs node_exporter — what differed getting DC01 in?
     - Did I remember to restrict the exporter port to only the Prometheus host? -->
