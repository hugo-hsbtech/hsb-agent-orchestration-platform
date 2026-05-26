# J1 — Agent Creation

> **Entry:** `New agent` on Agent Catalogue (`23:3`, button `25:30`). **Figma worked example:** create top-level chatbot `acme-concierge` (router, draft `v0.1`) routing to `payments`/`technical`/`knowledge` specialists — frames `106:221` → `106:333`. **Grounds:** [AD-19](../../01-PRD.md) (chatbot type), [AD-20](../../01-PRD.md)/[AD-33](../../01-PRD.md) (routing), [AD-30](../../01-PRD.md) (memory), [14-dsl-spec](../../deep-dive/14-dsl-spec.md), [v5-chatbot-layer](../../roadmap/v5-chatbot-layer.md), [21-conversation-memory](../../deep-dive/21-conversation-memory.md), [09-security-model](../../deep-dive/09-security-model.md) (safety/PII), [23-per-run-cost-cap](../../deep-dive/23-per-run-cost-cap.md).

The agent wizard is a **unified wizard with a diverging tail**: a shared spine (Type → Identity → Model), then it diverges by agent type. The chatbot tail (Routing → Knowledge & Memory → Safety → Review) **is the chatbot config surface** — there is no separate chatbot editor; later edits re-enter this same stepped surface. The workflow tail (Triggers → open Visual Builder) hands off to the existing Visual Workflow Builder (`35:34`).

---

## Step 0 — Shared spine

### Step 1 · Type

| Option | DSL effect | Leads to |
|---|---|---|
| **Chatbot** | `agent_type: chatbot` (DSL v5) — synchronous streaming; no `workflow.steps` block | Chatbot tail (Steps 4c–7c) |
| **Workflow** | `agent_type: workflow` — async, Temporal-backed | Workflow tail (Step 4w) |

Chatbot vs workflow is the **fundamental fork**. A chatbot's "run" is the long-lived conversation, not a Temporal workflow per turn ([AD-19](../../01-PRD.md)); a workflow is an async DAG of steps. The chatbot card notes its sub-modes (router vs specialist), chosen in the Routing step.

### Step 2 · Identity

Fields: `name` (unique per namespace, max 128 chars), `namespace` (`acme` prefilled), `description` (required), `owner`, `tags[]` (flat string array, indexed for filter).

- **Necessary action:** the namespace must already exist (J3) and the caller's token must carry `agents:write` for it ([09-security-model](../../deep-dive/09-security-model.md) §Authorization Scope Contract E-29).
- **Validation:** name collision within the namespace is rejected (`UNIQUE (namespace_id, name)`, [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md)); empty `description`/`name` rejected; `max_agents` namespace quota enforced at the provisioning API (reject if count ≥ limit).

### Step 3 · Model & Provider

| Field | Branch / values |
|---|---|
| Provider | `claude` / `openai` / `gemini` / `grok` (Provider Chip) — must be in the namespace's `allowed_providers` |
| Model | provider-specific `model_name` dropdown |
| Temperature | 0.0–2.0 (router chatbots typically 0.0–0.3 for deterministic classification) |
| Max tokens | per-turn / per-call cap |
| System prompt | for a **router** chatbot this is the **classifier prompt** (must emit `{intent, confidence, reasoning}`); for a specialist or workflow it's the domain instruction |
| Fallback provider *(optional)* | `model_config.fallback_provider` — backup provider + `trigger_on: [ProviderUnavailableError, RateLimitError]` |

- **Necessary action:** the chosen `provider` (and any `fallback_provider.provider`) must be configured platform-side and in the namespace allow-list ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) `settings.allowed_providers`); a verified provider credential must exist (J6). The validation service statically checks provider configuration (E-06).
- **Validation:** provider not in namespace allow-list → error; `grok` emits a `DEPRECATED_PROVIDER` warning ([14-dsl-spec](../../deep-dive/14-dsl-spec.md) §Validation Error Reference); a router whose `outputs` omit `confidence: number` fails the routing-activity contract ([AD-33](../../01-PRD.md)).

---

## Chatbot tail

### Step 4c · Routing & Specialists

The first branch is **router vs single specialist**.

