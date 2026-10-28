# Chapter 13: Risk Framework & Checklists

> Decision matrices, compliance mapping, and go-live checklists for infrastructure agents.

---

## Risk-to-Mitigation Matrix

Common risks of infrastructure agents, mapped to the mitigations covered in this guide:

| Risk | Why Agents Amplify It | Infra Failure Example | Mitigations | Guide Chapter |
|------|----------------------|----------------------|-------------|---------------|
| **Prompt injection** | Language is the control channel; agents read untrusted content | Ticket injects "run terraform destroy" | Sandboxing, tool allow-lists, structured intent, approval gates | Ch 4, 8 |
| **Over-privileged access** | Agents blur intent and execution; permissions default to broad | Agent with admin credentials deletes wrong resource | Least privilege, JIT credentials, scoped sandboxes, tool restrictions | Ch 5, 8 |
| **Secret leakage** | Agents read configs/logs; frameworks serialize state | LLM output includes API keys; malware targets agent dirs | Short-lived tokens, redaction, vault-backed retrieval, no secrets in prompts | Ch 5, 9 |
| **Hallucinated remediation** | Model fills gaps under uncertainty; tool calls make it real | Agent misdiagnoses root cause and applies wrong config | Pre-PR validation loops, plan verification, max iterations, HITL checkpoints | Ch 7, 8 |
| **Cost/latency runaway** | Multi-step loops; long context; agent fan-out | Agent loops on logs, burns $10K in tokens, delays humans | Hard budgets (time, tokens, tools), stop conditions, queue depth limits | Ch 2, 8 |
| **Poor observability** | Non-determinism; hidden reasoning; distributed tool calls | "Why did the agent create that PR?" can't be answered | Action trails, correlation IDs, OTel instrumentation, structured logs | Ch 9 |
| **Malicious skills/plugins** | Skills = markdown + code; documentation becomes executable | Malicious skill exfiltrates credentials | Skill allow-lists, no third-party skills without review, provenance checks | Ch 3 |
| **Silent failures** | Autonomous agents run without supervision | Drift scan fails for weeks, nobody notices | Failure-pattern alerts per launcher, canaries, daily digests, stuck run watchdog, escalation chains | Ch 9, 10 |
| **Runaway autonomy** | Many replicas, retries and self-scheduling multiply each other | One goal spawns dozens of sessions overnight | One owner per goal, retry budget tied to the source observation, wake and token budgets | Ch 10 |
| **Authority creep** | Token scopes are broader than intent; settings change mid-run | A "propose-only" agent's token can also merge | Policy frozen per run, server-owned merge gateway, branch protection, production opt-in | Ch 5, 8 |
| **Untrustworthy signals** | Agents act on whatever the scanners say | A failed scan auto-resolves every finding; empty cost data reads as zero | Signal eligibility rules, "empty is unknown", scanner vs user resolution | Ch 6, 10 |
| **Orphaned ownership** | Unattended work outlives the person who set it up | A departed member's integration silently breaks reviews | Audited owner fallback, re-homing on member removal | Ch 5, 10 |

---

## Autonomy Decision Matrix

