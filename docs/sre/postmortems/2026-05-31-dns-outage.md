# Postmortem: 2026-05-31 — DNS Misconfiguration Caused Agent Init Failures

## Summary

- **Duration**: ~2 hours (detected at 2026-05-31, resolved same day)
- **Impact**: `kubeopencode-agent-server` and `slack-agent-server` pods stuck in `Init:Error` / `Init:CrashLoopBackOff`. All agent services unavailable.
- **Severity**: P1
- **Trigger**: DHCP-assigned host IP changed; hardcoded DNS record in CoreDNS ConfigMap became stale.

## Timeline

| Time (approx) | Event |
|---------------|-------|
| T+0 | Host IP changed from `192.168.31.59` to `192.168.31.91` (DHCP renewal) |
| T+0 | New agent pods created (or existing pods restarted) |
| T+5m | Pods stuck in `Init:Error`; `git-init` container cannot clone GitHub |
| T+30m | Investigation started: checked pod logs, found DNS resolution failure |
| T+45m | Confirmed `host.k3s.internal` resolves to stale IP `192.168.31.59` |
| T+60m | Updated CoreDNS ConfigMap; restarted CoreDNS Deployment |
| T+75m | Deleted stuck pods; new pods started successfully |
| T+90m | Full service restored |

## Root Cause Analysis (5 Whys)

1. **Why did agent pods fail to start?**  
   `git-init` container could not reach GitHub via proxy (`host.k3s.internal:7897`).

2. **Why could it not reach the proxy?**  
   DNS resolution of `host.k3s.internal` returned a non-existent IP.

3. **Why was the IP wrong?**  
   The `coredns-custom` ConfigMap contained a hardcoded IP (`192.168.31.59`) that no longer matched the host's actual IP.

4. **Why did the IP change?**  
   The host uses DHCP; the IP assignment changed after a network event (likely router reboot or lease expiration).

5. **Why wasn't the DNS record updated automatically?**  
   No automation monitors the host IP and syncs it to the CoreDNS ConfigMap. The entry was manually added once and never maintained.

## Impact Assessment

- **Services affected**: All KubeOpenCode Agent functionality (PR review, Slack responses, scheduled tasks)
- **User impact**: No new tasks could be created; Slack bot unresponsive
- **Data loss**: None (tasks were not created, so no data was lost)
- **Recovery**: Manual intervention required; no automatic remediation

## Action Items

| Item | Owner | Due | Status |
|------|-------|-----|--------|
| Fix `host.k3s.internal` to use a static IP or hostname alias | @xue | 2026-06-07 | Done |
| Add Alertmanager rule for CoreDNS SERVFAIL responses | SRE | 2026-06-10 | Pending |
| Document DNS troubleshooting in runbook | SRE | 2026-06-10 | Pending |
| Evaluate using host network or NodePort instead of DNS for proxy access | SRE | 2026-06-15 | Pending |

## Lessons Learned

1. **Hardcoded IPs are fragile**. Any infrastructure that relies on a specific IP address must have an automated update mechanism or use DNS names with proper TTL.
2. **DHCP + static DNS = trap**. In home lab / single-node setups, consider assigning a static IP to the K3s node or using a hostname that follows the IP change.
3. **Init container failures are easy to miss**. The pod status `Init:Error` is not as visible as `CrashLoopBackOff`. A readiness probe on the init container (or a startup probe) could surface this faster.
4. **Single-node clusters have single points of failure everywhere**. This is acceptable for a personal project, but each failure mode should be documented with a recovery procedure.