| Sub-mode | What it sets | Required prerequisites |
|---|---|---|
| **Router** (`routing: true`) | `routing.specialists[]`, `routing.default_specialist`, `routing.confidence_threshold`, `routing.low_confidence_action`, `routing.static_fallback_message` | every named specialist must **already exist and be promoted to `production`** (a router routes to `latest`); a `default_specialist` must exist |
| **Single specialist** (no `routing` block) | a domain chatbot reachable directly or via a router | none beyond the spine |

**Routing-strategy branch** (router only):

| Strategy | Behaviour | Necessary action |
|---|---|---|
| **LLM classifier** (default, [AD-33](../../01-PRD.md)) | the router's own LLM call classifies each turn → `{intent, confidence}`; there is **no secondary classifier** | the classifier prompt (Step 3) must produce a `confidence` score; route fires per turn (not once per conversation, [v5](../../roadmap/v5-chatbot-layer.md) Epic 2) |
| **Rule-based** *(design assumption)* | deterministic keyword/regex → specialist map | **Not specified in the platform docs.** The DSL `routing` block models only LLM-confidence routing ([14-dsl-spec](../../deep-dive/14-dsl-spec.md) §Routing chatbot). Mark any rule-based UI as a design assumption; document it as "maps to a future routing strategy" rather than an existing field. |

**Confidence-threshold branch** — `confidence_threshold` (default `0.5`) slider plus `low_confidence_action`:

| Action | Behaviour |
|---|---|
| `ask_clarification` | sends a clarifying question, awaits next turn |
| `route_to_default` | immediately delegates to `default_specialist` |

UX confidence bands ([13-chatbot-ux-edge-cases](../../deep-dive/13-chatbot-ux-edge-cases.md) §7): high (>0.8) route directly · medium (0.5–0.8) route with disclaimer · low (<0.5) apply `low_confidence_action`.

- **Necessary actions (router):** specialists must be provisioned **and promoted to `production`** ([AD-29](../../01-PRD.md), [20-agent-version-promotion](../../deep-dive/20-agent-version-promotion.md) — a router resolves `latest`, which is the production-state version); circuit breakers exist per specialist ([AD-31](../../01-PRD.md)) so a flapping specialist falls back to `default_specialist`.
- **Validation:** a `routing.specialists` entry that doesn't resolve to an existing agent → error; missing `default_specialist` removes the low-confidence safety net; the Figma worked example shows three Production-state specialists (`payments`/`technical`/`knowledge`) at threshold `0.72`.

### Step 5c · Knowledge & Memory

**KB attach branch** (`knowledge_base` block, optional):

| Option | Effect |
|---|---|
| **Attach KB collection** | `knowledge_base.collection`, `filters`, `retrieval_strategy` (`semantic`/`hybrid`), `top_k` — scopes the specialist's lookups |
| **No KB** | specialist answers from the model only |

**Memory tiers** ([AD-30](../../01-PRD.md), [21-conversation-memory](../../deep-dive/21-conversation-memory.md)):

| Tier | Toggle / config | On = requires |
|---|---|---|
| **Working** (always on) | `context_window.strategy` = `sliding` / `summarize` / `semantic`, `max_tokens` | `summarize` needs a `summary_agent` (must exist) |
| **Episodic** | `episodic_memory.enabled`, `lookback_days` (30d default), `top_k`, `inject_on` | a stable `user_id` in conversation context; an episodic Qdrant collection per namespace; anonymous sessions skip episodic |
| **Semantic** | `semantic_memory.enabled`, `auto_inject`, `inject_fields[]` | `user_profiles` table + `user_profile_lookup`/`user_profile_update` tools; write needs `user_profile:write` scope (not granted by default) |

- **Necessary actions:** the KB collection must already be **ingested and indexed** (J4) — attaching an empty/unknown collection is valid config but yields no hits; the KB tool is the registered Qdrant capability ([AD-18](../../01-PRD.md)); filters are validated against the Qdrant schema at provisioning ([v5](../../roadmap/v5-chatbot-layer.md) Story 3.2). Episodic retrieval rides the KB circuit breaker — if `kb_lookup` is OPEN ([AD-31](../../01-PRD.md), Health screen `65:56`), episodic injection is skipped (degradation banner, Chat `54:56`).
- **Validation:** Qdrant filter that references a non-existent collection/field → provisioning error; `summarize` strategy without `summary_agent` → error.

