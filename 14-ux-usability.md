# Chapter 14: UX, Usability & Team Onboarding

> How to package infrastructure agents so everyone can adopt them safely — without reading a 13-chapter guide first.

---

## The Usability Problem

You've built the agent runtime, the sandbox, the credential broker, the policy engine, the observability stack. Congratulations — you now have a system that only you understand.

The real test isn't whether infrastructure agents *work*. It's whether a **platform team of 3 can serve 50 engineers across 5 teams** without becoming a bottleneck, and without a junior developer accidentally giving an agent admin credentials to production.

This chapter covers the UX and organizational patterns that make the difference between a research project and a production platform.

---

## Design Principle: Make the Safe Path the Easy Path

Every UX decision should follow one rule:

> **The default behavior should be the secure behavior. Doing the wrong thing should require more effort than doing the right thing.**

Examples:
- New agents default to plan-only mode (Tier 2) — getting apply access requires explicit escalation
- Credentials are scoped to one cloud account by default — cross-account access requires admin action
- Agent sessions are private by default — sharing requires explicit toggle
- PR review is per-repository opt-in — not a global flag
- Variables are org-scoped by default — narrowing to repo scope is an override

If you find yourself building "are you sure?" confirmation dialogs, the UX has already failed. The system should be designed so the dangerous action isn't reachable without deliberate, admin-level configuration.

---

## Multi-Tenancy: Organization as the Security Boundary

### Why Organizations, Not Users

The organization is the **top-level isolation boundary**. Everything — credentials, repositories, agents, policies, sessions, variables — is scoped to an organization. This means:

- A platform engineer at Company A **cannot** see Company B's infrastructure
- A team within an organization **can** share agent sessions with each other
- An admin for Org X **cannot** escalate to Org Y, even if they're the same person

```
┌──────────────────────────────────────────────────────────┐
│  Organization: "Acme Platform Team"                      │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │ AWS Prod    │  │ AWS Staging  │  │ Azure Dev   │      │
│  │ (creds)     │  │ (creds)     │  │ (creds)     │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │ infra-core  │  │ app-deploy  │  │ data-pipelines│     │
│  │ (repo)      │  │ (repo)      │  │ (repo)       │     │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│                                                          │
│  ┌─────────────────────────────────────────────────┐     │
│  │ Agents: remediation, drift, PR review, devops   │     │
│  │ Policies: "encrypt all S3", "tag everything"    │     │
│  │ Variables: TF_BACKEND, AWS_REGION, ENV          │     │
│  └─────────────────────────────────────────────────┘     │
│                                                          │
│  Members: alice (admin), bob (member), carol (member)    │
└──────────────────────────────────────────────────────────┘

  ← No data crosses this boundary →

┌──────────────────────────────────────────────────────────┐
│  Organization: "Acme App Team"                           │
│  (Separate credentials, repos, agents, policies)         │
└──────────────────────────────────────────────────────────┘
```

### Organization Types

Support multiple ways organizations get created:

```typescript
enum OrganizationType {
  PERSONAL    // Auto-created per user (solo workspace)
  GITHUB_ORG  // Imported from GitHub organization
  GITLAB_GROUP // Imported from GitLab group
  CUSTOM      // Manually created team workspace
}
```

Importing from VCS providers is powerful: when a team connects their GitHub org, repositories and team structure come for free. New VCS members can auto-join the organization if the admin enables it.

### Data Isolation Implementation

Every database query must include the organization filter. This is non-negotiable.

```typescript
// Every route is under /api/v1/organizations/:orgId/...
// Middleware resolves and validates org access before handlers run

// GOOD: org-scoped query
const pipelines = await db.iacPipeline.findMany({
  where: {
    organizationId: req.orgId,  // Injected by middleware
    deletedAt: null,
  },
});

// BAD: no org scope — can see other tenants' data
const pipelines = await db.iacPipeline.findMany({
  where: { deletedAt: null },
});
```

Enforce this with middleware that runs before every handler:

```typescript
// Middleware chain
app.use('/api/v1/organizations/:orgId/*',
  authenticate,              // Verify JWT or API key
  resolveOrganization,       // Map external org ID → internal ID
  requireOrganizationAccess, // Verify user has active membership
  enforceRestrictions,       // Apply role-based restrictions
);
```

