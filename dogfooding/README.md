# KubeOpenCode Dogfooding Environment

This directory contains resources for running KubeOpenCode in a dogfooding environment, where KubeOpenCode is used to automate tasks on its own repository.

## Architecture

A single unified `dogfooding` agent handles all tasks. Its capabilities are defined by skills in the repo itself (`.opencode/skills/` for OpenCode, `.claude/skills/` for Claude Code) — the agent discovers them automatically at runtime via the `mountPath: .` context (see [Repo as Agent Pattern](#repo-as-agent-pattern)).

```
GitHub / Slack / CronJobs
    │
    │ Events trigger Task creation
    │
    ▼
Task (kubeopencode-dogfooding namespace)
    │
    │ agentRef: dogfooding
    │
    ▼
Agent Pod (unified dogfooding agent)
    │
    │ Skills auto-discovered from .claude/skills/
    │
    ▼
Responds via GitHub API / Slack CLI / Creates PRs
```

### Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `github-respond` | GitHub @mention | Answer questions, make code changes, create PRs |
| `pr-review` | CronJob (daily) | Review open PRs for bugs, security, style |
| `tiny-refactor` | CronJob (every 3 days) | Static analysis + one safe refactoring |
| `opencode-update` | CronJob (weekly) | Check for new OpenCode releases, update Dockerfile |
| `slack-respond` | Slack @mention/DM | Respond to Slack messages via slack-cli |

## Directory Structure

```
dogfooding/
├── base/                              # Resources for kubeopencode-dogfooding namespace
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── rbac.yaml
│   ├── secrets.yaml                   # Git, GitHub App, OpenCode, Slack credentials
│   ├── kubeopencodeconfig.yaml        # Cluster-wide KubeOpenCode settings
│   └── agent-dogfooding.yaml          # Unified agent (gemini-3.1-pro, maxConcurrent: 3)
├── scheduled/                         # CronJob resources (kubeopencode-scheduled namespace)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── rbac.yaml
│   ├── cronjob-pr-review.yaml         # Daily PR review
│   ├── cronjob-tiny-refactor.yaml     # Refactoring every 3 days
│   └── cronjob-opencode-update.yaml   # Weekly OpenCode version check
└── slack/                             # Slack integration (kubeopencode-slack namespace)
    ├── kustomization.yaml
    ├── namespace.yaml
    ├── secrets.yaml                   # Slack Socket Mode credentials
    └── socket-mode-gateway.yaml       # Python gateway + RBAC + Deployment
```

## Setup

### Prerequisites

1. KubeOpenCode operator installed in the cluster
2. A GitHub App with write permissions configured for the repository

### Deploy

```bash
# Apply base resources (agent, RBAC, namespace)
kubectl apply -k dogfooding/base

# Apply scheduled CronJobs
kubectl apply -k dogfooding/scheduled

# Apply Slack gateway
kubectl apply -k dogfooding/slack

# Verify agent creation
kubectl get agents -n kubeopencode-dogfooding
```

## Repo as Agent Pattern

This environment uses the **Repo as Agent** pattern: the Git repository itself defines the agent's identity — skills, instructions, and contexts. The agent discovers capabilities automatically at runtime when the repo is mounted at the workspace root.

### How It Works

```
dogfooding repo (this repo)         Kubernetes cluster
┌──────────────────────┐          ┌─────────────────────────────┐
│ AGENTS.md            │ ──git──▶ │ /workspace/                 │
│ .opencode/skills/    │  clone   │   ├── AGENTS.md             │
│   ├── pr-review/     │  into    │   ├── .opencode/skills/     │
│   ├── slack-respond/ │ workspace│   ├── task.md  ← from Task  │
│   └── ...            │  root    │   └── kubeopencode/ ← biz   │
│ .claude/             │          │                             │
│   ├── CLAUDE.md      │          │ OpenCode auto-discovers:    │
│   └── skills/ ←used  │          │   • AGENTS.md (instructions)│
│        by symlinks   │          │   • .opencode/skills/*      │
└──────────────────────┘          └─────────────────────────────┘
```

### Repo Structure

The repo maintains both OpenCode-native and Claude Code-compatible formats via symlinks:

```
dogfooding/
├── AGENTS.md              → .claude/CLAUDE.md     # OpenCode native (takes precedence)
├── .opencode/skills/      → .claude/skills/*      # OpenCode native skill discovery
├── .claude/
│   ├── CLAUDE.md                                  # Source of truth for agent instructions
│   └── skills/                                    # Source of truth for skill definitions
│       ├── pr-review/SKILL.md
│       ├── github-respond/SKILL.md
│       └── ...
```

OpenCode discovers `AGENTS.md` and `.opencode/skills/` at the workspace root. Claude Code discovers `.claude/CLAUDE.md` and `.claude/skills/`. Both tools read the same content via symlinks.

### Agent Configuration

The key to the Repo as Agent pattern is `mountPath`:

| Context | `mountPath` | Volume Strategy | Purpose |
|---------|-------------|-----------------|---------|
| **Agent repo** (identity) | `.` (workspace root) | Merged into workspace emptyDir | AGENTS.md, skills auto-discovery |
| **Business repo** (target) | Named subdirectory | Normal subPath mount | Code the agent works on |

```yaml
apiVersion: kubeopencode.io/v1alpha1
kind: Agent
metadata:
  name: my-agent
spec:
  workspaceDir: /workspace          # All mounts relative to this
  contexts:
    # Agent identity repo — mountPath: . enables skill auto-discovery
    - name: agent-repo
      type: Git
      git:
        repository: https://github.com/org/agent-repo.git
        ref: main
      mountPath: .                  # Merged into /workspace/

    # Business repo — mountPath: <name> for normal mount
    - name: target-repo
      type: Git
      git:
        repository: https://github.com/org/target-repo.git
        ref: main
      mountPath: target             # Mounted at /workspace/target/
```

> **Note**: `mountPath: .` requires KubeOpenCode v0.0.3+ (PR [#73](https://github.com/kubeopencode/kubeopencode/pull/73)). Earlier versions have a bug where the git volume mount shadows `task.md`.

### Dogfooding Agent Settings

| Setting | Value |
|---------|-------|
| **Model** | gemini-3.1-pro (most capable) |
| **Small Model** | gemini-3-flash |
| **General Subagent** | gemini-3-flash-preview (faster/cheaper for subtasks) |
| **Max Concurrent Tasks** | 3 (handles parallel GitHub + scheduled + Slack) |
| **Rate Limit** | 200 task starts per 24 hours |
| **GitHub Credentials** | github-dev-credentials (write access) |

### Contexts

| Context | Type | Mount Path | Description |
|---------|------|------------|-------------|
| `dogfooding` | Git | `.` (workspace root) | This repo — AGENTS.md and .opencode/skills/ auto-discovered |
| `kubeopencode` | Git | `kubeopencode` | KubeOpenCode source code for review/refactoring |

## Scheduled Tasks (CronJobs)

CronJobs run in `kubeopencode-scheduled` namespace and create Tasks in `kubeopencode-dogfooding` namespace via cross-namespace RBAC.

| CronJob | Schedule | Description |
|---------|----------|-------------|
| `pr-review` | Daily at 7:00 UTC | Reviews open PRs without `ai-reviewed` label |
| `tiny-refactor` | Every 3 days at 8:00 UTC | One small safe refactoring in kubeopencode |
| `opencode-update` | Weekly Monday at 9:00 UTC | Checks for new OpenCode releases |

### Manual Trigger (Testing)

```bash
# Create a one-off Job from a CronJob
kubectl create job --from=cronjob/tiny-refactor tiny-refactor-manual -n kubeopencode-scheduled

# Check created Tasks
kubectl get tasks -n kubeopencode-dogfooding -l kubeopencode.io/scheduled=true
```

## Slack Integration

The Slack Socket Mode gateway runs in `kubeopencode-slack` namespace. When a user @mentions or DMs the bot, the gateway creates a KubeOpenCode Task with Slack context (channel, thread). The agent responds using `slack-cli send`.

## Argo Workflow Integration

KubeOpenCode Tasks can be integrated with [Argo Workflows](https://argoproj.github.io/workflows/) to build AI-powered pipelines. Argo Workflow's `resource` template can create Tasks, wait for completion, and extract results from Task annotations for use in subsequent steps.

### How It Works

1. **Create Task**: Argo's `resource` template creates a KubeOpenCode Task
2. **Wait for Completion**: Polls `status.phase` until `Completed` or `Failed`
3. **Extract Results**: Uses JQ filter or JSONPath to read Task annotations
4. **Pass to Next Step**: Extracted values become available as workflow parameters

### Passing Results via Annotations

AI agents can write results to Task annotations using `kubectl annotate`. Argo Workflows can then extract these annotations using JQ filters or JSONPath expressions.

**Agent writes results:**
```bash
kubectl annotate task $TASK_NAME -n $TASK_NAMESPACE \
  kubeopencode.io/pr-url="https://github.com/org/repo/pull/123" \
  kubeopencode.io/summary="Fixed the login bug"
```

**Argo extracts results:**
```yaml
outputs:
  parameters:
    - name: pr-url
      valueFrom:
        jqFilter: '.metadata.annotations["kubeopencode.io/pr-url"]'
    - name: summary
      valueFrom:
        jqFilter: '.metadata.annotations["kubeopencode.io/summary"]'
```
