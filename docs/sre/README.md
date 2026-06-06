# Site Reliability Engineering (SRE)

SRE practices for the KubeOpenCode Agent platform. This directory contains the full SRE implementation: SLOs, observability stack, alerting, postmortems, and runbooks.

## Current Cluster Status

Based on live cluster inspection (`kubectl`):

| Resource | Status |
|----------|--------|
| **Cluster** | Single-node K3s (`xue-b550`), Ubuntu 24.04, K8s v1.35.4+k3s1 |
| **CPU** | 5% used (803m / ~16 cores) — plenty of headroom |
| **Memory** | 42% used (6.3Gi / ~15.5Gi) — **healthy after vLLM cleanup** |
| **metrics-server** | Running (provides `kubectl top`) |
| **Agent** | `kubeopencode-agent`: READY column empty (investigating); `slack-agent`: healthy |
| **Tasks** | Multiple Completed CronTask pods; 2x `fix-vulnerabilities` pods **OOMKilled** |
| **PVCs** | 2x 1Gi `local-path` volumes, both Bound |

## Known Issues

1. **OOMKilled Tasks**: `fix-vulnerabilities` CronTask pods killed by OOM twice in the past 3 days. Memory limit too low for the workload.
2. **Agent Availability**: `kubeopencode-agent` shows no READY status. Needs investigation (see runbooks).
3. **DNS Drift**: Historical `host.k3s.internal` DNS misconfiguration caused agent init failures (see [postmortem](postmortems/2026-05-31-dns-outage.md)).

## Architecture

```
K3s Cluster (single-node, 16GB RAM)
├── kubeopencode-agent namespace
│   ├── kubeopencode-agent (AI agent)
│   ├── slack-agent (Slack integration)
│   └── CronTasks (pr-review, tiny-refactor, opencode-update, fix-vulnerabilities)
│
└── monitoring namespace  ← NEW
    ├── Prometheus (TSDB, scraping + remote-write from OTel)
    ├── Grafana (dashboards, visualizing SLOs)
    ├── Alertmanager (routing alerts to Slack)
    ├── kube-state-metrics (K8s resource metrics)
    └── otel-collector (receives OTLP from agent/task pods)
```

## Design Constraints

- **Single node**: No HA requirements, but single point of failure.
- **Memory constrained**: 42% baseline usage. Monitoring stack must be lightweight.
- **Local-path storage**: PVCs use `local-path` StorageClass. Prometheus needs local retention only.

## Document Index

| Document | Purpose |
|----------|---------|
| [slos.md](slos.md) | SLO/SLI/SLA definitions, error budget |
| [observability.md](observability.md) | Prometheus + Grafana deployment, scraping targets |
| [alerting.md](alerting.md) | PrometheusRules, Alertmanager, Slack integration |
| [postmortems/](postmortems/) | Incident retrospectives |
| [runbooks/](runbooks/) | Step-by-step operational procedures |

## Deploy

```bash
# Deploy the full SRE stack
kubectl apply -k deploy/sre/

# Verify
kubectl get pods -n monitoring
```

---

## Admin Operations Guide

Day-to-day operations manual for the SRE stack.

### Quick Health Check

```bash
# One-liner to check the full SRE stack
kubectl get pods -n monitoring

# Check whether all Prometheus targets are UP
kubectl exec -n monitoring sts/prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/targets' | \
  jq -r '.data.activeTargets[] | "\(.labels.job): \(.health)"' | sort | uniq -c

# Check currently firing alerts
kubectl exec -n monitoring sts/prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/alerts' | \
  jq -r '.data.alerts[] | select(.state == "firing") | "\(.labels.alertname) [\(.labels.severity)]"'
```

### Access Dashboards (Port-Forward)

All services are `ClusterIP`; access them via local port-forward:

```bash
# Grafana — view built-in dashboards (KubeOpenCode Overview / SLO Compliance)
kubectl port-forward -n monitoring svc/grafana 3000:3000
# Open: http://localhost:3000
# Anonymous access is enabled (no login). Admin login: admin / admin
#
# Direct dashboard URLs (after port-forward):
#   http://localhost:3000/d/kubeopencode-overview   — KubeOpenCode Overview
#   http://localhost:3000/d/slo-compliance          — SLO Compliance
# Or navigate via left sidebar: Dashboards → Browse

# Prometheus — query metrics, inspect targets, validate rules
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open: http://localhost:9090

# Alertmanager — view alert status, silence alerts, verify routing
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
# Open: http://localhost:9093
```

> Tip: Run all three commands in separate terminals; they do not conflict.

### Daily Checklist

Run these checks daily (or at the start of each session):

