# Runbooks

Step-by-step operational procedures for the KubeOpenCode Agent platform.

## Index

| Runbook | Symptoms | Severity |
|---------|----------|----------|
| [agent-not-ready](agent-not-ready.md) | `kubeopencode-agent` Pod not ready or missing | P0 |
| [task-oomkilled](task-oomkilled.md) | Task pods killed with `OOMKilled` status | P1 |
| [dns-resolution-failure](dns-resolution-failure.md) | Init containers cannot resolve external hosts | P1 |
| [github-api-rate-limit](github-api-rate-limit.md) | GitHub API returns 403/429 errors | P1 |
| [slack-bot-unresponsive](slack-bot-unresponsive.md) | Slack messages not getting responses | P1 |
| [prometheus-disk-full](prometheus-disk-full.md) | Prometheus PVC approaching capacity | P2 |

## How to Use

1. Identify the symptom from the index above
2. Follow the runbook steps in order
3. If the issue persists after all steps, escalate by creating an issue in the GitHub repo
4. Update the runbook if a new resolution step is discovered
