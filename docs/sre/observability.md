# Observability Stack

Lightweight Prometheus + Grafana + OpenTelemetry Collector deployment for a single-node K3s cluster with 16GB RAM.

## Architecture

```
monitoring namespace
├── prometheus (StatefulSet, 2Gi PVC, 512Mi memory limit)
│   ├── Scrapes: K8s API, kubelet, kube-state-metrics, agent pods
│   ├── Receives: Remote-write from OTel Collector
│   └── Retention: 15 days local storage
├── grafana (Deployment, 128Mi memory limit)
│   ├── Pre-configured dashboards (ConfigMap)
│   └── Anonymous read access (no auth for local use)
├── alertmanager (Deployment, 64Mi memory limit)
│   └── Routes alerts to Slack via webhook
├── kube-state-metrics (Deployment, 128Mi memory limit)
│   └── Exposes K8s resource metrics (Pod, PVC, CronTask status)
└── otel-collector (Deployment, 128Mi memory limit)
    ├── Receives: OTLP from KubeOpenCode agent/task pods (PR #236)
    ├── Forwards: Metrics to Prometheus via remote-write
    └── Logs: Traces to stdout (future: Jaeger/Tempo)
```

## Resource Budget

Total monitoring footprint: **~1Gi memory**, well within the cluster's remaining capacity.

| Component | Memory Request | Memory Limit | CPU Request | CPU Limit |
|-----------|---------------|--------------|-------------|-----------|
| Prometheus | 256Mi | 512Mi | 100m | 500m |
| Grafana | 64Mi | 128Mi | 50m | 100m |
| Alertmanager | 32Mi | 64Mi | 20m | 50m |
| kube-state-metrics | 64Mi | 128Mi | 50m | 100m |
| OTel Collector | 64Mi | 128Mi | 50m | 100m |

### How It Works

1. **KubeOpenCodeConfig** declares OTel settings (`observability.openTelemetry`)
2. **Controller** auto-injects OTLP environment variables into all agent/task pods:
   - `OTEL_EXPORTER_OTLP_ENDPOINT` — points to OTel Collector
   - `OTEL_EXPORTER_OTLP_HEADERS` — authentication headers
   - `OTEL_RESOURCE_ATTRIBUTES` — K8s metadata (pod, namespace, agent name)
   - `recordContent: true` in `KubeOpenCodeConfig` enables `experimental_telemetry.recordInputs/recordOutputs` — captures LLM prompts and responses in traces
3. **OpenCode runtime** (Layer 2) emits `ai.*` traces and metrics via AI SDK
4. **OTel Collector** receives OTLP, processes, and forwards metrics to Prometheus

### Configuration

```yaml
# deploy/kubeopencodeconfig.yaml
apiVersion: kubeopencode.io/v1alpha1
kind: KubeOpenCodeConfig
metadata:
  name: cluster
spec:
  observability:
    openTelemetry:
      enabled: true
      endpoint: "http://otel-collector.monitoring.svc.cluster.local:4318"
      recordContent: true
      resourceAttributes:
        deployment.environment: "production"
```

### Trace Hierarchy

```
ai.streamText (span)
└── ai.streamText.doStream (span)
    └── ai.toolCall (span, if tool use)
```

**Observed attributes**: `ai.model.id`, `ai.model.provider`, `ai.usage.promptTokens`, `ai.usage.completionTokens`, `ai.telemetry.functionId`, `ai.telemetry.metadata.*`.

