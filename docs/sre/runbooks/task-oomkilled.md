# Runbook: Task OOMKilled

## Symptoms

- Task pod status shows `OOMKilled` in `kubectl get pods -n kubeopencode-agent`
- Container was terminated with reason `OOMKilled`
- Task never reaches `Completed` or `Failed` phase

## Example

```bash
$ kubectl get pods -n kubeopencode-agent
NAME                                 READY   STATUS      RESTARTS   AGE
fix-vulnerabilities-1780293600-pod   0/1     OOMKilled   0          3d7h
fix-vulnerabilities-1780466400-pod   0/1     OOMKilled   0          31h
```

## Diagnosis Steps

### Step 1: Confirm OOMKilled

```bash
kubectl describe pod -n kubeopencode-agent <pod-name>
```

Look for:
```
Last State: Terminated
  Reason:    OOMKilled
  Exit Code: 137
```

Also check the events section:
```
Events:
  Type     Reason     Age   From               Message
  ----     ------     ----  ----               -------
  Warning  OOMKilling  10m   kubelet            Memory cgroup out of memory: Killed process ...
```

### Step 2: Check Memory Usage Pattern

```bash
# If metrics-server is available
kubectl top pod -n kubeopencode-agent <pod-name>

# Check the pod's memory limit
kubectl get pod -n kubeopencode-agent <pod-name> -o jsonpath='{.spec.containers[0].resources.limits.memory}'
```

### Step 3: Check Task Type

Different tasks have different memory profiles:

| Task | Typical Memory | Notes |
|------|---------------|-------|
| `pr-review` | 512Mi - 1Gi | Depends on PR size and repo |
| `tiny-refactor` | 512Mi - 1Gi | Depends on codebase size |
| `fix-vulnerabilities` | 1Gi - 2Gi | Runs `pnpm audit fix`, `go get`, may build from source |
| `opencode-update` | 256Mi - 512Mi | Simple version check |

The `fix-vulnerabilities` task is the most likely to OOM because it may compile dependencies.

## Resolution

### Option 1: Increase Memory Limit (Recommended)

Edit the CronTask or AgentTemplate:

```bash
# Find the CronTask
kubectl get crontask -n kubeopencode-agent

# Edit the specific CronTask
kubectl edit crontask fix-vulnerabilities -n kubeopencode-agent
```

In the `taskTemplate.spec`, add or increase memory limits:

```yaml
resources:
  limits:
    memory: "2Gi"  # Increase from previous value
  requests:
    memory: "1Gi"
```

### Option 2: Reduce Task Concurrency

If multiple tasks run simultaneously and exhaust node memory:

```bash
# Check current CronTask schedules
kubectl get crontasks -n kubeopencode-agent -o yaml | grep schedule

# Stagger schedules to avoid overlap
# Example: change fix-vulnerabilities from 6:00 to 5:00 UTC
kubectl patch crontask fix-vulnerabilities -n kubeopencode-agent \
  --type merge -p '{"spec":{"schedule":"0 5 * * *"}}'
```

### Option 3: Optimize the Task Itself

For `fix-vulnerabilities`:
- The task runs `pnpm audit fix` and `go get` which can be memory-intensive
- Consider splitting into separate tasks per language
- Or run with `--prefer-offline` to reduce network/memory overhead

### Option 4: Add Node Memory Alert

Prevent future OOMs by monitoring node memory:

```bash
# Check node memory usage
kubectl top node

# If node memory is consistently high (>80%), consider:
# - Adding memory to the node
# - Reducing other workloads
# - Enabling swap (not recommended for K8s, but acceptable for single-node labs)
```

## Verification

After increasing limits:

```bash
# Trigger the task manually
kubectl annotate crontask fix-vulnerabilities -n kubeopencode-agent kubeopencode.io/trigger=true

# Watch the pod
kubectl get pods -n kubeopencode-agent -l kubeopencode.io/crontask=fix-vulnerabilities -w

# Verify it completes
kubectl logs -n kubeopencode-agent <new-pod-name>
```

## Prevention

1. **Set memory requests = limits** for tasks to avoid noisy neighbor issues
2. **Monitor node memory** with Prometheus alert `NodeMemoryHigh`
3. **Review task resource usage monthly** and adjust limits based on actual usage
4. **Document memory requirements** in each workflow's README