---

## RBAC: Simple Roles, Structural Safety

### Keep Roles Simple

Complex role hierarchies create confusion. Three roles cover 95% of needs:

| Role | Can Do | Cannot Do |
|------|--------|-----------|
| **Admin** | Manage members, integrations, credentials, policies, agent configs, billing | Access other organizations |
| **Member** | Use agents, view sessions, create PRs, run scans, view findings | Modify integrations, manage members, change policies |
| **Viewer** | View dashboards, sessions, findings (read-only) | Trigger agents, modify anything |

```typescript
interface OrganizationMembership {
  userId: string;
  organizationId: string;
  role: 'admin' | 'member' | 'viewer';
  status: 'ACTIVE' | 'INVITED' | 'SUSPENDED';
  invitedBy: string;
  joinedAt: Date;
}
```

### Where Safety Lives: Structural, Not Role-Based

The key insight: **don't rely on roles to prevent junior developers from making mistakes. Structure the system so mistakes are impossible.**

| Risk | Bad Approach (role-based) | Good Approach (structural) |
|------|--------------------------|---------------------------|
| Junior gives agent prod creds | "Only admins can select integrations" | Agent can only access integrations admin has pre-configured for this org. There's no "enter credentials" field in the agent UI. |
| Developer enables apply mode | "Only admins can change agent tier" | Apply mode requires a separate credential set that members can't create. The agent config is admin-only. |
| New team member accesses wrong repo | "RBAC on repository access" | Repositories are per-org. The team member only sees repos connected to their org. |
| Someone shares a session with secrets | "Warn users about sharing" | Log redaction is automatic. Sessions with credential requests have redacted output by default. |

### Invitation Flow

Make team onboarding frictionless:

```typescript
// Smart invite: check if user already exists
async function inviteMember(orgId: string, email: string, role: string) {
  const existingUser = await db.user.findUnique({ where: { email } });

  if (existingUser) {
    // Direct invite — user can accept immediately
    await db.organizationMembership.create({
      data: {
        userId: existingUser.id,
        organizationId: orgId,
        role,
        status: 'INVITED',
      },
    });
    await sendInAppNotification(existingUser.id, 'INVITATION', { orgId });
  } else {
    // Email invite — user will see invite on first login
    await sendEmailInvitation(email, orgId, role);
  }
}
```

---

## Onboarding: Time-to-Value in Minutes

### The Two Mandatory Steps

Don't present a 15-step wizard. Start with exactly two things:

```
┌─────────────────────────────────────────────────────────────┐
│                    GET STARTED                               │
│                                                             │
│  You need two things to start using infrastructure agents:  │
│                                                             │
│  ┌─────────────────────┐   ┌─────────────────────────┐     │
│  │  1. CONNECT CLOUD   │   │  2. CONNECT GIT REPO    │     │
│  │                     │   │                         │     │
│  │  ☁️  AWS            │   │  ⚡ GitHub              │     │
│  │  ☁️  Azure          │   │  ⚡ GitLab              │     │
│  │  ☁️  GCP            │   │  ⚡ Azure DevOps        │     │
│  │                     │   │                         │     │
│  │  [Quick Connect]    │   │  [Connect Provider]     │     │
│  └─────────────────────┘   └─────────────────────────┘     │
│                                                             │
│  After these, agents can analyze your infrastructure        │
│  and start creating PRs.                                    │
└─────────────────────────────────────────────────────────────┘
```

Everything else is progressive — shown only after the mandatory steps are complete.

### Quick Connect: Cloud in Under 60 Seconds

The fastest path to value. No manual credential entry, no copying access keys.

**AWS Quick Connect**:
```
User clicks "Connect AWS"
  → Redirected to AWS Console
  → CloudFormation stack deploys a cross-account IAM role
  → Role automatically trusts the platform's AWS account
  → Integration auto-completes when stack is ready
  → Time: ~30 seconds
```

