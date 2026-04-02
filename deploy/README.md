# KubeOpenCode Agent — Deployment

Kubernetes deployment resources for the KubeOpenCode Agent. All resources run in a single `kubeopencode-agent` namespace.

## Architecture

```
GitHub / Slack / CronJobs
    |
    | Events trigger Task creation
    v
Task (kubeopencode-agent namespace)
    |
    | agentRef: kubeopencode-agent
    v
Agent Deployment (always running, with standby auto-suspend/resume)
    |
    | Task Pod connects via `opencode run --attach`
    | Skills auto-discovered from .claude/skills/
    v
Responds via GitHub API / Slack CLI / Creates PRs
```

## Directory Structure

```
deploy/
├── kustomization.yaml             # Single kustomization for all resources
├── namespace.yaml                 # kubeopencode-agent namespace
├── rbac.yaml                      # All ServiceAccounts, Roles, RoleBindings
├── secrets.yaml                   # All credentials (git, GitHub, OpenCode, Slack)
├── kubeopencodeconfig.yaml        # Cluster-wide KubeOpenCode settings
├── agenttemplate.yaml             # kubeopencode-base (shared images, credentials, contexts)
├── agent.yaml                     # kubeopencode-agent (references template, adds model config + standby)
├── crontasks/                     # Scheduled tasks (CronTask CRDs)
│   ├── crontask-pr-review.yaml        # Daily PR review
│   ├── crontask-tiny-refactor.yaml    # Refactoring every 3 days
│   ├── crontask-opencode-update.yaml  # Weekly OpenCode version check
│   └── crontask-fix-vulnerabilities.yaml # Weekly Dependabot vulnerability fix
└── socket-mode-gateway.yaml       # Slack Socket Mode gateway (ConfigMap + Deployment)
```

## Setup

### Prerequisites

1. KubeOpenCode operator installed in the cluster
2. A GitHub App with write permissions configured for the repository

### Deploy

```bash
# Apply all resources
kubectl apply -k deploy/

# Verify agent creation
kubectl get agents -n kubeopencode-agent
```

## Repo as Agent Pattern

This environment uses the **Repo as Agent** pattern: the Git repository itself defines the agent's identity. The agent discovers capabilities automatically at runtime when the repo is mounted at the workspace root.

### How It Works

```
agent repo (this repo)              Kubernetes cluster
+----------------------+          +-----------------------------+
| README.md (= agent)  | --git--> | /workspace/                 |
| CLAUDE.md -> README   |  clone   |   +-- AGENTS.md             |
| AGENTS.md -> README   |  into    |   +-- .opencode/skills/     |
| .opencode/skills/     | workspace|   +-- task.md  <- from Task  |
| .claude/skills/       |  root    |                             |
| deploy/               |          | Auto-discovers:             |
| docs/                 |          |   * AGENTS.md (instructions)|
| workflows/            |          |   * .opencode/skills/*      |
+----------------------+          +-----------------------------+
```

### Agent Settings

| Setting | Value |
|---------|-------|
| **Template** | kubeopencode-base |
| **Model** | gemini-3.1-pro |
| **Small Model** | gemini-3-flash |
| **General Subagent** | gemini-3-flash |
| **Max Concurrent Tasks** | 3 |
| **Rate Limit** | 200 task starts per 24 hours |
| **Standby** | Auto-suspend after 30m idle, auto-resume on new Task |
| **Persistence** | Sessions (1Gi PVC) |

### Context

| Context | Type | Mount Path | Description |
|---------|------|------------|-------------|
| `kubeopencode-agent` | Git | `.` (workspace root) | This repo — AGENTS.md, skills, workflows auto-discovered |

Reference repos (kubeopencode, skills, etc.) are managed via `repos/repos.yaml` and cloned on demand by `repos/sync-repos.sh` — no separate git context needed.

## Scheduled Tasks (CronTasks)

Scheduled tasks use the native KubeOpenCode `CronTask` CRD — a Task factory that creates Tasks on a cron schedule (analogous to CronJob creating Jobs, but without requiring kubectl Pods or RBAC for Task creation).

All CronTasks run in the same `kubeopencode-agent` namespace.

| CronTask | Schedule | Description |
|----------|----------|-------------|
| `pr-review` | Daily at 7:00 UTC | Reviews open PRs without `ai-reviewed` label |
| `tiny-refactor` | Every 3 days at 8:00 UTC | One small safe refactoring in kubeopencode |
| `opencode-update` | Weekly Monday at 9:00 UTC | Checks for new OpenCode releases |
| `fix-vulnerabilities` | Weekly Wednesday at 6:00 UTC | Fixes open Dependabot alerts via pnpm overrides / go get |

All CronTasks use `concurrencyPolicy: Forbid` and `maxRetainedTasks: 5`.

### Manual Trigger (Testing)

```bash
# Trigger via annotation (no kubectl Pod needed)
kubectl annotate crontask tiny-refactor kubeopencode.io/trigger=true -n kubeopencode-agent

# Check created Tasks
kubectl get tasks -n kubeopencode-agent -l kubeopencode.io/crontask=tiny-refactor

# Suspend a CronTask
kubectl patch crontask pr-review -n kubeopencode-agent --type merge -p '{"spec":{"suspend":true}}'
```

## Slack Integration

The Slack Socket Mode gateway runs as a Deployment in the `kubeopencode-agent` namespace. When a user @mentions or DMs the bot, the gateway creates a KubeOpenCode Task. The agent responds using `slack-cli send`.