Use this to choose the starting level for a use case. The level describes what a person still decides ([Chapter 10](./10-autonomy-notifications.md#autonomy-is-a-dial)); the ceiling that enforces it is covered in [Chapter 8](./08-policy-guardrails.md#autonomy-ceilings-in-practice).

```
                        Reversible, bounded         Sensitive or wide blast radius
                    ┌──────────────────────────┬──────────────────────────────┐
                    │                          │                              │
  High Confidence   │  Level 2: GOVERNED       │  Level 1: PROPOSE            │
  (well-understood  │  Gates land eligible     │  Agent opens the PR,         │
   problem, strong  │  PRs; people handle      │  a person merges             │
   track record)    │  exceptions              │                              │
                    │  Examples:               │  Examples:                   │
                    │  - Tag enforcement       │  - Multi-resource drift fixes│
                    │  - Diagnostic settings   │  - Network rule changes      │
                    │  - Closing public access │  - Cross-environment changes │
                    ├──────────────────────────┼──────────────────────────────┤
                    │                          │                              │
  Low Confidence    │  Level 1: PROPOSE        │  Level 0: REACTIVE / OBSERVE │
  (novel problem,   │  (or Observe while you   │  Agent analyzes when asked   │
   ambiguous req)   │   build a track record)  │  or reports; a person acts   │
                    │  Examples:               │  Examples:                   │
                    │  - Cost optimization     │  - Incident triage           │
                    │  - Architecture recs     │  - Root cause analysis       │
                    │  - Migration plans       │  - Anything touching identity│
                    └──────────────────────────┴──────────────────────────────┘
```

Irreversible actions (deleting data stores, identity, secrets, state) stay with a person at every level.

### Climbing the Ladder

Autonomy is earned by measurement, never granted. Promote one use case and one action class at a time, in one environment at a time, and make promotion a configuration change rather than a code change. Don't skip rungs.

```
OBSERVE
  │ Evidence: agent analysis matches a human's 90%+ of the time
  ▼
RECOMMEND (reports with a suggested fix)
  │ Evidence: recommendations accepted 80%+ of the time
  ▼
PROPOSE (opens PRs)
  │ Evidence: PRs merged without modification 70%+ of the time
  ▼
GOVERNED · non-production (gates land eligible PRs; start in shadow mode)
  │ Evidence: 95%+ of governed changes verify cleanly, one action class at a time
  ▼
GOVERNED · production
    Requires: a non-production track record, a low blast radius,
    a tested rollback and a post-apply health check
```

Merge rate is the trust metric to watch at the Propose rung. Across our customers, 79% of agent-authored PRs are merged by the customers' own engineers, which is the kind of number that justifies trying Governed for the narrowest action classes, and not yet for everything.

Shadow mode is how you collect evidence for the Governed rungs without taking the risk: the gateway records "would have landed" receipts while a person still clicks merge, and you compare its decisions with theirs ([Chapter 8](./08-policy-guardrails.md#receipts)). Move a use case back down when its numbers drop; demotion should be as cheap as promotion.

---

## Compliance Mapping

### NIST AI Risk Management Framework (AI RMF)

| NIST AI RMF Function | What It Means | How This Guide Addresses It |
|----------------------|---------------|---------------------------|
| **GOVERN** | Establish AI risk governance | Autonomy levels and ceilings, named goal owners, policy engine, org-level policies (Ch 8, 10) |
| **MAP** | Understand AI risks in context | Risk matrix above, agent-specific threat modeling |
| **MEASURE** | Assess and track risks | Observability, action trails, trajectory tests (Ch 9, 11) |
| **MANAGE** | Treat and monitor risks | Guardrails, sandboxing, credential isolation, notifications (Ch 4-8, 10) |

### OWASP LLM Top 10 (2025) Coverage

| OWASP Risk | Relevant Chapters |
|-----------|-------------------|
| LLM01: Prompt Injection | Ch 4 (Sandbox), Ch 8 (Policy), Ch 11 (Testing) |
| LLM02: Sensitive Information Disclosure | Ch 5 (Credentials), Ch 9 (Redaction) |
| LLM03: Supply Chain | Ch 3 (Skill and MCP server vetting) |
| LLM04: Data and Model Poisoning | Ch 6 (Governed agent memory, trusted sources); model training is out of scope |
| LLM05: Improper Output Handling | Ch 7 (PR validation), Ch 8 (Pre-execution validation) |
| LLM06: Excessive Agency | Ch 8 (Tool restrictions, autonomy ceilings, governed gateways) |
| LLM07: System Prompt Leakage | Ch 5 (No secrets in prompts), Ch 8 (Policy lives in code, not the prompt) |
| LLM08: Vector and Embedding Weaknesses | Ch 6 (RAG and tenant-scoped retrieval) |
| LLM09: Misinformation | Ch 7 (Deterministic validation), Ch 10 (Trust the signal first) |
| LLM10: Unbounded Consumption | Ch 8 (Budget limits), Ch 10 (Wake and token budgets) |

### OWASP Top 10 for Agentic Applications

OWASP's agentic list (published December 2025) fits infrastructure agents more closely than the LLM list, because it is about what agents do rather than what models say. The risks that matter most here are goal hijacking through injected content (fence outside text as untrusted and freeze goals at approval, Ch 10, 11), tool misuse and privilege abuse (ceilings, frozen per-run policy and short-lived credentials, Ch 5, 8), memory poisoning (Ch 6), cascading failures across agents and replicas (one owner per goal, Ch 10), and rogue or unmonitored agents (receipts, failure-pattern alerts and a tested stop button, Ch 8, 9, 10).

### Accountability and Regulation

An agent can't be accountable, so someone always is. European regulation is explicit about who:

- **NIS2** (Article 20) makes management bodies approve cybersecurity risk measures and lets them be held liable for them. **DORA** (Article 5) does the same for financial entities' ICT risk. If an agent changes your infrastructure, management needs to be able to see who set its level, who owns its goals, and what it did.
- **The EU AI Act's human oversight requirements** (Article 14) ask for the ability to understand, override and stop a system, and warn explicitly about automation bias. A named owner per goal, receipts for every action and an emergency stop that has actually been tested map directly onto this. The AI Omnibus adopted in 2026 moved the obligations for Annex III high-risk systems to 2027-12-02; check whether your use case is in scope rather than assuming it is or isn't.
- **ISO/IEC 42001** (AI management systems) and the **NIST AI 600-1** generative AI profile are the management-system and risk-profile references auditors increasingly ask about.

### SOC 2 Considerations

| SOC 2 Criteria | Agent-Relevant Control | Implementation |
|---------------|----------------------|----------------|
| CC6.1 - Logical access | Agent credential management | Short-lived tokens, scoped permissions (Ch 5) |
| CC6.3 - Authorized access | Tool allow/deny lists | Policy engine, autonomy ceilings frozen per run (Ch 8) |
| CC7.2 - Monitoring | Agent activity monitoring | Action trails, dashboards, alerts (Ch 9, 10) |
| CC8.1 - Change management | PR-based change control | Agent changes go through standard CI/CD; governed changes carry receipts (Ch 7, 8) |

---

## Go-Live Checklist

### Phase 1: Foundation (Week 1-2)

```
INFRASTRUCTURE
[ ] Task queue deployed and tested (Redis Streams / SQS / etc.)
[ ] Worker deployment pipeline established
[ ] Sandbox containers built with IaC tools pre-installed
[ ] Metadata endpoint blocked in sandbox networking
[ ] Credential broker and tool access layer running

SECURITY
[ ] Credential broker implemented and tested
[ ] Short-lived token generation for each cloud provider
[ ] Secret storage configured (Vault / KMS / Key Vault)
[ ] Network egress rules applied to sandbox

OBSERVABILITY
[ ] OpenTelemetry instrumentation in place
[ ] Action trail event emission working
[ ] Log redaction configured
[ ] Basic dashboard (active sessions, success rate, queue depth)
```

### Phase 2: First Agent (Week 3-4)

```
AGENT CONFIGURATION
[ ] System prompt with hard rules defined
[ ] Effective agent/subagent definitions resolved by layer and logged
[ ] Skills/tools defined and tested
[ ] Tool allow/deny list configured
[ ] Max iteration and budget limits set

CHANGE CONTROL
[ ] PR creation workflow tested end-to-end
[ ] CI pipeline triggers from agent PRs
[ ] Branch naming convention established
[ ] PR body template includes validation results

TESTING
[ ] Unit tests for all skills pass
[ ] At least 3 trajectory tests covering happy path
[ ] At least 2 adversarial tests (prompt injection)
[ ] Manual end-to-end test with real infrastructure
```

### Phase 3: Autonomous Operations (Week 5-6)

```
SCHEDULING
[ ] Cron/EventBridge/Timer configured for recurring scans
[ ] Staggered dispatch prevents queue flooding
[ ] Missed run catch-up mechanism in place

NOTIFICATIONS
[ ] Slack/Teams integration configured
[ ] Notification routing rules defined
[ ] Escalation chain for critical failures
[ ] Daily digest scheduled

AUTONOMY
[ ] Every use case starts at Observe or Propose; levels recorded per use case
[ ] Autonomy ceiling starts at Off; changes are admin-only and audited
[ ] Emergency stop tested end to end
[ ] Signal eligibility rules and fan-out caps in place before goals are enabled
[ ] One owner per goal across replicas; retry budget tied to the source observation
[ ] Wake cadence and daily wake/token budgets configured

MONITORING
[ ] End-to-end canary per launcher type (chat, schedule, review, goal), leader-leased
[ ] Failure-pattern alerts (new error class, bursts, per-org failure rate)
[ ] Stuck/stalled task detection active
[ ] Unacknowledged message recovery on worker startup
[ ] Alert on agent failure rate > threshold
[ ] Alert on queue depth > threshold
[ ] Alert on token usage anomalies
```

### Phase 4: Scale (Week 7+)

```
MULTI-AGENT
[ ] Multiple agent types configured (remediation, drift, PR review)
[ ] Task type routing to specialized worker pools
[ ] Concurrent session handling tested under load

SESSION MANAGEMENT
[ ] Session persistence working (git bundles + session state)
[ ] Session resume after worker restart tested
[ ] Human-in-the-loop input flow tested end-to-end
[ ] Message queuing for busy agents

GOVERNANCE
[ ] Organization-level policies defined and injected
[ ] Managed agent definitions reviewed; project/user definitions cannot expand beyond enterprise policy
[ ] Promotion evidence reviewed before any use case moves to Governed
[ ] Named owner for every standing goal; owner-removal fallback tested
[ ] Monthly action trail review process established
[ ] Incident response plan for agent failures documented
```

---

## When NOT to Use an Agent

Not every infrastructure task needs an AI agent. Use simpler automation when:

| Scenario | Better Approach |
|----------|----------------|
| Same fix every time, no variation | Shell script / CI pipeline |
| Simple resource provisioning | Terraform modules / CDK constructs |
| Alert → restart service | PagerDuty automation / Lambda |
| Log rotation / cleanup | Cron job |
| Certificate renewal | cert-manager / ACM |

**Use agents when**: The task requires understanding context, making judgment calls, or adapting to variations. If you can write an `if/else` that covers all cases, you don't need an agent.

---

## Architecture Decision Record Template

When making architectural choices for your agent system, document them:

```markdown
# ADR-001: Task Queue Technology

## Status
Accepted

## Context
We need a task queue for dispatching agent work to stateless workers.
Requirements: consumer groups, message persistence, real-time output streaming.

## Decision
Redis Streams with consumer groups for dispatch, separate streams for output.

## Alternatives Considered
- **BullMQ**: Good Node.js integration but tied to Redis anyway
- **AWS SQS**: No consumer groups; would need additional pub/sub for streaming
- **Temporal**: Too heavy for our current scale; might revisit at 100+ concurrent agents

## Consequences
- Need Redis 5.0+ in production
- Must handle PEL recovery on worker startup
- Stream trimming needed to prevent unbounded memory growth
```

---

## In Opengeni

Two Opengeni features map directly onto the decisions in this chapter. [EU-only inference](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/eu-only-inference.md) keeps every model call for an organization on EU-processing routes, enforced on the server and failing closed, which settles the "where does our data go" question before the first run. And because Opengeni is Apache-2.0 with a Helm chart and reference Terraform for Azure, AWS and GCP ([deployment](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/deployment.md)), "can we switch, and can we run it ourselves" has a concrete answer.

---

## Summary

The six planes of an infrastructure agent system:

1. **Ingestion** — how work enters the system (webhooks, schedules, chat, APIs)
2. **Policy** — what agents are allowed to do (autonomy ceilings, governed gates, tool lists, budgets)
3. **Execution** — how agents run in isolation (sandboxed containers, stateless workers)
4. **Integration** — how agents access external systems (short-lived credentials, typed skills)
5. **Change Control** — how changes land (PRs, validation loops, CI/CD)
6. **Observability** — how you see what happened (action trails, OTel, notifications)

Get them wrong and you get the incidents this guide keeps returning to: malicious skills, credential theft, runaway costs, agents that fail silently, and enterprise bans. Get them right and you can turn the dial up one use case at a time, because the guardrails are what let you go faster.

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
