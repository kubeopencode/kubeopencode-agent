# Workflows

Workflows are **user-triggered or scheduled** multi-phase processes. The user knows the workflow exists and invokes it by name, or a cron job triggers it on a schedule.

**Key trait:** The human (or scheduler) initiates. The agent follows the defined phases in order.

> **How workflows differ from skills:**
>
> | Layer | Analogy | Who initiates |
> |-------|---------|---------------|
> | **Skills** | Basic toolkit | Called by upper layers |
> | **Workflows** | Exam procedures / SOP | Human says "start" or cron triggers |
>
> See also: [Skills](../.claude/skills/README.md).

## Workflow Catalog

| Workflow | Description | Trigger |
|----------|-------------|---------|
| [daily-pr-review](daily-pr-review.md) | Review all open PRs without `ai-reviewed` label | CronTask daily / user request |
| [periodic-tiny-refactor](periodic-tiny-refactor.md) | One small safe refactoring in kubeopencode | CronTask every 3 days / user request |
| [weekly-opencode-update](weekly-opencode-update.md) | Check & update OpenCode version in agents/Makefile | CronTask weekly / user request |
| [weekly-fix-vulnerabilities](weekly-fix-vulnerabilities.md) | Fix open Dependabot alerts via pnpm overrides / go get | CronTask weekly / user request |

Workflow files are the **single source of truth** for scheduled task procedures. CronTask descriptions reference workflow files instead of embedding instructions — updating a workflow only requires a git commit, not a `kubectl apply`.

## Adding a New Workflow

1. Create `workflows/<workflow-name>.md` with the workflow phases
2. Create supporting sub-docs in `workflows/<workflow-name>/` if needed
3. Update this table
4. Update the Documentation Index in root `README.md`
5. Open a PR
