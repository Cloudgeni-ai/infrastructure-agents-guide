# Chapter 6: The Data Plane — Giving Agents Context

> Why agents without pre-built context burn tokens discovering what you already know — and how to serialize infrastructure knowledge into formats agents can actually use.

---

## The Problem: Discovery Is Expensive

Ask an infrastructure agent to "fix the unencrypted S3 bucket in production." Without pre-built context, here's what happens:

```
Agent: calls aws s3api list-buckets                     # 200+ buckets, 15s
Agent: for each bucket, calls get-bucket-encryption      # 200 API calls, 90s
Agent: for each unencrypted, calls get-bucket-tagging    # more API calls
Agent: cross-references with terraform state             # terraform state pull, 30s
Agent: reads tf files to find the resource address       # grep across repo
Agent: finally has enough context to write a fix         # 3+ minutes burned
```

This is the cloud equivalent of running `grep` on 10,000 files. It works at human scale (one bucket). It breaks at infrastructure scale (hundreds of resources, multiple accounts, four clouds).

The costs compound:
- **API rate limits** — cloud providers throttle after a few hundred calls
- **Token waste** — the LLM processes pages of JSON to extract three fields
- **Latency** — minutes of API calls before the agent starts actual work
- **No relationships** — which VPC does this subnet belong to? The agent discovers this from scratch every time
- **No organizational context** — who owns this resource? Is there an ADR about it? The agent has no way to know

The fix: don't make agents discover what you already know. Build a data plane that collects, correlates, and serializes infrastructure knowledge *before* the agent needs it.

---

## What Agents Need to Know

Infrastructure agents don't just need cloud resource lists. They need layered context that spans multiple systems:

```
┌──────────────────────────────────────────────────────────┐
│                    CONTEXT LAYERS                        │
│                                                          │
│  Layer 1: Cloud Resources                                │
│  ┌────────────────────────────────────────────────────┐  │
│  │ What exists? EC2, S3, VPC, VNet, GKE, ...          │  │
│  │ What state is it in? Config, tags, status          │  │
│  │ How do resources relate? VPC→Subnet→Instance       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Layer 2: Infrastructure as Code                         │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Which resources are managed by IaC?                │  │
│  │ What's the Terraform address? Which repo/file?     │  │
│  │ Which resources are unmanaged (ClickOps)?          │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Layer 3: Security & Compliance                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │ What findings exist? Which resource is affected?   │  │
│  │ What framework? CIS, SOC2, ISO 27001               │  │
│  │ What's the remediation recommendation?             │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Layer 4: Cost & Usage                                   │
│  ┌────────────────────────────────────────────────────┐  │
│  │ How much does this resource cost?                  │  │
│  │ Is it underutilized? Rightsizing candidates?       │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  Layer 5: Organizational Knowledge                       │
│  ┌────────────────────────────────────────────────────┐  │
│  │ Who owns this? Which team, which service?          │  │
│  │ What are the naming conventions?                   │  │
│  │ Are there ADRs, runbooks, past incidents?          │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

The agent that has all five layers can go from "fix the unencrypted bucket" to "here's a PR that adds SSE-KMS to `aws_s3_bucket.prod_logs` in `modules/storage/main.tf`, using the existing KMS key from the same account, following the Payments team's encryption standard" — in seconds.

The agent that only has Layer 1 writes a technically correct but organizationally wrong fix, and takes minutes to get there.

---

## The Core Pattern: Scan, Correlate, Serialize

Regardless of implementation, every data plane follows the same pattern:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   COLLECT    │────▶│    STORE     │────▶│  CORRELATE   │────▶│  SERIALIZE   │
│              │     │              │     │              │     │              │
│ Cloud APIs   │     │ Normalized   │     │ Cloud ↔ IaC  │     │ Markdown for │
│ IaC state    │     │ resources,   │     │ Cloud ↔ Sec  │     │ LLM context  │
│ Compliance   │     │ findings,    │     │ Resource ↔   │     │              │
│ Cost data    │     │ cost data    │     │ Resource     │     │ JSON for     │
│ Docs/wikis   │     │              │     │              │     │ skill APIs   │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
```

### Step 1: Collect

Scan cloud accounts on a schedule and pull data from external systems:

