# Chapter 3: Skill & Tool System

> How agents gain capabilities — safely and extensibly.

---

## The Problem With Raw Tool Access

The naive approach to agent tools is: register shell commands, let the model call them. This is how most early agents worked (AutoGPT, BabyAGI) and it's how most security incidents happen.

The issues:
- **Shell is too powerful** — `rm -rf /`, `curl | bash`, credential exfiltration
- **Tools are untyped** — the model can pass any arguments
- **No documentation** — the model guesses how tools work
- **No isolation** — tools share the same process/permissions as the agent

A better approach: **Skills as typed, documented, isolated capabilities**.

---

## The Skills-as-Files Pattern

Instead of registering tool handlers in code, write skills as files that the agent can read and execute:

```
.claude/skills/
├── terraform-plan/
│   ├── SKILL.md           # Instructions for the agent (documentation)
│   └── client.ts          # Typed API client (executable code)
├── drift-verification/
│   ├── SKILL.md
│   └── client.ts
├── git-operations/
│   ├── SKILL.md
│   └── client.ts
└── cloud-credentials/
    ├── SKILL.md
    └── client.ts
```

### Why This Pattern Is Elegant

1. **Self-documenting** — The agent reads SKILL.md to understand what the skill does, when to use it, and what constraints apply
2. **Typed execution** — client.ts provides a typed interface; the agent imports and calls functions
3. **Isolated** — Skills run as subprocess scripts with no access to the worker's internals
4. **Auditable** — Every skill file can be version-controlled and reviewed
5. **Runtime-injectable** — Skills can be added/removed per agent, per organization, at dispatch time

### Example: Terraform Plan Skill

**SKILL.md**:
```markdown
# Terraform Plan Skill

## When to Use
Use this skill after making changes to Terraform files to validate
your changes produce the expected plan output.

## How It Works
1. Call `triggerPlan(pipelineId)` to start a plan execution
2. Poll `getPlanStatus(runId)` until the run completes
3. Read the parsed output to verify:
   - No unexpected resource deletions
   - No drift remaining on target resources
   - Plan succeeds without errors

## Constraints
- NEVER call this on a pipeline you don't have permission for
- ALWAYS check the plan output before creating a PR
- If the plan shows unexpected changes, STOP and ask the user
- Maximum 10 plan iterations per session

## Error Handling
- If the pipeline fails, check the error output for syntax errors
- If the pipeline times out, wait 60 seconds and retry once
- If authentication fails, request fresh credentials
```

**client.ts**:
```typescript
const INTERNAL_API = 'http://localhost:3000';

interface PlanResult {
  runId: string;
  status: 'success' | 'failure' | 'timeout';
  driftResources: Array<{
    address: string;
    resourceId: string;
    changeType: 'create' | 'update' | 'delete' | 'no-op';
  }>;
  allResources: Array<{ address: string; resourceId: string }>;
  rawOutput?: string;
  error?: string;
}

export async function triggerPlan(pipelineId: string): Promise<{ runId: string }> {
  const res = await fetch(`${INTERNAL_API}/run-iac-pipeline`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ pipelineId, type: 'PLAN' }),
  });
  return res.json();
}

export async function getPlanStatus(runId: string): Promise<PlanResult> {
  const res = await fetch(`${INTERNAL_API}/get-iac-pipeline-status`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ runId }),
  });
  return res.json();
}
```

---

## Internal Server: The Capability Bridge

Skills can't access the database, cloud APIs, or worker internals directly. They communicate through a local HTTP server running inside the worker:

```mermaid
graph LR
    subgraph "Agent Worker Process"
        AGENT[LLM Agent] -->|import & call| SKILL[Skill Script<br/>subprocess]
        SKILL -->|HTTP fetch| INTERNAL[Internal Server<br/>localhost:3000]
        INTERNAL -->|RPC| API[API Server]
    end

    subgraph "External"
        API --> DB[(Database)]
        API --> CLOUD[Cloud APIs]
        API --> CICD[CI/CD]
    end
```