```typescript
// AWS Quick Connect — generate a CloudFormation console URL
function generateAwsQuickConnectUrl(orgId: string): string {
  const templateUrl = `https://your-platform.com/cfn/cross-account-role.yaml`;
  const stackName = `infra-agent-integration-${orgId}`;

  return `https://console.aws.amazon.com/cloudformation/home?#/stacks/create/review` +
    `?templateURL=${encodeURIComponent(templateUrl)}` +
    `&stackName=${stackName}` +
    `&param_ExternalId=${orgId}` +
    `&param_TrustedAccountId=${PLATFORM_AWS_ACCOUNT_ID}`;
}
```

**Azure Quick Connect**:
```
User clicks "Connect Azure"
  → OAuth consent flow opens Azure portal
  → Service principal created automatically
  → Subscription access granted via consent
  → Token stored (encrypted) — only short-lived tokens used by agents
  → Time: ~10 seconds
```

**Key principle**: Quick Connect should **never** require the user to copy/paste credentials. The platform should use standard delegation mechanisms (cross-account roles, OAuth, OIDC) that keep long-lived credentials out of human hands.

### Progressive Onboarding Steps

After the two mandatory steps, show optional steps based on value:

```typescript
interface OnboardingStep {
  type: string;
  title: string;
  description: string;
  estimatedMinutes: number;
  isRequired: boolean;
  dependencies: string[];   // Steps that must complete first
  isRecommended: boolean;
}

const ONBOARDING_STEPS: OnboardingStep[] = [
  // Mandatory (Phase 1)
  { type: 'CLOUD_CONNECT', title: 'Connect cloud account', isRequired: true, dependencies: [] },
  { type: 'GIT_CONNECT', title: 'Connect Git repository', isRequired: true, dependencies: [] },

  // Recommended (Phase 2 — shown after mandatory complete)
  { type: 'COMPLIANCE_SCAN', title: 'Run first compliance scan',
    dependencies: ['CLOUD_CONNECT'], estimatedMinutes: 5, isRecommended: true },
  { type: 'STATIC_ANALYSIS', title: 'Run static analysis on IaC',
    dependencies: ['GIT_CONNECT'], estimatedMinutes: 3, isRecommended: true },
  { type: 'FIRST_REMEDIATION', title: 'Fix your first finding with an agent',
    dependencies: ['COMPLIANCE_SCAN'], estimatedMinutes: 10, isRecommended: true },

  // Optional (Phase 3)
  { type: 'DRIFT_SETUP', title: 'Enable continuous drift detection',
    dependencies: ['GIT_CONNECT', 'CLOUD_CONNECT'], isRecommended: false },
  { type: 'PR_REVIEW', title: 'Enable automated PR review',
    dependencies: ['GIT_CONNECT'], isRecommended: false },
  { type: 'INVITE_TEAM', title: 'Invite team members',
    dependencies: [], isRecommended: false },
];
```

### Sidebar Progress Indicator

Show onboarding progress persistently until mandatory steps are complete:

```typescript
// Collapsed sidebar: circular progress ring
// Expanded sidebar: linear progress bar + "X of Y steps completed"
// Hidden once mandatory steps are done (don't nag)

function OnboardingProgress({ steps }) {
  const completed = steps.filter(s => s.status === 'COMPLETED').length;
  const mandatory = steps.filter(s => s.isRequired);
  const mandatoryComplete = mandatory.every(s => s.status === 'COMPLETED');

  if (mandatoryComplete) return null;  // Don't show once basics are done

  return <ProgressBar value={completed} max={steps.length} />;
}
```

---

## Self-Service Agent UX

### The Chat Interface: Entry Point for Everyone

The primary interface for agents should feel like a chat app, not a CI/CD dashboard:

```
┌─────────────────────────────────────────────────────────────┐
│  ┌──────────┐  ┌──────────┐  ┌─────────────────┐           │
│  │ Agent: ▼ │  │ Repo: ▼  │  │ Cloud Acct: ▼   │           │
│  │ DevOps   │  │ infra-   │  │ AWS Prod        │           │
│  │          │  │ core     │  │                 │           │
│  └──────────┘  └──────────┘  └─────────────────┘           │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Fix the S3 bucket encryption finding for           │   │
│  │  my-data-bucket                                      │   │
│  │                                                     │   │
│  │                                        [Send →]     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Three selectors below the input** — this is critical UX:

1. **Agent selector**: Pick from pre-configured agents (DevOps, Remediation, PR Review, etc.)
2. **Repository selector**: Choose one or more repositories (with branch override)
3. **Cloud integration selector**: Choose which cloud account the agent can access

