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

### Scenario C: Network Issues

If the pod logs show WebSocket connection timeouts:

```bash
# Check if pod can reach Slack
kubectl exec -n kubeopencode-agent deployment/slack-agent-server -- \
  wget -qO- https://slack.com/api/api.test

# Should return {"ok":true}
# If timeout, check cluster egress (DNS, firewall, proxy)
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
