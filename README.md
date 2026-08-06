# Monitoring & Observability Lab

A monitoring stack for my homelab — **Prometheus** scraping **node_exporter** across hosts, **Grafana** dashboards provisioned as code, and **Alertmanager** routing disk/CPU/service alerts to Discord — built and documented the same way I'd document production monitoring. Includes a milestone that catches a real (self-induced) problem end to end.

## Overview

You can't operate what you can't see. Monitoring is how a sysadmin finds out a disk is filling *before* it takes a service down, and observability — dashboards plus alerts — is the difference between "a user told me it's broken" and "I fixed it before anyone noticed." This lab builds the standard open-source stack: Prometheus as the time-series database and scraper, node_exporter as the agent exposing each host's metrics, Grafana for dashboards kept in version control (not clicked together and lost), and Alertmanager to turn threshold breaches into notifications. The final milestone deliberately breaks something and watches the alert fire.

The runbooks are the *how*; **[TAKEAWAYS.md](TAKEAWAYS.md) is the *why*** — the pull model, why dashboards-as-code, and why a good alert is rare and precious.

## Architecture

```
   ┌──────────────┐   ┌──────────────┐        each host runs node_exporter :9100
   │ node_exporter│   │ node_exporter│        exposing CPU/mem/disk/net metrics
   │   host A     │   │   host B     │
   └──────┬───────┘   └──────┬───────┘
          │  scrape (pull)   │
          └────────┬─────────┘
             ┌──────┴───────┐         ┌───────────────┐
             │  Prometheus  │──alerts─►│ Alertmanager  │──► Discord / email
             │  TSDB + rules│         └───────────────┘
             └──────┬───────┘
                    │ query
             ┌──────┴───────┐
             │   Grafana    │  dashboards provisioned from grafana/dashboards/*.json
             └──────────────┘
        Prometheus + Grafana + Alertmanager in Docker Compose on the homelab host
```

## Milestones

| # | Runbook |
|---|---------|
| 0 | [Deploy the stack (Prometheus, Grafana, Alertmanager)](runbooks/00-stack-deploy.md) |
| 1 | [node_exporter targets](runbooks/01-node-exporter-targets.md) |
| 2 | [Grafana dashboards as code](runbooks/02-grafana-dashboards.md) |
| 3 | [Alert rules (PromQL)](runbooks/03-alert-rules.md) |
| 4 | [Alertmanager routing to Discord/email](runbooks/04-alertmanager-routing.md) |
| 5 | [Detect a real problem end to end](runbooks/05-detect-a-real-problem.md) |

## What's in this repo

- [`compose/docker-compose.yml`](compose/docker-compose.yml) — the three services + `.env.example`.
- [`prometheus/prometheus.yml`](prometheus/prometheus.yml) — scrape config + rule/alertmanager wiring.
- [`prometheus/alert.rules.yml`](prometheus/alert.rules.yml) — disk/CPU/instance-down alert rules.
- [`alertmanager/alertmanager.yml`](alertmanager/alertmanager.yml) — receivers, routing, grouping.
- [`grafana/provisioning/`](grafana/provisioning/) + [`grafana/dashboards/node-overview.json`](grafana/dashboards/node-overview.json) — datasource + dashboards as code.

## Skills this lab exercises

Prometheus (scrape configs, the pull model, TSDB, PromQL), node_exporter, Grafana (provisioned datasources and dashboards, dashboards-as-code), alerting with PromQL rules and Alertmanager (receivers, routing trees, grouping, inhibition, silences), Docker Compose, and the operational judgement of what's worth alerting on versus what's just noise.

## What I learned

Filled in per milestone as the lab progresses — see the closing section of each runbook. The goal isn't a pretty dashboard; it's a stack that told me about a problem before a user did, at least once, on purpose.

---
Built and maintained by **Edward J. Penna** — [github.com/wardoep](https://github.com/wardoep)
