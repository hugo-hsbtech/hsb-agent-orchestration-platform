# Deployment Concepts

## Purpose

This document defines the conceptual deployment architecture for the agent platform. It establishes deployment principles, component topology, and operational concerns without prescribing specific container orchestrators, cloud providers, or infrastructure-as-code tools.

---

## Deployment Philosophy

### Infrastructure Agnosticism

The platform is designed to deploy to diverse environments:
- Container orchestrators (Kubernetes, ECS, Nomad, etc.)
- Virtual machines (managed or self-hosted)
- Serverless platforms (with constraints)
- Hybrid and multi-cloud configurations

**Principle**: Platform code makes no assumptions about the underlying infrastructure. Deployment concerns (service discovery, secret injection, scaling) are externalized.

### Separation of State and Compute

| Component | Characteristic | Deployment Implication |
|---|---|---|
| **Database** | Stateful, persistent | Managed service or replicated cluster |
| **Temporal** | Stateful (workflow history), persistent | Dedicated cluster, separate from application |
| **Workers** | Stateless, ephemeral | Horizontally scalable, replaceable |
| **API layer** | Stateless, session-less | Horizontally scalable, behind load balancer |

---

## Component Topology

### Core Services

```
┌─────────────────────────────────────────────────────────────┐
│                         EDGE                                │
│  (Load Balancer, TLS termination, rate limiting)            │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   API Service   │  │  API Service    │  │  API Service    │
│   (Instance 1)  │  │  (Instance 2)   │  │  (Instance N)   │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
              ┌───────────────────────────────┐
              │     Workflow Orchestrator     │
              │    (Polls database state)     │
              └───────────────┬───────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Temporal Worker │  │ Temporal Worker │  │ Temporal Worker │
│   (Pool A)      │  │   (Pool B)      │  │   (Pool C)      │
│ LLM-bound steps │  │  Tool/MCP steps │  │  General steps  │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
              ┌───────────────────────────────┐
              │     Service Dependencies        │
              │  ┌─────────┐    ┌─────────┐   │
              │  │Database │    │Temporal │   │
              │  │ (State) │    │ (Workflow) │
              │  └─────────┘    └─────────┘   │
              │  ┌─────────┐    ┌─────────┐   │
              │  │ Qdrant  │    │  Vault  │   │
              │  │   (KB)  │    │(Secrets)│   │
              │  └─────────┘    └─────────┘   │
              └───────────────────────────────┘
```

### Worker Pool Segmentation

Workers may be specialized by workload characteristics:
- **General workers**: Handle mixed step types
- **LLM-bound workers**: Handle `llm_call` steps (API-waiting, low CPU)
- **Tool workers**: Handle `tool_call` and `mcp_call` (I/O heavy, potential latency)
- **Compute workers**: Handle `transform` steps (CPU-bound)

**Principle**: Segmentation is optional. The platform works with a single worker pool; segmentation is an optimization for large-scale deployments.

---

## Configuration Management

### Environment-Specific Configuration

| Configuration Type | Examples | Management Approach |
|---|---|---|
| **Static** | Port numbers, timeout defaults | Built into container image, overridden by env vars |
| **Dynamic** | Database connection strings, provider endpoints | Injected at runtime (env vars, config service)
| **Secrets** | API keys, database passwords, vault tokens | Injected via secrets management (never in images) |
| **Feature flags** | New DSL features, beta providers | Runtime toggleable without redeployment |

### Configuration Validation

On startup, each service validates its configuration:
- Required parameters present
- Connection endpoints reachable
- Credentials valid (test query, not full operation)
- Feature flags reference known values

**Failure behavior**: Invalid configuration causes immediate exit with clear error message (fail fast).

---

## Scaling Patterns

### Horizontal Scaling

| Component | Scaling Trigger | Scaling Mechanism |
|---|---|---|
| **API instances** | Request rate, latency | Add/remove instances behind load balancer |
| **Workers** | Queue depth, processing lag | Add/remove workers (Temporal handles task distribution) |
| **Database** | Query latency, connection count | Read replicas, connection pooling (platform-agnostic) |
| **Temporal** | Workflow execution rate | Temporal cluster scaling (separate from application) |

### Vertical Scaling

Worker instances may scale vertically for:
- Memory-intensive workflows (large data transformations)
- CPU-intensive activities (complex calculations)

**Note**: Vertical scaling is less preferred than horizontal; it creates single points of contention.

---

## High Availability

### Redundancy Levels

| Tier | Redundancy Approach | RTO Target |
|---|---|---|
| **API layer** | Multi-instance behind LB | Seconds (auto-failover) |
| **Workers** | Multiple workers, Temporal reschedules | Minutes (Temporal redistributes tasks) |
| **Database** | Replicated, automatic failover | Minutes (dependent on replication) |
| **Temporal** | Clustered with persistence | Minutes (dependent on cluster health) |

### Failure Modes

| Failure | Impact | Response |
|---|---|---|
| **Single API instance** | Reduced capacity | Load balancer routes to healthy instances |
| **Single worker** | Reduced throughput | Temporal reschedules in-flight tasks |
| **All workers** | Queue buildup, eventual timeout | Alert Operator, investigate root cause |
| **Database replica** | Read degradation | Fallback to primary, alert Operator |
| **Database primary** | Write halt | Automatic failover to replica, brief unavailability |
| **Temporal unavailable** | New workflows cannot start | Queue incoming requests, alert Operator |

---

## Observability Integration

### Metrics Export

Platform services expose metrics in standard format:
- Prometheus-compatible endpoint (`/metrics`)
- Key metrics: workflow throughput, step latency, error rates, queue depth
- Cardinality control (bounded label sets)

### Distributed Tracing