**Why this works**: The member doesn't need to know about credentials, pipelines, or agent configurations. The admin has already set up the integrations and agent configs. The member just picks from dropdowns and asks a question.

### Prompt Discovery: Don't Make Users Guess

New users don't know what to ask. Provide categorized starting prompts:

```typescript
const PROMPT_CATEGORIES = {
  'Fix Compliance': [
    'Find my top 3 critical compliance issues and fix them',
    'Fix the unencrypted S3 bucket in the production account',
    'Remediate all HIGH severity findings from the last scan',
  ],
  'Create Resources': [
    'Create an S3 bucket with versioning and encryption enabled',
    'Set up a VPC with public and private subnets',
    'Create an RDS instance with encryption and automated backups',
  ],
  'Refactor': [
    'Refactor my EC2 instances to use a reusable module',
    'Convert my inline security groups to named resources',
    'Split this monolithic Terraform config into modules',
  ],
  'Analyze': [
    'Explain how my infrastructure is set up in this repository',
    'What resources are managed by this Terraform state?',
    'Show me all security groups with open SSH access',
  ],
};
```

### Deep-Link Support

Let agents be triggered from anywhere — compliance dashboards, Slack notifications, CI/CD pipelines:

```
/agent-sessions?agent=remediation&repositoryId=xxx&prompt=Fix+finding+ABC123
```

This means a "Fix with Agent" button on a compliance finding card can pre-populate the agent, repo, and prompt — one click to start remediation.

---

## Session Sharing & Collaboration

### Default-Private, Opt-In Sharing

Agent sessions often contain infrastructure details that not everyone should see. Default to private:

```typescript
// Organization-level default
interface OrgSettings {
  defaultChatSharing: 'PRIVATE' | 'SHARED';  // Admin configures this
}

// Session-level override (per session)
interface AgentSession {
  isSharedWithOrg: boolean | null;
  // null = follow org default
  // true = explicitly shared
  // false = explicitly private
}
```

### Sharing UX

In the session list, each session shows a small share toggle:
- **Private** (default): Only the creator can see it
- **Shared**: All org members can see it (read-only for non-creators)
- **Shared by others**: Shows "Shared with you by Alice" with a distinct icon

```
Session List:
┌──────────────────────────────────────────┐
│ ● Fix S3 encryption         🔒 Private  │
│ ● Drift remediation         🔗 Shared   │
│ ● VPC refactoring (Bob)     👥 From Bob │
│ ● Compliance scan review    🔒 Private  │
└──────────────────────────────────────────┘
```

### Why Not Real-Time Collaboration?

Unlike Google Docs, infrastructure agent sessions are primarily **sequential** — one user asks, the agent responds. Real-time multi-cursor editing adds complexity without proportional value. Instead:

- **Sharing** lets others view what the agent did and the PRs it created
- **Forking** lets another team member continue from a specific point
- **Session history** provides the audit trail for team review

---

## Policies as Plain Language

### Why Not a DSL?

Infrastructure teams already struggle with Rego, Sentinel, and OPA policy languages. Agent policies should be in **plain language** that any engineer can read and write:

```markdown
## Policy: Encryption Requirements

- All S3 buckets MUST have server-side encryption enabled (SSE-KMS preferred)
- All RDS instances MUST have storage encryption enabled
- When fixing encryption findings, use the organization's KMS key: alias/infra-key
- Never disable encryption on existing resources, even temporarily
```

The agent reads this as part of its context. The policy is versioned, auditable, and understandable by the person who wrote it, the person who reviews it, and the person who debugs why the agent did what it did.

### Policy Templates

Don't start from blank. Provide categorized templates that teams can customize:

```typescript
const POLICY_TEMPLATES = [
  {
    category: 'security',
    name: 'Encryption at Rest',
    content: '...',  // Pre-written policy text
  },
  {
    category: 'compliance',
    name: 'Tagging Standards',
    content: 'All resources MUST have these tags: Environment, Service, Owner, CostCenter, ManagedBy...',
  },
  {
    category: 'security',
    name: 'Network Access Controls',
    content: 'No security group should allow 0.0.0.0/0 ingress on ports other than 80 and 443...',
  },
];
```

