# 05 — Detect a real problem end to end

**Goal:** prove the whole chain — metric → rule → Alertmanager → Discord → resolved — by breaking something on purpose and watching it fire and clear. The monitoring equivalent of a restore drill.

## Steps

### Drill A — disk fills up

1. On a target host, create a big file to push root FS under the alert threshold (size it to your disk):
   ```bash
   # CAREFUL: use a scratch host or a safe size. This makes a 5 GB file.
   # Write to /var/tmp, NOT /tmp: on many systemd distros /tmp is a tmpfs (RAM),
   # so filling it consumes memory, not the root filesystem — and the DiskSpaceLow
   # rule excludes tmpfs, so it would never fire. /var/tmp is on the root FS.
   fallocate -l 5G /var/tmp/fill.img
   df -h /                       # confirm used % rose and free dropped below 15%
   ```
2. Watch the pipeline:
   - Prometheus *Status → Rules*: `DiskSpaceLow` goes **pending**, then **firing** after its `for: 5m`.
   - Alertmanager UI: the alert appears.
   - Discord: the notification arrives.
3. **Clean up and watch it resolve:**
   ```bash
   rm /var/tmp/fill.img
   ```
   Within a scrape or two the alert clears and (thanks to `send_resolved`) a "resolved" message arrives.

### Drill B — a service/instance goes down

1. Stop node_exporter on a target (or stop the container): `docker stop node-exporter`.
2. `InstanceDown` goes pending → firing after 2m; Discord notifies.
3. `docker start node-exporter` → resolves.

## Verify

- You watched at least one alert go **pending → firing → resolved**, and saw both the firing and resolved messages in Discord.
- Record which drill you ran and how long each transition took — this is your proof the stack actually works, not just that it's running.

## If it breaks

- **Disk dropped but no alert.** The rule's `mountpoint`/`fstype` filter doesn't match the filesystem you filled — check with the PromQL from [runbook 03](03-alert-rules.md); or you didn't wait out the `for:`.
- **Fired in Prometheus but nothing in Discord.** The Prometheus→Alertmanager or Alertmanager→bridge link is broken — re-check [runbook 04](04-alertmanager-routing.md); the test-alert curl isolates which half.
- **Never resolved.** `send_resolved` is off on the receiver, or the condition is still true (did the cleanup actually free space?).