> **Note**: `ai.usage.totalTokens` and provider-specific token breakdowns (e.g. cached/reasoning) are not currently emitted as standard span attributes by the AI SDK. They may appear in provider metadata or telemetry integration events, but should not be relied upon in trace attributes.
>
> See [Vercel AI SDK Telemetry documentation](https://sdk.vercel.ai/docs/ai-sdk-core/telemetry) for the authoritative span names and attribute reference.

### Layer Breakdown

| Layer | Source | Data | Destination |
|-------|--------|------|-------------|
| **Layer 1** | KubeOpenCode controller | Pod/task metadata | OTel Collector |
| **Layer 2** | OpenCode AI SDK | `ai.*` traces/metrics | OTel Collector |
| **Layer 3** | Kube-state-metrics | K8s resource state | Prometheus (direct scrape) |

## Scraping Targets

Prometheus discovers targets via Kubernetes SD:

| Job | Endpoint | Metrics |
|-----|----------|---------|
| `kubernetes-pods` | All pods with `prometheus.io/scrape: "true"` annotation | Application metrics |
| `kubernetes-nodes` | Kubelet `/metrics` | Node CPU, memory, disk |
| `kube-state-metrics` | `kube-state-metrics:8080/metrics` | Pod status, PVC, CronTask state |
| `prometheus-self` | `localhost:9090/metrics` | Prometheus health |

### K3s Control Plane

K3s bundles control plane components into a single process, but exposes separate metrics endpoints. Monitoring these is essential for single-node clusters because control plane degradation affects all workloads.

| Job | Endpoint | Metrics |
|-----|----------|---------|
| `k3s-etcd` | `localhost:2381/metrics` | etcd disk fsync latency, DB size, leader elections |
| `k3s-apiserver` | `kubernetes.default.svc:443/metrics` | API request latency, error rate, LIST operation duration |

> **Note**: For single-node K3s, etcd metrics are available on `localhost:2381`. API server metrics require service account token authentication when scraping from in-cluster Prometheus.

**Key metrics to alert on:**
- `etcd_disk_wal_fsync_duration_seconds` (histogram) — etcd write-ahead log fsync latency. P95 > 100ms indicates disk pressure.
- `apiserver_request_duration_seconds` (histogram) — API server response time. P99 > 1s suggests overloaded control plane.
- `process_resident_memory_bytes{job="k3s-server"}` — k3s process memory trend. Steady growth may indicate a memory leak.

### Agent Pod Scraping

To enable metrics scraping from agent pods, add this annotation to the Pod template:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

> Note: KubeOpenCode controller auto-injects OTel env vars. For Prometheus scraping, you may need a separate metrics endpoint or use OTel Collector's Prometheus exporter.

## Deployment

```bash
# Deploy the full stack
kubectl apply -k deploy/sre/

# Verify all pods are running
kubectl get pods -n monitoring

# Port-forward Grafana for local access
kubectl port-forward -n monitoring svc/grafana 3000:3000
# Open http://localhost:3000 (no login required)

# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Open http://localhost:9090
```

## Dashboards

Three pre-configured dashboards are loaded via ConfigMap:

### 1. KubeOpenCode Overview

- Agent health status (red/green)
- Current running Task count / queue depth
- Daily Task success rate trend
- CronTask execution calendar
- Resource usage (CPU / Memory / PVC)

### 2. Task Performance

- Task latency distribution (P50 / P95 / P99)
- Success rate by workflow (pr-review vs tiny-refactor vs fix-vulnerabilities)
- Task failure classification (timeout vs error vs cancelled)

### 3. SLO Compliance

- Monthly availability trend vs 99.5% SLO line
- Error budget remaining (burn rate)
- API call volume and error rate

### 4. K3s Control Plane Health

- etcd WAL fsync latency (P95) — alerts if > 100ms
- API server request duration (P99) — alerts if > 1s
- k3s server process memory trend
- Node Ready status and condition transitions

> Dashboard JSON definitions are embedded in `deploy/sre/grafana/dashboards-configmap.yaml`.

## Retention

Prometheus retains data for **15 days** with a **2Gi** PVC. This is sufficient for:
- Weekly SLO reviews
- Incident investigation (within 2 weeks)
- Capacity planning trends

To adjust retention or storage:

```bash
# Edit the Prometheus StatefulSet
kubectl edit statefulset prometheus -n monitoring
# Change --storage.tsdb.retention.time and PVC size
```