### Policy Audit Trail

Every policy change is tracked:

```typescript
interface PolicyChangeHistory {
  policyId: string;
  action: 'CREATED' | 'UPDATED' | 'ACTIVATED' | 'DEACTIVATED';
  changedBy: string;   // userId
  previousVersion: number;
  newVersion: number;
  diff: string;        // What changed
  timestamp: Date;
}
```

---

## Variable Scoping: Environment Separation Without Complexity

### Three-Level Precedence

Variables (environment config, backend addresses, region defaults) follow a clear override chain:

```
Repository scope  (highest priority — overrides everything)
  ↓
Integration scope (overrides org level)
  ↓
Organization scope (baseline defaults)
```

```typescript
interface Variable {
  key: string;
  value: string;       // Encrypted at rest
  isSecret: boolean;   // If true, value is never displayed in UI after save
  scope: 'ORGANIZATION' | 'INTEGRATION' | 'REPOSITORY';
  organizationId: string;
  integrationId?: string;   // Set when scope = INTEGRATION
  repositoryId?: string;    // Set when scope = REPOSITORY
}

// Resolution: most specific wins
function resolveVariable(
  key: string,
  orgId: string,
  integrationId?: string,
  repoId?: string
): string | undefined {
  // 1. Check repository scope
  if (repoId) {
    const repoVar = findVariable(key, 'REPOSITORY', { repoId });
    if (repoVar) return repoVar.value;
  }

  // 2. Check integration scope
  if (integrationId) {
    const intVar = findVariable(key, 'INTEGRATION', { integrationId });
    if (intVar) return intVar.value;
  }

  // 3. Fall back to organization scope
  return findVariable(key, 'ORGANIZATION', { orgId })?.value;
}
```

**Why this matters**: A platform team can set `TF_BACKEND_BUCKET=acme-terraform-state` at the org level, and individual teams can override it per-repo if they use a different backend — without either team needing to understand the other's configuration.

---

## Per-Repository Agent Configuration

### Automated PR Review: Opt-In Per Repository

Don't enable AI review globally. Let admins turn it on per-repo:

```typescript
interface Repository {
  // ...
  prReviewEnabled: boolean;       // Is automated review active?
  prReviewAgentId?: string;       // Which agent config to use
}
```

When a PR is opened on a repo with `prReviewEnabled: true`, the system automatically dispatches the configured review agent. Developers don't configure anything — they just see review comments appear on their PRs.

### Pipeline Configuration: Admin Sets Up, Members Use

IaC pipelines (Terraform plan, Bicep what-if) are configured by admins:

```typescript
interface IacPipeline {
  organizationId: string;
  repositoryId: string;
  provider: 'GITHUB_ACTIONS' | 'GITLAB_CI' | 'AZURE_PIPELINES';
  pipelineRef: string;       // Workflow file or pipeline ID
  defaultBranch: string;

  // Per-environment overrides
  environments: IacPipelineEnvironment[];

  // Artifact configuration
  outputs: IacPipelineOutput[];
}

interface IacPipelineEnvironment {
  name: string;              // "dev", "staging", "prod"
  branch?: string;           // Override default branch
  parameterDefaults?: Json;  // Environment-specific params
}
```

Members interact with pipelines through agents — "validate my changes" — not by configuring pipeline definitions.

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

### Concrete Patterns

| Mistake | Prevention (Level) |
|---------|-------------------|
| Giving agent prod credentials | Cloud integrations are admin-configured. Members select from dropdown, never enter creds. **(Impossible)** |
| Running terraform apply directly | Agent system prompts have hard rules against apply. Tool allow-list excludes apply. **(Impossible)** |
| Accessing another team's resources | Organization boundary isolates all data. No cross-org query path exists. **(Impossible)** |
| Creating overly broad IAM policies | Custom policies define org standards. Agent reads and follows them. **(Hard — requires policy to be wrong)** |
| Pushing to main branch | Agent hard rules + branch protection rules. **(Hard)** |
| Sharing sessions with sensitive data | Default-private sessions + automatic log redaction. **(Hard)** |
| Misconfiguring a pipeline | Pipeline config is admin-only. Members use pipelines through agents. **(Impossible for members)** |
| Forgetting to validate before PR | Agent's validation loop runs automatically. No skip option. **(Impossible)** |
| Using wrong variable values | Three-level variable precedence resolves automatically. **(Invisible — correct by default)** |

