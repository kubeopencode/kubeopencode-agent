# Postmortem: 2026-06-11 — DNS IP Drift Caused Slack Agent Socket Mode Failure (Recurring)

## Summary

- **Duration**: ~14 hours (07:05 UTC detected → ~13:30 UTC resolved)
- **Impact**: Slack bot completely unresponsive — all @mentions and DMs ignored. P0 alerts fired every ~65 minutes.
- **Severity**: P0
- **Trigger**: DHCP renewed host IP from `192.168.31.59` to `192.168.31.91`, then back to `192.168.31.59`. CoreDNS `host.k3s.internal` entry became stale.

## Timeline

| Time (approx) | Event |
|---------------|-------|
| 2026-05-31 | Original incident: DNS hardcoded to `.59`, IP changed to `.91`. Fixed by updating CoreDNS to `.91`. |
| 2026-06-05 | slack-agent pod restarted (log timestamp), Socket Mode reconnected successfully |
| 2026-06-11 07:05 | First P0 alert: `SlackAgentDown` firing. `host.k3s.internal` resolving to `.91` but actual IP is `.59` again. |
| 2026-06-11 07:05 – ~13:00 | Alerts repeat every ~65 min. Slack bot unresponsive the entire time. |
| 2026-06-11 ~13:25 | Investigation started. Checked pod status, logs, connectivity. |
| 2026-06-11 ~13:28 | Root cause identified: `host.k3s.internal` → `192.168.31.91` (wrong) vs node IP `192.168.31.59` (correct). |
| 2026-06-11 ~13:29 | Updated `coredns-custom` ConfigMap to point `.59`. Restarted CoreDNS. |
| 2026-06-11 ~13:30 | Deleted old slack-agent pod. New pod started, Socket Mode connected successfully. |
| 2026-06-11 ~13:31 | Full service restored. |

## Root Cause Analysis (5 Whys)

1. **Why was Slack bot unresponsive?**  
   Slack Socket Mode WebSocket and Web API requests were failing with `ECONNREFUSED` — the plugin couldn't reach `slack.com` via the proxy.

2. **Why couldn't it reach the proxy?**  
   `host.k3s.internal:7897` resolved to `192.168.31.91`, which is unreachable from pods (No route to host). The actual node IP is `192.168.31.59`.

3. **Why was the DNS entry wrong?**  
   The `coredns-custom` ConfigMap had a hardcoded IP `192.168.31.91` — set during the 2026-05-31 incident fix. The host IP subsequently changed back to `192.168.31.59` (DHCP reassignment).

4. **Why does the host IP keep changing?**  
   The K3s node uses DHCP, which can reassign IPs on lease renewal or router reboot.

5. **Why is there no automated mechanism to keep DNS in sync?**  
   The `coredns-custom` entry is managed manually. No automation monitors the host IP or updates DNS. This is the same root cause as the 2026-05-31 incident — the action item to "fix host.k3s.internal to use a static IP" was marked "Done" but was only a point-in-time fix, not a systemic one.

## Impact Assessment

- **Services affected**: Slack integration (all @mentions and DMs to the bot). GitHub agent, CronTasks, and other services were not affected.
- **User impact**: ~14 hours of Slack bot downtime. P0 alerts were firing but the bot was unresponsive.
- **Data loss**: None. Messages sent during downtime were not processed (no queue/buffer).
- **Recovery**: Required manual intervention to update CoreDNS and restart the pod.

## Action Items

| Item | Owner | Due | Status |
|------|-------|-----|--------|
| Assign a static IP to the K3s node (e.g., `192.168.31.59` DHCP reservation or static config) | @xue | 2026-06-18 | Open |
| Create a CronJob or DaemonSet that watches the node IP and auto-updates `coredns-custom` ConfigMap | SRE | 2026-06-18 | Open |
| Add Prometheus alert for `host.k3s.internal` DNS resolution mismatch with node IP | SRE | 2026-06-14 | Open |
| Evaluate using `hostNetwork: true` or a Kubernetes Service instead of DNS for proxy access | SRE | 2026-06-25 | Open |
| Restart slack-agent pod after CoreDNS changes (added to runbook) | Done | — | Done |

## Lessons Learned

1. **Point-in-time DNS fixes are not enough.** The 2026-05-31 incident was "resolved" by updating CoreDNS, but the underlying cause (DHCP IP drift) was not addressed. This is a textbook recurring incident.
2. **DHCP + manual DNS = guaranteed recurrence.** Every DHCP lease renewal or router reboot can change the IP. A static IP assignment or automated DNS sync is mandatory.
3. **P0 alerts need a response SLA.** The SlackAgentDown alert fired for 14 hours before anyone investigated. Consider routing P0 alerts to an on-call mechanism beyond Slack itself.
4. **Pod restart after DNS change is necessary.** Even after fixing CoreDNS, the slack-agent pod retained stale DNS cache. A pod restart was required.