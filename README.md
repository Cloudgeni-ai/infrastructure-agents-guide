# Infrastructure Agents Guide

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Code License: MIT](https://img.shields.io/badge/Code_License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
[![Chapters: 13](https://img.shields.io/badge/Chapters-13-green.svg)](#guide-structure)

> How to design, build, and operate AI agents for infrastructure teams — safely.

AI agents can write IaC, fix compliance findings, detect drift, review PRs, and respond to incidents — all autonomously. Autonomous here means the agent is capable of doing the whole job, and your team decides how much of it the agent is allowed to do. Autonomy without guardrails is a liability. Agents that can `terraform apply` can also `terraform destroy`. Agents that read configs can leak secrets. Agents that loop can burn budgets.

**This guide covers every architectural decision** you need to make when building infrastructure agents — with real patterns, code snippets, multiple alternatives, and the risk framework to evaluate your choices.

If you're looking for the broader, non-infrastructure version, see the [Enterprise Agent Guide](https://github.com/Cloudgeni-ai/enterprise-agent-guide).

If you want to see these patterns running, look at **[Opengeni](https://opengeni.ai)** ([source](https://github.com/Cloudgeni-ai/opengeni), Apache-2.0), the open agentic runtime we built while writing this guide and the one Cloudgeni's own infrastructure agents run on. Every chapter ends with a short *In Opengeni* section that points to the matching part of it.

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
| 2 | [Agent Runtime & Orchestration](./02-agent-runtime.md) | LLM runtimes (Claude Agent SDK, OpenAI, LangChain, custom), task queuing, worker isolation |
| 3 | [Tools, CLIs & Skills](./03-tools-skills.md) | CLI tooling, skills, managed subagents, MCP, and capability management |
| 4 | [Sandboxed Execution](./04-sandboxed-execution.md) | Container isolation, managed sandbox APIs, egress controls |
| 5 | [Credential Management](./05-credential-management.md) | Short-lived tokens, vault patterns, blast radius control |
| 6 | [The Data Plane](./06-data-plane.md) | Infrastructure knowledge layer, resource graphs, context serialization |
| 7 | [Change Control & GitOps](./07-change-control.md) | PR-based workflows, drift verification, validation loops |
| 8 | [Policy & Guardrails](./08-policy-guardrails.md) | Tool restrictions, autonomy ceilings, governed merge/apply gates, receipts |
| 9 | [Observability & Audit](./09-observability.md) | OpenTelemetry, action trails, debugging agent failures |
| 10 | [Autonomous Operations & Notifications](./10-autonomy-notifications.md) | The autonomy dial, signals and goals, running unattended agents, notifications |
| 11 | [Testing & Hardening](./11-testing-hardening.md) | Trajectory tests, prompt injection defense, security benchmarks |
| 12 | [UX, Usability & Team Onboarding](./12-ux-usability.md) | Multi-tenancy, RBAC, onboarding, team collaboration, error prevention |
| 13 | [Risk Framework & Checklists](./13-risk-framework.md) | Autonomy promotion ladder, compliance and regulation mapping, go-live checklists |

---

## Core Principles

1. **Agents Default to PR-First Change Control** — Every infrastructure change flows through a pull request. The agent produces diffs, not deployments. When a team lets eligible changes land without a human click, a server-owned gate merges the PR and the normal pipeline applies it; the agent itself never merges or applies. If you support direct execution at all, treat it as a separate break-glass architecture with separate credentials, approval paths, and audit controls.
2. **Least Privilege by Default** — Agents get the minimum credentials and tool access needed. Privileges are scoped, time-limited, and auditable.
3. **Observability Is Not Optional** — Every tool call, credential request, and decision point is logged and traced end-to-end.
4. **Fail Safe, Not Fail Open** — When in doubt, the agent stops and asks a human. Timeouts and policy gates are structural — not suggestions.
5. **The Agent Is Not Special** — Agent-initiated changes go through the same review, CI, and deployment pipelines as human-initiated changes.
6. **Autonomy Is a Dial, Earned on Evidence** — How far an agent may go is a setting the team chooses per use case, not a switch the agent flips. Start low, promote one action class at a time on measured results, and keep a tested stop button. Whatever the level, a person should end up with one decision: accept this change or not.

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
                  │ (autonomy, gates)   │
                  └──────────┬──────────┘
                             │
            ┌────────────────▼────────────────┐
            │          AGENT RUNTIME          │
            │  Skills · Subagents · Tools     │  ← How agents DO it
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
| **LLM Runtime** | Claude Agent SDK / Managed Agents, OpenAI Agents SDK / Sandbox Agents / Codex CLI, LangChain/LangGraph, direct API, open agent platforms (Opengeni) |
| **Agent Roles** | Managed/project subagents, app-level agent configs, plugins |
| **Task Queue** | Redis Streams, BullMQ, AWS SQS, RabbitMQ, Temporal |
| **Sandboxing** | Docker/Kubernetes, Modal, E2B, Daytona, Vercel Sandbox, Cloudflare Sandbox, Northflank, Azure Dynamic Sessions, AWS AgentCore, Cloud Run Sandboxes |
| **Credential Store** | HashiCorp Vault, AWS Secrets Manager, Azure Key Vault, 1Password |
| **Change Control** | GitHub Actions, GitLab CI, Azure Pipelines, Atlantis, Spacelift |
| **Observability** | OpenTelemetry + GenAI semantic conventions, Grafana, Datadog, Dash0, New Relic |
| **Interoperability** | MCP (stateless core + extensions), A2A 1.0 |
| **Notifications** | Slack, Microsoft Teams (Workflows), PagerDuty, Jira Service Management, email, webhooks |
| **Scheduling** | Cron (systemd/k8s), Temporal, AWS EventBridge Scheduler, Azure Timer Triggers, durable agent goals |
| **State Storage** | PostgreSQL, Redis, Azure Blob, S3, SQLite |

---

## See It Running: Opengeni

[Opengeni](https://opengeni.ai) is the reference implementation of this guide. It is not the agent; it is everything an agent needs around it: sessions that run for hours or days and survive restarts, a replayable event log, human approvals, governed credentials and knowledge, and a choice of managed sandboxes or your own machines. It grew out of running agents against production cloud infrastructure at Cloudgeni, and Cloudgeni's agents run on it today. You can use it at [app.opengeni.ai](https://app.opengeni.ai) or self-host it with the Helm chart and reference Terraform for Azure, AWS and GCP.

| Chapter | Where to look in Opengeni |
|---------|---------------------------|
| 1 Architecture | [Architecture reference](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/architecture.md), [product integration](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/product-integration.md) |
| 2 Agent runtime | [Run lifecycle](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/run-lifecycle.md), [context compaction](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/context-compaction.md), [model providers](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/model-providers.md) |
| 3 Tools & skills | [Capability catalog](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/capabilities.md), [MCP surfaces](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/mcp-surfaces.md), [per-session MCP servers](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/session-mcp-servers.md) |
| 4 Sandboxing | [Compute backend](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/compute-backend.md), [Connected Machines](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/connected-machines.md), [Sandbox Environments](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/rigs.md) |
| 5 Credentials | [Credential taxonomy](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/credentials.md), [secrets](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/secrets.md), [workspace credential provider](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/workspace-integrations.md) |
| 6 Data plane | [Knowledge and Agent learning](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/knowledge.md) |
| 8 Policy | [Tool approvals](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/tool-approvals.md), [human input](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/human-input.md), [task usage limits](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/session-usage-limits.md) |
| 9 Observability | [Application observability](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/application-observability.md) |
| 10 Autonomy | [Session goals](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/goals.md), [scheduled task access](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/scheduled-task-access.md), [workspace pause timers](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/workspace-pause-timers.md) |
| 11 Testing | [Agent behavior eval](https://github.com/Cloudgeni-ai/opengeni/blob/main/scripts/agent-behavior-eval/README.md) |
| 12 UX | [Product integration](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/product-integration.md), [organization tenancy](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/organization-tenancy.md) |
| 13 Risk | [EU-only inference](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/eu-only-inference.md), [deployment](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/deployment.md) |

The guide stays vendor-neutral: every pattern is described so you can build it on your own stack. Opengeni is there for when you would rather not.

---

## What Changed in October 2026

This edition folds in what we learned from running autonomous agents in production between August and October 2026:

- **Autonomy, rewritten.** The three different tier tables in earlier chapters are replaced by one model: four levels defined by what a person still decides (Reactive, Propose, Governed, Goal-driven), enforced by runtime ceilings (Off, Observe, Propose, Governed) with a production opt-in and an emergency stop. See Chapters [1](./01-architecture.md#core-concept-the-autonomy-dial), [8](./08-policy-guardrails.md#autonomy-ceilings-in-practice), [10](./10-autonomy-notifications.md#autonomy-is-a-dial) and [13](./13-risk-framework.md#autonomy-decision-matrix).
- **Governed actions.** How eligible changes can land without a human click, through server-owned merge and apply gateways with blast-radius limits, verification, exact rollback and append-only receipts ([Chapter 8](./08-policy-guardrails.md#governed-actions)).
- **Signals, goals and running unattended.** Signal eligibility rules, the goal lifecycle, cadence budgets, one owner per goal across replicas, and the identity problems unattended agents run into ([Chapter 10](./10-autonomy-notifications.md#from-signals-to-goals)), each with the production incident that taught it.
- **Opengeni references.** Every chapter now ends with an *In Opengeni* section pointing to the matching implementation, and the README maps chapters to its docs ([See It Running](#see-it-running-opengeni)).
- **Operational lessons** across credentials, sandboxing, change control, observability and testing, plus corrections where the landscape moved (retired models, MCP server deprecations, OWASP 2025, Rego v1, Teams webhooks, the EU AI Act timeline).

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

We also released **[Opengeni](https://opengeni.ai)** as the concrete implementation of this guide: an open, self-hostable agentic runtime with durable sessions, approvals, governed credentials and memory, source available at [Cloudgeni-ai/opengeni](https://github.com/Cloudgeni-ai/opengeni) under Apache-2.0.

---

## License

The guide text is released under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use it, adapt it, share it — just give credit.

Code snippets are released under [MIT](https://opensource.org/licenses/MIT).
