# Docs

Reference documentation for the KubeOpenCode Agent. These files are loaded **on demand** by skills and workflows — not all at once.

## Progressive Disclosure

- Top-level docs are **indexes** with links to detailed sub-files
- Load sub-files only when the task requires that specific knowledge
- See [README.md](../README.md#progressive-disclosure-critical) for the full principle

## Document Index

### Agent Capabilities

| Document | Description |
|----------|-------------|
| *(docs will be added as the agent's capabilities grow)* | |

### Site Reliability Engineering (SRE)

| Document | Description |
|----------|-------------|
| [sre/README.md](sre/README.md) | SRE overview, current cluster status, and architecture |
| [sre/slos.md](sre/slos.md) | SLO/SLI/SLA definitions and error budget policy |
| [sre/observability.md](sre/observability.md) | Observability stack (Prometheus + Grafana on K3s) |
| [sre/alerting.md](sre/alerting.md) | Alerting rules and Slack integration |
| [sre/postmortems/](sre/postmortems/) | Postmortem reports for incidents |
| [sre/runbooks/](sre/runbooks/) | Operational runbooks for common issues |
