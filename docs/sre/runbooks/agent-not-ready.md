# Runbook: Agent Not Ready

## Symptoms

- `kubectl get agents -n kubeopencode-agent` shows `kubeopencode-agent` with empty `READY` column
- `kubectl get pods -n kubeopencode-agent` shows no `kubeopencode-agent-server-*` pod, or pod status is not `Running`
- Scheduled tasks are not being created; Slack/GitHub events are not processed

## Diagnosis Steps

### Step 1: Check Agent CR Status

```bash
kubectl get agent kubeopencode-agent -n kubeopencode-agent -o yaml
```

Look for:
- `status.conditions`: any `False` conditions with messages
- `status.phase`: should be `Active` or `Standby`

### Step 2: Check for Existing Pods

```bash
kubectl get pods -n kubeopencode-agent -l app.kubernetes.io/name=kubeopencode-agent
```

**If no pods exist**:
- Agent may be in standby mode (auto-suspend after 30m idle)
- Trigger a task or check the Agent CR for suspend status

**If pod exists but not Running**:
```bash
kubectl describe pod -n kubeopencode-agent <agent-pod-name>
kubectl logs -n kubeopencode-agent <agent-pod-name>
```

### Step 3: Check Init Containers

Agent pods use init containers (`git-init-*`) to clone the repo:

```bash
kubectl logs -n kubeopencode-agent <agent-pod-name> -c git-init-0
```

Common init failures:
- **DNS resolution**: Cannot resolve GitHub or proxy (`host.k3s.internal`)
- **Network timeout**: Firewall blocking outbound HTTPS
- **Auth failure**: Invalid git credentials in Secret

### Step 4: Check Secrets

```bash
kubectl get secrets -n kubeopencode-agent
```

Verify these exist:
- `git-creds` (for repo cloning)
- `github-app-creds` (for GitHub API)
- `opencode-creds` (for model API access)

### Step 5: Check Events

```bash
kubectl get events -n kubeopencode-agent --field-selector reason=FailedMount,reason=FailedScheduling,reason=Failed
```

## Resolution

### Scenario A: Agent in Standby

Normal behavior. The agent auto-suspends after 30 minutes of idle time. It auto-resumes when a new Task is created.

**To manually wake up**:
```bash
# Trigger via annotation on any CronTask
kubectl annotate crontask pr-review -n kubeopencode-agent kubeopencode.io/trigger=true
```

### Scenario B: Init Container Fails (DNS/Network)

Follow [dns-resolution-failure](dns-resolution-failure.md) runbook.

### Scenario C: OOMKilled

The agent pod exceeded its memory limit. Follow [task-oomkilled](task-oomkilled.md) runbook.

### Scenario D: Image Pull Failure

```bash
kubectl describe pod -n kubeopencode-agent <agent-pod-name>
```

If `ImagePullBackOff`:
```bash
# Check image name and tag
kubectl get agenttemplate kubeopencode-base -n kubeopencode-agent -o yaml | grep image

# If using a private registry, verify image pull secret
kubectl get serviceaccount kubeopencode-agent -n kubeopencode-agent -o yaml
```

### Scenario E: PVC Not Bound

```bash
kubectl get pvc -n kubeopencode-agent
```

If `Pending`:
- Check StorageClass: `kubectl get storageclass`
- Ensure `local-path-provisioner` is running: `kubectl get pods -n kube-system | grep local-path`

## Verification

After applying the fix:

```bash
# Wait for pod to be ready
kubectl wait --for=condition=ready pod -n kubeopencode-agent -l app.kubernetes.io/name=kubeopencode-agent --timeout=120s

# Check agent CR status
kubectl get agent kubeopencode-agent -n kubeopencode-agent

# Test by triggering a task
kubectl annotate crontask pr-review -n kubeopencode-agent kubeopencode.io/trigger=true
kubectl get tasks -n kubeopencode-agent -w
```

## Escalation

If the agent is still not ready after all steps:

1. Collect diagnostics:
   ```bash
   kubectl get all -n kubeopencode-agent -o yaml > /tmp/agent-debug.yaml
   kubectl logs -n kubeopencode-agent deployment/kubeopencode-agent-server > /tmp/agent-logs.txt
   ```
2. Create a GitHub issue with the debug output
3. Manually run critical tasks from your local machine if urgent
