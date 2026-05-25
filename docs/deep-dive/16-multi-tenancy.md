# Multi-Tenancy

## Purpose

This document establishes the multi-tenancy model for the Agent Platform. It defines the namespace concept, its impact on the database schema, provisioning API, credentials isolation, and runtime behaviour.

This fills a gap identified during design review: the original architecture treats each operator as a single user of the platform, but the business model implies multiple operators (tenants) sharing one platform instance. Retrofitting multi-tenancy after v6 would require breaking schema changes and API redesigns. This document anchors the decision early so all subsequent phases build on a solid foundation.

---

## AD-26: Namespace-based multi-tenancy

**Decision**: The platform is multi-tenant by default, scoped by a `namespace` identifier. Every resource (agents, tools, MCPs, workflow runs, conversations, credentials) belongs to exactly one namespace. Cross-namespace access is prohibited by default and explicitly allowlisted by Platform Admins.

**Rationale**: The platform's stated value proposition is enabling "a website operator to provision a complete customer-support chatbot system." When multiple operators use the same platform instance, their configurations, data, and credentials must be completely isolated. Building in `DEFAULT` namespace semantics from day one costs almost nothing but prevents a painful migration later. A platform that works correctly with one namespace trivially scales to many.

**Implication**:
- `namespace` is a required field on all agent DSL documents.
- All database tables that hold per-operator resources include a `namespace_id` column.
- All API endpoints enforce namespace-scoped authorization: a caller authenticated to namespace A cannot read or write namespace B resources.
- Workers load only resources belonging to the namespace of the workflow they are executing.

---

## Deployment Modes

The namespace model supports three deployment topologies without code changes:

| Mode | Description | Namespace Usage |
|---|---|---|
| **Single-tenant** | One organization runs its own platform instance | Single namespace (`default`); multi-tenancy machinery is present but transparent |
| **Multi-tenant SaaS** | Platform operator serves multiple customers | Each customer gets a namespace; complete data isolation |
| **Internal multi-team** | One company with multiple product teams | Each team gets a namespace; optional cross-namespace agent sharing |

The platform does not hardcode any of these modes. The `namespace` field and the authorization layer make all three possible from the same codebase.

---

## Namespace Properties

```json
{
  "id": "ns-uuid",
  "slug": "acme-corp",
  "display_name": "Acme Corporation",
  "created_at": "ISO 8601",
  "status": "active | suspended | deleted",
  "quotas": {
    "max_agents": 100,
    "max_workflow_runs_per_day": 10000,
    "max_concurrent_runs": 50,
    "max_chatbot_sessions": 500
  },
  "settings": {
    "allowed_providers": ["claude", "openai"],
    "default_provider": "claude",
    "data_residency_region": "us-east-1"
  }
}
```

| Field | Description |
|---|---|
| `slug` | URL-safe identifier used in API paths and DSL `namespace` field. Unique, immutable after creation. |
| `status` | `suspended` blocks new workflow starts and chatbot sessions but does not cancel in-flight work. |
| `quotas` | Per-namespace resource limits enforced at runtime (not just configuration). |
| `allowed_providers` | Restricts which LLM providers agents in this namespace may use. |
| `data_residency_region` | Hints for data placement (application database, Qdrant collection). Enforcement is infrastructure-level. |

---

## Database Schema Impact

Every table that holds per-namespace data includes a `namespace_id` foreign key. This is established from v1 — even though v1 may only ever see one namespace, the column is present and indexed.

### Tables with `namespace_id`

| Table | Notes |
|---|---|
| `agents` | All agent definitions are namespace-scoped. |
| `agent_versions` | Versions belong to the same namespace as their parent agent. |
| `workflow_runs` | Runs are scoped by the namespace of the agent that was invoked. |
| `workflow_step_runs` | Derived from workflow run; implicitly namespace-scoped. |
| `credentials` | Credentials are namespace-scoped. Namespace A's credentials are invisible to namespace B. |
| `conversations` | Conversations belong to a namespace. |
| `conversation_turns` | Derived from conversation; implicitly namespace-scoped. |
| `conversation_pending_tasks` | Derived from conversation; implicitly namespace-scoped. |
| `tools` | Tools may be global (platform-level) or namespace-scoped (overrides or private tools). |
| `mcps` | MCPs may be global or namespace-scoped. |

### Schema sketch

```sql
CREATE TABLE namespaces (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  slug          TEXT NOT NULL UNIQUE,
  display_name  TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'active',
  quotas        JSONB NOT NULL DEFAULT '{}',
  settings      JSONB NOT NULL DEFAULT '{}',
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE agents (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  namespace_id  UUID NOT NULL REFERENCES namespaces(id),
  name          TEXT NOT NULL,
  -- ... other fields
  UNIQUE (namespace_id, name)
);

CREATE INDEX idx_agents_namespace ON agents (namespace_id);
```

The `UNIQUE (namespace_id, name)` constraint is important: agent names are unique within a namespace, not globally.

---

## API Impact

### Namespace in the URL

All API endpoints are namespace-scoped. The namespace slug appears as a path parameter:

```
POST   /namespaces/{namespace}/agents
GET    /namespaces/{namespace}/agents/{id}
POST   /namespaces/{namespace}/agents/{id}/invoke
GET    /namespaces/{namespace}/workflow-runs
GET    /namespaces/{namespace}/workflow-runs/{run_id}
POST   /namespaces/{namespace}/tools
GET    /namespaces/{namespace}/mcps
```

Platform Admin endpoints for namespace management:

```
POST   /admin/namespaces
GET    /admin/namespaces/{namespace}
PATCH  /admin/namespaces/{namespace}
GET    /admin/namespaces/{namespace}/quota-usage
```

