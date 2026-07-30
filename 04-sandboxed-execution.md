# Chapter 4: Sandboxed Execution

> How to isolate agent workloads so a compromised agent can't destroy your infrastructure.

---

## Why You Need Sandboxing

Infrastructure agents execute code, run CLI tools, and interact with cloud APIs. If the LLM is manipulated (via prompt injection, hallucination, or a malicious skill), the blast radius equals the agent's access level.

Without sandboxing:
- Agent can access the host filesystem (your secrets, other tenants' data)
- Agent can make network calls to arbitrary endpoints
- Agent can escalate privileges via the host OS
- A crash can take down the whole system

Sandboxing is your fallback when the policy layer fails. Even if prompt rules are bypassed, the sandbox limits damage.

---

## Sandboxing Spectrum

```
Less Isolation                                         More Isolation
─────────────────────────────────────────────────────────────────
  Process      Container    Container +      MicroVM      Full VM
  (Node.js     (Docker)     Network          (Firecracker) (EC2/Azure
   subprocess)              Policies                       VM)

  Fast          Fast        Medium           Fast          Slow
  No isolation  Good        Very good        Excellent     Maximum
  Free          Free        Free             Complex       Expensive
```

For most infrastructure agents, **container + network policies** is the right tradeoff.

---

## Option 1: Docker Containers (Self-Managed)

The most common approach. Run each agent task in an ephemeral container.

### Architecture

```mermaid
graph TB
    subgraph "Host Machine"
        WORKER[Worker Process] -->|docker run| C1[Agent Container 1<br/>ephemeral]
        WORKER -->|docker run| C2[Agent Container 2<br/>ephemeral]

        C1 -->|exits| CLEANUP1[Container removed]
        C2 -->|exits| CLEANUP2[Container removed]
    end

    C1 -.->|limited network| CLOUD[Cloud APIs]
    C2 -.->|limited network| GIT[Git Provider]
```

### Docker Compose Example

```yaml
# docker-compose.yml — Agent worker with sandbox
services:
  task-worker:
    build:
      context: .
      dockerfile: Dockerfile.worker
    environment:
      - REDIS_URL=redis://redis:6379
      - QUEUE_NAME=agent-tasks
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock  # For spawning sandbox containers
    networks:
      - internal

  # Each agent task spawns one of these
  agent-sandbox:
    build:
      context: .
      dockerfile: Dockerfile.sandbox
    profiles: ["sandbox"]  # Not started by default
    read_only: true         # Read-only filesystem
    tmpfs:
      - /tmp:size=1G        # Writable temp space
      - /workspace:size=5G  # Git workspace
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_RAW             # For DNS resolution
    mem_limit: 4g
    cpus: 2
    pids_limit: 256
    networks:
      - sandbox-net

networks:
  internal:
  sandbox-net:
    driver: bridge
    internal: false  # Allow outbound for git/cloud APIs
```

### Dockerfile for Agent Sandbox

```dockerfile
FROM node:22-slim

# Install IaC tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    git curl unzip jq ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Terraform
RUN curl -fsSL https://releases.hashicorp.com/terraform/1.9.0/terraform_1.9.0_linux_amd64.zip \
    -o /tmp/terraform.zip && unzip /tmp/terraform.zip -d /usr/local/bin/ && rm /tmp/terraform.zip

# Azure CLI (if needed)
RUN curl -sL https://aka.ms/InstallAzureCLIDeb | bash

# Non-root user
RUN useradd -m -s /bin/bash agent
USER agent
WORKDIR /workspace

# No long-lived credentials baked in
# Credentials are injected at runtime via the credential broker
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Full control over environment | You manage the infrastructure |
| No cold start (images pre-pulled) | Docker socket access is a security concern |
| Works anywhere Docker runs | Container orchestration complexity |
| Easy to debug locally | Need to manage image updates |

---

## Option 2: Modal (Serverless Containers)

[Modal](https://modal.com) provides serverless container execution with sub-second cold starts. Excellent for bursty agent workloads.

### Architecture

```mermaid
graph LR
    API[API Server] -->|dispatch| MODAL[Modal Function]
    MODAL -->|runs in| SANDBOX[Ephemeral Container<br/>isolated network<br/>auto-scaled]
    SANDBOX -->|results| API
```

### Implementation

```python
# modal_worker.py
import modal

app = modal.App("infra-agent-worker")

# Define the container image
agent_image = (
    modal.Image.debian_slim(python_version="3.12")
    .apt_install("git", "curl", "unzip", "jq")
    .run_commands(
        "curl -fsSL https://releases.hashicorp.com/terraform/1.9.0/"
        "terraform_1.9.0_linux_amd64.zip -o /tmp/tf.zip",
        "unzip /tmp/tf.zip -d /usr/local/bin/",
    )
    .pip_install("anthropic", "redis", "httpx")
)

@app.function(
    image=agent_image,
    timeout=30 * 60,        # 30 min max
    memory=4096,             # 4GB RAM
    cpu=2.0,
    secrets=[modal.Secret.from_name("agent-redis-credentials")],
    # Network: Modal handles isolation automatically
)
async def run_agent_task(task_payload: dict):
    """Execute a single agent task in an isolated Modal container."""
    import redis
    import anthropic

    # Connect to Redis for event streaming
    r = redis.from_url(os.environ["REDIS_URL"])

    # Run the agent
    client = anthropic.Anthropic()
    # ... agent execution logic ...

    # Emit completion event
    r.xadd(f"task:run:{task_payload['runId']}", {
        "type": "completed",
        "data": json.dumps(result),
    })
```

```typescript
// Dispatch from Node.js API server
async function dispatchToModal(task: AgentTask) {
  // Modal provides a REST API to trigger functions
  const response = await fetch('https://your-app--run-agent-task.modal.run', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${MODAL_TOKEN}`,
    },
    body: JSON.stringify(task.payload),
  });
}
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Zero infrastructure management | Vendor dependency |
| Auto-scaling (0 to N) | Cold starts (~2s first call) |
| Pay-per-second billing | Requires Python SDK for config |
| Built-in secrets management | Egress costs |
| GPU support for ML workloads | Less control over networking |

---

## Option 3: Azure Container Apps Jobs

[Azure Container Apps Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs) run containers on-demand with managed identity. Ideal for Azure-native architectures.

### Architecture

```mermaid
graph LR
    API[API Server] -->|trigger job| ACA[Azure Container<br/>Apps Job]
    ACA -->|pulls image| ACR[Azure Container<br/>Registry]
    ACA -->|runs in| CONTAINER[Ephemeral Container<br/>managed identity<br/>isolated]
    CONTAINER -->|results via Redis| API
```

### Implementation

```typescript
// Dispatch — trigger an Azure Container App Job
import { ContainerAppsClient } from '@azure/arm-appcontainers';
import { DefaultAzureCredential } from '@azure/identity';

async function dispatchToContainerApp(task: AgentTask) {
  const client = new ContainerAppsClient(new DefaultAzureCredential(), SUBSCRIPTION_ID);

  await client.jobs.beginStart(
    RESOURCE_GROUP,
    JOB_NAME,
    {
      template: {
        containers: [{
          name: 'agent-worker',
          image: `${ACR_NAME}.azurecr.io/agent-worker:latest`,
          env: [
            { name: 'TASK_ID', value: task.id },
            { name: 'TASK_PAYLOAD', value: JSON.stringify(task.payload) },
            { name: 'REDIS_URL', secretRef: 'redis-url' },
          ],
          resources: { cpu: 2, memory: '4Gi' },
        }],
      },
    }
  );
}
```

```yaml
# Azure Container App Job definition (Bicep)
resource agentJob 'Microsoft.App/jobs@2024-03-01' = {
  name: 'agent-worker-job'
  location: location
  identity: {
    type: 'SystemAssigned'  # Managed Identity — no stored credentials
  }
  properties: {
    environmentId: containerAppEnv.id
    configuration: {
      triggerType: 'Manual'   # Triggered by API call
      replicaTimeout: 1800    # 30 min max
      replicaRetryLimit: 1
      secrets: [
        { name: 'redis-url', value: redisConnectionString }
      ]
    }
    template: {
      containers: [
        {
          name: 'agent-worker'
          image: '${acrName}.azurecr.io/agent-worker:latest'
          resources: { cpu: json('2.0'), memory: '4Gi' }
        }
      ]
    }
  }
}
```

### Pros & Cons

| Pros | Cons |
|------|------|
| Managed Identity (no stored credentials) | Azure-only |
| Auto-scaling to zero | ~10s cold start |
| Integrated with Azure networking | Limited customization |
| Built-in monitoring via Azure Monitor | Job orchestration is basic |
| VNET integration for private resources | Image pull time adds to startup |

---

## Option 4: AWS Lambda + ECS Tasks

For AWS-native architectures, use Lambda for short tasks and ECS Fargate for longer ones.

```typescript
// Short tasks (<15min): Lambda
import { Lambda } from '@aws-sdk/client-lambda';

async function dispatchToLambda(task: AgentTask) {
  const lambda = new Lambda();
  await lambda.invoke({
    FunctionName: 'agent-worker',
    InvocationType: 'Event', // Async
    Payload: JSON.stringify(task.payload),
  });
}

// Long tasks (>15min): ECS Fargate
import { ECS } from '@aws-sdk/client-ecs';

async function dispatchToECS(task: AgentTask) {
  const ecs = new ECS();
  await ecs.runTask({
    cluster: 'agent-cluster',
    taskDefinition: 'agent-worker',
    launchType: 'FARGATE',
    overrides: {
      containerOverrides: [{
        name: 'agent-worker',
        environment: [
          { name: 'TASK_ID', value: task.id },
          { name: 'TASK_PAYLOAD', value: JSON.stringify(task.payload) },
        ],
      }],
    },
    networkConfiguration: {
      awsvpcConfiguration: {
        subnets: [PRIVATE_SUBNET],
        securityGroups: [AGENT_SG],
      },
    },
  });
}
```

---

## Option 5: Kubernetes Jobs

Maximum control. Run agent tasks as Kubernetes Jobs with pod security policies.

```yaml
# agent-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: agent-task-${TASK_ID}
  namespace: agent-workers
spec:
  ttlSecondsAfterFinished: 300  # Clean up after 5 min
  backoffLimit: 1
  template:
    spec:
      serviceAccountName: agent-worker  # RBAC-scoped
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: agent
          image: registry.example.com/agent-worker:latest
          resources:
            requests: { cpu: "1", memory: "2Gi" }
            limits: { cpu: "2", memory: "4Gi" }
          env:
            - name: TASK_PAYLOAD
              value: "${TASK_PAYLOAD}"
            - name: REDIS_URL
              valueFrom:
                secretKeyRef:
                  name: agent-secrets
                  key: redis-url
      restartPolicy: Never
      # Network policy applied at namespace level
```

```yaml
# Network policy: restrict agent egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: agent-egress
  namespace: agent-workers
spec:
  podSelector: {}
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock: { cidr: 0.0.0.0/0 }  # Allow outbound
      ports:
        - port: 443    # HTTPS (git, cloud APIs)
        - port: 6379   # Redis
    - to:
        - namespaceSelector:
            matchLabels: { name: kube-system }
      ports:
        - port: 53     # DNS
          protocol: UDP
```

---

## Option 6: Managed Agent Sandbox APIs

There is now a distinct market for programmatically created agent workspaces. Unlike a job runner, these products commonly expose files, command execution, background processes, ports/previews, pause/resume, and snapshots as a session API.

| Provider / Service | Execution and Lifecycle | Best Fit | Watch For |
|--------------------|-------------------------|----------|-----------|
| [E2B](https://e2b.dev/docs) | MicroVM sandboxes; pause/resume, auto-resume, filesystem + memory snapshots, desktop options | Stateful coding/data agents and resumable workspaces | Reconnect streams after snapshot/pause; make lifecycle costs explicit |
| [Daytona](https://www.daytona.io/docs/sandboxes) | Containers by default; Linux/Windows VMs, GPU, snapshots, fork, pause/archive/recover | Long-running workspaces and OS/GPU variety | Choose container vs VM boundary deliberately |
| [Modal Sandboxes](https://modal.com/docs/guide/sandboxes) | gVisor-isolated containers; filesystem/directory/memory snapshots, volumes, egress allowlists | Bursty workloads, custom images, data/GPU work | Sandboxes have bounded lifetimes; snapshot/restore long work |
| [Vercel Sandbox](https://vercel.com/docs/sandbox) | Firecracker microVMs, snapshots, network firewall, credential brokering | High-concurrency short-lived code execution and previews | Platform/runtime coupling; verify required CLI and OS packages |
| [Cloudflare Sandbox](https://developers.cloudflare.com/sandbox/) | Isolated containers controlled from Workers/Durable Objects; files, processes, previews | TypeScript/edge-native agent products | Container rather than per-sandbox VM boundary |
| [Northflank Sandboxes](https://northflank.com/docs/v1/application/sandboxes/sandboxes-on-northflank) | MicroVM-backed containers, persistence, GPU, pause/destroy, hosted or BYOC | Multi-tenant platforms needing VPC/BYOC control | Public port exposure and persistence must be policy-gated |
| [Azure Container Apps Dynamic Sessions](https://learn.microsoft.com/en-us/azure/container-apps/sessions) | Prewarmed Hyper-V-isolated code-interpreter or custom-container pools | Azure-native interactive execution with fast allocation | Code-interpreter and custom-container observability differ |
| [AWS AgentCore Code Interpreter](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/code-interpreter-tool.html) | Managed code sandbox with IAM, network modes, and CloudTrail integration | AWS-native agent tools and governed code execution | Narrower than a general-purpose dev box; execution role scope is critical |
| [Cloud Run Sandboxes](https://docs.cloud.google.com/run/docs/code-execution) | Nested ephemeral sandboxes inside an existing Cloud Run service | GCP-native agents needing fast local code/subagent execution | Preview maturity; agent host and nested sandbox have distinct boundaries |

Runtime vendors are converging on a provider-neutral harness/compute split. [OpenAI Sandbox Agents](https://developers.openai.com/api/docs/guides/agents/sandboxes) (beta) ships clients for several providers, while [Claude Managed Agents](https://claude.com/blog/claude-managed-agents-updates) can use self-hosted or managed sandboxes. Keep your own sandbox contract small so switching providers does not rewrite agent logic.

### Define a Provider-Neutral Sandbox Contract

At minimum, normalize:

```typescript
interface SandboxProvider {
  create(spec: WorkspaceSpec): Promise<SandboxHandle>;
  resume(state: SerializedSandboxState): Promise<SandboxHandle>;
  exec(handle: SandboxHandle, command: Command): Promise<ProcessHandle>;
  upload(handle: SandboxHandle, files: FileInput[]): Promise<void>;
  download(handle: SandboxHandle, paths: string[]): Promise<ArtifactRef[]>;
  snapshot(handle: SandboxHandle): Promise<SnapshotRef>;
  inspect(handle: SandboxHandle): Promise<SandboxStatus>;
  terminate(handle: SandboxHandle): Promise<void>;
  listOwned(): Promise<SandboxHandle[]>; // cleanup/reconciliation
}
```

Declare optional capabilities — ports, PTY, desktop, GPU, fork, memory snapshot, domain allowlist — instead of pretending every provider behaves identically. Persist provider ID, provider sandbox ID, workspace revision, creation generation, lease owner, resource profile, and last verified status outside the worker process.

### Recovery Must Be Fenced and Verifiable

A provider reporting "running" does not prove that the right workspace, process, or credentials survived. Use a generation-fenced recovery flow:

1. acquire a durable recovery lease for the session and expected generation
2. inspect the provider object and verify its identity
3. restore or rematerialize the expected workspace revision
4. reissue short-lived credentials and rebuild mounts; never recover expired secrets from a snapshot
5. run readiness checks for required files, tools, network policy, and process routing
6. publish the new active generation atomically
7. terminate superseded provider objects and record cleanup success or failure

Archive or snapshot creation is not success until its digest, contents, and restore path are verified. Run an orphan sweeper against `listOwned()` and compare provider inventory with durable session state. Record cleanup failures as evidence; do not erase the provider ID and declare the leak gone.

This pattern is backported from [OpenGeni's sandbox/control-plane architecture](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/architecture.md), where worker restarts and provider recovery are treated as distributed ownership changes rather than simple container restarts.

---

## Worker Deployment Comparison

| Feature | Docker | Modal | Azure Container Apps | AWS Lambda/ECS | Kubernetes |
|---------|--------|-------|---------------------|----------------|------------|
| **Cold start** | ~0s (warm) | ~2s | ~10s | Lambda: ~1s, ECS: ~30s | ~5-30s |
| **Max runtime** | Unlimited | 24h | 24h | Lambda: 15m, ECS: unlimited | Unlimited |
| **Isolation** | Container | Container + network | Container + VNET | Lambda: microVM | Pod + network policy |
| **Scaling** | Manual | Automatic | Automatic | Automatic | HPA/KEDA |
| **Cost model** | Fixed | Per-second | Per-second | Per-request/second | Cluster + pod |
| **Managed identity** | DIY | Secrets mgmt | Native | IAM roles | Service accounts |
| **GPU support** | Yes | Yes | No | No (Lambda) | Yes |
| **Local dev** | Native | Modal CLI | Local emulator | SAM/LocalStack | Minikube/Kind |
| **Best for** | Dev/small scale | Bursty, serverless | Azure-native | AWS-native | Multi-cloud, full control |

---

## Network Egress Controls

Regardless of sandbox technology, restrict outbound network access:

```
ALLOW:
  ├── Git providers (github.com, gitlab.com, dev.azure.com)
  ├── Cloud APIs (*.amazonaws.com, *.azure.com, *.googleapis.com)
  ├── IaC registries (registry.terraform.io)
  ├── Your Redis/message bus
  └── DNS resolution

DENY:
  ├── Internal networks (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
  ├── Metadata endpoints (169.254.169.254)  ← CRITICAL
  ├── Other tenants' resources
  └── Everything else by default
```

> **Critical**: Always block cloud metadata endpoints (169.254.169.254). A compromised agent hitting the metadata endpoint can steal instance credentials and escalate privileges.

---

## Next Chapter

[Chapter 5: Credential Management →](./05-credential-management.md)
