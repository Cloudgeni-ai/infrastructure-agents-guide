# Chapter 15: The Data Plane — Giving Agents Context

> Why `aws ec2 describe-instances` in a loop is the cloud equivalent of `grep` on 10,000 files — and how to build the infrastructure knowledge layer agents actually need.

---

## The Problem: Agents Without Context Are Expensive and Slow

Ask an infrastructure agent to "fix the unencrypted S3 bucket in production." Without a data plane, here's what happens:

```
Agent: calls aws s3api list-buckets                     # 200+ buckets, 15s
Agent: for each bucket, calls aws s3api get-bucket-encryption   # 200 API calls, 90s
Agent: for each unencrypted, calls aws s3api get-bucket-tagging # more API calls
Agent: cross-references with terraform state            # terraform state pull, 30s
Agent: reads the tf files to find the resource address  # grep across repo
Agent: finally has enough context to write a fix        # 3+ minutes burned
```

This is the cloud equivalent of running `grep` on a filesystem with 10,000 files. It works at human scale (one bucket). It breaks at infrastructure scale (hundreds of resources, multiple accounts, four clouds).

The costs compound:
- **API rate limits** — cloud providers throttle after a few hundred calls
- **Token waste** — LLM processes pages of JSON to extract three fields
- **Latency** — minutes of API calls before the agent starts actual work
- **Stale data** — by the time the agent finishes scanning, things may have changed
- **No relationships** — which VPC does this subnet belong to? Which security group is attached? The agent has to discover this from scratch every time.

---

## The Fix: A Pre-Built Infrastructure Knowledge Layer

Instead of making agents query cloud APIs on every task, build a data plane that:

1. **Scans** cloud accounts on a schedule and stores resources in a database
2. **Extracts relationships** between resources (VPC contains subnet, NSG secures NIC)
3. **Correlates** cloud resources with IaC state (this S3 bucket is `aws_s3_bucket.logs` in `modules/storage/main.tf`)
4. **Links** compliance findings to the resources they affect
5. **Exposes** all of this to agents through typed skill APIs — not raw CLI output

```
┌─────────────────────────────────────────────────────────┐
│              CLOUD PROVIDERS                             │
│  AWS    Azure    GCP    OCI                             │
└────────────┬────────────────────────────────────────────┘
             │  (scheduled scans)
             ▼
┌─────────────────────────────────────────────────────────┐
│              RESOURCE SCANNERS                           │
│  Per-type scanners: EC2, S3, VPC, VNet, GKE, ...       │
│  Extract: properties, tags, status, relationships       │
└────────────┬────────────────────────────────────────────┘
             │  (upsert)
             ▼
┌─────────────────────────────────────────────────────────┐
│              INFRASTRUCTURE DATABASE                     │
│                                                         │
│  cloud_resources          cloud_resource_relationships  │
│  ┌─────────────┐         ┌──────────────────────┐      │
│  │ resource_id  │────────│ source → target       │      │
│  │ type         │         │ relationship_type     │      │
│  │ properties   │         │ (CONTAINS, SECURED_BY │      │
│  │ tags         │         │  ROUTES_VIA, USES...) │      │
│  │ cost_data    │         └──────────────────────┘      │
│  │ state        │                                       │
│  └─────────────┘         iac_resources                  │
│                          ┌──────────────────────┐      │
│  compliance_findings     │ iac_address           │      │
│  ┌─────────────┐        │ repository            │      │
│  │ severity     │        │ cloud_resource_id ────│──┐   │
│  │ resource_id  │        └──────────────────────┘  │   │
│  │ remediation  │              ▲                    │   │
│  └─────────────┘              └────────────────────┘   │
│                                                         │
└────────────┬────────────────────────────────────────────┘
             │  (skill APIs)
             ▼
┌─────────────────────────────────────────────────────────┐
│              AGENT                                       │
│  getCloudResources({ type: 'S3_BUCKET', state: 'DISCOVERED' })
│  getComplianceFindings({ severity: 'HIGH', status: 'OPEN' })
│  getRelationships({ resourceId: 'arn:aws:...' })       │
│  → structured JSON, not raw CLI output                  │
└─────────────────────────────────────────────────────────┘
```