### API Key Scoping

API keys for programmatic access are organization-scoped at creation time:

```typescript
interface ApiKey {
  id: string;
  organizationId: string;  // Bound at creation — cannot be changed
  name: string;
  keyHash: string;         // Only the hash is stored
  lastUsedAt?: Date;
  expiresAt?: Date;
}

// Middleware enforces: API key's org must match the route's org
if (apiKey.organizationId !== routeOrgId) {
  throw new ForbiddenError('API_KEY_ORG_MISMATCH');
}
```

---

## Notification Preferences: Don't Flood, Don't Miss

### Per-User Control

```typescript
interface NotificationPreference {
  userId: string;

  // Category toggles
  securityNotifications: boolean;       // Compliance findings, vulnerabilities
  remediationNotifications: boolean;    // Agent PR created, remediation status
  productNotifications: boolean;        // Feature updates, tips
  organizationNotifications: boolean;   // Member joined/left, role changes

  // Email digests
  emailDigestEnabled: boolean;
  weeklySecurityDigest: boolean;
  monthlySecurityDigest: boolean;

  // Quiet hours (don't send during off-hours)
  quietHoursEnabled: boolean;
  quietHoursStart: string;   // "22:00"
  quietHoursEnd: string;     // "08:00"

  // Per-event-type overrides
  eventPreferences: Record<string, boolean>;
}
```

### Organization-Level Slack

Slack notifications are configured once at the org level — not per-user:

```typescript
interface OrgSettings {
  slackWebhookUrl?: string;
  slackNotificationsEnabled: boolean;
  slackChannelName?: string;            // Display label
  slackEventPreferences: {              // Per-event toggles
    'remediation.completed': boolean;
    'scan.completed': boolean;
    'drift.detected': boolean;
    'agent.failed': boolean;
    // ...
  };
}
```

This means the admin configures Slack once, and the whole team benefits without individual setup.

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

## Implementation Checklist

### Before Launch

```
MULTI-TENANCY
[ ] Organization model with full data isolation
[ ] Middleware enforces org scope on every API route
[ ] API keys are org-scoped at creation
[ ] Cross-org access returns 403, not 404 (don't leak existence)

RBAC
[ ] Admin / Member / Viewer roles implemented
[ ] Credential management restricted to Admin
[ ] Agent configuration restricted to Admin
[ ] Pipeline configuration restricted to Admin
[ ] Member can use agents, view sessions, see findings

ONBOARDING
[ ] Two mandatory steps: cloud connect + git connect
[ ] Quick connect for at least one cloud provider
[ ] Progressive optional steps after mandatory complete
[ ] Sidebar progress indicator (hidden after mandatory done)
[ ] Categorized prompt suggestions for new users

SELF-SERVICE
[ ] Chat interface with agent/repo/cloud selectors
[ ] Deep-link support for triggering agents from dashboards
[ ] Policy templates for common standards
[ ] Per-repository PR review toggle (admin-configurable)

COLLABORATION
[ ] Session sharing (default-private, opt-in sharing)
[ ] Team member invitation flow (email + in-app)
[ ] Org-level Slack integration (one-time admin setup)
[ ] Per-user notification preferences with quiet hours

SAFETY
[ ] Credentials only in admin-accessible settings
[ ] Variable scoping (org → integration → repo precedence)
[ ] Automatic log redaction for sensitive data
[ ] No raw credential entry in agent chat interface
```

---

## Key Takeaways

1. **Organization = security boundary** — all data, all credentials, all agents scoped to org
2. **Admin sets up, members use** — credentials, pipelines, and policies are admin territory
3. **Quick Connect** reduces cloud onboarding from hours to seconds
4. **Two mandatory steps** — cloud + git. Everything else is progressive.
5. **Chat-first UX** with smart selectors — members pick from pre-configured options
6. **Structural prevention** over role-based prevention — make mistakes impossible, not forbidden
7. **Default-private, opt-in sharing** — respect that infra details are sensitive
8. **Plain language policies** — no DSL to learn, version-controlled, auditable
9. **Per-user notifications with quiet hours** — don't burn out the team

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