| Data Source | How to Collect | Frequency |
|------------|---------------|-----------|
| Cloud resources | Provider APIs (describe/list calls) or asset inventory tools | Every 1-6 hours |
| IaC state | Parse Terraform state files, Bicep what-if output | On git push (webhook) |
| Compliance findings | Security scanners (Prowler, Checkov, Trivy, provider-native) | Nightly or on-demand |
| Cost data | Cloud billing APIs (Cost Explorer, Cost Management) | Daily |
| Resource relationships | Extract during cloud resource scan | With resource scan |
| Documentation | Wiki APIs (Confluence, Notion), git repos (READMEs, ADRs) | Daily or on change |

### Step 2: Store with a Normalized Model

Every cloud resource, regardless of provider, should normalize to a common structure. The specifics depend on your implementation, but the key fields are:

```typescript
// Core identity — same across all providers
interface NormalizedResource {
  providerResourceId: string;    // AWS ARN, Azure Resource ID, GCP self-link
  providerResourceType: string;  // 'AWS_S3_BUCKET', 'Microsoft.Storage/storageAccounts'
  providerResourceName: string;  // Human-readable name
  provider: string;              // 'AWS', 'AZURE', 'GCP', 'OCI'
  region: string;

  // Full configuration as JSON — expensive to scan, cheap to query
  properties: Record<string, unknown>;
  tags: Record<string, string>;
  status: string;

  // IaC correlation
  managedBy?: {
    tool: string;              // 'terraform', 'bicep', 'pulumi'
    address: string;           // 'aws_s3_bucket.prod_logs'
    repository: string;        // 'github.com/org/infra'
    filePath: string;          // 'modules/storage/main.tf'
  };
}
```

The `properties` field is critical: it stores the full resource configuration as the provider returned it. The agent reads this from a database query instead of making a live API call.

### Step 3: Correlate

Correlation is what turns a flat list of resources into actionable context:

**Cloud ↔ IaC:** Match cloud resources to the code that manages them by comparing provider resource IDs (ARNs, Azure Resource IDs) against IaC state files. This tells the agent *where to make the fix* — which repo, which file, which Terraform address. Resources without an IaC match are unmanaged (created manually or via ClickOps), and the agent handles them differently: import into IaC or recommend deletion.

**Cloud ↔ Compliance:** Compliance findings reference cloud resources by their provider ID. Link them so the agent can go from "HIGH severity finding" to "this S3 bucket, in this repo, at this Terraform address" in one step.

**Resource ↔ Resource:** Infrastructure is a graph. A VPC contains subnets, subnets are secured by security groups, instances are connected to subnets. Store these relationships as an edge table so agents can traverse the graph: "what's connected to this VPC?" returns subnets, route tables, security groups, and instances — without any API calls.

### Step 4: Serialize for LLM Consumption

This is the step most implementations skip, and it matters more than you'd expect.

Raw cloud API responses are verbose JSON — nested structures, pagination tokens, provider-specific quirks, fields the agent doesn't need. An S3 bucket's full API response is ~2,000 tokens. The five fields the agent actually needs fit in ~50 tokens.

**Serialize to markdown, not JSON:**

```typescript
function serializeResourceForAgent(resource: NormalizedResource): string {
  const lines: string[] = [
    `#### ${resource.providerResourceName}`,
    `- **Type:** ${resource.providerResourceType}`,
    `- **Provider:** ${resource.provider} / ${resource.region}`,
    `- **Resource ID:** ${resource.providerResourceId}`,
  ];

  if (Object.keys(resource.tags).length > 0) {
    lines.push(`- **Tags:** ${JSON.stringify(resource.tags)}`);
  }

  if (resource.managedBy) {
    lines.push(`- **IaC:** \`${resource.managedBy.address}\` in ${resource.managedBy.repository}`);
  } else {
    lines.push(`- **IaC:** Not managed (no Terraform/Bicep resource found)`);
  }

  return lines.join('\n');
}
```

Why markdown over JSON:
- **Fewer tokens per fact** — no braces, brackets, or quote characters
- **Headers help the model navigate** — `#### prod-logs-bucket` is easier to parse than a nested JSON key
- **Selective** — include only what the agent needs, not the full cloud API response
- **Truncatable** — you can limit context to 2,000 characters and it still reads naturally

The same principle applies to every data type: compliance findings, drift diffs, cost reports. Serialize each into a compact, readable format before injecting it into the agent's context.