### Authentication Binding

API tokens (or other auth mechanisms) are scoped to a namespace. A request carrying namespace A's token accessing `/namespaces/namespace-b/...` receives a `403 Forbidden`. Platform Admins carry a super-token that spans all namespaces.

---

## Credentials Isolation

The `CredentialsProvider` interface (introduced in v3) resolves credentials by `(namespace_id, scope)`. Two namespaces may have credentials with the same name (e.g., both have a `jira_api_key`) — they are completely separate records and neither is visible to the other.

At workflow start, the credentials snapshot loaded into the workflow context is scoped to the workflow's namespace. A credential lookup for namespace A will never return a record belonging to namespace B.

---

## Cross-Namespace Agent Calls

By default, `agent_call` steps can only call agents within the same namespace. Cross-namespace calls require an explicit allowlist entry, managed by the Platform Admin:

```json
{
  "source_namespace": "acme-corp",
  "target_namespace": "shared-utilities",
  "allowed_agents": ["text-normalizer", "language-detector"]
}
```

When a cross-namespace call is allowed:
- The called agent runs with the *calling* namespace's security context (it cannot access the target namespace's credentials or data).
- The call is logged with both namespace IDs for audit.
- The target namespace's quota is charged for the execution.

**The validation service enforces this**: a DSL document referencing an agent in another namespace fails validation unless the allowlist entry exists.

---

## Tool and MCP Registry Scoping

Tools and MCPs have two registration levels:

| Level | Scope | Managed By |
|---|---|---|
| **Platform-level** | Available to all namespaces | Platform Admin |
| **Namespace-level** | Available only within the namespace | Namespace Admin (Platform Admin role within namespace) |

Namespace-level registrations can shadow platform-level ones (same name, different implementation). The namespace-level registration takes precedence.

This allows a namespace to register a private tool or override the default knowledge base tool with a custom implementation without affecting other namespaces.

---

## Quota Enforcement

Quotas are enforced at the API and runtime layers:

| Quota | Enforcement Point |
|---|---|
| `max_agents` | Provisioning API: reject if count ≥ limit |
| `max_workflow_runs_per_day` | Invocation API: reject if daily count ≥ limit |
| `max_concurrent_runs` | Invocation API: reject if active run count ≥ limit |
| `max_chatbot_sessions` | Chatbot endpoint: reject new sessions if count ≥ limit |
| `max_tokens_per_day` | Invocation API / async enforcer: reject or alert when daily token budget is exhausted |
| `max_estimated_cost_per_day_usd` | Alerting-only or hard-reject (configurable per namespace) |

Quota counters are maintained in the database (not in-memory) so they survive service restarts and work correctly across API instances.

The orchestrator/monitor queries quota usage and exposes it via the Operator API for alerting.

### Atomic Quota Enforcement (E-22)

All quota checks use **atomic database operations** to prevent TOCTOU race conditions under concurrent invocations. Example for `max_concurrent_runs`:

```sql
UPDATE namespace_quotas
SET concurrent_runs = concurrent_runs + 1
WHERE namespace_id = $1
  AND concurrent_runs < max_concurrent_runs
RETURNING concurrent_runs;
```

If the update affects 0 rows (quota already at limit), the API returns `429 Too Many Requests`. On workflow completion, the counter is decremented atomically in the same pattern. This pattern applies to all mutable quota counters.

### Cost Tracking (E-18)

The `workflow_runs` table tracks per-run token consumption and estimated cost:

| Column | Type | Notes |
|---|---|---|
| `total_tokens_input` | integer | Summed from all `llm_call` step metadata |
| `total_tokens_output` | integer | Summed from all `llm_call` step metadata |
| `estimated_cost_usd` | decimal | Calculated using the platform's provider pricing table |

Namespace-level daily quotas for `max_tokens_per_day` and `max_estimated_cost_per_day_usd` are tracked in a `namespace_daily_usage` table, reset at UTC midnight. Enforcement mode is configurable per namespace:
- `hard`: reject new invocations when the budget is exhausted
- `alert`: allow invocations but alert the Operator

**Cost API**: `GET /namespaces/{ns}/cost-summary?from=&to=` returns aggregated token and cost data.

**Prometheus metrics**:
- `agent_platform_tokens_total{namespace, agent, provider, model, step_type}`
- `agent_platform_estimated_cost_usd_total{namespace, provider, model}`

---

## Migration from Single-Namespace

For teams adopting the platform before multi-tenancy was formalized, migration is:

1. Create a `default` namespace.
2. Run a migration that sets `namespace_id = default_namespace_id` on all existing rows.
3. Issue new API tokens scoped to the `default` namespace.
4. Update DSL documents to include `"namespace": "default"`.

This migration is additive, idempotent, and non-breaking for existing code.

---

## What to Read Next

- **[01-PRD](./01-PRD.md)** — AD-26 (multi-tenancy decision) in the full decision log
- **[09-Security Model](./09-security-model.md)** — Tenant isolation and authorization
- **[02-v1](./02-v1-agent-provisioning-foundation.md)** — v1 schema where `namespace_id` is first introduced
- **[14-dsl-spec.md](./14-dsl-spec.md)** — `namespace` field in top-level DSL structure

---

## Related Documents

- [01-PRD](./01-PRD.md) — AD-26 (this decision), AD-07 (credentials), AD-09 (provider configuration)
- [09-Security Model](./09-security-model.md) — Persona-based access control, workflow isolation
- [10-Deployment Concepts](./10-deployment-concepts.md) — Data residency, network segmentation
