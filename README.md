# Building Infrastructure Agents: The Definitive Guide

> How to design, build, and operate AI agents for infrastructure teams — safely.

**By [Cloudgeni](https://cloudgeni.ai)** — the team that built and operates infrastructure agents in production.

---

## Why This Guide Exists

AI agents are transforming how infrastructure teams work. They can write IaC, fix compliance findings, detect drift, review PRs, and respond to incidents — all autonomously.

But autonomy without guardrails is a liability. Agents that can `terraform apply` can also `terraform destroy`. Agents that read configs can leak secrets. Agents that loop can burn budgets. The industry has learned this the hard way through malicious skill marketplaces, enterprise bans, framework CVEs, and runaway cost incidents.

**This guide is the missing manual.** It covers every architectural decision you need to make when building infrastructure agents — with real patterns, code snippets, alternatives, and the risk framework to evaluate your choices.

Every pattern here is battle-tested in production at Cloudgeni, where we operate autonomous agents that manage infrastructure across AWS, Azure, GCP, and OCI for enterprise teams.

---

## Who This Is For

- **Platform engineers** evaluating whether to build or buy agent capabilities
- **SREs** designing safe automation for incident response and remediation
- **DevOps leads** building self-service IaC platforms
- **Engineering leaders** who need a defensible architecture for AI-powered infrastructure

---

## Guide Structure

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 1 | [Architecture Overview](./01-architecture.md) | The six planes of a robust infra-agent system |
| 2 | [Agent Runtime & Orchestration](./02-agent-runtime.md) | Task queuing, worker isolation, consumer groups |
| 3 | [Skill & Tool System](./03-skill-system.md) | How agents gain capabilities safely |
| 4 | [Sandboxed Execution](./04-sandboxed-execution.md) | Container isolation with Docker, Modal, Azure Container Apps |
| 5 | [Credential Management](./05-credential-management.md) | Short-lived tokens, vault patterns, blast radius control |
| 6 | [Change Control & GitOps](./06-change-control.md) | PR-based workflows, drift verification, validation loops |
| 7 | [Policy & Guardrails](./07-policy-guardrails.md) | Tool restrictions, approval gates, autonomy tiers |
| 8 | [Observability & Audit](./08-observability.md) | OpenTelemetry, action trails, debugging agent failures |
| 9 | [Scheduled & Autonomous Operations](./09-scheduled-autonomous.md) | Cron-triggered scans, continuous drift detection, autonomous remediation |
| 10 | [Notifications & Alerting](./10-notifications.md) | Slack/Teams/email integration, escalation chains, status dashboards |
| 11 | [Session & State Management](./11-session-state.md) | Multi-turn conversations, persistence, session forking |
| 12 | [Testing & Hardening](./12-testing-hardening.md) | Trajectory tests, prompt injection defense, security benchmarks |
| 13 | [UX & Usability](./13-ux-usability.md) | Multi-tenancy, RBAC, onboarding, team collaboration, error prevention |
| 14 | [Risk Framework & Checklists](./14-risk-framework.md) | Decision matrices, compliance mapping, go-live checklists |

---

## Core Principles

Before diving into architecture, these principles guide every design decision:

### 1. Agents Never Deploy Directly
Every infrastructure change flows through a pull request. The agent produces diffs, not deployments. Humans (or automated CI) decide when to merge and apply.

### 2. Least Privilege by Default
Agents get the minimum credentials and tool access needed for their task. Privileges are scoped, time-limited, and auditable.

### 3. Observability Is Not Optional
Every tool call, credential request, and decision point is logged with correlation IDs. If you can't explain why the agent did something, you can't trust it.

### 4. Fail Safe, Not Fail Open
When in doubt, the agent stops and asks a human. Timeouts, iteration limits, and policy gates are structural — not suggestions the model can ignore.

### 5. The Agent Is Not Special
Agent-initiated changes go through the same review, CI, and deployment pipelines as human-initiated changes. No shortcuts.

---

## Quick Start: Mental Model

```
┌─────────────────────────────────────────────────────────────┐
│                    YOUR INFRASTRUCTURE                       │
│  AWS / Azure / GCP / OCI    Terraform / Bicep / Pulumi      │
│  GitHub / GitLab / ADO      Prowler / Checkov / Custom      │
└──────────────────────────────┬──────────────────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │   POLICY PLANE      │  ← What agents CAN do
                    │  (rules, approvals) │
                    └──────────┬──────────┘
                               │
              ┌────────────────▼────────────────┐
              │       AGENT RUNTIME             │
              │  ┌─────────┐  ┌──────────────┐  │
              │  │  Skills  │  │  Tool Access  │ │  ← How agents DO it
              │  └─────────┘  └──────────────┘  │
              │  ┌─────────┐  ┌──────────────┐  │
              │  │ Session  │  │  Credentials  │ │
              │  └─────────┘  └──────────────┘  │
              └────────────────┬────────────────┘
                               │
                    ┌──────────▼──────────┐
                    │  CHANGE CONTROL     │  ← How changes LAND
                    │  (PRs, validation)  │
                    └──────────┬──────────┘
                               │
                    ┌──────────▼──────────┐
                    │  OBSERVABILITY      │  ← How you SEE it
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

## License

This guide is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt it, share it — just give credit.

The code snippets are released under [MIT](https://opensource.org/licenses/MIT).

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
