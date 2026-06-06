# Alerting

Prometheus alerting rules and Alertmanager routing to Slack.

## Alert Severity Levels

| Level | Response Time | Notification Channel | Example |
|-------|--------------|---------------------|---------|
| **P0 (page)** | Immediate | Slack #alerts + mention | Agent down, Task failure spike |
| **P1 (high)** | 30 minutes | Slack #alerts | CronTask missed, high API error rate |
| **P2 (warn)** | 4 hours | Slack #alerts | PVC filling up, standby flapping |

## Prometheus Rules

All rules are defined in `deploy/sre/rules/prometheus-rules.yaml`.

### P0 — Critical

#### AgentDown
```yaml
alert: AgentDown
expr: kube_pod_status_ready{namespace="kubeopencode-agent", pod=~"kubeopencode-agent-server-.*"} == 0
for: 5m
labels:
  severity: p0
annotations:
  summary: "kubeopencode-agent is not ready"
  description: "Agent pod has been not ready for more than 5 minutes."
```

#### TaskFailSpike
```yaml
alert: TaskFailSpike
expr: (
  sum(increase(kube_pod_container_status_terminated_reason{namespace="kubeopencode-agent", reason="Error"}[10m]))
  /
  sum(increase(kube_pod_container_status_terminated_reason{namespace="kubeopencode-agent"}[10m]))
) > 0.2
for: 5m
labels:
  severity: p0
annotations:
  summary: "Task failure rate above 20%"
  description: "More than 20% of tasks failed in the last 10 minutes."
```

#### OOMKilledPod
```yaml
alert: OOMKilledPod
expr: kube_pod_container_status_terminated_reason{namespace="kubeopencode-agent", reason="OOMKilled"} == 1
for: 0m
labels:
  severity: p0
annotations:
  summary: "Pod OOMKilled in kubeopencode-agent"
  description: "Pod {{ $labels.pod }} was killed due to out of memory."
```

#### K3sNodeNotReady
```yaml
alert: K3sNodeNotReady
expr: kube_node_status_condition{condition="Ready",status="true"} == 0
for: 2m
labels:
  severity: p0
annotations:
  summary: "K3s node is NotReady"
  description: "Node {{ $labels.node }} has been NotReady for more than 2 minutes. All cluster workloads are at risk."
```

### P1 — High

#### CronTaskMissed
```yaml
alert: CronTaskMissed
expr: time() - kube_cronjob_next_schedule_time{namespace="kubeopencode-agent"} > 600
for: 5m
labels:
  severity: p1
annotations:
  summary: "CronTask missed its schedule"
  description: "CronTask {{ $labels.cronjob }} is more than 10 minutes overdue."
```

#### HighAPIErrorRate
```yaml
alert: HighAPIErrorRate
expr: rate(github_api_errors_total[5m]) / rate(github_api_calls_total[5m]) > 0.05
for: 10m
labels:
  severity: p1
annotations:
  summary: "GitHub API error rate elevated"
  description: "GitHub API error rate is above 5%."
```

#### K3sEtcdHighFsyncLatency
```yaml
alert: K3sEtcdHighFsyncLatency
expr: histogram_quantile(0.95, sum(rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) by (le, instance)) > 0.1
for: 5m
labels:
  severity: p1
annotations:
  summary: "K3s etcd fsync latency high"
  description: "Etcd WAL fsync P95 latency is above 100ms. Disk IO pressure may be affecting control plane stability."
```

#### K3sAPIServerHighLatency
```yaml
alert: K3sAPIServerHighLatency
expr: histogram_quantile(0.99, sum(rate(apiserver_request_duration_seconds_bucket[5m])) by (le, instance)) > 1
for: 5m
labels:
  severity: p1
annotations:
  summary: "K3s API server latency high"
  description: "API server request P99 latency is above 1 second. Control plane may be overloaded."
```

### P2 — Warning

#### PVCFull
```yaml
alert: PVCFull
expr: kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.2
for: 10m
labels:
  severity: p2
annotations:
  summary: "PVC filling up"
  description: "PVC {{ $labels.persistentvolumeclaim }} is over 80% full."
```

#### CoreDNSResolutionFail
```yaml
alert: CoreDNSResolutionFail
expr: increase(coredns_dns_response_rcode_count{rcode="SERVFAIL"}[5m]) > 0
for: 5m
labels:
  severity: p2
annotations:
  summary: "CoreDNS returning SERVFAIL"
  description: "CoreDNS is failing to resolve queries. Check custom DNS entries."
```

## Slack Integration

Alertmanager routes alerts to a Slack channel via webhook.

### Configuration

The Slack webhook URL is stored in a Kubernetes Secret:

```yaml
# deploy/sre/alertmanager/slack-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-slack
  namespace: monitoring
type: Opaque
stringData:
  webhook-url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

### Routing

```yaml
# Alertmanager config (deploy/sre/alertmanager/alertmanager.yaml)
route:
  receiver: slack
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
- name: slack
  slack_configs:
  - channel: '#alerts'
    title: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    send_resolved: true
```

### Setup Steps

1. Create a Slack app: https://api.slack.com/apps
2. Enable "Incoming Webhooks"
3. Add to your workspace and select channel `#alerts`
4. Copy the webhook URL
5. Update the secret:
   ```bash
   kubectl create secret generic alertmanager-slack \
     --from-literal=webhook-url='https://hooks.slack.com/services/...' \
     -n monitoring --dry-run=client -o yaml | kubectl apply -f -
   ```
6. Restart Alertmanager to pick up the config:
   ```bash
   kubectl rollout restart deployment alertmanager -n monitoring
   ```

## Testing Alerts

```bash
# Manually fire a test alert (requires amtool)
amtool alert add --alertmanager.url=http://alertmanager.monitoring:9093 \
  alertname=TestAlert severity=p2 \
  --annotation=summary="Test alert" \
  --annotation=description="This is a test. Ignore."

# Check Alertmanager UI
kubectl port-forward -n monitoring svc/alertmanager 9093:9093
# Open http://localhost:9093
```