```typescript
function serializeFindingForAgent(finding: ComplianceFinding): string {
  return [
    `### ${finding.title}`,
    `**Severity:** ${finding.severity} | **Framework:** ${finding.framework} (${finding.controlId})`,
    `**Resource:** ${finding.resourceId}`,
    `**Remediation:** ${finding.remediationRecommendation}`,
  ].join('\n');
}
```

---

## Why This Matters for Agent Quality

Pre-built context doesn't just save tokens — it changes what agents can do.

### Without a data plane

The agent is blind. Every task starts with discovery. The agent spends most of its token budget and iteration count *finding things*, not *fixing things*. It can only work on resources it can discover through CLI tools, which means:
- It misses relationships (doesn't know the subnet's security group)
- It can't compare across accounts or regions
- It doesn't know if a resource is managed by IaC or was created manually
- It has no organizational context (team ownership, naming conventions, past incidents)

### With a data plane

The agent starts with everything it needs. It knows what the resource looks like, where it's defined in code, what compliance findings affect it, and what related resources exist. This means:
- **Fewer iterations** — the agent fixes on the first try instead of spending 5 turns discovering context
- **Better fixes** — the agent writes code that matches existing patterns because it can see the surrounding infrastructure
- **Safer decisions** — the agent knows if a resource is shared across environments and acts accordingly
- **Organizational awareness** — the agent follows naming conventions, respects ADRs, and references past remediation patterns

The difference is measurable. An agent with pre-built context typically resolves findings in 1-2 tool calls. Without it, the same agent takes 5-10 tool calls and produces lower-quality fixes.

---

## Resource Relationships: The Graph

Flat resource lists miss the most important context: **how resources relate to each other**.

An agent fixing a security group rule needs to know which VMs are behind it. An agent remediating a storage encryption finding needs to know if a KMS key already exists in the same account.

### Relationship Types

Define a vocabulary of directed edges between resources:

```
CONTAINS      VPC → Subnet, Resource Group → VM
BELONGS_TO    Subnet → VPC
SECURED_BY    NIC → NSG, Subnet → NSG
ROUTES_VIA    Subnet → Route Table
CONNECTED_TO  NIC → Subnet, Instance → VPC
PEERED_WITH   VPC → VPC
ATTACHED_TO   NIC → VM, Disk → VM
USES          Container App → Container Registry
ENCRYPTED_BY  Storage Account → Key Vault Key
HOSTS         App Service Plan → Web App
```

Store these as an edge table in your database: `source_id`, `target_id`, `relationship_type`. No graph database required — a simple adjacency list with `source_id`, `target_id`, and `type` handles thousands of resources efficiently. PostgreSQL recursive CTEs can traverse 2-3 hops, which covers most agent use cases.

### Why agents need the graph

When an agent gets a finding "S3 bucket `prod-logs` is not encrypted," it needs to:
1. Find the bucket (Layer 1)
2. Find the IaC address (Layer 2)
3. Check if a KMS key exists in the same account (graph traversal)
4. If yes, reference it. If no, create one.

Without the graph, step 3 requires additional API calls. With the graph, it's a database query: "show me all resources in this account with type `AWS_KMS_KEY`."

---

## Organizational Knowledge

Infrastructure agents don't just need cloud data. They need organizational context — and without it, they produce technically correct but organizationally wrong output.

An agent fixing a compliance finding in `prod-payments-vpc` should know:
- This VPC belongs to the Payments team
- The Payments team uses a specific naming convention for resources
- There's an ADR about why this VPC is network-isolated
- A similar finding was fixed last month — here's how

### Sources

| Source | What It Provides | How to Ingest |
|--------|-----------------|---------------|
| **Cloud resource tags** | Team ownership, environment, cost center | Extracted during cloud scan |
| **Git repositories** | IaC code, READMEs, ADRs, module docs | Clone and index on scan |
| **Wiki/docs** (Confluence, Notion) | Architecture docs, runbooks, naming conventions | API sync on schedule |
| **Service catalogs** (Backstage, ServiceNow) | Service ownership, dependencies, SLOs | API sync |
| **Incident management** (PagerDuty, OpsGenie) | Past incidents per service/resource | Webhook or API sync |
| **Past agent sessions** | Previous decisions, reasoning, patterns | Already in session store |
| **Custom policies** | Org-specific rules in plain language | Admin-authored, stored in DB |

### Ingestion Tiers

Not all knowledge is equally structured:

```
Tier 1: Structured data  → Cloud resources, IaC state, compliance findings, cost data
                           Store in database with typed columns.

