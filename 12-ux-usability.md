# Chapter 12: UX, Usability & Team Onboarding

> How to package infrastructure agents so everyone can adopt them safely — without reading a 13-chapter guide first.

---

## The Usability Problem

You've built the agent runtime, the sandbox, the credential broker, the policy engine, the observability stack. You now have a system that only you understand.

The hard part isn't making agents work. It's making them usable by a **platform team of 3 serving 50 engineers across 5 teams** without becoming a bottleneck, and without a junior developer accidentally giving an agent admin credentials to production.

---

## Design Principle: Make the Safe Path the Easy Path

Every UX decision should follow one rule:

> **The default behavior should be the secure behavior. Doing the wrong thing should require more effort than doing the right thing.**

Examples:
- New agents default to plan-only mode — getting apply access requires explicit escalation
- Credentials are scoped to one cloud account by default — cross-account access requires admin action
- Agent sessions are private by default — sharing requires explicit toggle
- PR review is per-repository opt-in — not a global flag

If you find yourself building "are you sure?" confirmation dialogs, the UX has already failed. The system should be designed so the dangerous action isn't reachable without deliberate, admin-level configuration.

---

## Multi-Tenancy: Organization as the Security Boundary

The organization is the **top-level isolation boundary**. Everything — credentials, repositories, agents, policies, sessions, variables — is scoped to an organization:

- A platform engineer at Company A **cannot** see Company B's infrastructure
- A team within an organization **can** share agent sessions with each other
- An admin for Org X **cannot** escalate to Org Y, even if they're the same person

```
┌────────────────────────────────────────────────────────────┐
│  Organization: "Acme Platform Team"                        │
│                                                            │
│  Cloud accounts: AWS Prod, AWS Staging, Azure Dev          │
│  Repositories:   infra-core, app-deploy, data-pipelines    │
│  Agents:         remediation, drift, PR review, devops     │
│  Policies:       "S3 uses org KMS key", "tag everything"   │
│  Members:        alice (admin), bob (member), carol        │
└────────────────────────────────────────────────────────────┘

  ← No data crosses this boundary →

┌────────────────────────────────────────────────────────────┐
│  Organization: "Acme App Team"                             │
│  (Separate credentials, repos, agents, policies)           │
└────────────────────────────────────────────────────────────┘
```

The key implementation rule: **every database query must include the organization filter**. Enforce this with middleware that validates org membership before any handler runs. Cross-org access should return 403, not 404 — don't leak existence.

Organizations can map to your users' VCS structure (GitHub organizations, GitLab groups) so repositories and team membership import automatically.

---

## RBAC: Simple Roles, Structural Safety

### Keep Roles Simple

Complex role hierarchies create confusion. Three roles cover 95% of needs:

| Role | Can Do | Cannot Do |
|------|--------|-----------|
| **Admin** | Manage members, integrations, credentials, policies, agent configs | Access other organizations |
| **Member** | Use agents, view sessions, create PRs, run scans, view findings | Modify integrations, manage members, change policies |
| **Viewer** | View dashboards, sessions, findings (read-only) | Trigger agents, modify anything |

### Where Safety Lives: Structural, Not Role-Based

The key insight: **don't rely on roles to prevent mistakes. Structure the system so mistakes are impossible.**

| Risk | Bad Approach (role-based) | Good Approach (structural) |
|------|--------------------------|---------------------------|
| Junior gives agent prod creds | "Only admins can select integrations" | Agent can only access integrations admin has pre-configured. There's no "enter credentials" field in the agent UI. |
| Developer enables apply mode | "Only admins can change agent tier" | Apply mode requires a separate credential set that members can't create. The agent config is admin-only. |
| New team member accesses wrong repo | "RBAC on repository access" | Repositories are per-org. The member only sees repos connected to their org. |
| Someone shares a session with secrets | "Warn users about sharing" | Log redaction is automatic. Sessions with credential requests have redacted output by default. |

---

## Onboarding: Time-to-Value in Minutes

### Start With Two Steps, Not Fifteen

Don't present a wizard. Start with exactly two things:

```
GET STARTED
You need two things to start using infrastructure agents:

  1. CONNECT CLOUD            2. CONNECT GIT REPO
     AWS / Azure /               GitHub / GitLab /
     GCP / OCI                   Azure DevOps
     [Quick Connect]             [Connect Provider]

After these, agents can analyze your infrastructure
and start creating PRs.
```

Everything else is progressive — shown only after the mandatory steps are complete.

