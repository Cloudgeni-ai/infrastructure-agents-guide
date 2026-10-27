# Chapter 3: Tools, CLIs & Skills

> How agents gain capabilities — from CLI tools to structured skill systems.

---

## The Tool Landscape

Infrastructure agents need access to real tools: `terraform`, `aws`, `az`, `gcloud`, `kubectl`, `helm`, `git`, `checkov`, `tflint`, and dozens more. The question isn't whether agents need tools — it's how to provide access safely.

There's a spectrum from raw shell access (maximum power, minimum safety) to structured skill systems (constrained power, maximum safety). Most production systems use a combination.

---

## CLI Tools: The Foundation

Every infrastructure agent ultimately needs to run CLI commands. The IaC ecosystem is built around CLIs — there's no Terraform API that replaces `terraform plan`, no Azure SDK that replaces everything `az` does. The agent's sandbox image is its tool inventory.

### Essential CLI Tooling

| Category | Tools | Purpose |
|----------|-------|---------|
| **IaC** | `terraform`, `tofu`, `pulumi`, `bicep` | Write, plan, validate infrastructure code |
| **Cloud CLIs** | `aws`, `az`, `gcloud`, `oci` | Query resources, manage configurations |
| **Kubernetes** | `kubectl`, `helm`, `kustomize` | Cluster operations, deployments |
| **Git** | `git`, `gh`, `glab` | Version control, PR creation |
| **Security** | `checkov`, `tflint`, `trivy`, `prowler` | Static analysis, compliance scanning |
| **Utilities** | `jq`, `yq`, `curl`, `envsubst` | Data parsing, HTTP calls, templating |

### Making CLI Access Safe

Giving an agent unrestricted `bash` is the simplest approach and the most dangerous. Here's how to tighten it:

**1. Allowlisted commands**: Only permit specific binaries. The agent can run `terraform plan` but not arbitrary `curl` or `bash -c`.

**2. Credential injection via environment**: CLIs read credentials from environment variables. Use this only for short-lived tokens injected at runtime, never for long-lived keys or client secrets, and never bake anything into the image:

```
AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY + AWS_SESSION_TOKEN  (STS session, minutes to hours)
AZURE_CLIENT_ID + AZURE_TENANT_ID + AZURE_FEDERATED_TOKEN_FILE (workload identity federation)
GOOGLE_APPLICATION_CREDENTIALS                                  (short-lived token or external-account config, not a key file)
GITHUB_TOKEN                                                    (installation token, 1h)
```

See [Chapter 5: Credential Management](./05-credential-management.md) for the full credential broker pattern.

**3. Version pinning**: Pin every tool to a specific version in the sandbox Dockerfile. A Terraform upgrade from 1.8 to 1.9 can change plan output format, break providers, or introduce new behaviors.

**4. Machine-readable output**: CLI output is messy — progress bars, color codes, pagination, warnings mixed with data. Always prefer structured output:

```bash
terraform plan -json              # Structured JSON instead of human-readable
terraform show -json              # JSON state output
aws ... --output json             # JSON instead of table format
az ... --output json              # JSON instead of table format
kubectl get ... -o json           # JSON output
checkov -o json                   # JSON findings
```

When JSON output isn't available, the model parses text — which works but is fragile across tool versions.

**5. Typed CLI wrappers**: For high-risk or frequently-used commands, wrap the CLI in a typed function that validates inputs and parses outputs:

```typescript
interface PlanResult {
  success: boolean;
  resources: { address: string; action: 'create' | 'update' | 'delete' | 'no-op' }[];
  error?: string;
}

async function terraformPlan(workDir: string): Promise<PlanResult> {
  const { stdout, exitCode } = await exec(
    'terraform plan -json -no-color',
    { cwd: workDir }
  );
  return parsePlanJson(stdout);
}
```

This is the bridge between raw CLI and structured capabilities — the agent calls a typed function, but under the hood it's running `terraform plan`.

### CLI vs Structured Tools: When to Use Which

