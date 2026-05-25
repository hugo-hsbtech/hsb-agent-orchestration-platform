# Agent Version Promotion and Traffic Management

## Purpose

This document specifies the lifecycle for safely promoting a new agent version from development through staging to full production traffic. It closes a gap in the documentation set: `07-v6-versioning-and-hardening.md` specifies *immutable versioning* (AD-16) and *deprecation*, but contains no process for how a new version is validated and promoted before it becomes `latest`. Without a promotion model, the only safe option is "test locally, deploy to `latest`, hope for the best" — which is not production-grade.

This document defines three promotion mechanisms: **environment-tier routing**, **canary traffic splitting**, and **A/B evaluation**. All three operate within the existing versioning model and require no DSL changes.

---

## The Gap: Versioning Without Promotion

AD-16 establishes that:
- Every agent update creates a new immutable version.
- A `latest` pointer serves new traffic.
- Old versions remain available for pinned callers.

What is missing is: **how does a new version become `latest` safely?** In the current model, any `PATCH /namespaces/{ns}/agents/{id}` immediately creates a new version AND updates `latest`. There is no staging gate, no traffic split, no rollback trigger.

---

## Concepts

### Environment Tiers

The platform supports a logical concept of environment tiers assigned to agent versions:

| Tier | Receives Traffic From | Promotion Gate |
|---|---|---|
| `development` | Explicit version pin by Agent Builder (dry-run, test invocations) | None — builder's own calls |
| `staging` | Staging traffic slice (synthetic or canary real users) | Manual or automated review |
| `production` | All unversioned invocations (the `latest` pointer) | Promotion decision |

Tiers are **labels on agent versions**, not separate deployments. The platform is a single deployment; tiers are a routing and governance layer.

### Version Lifecycle States

```
draft ──► staging ──► canary ──► production (latest)
                                     │
                                     ▼
                               deprecated ──► archived
```

| State | Meaning | Who Sets It |
|---|---|---|
| `draft` | Created, not yet validated for staging | Agent Builder |
| `staging` | Receiving staging traffic; under evaluation | Agent Builder → promotes to staging |
| `canary` | Receiving a configurable % of production traffic | Platform Admin / Agent Builder |
| `production` | Current `latest`; receives all unversioned traffic | Promotion (manual or auto) |
| `deprecated` | Blocked from new `latest` assignment; pinned callers still work | Platform Admin |
| `archived` | No active runs; historical read-only | Automatic after all runs complete |

---

## Promotion Workflow

### Step 1 — Create a Draft Version

When an Agent Builder updates an agent, a new version is created in `draft` state. The `latest` pointer is **not updated** — the previous production version continues serving traffic.

```
POST /namespaces/{ns}/agents/{id}
Body: { ...new DSL... }

Response:
{
  "version_id": "v8",
  "state": "draft",
  "latest_updated": false
}
```

The Agent Builder can test this version explicitly by pinning:

```
POST /namespaces/{ns}/agents/{id}/invoke?version=v8
```

### Step 2 — Promote to Staging

The Agent Builder explicitly promotes the draft to `staging`:

```
POST /namespaces/{ns}/agents/{id}/versions/v8/promote
Body: { "target_state": "staging" }
```

Staging traffic can be:
- **Synthetic**: invocations from the platform's own E2E test suite, routed to the staging version.
- **Shadow**: a subset of real production requests are duplicated and sent to the staging version in parallel (results discarded; only for observation).

Shadow routing configuration in the namespace settings:

```json
{
  "staging_shadow_rate": 0.05
}
```

When `staging_shadow_rate` is set, 5% of production invocations are duplicated and sent to the `staging` version. The shadow invocations run asynchronously; their outputs are logged for comparison but not returned to callers.

### Step 3 — Promote to Canary

Once the Agent Builder is satisfied with staging behavior, they promote to `canary`:

```
POST /namespaces/{ns}/agents/{id}/versions/v8/promote
Body: {
  "target_state": "canary",
  "canary_percent": 10,
  "canary_duration_minutes": 60,
  "auto_promote_on_success": false,
  "auto_rollback_on": {
    "error_rate_above": 0.05,
    "latency_p95_above_ms": 8000
  }
}
```

#### Canary Traffic Splitting

During canary, the invocation router splits traffic:
- `canary_percent` of new unversioned invocations go to `v8`.
- `(100 - canary_percent)` go to the current production version.
- Pinned invocations always go to their pinned version.

Traffic splitting is deterministic per conversation (chatbots) or per caller (workflow agents), not per-request, to avoid mid-conversation version switches.

#### Auto-Rollback

If `auto_rollback_on` thresholds are breached within `canary_duration_minutes`:
1. The canary version is automatically demoted to `staging`.
2. All traffic returns to the production version.
3. A `CanaryRollbackEvent` is emitted (Prometheus metric, operator alert).
4. The Agent Builder is notified via the configured alert channel.

#### Auto-Promotion

If `auto_promote_on_success: true` and canary runs for `canary_duration_minutes` without triggering `auto_rollback_on` thresholds, the canary version is automatically promoted to `production` and `latest` is updated.

### Step 4 — Promote to Production

Manual promotion:

```
POST /namespaces/{ns}/agents/{id}/versions/v8/promote
Body: { "target_state": "production" }
```

This atomically:
1. Updates the `latest` pointer to `v8`.
2. Sets `v8` state to `production`.
3. Sets the previously-latest version to `deprecated` (soft, with a grace period).

