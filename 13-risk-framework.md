# Chapter 13: Risk Framework & Checklists

> Decision matrices, compliance mapping, and go-live checklists for infrastructure agents.

---

## Risk-to-Mitigation Matrix

Every risk in the report mapped to concrete mitigations covered in this guide:

| Risk | Why Agents Amplify It | Infra Failure Example | Mitigations | Guide Chapter |
|------|----------------------|----------------------|-------------|---------------|
| **Prompt injection** | Language is the control channel; agents read untrusted content | Ticket injects "run terraform destroy" | Sandboxing, tool allow-lists, structured intent, approval gates | Ch 4, 7 |
| **Over-privileged access** | Agents blur intent and execution; permissions default to broad | Agent with admin credentials deletes wrong resource | Least privilege, JIT credentials, scoped sandboxes, tool restrictions | Ch 5, 7 |
| **Secret leakage** | Agents read configs/logs; frameworks serialize state | LLM output includes API keys; malware targets agent dirs | Short-lived tokens, redaction, vault-backed retrieval, no secrets in prompts | Ch 5, 8 |
| **Hallucinated remediation** | Model fills gaps under uncertainty; tool calls make it real | Agent misdiagnoses root cause and applies wrong config | Pre-PR validation loops, plan verification, max iterations, HITL checkpoints | Ch 6, 7 |
| **Cost/latency runaway** | Multi-step loops; long context; agent fan-out | Agent loops on logs, burns $10K in tokens, delays humans | Hard budgets (time, tokens, tools), stop conditions, queue depth limits | Ch 2, 7 |
| **Poor observability** | Non-determinism; hidden reasoning; distributed tool calls | "Why did the agent create that PR?" can't be answered | Action trails, correlation IDs, OTel instrumentation, structured logs | Ch 8 |
| **Malicious skills/plugins** | Skills = markdown + code; documentation becomes executable | Malicious skill exfiltrates credentials | Skill allow-lists, no third-party skills without review, provenance checks | Ch 3 |
| **Silent failures** | Autonomous agents run without supervision | Drift scan fails for weeks, nobody notices | Notifications, daily digests, stuck run watchdog, escalation chains | Ch 9, 10 |

---

## Autonomy Decision Matrix

Use this to determine the right autonomy level for each use case:

```
                        Low Impact              High Impact
                    ┌─────────────────────┬─────────────────────┐
                    │                     │                     │
  High Confidence   │  Tier 2: DRAFT      │  Tier 3: SANDBOX    │
  (well-understood  │  Auto-create PRs    │  Execute in sandbox │
   problem, good    │  No approval needed │  + human approval   │
   training data)   │                     │                     │
                    │  Examples:          │  Examples:          │
                    │  - Encryption fixes │  - Multi-resource   │
                    │  - Tag enforcement  │    drift fixes      │
                    │  - Naming fixes     │  - Cross-env changes│
                    ├─────────────────────┼─────────────────────┤
                    │                     │                     │
  Low Confidence    │  Tier 1: RECOMMEND  │  Tier 0: OBSERVE    │
  (novel problem,   │  Suggest changes    │  Read-only analysis │
   ambiguous req)   │  Human decides      │  Human acts         │
                    │                     │                     │
                    │  Examples:          │  Examples:          │
                    │  - Cost optimization│  - Incident triage  │
                    │  - Architecture recs│  - Root cause       │
                    │  - Migration plans  │    analysis         │
                    └─────────────────────┴─────────────────────┘
```

### Tier Progression

Start every new use case at Tier 0. Promote based on evidence:

```
Tier 0 (OBSERVE)
  │ Evidence: Agent analysis matches human analysis 90%+ of the time
  ▼
Tier 1 (RECOMMEND)
  │ Evidence: Agent recommendations accepted by humans 80%+ of the time
  ▼
Tier 2 (DRAFT)
  │ Evidence: Agent PRs merged without modification 70%+ of the time
  ▼
Tier 3 (SANDBOX EXECUTE)
  │ Evidence: Sandbox runs succeed without human intervention 95%+
  ▼
Tier 4 (PROD WITH GATES)
  │ Only for well-understood, low-blast-radius operations
  │ Always with human approval gate
```