| Use CLI Directly | Use Typed Wrappers / Skills |
|------------------|---------------------------|
| One-off queries (`aws s3 ls`) | Core agent loop operations (plan, validate, PR) |
| Tool has good `-json` output | Output needs parsing or error classification |
| Low-risk read-only commands | High-risk write operations |
| Exploratory/debugging tasks | Repeated operations that must be reliable |

---

## Structured Capability Systems

Beyond CLIs, there are several ways to give agents higher-level, safer capabilities. A good system provides:

- **Documentation** — The agent understands what a capability does, when to use it, and what constraints apply
- **Typing** — Inputs and outputs have schemas, not arbitrary strings
- **Scoping** — Different agents get different capabilities. A PR reviewer shouldn't have `terraform apply`
- **Auditability** — Every invocation is logged with inputs, outputs, and context

### 1. Skills as Files (Document-First)

Write skills as files that the agent reads and executes. The layout below follows the [Agent Skills](https://agentskills.io) open standard, which Claude Code, Codex and other runtimes load from their own skill directories (for example `.claude/skills/`):

```
skills/
├── terraform-plan/
│   └── SKILL.md     # Instructions, constraints, examples
├── git-operations/
│   └── SKILL.md
└── cloud-credentials/
    └── SKILL.md
```

The agent reads `SKILL.md` to understand when and how to use the skill. Instructions can include constraints ("never run on production without approval"), error handling ("if plan times out, retry once"), and examples.

```markdown
# Terraform Plan Skill

## When to Use
After making changes to Terraform files, to validate your changes
produce the expected plan output.

## Constraints
- ALWAYS check the plan output before creating a PR
- If the plan shows unexpected changes, STOP and ask the user
- Maximum 10 plan iterations per session

## Error Handling
- If the pipeline fails, check the error output for syntax errors
- If authentication fails, request fresh credentials
```

**Pros**: Rich documentation; agent can reason about when to use a skill; easy to version-control and review; skills can be injected per-agent at runtime.
**Cons**: Requires the agent to parse and follow instructions (model-dependent); less structured than typed function calls.

### 2. Subagents as Role-Scoped Capability Bundles

Subagents package an agent role into a reusable definition: prompt, model, tool scope, permission mode, max turns, skills, hooks, and MCP access. In Claude Code, these can live at multiple scopes:

| Scope | Typical Use |
|-------|-------------|
| **Managed / enterprise** | Organization-wide baseline roles and restrictions |
| **Project** (`.claude/agents/`) | Repository-specific reviewers, remediators, and workflow helpers |
| **User** (`~/.claude/agents/`) | Personal helpers available across projects |
| **Plugin / marketplace** | Packaged roles distributed with related skills and integrations |

Managed subagents are especially relevant for infrastructure teams because they can cover several layers at once:

- **Role layer** — `security-reviewer`, `terraform-reviewer`, `drift-remediator`, `incident-triage`
- **Prompt layer** — role-specific instructions, review criteria, and escalation rules
- **Runtime layer** — model choice, effort level, max turns, and delegation boundaries
- **Capability layer** — allowed tools, preloaded skills, and scoped MCP servers
- **Governance layer** — enterprise-managed definitions can override unsafe project or user variants

This makes managed subagents a practical enterprise control for standardizing how agents behave across repositories. They do **not** replace sandboxing, credential brokering, approval gates, deterministic validation, or action trails. Treat them as a high-level role/capability policy that still needs lower-level enforcement.

For multi-tenant products, the same pattern applies even if you don't use Claude Code directly: store resolved agent definitions as versioned artifacts, review them like policy, and make the effective role configuration visible per run.

### 3. MCP (Model Context Protocol)

The open standard for connecting agents to external tools and data sources. In practice, MCP has grown from "typed tool bridge" into a broader interoperability layer for **tools, resources, prompts, and optional extensions**. MCP servers are typically exposed over stdio or HTTP transports:

```typescript
import { McpServer } from '@modelcontextprotocol/server';
import * as z from 'zod/v4';

const server = new McpServer({ name: 'infra-tools', version: '1.0.0' });

server.registerTool(
  'terraform_plan',
  {
    description: 'Trigger a Terraform plan for a registered pipeline',
    inputSchema: z.object({ pipelineId: z.string() }),
  },
  async ({ pipelineId }) => {
    const result = await triggerPlan(pipelineId);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }
);
```

The [`2026-07-28` specification](https://modelcontextprotocol.io/specification/2026-07-28) is a significant operational change:

- the core protocol is **stateless** and each request carries its version and capabilities
- `server/discover` replaces the mandatory initialization handshake
- application state is represented by explicit handles rather than transport sessions
- list and resource responses can declare cache lifetimes and scope
- W3C trace context has standard `_meta` keys
- long-running **Tasks**, **Skills over MCP**, and **MCP Apps** are opt-in extensions rather than core assumptions

This makes horizontally scaled HTTP servers easier to route and cache, but it is a breaking migration from session-oriented `2025-11-25` implementations. Tasks were experimental in that earlier revision and changed shape when they moved to an extension; do not build a workflow engine that assumes every client implements them.

**Pros**: Standard protocol; typed schemas; supported by multiple LLM providers; fast-growing ecosystem of vendor and community servers.
**Cons**: Still requires operational discipline: version pinning, auth hardening, allowlists, and compatibility testing across clients.

**Operational guidance**:

- Pin both the **server version** and the **MCP spec version** you tested against
- Test upgrade and downgrade behavior explicitly; do not silently lose required extensions
- Treat registry presence as **discoverability**, not trust
- Prefer an **internal registry or allowlist** for production
- Treat tool descriptions and annotations as untrusted input until reviewed
- For HTTP servers, follow the current [MCP authorization profile](https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization): use protected-resource metadata, PKCE, issuer validation, resource indicators, token audience validation, and step-up scopes
- Never pass an MCP access token through to an upstream API. The MCP server is one OAuth resource; downstream services require separate audience-bound credentials

### 4. A2A (Agent-to-Agent Protocols)

MCP is not the right abstraction for every integration. When one agent needs to delegate to another autonomous application with its own memory, approvals, artifacts, and lifecycle, use an **agent-to-agent protocol** such as **[A2A 1.0](https://a2a-protocol.org/latest/specification)**.

A2A 1.0 defines agent discovery through Agent Cards, stateful Tasks, Messages, Artifacts, streaming and push updates, and HTTP/JSON, JSON-RPC, and gRPC bindings. That is a more mature interoperability surface than the early 0.x protocol, but production clients should still send an explicit `A2A-Version`, verify Agent Card identity/signatures where available, and authorize each remote agent as its own security principal.

Use **A2A** when:

- the remote system owns its own credentials and policy enforcement
- work is long-running and task-oriented
- responses may include artifacts, status changes, follow-up questions, or resumable work items

Use **MCP** when:

- you are exposing tools, resources, or prompts to a model runtime
- calls are narrow, typed, and subordinate to the calling agent

In practice, many infra systems use **MCP inside a worker** and **A2A across product boundaries**. Both protocols, along with [AGENTS.md](https://agents.md) (a plain markdown file of repository instructions for coding agents), are now hosted by the Linux Foundation's [Agentic AI Foundation](https://aaif.io), which makes them reasonable long-term bets. Put repository conventions in `AGENTS.md` so every runtime reads the same instructions, and keep task-specific procedures in skills.

### 5. LangChain / LangGraph Tools

Tool registration with decorators (Python shown; LangChain.js has an equivalent `tool()` helper):

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def terraform_plan(pipeline_id: str) -> str:
    """Trigger a Terraform plan and return the result."""
    result = requests.post(f"{API_URL}/run-pipeline", json={"id": pipeline_id})
    return result.json()

agent = create_agent(llm, tools=[terraform_plan, ...])
```

**Pros**: Simple API in Python and TypeScript; large ecosystem; works with any LLM.
**Cons**: Tool docs limited to docstrings or descriptions; no built-in isolation.

### 6. OpenAI-Style Function Calling

Define tools as JSON schemas, let the model generate structured arguments:

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
```

**Pros**: Model generates validated JSON; clean separation between schema and execution.
**Cons**: Schema-only documentation (no rich instructions); tied to providers that support function calling.

### Comparison

| Approach | Documentation | Typing | Isolation | Ecosystem |
|----------|--------------|--------|-----------|-----------|
| **CLI tools** (direct) | Man pages / `--help` | None (strings) | Sandbox-level | Entire IaC ecosystem |
| **Typed CLI wrappers** | Code comments | Strong | Sandbox-level | Build your own |
| **Skills as files** | Rich (full markdown) | Via code | Per-file | Growing (open standard, many runtimes) |
| **Subagents / agent roles** | Role prompt + frontmatter/config | Via configured tools | Per-agent/session | Growing in Claude-style runtimes |
| **MCP** | Schema + description | Strong (JSON Schema) | Per-server/request | Mature core; extensions vary |
| **A2A 1.0** | Agent Cards + task/artifact schema | Strong | App boundary | Versioned interoperability standard |
| **LangChain tools** | Docstrings | Python/TS types | None (same process) | Largest |
| **Function calling** | Schema only | JSON Schema | Your responsibility | Universal |

### Combining Approaches: Subagents + Skills + MCP + A2A + CLIs

These approaches aren't mutually exclusive — they layer together:

- **CLIs** are the foundation. The agent runs `terraform`, `aws`, `git` in its sandbox. This is the raw capability layer.
- **Skills** define *how* to use those CLIs well. A skill says "when writing Terraform, follow these conventions, validate with plan, max 10 iterations." It's the best-practices layer.
- **Subagents / managed agent roles** package *who* is doing the work: role prompt, model, allowed tools, skills, MCP access, and safety posture. They can cover parts of the runtime, capability, and policy layers at the same time.
- **MCP** connects to *live data and APIs*. An MCP server gives the agent access to the Terraform Registry, workspace state, or cloud resource inventory. It's the data layer.
- **A2A** connects to *other autonomous systems*. A remote remediation agent, ticketing copilot, or approval service can own its own state while still participating in a broader workflow.

HashiCorp's Claude plugin demonstrates three of these layers directly: CLI tools installed in the environment, skills for Terraform code generation patterns, and an MCP server for live Terraform Registry and Cloud API access.

---

## Version the Effective Tool Policy

The configured catalog is not enough for audit or recovery. At session creation, resolve the effective tools from platform defaults, organization policy, repository config, user selection, and temporary grants, then persist an immutable policy revision:

```typescript
interface EffectiveToolPolicy {
  revision: string;                 // content hash or immutable version
  tools: {
    serverId: string;
    toolName: string;
    mode: 'allow' | 'deny' | 'approval-required';
    credentialGrantId?: string;     // reference only; never the secret
  }[];
  resolvedAt: string;
  resolvedFrom: string[];           // policy/config revisions
}
```

Every tool call, approval, retry, and recovered run must bind to that revision. Mid-session policy changes create a new audited revision; they do not silently rewrite what an already-approved attempt was allowed to do. Credentials for third-party MCP servers should be write-only to users after creation, rotatable, scoped per session or connection, and redacted at both the tool boundary and persistence boundary.

This is one of the most useful patterns to backport from [Opengeni's MCP surfaces](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/mcp-surfaces.md): tool discovery, tool authorization, credential authority, and human approval are separate decisions even when the UI presents one catalog.

---

## Tool Allow/Deny Lists

Restrict what tools each agent type can access:

```typescript
// Example agent configurations
const AGENT_CONFIGS = {
  'pr-reviewer': {
    allowedTools: ['git-diff', 'pr-comment', 'iac-lint'],
    maxTurns: 20,
    producesCodeChanges: false,
  },
  'compliance-remediation': {
    allowedTools: ['*'],  // All tools (needs to write code, run plans, create PRs)
    maxTurns: 50,
    producesCodeChanges: true,
  },
  'drift-detection': {
    allowedTools: ['terraform-plan', 'drift-verification', 'notify-slack'],
    maxTurns: 10,
    producesCodeChanges: false,
  },
};
```

Use `['*']` to mean all tools and treat an empty list as no tools, the same convention as [Chapter 8](./08-policy-guardrails.md#tool-allowdeny-lists). An empty list that silently means "everything" is the kind of default that turns a configuration typo into full access.

The key principle: **start restrictive, expand as needed.** A drift detection agent doesn't need git push. A PR reviewer doesn't need cloud credentials.

---

## Real-World IaC Skills Ecosystem

Vendors, cloud providers, and the community have published production-grade skills and MCP servers. You can adopt them directly, use them as templates, or just study the patterns.

### The Agent Skills Format

[Agent Skills](https://agentskills.io) began at Anthropic and is now an open standard with its own specification; MCP also has an official Skills extension so a server can ship skills over the same connection as its tools. The format is structurally simple: a directory with a required `SKILL.md` file containing YAML frontmatter (name, description) plus optional scripts and assets. The frontmatter follows a **three-level progressive disclosure** model:

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

The agent loads full instructions only when a skill is activated — names and descriptions stay in context for discovery.

For multi-tenant products, resolve skills and role definitions in **layers**:

1. vendor-curated or platform-curated base skills
2. organization-level overrides and custom skills
3. repository-local skills for team-specific workflows

Version the resolved skill and subagent set, cache it, and make the active manifest inspectable in logs and UI. This matters as much as the individual file content.

### Vendor-Official Skills

#### HashiCorp — [hashicorp/agent-skills](https://github.com/hashicorp/agent-skills)

The most complete vendor skill set for Terraform. It is organized into two plugins, `plugins/terraform` and `plugins/packer`, each installable in Claude Code and Codex:

| Plugin | Skills Included | Focus |
|--------|----------------|-------|
| `terraform` (authoring) | `terraform-style-guide`, `terraform-test`, `azure-verified-modules`, `refactor-module`, `terraform-stacks` | Writing correct HCL, testing, module extraction, HCP Terraform Stacks |
| `terraform` (policy and import) | `terraform-policy`, `terraform-search-import` | Policy authoring, discovering and importing existing resources |
| `terraform` (providers) | `new-terraform-provider`, `provider-resources`, `provider-actions`, `provider-ephemeral-resources`, `provider-test-patterns`, and related `provider-*` skills | Building and testing Terraform providers |
| `packer` | `aws-ami-builder`, `azure-image-builder`, `windows-builder`, `push-to-registry` | Building and publishing machine images |

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

The AWS MCP monorepo removed its deprecated `cfn-mcp-server` and `terraform-mcp-server` in April 2026 and marks `ccapi-mcp-server` (Cloud Control API CRUD) as deprecated. AWS points IaC users to `aws-iac-mcp-server`, and Terraform users to HashiCorp's official Terraform MCP server above. AWS also describes its newer Agent Toolkit for AWS as the successor to the awslabs servers, so check the repository before pinning anything. The IaC-relevant servers today:

| Server | Capabilities | Risk Level |
|--------|-------------|------------|
| **aws-iac-mcp-server** | CloudFormation/CDK docs search, template validation, compliance checks, deployment troubleshooting | Low (read-only, guidance) |
| **aws-api-mcp-server** | Runs arbitrary AWS CLI commands with whatever credentials it is given | **Very High** (any API call the role allows) |
| **ccapi-mcp-server** (deprecated) | Direct CRUD of AWS resource types via Cloud Control API | **Very High** (can create/delete resources) |

> **Critical distinction**: `aws-iac-mcp-server` is guidance-only and low risk. `aws-api-mcp-server` is effectively a shell for the AWS API: its blast radius is exactly the IAM role you hand it, so give it a read-only role or route mutating calls through the same approval gates as any tool that can change live infrastructure. Do not adopt the deprecated `ccapi-mcp-server` for new work.

#### Pulumi MCP Server

Pulumi's documented path is a [hosted remote MCP server](https://www.pulumi.com/docs/ai/mcp-server/) at `https://mcp.ai.pulumi.com/mcp`, connected over HTTP with an OAuth-style browser flow in which you authenticate and pick a Pulumi organization. It interacts with Pulumi Cloud for stack preview, deploy, output retrieval, and registry queries. A local stdio server (`@pulumi/mcp-server` on npm, driven by a `PULUMI_ACCESS_TOKEN`) still exists for offline or self-managed setups; scope that token to one organization and treat it like any other long-lived secret.

#### Other Official Infrastructure Servers

| Vendor | Server | Infrastructure-Relevant Surface |
|--------|--------|---------------------------------|
| Microsoft | [Azure MCP Server](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/overview) | Azure resource discovery and operations using Entra ID, managed identity, and Azure RBAC |
| Google | [Google MCP catalog](https://github.com/google/mcp) and [gcloud MCP](https://github.com/googleapis/gcloud-mcp) | Managed Google/Google Cloud endpoints plus `gcloud`-backed resource operations |
| Spacelift | [Spacelift MCP](https://docs.spacelift.io/concepts/intelligence/spacelift-mcp/connecting) | OAuth-protected access to IaC stacks, runs, and platform workflows |

Treat a vendor badge as provenance, not blanket approval. Enumerate the exact tools enabled, separate read and mutation scopes, and route deployment-capable calls through the same plan/approval pipeline as direct CLI operations.

### Community Skills (Notable)

| Repository | Focus | Why It's Notable |
|-----------|-------|-----------------|
| [antonbabenko/terraform-skill](https://github.com/antonbabenko/terraform-skill) | Terraform & OpenTofu | By Anton Babenko (prolific TF community contributor). Comprehensive single-file SKILL.md covering testing, modules, CI/CD, and production patterns |
| [akin-ozer/cc-devops-skills](https://github.com/akin-ozer/cc-devops-skills) | Multi-tool DevOps | Skills spanning Terraform, Terragrunt, Ansible, Kubernetes, Helm, CI/CD, and PromQL |
| [terramate-io/agent-skills](https://github.com/terramate-io/agent-skills) | Terraform, OpenTofu, Terramate | State splitting, drift reconciliation, stack management |
| [dirien/claude-skills](https://github.com/dirien/claude-skills) | Pulumi (TS/Go/Python) | Pulumi community skills emphasizing ESC + OIDC patterns |
| [realsigridjin/hello-ansible-skills](https://github.com/realsigridjin/hello-ansible-skills) | Ansible | Playbook development, debugging, shell-to-ansible conversion |

### Curated Skill Directories

For discovering more skills:

| Directory | Description |
|----------|-------------|
| [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) | Large multi-platform agent skills directory |
| [travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) | Curated Claude skills list |

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
  (tool       │  terraform-mcp (registry)    │  aws-api-mcp-server (any CLI)    │
   calls)     │  pulumi-mcp (query stacks)   │  ccapi-mcp-server (CRUD, depr.)  │
              │  checkov scanning            │  terraform-mcp (HCP TF runs)     │
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
[ ] Registry entry or marketplace presence is treated as discovery, not trust
[ ] Reviewed SKILL.md for suspicious instructions (fetch URLs, run scripts, disable security)
[ ] No embedded scripts that execute on install or setup
[ ] No references to external URLs that fetch content at runtime
[ ] Pinned to a specific version/commit hash (never "latest")
[ ] Pinned MCP server image/package/version if the skill depends on MCP
[ ] If from a marketplace: check scanning results, publisher history, star count
[ ] If it touches credentials: verify it recommends OIDC/short-lived tokens, not static keys
[ ] If it can run apply/destroy: ensure it's behind approval gates in your system

ONGOING:

[ ] Re-scan periodically (skills can be updated maliciously)
[ ] Monitor for CVEs in skill dependencies
[ ] Maintain an internal allowlist of approved skills
[ ] Mirror critical skills and MCP servers into an internal registry when possible
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

## Next Chapter

[Chapter 4: Sandboxed Execution →](./04-sandboxed-execution.md)

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