The agent never runs `aws s3api list-buckets`. It calls `getCloudResources()` and gets back a typed response in milliseconds.

---

## Resource Model

Every cloud resource, regardless of provider, normalizes to a common structure:

```typescript
interface CloudResource {
  id: string;
  organizationId: string;

  // Identity
  providerResourceId: string;    // AWS ARN, Azure Resource ID, GCP self-link
  providerResourceType: string;  // 'AWS_S3_BUCKET', 'Microsoft.Storage/storageAccounts'
  providerResourceName: string;  // Human-readable name
  provider: 'AWS' | 'AZURE' | 'GCP' | 'OCI';
  region: string;

  // Properties (the full resource configuration as JSON)
  providerResourceProperties: Record<string, unknown>;
  providerResourceTags: Record<string, string>;
  providerResourceStatus: string;

  // Enrichment
  costData?: {
    monthToDate: number;
    lastMonth: number;
    currency: string;
  };
  observabilityData?: {
    alerts: Alert[];
    metrics: MetricSummary[];
  };

  // IaC correlation
  state: 'MANAGED' | 'DISCOVERED';  // MANAGED = linked to IaC, DISCOVERED = cloud-only
  iacResource?: {
    iacAddress: string;       // e.g., 'aws_s3_bucket.logs'
    iacTool: 'TERRAFORM' | 'BICEP';
    repositoryId: string;
    costEstimate?: CostEstimate;
  };

  // Lifecycle
  origin: 'DISCOVERED' | 'PLANNED';
  lastScannedAt: Date;
}
```

The `providerResourceProperties` field is the key: it stores the full resource configuration as JSON, exactly as returned by the cloud provider's API. This is expensive to scan but cheap to query — the agent reads it once from the database instead of making live API calls.

### Resource Type Registries

Each cloud provider has a typed registry of scannable resource types:

```typescript
interface ResourceDefinition {
  resourceType: string;          // 'AWS_S3_BUCKET'
  displayName: string;           // 'S3 Bucket'
  description: string;
  productCategory: string;       // 'Storage', 'Compute', 'Networking'
  enabled: boolean;              // Can be disabled per-org
  priority: number;              // Scan order
  complexity: 'LOW' | 'MEDIUM' | 'HIGH';
  dependencies: string[];        // Other types that must be scanned first
}

// AWS: 30+ resource types (S3, VPC, EC2, Lambda, RDS, IAM, ECS, EKS, ...)
// Azure: 60+ resource types (VNet, VM, Storage, AKS, App Service, ...)
// GCP: GKE, Cloud Storage, Compute, Cloud SQL, VPC, ...
// OCI: Compute, VCN, Object Storage, Autonomous DB, ...
```

Scanning 60 resource types across 3 cloud accounts takes minutes, not hours. The results are stored once and queried many times — the agent doesn't pay the scan cost per task.

---

## Infrastructure Relationship Graph

Flat resource lists miss the most valuable context: **how resources relate to each other**. An agent fixing a security group rule needs to know which VMs are behind it. An agent remediating a storage encryption finding needs to know if a Key Vault key already exists.

### Relationship Types

Define a vocabulary of directed edges between resources:

```typescript
type RelationshipType =
  | 'CONTAINS'       // VPC contains Subnet, Resource Group contains VM
  | 'BELONGS_TO'     // Subnet belongs to VPC
  | 'SECURED_BY'     // NIC secured by NSG, Subnet secured by NSG
  | 'ROUTES_VIA'     // Subnet routes via Route Table
  | 'CONNECTED_TO'   // NIC connected to Subnet
  | 'PEERED_WITH'    // VPC peered with VPC
  | 'DELEGATED_TO'   // Subnet delegated to PostgreSQL
  | 'PROTECTED_BY'   // VNet protected by DDoS Plan
  | 'LINKED_TO'      // Private Endpoint linked to Storage
  | 'USES'           // Container App uses Container Registry
  | 'ATTACHED_TO'    // NIC attached to VM, Disk attached to VM
  | 'HOSTS'          // App Service Plan hosts Web App
  | 'ENCRYPTED_BY'   // Storage encrypted by Key Vault key
  | 'SNAPSHOT_OF'    // Snapshot of Disk
  | 'CREATED_FROM';  // VM created from Image

interface CloudResourceRelationship {
  id: string;
  sourceResourceId: string;
  targetResourceId: string;
  relationshipType: RelationshipType;
  isInferred: boolean;           // Extracted vs inferred
  metadata?: Record<string, unknown>;  // Peering state, delegation type, etc.
  origin: 'DISCOVERED' | 'PLANNED';
}
```

### Relationship Extraction

Relationships are extracted during the cloud scan, not discovered by the agent at runtime. Each resource scanner knows which relationships to extract:

```typescript
// Azure VNet scanner extracts:
// - VNet → Subnet (CONTAINS)
// - VNet → VNet (PEERED_WITH, from peering config)
// - VNet → DDoS Plan (PROTECTED_BY)
// - Subnet → NSG (SECURED_BY)
// - Subnet → Route Table (ROUTES_VIA)
// - Subnet → NAT Gateway (USES)
// - Subnet → Service (DELEGATED_TO)

// AWS VPC scanner extracts:
// - VPC → Subnet (CONTAINS)
// - VPC → Internet Gateway (USES)
// - Subnet → Route Table (ROUTES_VIA)
// - Security Group → VPC (BELONGS_TO)
// - Instance → Subnet (CONNECTED_TO)
// - Instance → Security Group (SECURED_BY)
```

The result is a graph stored as an edge table in PostgreSQL — no graph database required. A simple adjacency list with `source_id`, `target_id`, and `relationship_type` handles thousands of resources efficiently.

### Dangling References

Sometimes a relationship target hasn't been scanned yet (different region, different account, disabled resource type). Store the raw cloud ID:

```typescript
interface CloudResourceRelationship {
  // ... standard fields ...

  // When target isn't in the database yet:
  targetProviderResourceId?: string;   // Raw ARN or Azure Resource ID
  targetProviderResourceType?: string;
}

// When the target is eventually scanned, link it:
async function linkDanglingReferences(newResource: CloudResource) {
  await db.cloudResourceRelationship.updateMany({
    where: {
      targetProviderResourceId: newResource.providerResourceId,
      targetResourceId: null,
    },
    data: {
      targetResourceId: newResource.id,
    },
  });
}
```

### Planned Resource Overlay

Agents can register *planned* resources and relationships from IaC plan output. This creates a "what-if" layer on top of the discovered graph:

```typescript
// Agent parses terraform plan and registers planned changes
await registerPlannedResource({
  providerResourceId: 'aws_s3_bucket.new_logs',
  providerResourceType: 'AWS_S3_BUCKET',
  planningAction: 'CREATE',
  planningSessionId: session.id,
  providerResourceProperties: { /* from plan output */ },
});

await registerPlannedRelationship({
  sourceProviderResourceId: 'aws_s3_bucket.new_logs',
  targetProviderResourceId: 'arn:aws:kms:us-east-1:123:key/abc',
  relationshipType: 'ENCRYPTED_BY',
  origin: 'PLANNED',
});
```

This lets you visualize "current state + planned changes" before a PR is even created.

---

## Correlating Cloud, IaC, and Compliance

The data plane's real power is correlation — connecting cloud resources to the code that manages them and the findings that affect them.

### Cloud ↔ IaC Correlation

Every resource has a `state` field:

```
MANAGED    = Cloud resource matched to an IaC resource (Terraform address known)
DISCOVERED = Cloud resource with no matching IaC (ClickOps, manual creation, drift)
```

The correlation happens by matching `providerResourceId`:
- Parse Terraform state → extract ARNs/Azure Resource IDs
- Match against `cloud_resources.providerResourceId`
- Link the `iac_resource` to the `cloud_resource`