Tier 2: Semi-structured   → READMEs, ADRs, wiki pages, module docs
                           Parse to text, extract metadata, index for search.

Tier 3: Unstructured       → Architecture diagrams, whiteboard photos, meeting recordings
                           Use vision models for images, speech-to-text for audio.
                           Convert to text and index alongside Tier 2.
```

Start with Tier 1 — it gives agents the most value per engineering effort. Add Tier 2 when agents are producing organizationally wrong fixes. Add Tier 3 only if you have significant unstructured knowledge that's critical for agent decisions.

### Making Knowledge Accessible

Two approaches, not mutually exclusive:

**Inject at dispatch time** — when a task is dispatched, the system pre-fetches relevant context and includes it in the agent's initial prompt. This works for focused tasks ("fix this finding") where you know what context the agent needs.

**Queryable skill APIs** — give agents skills to search and retrieve context on demand. This works for exploratory tasks ("explain this infrastructure") where the agent decides what to look up.

In practice, use both: inject the obvious context up front, and give the agent skills to pull more when it needs it.

---

## Exposing Data to Agents

Raw database access is too dangerous and too unstructured. Expose the data plane through typed APIs (tools, skills, or MCP resources) that return exactly what agents need. Example query patterns:

```typescript
// Cloud resources — filtered, paginated, pre-serialized
const resources = await queryResources({
  provider: 'AWS',
  resourceType: 'AWS_S3_BUCKET',
  managedByIaC: false,          // Only unmanaged resources
  search: 'prod',
});

// Compliance findings — linked to affected resources
const findings = await queryFindings({
  severity: 'HIGH',
  status: 'OPEN',
  resourceType: 'AWS_S3_BUCKET',
});

// Resource relationships — graph traversal
const graph = await queryResourceGraph({
  resourceId: 'arn:aws:ec2:us-east-1:123:vpc/vpc-abc',
  depth: 2,
});
// Returns: VPC → Subnets → Instances, VPC → IGW, VPC → Security Groups

// Organizational knowledge — searchable
const docs = await searchKnowledge({
  query: 'VPC isolation policy payments team',
  sources: ['wiki', 'adr', 'policy'],
});
```

The agent receives structured, pre-formatted responses. No raw CLI output, no pagination, no rate limiting. Whether you implement these as tool functions, MCP resources, or REST endpoints is an implementation choice — the key is typed, scoped access to each data layer.

---

## How Agents Consume Data: From Context Injection to RAG

You've collected, correlated, and serialized your infrastructure data. The remaining question: **how does it actually get into the agent's context?** There's a spectrum of approaches, and the right choice depends on data volume and query predictability.

### Level 1: Direct Context Injection (Small Data, Known Queries)

For focused tasks where you know what the agent needs upfront, just put the data in the prompt. No retrieval system needed.

```
System prompt:
  "You are a compliance remediation agent."

Injected context:
  ## Finding
  S3 bucket `prod-logs` missing server-side encryption.
  Severity: HIGH | Framework: CIS AWS v3.0 | Control: 2.1.1

  ## Affected Resource
  - Type: AWS_S3_BUCKET
  - Region: us-east-1
  - IaC: `aws_s3_bucket.prod_logs` in modules/storage/main.tf

  ## Related Resources
  - KMS Key `alias/infra-key` exists in same account (arn:aws:kms:us-east-1:...)
  - Bucket policy attached, no public access

  ## Organization Policies
  - All S3 buckets MUST use SSE-KMS with alias/infra-key
  - Never disable encryption, even temporarily

User prompt:
  "Fix this finding."
```

This works when:
- The data fits comfortably in context (~2,000-10,000 tokens of context)
- You can predict what the agent needs before it starts
- The task is scoped to a single resource or a small set of resources

For a compliance remediation agent, this covers 80%+ of tasks. The system pre-fetches the finding, the affected resource, related resources from the graph, the IaC location, and relevant policies — and injects all of it before the agent writes a single line of code.

**This is where most infrastructure agents should start.** If you can solve the problem with direct injection, you don't need search or RAG.

### Level 2: Filtered Database Queries (Medium Data, Structured Queries)

When the agent needs to explore beyond what was pre-injected — browsing multiple resources, comparing across accounts, or filtering by dimensions it decides at runtime — give it query tools backed by your database.

These are the typed API patterns from the previous section: `queryResources`, `queryFindings`, `queryResourceGraph`. The agent specifies filters (provider, resource type, region, severity, tags) and gets back pre-serialized results.

This works when:
- The agent needs to search across hundreds or thousands of resources
- Queries are structured — filtering by known dimensions, not free-text
- The data is in your database with indexed columns

For infrastructure data (Layers 1-4: resources, IaC, compliance, cost), filtered queries are almost always sufficient. You're searching by resource type, region, severity, tag values — not by semantic meaning. A SQL `WHERE` clause beats vector search for these queries.

### Level 3: Full-Text Search (Semi-Structured Data)

For organizational knowledge — ADRs, runbooks, wiki pages, module docs, past incident reports — the agent can't filter by structured dimensions because the data is text. Full-text search with keyword matching handles this.

```
Agent asks: "How did we handle VPC isolation for the payments team?"

