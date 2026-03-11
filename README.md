# KubeOpenCode Dogfooding

Deployment configurations for the [KubeOpenCode](https://github.com/kubeopencode/kubeopencode) dogfooding environment, where KubeOpenCode is used to automate tasks on its own repository.

## Repo as Agent

This repo **is** the agent. The `.claude/skills/` directory defines all capabilities. Adding a new capability = adding a SKILL.md file via PR. No more remembering which agent does what.

```
.claude/
├── CLAUDE.md                      # Base agent identity & shared rules
└── skills/
    ├── github-respond/SKILL.md    # GitHub @mention handling (Q&A + code changes)
    ├── pr-review/SKILL.md         # Automated PR code review
    ├── tiny-refactor/SKILL.md     # Automated code refactoring
    ├── opencode-update/SKILL.md   # Dependency version checking
    └── slack-respond/SKILL.md     # Slack bot responses
```

## Structure

```
dogfooding/
├── base/           # Core resources (unified agent, RBAC, namespace)
├── scheduled/      # CronJobs (PR review, tiny refactor, opencode update)
└── slack/          # Slack bot integration (socket-mode gateway)
```

See [`dogfooding/README.md`](dogfooding/README.md) for detailed architecture and deployment instructions.
