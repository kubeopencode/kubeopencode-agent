# Runbook: Slack Bot Unresponsive

## Symptoms

- Slack messages in DM or channel mentions get no response
- `slack-agent-server` pod is running but not processing messages
- Socket Mode connection shows as disconnected in Slack app settings

## Diagnosis Steps

### Step 1: Check Slack Agent Pod

```bash
kubectl get pods -n kubeopencode-agent -l app.kubernetes.io/name=slack-agent
kubectl logs -n kubeopencode-agent deployment/slack-agent-server
```

Look for:
- Socket Mode connection errors
- Authentication failures
- Crash loop errors

### Step 2: Check Slack App Status

1. Go to https://api.slack.com/apps
2. Select your KubeOpenCode app
3. Check "Socket Mode" page:
   - Connection count should be > 0
   - No error messages

### Step 3: Verify Tokens

```bash
# Check Slack tokens are valid
kubectl get secret slack-creds -n kubeopencode-agent -o jsonpath='{.data.bot-token}' | base64 -d
kubectl get secret slack-socket-mode-creds -n kubeopencode-agent -o jsonpath='{.data.app-token}' | base64 -d
```

**Token format**:
- Bot token: `xoxb-...` (starts with `xoxb`)
- App token: `xapp-...` (starts with `xapp`)

### Step 4: Test Token Validity

```bash
export SLACK_BOT_TOKEN=$(kubectl get secret slack-creds -n kubeopencode-agent -o jsonpath='{.data.bot-token}' | base64 -d)
curl -H "Authorization: Bearer $SLACK_BOT_TOKEN" https://slack.com/api/auth.test
```

Should return `{"ok": true, ...}`. If `{"ok": false, "error": "invalid_auth"}`, token is revoked or expired.

## Resolution

### Scenario A: Token Expired or Revoked

1. Go to Slack app settings → OAuth & Permissions
2. Reinstall the app to your workspace
3. Copy the new Bot User OAuth Token
4. Update the secret:
   ```bash
   kubectl create secret generic slack-creds \
     --from-literal=bot-token='xoxb-NEW-TOKEN' \
     -n kubeopencode-agent --dry-run=client -o yaml | kubectl apply -f -
   
   kubectl rollout restart deployment slack-agent-server -n kubeopencode-agent
   ```

### Scenario B: Socket Mode App Token Invalid

1. Go to Slack app settings → Basic Information → App-Level Tokens
2. Generate a new token with `connections:write` scope
3. Update the secret:
   ```bash
   kubectl create secret generic slack-socket-mode-creds \
     --from-literal=app-token='xapp-NEW-TOKEN' \
     -n kubeopencode-agent --dry-run=client -o yaml | kubectl apply -f -
   
   kubectl rollout restart deployment slack-agent-server -n kubeopencode-agent
   ```

### Scenario C: Network Issues — DNS IP Drift (Most Common)

If the pod logs show `ECONNREFUSED` or `pong wasn't received` errors, the most likely cause is `host.k3s.internal` DNS resolving to a stale IP. The K3s node uses DHCP, so its IP can change on lease renewal.

```bash
# 1. Check what host.k3s.internal resolves to inside the pod
kubectl exec -n kubeopencode-agent deployment/slack-agent-server -- \
  sh -c 'cat /etc/resolv.conf; echo "---"; wget -O- http://host.k3s.internal:7897/ 2>&1 | head -3'
# If "No route to host" → DNS points to wrong IP

# 2. Verify the actual node IP
kubectl get nodes -o wide
# Compare INTERNAL-IP with what host.k3s.internal resolves to

# 3. Fix CoreDNS — update coredns-custom ConfigMap with the correct IP
# Replace 192.168.31.XX with the actual node IP from step 2
kubectl create configmap coredns-custom \
  --from-literal=host.k3s.internal.server='host.k3s.internal {
    hosts {
      192.168.31.XX host.k3s.internal
      fallthrough
    }
  }
  ' -n kube-system --dry-run=client -o yaml | kubectl apply -f -

# 4. Restart CoreDNS to pick up changes
kubectl rollout restart deployment coredns -n kube-system
kubectl rollout status deployment coredns -n kube-system

# 5. Verify DNS now resolves correctly
dig +short host.k3s.internal @10.43.0.10
# Should return the correct node IP

# 6. Restart slack-agent pod (DNS cache inside the pod must be flushed)
kubectl rollout restart deployment slack-agent-server -n kubeopencode-agent
# If rollout doesn't create a new pod (PVC ReadWriteOnce), delete old pod:
kubectl delete pod -n kubeopencode-agent -l kubeopencode.io/agent=slack-agent

# 7. Verify connection
kubectl logs deployment/slack-agent-server -n kubeopencode-agent --tail=5
# Should show: [slack-plugin] Slack Socket Mode connected
```

Also check direct Slack connectivity (bypassing proxy):

```bash
kubectl exec -n kubeopencode-agent deployment/slack-agent-server -- \
  wget --no-proxy -qO- https://slack.com/api/api.test
# Should return {"ok":true,...}
```

### Scenario D: Plugin Not Loaded

If the pod starts but does not process messages:

```bash
# Check plugin configuration
kubectl get agent slack-agent -n kubeopencode-agent -o yaml | grep -A5 plugin

# Should show: plugin: "@kubeopencode/opencode-slack-plugin"
# If missing, edit the agent CR
```

## Verification

```bash
# Wait for pod to be ready
kubectl wait --for=condition=ready pod -n kubeopencode-agent -l app.kubernetes.io/name=slack-agent --timeout=60s

# Check logs for successful Socket Mode connection
kubectl logs -n kubeopencode-agent deployment/slack-agent-server | grep -i "connected\|socket mode"

# Send a test message in Slack
# The bot should respond within 10-30 seconds
```

## Prevention

1. **Monitor Slack agent health** with Prometheus (if exposed)
2. **Set up token expiry alerts** (Slack tokens don't expire by default, but can be revoked)
3. **Document token rotation procedure** in team onboarding
4. **Test Slack integration** after every agent deployment