---

## Compliance Mapping

### NIST AI Risk Management Framework (AI RMF)

| NIST AI RMF Function | What It Means | How This Guide Addresses It |
|----------------------|---------------|---------------------------|
| **GOVERN** | Establish AI risk governance | Autonomy tiers, policy engine, org-level policies (Ch 7) |
| **MAP** | Understand AI risks in context | Risk matrix above, agent-specific threat modeling |
| **MEASURE** | Assess and track risks | Observability, action trails, trajectory tests (Ch 8, 12) |
| **MANAGE** | Treat and monitor risks | Guardrails, sandboxing, credential isolation, notifications (Ch 4-7, 10) |

### OWASP LLM Top 10 Coverage

| OWASP Risk | Relevant Chapters |
|-----------|-------------------|
| LLM01: Prompt Injection | Ch 4 (Sandbox), Ch 7 (Policy), Ch 12 (Testing) |
| LLM02: Insecure Output Handling | Ch 6 (PR validation), Ch 8 (Redaction) |
| LLM03: Training Data Poisoning | Out of scope (use trusted LLM providers) |
| LLM04: Model Denial of Service | Ch 7 (Budget limits), Ch 2 (Queue management) |
| LLM05: Supply Chain Vulnerabilities | Ch 3 (Skill system, no untrusted skills) |
| LLM06: Sensitive Information Disclosure | Ch 5 (Credentials), Ch 8 (Redaction) |
| LLM07: Insecure Plugin Design | Ch 3 (Skills as files, typed interfaces) |
| LLM08: Excessive Agency | Ch 7 (Tool restrictions, autonomy tiers) |
| LLM09: Overreliance | Ch 6 (Human review), Ch 7 (HITL checkpoints) |
| LLM10: Model Theft | Out of scope (use hosted LLM APIs) |

### SOC 2 Considerations

| SOC 2 Criteria | Agent-Relevant Control | Implementation |
|---------------|----------------------|----------------|
| CC6.1 - Logical access | Agent credential management | Short-lived tokens, scoped permissions (Ch 5) |
| CC6.3 - Authorized access | Tool allow/deny lists | Policy engine, tier-based restrictions (Ch 7) |
| CC7.2 - Monitoring | Agent activity monitoring | Action trails, dashboards, alerts (Ch 8, 10) |
| CC8.1 - Change management | PR-based change control | Agent changes go through standard CI/CD (Ch 6) |

---

## Go-Live Checklist

### Phase 1: Foundation (Week 1-2)

```
INFRASTRUCTURE
[ ] Task queue deployed and tested (Redis Streams / SQS / etc.)
[ ] Worker deployment pipeline established
[ ] Sandbox containers built with IaC tools pre-installed
[ ] Metadata endpoint blocked in sandbox networking
[ ] Internal server (capability bridge) running

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
[ ] Skill files written (SKILL.md + client.ts)
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

MONITORING
[ ] Stuck run watchdog active
[ ] PEL recovery on worker startup
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
[ ] Session persistence working (git bundles + session files)
[ ] Session resume after worker restart tested
[ ] Follow-up messages (WAITING_INPUT) flow tested
[ ] Message queuing for busy agents

GOVERNANCE
[ ] Organization-level policies defined and injected
[ ] Autonomy tiers assigned per use case
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

## Summary

Building infrastructure agents safely requires disciplined engineering across six planes:

1. **Ingestion** — how work enters the system (webhooks, schedules, chat, APIs)
2. **Policy** — what agents are allowed to do (tiers, tool lists, budgets)
3. **Execution** — how agents run in isolation (sandboxed containers, stateless workers)
4. **Integration** — how agents access external systems (short-lived credentials, typed skills)
5. **Change Control** — how changes land (PRs, validation loops, CI/CD)
6. **Observability** — how you see what happened (action trails, OTel, notifications)

Get these right, and you have a system that **multiplies your team's capacity** without multiplying your risk.

Get them wrong, and you have the incidents documented in the report: malicious skills, credential theft, runaway costs, and enterprise bans.

The difference is architecture.

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
