# 03 — Alert rules (PromQL)

**Goal:** understand and verify the alerting rules — disk, CPU, memory, and instance-down — and why each is written the way it is.

## The rules

[`prometheus/alert.rules.yml`](../prometheus/alert.rules.yml) defines:

| Alert | Fires when | `for:` |
|---|---|---|
| `InstanceDown` | `up{job="node"} == 0` | 2m |
| `DiskSpaceLow` | root FS < 15% free | 5m |
| `DiskSpaceCritical` | root FS < 5% free | 2m |
| `HighCpuLoad` | CPU busy > 90% (5m avg) | 10m |
| `LowMemory` | available memory < 10% | 5m |

The `for:` clause is the point: it requires the condition to hold for that long before firing, which kills alerts on momentary spikes. The annotations use `{{ $labels.instance }}` and `{{ $value }}` so the notification names the host and the number.

## Steps

1. **See the rules loaded:** Prometheus → *Status → Rules*. Each group/rule shows its state (inactive/pending/firing) and health.

2. **Test the PromQL directly** in the Prometheus *Graph* tab, e.g. the disk expression:
   ```promql
   (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
   ```
   Confirm it returns a sane percentage per instance.

3. **Understand pending vs firing.** When a condition first becomes true the alert is **pending**; only after `for:` elapses does it become **firing** and get sent to Alertmanager. You'll watch this transition in milestone 5.

4. **Tune thresholds** to your hardware. On a small disk 15% might be too tight or too loose; edit the expression/threshold in the rules file and reload Prometheus.

## Verify

- *Status → Rules* lists all five rules as **inactive** on a healthy system (nothing wrong = nothing firing).
- The disk/CPU/memory PromQL expressions each return values in the Graph tab.

## If it breaks

- **A rule shows "err" in Rule Health.** PromQL or YAML mistake — `docker logs prometheus` names it; a common one is an unescaped quote in an annotation.
- **DiskSpaceLow never fires even when low.** The `mountpoint`/`fstype` filter may not match your system's device — check what `node_filesystem_avail_bytes` actually labels your root filesystem as.
- **Alert fires instantly on a blip.** The `for:` is too short (or zero) — lengthen it.

## What I learned

*Filled in after I complete this milestone.*

<!-- Prompts to answer once done:
     - Pending vs firing — did watching the transition change how I think about `for:`?
     - What thresholds actually fit my hardware vs the defaults?
     - Which metric was hardest to write a correct PromQL expression for? -->
