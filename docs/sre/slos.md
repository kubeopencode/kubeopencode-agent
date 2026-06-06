# SLO / SLI / SLA

Service Level Objectives for the KubeOpenCode Agent platform.

## Definitions

| Term | Definition | Example |
|------|------------|---------|
| **SLI** | Service Level Indicator — a measurable metric | Agent Pod ready percentage |
| **SLO** | Service Level Objective — the target value for an SLI | 99.5% ready per month |
| **SLA** | Service Level Agreement — external commitment (not formal yet) | Based on SLO + penalty/bonus |

## SLOs

### 1. Agent Availability

| Field | Value |
|-------|-------|
| **SLI** | `kubeopencode-agent` Pod `Ready` condition = `True` |
| **SLO** | 99.5% availability per calendar month |
| **Measurement** | Prometheus: `kube_pod_status_ready{pod=~"kubeopencode-agent-server-.*"}` |
| **Window** | Rolling 30 days |
| **Rationale** | Agent is the core service. Downtime blocks all PR reviews, Slack responses, and scheduled tasks. |

**Error Budget**: 0.5% unavailability = ~3.6 hours/month.
**Policy**: When error budget is exhausted (>50% burned in <7 days), freeze all non-critical feature work and focus on reliability.

### 2. Task Success Rate

| Field | Value |
|-------|-------|
| **SLI** | Tasks with `status.phase == "Succeeded"` / total created Tasks |
| **SLO** | 99% success rate per week |
| **Measurement** | KubeOpenCode Task status via custom exporter or API polling |
| **Window** | Rolling 7 days |
| **Rationale** | Failed tasks waste compute and may miss critical PRs or security fixes. |

### 3. CronTask Punctuality

| Field | Value |
|-------|-------|
| **SLI** | CronTasks executed within 5 minutes of scheduled time |
| **SLO** | 99% on-time per week |
| **Measurement** | Compare CronTask `status.lastScheduleTime` to `spec.schedule` |
| **Window** | Rolling 7 days |
| **Rationale** | Daily PR review and vulnerability fixes are time-sensitive. |

### 4. Task P95 Duration

| Field | Value |
|-------|-------|
| **SLI** | Time from Task creation to completion (successful) |
| **SLO** | P95 < 30 minutes |
| **Measurement** | Task `metadata.creationTimestamp` → `status.completionTime` |
| **Window** | Rolling 7 days |
| **Rationale** | Long-running tasks block concurrency slots and delay responses. |

### 5. API Error Rate

| Field | Value |
|-------|-------|
| **SLI** | GitHub/Slack API 4xx/5xx errors / total API calls |
| **SLO** | < 1% error rate per day |
| **Measurement** | Log-based metric from agent stdout (via Promtail or file-based scraping) |
| **Window** | Rolling 24 hours |
| **Rationale** | High API error rates indicate credential issues, rate limits, or upstream outages. |

## Error Budget Policy

```
Error Budget = 1 - SLO (e.g., 0.5% for Agent Availability)

Burn Rate Alerts:
- Burn rate > 14.4x  (exhaust in <2 days)   → Page immediately
- Burn rate > 6x     (exhaust in <3 days)   → High-priority ticket
- Burn rate > 2x     (exhaust in <1 week)   → Warn in Slack

When budget is exhausted:
1. Freeze all non-critical deployments
2. Weekly reliability review
3. No new features until SLO is back on track
```

## SLO Dashboard

See [dashboards](observability.md#dashboards) for Grafana panels tracking these SLOs in real time.
