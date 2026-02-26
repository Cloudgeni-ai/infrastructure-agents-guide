# Infrastructure Agents Guide

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Code License: MIT](https://img.shields.io/badge/Code_License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Chapters: 13](https://img.shields.io/badge/Chapters-13-green.svg)](#guide-structure)

> How to design, build, and operate AI agents for infrastructure teams — safely.

AI agents can write IaC, fix compliance findings, detect drift, review PRs, and respond to incidents — all autonomously. But autonomy without guardrails is a liability. Agents that can `terraform apply` can also `terraform destroy`. Agents that read configs can leak secrets. Agents that loop can burn budgets.

**This guide covers every architectural decision** you need to make when building infrastructure agents — with real patterns, code snippets, multiple alternatives, and the risk framework to evaluate your choices.

---

## Who This Is For

- **Platform engineers** evaluating whether to build or buy agent capabilities
- **SREs** designing safe automation for incident response and remediation
- **DevOps leads** building self-service IaC platforms
- **Engineering leaders** who need a reviewed architecture for AI-driven infrastructure

---

## Guide Structure

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 1 | [Architecture Overview](./01-architecture.md) | The six planes of an infra-agent system |
| 2 | [Agent Runtime & Orchestration](./02-agent-runtime.md) | Task queuing, worker isolation, consumer groups |
| 3 | [Tools, CLIs & Skills](./03-tools-skills.md) | CLI tooling, skill systems, MCP, and capability management |
| 4 | [Sandboxed Execution](./04-sandboxed-execution.md) | Container isolation with Docker, Modal, Azure Container Apps |
| 5 | [Credential Management](./05-credential-management.md) | Short-lived tokens, vault patterns, blast radius control |
| 6 | [The Data Plane](./06-data-plane.md) | Infrastructure knowledge layer, resource graphs, context serialization |
| 7 | [Change Control & GitOps](./07-change-control.md) | PR-based workflows, drift verification, validation loops |
| 8 | [Policy & Guardrails](./08-policy-guardrails.md) | Tool restrictions, approval gates, autonomy tiers |
| 9 | [Observability & Audit](./09-observability.md) | OpenTelemetry, action trails, debugging agent failures |
| 10 | [Autonomous Operations & Notifications](./10-autonomy-notifications.md) | Scheduling, autonomous agents, notification routing, escalation chains |
| 11 | [Testing & Hardening](./11-testing-hardening.md) | Trajectory tests, prompt injection defense, security benchmarks |
| 12 | [UX & Usability](./12-ux-usability.md) | Multi-tenancy, RBAC, onboarding, team collaboration, error prevention |
| 13 | [Risk Framework & Checklists](./13-risk-framework.md) | Decision matrices, compliance mapping, go-live checklists |

---

## Core Principles

1. **Agents Never Deploy Directly** — Every infrastructure change flows through a pull request. The agent produces diffs, not deployments.
2. **Least Privilege by Default** — Agents get the minimum credentials and tool access needed. Privileges are scoped, time-limited, and auditable.
3. **Observability Is Not Optional** — Every tool call, credential request, and decision point is logged with correlation IDs.
4. **Fail Safe, Not Fail Open** — When in doubt, the agent stops and asks a human. Timeouts and policy gates are structural — not suggestions.
5. **The Agent Is Not Special** — Agent-initiated changes go through the same review, CI, and deployment pipelines as human-initiated changes.

---

## Quick Start: Mental Model

```
┌───────────────────────────────────────────────────────────┐
│                    YOUR INFRASTRUCTURE                    │
│  AWS / Azure / GCP / OCI    Terraform / Bicep / Pulumi    │
│  GitHub / GitLab / ADO      Prowler / Checkov / Custom    │
└────────────────────────────┬──────────────────────────────┘
                             │
                  ┌──────────▼──────────┐
                  │    POLICY PLANE     │  ← What agents CAN do
                  │  (rules, approvals) │
                  └──────────┬──────────┘
                             │
            ┌────────────────▼────────────────┐
            │          AGENT RUNTIME          │
            │  Skills · Tool Access · Session │  ← How agents DO it
            │  Credentials · Sandboxing       │
            └────────────────┬────────────────┘
                             │
                  ┌──────────▼──────────┐
                  │   CHANGE CONTROL    │  ← How changes LAND
                  │  (PRs, validation)  │
                  └──────────┬──────────┘
                             │
                  ┌──────────▼──────────┐
                  │    OBSERVABILITY    │  ← How you SEE it
                  │  (traces, alerts)   │
                  └─────────────────────┘
```

---

## Alternatives Covered

This guide doesn't prescribe a single stack. For each architectural layer, we cover multiple approaches:

| Layer | Options Covered |
|-------|----------------|
| **LLM Runtime** | Claude Code SDK, OpenAI Assistants, LangChain/LangGraph, direct API |
| **Task Queue** | Redis Streams, BullMQ, AWS SQS, RabbitMQ, Temporal |
| **Sandboxing** | Docker, Modal, Azure Container Apps Jobs, AWS Lambda, Firecracker |
| **Credential Store** | HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, 1Password |
| **Change Control** | GitHub Actions, GitLab CI, Azure Pipelines, Atlantis, Spacelift |
| **Observability** | OpenTelemetry + Grafana, Datadog, Dash0, New Relic |
| **Notifications** | Slack, Microsoft Teams, PagerDuty, Opsgenie, email, webhooks |
| **Scheduling** | Cron (systemd/k8s), Temporal, AWS EventBridge, Azure Timer Triggers |
| **State Storage** | PostgreSQL, Redis, Azure Blob, S3, SQLite |

---

## Contributing

Found an error? Have a better pattern? Contributions are welcome.

- **Issues** — Open an issue for questions, suggestions, or corrections
- **Pull Requests** — Submit a PR for content improvements or new examples
- **Discussions** — Use GitHub Discussions for broader architectural questions

Please keep contributions focused on patterns and architecture — not vendor-specific marketing.

---

## About

This guide is built by the team at **[Cloudgeni](https://cloudgeni.ai)**, where we design, build, and operate autonomous infrastructure agents in production across AWS, Azure, GCP, and OCI for enterprise teams.

Every pattern here comes from running these systems in production. We open-sourced it because we kept answering the same architectural questions — writing them down once seemed more useful.

---

## License

The guide text is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt it, share it — just give credit.

Code snippets are released under [MIT](https://opensource.org/licenses/MIT).
