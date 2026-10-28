# Chapter 9: Observability & Audit

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

OpenTelemetry now maintains dedicated [GenAI semantic conventions](https://github.com/open-telemetry/semantic-conventions-genai) for model calls, agent/workflow invocation, tool execution, retrieval, token usage, and MCP. Use those standard `gen_ai.*` attributes where they fit, and keep a separate application namespace for infrastructure-specific policy and lifecycle data.

| Concern | Prefer |
|---------|--------|
| Provider, requested/response model, token usage, finish reason | `gen_ai.*` semantic conventions |
| Agent/workflow and tool invocation spans | Standard `invoke_agent`, `invoke_workflow`, and `execute_tool` operation names |
| Session/run, organization, autonomy level and ceiling, policy revision, credential scope | Your `agent.*` or product namespace |
| Prompts, tool arguments/results, retrieved documents | Redacted references or opt-in content capture, never raw by default |

The GenAI conventions evolve separately from core OpenTelemetry semantic conventions and the repository has no tagged releases yet, so pin the commit (or the instrumentation library version) your spans and dashboards were built against, and treat attribute renames as a migration. Define a consistent namespace for the remaining domain-specific spans:

| Category | Attribute | Example Value |
|----------|-----------|---------------|
| **Correlation** | `agent.request.id` | `req-abc123` |
| | `agent.correlation.id` | `corr-xyz789` |
| | `agent.traceparent` | W3C traceparent |
| | `agent.tracestate` | W3C tracestate |
| **Agent context** | `agent.session.id` | `sess-456` |
| | `agent.run.id` | `run-789` |
| | `agent.type` | `remediation` |
| | `agent.definition.source` | `managed` |
| | `agent.definition.version` | Git SHA or policy version |
| | `agent.permission.mode` | `read-only` / `validate` |
| | `agent.organization.id` | `org-123` |
| **Task context** | `agent.task.type` | `compliance-fix` |
| | `agent.task.priority` | `high` |
| | `agent.trigger.source` | `schedule` |
| **Execution** | `agent.iteration.count` | `3` |
| | `gen_ai.usage.input_tokens` (standard) | `12500` |
| | `gen_ai.usage.output_tokens` (standard) | `3200` |
| | `agent.tool_calls.count` | `8` |
| **Performance** | `agent.perf.repo_clone_ms` | `4500` |
| | `agent.perf.pipeline_wait_ms` | `62000` |
| | `agent.perf.total_duration_ms` | `180000` |
| **Queue** | `agent.queue.depth` | `12` |
| | `agent.queue.wait_ms` | `3200` |
| **Credentials** | `agent.credential.scope` | `management.azure.com` |
| | `agent.credential.expires_at` | ISO timestamp |

The key principle: every span should carry enough context to answer "which agent, which task, which organization, how long, how much?"

### Propagating Trace Context Across Async Boundaries

Long-running agents cross queues, streams, RPC boundaries, SSE channels, and resume flows. Correlation IDs help, but they are not enough on their own. Propagate **W3C trace context** (`traceparent`, `tracestate`, `baggage`) through:

- task dispatch messages
- task output events
- RPC requests/responses
- action trail records
- persisted run metadata when a session resumes on a new worker

Without this, you can observe each component but still fail to reconstruct one logical run end-to-end.

### Instrumenting the Agent Lifecycle

Create nested spans for each phase of agent execution. This gives you a trace tree that shows exactly where time was spent:

```typescript
import { trace, SpanStatusCode } from '@opentelemetry/api';

const tracer = trace.getTracer('agent-worker');

async function processAgentTask(task: AgentTask) {
  return tracer.startActiveSpan('agent.process_task', async (span) => {
    span.setAttributes({
      'agent.session.id': task.sessionId,
      'agent.type': task.agentType,
      'agent.task.type': task.type,
      'agent.trigger.source': task.trigger.source,
    });

    try {
      // Clone repository
      await tracer.startActiveSpan('agent.clone_repo', async (cloneSpan) => {
        const startMs = Date.now();
        await cloneRepository(task.repository);
        cloneSpan.setAttribute('agent.perf.repo_clone_ms', Date.now() - startMs);
        cloneSpan.end();
      });

      // Run agent
      const result = await tracer.startActiveSpan('agent.run', async (runSpan) => {
        const result = await executeAgent(task);
        runSpan.setAttributes({
          'agent.iteration.count': result.iterations,
          'gen_ai.usage.input_tokens': result.inputTokens,
          'gen_ai.usage.output_tokens': result.outputTokens,
          'agent.tool_calls.count': result.toolCalls,
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
  agentDefinition?: {
    name: string;
    source: 'managed' | 'project' | 'user' | 'plugin' | 'runtime';
    version?: string;
    tools?: string[];
    permissionMode?: string;
  };
  traceContext?: {
    traceparent?: string;
    tracestate?: string;
    baggage?: string;
  };
  type:
    | 'tool_call'
    | 'tool_result'
    | 'credential_issued'
    | 'policy_evaluated'
    | 'pipeline_triggered'
    | 'pr_created'
    | 'human_input_requested'
    | 'human_input_received'
    | 'state_restored'
    | 'run_resumed'
    | 'orphan_recovered'
    | 'policy_denied'
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

## Failure Taxonomy and Recovery Events

Don't stop at "success" and "failed." Instrument *why* the run ended and whether recovery happened.

| Category | What It Means | Why It Matters |
|----------|---------------|----------------|
| **AUTH_EXPIRED** | Token/session expired and refresh was attempted | Distinguishes credential churn from real task failure |
| **TRANSIENT** | Network, 429, temporary provider issue | Shows retry pressure and queue amplification |
| **PERMANENT** | Invalid request, missing permissions, bad repo state | Indicates agent or configuration defect |
| **TIMEOUT** | Agent or external pipeline exceeded budget | Drives budget and watchdog tuning |
| **WAITING_INPUT** | Run paused on human checkpoint | Important for UX latency and abandonment tracking |
| **RECOVERED** | Worker crashed but run resumed/restored successfully | Proves durability design is working |

Emit explicit events for transitions such as `waiting_input_entered`, `waiting_input_timed_out`, `state_restored`, and `orphan_recovered`. Recovery should be visible in traces and dashboards, not inferred from gaps in logs.

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
  otlphttp/loki:                 # Loki 3.x ingests OTLP natively; the old `loki` exporter was removed
    endpoint: http://loki:3100/otlp
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
      exporters: [otlphttp/loki]
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
| Active sessions | `count(agent_sessions_active)` | Current load |
| Avg completion time | `histogram_quantile(0.95, agent_task_duration_seconds_bucket)` | Performance |
| Success rate | `rate(agent_tasks_total{status="success"}[1h]) / rate(agent_tasks_total[1h])` | Reliability |
| Token usage | `sum(rate(agent_tokens_total[1h])) by (agent_type)` | Cost tracking |
| Queue depth | `agent_queue_depth` | Backlog |
| PRs created | `increase(agent_prs_created_total[24h])` | Output |
| Credential issuances | `rate(agent_credentials_issued_total[1h])` | Security audit |
| Waiting-input age | `max(agent_waiting_input_age_seconds)` | Human bottlenecks |
| Orphan recoveries | `increase(agent_orphan_recoveries_total[24h])` | Durability health |
| Policy denials | `increase(agent_policy_denials_total[24h])` | Guardrail pressure |

---

## Alert on Failure Patterns, Not Just Volume

Dashboards show load; they rarely show that one kind of work has quietly stopped. We found three separate silent failures in our own platform by writing ad-hoc SQL against run history, each of them weeks old, before we built pattern alerts. The approach that works:

1. **Normalize error messages into classes.** Strip IDs, timestamps, URLs, and numbers so "clone failed for ref abc123" and "clone failed for ref def456" become one class.
2. **Alert on four signals:**

| Signal | Example threshold |
|--------|-------------------|
| New error class | ≥3 occurrences in 1h, none in the previous 7 days |
| Burst of one class | ≥10 occurrences in 1h |
| Per-organization failure rate | ≥90% of runs failed over 24h |
| Overall failure rate | ≥30% of runs failed over 1h |

3. **Evaluate automated work as its own slice.** Compute the same signals separately for runs started by people and runs started by schedules, automated reviews, and goals. An organization whose automated reviews all fail can look healthy overall because its interactive chat works.

### Every Await Needs a Deadline

Every call to an external dependency (model API, Git provider, Redis, the sandbox provider) needs a timeout, including calls to infrastructure you consider local. A single Redis command that never returned left one replica's review dispatcher waiting forever, and 11 PR reviews were silently dropped before anyone noticed. Work that continues after you return HTTP 200 also needs its own "still pending" signal, such as a pending-age metric or a watchdog that flags requests accepted but not started within a few minutes, because the caller has already been told everything is fine.

### One Telemetry Backend for Host and Runtime

If the agent runtime runs as a separate service, pull its logs and traces into the same backend as the host application, joined by the same trace context. Otherwise a failure that starts in the host and surfaces in the runtime (or the reverse) has to be reconstructed by hand from two systems.

### Periodic Jobs Need a Leader

In a service with several replicas, every timer-driven job (canaries, sweeps, digest senders) needs a leader lease so it runs once per interval, not once per replica. Our health canary ran on every replica: it created between 357 and 630 sandbox-booting sessions a day and reported each failure once per replica, which inflated both cost and the apparent failure rate.

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

## In Opengeni

Opengeni's replayable event log is the action trail: every event of every session is stored in Postgres, and live views are built from it rather than alongside it. Its [application observability boundary](https://github.com/Cloudgeni-ai/opengeni/blob/main/docs/application-observability.md) keeps raw prompts, tool content and SQL out of telemetry, and records operational failures with a closed `errorClass` and `errorCode`, which is what makes the failure-pattern alerts above possible.

---

## Next Chapter

[Chapter 10: Autonomous Operations & Notifications →](./10-autonomy-notifications.md)

---

*Built by the team at [Cloudgeni](https://cloudgeni.ai) — Scale your infrastructure team. With Agents. Safely.*