### Step 6c · Safety

| Control | Branch / values | Necessary action |
|---|---|---|
| Moderation level | Off / **Standard** / Strict *(level labels are a UI design assumption — the docs specify PII categories per namespace, not named tiers)* | safety check runs at the routing layer before the LLM call ([13-chatbot-ux-edge-cases](../../deep-dive/13-chatbot-ux-edge-cases.md) §13) |
| PII detection/redaction | toggle + categories (`CREDIT_CARD`, `SSN`, `EMAIL`, `PHONE`, `IBAN`, `FULL_NAME`) | namespace `settings.pii_detection_categories` drives detection; two-track persistence stores `raw_message` (encrypted) + redacted `message` ([09-security-model](../../deep-dive/09-security-model.md) E-16) |
| Blocked topics | list | refusal pattern: single generic refusal then human-handoff offer |
| Refusal message | copy | shown on flagged content |
| Human handoff *(optional)* | `human_handoff.strategy` = `message` / `webhook` / `ticket` | `webhook` needs a reachable URL returning 2xx (retries 3×, falls back to `message`); `ticket` is reserved/falls back to `message` until ticketing MCP exists ([v5](../../roadmap/v5-chatbot-layer.md) Epic 7) |
| Degraded message *(optional)* | `degraded_message` | shown when this specialist's circuit breaker is OPEN ([22-chatbot-degradation](../../deep-dive/22-chatbot-degradation.md)) |

- **Validation:** redaction depends on the namespace PII policy being active; if disabled at namespace level, the per-agent toggle has nothing to act on. `webhook` handoff with a dead URL → `escalation_outcome: webhook_failed_fallback` at runtime.

### Step 7c · Review & Create

Two-column config summary + Draft `v0.1` rail. Primary action: **Create as draft** → lands in Catalogue with a `Draft` badge; secondary `Test in Playground`.

- **Necessary action:** create writes a **`draft`** version — it does **not** become `latest` ([AD-29](../../01-PRD.md)). To serve real traffic it must be promoted draft → staging → canary → production via Version Promotion (`48:40`). A router created before its specialists are in production will fail to resolve them at runtime.
- **Validation:** the validation service collects **all** errors at once ([AD-13](../../01-PRD.md)) — provider, references, schema, routing-contract — and blocks create until fixed.

---

## Workflow tail

### Step 4w · Triggers → Open in Visual Builder

A single frame (`106:333`): a trigger-type picker (Manual / Cron / Webhook / Event — conceptually journey [J5](./trigger-creation.md)) then `Create draft & open builder` → handoff to the Visual Workflow Builder (`35:34`).

**Trigger branch:** Manual (invoke via API) is the default; Cron/Webhook/Event configure a first-class trigger resource (full config in [J5](./trigger-creation.md)).

**The complete workflow body (Visual Builder step types).** The Figma shows only the wizard tail; the actual workflow is authored in the Builder as a node graph + DSL split-view ([14-dsl-spec](../../deep-dive/14-dsl-spec.md)). Every step type the builder offers:

| Step type | Purpose | Key fields / branches |
|---|---|---|
| `llm_call` | invoke the model (may call tools/MCPs) | `prompt_template`, `tools[]`, `mcps[]`, `model_config_override`; multi-turn tool loops within one step |
| `tool_call` | call a registered tool directly | `tool` (must be registered); inputs validated vs tool `input_schema` |
| `mcp_call` | call an MCP capability | `mcp` + `capability` (must be in the cached manifest); auth injected from context |
| `transform` | deterministic data mapping (no LLM) | `mapping` with `_each` directive; no retry by default |
| `fetch` ⚠ | ad-hoc HTTP/DB call (soft-deprecated v3+) | `request` block; `{{credentials.*}}` refs; emits a warning, prefer `tool_call`/`mcp_call` |
| `noop` | convergence point / placeholder | `next` only |
| `agent_call` | invoke a sub-agent | `agent` (or `catalogue://slug`), `version`, `mode` = `wait` / `fire_and_forget` (+ `on_orphan_failure`) |
| `parallel` | concurrent branches | `failure_policy` = `fail_fast` / `collect_all`; `max_concurrency` **required** |
| `for_each` | fan-out over an array | `over`, `item_variable`, `max_concurrency` **required**, `item_failure_policy` = `fail_fast` / `skip` / `collect_errors` |
| `branch` | 2-way condition | `condition`, `then`, `else` |
| `switch` | multi-way match | `value`, `cases[]`, `default` |
| `while` | pre-condition loop | `condition`, `max_iterations` (required, default 10), `loop_state_update` |
| `until` | post-condition loop (body runs ≥1) | same as `while`, condition evaluated after body |