This gives agents critical context:
```
"S3 bucket 'prod-logs' is managed by Terraform at
 aws_s3_bucket.prod_logs in repos/infrastructure/modules/storage/main.tf"
```

vs.

```
"S3 bucket 'temp-debug-data' has no IaC — it was created manually
 and has no associated Terraform resource"
```

The agent handles these two cases completely differently: one gets a PR with a code fix, the other gets an import or a deletion recommendation.

### Cloud ↔ Compliance Correlation

Compliance findings reference cloud resources by their provider ID:

```typescript
interface ComplianceFinding {
  severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
  status: 'OPEN' | 'RESOLVED' | 'ACKNOWLEDGED' | 'FALSE_POSITIVE';
  resourceId: string;       // === CloudResource.providerResourceId
  resourceType: string;
  findingTitle: string;
  remediationRecommendation: string;
  framework: string;        // 'CIS AWS 3.0', 'SOC2', 'ISO 27001'
  controlId: string;        // '2.1.1'
}
```

When the agent queries `getComplianceFindings({ severity: 'HIGH' })`, each finding comes with the resource identity. The agent then calls `getCloudResources({ providerResourceId })` to get the full resource config, checks `iacResource` to find the Terraform address, and has everything it needs to write a fix — without a single cloud API call.

### IaC ↔ Static Analysis Correlation

Static analysis tools (Checkov, tfsec, Trivy) produce findings at the code level:

```typescript
interface StaticAnalysisFinding {
  checkId: string;              // 'CKV_AWS_126'
  checkName: string;            // 'Ensure S3 bucket has server-side encryption'
  resource: string;             // 'aws_s3_bucket.prod_logs'
  filePath: string;             // 'modules/storage/main.tf'
  fileLineRange: [number, number];
  codeBlock: [number, string][];  // Line-by-line code context
  fixedDefinition?: string;       // Checkov's suggested fix
  severity: string;
  benchmarks: Record<string, string[]>;  // { 'CIS AWS': ['2.1.1'] }
}
```

The agent gets the exact file, line number, and even a suggested fix — no searching required.

---

## Exposing Data to Agents

Raw database access is too dangerous and too unstructured. Expose the data plane through typed skill APIs that return exactly what agents need.

### Skill: Cloud Resources

```typescript
// Agent calls:
const resources = await getCloudResources({
  provider: 'AWS',
  resourceType: 'AWS_S3_BUCKET',
  state: 'DISCOVERED',        // Only unmanaged resources
  search: 'prod',             // Name/ID search
});

// Returns typed response:
interface CloudResourceResponse {
  resources: {
    providerResourceId: string;
    providerResourceType: string;
    providerResourceName: string;
    state: 'MANAGED' | 'DISCOVERED';
    providerResourceTags: Record<string, string>;
    iacResource?: {
      repositoryId: string;
      iacAddress: string;       // 'aws_s3_bucket.prod_logs'
    };
    costData?: { monthToDate: number; lastMonth: number };
  }[];
  total: number;
}
```

### Skill: Compliance Findings

```typescript
const findings = await getComplianceFindings({
  severity: 'HIGH',
  status: 'OPEN',
  resourceType: 'AWS_S3_BUCKET',
});

// Each finding includes the resource correlation
// Agent doesn't need to discover which bucket is affected
```

### Skill: Resource Relationships

```typescript
const graph = await getResourceGraph({
  resourceId: 'arn:aws:ec2:us-east-1:123:vpc/vpc-abc',
  depth: 2,  // Two hops from the VPC
});

// Returns: { nodes: CloudResource[], edges: CloudResourceRelationship[] }
// Agent sees: VPC → Subnets → Instances, VPC → IGW, VPC → Security Groups
```

The agent receives structured JSON, not raw CLI output. No parsing, no pagination, no rate limiting.

---

## Beyond Cloud: Organizational Knowledge