```typescript
// Internal server — runs inside the worker process
import { serve } from 'bun'; // or express, fastify, etc.

const server = serve({
  port: 3000,
  async fetch(req) {
    const url = new URL(req.url);

    switch (url.pathname) {
      case '/run-iac-pipeline':
        return handleRunPipeline(req);
      case '/get-cloud-credentials':
        return handleGetCredentials(req);
      case '/create-pull-request':
        return handleCreatePR(req);
      case '/ask-user':
        return handleAskUser(req);
      default:
        return new Response('Not found', { status: 404 });
    }
  },
});

async function handleGetCredentials(req: Request): Response {
  const { integrationId, scope } = await req.json();

  // Validate the request against the current task context
  if (!isAllowedIntegration(integrationId)) {
    return Response.json({ error: 'Integration not authorized' }, { status: 403 });
  }

  // RPC to API server — which generates short-lived tokens
  const token = await rpcCall('generateCloudToken', {
    integrationId,
    organizationId: currentContext.organizationId,
    scope,
  });

  return Response.json({ token, expiresIn: 3600 });
}
```

### Why Not Direct MCP/Tool Registration?

| Approach | Pros | Cons |
|----------|------|------|
| **Skills as files** (recommended) | Agent can read docs; typed clients; easy to add/remove per context; version-controlled | Slightly more setup; requires internal server |
| **MCP tool registration** | Standard protocol; some LLMs support natively | Less documentation; harder to scope per-agent; tight coupling |
| **Direct function calls** | Simplest; no HTTP overhead | No isolation; hard to audit; agent can call anything |
| **OpenAI-style function calling** | Model generates structured args; you validate | Schema-only docs; no rich documentation; tied to one provider |

---

## Skill Categories for Infrastructure Agents

### IaC Lifecycle Skills

| Skill | Purpose | Risk Level |
|-------|---------|------------|
| `terraform-plan` | Trigger and read plan output | Low (read-only) |
| `terraform-validate` | Syntax/config validation | Low |
| `bicep-what-if` | Azure deployment preview | Low (read-only) |
| `iac-lint` | Static analysis (tflint, checkov) | Low |
| `drift-verification` | Compare state vs. code | Low (read-only) |

### Git & PR Skills

| Skill | Purpose | Risk Level |
|-------|---------|------------|
| `git-branch` | Create/switch branches | Low |
| `git-commit` | Stage and commit changes | Medium |
| `git-push` | Push to remote | Medium |
| `create-pr` | Open a pull request | Medium |
| `pr-comment` | Post review comments | Low |

### Cloud Interaction Skills

| Skill | Purpose | Risk Level |
|-------|---------|------------|
| `cloud-credentials` | Request short-lived tokens | High (gated) |
| `resource-describe` | Read cloud resource state | Low |
| `resource-import` | Import into Terraform state | High (gated) |

### Communication Skills

| Skill | Purpose | Risk Level |
|-------|---------|------------|
| `ask-user` | Pause for human input | Low |
| `notify-slack` | Post status to Slack | Low |
| `create-ticket` | Open a Jira/Linear issue | Low |

---

## Real-World IaC Skills Ecosystem

The infrastructure skills ecosystem has matured rapidly. Vendors, cloud providers, and the community have published production-grade skills and MCP servers that you can learn from, adopt, or use as templates for your own.

### The Agent Skills Format