Full-text search tool searches across:
  - ADRs in git repos
  - Wiki pages (Confluence, Notion)
  - Past agent session summaries
  - README files in infrastructure repos

Returns: Top 5 results ranked by relevance, with snippets.
```

PostgreSQL `tsvector` + `ts_rank`, Elasticsearch, or Meilisearch all work here. The queries are keyword-based ("VPC isolation payments"), and the corpus is typically small enough (hundreds to low thousands of documents) that basic full-text search finds the right results.

This works when:
- The corpus is small to medium (< 50K documents)
- Queries contain specific keywords that appear in the target documents
- Infrastructure vocabulary is precise enough for keyword matching (resource names, service names, team names)

### Level 4: RAG — Retrieval-Augmented Generation (Large Data, Semantic Queries)

RAG adds a vector search layer: documents are chunked, embedded into vectors, stored in a vector database, and retrieved by semantic similarity at query time. The retrieved chunks are injected into the agent's context alongside the user's prompt.

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│  User    │────▶│  Embed   │────▶│  Vector  │────▶│  Top-K   │
│  Query   │     │  Query   │     │  Search  │     │  Results  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                                        │
                                                        ▼
                                                   ┌──────────┐
                                                   │  Inject   │
                                                   │  into     │
                                                   │  Prompt   │
                                                   └──────────┘
```

Use RAG when:
- **Large documentation corpus** — thousands of wiki pages, hundreds of runbooks, years of incident reports
- **Semantic queries** — the agent asks questions where the answer uses different words than the query ("What's our policy on public-facing storage?" should find a doc about "S3 bucket ACLs")
- **Cross-domain knowledge** — the agent needs to connect information from different sources (an ADR, a past incident, and a naming convention doc)

**Don't use RAG when:**
- You're searching structured infrastructure data — use filtered queries instead
- Your corpus is small (< 1K documents) — full-text search is simpler and often more accurate
- Queries contain specific identifiers (ARNs, resource names, finding IDs) — keyword search is better for exact matches

Common RAG pitfalls for infrastructure data:
- **Chunking infrastructure docs poorly** — a Terraform module doc chunked mid-resource-block produces garbage context
- **Embedding infrastructure jargon** — generic embedding models may not distinguish "NSG" from "security group" from "firewall rule"
- **Over-indexing** — embedding every cloud API response creates noise. Only embed documents that contain organizational knowledge, not raw resource data

#### Vector Database Options

| Option | When to Use |
|--------|------------|
| **pgvector** (PostgreSQL extension) | Start here if you already use PostgreSQL. No new infrastructure. Handles millions of vectors. |
| **Pinecone / Weaviate / Qdrant** | Dedicated vector DBs when you need advanced features (hybrid search, filtering, multi-tenancy) at scale |
| **In-memory** (FAISS, HNSWlib) | Prototyping, small corpora (< 100K vectors), or when you want zero infrastructure |

### Level 5: MCP Resources (Browsable Data)

MCP (Model Context Protocol) supports a `resources` primitive — the agent can browse a list of available data sources and read specific ones on demand. This is useful for exposing structured data that the agent may or may not need.

```
MCP server exposes:
  resource://cloud/aws/us-east-1/s3-buckets       → list of all S3 buckets
  resource://compliance/findings/high              → all HIGH severity findings
  resource://docs/adrs/vpc-isolation               → specific ADR document
  resource://policies/encryption                   → encryption policy

Agent browses the resource list, reads what it needs.
```

This works as a complement to the other levels — MCP resources give the agent self-service access to data it might need, without pre-injecting everything.

### Choosing the Right Level