### Quick Connect: No Credential Copy-Paste

The fastest path to value. Users should never manually enter access keys.

- **AWS**: One-click CloudFormation stack deploys a cross-account IAM role that trusts your platform. ~30 seconds.
- **Azure**: OAuth consent flow creates a service principal automatically. ~10 seconds.
- **GCP**: Workload Identity Federation or OAuth consent. No service account key files.

**Key principle**: Quick Connect should use standard delegation mechanisms (cross-account roles, OAuth, OIDC) that keep long-lived credentials out of human hands.

### Progressive Disclosure

After the two mandatory steps, reveal features based on what the user has unlocked:

```
Phase 1 (mandatory):  Connect cloud + Connect Git
Phase 2 (suggested):  Run first compliance scan → Run static analysis → Fix first finding
Phase 3 (optional):   Enable drift detection, automated PR review, invite team members
```

Show onboarding progress persistently until mandatory steps are complete, then hide it. Don't nag users who are past the basics.

---

## Self-Service Agent Interface

### Chat as the Primary Interface

The primary interface should feel like a conversation, not a CI/CD dashboard. Three selectors provide the context an agent needs:

```
[Agent: DevOps ▼]  [Repo: infra-core ▼]  [Cloud: AWS ▼]

┌─────────────────────────────────────────────────────┐
│  Move my-data-bucket to our org KMS key             │
│  (the encryption policy finding)                    │
│                                          [Send ->]  │
└─────────────────────────────────────────────────────┘
```

**Why this works**: Members don't need to know about credentials, pipelines, or agent configurations. The admin has already set up the integrations. The member just picks from dropdowns and asks a question.

### Prompt Discovery: Don't Make Users Guess

New users don't know what to ask. Provide categorized starting prompts:

- **Fix Compliance**: "Find my top 3 critical issues and fix them", "Remediate all HIGH severity findings"
- **Create Resources**: "Create an S3 bucket with versioning and encryption", "Set up a VPC with public and private subnets"
- **Refactor**: "Convert inline security groups to named resources", "Split this monolithic config into modules"
- **Analyze**: "Explain how my infrastructure is set up", "Show me all security groups with open SSH access"

### Deep-Link Support

Let agents be triggered from anywhere — compliance dashboards, Slack, CI/CD:

```
/agent-sessions?agent=remediation&repo=xxx&prompt=Fix+finding+ABC123
```

A "Fix with Agent" button on a compliance finding card can pre-populate agent, repo, and prompt — one click to start remediation.

---

## Session Sharing & Collaboration

### Default-Private, Opt-In Sharing

Agent sessions often contain infrastructure details that not everyone should see. Default to private, with an explicit toggle to share within the organization.

```
Session List:
┌──────────────────────────────────────────┐
│ ● Fix S3 encryption           Private    │
│ ● Drift remediation           Shared     │
│ ● VPC refactoring (Bob)       From Bob   │
│ ● Compliance scan review      Private    │
└──────────────────────────────────────────┘
```

Real-time multi-cursor collaboration is unnecessary — agent sessions are sequential (one user asks, agent responds). Instead, provide **sharing** (others view what happened), **forking** (continue from a specific point), and **session history** (audit trail for team review).

---

## Policies as Plain Language

From a usability point of view, the important choice is that teams write agent policies in plain language rather than Rego or Sentinel, so the person who wrote a rule, the person who reviews it, and the person debugging the agent all read the same text. Ship **policy templates** for common standards (encryption, tagging, network access) so teams don't start from a blank page, and show version history with diffs for every change. [Chapter 8](./08-policy-guardrails.md#organization-level-policies) covers how these policies are stored, injected, and turned into enforceable checks.

---

## Variable Scoping: Environment Separation Without Complexity

Variables (backend addresses, region defaults, environment config) follow a three-level override chain:

```
Repository scope    (highest priority — overrides everything)
  ↓
Integration scope   (overrides org level)
  ↓
Organization scope  (baseline defaults)
```

A platform team sets `TF_BACKEND_BUCKET=acme-terraform-state` at the org level. Individual teams override it per-repo if they use a different backend — without either team needing to understand the other's configuration. Secret variables are encrypted at rest and hidden in the UI after save.

---

## Per-Repository Agent Configuration

Not every feature should be on globally. Key per-repo controls:

- **Automated PR review**: Opt-in per repository. When enabled, the configured review agent is dispatched automatically on new PRs with IaC changes. Developers see review comments appear without configuring anything.
- **IaC pipelines**: Configured by admins (which CI system, which workflow, per-environment overrides). Members interact with pipelines through agents — "validate my changes" — not by editing pipeline definitions.

---

## Preventing Junior Developer Mistakes

### The Error-Prevention Hierarchy

From most to least effective:

```
1. MAKE IT IMPOSSIBLE  — Structure prevents the mistake entirely
2. MAKE IT HARD        — Extra steps required, admin approval needed
3. MAKE IT VISIBLE     — Clear warnings, confirmation dialogs
4. MAKE IT REVERSIBLE  — Easy rollback, audit trail
```

| Mistake | Prevention (Level) |
|---------|-------------------|
| Giving agent prod credentials | Cloud integrations are admin-configured. Members select from dropdown, never enter creds. **(Impossible)** |
| Running terraform apply directly | System prompts have hard rules against apply. Tool allow-list excludes apply. **(Impossible)** |
| Accessing another team's resources | Organization boundary isolates all data. No cross-org query path exists. **(Impossible)** |
| Pushing to main branch | Agent hard rules + branch protection rules. **(Hard)** |
| Sharing sessions with sensitive data | Default-private sessions + automatic log redaction. **(Hard)** |
| Forgetting to validate before PR | Agent's validation loop runs automatically. No skip option. **(Impossible)** |

API keys for programmatic access should be organization-scoped at creation time and cannot be reassigned to a different org.

---

## Notification Preferences

Notification fatigue kills adoption. Two levels of control:

- **Organization-level**: Admin configures Slack/Teams webhook once, toggles which event types go to the channel. Whole team benefits without individual setup.
- **Per-user**: Each member controls their own email digest frequency, quiet hours, and per-category toggles (security findings, remediation status, product updates).

---

## Explaining What Autonomous Agents Are Doing

Once agents run on their own, the UI's main job shifts from accepting requests to explaining state. People need to tell at a glance whether the agent is working, waiting for them, or stuck, and why. Four rules came out of our own production operation:

- **Show the agent's own reason when it pauses itself.** If an agent stops because it needs something, display its rationale in plain words ("Blocked: needs read access to the payments subscription"), not an internal status such as `goal_inactive`. An internal status tells the user something stopped; the rationale tells them what to do about it.
- **Distinguish who paused the work.** "The agent paused this goal" and "the controller paused this goal after three failed attempts" call for different responses. Label agent-initiated and controller-initiated pauses differently, and say which rule triggered a controller pause.
- **Keep one source of truth for "is the agent on".** During a staged rollout, the autonomy mode a workspace had selected could still be blocked by deployment flags or a workspace allowlist, so the settings page showed a mode as active while another layer blocked it. Compute an effective state on the server from every layer and show that, along with the layer that is blocking it.
- **Show the real cause of platform errors.** When the model provider rejected requests because the account's credits were exhausted, users saw only a generic "temporarily unavailable" message, which told them nothing about who could fix it. Map known causes (credits or quota exhausted, provider maintenance, credential expired, policy denial) to specific messages, each with the action and the person who can fix it.

---

## UX Comparison: Approaches for Team Onboarding

| Approach | Time to First Value | Admin Effort | Junior Safety | Collaboration |
|----------|-------------------|-------------|--------------|---------------|
| **Org-based multi-tenancy** (recommended) | Minutes (quick connect) | Low (one-time setup) | High (structural isolation) | Good (shared sessions, per-repo config) |
| **Per-user permissions** | Hours (configure each user) | High (ongoing) | Medium (role-dependent) | Poor (manual sharing) |
| **Namespace/project-based** | Medium | Medium | Medium | Medium |
| **Single-tenant deployment** | Days (deploy per team) | Very high | High (full isolation) | None (no cross-team visibility) |
| **No isolation** (everyone is admin) | Instant | None | None | Full (no privacy) |

---

## In Opengeni

Opengeni comes with a web app for working with agents directly and a `SessionConversation` React component that a product can embed behind a tenant- and user-scoped proxy, so the organization API key stays on the product's server ([product integration](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/product-integration.md)). That is how Cloudgeni's own agent chat is built. Organizations, workspaces and personal workspaces follow the tenancy model in [organization tenancy](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/organization-tenancy.md). You can use it at [app.opengeni.ai](https://app.opengeni.ai) or self-host the same stack.

---

## Next Chapter

[Chapter 13: Risk Framework & Checklists →](./13-risk-framework.md)

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
