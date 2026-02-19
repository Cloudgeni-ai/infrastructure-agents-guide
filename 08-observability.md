# Chapter 8: Observability & Audit

> OpenTelemetry, action trails, correlation IDs, and debugging agent failures.

---

## Why Agent Observability Is Different

Traditional observability answers: "Is my service healthy?"
Agent observability must answer: **"Why did the agent do that?"**

When an agent creates a PR that breaks staging, you need to trace back through:
- What prompt triggered the agent?
- What context did it assemble?
- What tools did it call, with what arguments?
- What did those tools return?
- What policy decisions were made?
- What credentials were issued?

Without this trail, agent failures are black boxes.

---

## The Three Pillars + Action Trails

```
Traditional Observability          Agent Observability
─────────────────────              ─────────────────────
Logs                               Logs
Metrics                            Metrics
Traces                             Traces
                                   + Action Trails ← NEW
```

**Action trails** are structured event logs of everything the agent did, separate from the LLM conversation. They're queryable, auditable, and don't depend on parsing chat transcripts.

---

## OpenTelemetry Attribute Schema

Define a consistent attribute namespace for all agent spans:

```typescript
// Attribute constants for agent telemetry
const ATTR = {
  // Request correlation
  REQUEST_ID: 'infra_agent.request.id',
  CORRELATION_ID: 'infra_agent.correlation.id',

  // Agent context
  AGENT_SESSION_ID: 'infra_agent.session.id',
  AGENT_RUN_ID: 'infra_agent.run.id',
  AGENT_SLUG: 'infra_agent.slug',
  ORGANIZATION_ID: 'infra_agent.organization.id',

  // Task context
  TASK_TYPE: 'infra_agent.task.type',
  TASK_PRIORITY: 'infra_agent.task.priority',
  TRIGGER_SOURCE: 'infra_agent.trigger.source',

  // Execution metrics
  ITERATION_COUNT: 'infra_agent.iteration.count',
  MAX_ITERATIONS: 'infra_agent.iteration.max',
  TOKEN_USAGE_INPUT: 'infra_agent.tokens.input',
  TOKEN_USAGE_OUTPUT: 'infra_agent.tokens.output',
  TOOL_CALLS_COUNT: 'infra_agent.tool_calls.count',

  // Performance
  SESSION_RESTORE_MS: 'infra_agent.perf.session_restore_ms',
  REPO_CLONE_MS: 'infra_agent.perf.repo_clone_ms',
  PIPELINE_WAIT_MS: 'infra_agent.perf.pipeline_wait_ms',
  TOTAL_DURATION_MS: 'infra_agent.perf.total_duration_ms',

  // Queue metrics
  QUEUE_DEPTH: 'infra_agent.queue.depth',
  QUEUE_WAIT_MS: 'infra_agent.queue.wait_ms',

  // Credential tracking
  CREDENTIAL_ISSUED: 'infra_agent.credential.issued',
  CREDENTIAL_SCOPE: 'infra_agent.credential.scope',
  CREDENTIAL_EXPIRES_AT: 'infra_agent.credential.expires_at',
} as const;
```

### Instrumenting the Agent Lifecycle

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('infra-agent-worker');

async function processAgentTask(task: AgentTask) {
  return tracer.startActiveSpan('agent.process_task', async (span) => {
    span.setAttributes({
      [ATTR.AGENT_SESSION_ID]: task.sessionId,
      [ATTR.AGENT_SLUG]: task.agentSlug,
      [ATTR.TASK_TYPE]: task.type,
      [ATTR.TRIGGER_SOURCE]: task.trigger.source,
    });

    try {
      // Clone repository
      await tracer.startActiveSpan('agent.clone_repo', async (cloneSpan) => {
        const startMs = Date.now();
        await cloneRepository(task.repository);
        cloneSpan.setAttribute(ATTR.REPO_CLONE_MS, Date.now() - startMs);
        cloneSpan.end();
      });

      // Run agent
      const result = await tracer.startActiveSpan('agent.run', async (runSpan) => {
        const result = await executeAgent(task);
        runSpan.setAttributes({
          [ATTR.ITERATION_COUNT]: result.iterations,
          [ATTR.TOKEN_USAGE_INPUT]: result.inputTokens,
          [ATTR.TOKEN_USAGE_OUTPUT]: result.outputTokens,
          [ATTR.TOOL_CALLS_COUNT]: result.toolCalls,
        });
        runSpan.end();
        return result;
      });

      span.setStatus({ code: SpanStatusCode.OK });
      return result;
    } catch (error) {
      span.setStatus({ code: SpanStatusCode.ERROR, message: error.message });
      span.recordException(error);
      throw error;
    } finally {
      span.end();
    }
  });
}
```

---

## Action Trail Events

Structured events emitted to a dedicated stream (not mixed with LLM conversation):

```typescript
interface ActionTrailEvent {
  timestamp: string;
  sessionId: string;
  runId: string;
  correlationId: string;
  type:
    | 'tool_call'
    | 'tool_result'
    | 'credential_issued'
    | 'policy_evaluated'
    | 'pipeline_triggered'
    | 'pr_created'
    | 'human_input_requested'
    | 'human_input_received'
    | 'error'
    | 'budget_warning';
  data: Record<string, unknown>;
}