| Check | Command | Healthy Criteria |
|-------|---------|-----------------|
| Monitoring pods | `kubectl get pods -n monitoring` | All `1/1 Running`, RESTARTS at 0 or stable |
| Node resources | `kubectl top node` | CPU < 80%, Memory < 80% |
| Prometheus targets | `kubectl exec ... /api/v1/targets` | All `up` |
| Firing alerts | `kubectl exec ... /api/v1/alerts` | No P0/P1 firing |
| PVC capacity | `kubectl get pvc -n monitoring` | `prometheus-storage` < 85% full |
| Core agent tasks | `kubectl get cronjobs -n kubeopencode-agent` | `ACTIVE` is 0 (off-hours), no long `SUSPEND` |

### Alert Response Playbook

Response flow after receiving a Slack alert:

**P0 — Respond Immediately** (AgentDown, OOMKilledPod, K3sNodeNotReady, TaskFailSpike)
1. `kubectl get pods -n kubeopencode-agent` to identify affected pods
2. If OOMKilled: check `kubectl logs --previous`; consider raising the memory limit
3. If AgentDown: verify whether the Deployment `spec.replicas` is 0 (intentional scale-down → no action needed)
4. If NodeNotReady: inspect the node with `kubectl describe node xue-b550`

**P1 — Respond within 30 min** (CronTaskMissed, TaskPodPending, AgentHighMemoryUsage)
1. List pending pods: `kubectl get pods -n kubeopencode-agent | grep Pending`
2. Check node resource availability: `kubectl describe node`
3. Inspect CronJob schedule: `kubectl get cronjob <name> -o yaml | grep schedule`

**P2 — Handle within 4 hours** (PVCFull, NodeMemoryHigh, CoreDNSSERVFAIL)
1. PVC full: clean old data or expand the PVC
2. DNS issues: check `coredns` pod logs; verify custom resolutions such as `host.k3s.internal`

### Common Troubleshooting

**Prometheus CrashLoopBackOff**
```bash
# 99% caused by a PromQL syntax error in alerting rules
kubectl logs -n monitoring sts/prometheus -c prometheus | grep "loading groups failed"
# After fix: kubectl apply -k deploy/sre/ && kubectl delete pod -n monitoring -l app=prometheus
```

**A Target Shows `down`**
```bash
# Inspect the exact error
kubectl exec -n monitoring sts/prometheus -c prometheus -- \
  wget -qO- 'http://localhost:9090/api/v1/targets' | \
  jq '.data.activeTargets[] | select(.health == "down") | {job, last_error}'
```

**Grafana Dashboard Shows No Data**
- Verify Prometheus targets are UP
- Run the panel's PromQL manually in Grafana Explore to confirm data exists
- Check that the time range falls within the retention window (15 days by default)

**Disk Space Alert (PrometheusDiskFull)**
```bash
# Check Prometheus PVC usage
kubectl exec -n monitoring sts/prometheus -c prometheus -- \
  df -h /prometheus
# To expand: kubectl edit pvc prometheus-storage-prometheus-0 -n monitoring
```

### Configuration Updates

**Update Slack Webhook**
```bash
# Edit the placeholder file and apply, or create directly from CLI
kubectl create secret generic alertmanager-slack \
  --from-literal=webhook-url='https://hooks.slack.com/services/...' \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -
# Alertmanager hot-reloads the config automatically; no restart needed
```

**Modify Alerting Rules**
```bash
# Edit deploy/sre/rules/prometheus-rules.yaml
# Then apply and restart Prometheus to pick up the new rules
kubectl apply -k deploy/sre/
kubectl delete pod -n monitoring -l app=prometheus
```

**Modify Grafana Dashboards**
```bash
# Dashboard JSON is embedded in the ConfigMap in deploy/sre/grafana/grafana.yaml
# Recommended workflow: edit in Grafana UI, export JSON, paste back into the ConfigMap
kubectl apply -k deploy/sre/
kubectl rollout restart deployment/grafana -n monitoring
```

### Backup & Recovery

- **Prometheus data**: Stored on a `local-path` PVC with single-node local retention only; no automatic backup. For long-term storage, consider remote storage (Thanos / Mimir / VictoriaMetrics).
- **Dashboards / rules**: All defined as YAML in `deploy/sre/`; Git is the backup. To restore, run `kubectl apply -k deploy/sre/`.
- **Slack Secret**: The webhook URL is not in Git. If lost, re-create the secret from the Slack App dashboard.

### Resource Limits & Scaling

The monitoring stack currently uses roughly **~600 Mi** in practice (actual usage, not limit). If the cluster grows:

| Scenario | Suggested Adjustment |
|----------|---------------------|
| Nodes > 3 | Prometheus limit 512 Mi → 1 Gi; scrape interval 15 s → 30 s |
| Pods > 100 | kube-state-metrics limit 128 Mi → 256 Mi |
| Alert volume spikes | Alertmanager limit 64 Mi → 128 Mi |
| Many dashboard users | Grafana limit 128 Mi → 256 Mi; add a reverse-proxy cache |