---

## A/B Evaluation Mode

Canary traffic splitting provides production validation. A/B evaluation is a separate, deliberate experiment comparing two versions on quality metrics — not just error rates.

### A/B Configuration

```
POST /namespaces/{ns}/agents/{id}/ab-tests
Body: {
  "name": "system_prompt_v8_vs_v7",
  "version_a": "v7",
  "version_b": "v8",
  "split_percent_b": 20,
  "duration_minutes": 1440,
  "evaluation_criteria": ["response_quality_score", "task_completion_rate"]
}
```

### Evaluation Criteria

A/B results are evaluated along configurable dimensions. Built-in dimensions:

| Criterion | Measurement | Source |
|---|---|---|
| `error_rate` | % of runs ending in error | Workflow run records |
| `latency_p95_ms` | 95th percentile step latency | Step run records |
| `token_cost_per_run` | Average tokens × price per run | Step output metadata |
| `response_quality_score` | LLM-graded score (0–1) using a judge model | External evaluation call |
| `task_completion_rate` | % of runs where output passes a declared validator | Step output validation |

`response_quality_score` uses a configurable judge LLM call: a separate `llm_call` invocation sent the original input, both version outputs, and a rubric defined in the A/B configuration. The judge scores each output 0–10. This is similar to LLM-as-judge evaluation frameworks.

### A/B Results API

```
GET /namespaces/{ns}/agents/{id}/ab-tests/{test_id}/results

{
  "test_id": "ab-1234",
  "status": "running | completed | stopped",
  "version_a": { "version": "v7", "sample_count": 400, "metrics": { ... } },
  "version_b": { "version": "v8", "sample_count": 100, "metrics": { ... } },
  "recommendation": "promote_b | keep_a | inconclusive",
  "statistical_significance": 0.95
}
```

`recommendation` is computed using a standard z-test for proportions (error rate) and Mann-Whitney U test for continuous metrics (latency). `inconclusive` is returned when sample size is insufficient for statistical confidence.

---

## Operator CLI Commands

The `agent-platform` CLI (see `17-operator-cli.md`) gains the following commands in v6:

```
agent-platform agents promote <agent-name> <version> --to staging|canary|production
agent-platform agents canary status <agent-name>
agent-platform agents canary abort <agent-name>
agent-platform agents ab-test start <agent-name> <version-a> <version-b> [options]
agent-platform agents ab-test results <agent-name> <test-id>
```

---

## DSL Additions

No DSL changes are required. Version promotion is a platform-level lifecycle operation, not something declared in the agent's DSL. The version's DSL is immutable once created (AD-16); only its `state` label changes during promotion.

---

## Interaction with Existing Systems

### Chatbot Sessions

For chatbot agents (v5), traffic splitting must be session-scoped to avoid switching versions mid-conversation. Implementation: when a new conversation is created, the routing layer assigns a version (based on the canary split) and pins that version for the conversation's lifetime. The `conversations` table gains a `agent_version_id` column.

### Sub-Agent Calls (v4)

When a parent agent calls a sub-agent without a pinned version, the sub-agent's invocation uses the sub-agent's current production version at the time of the parent's workflow start — not the canary. Cross-version promotion interactions are not in scope for v6; explicit version pinning in `agent_call` steps is the recommended pattern for stable multi-agent compositions.

### Versioning Table Schema Addition

The `agent_versions` table gains:

```sql
ALTER TABLE agent_versions ADD COLUMN state TEXT NOT NULL DEFAULT 'production';
ALTER TABLE agent_versions ADD COLUMN canary_percent INTEGER;
ALTER TABLE agent_versions ADD COLUMN promoted_at TIMESTAMPTZ;
ALTER TABLE agent_versions ADD COLUMN promoted_by TEXT;
```

---

## Implementation Phase

| Feature | Phase |
|---|---|
| `draft` state on version creation (don't auto-update `latest`) | v6 (alongside versioning Epic 1) |
| Staging promotion and shadow routing | v6 (new story in Epic 1) |
| Canary traffic splitting and auto-rollback | v6 (new story in Epic 1) |
| A/B evaluation framework | v7 (post-v6 enhancement) |
| Operator CLI promote commands | v6 (alongside CLI Epic 6) |

---

## What to Read Next

With version promotion understood, explore the systems that depend on it:

→ **[07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md)** — The full versioning Epic 1 that this document extends, plus pause/resume/cancel and debug mode.

Or continue with related lifecycle concerns:
- [18-enhancements.md](./18-enhancements.md#e-39) — E-39 (deprecation notification channels) — how builders learn their version is being retired
- [24-dsl-migration-tooling.md](./24-dsl-migration-tooling.md) — Migrating DSL definitions to new versions before promoting
- [11-testing-strategy.md](./11-testing-strategy.md) — Evolution testing and canary validation methodology
- [17-operator-cli.md](./17-operator-cli.md) — CLI commands for promote, canary, and A/B test operations

---

## Related Documents

- [01-PRD.md](./01-PRD.md) — AD-16 (immutable versioning), AD-29 (this decision), PD-07 (version deprecation)
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Agent versioning Epic 1
- [11-testing-strategy.md](./11-testing-strategy.md) — Evolution testing, canary testing concepts
- [17-operator-cli.md](./17-operator-cli.md) — Operator CLI commands for version management
- [06-v5-chatbot-layer.md](./06-v5-chatbot-layer.md) — Session-scoped routing for chatbots