// Emit action trail events
async function emitActionTrail(event: ActionTrailEvent) {
  // 1. Redis stream (real-time consumption)
  await redis.xadd(`action-trail:${event.sessionId}`, '*', {
    type: event.type,
    data: JSON.stringify(event.data),
    correlationId: event.correlationId,
    timestamp: event.timestamp,
  });

  // 2. Structured log (long-term storage)
  logger.info({
    msg: `action_trail:${event.type}`,
    ...event,
  });

  // 3. OpenTelemetry span event (trace correlation)
  const span = trace.getActiveSpan();
  if (span) {
    span.addEvent(event.type, event.data);
  }
}
```

### Example Trail for a Remediation Task

```json
[
  { "type": "tool_call", "data": { "tool": "read-file", "args": { "path": "main.tf" } } },
  { "type": "tool_result", "data": { "tool": "read-file", "size": 2340 } },
  { "type": "tool_call", "data": { "tool": "write-file", "args": { "path": "main.tf" } } },
  { "type": "credential_issued", "data": { "scope": "management.azure.com", "ttl": 3600 } },
  { "type": "pipeline_triggered", "data": { "pipelineId": "p-123", "type": "PLAN" } },
  { "type": "tool_result", "data": { "pipeline": "success", "drift": 0 } },
  { "type": "pr_created", "data": { "url": "https://github.com/org/repo/pull/42", "branch": "agent/fix-123" } }
]
```

---

## Observability Stack Alternatives

### Option 1: OpenTelemetry + Grafana Stack (Self-Hosted)

```yaml
# docker-compose.yml — observability stack
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # gRPC
      - "4318:4318"   # HTTP

  tempo:
    image: grafana/tempo:latest
    # Trace storage

  loki:
    image: grafana/loki:latest
    # Log aggregation

  prometheus:
    image: prom/prometheus:latest
    # Metrics

  grafana:
    image: grafana/grafana:latest
    # Dashboards
    ports:
      - "3001:3000"
```

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc: { endpoint: "0.0.0.0:4317" }
      http: { endpoint: "0.0.0.0:4318" }

processors:
  batch:
    timeout: 5s

exporters:
  otlphttp/tempo:
    endpoint: http://tempo:4318
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  prometheus:
    endpoint: "0.0.0.0:8889"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/tempo]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
```

### Option 2: Datadog

```typescript
import tracer from 'dd-trace';
tracer.init({ service: 'infra-agent-worker' });

// Datadog auto-instruments most frameworks
// Custom spans for agent-specific operations
const span = tracer.startSpan('agent.run_task');
span.setTag('agent.slug', task.agentSlug);
// ...
span.finish();
```

### Option 3: Dash0 / Grafana Cloud / New Relic

All support OpenTelemetry natively — instrument once with OTel SDK, export to any backend.

---

## Dashboard Essentials

### Agent Operations Dashboard

| Panel | Query (PromQL) | Purpose |
|-------|---------------|---------|
| Active sessions | `count(infra_agent_sessions_active)` | Current load |
| Avg completion time | `histogram_quantile(0.95, infra_agent_task_duration_seconds_bucket)` | Performance |
| Success rate | `rate(infra_agent_tasks_total{status="success"}[1h]) / rate(infra_agent_tasks_total[1h])` | Reliability |
| Token usage | `sum(rate(infra_agent_tokens_total[1h])) by (agent_slug)` | Cost tracking |
| Queue depth | `infra_agent_queue_depth` | Backlog |
| PRs created | `increase(infra_agent_prs_created_total[24h])` | Output |
| Credential issuances | `rate(infra_agent_credentials_issued_total[1h])` | Security audit |

---

## Log Redaction

Agent logs often contain sensitive data (resource IDs, config values, error messages with secrets). Redact by default:

```typescript
const REDACTION_PATTERNS = [
  /(?:api[_-]?key|token|secret|password|credential)["\s:=]+["']?([a-zA-Z0-9+/=_-]{20,})["']?/gi,
  /AKIA[0-9A-Z]{16}/g,                          // AWS Access Key
  /(?:ghp|gho|ghu|ghs|ghr)_[A-Za-z0-9_]{36}/g, // GitHub tokens
  /eyJ[A-Za-z0-9-_]+\.eyJ[A-Za-z0-9-_]+/g,     // JWT tokens
];

function redactSensitive(text: string): string {
  let redacted = text;
  for (const pattern of REDACTION_PATTERNS) {
    redacted = redacted.replace(pattern, '[REDACTED]');
  }
  return redacted;
}
```

---

## Key Takeaways

1. **Action trails** are the core primitive — structured events of what the agent did, not chat transcripts
2. **Correlation IDs** tie everything together: dispatch → execution → output → PR
3. **OpenTelemetry** as the instrumentation standard — export to any backend
4. **Redact by default** — agent logs will contain sensitive data
5. **Dashboard for operations** — active sessions, success rate, token usage, queue depth

---

## Next Chapter

[Chapter 9: Scheduled & Autonomous Operations →](./09-scheduled-autonomous.md)
