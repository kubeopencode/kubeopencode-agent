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
├── slack-agent.yaml               # slack-agent (Slack integration via opencode-slack-plugin)
├── crontasks/                     # Scheduled tasks (CronTask CRDs)
│   ├── crontask-pr-review.yaml        # Daily PR review
│   ├── crontask-tiny-refactor.yaml    # Refactoring every 3 days
│   └── crontask-opencode-update.yaml  # Weekly OpenCode version check
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

### kubeopencode-agent Settings

| Setting | Value |
|---------|-------|
| **Template** | kubeopencode-base |
| **Provider** | OpenCode Go |
| **Model** | deepseek-v4-flash (reasoningEffort: high) |
| **Small Model** | deepseek-v4-flash (reasoningEffort: high) |
| **General Subagent** | deepseek-v4-flash (reasoningEffort: high) |
| **Max Concurrent Tasks** | 3 |
| **Rate Limit** | 200 task starts per 24 hours |
| **Standby** | Auto-suspend after 30m idle, auto-resume on new Task |
| **Persistence** | Sessions (1Gi PVC) |

### slack-agent Settings

| Setting | Value |
|---------|-------|
| **Template** | kubeopencode-base |
| **Plugin** | `@kubeopencode/opencode-slack-plugin` |
| **Model** | deepseek-v4-flash (reasoningEffort: high) |
| **Max Concurrent Tasks** | 3 |
| **Rate Limit** | 200 task starts per 24 hours |
| **Persistence** | Sessions (1Gi PVC) |

### Context

| Context | Type | Mount Path | Description |
|---------|------|------------|-------------|
| `kubeopencode-agent` | Git | `.` (workspace root) | This repo — AGENTS.md, skills, workflows auto-discovered |

Reference repos (kubeopencode, skills, etc.) are managed via `repos/repos.yaml` and cloned on demand by `repos/sync-repos.sh` — no separate git context needed.

## Model Configuration

### Reasoning / Thinking Effort

OpenCode models can support configurable reasoning depth. The mechanism depends on the model's provider and npm SDK:

| Provider SDK | Parameter | Values |
|---|---|---|
| `@ai-sdk/openai` | `reasoningEffort` | `none`, `minimal`, `low`, `medium`, `high`, `xhigh` |
| `@ai-sdk/openai-compatible` | `reasoningEffort` (via `providerOptions.openaiCompatible`) | `low`, `medium`, `high`, `max` |
| `@ai-sdk/anthropic` | `thinking.budgetTokens` | numeric token budget |
| `@ai-sdk/google` | `thinkingConfig.thinkingBudget` / `thinkingLevel` | numeric or `low`/`high` |

### How to Check if a Model Supports Reasoning Effort

1. **In the TUI**: Run `/models`, select a model, then cycle variants with the variant cycle keybind. If variants appear (e.g. `low`, `medium`, `high`), the model supports configurable reasoning.

2. **Via OpenCode source code** (in `repos/deps/anomalyco/opencode`):
   - Check `packages/opencode/src/provider/transform.ts` — the `variants()` function (line ~629) determines which reasoning variants a model gets.
   - Models in the **exclusion list** (containing `deepseek-chat`, `deepseek-reasoner`, `deepseek-r1`, `deepseek-v3`, `minimax`, `glm`, `kimi`, `k2p`, `qwen`, `big-pickle`) return `{}` — no reasoning effort control.
   - Models NOT in the exclusion list that use `@ai-sdk/openai-compatible` (including `deepseek-v4-*` via `opencode-go`) get the standard `["low", "medium", "high", "max"]` reasoning effort variants.
   - Anthropic models get `thinking` with `budgetTokens` variants.
   - Google models get `thinkingConfig` variants.

3. **Via DeepSeek V4 API docs**: DeepSeek V4 models (`deepseek-v4-flash`, `deepseek-v4-pro`) support `reasoning_effort: "high" | "max"` and `thinking: { type: "enabled" | "disabled" }`. The `opencode-go` provider routes these via the OpenAI-compatible SDK.

### Current Configuration

The agent and template use `deepseek-v4-flash` with `reasoningEffort: high`:

```yaml
config:
  model: opencode-go/deepseek-v4-flash
  small_model: opencode-go/deepseek-v4-flash
  provider:
    opencode-go:
      models:
        deepseek-v4-flash:
          options:
            reasoningEffort: high
  agent:
    general:
      model: opencode-go/deepseek-v4-flash
```

To change reasoning depth, update the `options` block. For DeepSeek V4:
- `reasoningEffort: low` — lighter thinking
- `reasoningEffort: high` — balanced (default for V4)
- `reasoningEffort: max` — maximum reasoning, requires larger context window

## Scheduled Tasks (CronTasks)

Scheduled tasks use the native KubeOpenCode `CronTask` CRD — a Task factory that creates Tasks on a cron schedule (analogous to CronJob creating Jobs, but without requiring kubectl Pods or RBAC for Task creation).

All CronTasks run in the same `kubeopencode-agent` namespace.

| CronTask | Schedule | Description |
|----------|----------|-------------|
| `pr-review` | Daily at 7:00 UTC | Reviews open PRs without `ai-reviewed` label |
| `tiny-refactor` | Every 3 days at 8:00 UTC | One small safe refactoring in kubeopencode |
| `opencode-update` | Weekly Monday at 9:00 UTC | Checks for new OpenCode releases |

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

Slack integration is handled by the dedicated **`slack-agent`** running the **`@kubeopencode/opencode-slack-plugin`**.

### Architecture

```
Slack (DM / @mention)
    |
    | Socket Mode (outbound WebSocket)
    v
slack-agent Deployment
    |
    | opencode-slack-plugin (inside opencode serve)
    | Maps each Slack thread → independent OpenCode session
    v
Direct Slack API replies (no Task creation needed)
```

- The plugin runs **inside** the `opencode serve` process — zero port exposure
- Each Slack thread creates an independent OpenCode session with its own context
- Permission requests and tool call progress are forwarded to Slack threads in real time


### Credentials

The `slack-agent` inherits credentials from the `kubeopencode-base` AgentTemplate:

| Secret | Key | Env Var | Purpose |
|--------|-----|---------|---------|
| `slack-socket-mode-creds` | `app-token` | `SLACK_APP_TOKEN` | Socket Mode WebSocket connection |
| `slack-creds` | `bot-token` | `SLACK_BOT_TOKEN` | Slack Web API (send messages) |

### Deployment

```bash
# Deploy (included in kustomization.yaml)
kubectl apply -k deploy/

# Verify
kubectl get agents -n kubeopencode-agent
kubectl get deployment slack-agent-server -n kubeopencode-agent
```
