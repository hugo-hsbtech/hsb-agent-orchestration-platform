# Performance and Scaling Guidelines

## Purpose

This document establishes conceptual performance boundaries and scaling principles for the agent platform. It provides the framework for capacity planning without prescribing implementation-specific tuning parameters.

---

## Performance Philosophy

The platform adopts a **tiered latency model** that reflects the fundamental differences between execution modes:

| Execution Tier | Expected Latency | Use Case |
|---|---|---|
| **Synchronous (Chatbot)** | Sub-second to first token | Real-time user conversation |
| **Asynchronous (Workflow)** | Seconds to minutes per step | Background processing, complex multi-agent work |
| **Fire-and-forget** | No blocking latency | Fan-out operations, notifications |

This separation allows each tier to optimize for its constraints without compromising the others.

---

## Scaling Dimensions

### 1. Horizontal Worker Scaling

**Principle**: Workers are stateless and horizontally scalable per workload type.

**Scaling Triggers**:
- Workflow queue depth exceeding processing capacity
- Step type-specific saturation (LLM-bound vs. compute-bound steps)
- Fan-out parallelism limits

**Key Insight**: Not all steps scale identically. An `llm_call` step is API-bound (latency dominated by provider response time); a `transform` step is CPU-bound; a `tool_call` step may be I/O-bound. Worker pools should be segmentable by step characteristics.

### 2. Fan-Out Limits

**Principle**: Unbounded parallelism is a reliability risk.

**Control Mechanisms**:
- Per-`parallel` block: maximum concurrent children
- Per-`for_each`: concurrency limit with ordered output preservation
- Global: workflow run concurrency quotas per agent

**Conceptual Bound**: The `for_each` construct must specify a `max_concurrency` bound. The platform rejects DSL documents with unbounded collection processing.

### 3. Database Connection Scaling

**Principle**: The database is the state machine; its capacity bounds the platform.

**Scaling Considerations**:
- Workflow run and step run tables are write-heavy during execution
- Chatbot conversation tables are read/write balanced
- Credential snapshots are read-once-per-workflow

**Architectural Response**: Connection pooling, read replicas for Operator queries, and archival of completed runs to cold storage.

---

## Latency Budgets by Step Type

| Step Type | Expected Latency | Dominant Factor | Retry Strategy |
|---|---|---|---|
| `llm_call` | 500ms–10s | Provider API response | Exponential backoff, provider-aware |
| `tool_call` | 10ms–5s | Tool implementation (in-memory vs. external) | Fast retry for transient failures |
| `mcp_call` | 100ms–30s | External service latency | Configurable per MCP, circuit breaker pattern |
| `transform` | <100ms | Data size and expression complexity | No retry (deterministic) |
| `agent_call` (sync) | Sum of child workflow | Child workflow complexity | Inherited from child |
| `agent_call` (fire-and-forget) | <100ms | Orchestrator dispatch | No retry (orphan risk) |

**Design Implication**: Steps with high variance (`mcp_call`) need explicit timeout declarations in the DSL. Steps with low variance (`transform`) need no retry configuration.

---

## Conversation Streaming Performance

### Time-to-First-Token (TTFT)

The chatbot layer optimizes for perceived responsiveness. TTFT targets:

- **Routing classification**: <200ms (fast classifier or constrained LLM call)
- **Specialist response start**: <500ms (includes KB lookup initiation if applicable)
- **Full token stream**: Continuous, with backpressure handling for slow clients

### Backpressure and Buffering

Streaming endpoints handle backpressure through:
- Client-side buffering with timeout thresholds
- Partial response persistence for reconnect scenarios
- Graceful degradation to non-streaming for capacity-constrained situations

---

## Knowledge Base Query Performance

The Qdrant-backed knowledge base has variable latency based on retrieval strategy:

| Strategy | Expected Latency | Accuracy Trade-off |
|---|---|---|
| Semantic only | 50–200ms | Fast, may miss exact matches |
| Hybrid (semantic + keyword) | 100–500ms | Balanced accuracy and speed |
| Reranked | 200ms–1s | Highest accuracy, highest latency |

**Per-Specialist Tuning**: Specialists declare their KB strategy in the DSL. High-volume customer support might prefer semantic-only; technical documentation lookup might prefer hybrid.

---

## Capacity Planning Framework

### Workload Classification

| Workload Pattern | Characteristics | Scaling Approach |
|---|---|---|
| **Burst** | Short spikes (campaign launches, announcements) | Pre-scaled worker pools, queue buffering |
| **Steady** | Consistent baseline (ongoing support) | Auto-scaling based on queue depth |
| **Batch** | Large fan-outs (report generation, data processing) | Concurrency-limited execution, schedule during low-traffic |
| **Interactive** | Chatbot turns with real-time expectations | Dedicated streaming infrastructure, latency prioritization |

### Resource Isolation

**Principle**: Critical workloads must not be starved by background batch work.

**Mechanism**: Task queue segmentation (conceptual — implementation may use separate queues, priority classes, or worker pool isolation). The DSL does not expose queue selection; the platform assigns workloads based on agent type and step characteristics.

**Worker auto-scaling trigger (E-38)**: The Orchestrator/Monitor emits a `high_pending_runs_total` counter when the count of `RUNNING` workflow runs exceeds `MONITOR_SCALE_THRESHOLD` (default: `1000`). This is the canonical metric for triggering worker pool scaling. Infrastructure teams attach a Kubernetes HPA or Keda `ScaledObject` to this metric. The monitor does not scale directly; it only signals.

---

## Failure Mode Performance Impact

| Failure Type | Impact Scope | Recovery Time |
|---|---|---|
| Single step failure | One workflow run | Retry policy dependent |
| Provider outage | All `llm_call` steps to that provider | Failover to backup provider (if configured) |
| MCP unavailability | Steps referencing that MCP | Degradation to cached manifest, fallback behavior |
| Worker death | In-flight steps on that worker | Temporal reschedules to healthy worker |
| Database degradation | Platform-wide | Graceful degradation (queue buildup, alert Operator) |

---

## Monitoring Indicators

**Leading indicators** (predict problems):
- Queue depth growth rate
- Step latency p95/p99 trends
- Provider error rate changes
- Fan-out execution time vs. linear projection

**Lagging indicators** (confirm problems):
- Workflow run completion rate drops
- Conversation abandonment rate increases
- Operator intervention frequency

---

## Out of Scope (Implementation Detail)

The following are intentionally deferred to implementation tasks:
- Specific connection pool sizes
- Exact auto-scaling thresholds
- Provider-specific rate limit handling
- CDN or caching layer configuration
- Geographic distribution strategy

---

## Related Documents

- [03-v2 — Workflow Orchestration](./03-v2-workflow-orchestration.md) — step execution and retry policies
- [05-v4 — Multi-Agent Orchestration](./05-v4-multi-agent-orchestration.md) — fan-out and parallelism
- [06-v5 — Chatbot Layer](./06-v5-chatbot-layer.md) — streaming and conversation persistence
- [07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md) — observability and metrics

---

## What to Read Next

Related conceptual documents:

- **[09-Security Model](./09-security-model.md)** — Threat boundaries that impact performance isolation
- **[10-Deployment Concepts](./10-deployment-concepts.md)** — Worker topology and scaling patterns
- **[12-Upgrades and Migrations](./12-upgrades-and-migrations.md)** — Database evolution and archival strategies

Or return to the phases:
- [05-v4](./05-v4-multi-agent-orchestration.md) — Review fan-out execution
- [06-v5](./06-v5-chatbot-layer.md) — Streaming performance targets
