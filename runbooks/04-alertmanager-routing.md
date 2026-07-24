# 04 — Alertmanager routing to Discord/email

**Goal:** firing alerts delivered as useful, de-duplicated notifications — grouped, throttled, and inhibited so the channel stays trustworthy.

## How it's wired

Prometheus decides *what* is wrong and hands firing alerts to Alertmanager, which decides *who hears about it and how often*. [`alertmanager/alertmanager.yml`](../alertmanager/alertmanager.yml) does the grouping/routing and posts to Discord. Alertmanager **v0.28+ speaks Discord natively** (`discord_configs`), so no bridge/translator container is needed — earlier versions (pre-0.25) couldn't format Discord's payload and relied on a sidecar like `alertmanager-discord`; that's now obsolete and this lab pins v0.28.1.

## Steps

1. **Set the Discord webhook.** Create a webhook in your Discord channel and write the URL to a gitignored secret file the Alertmanager container mounts (the config references it via `webhook_url_file`, so no secret is committed or passed as an env var):
   ```bash
   echo 'https://discord.com/api/webhooks/xxxx/yyyy' > alertmanager/discord.secret
   docker compose up -d --force-recreate alertmanager
   ```

2. **Understand the routing** in `alertmanager.yml`:
   - `group_by: [alertname, instance]` — ten disk alerts on one host arrive as one grouped message.
   - `group_wait` / `group_interval` / `repeat_interval` — how long to batch, and how often to re-notify a still-firing alert (criticals nag hourly; the rest every 4h).
   - **inhibition** — if a `critical` is firing for a host, matching `warning`s for the same host are suppressed (no double noise).
   - `send_resolved: true` — you also get a message when the alert clears.

3. **Send a test alert** straight into Alertmanager to prove delivery without waiting for a real problem:
   ```bash
   curl -s -XPOST http://localhost:9093/api/v2/alerts -H 'Content-Type: application/json' -d '[
     {"labels":{"alertname":"TestPing","severity":"warning","instance":"lab-test"},
      "annotations":{"summary":"Test alert from runbook 04"}}
   ]'
   ```
   Expected: a message appears in the Discord channel within `group_wait`.

4. **(Optional) email.** Uncomment the `email_configs` receiver in `alertmanager.yml`, fill in the SMTP details, and point a route at it.

## Verify

- The test alert above lands in Discord.
- Alertmanager UI (`http://<host>:9093`) shows the alert under *Alerts*, and *Status* shows the config loaded.
- Firing then resolving an alert produces both a firing and a resolved message.

## If it breaks

- **No Discord message.** The webhook URL in `alertmanager/discord.secret` is wrong/empty, or the file isn't mounted — `docker logs alertmanager` and re-check the secret file, then `docker compose up -d --force-recreate alertmanager`.
- **Config won't load / "no discord webhook URL provided".** `webhook_url_file` needs Alertmanager **v0.28+** (this lab pins v0.28.1) and the mounted file must contain a valid webhook URL. Validate with `docker compose exec alertmanager amtool check-config /etc/alertmanager/alertmanager.yml`.
- **Getting spammed.** `repeat_interval` too short or grouping too granular — widen `group_by`/intervals.

## What I learned

*Filled in after I complete this milestone.*

<!-- Prompts to answer once done:
     - Why keep the webhook in a mounted secret file (webhook_url_file) instead of the committed config or an env var?
     - Which knob (grouping, repeat_interval, inhibition) mattered most for keeping the channel usable?
     - Did send_resolved change how I'd react to an alert? -->
