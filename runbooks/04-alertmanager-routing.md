# 04 — Alertmanager routing to Discord/email

**Goal:** firing alerts delivered as useful, de-duplicated notifications — grouped, throttled, and inhibited so the channel stays trustworthy.

## How it's wired

Prometheus decides *what* is wrong and hands firing alerts to Alertmanager, which decides *who hears about it and how often*. [`alertmanager/alertmanager.yml`](../alertmanager/alertmanager.yml) does the grouping/routing; the `alertmanager-discord` bridge container reformats Alertmanager's webhook into a Discord message.

## Steps

1. **Set the Discord webhook.** Create a webhook in your Discord channel, put the URL in `compose/.env` as `DISCORD_WEBHOOK_URL`, and recreate the bridge:
   ```bash
   docker compose up -d alertmanager-discord
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

- **No Discord message.** The webhook URL is wrong or the bridge didn't get it — `docker logs alertmanager-discord`; re-check `.env` and recreate the container.
- **Alert reaches Alertmanager but not Discord.** The receiver `url` must point at the bridge (`http://alertmanager-discord:9094`), not directly at Discord (Alertmanager's payload isn't Discord's format — that's what the bridge is for).
- **Getting spammed.** `repeat_interval` too short or grouping too granular — widen `group_by`/intervals.

## What I learned

*Filled in after I complete this milestone.*

<!-- Prompts to answer once done:
     - Why can't Alertmanager post to Discord directly — what does the bridge translate?
     - Which knob (grouping, repeat_interval, inhibition) mattered most for keeping the channel usable?
     - Did send_resolved change how I'd react to an alert? -->