```
"Fix this specific S3 encryption finding"
  → Level 1: Direct injection. You know exactly what context is needed.

"Show me all unencrypted S3 buckets across our AWS accounts"
  → Level 2: Filtered query. Structured search by resource type + property.

"How did we handle encryption for the payments team last quarter?"
  → Level 3: Full-text search. Keyword search across ADRs and past sessions.

"What are the best practices for encrypting data at rest in our org?"
  → Level 4: RAG. Semantic search across docs, policies, and past decisions.

"I need to understand this infrastructure before making changes"
  → Level 5: MCP resources. Agent browses available data and pulls what it needs.
```

In practice, most infrastructure agents combine Levels 1 + 2: pre-inject the known context, give the agent query tools for the rest. Add Level 3 when agents need organizational knowledge. Add Level 4 only when full-text search fails to find semantically related documents at scale.

---

## Tools and Alternatives

### Asset Inventory (Layer 1: What exists?)

These tools handle the collection and normalization of cloud resources:

| Tool | What It Does | Approach |
|------|-------------|----------|
| **Your own scanners** | Custom scanners using cloud provider SDKs | Maximum control over what's scanned and how it's stored |
| [CloudQuery](https://github.com/cloudquery/cloudquery) | Syncs cloud resources to PostgreSQL or any database. 100+ plugins. | Best general-purpose option if you don't want to build scanners |
| [Steampipe](https://github.com/turbot/steampipe) | SQL interface to cloud APIs. Query AWS/Azure/GCP with SQL. | Real-time queries, no persistence (query → API call) |
| [Cartography](https://github.com/lyft/cartography) | Builds a Neo4j graph of assets and relationships (by Lyft) | Graph-native, includes IAM relationship mapping |

**Cloud provider inventories** — AWS Config + Resource Explorer, Azure Resource Graph, GCP Cloud Asset Inventory, OCI Search — are useful as supplements for real-time queries, but they're per-provider with no cross-cloud view and no custom enrichment (IaC correlation, cost data, compliance linking).

### Security & Compliance (Layer 3: What's wrong?)

These tools produce findings that feed into the data plane — they don't build it:

| Tool | What It Does |
|------|-------------|
| [Prowler](https://github.com/prowler-cloud/prowler) | Cloud security assessment for AWS, Azure, GCP. CIS, SOC2, HIPAA benchmarks. |
| [Checkov](https://github.com/bridgecrewio/checkov) | Static analysis for IaC (Terraform, CloudFormation, Kubernetes). |
| [Trivy](https://github.com/aquasecurity/trivy) | Vulnerability scanner for containers, IaC, and cloud. |
| Cloud-native tools | AWS Security Hub, Azure Defender, GCP Security Command Center |

These tools produce compliance findings. Your data plane stores those findings, links them to the cloud resources they affect, and makes them available to agents through skill APIs.

### Storage (Where to put it)

| Option | When to Use |
|--------|------------|
| **PostgreSQL** (with JSON columns) | Start here. One database for resources, relationships (as edge table), findings, IaC state. Recursive CTEs handle 2-3 hop graph queries. |
| **Neo4j / Neptune** | Only if you need complex multi-hop traversals that CTEs can't handle efficiently. Adds infrastructure and sync complexity. |
| **pgvector** / vector database | Only when agents fail to find relevant documentation via keyword search. Most infrastructure queries are filtered by known dimensions — vector search is overkill for resource lookups. |

---

## Data Freshness

Stale data is worse than no data — an agent acting on a deleted resource wastes time and creates confusion.

| Data Type | Freshness Needed | Strategy |
|-----------|-----------------|----------|
| Cloud resources | Hours | Scheduled scan every 1-6 hours |
| Resource properties | Hours | Full scan or event-driven (CloudTrail, Activity Log) |
| Relationships | Hours | Extracted during resource scan |
| IaC state | Minutes | Parse on git push (webhook-triggered) |
| Compliance findings | Hours | Scheduled scan (nightly or on-demand) |
| Cost data | Daily | Daily sync from billing APIs |
| Documentation | Daily | Scheduled sync or webhook on change |

For most infrastructure agents, **hourly cloud scans + git webhook triggers for IaC** provide sufficient freshness. Event-driven updates (CloudTrail → update database) are worth the complexity only for incident response agents that need to act within minutes.

---

## Next Chapter

[Chapter 7: Change Control & GitOps →](./07-change-control.md)

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
