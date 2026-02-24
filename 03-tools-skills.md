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

**2. Credential injection via environment**: CLIs read credentials from environment variables. Inject short-lived tokens at runtime rather than baking anything into the image:

```
AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY + AWS_SESSION_TOKEN  (STS)
AZURE_CLIENT_ID + AZURE_CLIENT_SECRET + AZURE_TENANT_ID       (Service Principal)
GOOGLE_APPLICATION_CREDENTIALS                                  (Service Account)
GITHUB_TOKEN                                                    (Installation token)
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

Write skills as files that the agent reads and executes. Popularized by Claude Code's `.claude/skills/` convention:

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

### 2. MCP (Model Context Protocol)

Anthropic's open standard for connecting agents to external tools and data sources. Tools are registered as typed functions served over stdio or HTTP:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server';

const server = new McpServer({ name: 'infra-tools' });

server.tool('terraform_plan',
  { pipelineId: z.string() },
  async ({ pipelineId }) => {
    const result = await triggerPlan(pipelineId);
    return { content: [{ type: 'text', text: JSON.stringify(result) }] };
  }
);
```

**Pros**: Standard protocol; typed schemas; supported by multiple LLM providers; growing ecosystem of pre-built servers.
**Cons**: Less room for rich documentation; tool descriptions are typically one-liners.

### 3. LangChain / LangGraph Tools

Python-native tool registration with decorators:

```python
from langchain.tools import tool

@tool
def terraform_plan(pipeline_id: str) -> str:
    """Trigger a Terraform plan and return the result."""
    result = requests.post(f"{API_URL}/run-pipeline", json={"id": pipeline_id})
    return result.json()

agent = create_react_agent(llm, [terraform_plan, ...])
```

**Pros**: Simple Python-native API; large ecosystem; works with any LLM.
**Cons**: Python-only; tool docs limited to docstrings; no built-in isolation.

### 4. OpenAI-Style Function Calling

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
| **Skills as files** | Rich (full markdown) | Via code | Per-file | Growing (Claude, community) |
| **MCP** | Schema + description | Strong (Zod) | Per-server | Growing fast (vendor-backed) |
| **LangChain tools** | Docstrings | Python types | None (same process) | Largest |
| **Function calling** | Schema only | JSON Schema | Your responsibility | Universal |

### Combining Approaches: Skills + MCP + CLIs

These approaches aren't mutually exclusive — they layer together:

- **CLIs** are the foundation. The agent runs `terraform`, `aws`, `git` in its sandbox. This is the raw capability layer.
- **Skills** define *how* to use those CLIs well. A skill says "when writing Terraform, follow these conventions, validate with plan, max 10 iterations." It's the best-practices layer.
- **MCP** connects to *live data and APIs*. An MCP server gives the agent access to the Terraform Registry, workspace state, or cloud resource inventory. It's the data layer.

HashiCorp's Claude plugin demonstrates all three: CLI tools installed in the environment, skills for Terraform code generation patterns, and an MCP server for live Terraform Registry and Cloud API access.

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
    allowedTools: [],  // All tools (needs to write code, run plans, create PRs)
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

The key principle: **start restrictive, expand as needed.** A drift detection agent doesn't need git push. A PR reviewer doesn't need cloud credentials.

---

## Real-World IaC Skills Ecosystem

Vendors, cloud providers, and the community have published production-grade skills and MCP servers. You can adopt them directly, use them as templates, or just study the patterns.

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

The agent loads full instructions only when a skill is activated — names and descriptions stay in context for discovery.

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

## Next Chapter

[Chapter 4: Sandboxed Execution →](./04-sandboxed-execution.md)
