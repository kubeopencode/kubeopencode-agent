# Runbook: Prometheus Disk Full

## Symptoms

- Prometheus pod stuck in `Pending` or `CrashLoopBackOff`
- `kubectl describe pod` shows `FailedMount` or disk pressure events
- Prometheus UI shows `out of space` errors
- Alert `PrometheusDiskFull` fires

## Diagnosis Steps

### Step 1: Check PVC Usage

```bash
kubectl get pvc -n monitoring
kubectl describe pvc prometheus-storage-prometheus-0 -n monitoring

# Check actual usage from inside the pod
kubectl exec -n monitoring prometheus-0 -- df -h /prometheus
```

### Step 2: Check Prometheus Metrics

```bash
# Port-forward Prometheus
kubectl port-forward -n monitoring svc/prometheus 9090:9090

# Check TSDB status
curl -s http://localhost:9090/api/v1/status/tsdb | jq '.data.headStats'
```

Look for:
- `numSeries`: number of active time series
- `chunkCount`: number of chunks
- `maxTime`: latest timestamp

### Step 3: Check Retention Settings

```bash
kubectl get statefulset prometheus -n monitoring -o yaml | grep retention
```

Current settings:
- `--storage.tsdb.retention.time=15d`
- `--storage.tsdb.retention.size=1.5GB`

## Resolution

### Option 1: Reduce Retention (Fastest)

Shorten retention to free up space immediately:

```bash
# Edit the StatefulSet
kubectl edit statefulset prometheus -n monitoring
```

Change:
```yaml
args:
  - '--storage.tsdb.retention.time=7d'  # Reduce from 15d to 7d
  - '--storage.tsdb.retention.size=1GB' # Reduce from 1.5GB to 1GB
```

Then restart:
```bash
kubectl rollout restart statefulset prometheus -n monitoring
```

### Option 2: Expand PVC

If you have storage capacity on the node:

```bash
# Note: local-path provisioner may not support expansion
# Check if expansion is supported
kubectl get storageclass local-path -o yaml | grep allowVolumeExpansion

# If supported:
kubectl patch pvc prometheus-storage-prometheus-0 -n monitoring \
  --type merge -p '{"spec":{"resources":{"requests":{"storage":"3Gi"}}}}'

# If not supported, you need to:
# 1. Scale down Prometheus
# 2. Delete old PVC
# 3. Create new PVC with larger size
# 4. Scale up Prometheus (data will be lost)
```

### Option 3: Reduce Scraped Metrics

If too many time series are being collected:

```bash
# Edit Prometheus ConfigMap
kubectl edit configmap prometheus-config -n monitoring
```

Reduce scraping frequency or drop unnecessary metrics:
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    scrape_interval: 30s  # Increase from 15s to 30s
    # Or add metric_relabel_configs to drop high-cardinality labels
```

### Option 4: Manual Cleanup (Emergency)

```bash
# Exec into Prometheus pod and manually delete old blocks
kubectl exec -it -n monitoring prometheus-0 -- /bin/sh

# List TSDB blocks
cd /prometheus
ls -la

# Remove blocks older than 7 days (use with caution!)
# find . -name "*.db" -mtime +7 -delete
```

> Warning: Manual deletion can corrupt TSDB. Only use if other options fail.

## Verification

```bash
# Check PVC usage after cleanup
kubectl exec -n monitoring prometheus-0 -- df -h /prometheus

# Check Prometheus is healthy
kubectl port-forward -n monitoring svc/prometheus 9090:9090
curl -s http://localhost:9090/-/healthy
curl -s http://localhost:9090/-/ready

# Check no disk full alerts firing
curl -s 'http://localhost:9090/api/v1/alerts' | jq '.data.alerts[] | select(.labels.alertname == "PrometheusDiskFull")'
```

## Prevention

1. **Monitor disk usage** with the `PrometheusDiskFull` alert (fires at 85%)
2. **Review retention monthly** based on actual usage patterns
3. **Track time series count** — high cardinality labels can explode storage
4. **Set up capacity planning** — if consistently >70%, plan expansion
5. **Document max capacity** — 2Gi PVC with 15d retention is the current design limit