Infrastructure agents don't just need cloud data. They need organizational context — documentation, architecture decisions, naming conventions, team ownership.

### The Problem

An agent fixing a compliance finding in `prod-payments-vpc` should know:
- This VPC belongs to the Payments team
- The Payments team uses a specific naming convention
- There's an architecture decision record (ADR) about why this VPC is isolated
- The last incident in this VPC was 3 months ago

Without this context, the agent produces technically correct but organizationally wrong fixes.

### Sources of Organizational Knowledge

| Source | What It Contains | How to Ingest |
|--------|-----------------|---------------|
| **Git repositories** | IaC code, READMEs, ADRs, `.terraform-docs.yml` | Clone and index on scan |
| **Cloud resource tags** | Team ownership, environment, cost center | Extracted during cloud scan |
| **Wiki/docs** (Confluence, Notion, GitBook) | Architecture docs, runbooks, conventions | API sync on schedule |
| **Terraform module registries** | Internal module docs, input/output specs | Registry API |
| **Incident management** (PagerDuty, OpsGenie) | Past incidents per service/resource | Webhook or API sync |
| **CMDB** (ServiceNow, Backstage) | Service catalog, ownership, dependencies | API sync |
| **Chat history** | Past agent sessions, decisions, reasoning | Already in session store |
| **Custom policies** | Org-specific rules in plain language | Admin-authored, stored in DB |

### Ingestion Patterns

**Tier 1: Structured data (scan and store)**
Cloud resources, IaC state, compliance findings, cost data. These have stable schemas and change predictably. Store in PostgreSQL with typed columns.

**Tier 2: Semi-structured data (parse and index)**
Git READMEs, ADRs, Terraform module docs, wiki pages. These are text with some structure. Parse to markdown, extract metadata (title, tags, last updated), store as searchable text.

**Tier 3: Unstructured data (enrich with AI)**
Architecture diagrams (images), meeting recordings (audio), whiteboard photos. Use vision models for diagrams, speech-to-text for audio, OCR for whiteboards. Convert to text and index.

```
Tier 1: Structured     → PostgreSQL (typed columns, JSON properties)
Tier 2: Semi-structured → Text index (BM25/Lucene, metadata extraction)
Tier 3: Unstructured    → AI enrichment → Text index
```

This mirrors the 3-tier enrichment model: basic metadata extraction → local processing (OCR, speech-to-text) → AI vision analysis for complex artifacts.

### Making Knowledge Accessible to Agents

Two approaches, not mutually exclusive:

**Approach A: Inject at dispatch time**
When a task is dispatched, the system pre-fetches relevant context and injects it into the agent prompt:

```typescript
async function buildAgentContext(task: AgentTask): Promise<string> {
  const sections: string[] = [];

  // 1. Cloud resource context
  if (task.finding) {
    const resource = await getCloudResource(task.finding.resourceId);
    sections.push(formatResourceContext(resource));

    // 2. Relationship context (what's connected to this resource)
    const neighbors = await getRelationships(resource.id, { depth: 1 });
    sections.push(formatRelationshipContext(neighbors));
  }

  // 3. IaC context (where in code)
  if (resource?.iacResource) {
    const repo = await getRepository(resource.iacResource.repositoryId);
    sections.push(`Terraform address: ${resource.iacResource.iacAddress}`);
    sections.push(`Repository: ${repo.provider}/${repo.owner}/${repo.name}`);
  }

  // 4. Policy context (org-specific rules)
  const policies = await getOrgPolicies(task.organizationId);
  sections.push(formatPolicyDigest(policies));

  // 5. Past remediation context (similar findings)
  const similar = await getSimilarFindings(task.finding, { limit: 3, status: 'RESOLVED' });
  if (similar.length) {
    sections.push(formatPastRemediations(similar));
  }

  return sections.join('\n\n---\n\n');
}
```

**Approach B: Queryable skill APIs**
Give agents skills to search and retrieve context on demand:

```typescript
// Agent decides what context it needs and queries for it
const wiki = await searchKnowledgeBase({
  query: 'VPC isolation policy payments team',
  sources: ['wiki', 'adr', 'policy'],
  limit: 5,
});

const history = await getResourceHistory({
  resourceId: 'arn:aws:ec2:us-east-1:123:vpc/vpc-abc',
  events: ['incident', 'change', 'drift'],
  since: '90d',
});
```

**Approach A** is better for focused tasks (remediate this finding — here's everything you need). **Approach B** is better for exploratory tasks (explain this infrastructure — let the agent decide what to look up).

In practice, use both: inject the obvious context at dispatch time, and give the agent skills to query for more when it needs it.

---

## Search and Retrieval Patterns

### Option 1: PostgreSQL Full-Text Search (Start Here)

For most infrastructure agent use cases, PostgreSQL's built-in text search is sufficient:

```sql
-- Find resources by name, type, or tag
SELECT * FROM cloud_resources
WHERE organization_id = $1
  AND (
    provider_resource_name ILIKE '%prod%'
    OR provider_resource_id ILIKE '%prod%'
    OR provider_resource_tags::text ILIKE '%prod%'
  )
  AND provider_resource_type = 'AWS_S3_BUCKET';
```

This works because infrastructure queries are usually filtered by known dimensions (provider, type, region, tags) — not free-text semantic search.

### Option 2: BM25 / Lucene (For Documentation)

When you add wiki pages, ADRs, and runbooks to the knowledge layer, keyword-based search becomes more useful:

| Engine | Deployment | Best For |
|--------|-----------|----------|
| [MeiliSearch](https://github.com/meilisearch/meilisearch) | Self-hosted, simple | Typo-tolerant search, fast setup |
| [Typesense](https://github.com/typesense/typesense) | Self-hosted or cloud | Faceted search, geo-search |
| PostgreSQL `tsvector` | Already running | Good enough for <100k documents |
| Elasticsearch / OpenSearch | Self-hosted or AWS | Enterprise scale, complex queries |
| Apache Lucene (direct) | Library | Maximum control, embedded use |

### Option 3: Vector Search / RAG (For Semantic Queries)

When agents need to answer questions like "what's our policy on public-facing S3 buckets?" — keyword search may miss relevant documents that use different terminology. Vector search finds semantically similar content:

| Engine | Deployment | Best For |
|--------|-----------|----------|
| [pgvector](https://github.com/pgvector/pgvector) | PostgreSQL extension | Keep it in one database |
| [Qdrant](https://github.com/qdrant/qdrant) | Self-hosted or cloud | Production vector search |
| [Chroma](https://github.com/chroma-core/chroma) | Embedded or self-hosted | Quick prototyping |
| [Weaviate](https://github.com/weaviate/weaviate) | Self-hosted or cloud | Hybrid search (BM25 + vector) |

Most infrastructure agent workloads don't need vector search yet — the queries are specific enough for filtered SQL or keyword search. Add it when you have a large documentation corpus and agents are failing to find relevant context.

---

## Serialization: Making Cloud Data LLM-Readable

Raw cloud API responses are verbose JSON with nested structures, pagination tokens, and provider-specific quirks. The data plane should serialize resources into a format LLMs can process efficiently.

### Resource Serialization

```typescript
function serializeResourceForAgent(resource: CloudResource): string {
  const lines: string[] = [
    `#### ${resource.providerResourceName || resource.providerResourceId}`,
    `- **Type:** ${resource.providerResourceType}`,
    `- **Provider:** ${resource.provider}`,
    `- **Region:** ${resource.region}`,
    `- **Resource ID:** ${resource.providerResourceId}`,
    `- **State:** ${resource.state}`,
  ];

  if (Object.keys(resource.providerResourceTags).length > 0) {
    lines.push(`- **Tags:** ${JSON.stringify(resource.providerResourceTags)}`);
  }

  if (resource.iacResource) {
    lines.push(`- **Terraform Address:** ${resource.iacResource.iacAddress}`);
    lines.push(`- **Repository:** ${resource.iacResource.repositoryId}`);
  } else {
    lines.push(`- **IaC:** Not managed (no Terraform/Bicep resource found)`);
  }

  if (resource.costData) {
    lines.push(`- **Cost (MTD):** $${resource.costData.monthToDate.toFixed(2)}`);
  }

  return lines.join('\n');
}
```

Markdown is a better serialization format than JSON for LLM consumption:
- Fewer tokens per fact (no braces, brackets, quotes)
- Natural language headers help the model navigate
- Selective — only include fields the agent needs, not the full cloud API response

### Drift Serialization

```typescript
function serializeDriftForAgent(drift: DriftFinding): string {
  return [
    `### Drift: ${drift.driftTitle}`,
    `**Severity:** ${drift.severity}`,
    `**Summary:** ${drift.driftSummary}`,
    `**Impact:** ${drift.driftImpactAnalysis}`,
    '',
    '**Before (expected):**',
    '```json',
    JSON.stringify(drift.driftDetails.before, null, 2).slice(0, 2000),
    '```',
    '',
    '**After (actual):**',
    '```json',
    JSON.stringify(drift.driftDetails.after, null, 2).slice(0, 2000),
    '```',
  ].join('\n');
}
```

Truncate large JSON blobs. The agent needs enough to understand the drift, not the full resource config. 2000 characters per block is usually sufficient.

---

## Alternatives: How to Build the Data Plane

### Option A: Build on Your Database (PostgreSQL)

Store everything in PostgreSQL with JSON columns for flexible properties:

```
cloud_resources              (typed columns + properties JSON)
cloud_resource_relationships (edge table: source_id, target_id, type)
iac_resources                (terraform address, linked to cloud_resource)
compliance_findings          (severity, resource_id, remediation)
```

**Pros**: One database, ACID transactions, joins across tables, no additional infrastructure.
**Cons**: Graph queries (traverse 3 hops) get complex with recursive CTEs.

### Option B: Graph Database (Neo4j, Amazon Neptune)

Store resources as nodes and relationships as edges in a native graph database:

```cypher
MATCH (vpc:Resource {type: 'AWS_VPC'})-[:CONTAINS]->(subnet)-[:SECURED_BY]->(nsg)
WHERE vpc.name = 'prod-payments-vpc'
RETURN subnet, nsg
```

**Pros**: Natural fit for relationship traversal, powerful query language.
**Cons**: Additional infrastructure, data sync complexity, harder to correlate with relational data (findings, IaC state).

### Option C: Cloud Provider Asset Inventories

Use the cloud providers' own inventory services:

| Provider | Service | What It Does |
|----------|---------|-------------|
| AWS | [Config](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html) + [Resource Explorer](https://docs.aws.amazon.com/resource-explorer/latest/userguide/what-is-resource-explorer.html) | Records resource configurations, supports SQL-like queries |
| Azure | [Resource Graph](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview) | KQL queries across all subscriptions, relationship tracking |
| GCP | [Cloud Asset Inventory](https://cloud.google.com/asset-inventory/docs/reference/rest) | Resource and IAM policy history, export to BigQuery |
| OCI | [Search](https://docs.oracle.com/en-us/iaas/Content/Search/Concepts/queryoverview.htm) | Free-text and structured queries across compartments |

**Pros**: No scanning infrastructure to maintain, always up-to-date.
**Cons**: Per-provider (no cross-cloud view), limited relationship types, API rate limits, can't add custom enrichment (IaC correlation, cost data, compliance findings).

### Option D: Open-Source CSPM / Asset Inventory

| Tool | What It Does | Stars |
|------|-------------|-------|
| [CloudQuery](https://github.com/cloudquery/cloudquery) | Syncs cloud resources to PostgreSQL/any database. 100+ provider plugins. | ~6k |
| [Steampipe](https://github.com/turbot/steampipe) | SQL interface to cloud APIs. Query AWS/Azure/GCP with SQL in real-time. | ~7k |
| [Cartography](https://github.com/lyft/cartography) | Builds a Neo4j graph of infrastructure assets and relationships (by Lyft). | ~3k |
| [Prowler](https://github.com/prowler-cloud/prowler) | Cloud security assessment — outputs resource inventory as a side effect. | ~12k |

**CloudQuery** is the closest to a general-purpose data plane: it syncs cloud resources into your database on a schedule, and you query them with SQL. It doesn't do relationship extraction or IaC correlation, but it handles the scanning and normalization.

**Cartography** is interesting for graph-native use cases: it builds a Neo4j graph including IAM relationships (who can access what), which is valuable for security agents.

### Recommendation

Start with **PostgreSQL + your own scanners** (Option A). Cloud provider inventories (Option C) are useful as a supplement for real-time queries, but you need your own database for cross-cloud views, IaC correlation, and compliance linking. Add a graph database (Option B) only if you need complex multi-hop traversals that recursive CTEs can't handle efficiently.

---

## Data Freshness

Stale data is worse than no data — an agent acting on a resource that was deleted 2 hours ago wastes time and creates confusion.

### Scan Frequency by Data Type

| Data Type | Freshness Required | Update Strategy |
|-----------|-------------------|-----------------|
| Resource inventory | Hours | Scheduled scan (every 1-6 hours) |
| Resource properties | Hours | Full scan or event-driven (CloudTrail, Activity Log) |
| Relationships | Hours | Extracted during resource scan |
| Compliance findings | Hours | Scheduled scan (Prowler/Checkov on schedule) |
| IaC state | Minutes | Parse on git push (webhook-triggered) |
| Cost data | Daily | Daily sync from billing APIs |
| Wiki/docs | Daily | Scheduled sync or webhook on change |

### Event-Driven Updates

Supplement scheduled scans with event-driven updates for fast-changing data:

```typescript
// CloudTrail event → update resource in database
async function handleCloudEvent(event: CloudTrailEvent) {
  if (event.eventName === 'DeleteBucket') {
    await db.cloudResource.update({
      where: { providerResourceId: event.resources[0].ARN },
      data: { providerResourceStatus: 'DELETED', deletedAt: new Date() },
    });
  }
}
```

For most infrastructure agents, **hourly scans + git webhook triggers** provide sufficient freshness. Real-time event processing is worth the complexity only for incident response agents that need to act within minutes.

---

## Implementation Checklist

```
DATA INGESTION
[ ] Cloud resource scanners for each provider (AWS, Azure, GCP, OCI)
[ ] Resource type registry with per-type scanner definitions
[ ] Scheduled scan pipeline (every 1-6 hours)
[ ] IaC state parsing (Terraform plan/state, Bicep what-if)
[ ] Compliance scan integration (Prowler, Checkov, or provider-native)

STORAGE
[ ] Cloud resources table with typed columns + JSON properties
[ ] Relationship edge table (source, target, type)
[ ] IaC resources table linked to cloud resources
[ ] Compliance findings table linked by provider resource ID
[ ] Dangling reference handling (link when target is eventually scanned)

CORRELATION
[ ] Cloud ↔ IaC matching by provider resource ID
[ ] Cloud ↔ Compliance finding matching by provider resource ID
[ ] IaC ↔ Static analysis matching by Terraform address
[ ] Planned resource overlay (from agent IaC plan output)

AGENT ACCESS
[ ] Skill API: getCloudResources (filtered, paginated)
[ ] Skill API: getComplianceFindings (filtered, paginated)
[ ] Skill API: getResourceGraph (nodes + edges, configurable depth)
[ ] Skill API: registerPlannedResource / registerPlannedRelationship
[ ] Serialization: markdown format for LLM-readable context
[ ] Context injection at dispatch time for focused tasks

FRESHNESS
[ ] Scheduled scan pipeline with configurable frequency
[ ] Git webhook triggers for IaC state updates
[ ] Event-driven updates for fast-changing data (optional)
[ ] Last-scanned timestamp on every resource
```

---

## Next Chapter

[Chapter 14: Risk Framework & Checklists →](./14-risk-framework.md)

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