The standard format — popularized by [Anthropic's skills spec](https://github.com/anthropics/skills) (72K+ stars) — is structurally simple: a directory with a required `SKILL.md` file containing YAML frontmatter (name, description) plus optional scripts and assets. The frontmatter follows a **three-level progressive disclosure** model:

```yaml
---
# Level 1: Frontmatter (always loaded into agent context for discovery)
name: terraform-style-guide
description: Write and review Terraform following HashiCorp style conventions
---

# Level 2: SKILL.md body (loaded on demand when skill is activated)
## When to Use
Use when writing new Terraform configurations or reviewing existing HCL...

## Instructions
Follow these conventions...

## References
- [reference: ./hashicorp-style-guide.md]  # Level 3: linked files (loaded when needed)
```

This progressive disclosure keeps context manageable — the agent sees all skill names/descriptions upfront but only loads full instructions when relevant.

### Skills + MCP Together

A powerful pattern now appearing in official ecosystems: **Skills provide the "textbook" (best practices, workflows, guardrails) while MCP provides the "pipe" to live data and tools.**

```
┌──────────────────────────────────────────────────────┐
│  SKILL.md                                            │
│  "When writing Terraform, follow these conventions,  │
│   use these module patterns, validate with plan..."  │
│                                                      │
│  ┌────────────────────────────────────────────────┐  │
│  │  MCP Server (wired via plugin.json)            │  │
│  │  - Query Terraform Registry for providers      │  │
│  │  - Read workspace state from Terraform Cloud   │  │
│  │  - Search module documentation                 │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

HashiCorp's Claude plugin demonstrates this: it bundles skills for Terraform code generation alongside a Docker-run MCP server that gives the agent live access to the Terraform Registry and Terraform Cloud APIs.

### Vendor-Official Skills

#### HashiCorp — [hashicorp/agent-skills](https://github.com/hashicorp/agent-skills)

The most complete vendor skill set for Terraform. Organized into three plugin bundles:

| Plugin | Skills Included | Focus |
|--------|----------------|-------|
| `terraform-code-generation` | `terraform-style-guide`, `terraform-test`, `azure-verified-modules` | Writing correct HCL, testing |
| `terraform-module-generation` | `refactor-module`, `terraform-stacks` | Module extraction, HCP Terraform Stacks |
| `terraform-provider-development` | `new-terraform-provider`, `provider-actions`, `provider-resources` | Building Terraform providers |

The `refactor-module` skill is notable because it covers **state migration patterns** (`moved` blocks, `terraform state mv`) — high blast radius if executed incorrectly. The `terraform-stacks` skill explicitly recommends **workload identity (OIDC)** and ephemeral tokens.

#### Pulumi — [pulumi/agent-skills](https://github.com/pulumi/agent-skills)

Covers both authoring and migration workflows:

| Category | Skills | Focus |
|----------|--------|-------|
| **Authoring** | `pulumi-best-practices`, `pulumi-component`, `pulumi-automation-api`, `pulumi-esc` | Writing Pulumi programs, ESC secrets management |
| **Migration** | `pulumi-terraform-to-pulumi`, `cloudformation-to-pulumi`, `pulumi-cdk-to-pulumi`, `pulumi-arm-to-pulumi` | Full migration workflows with state translation |

The migration skills emphasize a **"zero-diff preview" requirement** — after importing state, the Pulumi preview must show no changes before considering the migration successful. This is a key safety pattern for any state migration skill.

### Vendor MCP Servers

#### HashiCorp — [hashicorp/terraform-mcp-server](https://github.com/hashicorp/terraform-mcp-server)

Provides live access to the Terraform ecosystem via MCP:
- **Terraform Registry**: query providers, modules, policies
- **Terraform Cloud/Enterprise**: workspace CRUD, run management, private registry
- Supports both `stdio` and `StreamableHTTP` transports

> **Security note**: The repo explicitly warns to restrict `MCP_ALLOWED_ORIGINS` to mitigate cross-origin/DNS rebinding attacks, and recommends local-only use with trusted clients.

#### AWS — [awslabs/mcp](https://github.com/awslabs/mcp)

The AWS MCP monorepo (8K+ stars) contains three IaC-relevant servers:

| Server | Capabilities | Risk Level |
|--------|-------------|------------|
| **aws-iac-mcp-server** | CloudFormation/CDK docs search, template validation, compliance checks, deployment troubleshooting | Low (read-only, guidance) |
| **cfn-mcp-server** | Direct CRUD of 1,100+ AWS resource types via Cloud Control API, IaC Generator | **Very High** (can create/delete resources) |
| **terraform-mcp-server** | AWS Terraform best practices, Checkov scanning, `terraform plan/apply/destroy` execution | **Very High** (can run apply/destroy) |

> **Critical distinction**: `aws-iac-mcp-server` is guidance-only and safe. `cfn-mcp-server` and `terraform-mcp-server` can make destructive changes — treat them as Tier 3/4 tools requiring explicit approval gates.

#### Pulumi MCP Server

Available as `@pulumi/mcp-server` on npm and `docker pull mcp/pulumi`. Interacts with Pulumi Cloud for stack preview, deploy, output retrieval, and registry queries. Requires OAuth flow and Pulumi Access Token with org scoping.

### Community Skills (Notable)

| Repository | Stars | Focus | Why It's Notable |
|-----------|-------|-------|-----------------|
| [antonbabenko/terraform-skill](https://github.com/antonbabenko/terraform-skill) | 1.1K+ | Terraform & OpenTofu | By Anton Babenko (prolific TF community contributor). Comprehensive single-file SKILL.md covering testing, modules, CI/CD, and production patterns |
| [akin-ozer/cc-devops-skills](https://github.com/akin-ozer/cc-devops-skills) | 70+ | Multi-tool DevOps | 31 skills spanning Terraform, Terragrunt, Ansible, Kubernetes, Helm, GitHub Actions, GitLab CI, Jenkins, PromQL, and more |
| [terramate-io/agent-skills](https://github.com/terramate-io/agent-skills) | 25+ | Terraform, OpenTofu, Terramate | State splitting, drift reconciliation, stack management |
| [dirien/claude-skills](https://github.com/dirien/claude-skills) | — | Pulumi (TS/Go/Python) | Pulumi community skills emphasizing ESC + OIDC patterns |
| [sigridjineth/hello-ansible-skills](https://github.com/sigridjineth/hello-ansible-skills) | 23+ | Ansible | Playbook development, debugging, shell-to-ansible conversion |

### Curated Skill Directories

For discovering more skills:

| Directory | Stars | Description |
|----------|-------|-------------|
| [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) | 7.5K+ | 300+ agent skills directory, multi-platform |
| [travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | 7.3K+ | Curated Claude skills list |

---

## IaC Skill Risk Matrix

Not all skills carry equal risk. Map each to the right controls:

```
                    READ-ONLY                              WRITE / EXECUTE
              ┌──────────────────────────────┬──────────────────────────────────┐
              │                              │                                  │
  GUIDANCE    │  LOW RISK                    │  MEDIUM RISK                     │
  (no tool    │  terraform-style-guide       │  refactor-module (state mv)      │
   calls)     │  pulumi-best-practices       │  terraform-stacks (deploy)       │
              │  aws-iac-mcp-server          │  migration skills (state import) │
              │                              │                                  │
              │  Controls: code review only  │  Controls: + validation gates    │
              ├──────────────────────────────┼──────────────────────────────────┤
              │                              │                                  │
  EXECUTION   │  MEDIUM RISK                 │  VERY HIGH RISK                  │
  (tool       │  terraform-mcp (registry)    │  cfn-mcp-server (CRUD 1100+)    │
   calls)     │  pulumi-mcp (query stacks)   │  terraform-mcp (apply/destroy)  │
              │  checkov scanning            │  aws terraform-mcp (apply)       │
              │                              │                                  │
              │  Controls: + credential      │  Controls: + sandbox + approval  │
              │  scoping + audit             │  gates + separate prod creds     │
              └──────────────────────────────┴──────────────────────────────────┘
```

---

## Skill Supply Chain Security

Skills are not "just documentation." They are **executable procedures** that induce tool calls, shell commands, and credential exposure. The industry learned this the hard way:

- **Malicious skill marketplaces**: mass uploads of stealer malware disguised as useful skills
- **"Markdown is an installer"**: a skill's setup instructions led to malicious infrastructure and a staged payload chain
- **Prompt injection via skills**: instructions embedded in SKILL.md can manipulate agent behavior
- **Remote content fetching**: skills that `curl` external URLs at runtime are indirect injection vectors

### Vetting Checklist for IaC Skills

```
BEFORE INSTALLING ANY SKILL:

[ ] Source is vendor-official or from a known, trusted publisher
[ ] Reviewed SKILL.md for suspicious instructions (fetch URLs, run scripts, disable security)
[ ] No embedded scripts that execute on install or setup
[ ] No references to external URLs that fetch content at runtime
[ ] Pinned to a specific version/commit hash (never "latest")
[ ] If from a marketplace: check scanning results, publisher history, star count
[ ] If it touches credentials: verify it recommends OIDC/short-lived tokens, not static keys
[ ] If it can run apply/destroy: ensure it's behind approval gates in your system

ONGOING:

[ ] Re-scan periodically (skills can be updated maliciously)
[ ] Monitor for CVEs in skill dependencies
[ ] Maintain an internal allowlist of approved skills
[ ] Block unknown publishers by default
```

### The Safe Default: Read-Only + PR-First

For IaC skills, the safe adoption posture is:

1. **In development**: Skills can generate code and propose diffs
2. **In CI**: Skills can run `plan`/`preview`/`validate` and produce reports
3. **In production**: Skills open PRs — they never `apply` directly
4. **For apply/destroy**: Require explicit "break-glass" approval with separate credentials

This matches how the vendor skills are designed: HashiCorp's skills produce guidance and encourage plan verification; Pulumi's migration skills mandate "zero-diff preview" before considering success.

---

## Skill Hydration: Loading Skills at Runtime

Before the agent starts, the worker writes skill files to the working directory based on the agent's configuration:

```typescript
interface SkillDefinition {
  name: string;
  markdown: string;    // SKILL.md content
  clientCode: string;  // client.ts content
  category: 'iac' | 'git' | 'cloud' | 'communication';
}

async function hydrateSkills(
  workDir: string,
  agentConfig: AgentConfig,
  orgPolicies: PolicyDigest
): Promise<number> {
  const skillsDir = path.join(workDir, '.claude', 'skills');
  await fs.mkdir(skillsDir, { recursive: true });

  let count = 0;

  for (const skill of getSkillsForAgent(agentConfig)) {
    // Check if agent is allowed to use this skill
    if (agentConfig.allowedTools.length > 0 &&
        !agentConfig.allowedTools.includes(skill.name)) {
      continue;
    }

    const skillDir = path.join(skillsDir, skill.name);
    await fs.mkdir(skillDir, { recursive: true });

    // Inject policy constraints into the SKILL.md
    const enrichedMarkdown = injectPolicyConstraints(
      skill.markdown,
      orgPolicies
    );

    await fs.writeFile(path.join(skillDir, 'SKILL.md'), enrichedMarkdown);
    await fs.writeFile(path.join(skillDir, 'client.ts'), skill.clientCode);
    count++;
  }

  return count;
}
```

---

## Tool Allow/Deny Lists

Restrict what tools each agent type can access:

```typescript
// Agent configuration — stored in database
interface AgentConfig {
  slug: string;                 // 'infrastructure', 'pr-reviewer', etc.
  allowedTools: string[];       // Empty = all allowed
  maxTurns: number;             // Iteration limit (e.g., 50)
  producesCodeChanges: boolean; // Whether this agent modifies files
  requiresRepository: boolean;  // Whether git context is needed
}

// Example configurations
const AGENT_CONFIGS = {
  'pr-reviewer': {
    allowedTools: ['git-diff', 'pr-comment', 'iac-lint'],  // Read-only + comment
    maxTurns: 20,
    producesCodeChanges: false,
    requiresRepository: true,
  },
  'compliance-remediation': {
    allowedTools: [],  // All tools (needs to write code, run plans, create PRs)
    maxTurns: 50,
    producesCodeChanges: true,
    requiresRepository: true,
  },
  'drift-detection': {
    allowedTools: ['terraform-plan', 'drift-verification', 'notify-slack'],
    maxTurns: 10,
    producesCodeChanges: false,
    requiresRepository: true,
  },
};
```

---

## Alternatives for Skill/Tool Systems

### LangChain Tools

```python
from langchain.tools import tool

@tool
def terraform_plan(pipeline_id: str) -> str:
    """Trigger a Terraform plan and return the result."""
    result = requests.post(f"{API_URL}/run-pipeline", json={"id": pipeline_id})
    return result.json()

# Register with agent
agent = create_react_agent(llm, [terraform_plan, ...])
```

### OpenAI Function Calling

```typescript
const tools = [{
  type: 'function',
  function: {
    name: 'terraform_plan',
    description: 'Trigger a Terraform plan execution',
    parameters: {
      type: 'object',
      properties: {
        pipeline_id: { type: 'string', description: 'The pipeline to run' },
      },
      required: ['pipeline_id'],
    },
  },
}];

// Agent loop
const response = await openai.chat.completions.create({
  model: 'gpt-4',
  messages,
  tools,
});

if (response.choices[0].message.tool_calls) {
  for (const call of response.choices[0].message.tool_calls) {
    const result = await executeToolCall(call);
    messages.push({ role: 'tool', content: result, tool_call_id: call.id });
  }
}
```

### MCP (Model Context Protocol)

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server';

const server = new McpServer({ name: 'infra-tools' });

server.tool('terraform_plan', { pipelineId: z.string() }, async ({ pipelineId }) => {
  const result = await triggerPlan(pipelineId);
  return { content: [{ type: 'text', text: JSON.stringify(result) }] };
});
```

---

## Key Takeaways

1. **Skills = documentation + typed code** — the agent learns by reading and acts by executing
2. **Internal HTTP server** is the bridge between sandboxed skill code and platform capabilities
3. **Allow/deny lists** constrain what each agent type can do
4. **Hydrate at runtime** — inject only the skills needed for this specific task
5. **Policy enrichment** — weave org-specific constraints into skill documentation

---

## Next Chapter

[Chapter 4: Sandboxed Execution →](./04-sandboxed-execution.md)
