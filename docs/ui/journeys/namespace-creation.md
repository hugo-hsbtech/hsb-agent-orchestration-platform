# J3 — Namespace Creation

> **Entry:** `New namespace` in the Admin content header (`57:56`, button `215:92`). **Figma worked example:** create `acme-eu` (EU data residency) — frames `211:62` → `212:110`. **Grounds:** [AD-26](../../01-PRD.md) (namespace multi-tenancy), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (namespace properties, quotas, scoping), [23-per-run-cost-cap](../../deep-dive/23-per-run-cost-cap.md) (namespace default budget), [19-provider-rate-limiting](../../deep-dive/19-provider-rate-limiting.md) (per-namespace rate share), [35-compliance-governance](../../deep-dive/35-compliance-governance.md) (residency/retention).

A namespace is the **isolation boundary**: every resource (agents, MCPs, runs, conversations, credentials) belongs to exactly one namespace, and cross-namespace access is prohibited by default ([AD-26](../../01-PRD.md)). Only Platform Admins create namespaces (`/admin/namespaces`). The 4-step wizard: **Identity → Quotas & Budget → Defaults → Review**.

---

## Step 1 · Identity

| Field | Branch / values | Necessary action / validation |
|---|---|---|
| `display_name` | free text | required |
| `slug` | URL-safe, **unique, immutable after creation** | appears in API paths and the DSL `namespace` field; collision → reject ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) `slug` is "unique, immutable after creation") |
| `description` | free text | — |
| **Region / data residency** | `data_residency_region` dropdown (e.g. `us-east-1`, `eu-west-1 (Frankfurt)`) | **hint for data placement** of the application DB and Qdrant collection; **enforcement is infrastructure-level** ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md): "Enforcement is infrastructure-level") |

- **Necessary action (residency):** choosing an EU region only takes effect if the underlying DB/Qdrant infrastructure for that region exists and is provisioned; the field is a placement hint, not a self-enforcing switch. MCP calls should also target region-appropriate endpoints ([09-security-model](../../deep-dive/09-security-model.md) §Data Residency). Ties to Compliance (`72:56`).
- **Validation:** slug collision; the caller must hold `namespaces:admin` (super-scope, Platform Admin only, [09-security-model](../../deep-dive/09-security-model.md) E-29).

## Step 2 · Quotas & Budget

Per-namespace `quotas` (enforced at runtime, not just config — [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) §Quota Enforcement):

| Field | Enforcement point |
|---|---|
| Monthly/daily token cap (`max_tokens_per_day`) | invocation API / async enforcer — reject or alert |
| Cost cap (`max_estimated_cost_per_day_usd`) | alerting-only **or** hard-reject — **mode branch** below |
| Provider rate limit (`max_rpm_share` / `max_tpm_share`) | per-provider, per-namespace token bucket ([19-provider-rate-limiting](../../deep-dive/19-provider-rate-limiting.md)) |
| `max_agents` | provisioning API — reject if count ≥ limit |
| (`max_concurrent_runs`, `max_workflow_runs_per_day`, `max_chatbot_sessions`) | invocation/chatbot API — `429` when at limit |

**Cost-cap enforcement-mode branch** ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) E-18): `hard` (reject new invocations when budget exhausted) vs `alert` (allow but alert the Operator).

**Namespace default budget branch** ([23-per-run-cost-cap](../../deep-dive/23-per-run-cost-cap.md)): a namespace may set a `workflow_budget` default (`max_total_tokens` / `max_estimated_cost_usd` + `on_exceeded`) applied to any workflow agent that omits its own DSL `budget`. A DSL-declared budget overrides the namespace default entirely (no merge).

- **Necessary actions:** the **per-provider rate-limit budget** the namespace draws from must be configured in the provider registry (`requests_per_minute`, `tokens_per_minute`), and the namespace share must be ≤ the platform total ([19-provider-rate-limiting](../../deep-dive/19-provider-rate-limiting.md) §Token Bucket). Quota counters live in the DB (survive restarts) and use atomic updates to avoid TOCTOU races (E-22).
- **Validation:** a namespace rate share above the platform total is meaningless (lower of the two applies); excessively low token caps cause invocations to be deferred/rejected pre-flight ([19-provider-rate-limiting](../../deep-dive/19-provider-rate-limiting.md) §Pre-Flight check, modes `defer`/`warn_only`).

## Step 3 · Defaults

| Field | Branch / values | Effect |
|---|---|---|
| **Default provider** | one Provider Chip (`Claude` worked example) | `settings.default_provider` |
| **Allowed providers** | multi-checkbox (`claude`/`openai`/`gemini`/`grok`) | `settings.allowed_providers` — agents in this namespace may only use these ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md)) |
| **Safety defaults** | tags: `Standard moderation`, `PII redaction on`, PII categories | seeds `settings.pii_detection_categories` and namespace-level safety defaults; per-agent safety (J1 Step 6c) acts within these |

- **Necessary actions:** every allowed provider must be configured platform-side **and** have a verified credential (J6) before agents can actually call it; the default provider must be in the allowed set. PII redaction defaults only take effect if categories are declared — agents inherit these unless overridden.
- **Validation:** default provider not in the allowed set → inconsistent config; an empty allowed-providers set blocks all agent creation in the namespace (no provider passes the J1 Step-3 check).

## Step 4 · Review & Create

Three-section summary (Identity / Quotas & budget / Defaults). Primary: **Create namespace** → `status: active`.

- **What this enables:** the namespace becomes a valid `namespace` value in DSL, API tokens can be scoped to it, and J1/J2/J4/J5/J6 can target it. Cross-namespace agent calls remain prohibited until a Platform Admin adds an explicit allowlist entry ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) §Cross-Namespace Agent Calls).
- **Lifecycle note:** `status` may later be `suspended` (blocks new runs/sessions but doesn't cancel in-flight work) or `deleted`.

---

## Failure-state summary

| Step | Failure | Fix |
|---|---|---|
| Identity | slug collision / not Platform Admin / region infra absent | unique slug; use a super-scoped token; provision the region first |
| Quotas | rate share > platform total / token cap too low | cap share ≤ platform; raise caps or expect `defer`/`429` |
| Defaults | default provider not allow-listed / empty allow-list / providers unconfigured | align default with allow-list; configure + verify providers (J6) |

## Cross-references

- Decisions: [AD-26](../../01-PRD.md), [AD-28](../../01-PRD.md) (provider rate limiting), [AD-32](../../01-PRD.md) (budget).
- Deep-dives: [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (namespace properties, quotas, scoping, E-18/E-22), [23-per-run-cost-cap](../../deep-dive/23-per-run-cost-cap.md) (namespace default budget), [19-provider-rate-limiting](../../deep-dive/19-provider-rate-limiting.md) (per-namespace shares, pre-flight), [35-compliance-governance](../../deep-dive/35-compliance-governance.md) (residency/retention), [09-security-model](../../deep-dive/09-security-model.md) (scopes, data residency).
- Figma (worked example): `211:62` (Identity) · `211:101` (Quotas & Budget) · `212:62` (Defaults) · `212:110` (Review). Parent screen: Admin `57:56`. Related: Compliance `72:56`, Analytics `59:56` (cost vs budget).