Optional top-level `budget` block ([AD-32](../../01-PRD.md), [23-per-run-cost-cap](../../deep-dive/23-per-run-cost-cap.md)): `max_total_tokens` / `max_estimated_cost_usd` + `on_exceeded` = `cancel` / `warn_and_continue` / `cancel_with_partial_output`. (Chatbots do **not** support per-run budgets — cost control is namespace daily quota + per-turn `max_tokens`.)

- **Necessary actions:** every `tool_call.tool` must be registered ([AD-11](../../01-PRD.md)); every `mcp_call.mcp`+`capability` must be a registered MCP with the capability present in its cached manifest (J2); `agent_call` targets must exist (cross-namespace needs an allowlist entry, [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md)); `fetch`/`mcp_call` credential refs must exist in the credential store (J6).
- **Validation** (collect-all, [AD-13](../../01-PRD.md) / Validation screen `41:36`): `UNKNOWN_TOOL`, `UNRESOLVABLE_REFERENCE` (output not declared), `MISSING_MAX_CONCURRENCY` (parallel/for_each), `CYCLE_DETECTED` (agent_call graph, depth limit 10), `BUDGET_UNREACHABLE`/`BUDGET_PRICE_UNKNOWN`; `fetch` and missing `mcp_call` timeouts emit warnings.

---

## Failure-state summary

| Where | Failure | Fix |
|---|---|---|
| Identity | name collision / quota hit | rename / raise `max_agents` (J3) |
| Model | provider not allow-listed / no credential | add provider to namespace, verify a key (J6) |
| Routing | specialist missing or not in production | provision + promote specialist first ([AD-29](../../01-PRD.md)) |
| Knowledge | KB collection empty/unknown / filter invalid | ingest source first (J4); fix Qdrant filter |
| Safety | PII policy off at namespace / dead handoff webhook | enable namespace PII categories; point webhook at a live 2xx endpoint |
| Workflow body | unknown tool/MCP/agent, missing concurrency, cycle | register the dependency; declare `max_concurrency`; break the cycle |

## Cross-references

- Decisions: [AD-13](../../01-PRD.md), [AD-19](../../01-PRD.md), [AD-20](../../01-PRD.md), [AD-29](../../01-PRD.md), [AD-30](../../01-PRD.md), [AD-31](../../01-PRD.md), [AD-32](../../01-PRD.md), [AD-33](../../01-PRD.md).
- Deep-dives: [14-dsl-spec](../../deep-dive/14-dsl-spec.md), [v5-chatbot-layer](../../roadmap/v5-chatbot-layer.md), [21-conversation-memory](../../deep-dive/21-conversation-memory.md), [13-chatbot-ux-edge-cases](../../deep-dive/13-chatbot-ux-edge-cases.md), [09-security-model](../../deep-dive/09-security-model.md), [20-agent-version-promotion](../../deep-dive/20-agent-version-promotion.md), [23-per-run-cost-cap](../../deep-dive/23-per-run-cost-cap.md).
- Figma (worked example, chatbot spine + workflow tail): `106:221` (Type) · `106:237` (Identity) · `106:253` (Model) · `106:269` (Routing) · `106:285` (Knowledge & Memory) · `106:301` (Safety) · `106:317` (Review) · `106:333` (Workflow tail). Builder handoff: `35:34`. Related screens: Validation `41:36`, Playground `42:36`, Promotion `48:40`, Chat `54:56`, Memory `74:56`.
