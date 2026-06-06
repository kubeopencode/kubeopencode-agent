# Runbook: GitHub API Rate Limit

## Symptoms

- Agent logs show `403 Forbidden` or `429 Too Many Requests` from GitHub API
- Tasks fail when calling GitHub (creating PR comments, reviews, etc.)
- Error message: `API rate limit exceeded` or `You have exceeded a secondary rate limit`

## Diagnosis Steps

### Step 1: Check Rate Limit Status

```bash
# Using your GitHub token
curl -H "Authorization: token $(kubectl get secret github-app-creds -n kubeopencode-agent -o jsonpath='{.data.token}' | base64 -d)" \
  https://api.github.com/rate_limit
```

Look for:
- `rate.remaining`: if 0, primary rate limit hit
- `rate.reset`: Unix timestamp when limit resets
- `resources.core.remaining`: core API limit
- `resources.search.remaining`: search API limit

### Step 2: Check Agent Logs for API Errors

```bash
kubectl logs -n kubeopencode-agent deployment/kubeopencode-agent-server | grep -i "rate limit\|403\|429"
```

### Step 3: Identify Which API Is Being Rate Limited

GitHub has different rate limits:

| API Type | Limit (Authenticated) | Reset |
|----------|----------------------|-------|
| REST API | 5,000 requests/hour | Hourly |
| GraphQL | 5,000 points/hour | Hourly |
| Search | 30 requests/minute | Per minute |
| GitHub App | 15,000 requests/hour (installation) | Hourly |

## Resolution

### Scenario A: Primary Rate Limit (5,000/hour)

**Wait it out**: Limits reset every hour.

**Reduce usage**:
```bash
# Temporarily suspend non-essential CronTasks
kubectl patch crontask tiny-refactor -n kubeopencode-agent --type merge -p '{"spec":{"suspend":true}}'
kubectl patch crontask opencode-update -n kubeopencode-agent --type merge -p '{"spec":{"suspend":true}}'

# Resume after 1 hour
kubectl patch crontask tiny-refactor -n kubeopencode-agent --type merge -p '{"spec":{"suspend":false}}'
```

### Scenario B: Secondary Rate Limit (Abuse Detection)

GitHub triggers this for:
- Rapid consecutive requests
- Concurrent requests to the same resource
- Large number of POST/PUT/DELETE requests

**Actions**:
1. Add delays between API calls in workflows
2. Reduce `maxConcurrentTasks` on the agent:
   ```bash
   kubectl patch agent kubeopencode-agent -n kubeopencode-agent --type merge -p '{"spec":{"maxConcurrentTasks":1}}'
   ```
3. Wait 1-2 minutes before retrying

### Scenario C: Search Rate Limit (30/min)

If using GitHub search API extensively:
- Cache search results
- Reduce search frequency
- Use GraphQL for more efficient queries

### Scenario D: Token Issues

If rate limit is unexpectedly low:
```bash
# Check token type
kubectl get secret github-app-creds -n kubeopencode-agent -o yaml | grep -i "app\|token"

# GitHub Apps have higher limits (15K/hour) than personal tokens (5K/hour)
# If using PAT, consider migrating to GitHub App
```

## Verification

```bash
# Check rate limit again
curl -H "Authorization: token YOUR_TOKEN" https://api.github.com/rate_limit

# Trigger a small task and verify it succeeds
kubectl annotate crontask opencode-update -n kubeopencode-agent kubeopencode.io/trigger=true
kubectl logs -n kubeopencode-agent job/opencode-update-...
```

## Prevention

1. **Monitor API usage** with Prometheus metric `github_api_calls_total`
2. **Add rate limiting** in workflow code (sleep between calls)
3. **Use GitHub App** instead of PAT for higher limits
4. **Batch operations** instead of individual API calls
5. **Set up alerting** for rate limit remaining < 10%