Trace propagation across components:
- API edge generates trace ID
- Carried through to Temporal workflows (as workflow attribute)
- Propagated to activities, provider calls, tool/MCP invocations
- Database operations tagged with trace ID

### Logging

Structured logging with consistent fields:
- Timestamp, severity, service name
- Trace ID, workflow run ID, step ID
- Message, context (structured, not free-form)
- No sensitive data (credentials, PII) in logs

---

## Deployment Lifecycle

### States of a Deployment

1. **Provisioning**: Infrastructure resources created (networks, storage, compute)
2. **Configuration**: Secrets injected, environment variables set
3. **Startup**: Services initialize, validate configuration, connect to dependencies
4. **Health check**: Readiness probes confirm service functional
5. **Traffic serving**: Load balancer routes requests to healthy instances
6. **Monitoring**: Metrics, logs, alerts active

### Shutdown Sequence

1. **Drain**: Stop accepting new requests/workflows
2. **Complete in-flight**: Finish active requests, checkpoint workflows
3. **Disconnect**: Close database connections, release resources
4. **Terminate**: Exit cleanly

**Graceful degradation**: If shutdown is forced, in-flight work may be interrupted. Temporal will reschedule interrupted workflows to healthy workers.

---

## Security in Deployment

### Network Segmentation

| Zone | Access Pattern |
|---|---|
| **Public** | Edge/API layer only |
| **Internal** | Services communicate (API → Database, Workers → Temporal) |
| **Restricted** | Secrets vault, database primary |

**Principle**: Services have network access only to their dependencies (least privilege).

### Secret Injection

Secrets are injected at runtime, never baked into images:
- Environment variables (for simple cases)
- Mounted files (for certificates, multi-line secrets)
- Sidecar injection (for dynamic vault integration)

### Image Security

- Base images from trusted sources
- Minimal image layers (no build tools, debug utilities in production)
- Vulnerability scanning in build pipeline
- Image signing and verification

---

## Operations Runbook (Conceptual)

### Scenario: Elevated Error Rate

1. Check metrics dashboard for error type distribution
2. If provider errors: Check provider status, consider failover
3. If database errors: Check connection pool, query performance
4. If MCP errors: Check MCP availability, circuit breaker status
5. If unclear: Enable debug logging, identify affected workflow pattern

### Scenario: Queue Backlog

1. Check worker pool utilization (are workers saturated?)
2. If yes: Scale workers horizontally
3. If no: Check for blocked workflows (paused, failed retries)
4. Check for upstream slowness (slow MCP, rate-limited provider)

### Scenario: Database Performance

1. Check query latency metrics
2. Identify slow query patterns
3. Check connection pool saturation
4. Consider read replica scaling for Operator queries
5. Plan archival of old workflow runs if table size is factor

---

## Temporal / DB Reconciliation (E-08)

The platform uses two state stores with different roles:
- **Temporal**: execution truth — the authoritative record of what happened and in what order.
- **Application DB**: the projection — the product-visible state machine that all APIs query.

### The Split-Brain Risk

If Temporal progresses a workflow step while the application DB is temporarily unavailable to record it, the two stores diverge. When the DB recovers, workflows visible as `running` in Temporal may appear stale or inconsistent in the DB.

### Contract: All Activity Writes are Idempotent

Every Temporal activity that writes a step result to the application DB must be idempotent. The step run row primary key is derived from `(workflow_run_id, step_id, attempt_number)` — an upsert on this key is always safe to replay.

### Recovery Procedure

On DB recovery:
1. A **reconciliation job** queries Temporal for workflow runs in `RUNNING` state.
2. For each run, it queries the application DB for the corresponding `workflow_runs` row.
3. If the DB record is stale (missing step completions that Temporal history shows), the reconciliation job replays the activity output events from Temporal history into the DB via idempotent upserts.
4. The acceptable divergence window is bounded by the reconciliation job's polling interval (configurable, default: `30s`).

### Authority Rule

> **Temporal is the execution truth. The application DB is the projection. Never update Temporal to match the DB; always update the DB to match Temporal.**

If a DB record and Temporal history conflict, Temporal history wins. The reconciliation job never writes to Temporal.

---

## Out of Scope (Implementation Detail)

The following are deferred to infrastructure-specific implementation:
- Container orchestrator choice (Kubernetes, ECS, Nomad)
- Infrastructure-as-code tools (Terraform, Pulumi, CloudFormation)
- CI/CD pipeline (GitHub Actions, GitLab CI, Jenkins)
- Service mesh (Istio, Linkerd) or direct service communication
- Specific cloud provider services (ALB, CloudWatch, etc.)
- Container registry and image build process
- Certificate management (Let's Encrypt, ACM, cert-manager)

---

## What to Read Next

Deployment enables all platform capabilities:

- **[08-Performance and Scaling](./08-performance-and-scaling.md)** — Capacity planning for your deployment topology
- **[09-Security Model](./09-security-model.md)** — Network segmentation and security boundaries
- **[12-Upgrades and Migrations](./12-upgrades-and-migrations.md)** — Deployment procedures and rollback strategies
- **[11-Testing Strategy](./11-testing-strategy.md)** — Environment hierarchy and staging parity

Or review phase documents:
- [03-v2](./03-v2-workflow-orchestration.md) — Worker deployment patterns
- [07-v6](./07-v6-versioning-and-hardening.md) — Observability integration

---

## Related Documents

- [08-Performance and Scaling](./08-performance-and-scaling.md) — Capacity planning, scaling triggers
- [09-Security Model](./09-security-model.md) — Network security, secret management
- [12-Upgrades and Migrations](./12-upgrades-and-migrations.md) — Deployment versioning, rollback
- [07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md) — Observability, metrics, tracing
