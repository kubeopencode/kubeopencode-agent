# Skills

Skills are the agent's **atomic capabilities** — reusable, composable building blocks. Each skill is a directory under `.claude/skills/` containing a `SKILL.md` with a step-by-step checklist.

Skills are called by upper layers (workflows, user requests, scheduled tasks). They do one thing well.

> **How skills differ from workflows:**
>
> | Layer | Analogy | Who initiates |
> |-------|---------|---------------|
> | **Skills** | Basic toolkit | Called by upper layers |
> | **Workflows** | Exam procedures / SOP | Human says "start" or cron triggers |
>
> See also: [Workflows](../../workflows/README.md).

## Skills Catalog

| Skill | Trigger | Description |
|-------|---------|-------------|
| [github-respond](github-respond/SKILL.md) | GitHub @mention | Answer questions, make code changes, create PRs |
| [slack-respond](slack-respond/SKILL.md) | Slack @mention/DM | Respond to Slack messages via slack-cli |

> **Note:** `pr-review`, `tiny-refactor`, and `opencode-update` have been migrated to
> [workflows](../../workflows/README.md) — they are scheduled multi-phase processes,
> not reactive skills.

## Adding a New Skill

1. Create `.claude/skills/<skill-name>/SKILL.md` with step-by-step checklist
2. Add optional script files (`.sh`, `.py`) as needed
3. Update this table
4. Update the Documentation Index in root `README.md` if the skill introduces new doc categories
5. Open a PR
