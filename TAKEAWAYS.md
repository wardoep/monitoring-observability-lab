# What this lab is really teaching — and why

The runbooks say *how* to stand up the stack. This says *why* it's built this way and what each piece teaches about running systems you can see. If I can explain everything here without notes, the lab did its job.

## The big picture

Monitoring answers "is it healthy right now and was it healthy before"; observability is the broader ability to *ask questions* of a running system — dashboards to see, alerts to be told, and metrics rich enough to explain *why*. The reason this matters for a sysadmin job is blunt: the alternative to monitoring is finding out from an angry user, which means the problem already had impact. **Good monitoring turns outages into near-misses.**

A mental model that holds up: **Prometheus doesn't wait to be told about problems; it goes and asks.** Every 15 seconds it scrapes each target and stores the numbers. That "pull" design is why adding a host is just adding a scrape target, why a target going silent is *itself* a signal (the `up` metric drops to 0), and why the metrics are a time series you can look *backwards* through, not just a live gauge.

## Milestone-by-milestone: what, why, takeaway

### 0 — Deploy the stack
**Builds:** Prometheus (scraper + time-series DB), Grafana (dashboards), Alertmanager (notifications), wired together in compose with their config mounted from the repo.
**Why config-in-git:** the entire stack — scrape targets, alert rules, dashboards, routing — lives in files in this repo. If it vanished, `docker compose up` rebuilds it exactly. That's the whole "infrastructure as code" idea applied to monitoring.
**Takeaway:** the config *is* the system; keep it in version control and the stack is reproducible.

### 1 — node_exporter targets
**Builds:** an agent on each host exposing CPU/mem/disk/network as metrics, and Prometheus scraping them.
**Why the `up` metric is special:** Prometheus records, for every target, whether the last scrape succeeded (`up = 1/0`). So "the exporter is unreachable" needs no special heartbeat mechanism — it falls out of the pull model for free, and it's the basis of the `InstanceDown` alert.
**Takeaway:** in a pull system, silence is a signal — a target that stops answering is detectable without any extra plumbing.

### 2 — Dashboards as code
**Builds:** Grafana dashboards provisioned from JSON files on disk, and the datasource provisioned too.
**Why not click them together:** a dashboard built by clicking exists in one Grafana's database and dies with it. Provisioned from a file, it's reproducible, reviewable in a diff, and shareable. The datasource is provisioned for the same reason — a fresh Grafana comes up already knowing where Prometheus is.
**Takeaway:** anything worth keeping shouldn't live only in a tool's clickable state — put it in a file.

### 3 — Alert rules
**Builds:** PromQL rules that fire on disk/CPU/memory/instance-down conditions.
**Why `for:` matters:** an alert without a duration fires on a single momentary spike and trains you to ignore it. `for: 5m` means "only alert if this has been true for five minutes" — which kills flapping and is the difference between an alert you trust and noise you mute. The annotations matter too: a good alert says *what*, *where*, and *how bad* right in the message.
**Takeaway:** a good alert is rare, sustained, and self-explanatory; the `for:` clause is what keeps it from becoming noise.

### 4 — Alertmanager routing
**Builds:** grouping, throttling, inhibition, and delivery to Discord/email.
**Why a separate component:** Prometheus decides *what* is wrong; Alertmanager decides *who hears about it and how often*. Grouping means ten disk alerts on one host arrive as one message; `repeat_interval` stops it nagging every 15s; **inhibition** suppresses the warning when a critical for the same host is already firing. All of that is "don't cry wolf" engineering — the fastest way to make people ignore alerts is to send too many.
**Takeaway:** notification is its own discipline — group, throttle, and inhibit, because an ignored alert channel is worse than none.

### 5 — Detect a real problem
**Builds:** proof the whole chain works, by breaking something on purpose and watching the alert fire and clear.
**Why induce it:** an alerting stack you've never seen fire is untested. Filling a disk with `fallocate` or stopping a service and watching the notification arrive (then clear on `send_resolved`) is the monitoring equivalent of a restore drill.
**Takeaway:** an alert you've never watched fire is a guess; test the whole path end to end at least once.

## If I only remember five things

1. Prometheus pulls — it asks every target every 15s, so silence (`up == 0`) is itself a signal.
2. Keep scrape config, rules, and dashboards in git; the config *is* the system.
3. `for:` turns a spiky metric into a trustworthy alert — no duration, no trust.
4. Alertmanager is "don't cry wolf": group, throttle, inhibit, or people tune the channel out.
5. Break something on purpose once and watch the alert fire and resolve — untested alerting is a guess.

---
Built and maintained by **Edward J. Penna** — [github.com/wardoep](https://github.com/wardoep)
