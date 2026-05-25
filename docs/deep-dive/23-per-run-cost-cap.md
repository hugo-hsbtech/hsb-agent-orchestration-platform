# Per-Run Cost Cap and Workflow Budget Enforcement

## Purpose

This document specifies the per-run cost cap mechanism — a safety valve that terminates a single workflow run when its accumulated token or cost consumption exceeds a declared budget. It is distinct from and complementary to:

- **E-18** (`18-enhancements.md`): Per-namespace daily quota (prevents new runs when the daily budget is exhausted).
- **`19-provider-rate-limiting.md`**: Pre-flight TPM estimate check (signals high risk before a run starts).
- This document: **mid-run enforcement** — a kill switch that fires when a specific run actually exceeds its declared cap.

### Why This Is Needed

The existing DSL supports `while` loops with a `max_iterations` guard and `parallel` blocks with `max_concurrency`. Both prevent runaway *structural* expansion. But a workflow can stay within these structural bounds and still consume unbounded tokens if:

- Each `llm_call` step returns near-maximum tokens repeatedly.
- A long `while` loop each iteration calls a 4,096-token LLM call.
- A `for_each` fans out to 10 agents, each running a 3-step workflow with expensive prompts.

Without a cost cap, a single misconfigured run can consume hundreds of dollars of LLM credits before any operator notices.

---

## DSL: Workflow-Level Budget Declaration

### Top-Level `budget` Field

A new optional `budget` field at the top level of workflow agent DSL definitions:

```json
{
  "dsl_version": "4",
  "agent_type": "workflow",
  "name": "research-pipeline",
  "budget": {
    "max_total_tokens": 200000,
    "max_estimated_cost_usd": 5.00,
    "on_exceeded": "cancel | warn_and_continue | cancel_with_partial_output"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `max_total_tokens` | integer | No | Maximum cumulative input + output tokens across all `llm_call` steps in this run. |
| `max_estimated_cost_usd` | number | No | Maximum estimated cost (USD) based on the platform's configured price table. If both fields are declared, whichever is exceeded first triggers the action. |
| `on_exceeded` | enum | Yes (if budget declared) | Action to take when the cap is exceeded. |

At least one of `max_total_tokens` or `max_estimated_cost_usd` must be declared when `budget` is present.

### `on_exceeded` Values

| Value | Behavior |
|---|---|
| `cancel` | Immediately cancel the workflow with `BudgetExceededError`. No further steps run. |
| `warn_and_continue` | Emit a `BudgetExceededWarning` metric and log entry. Continue execution. (Use for soft limits.) |
| `cancel_with_partial_output` | Cancel the workflow but collect all step outputs completed so far and write them to the workflow run record as partial output. Useful for research pipelines where partial results have value. |

---

## Enforcement Mechanism

### Where Enforcement Happens

Budget enforcement runs **at the Temporal workflow level**, not at the activity level. After every `llm_call` activity completes, the workflow code:

1. Reads the activity's output envelope `metadata.tokens` (input + output).
2. Adds to the cumulative counter stored as a Temporal workflow variable.
3. Computes the estimated cost: `tokens.total × price_per_token[provider][model]` (from the platform's price table).
4. Compares against the declared `budget.max_total_tokens` and `budget.max_estimated_cost_usd`.
5. If exceeded, executes the `on_exceeded` action.

This happens **between steps**, not mid-stream. A step that is already executing is allowed to complete (no mid-activity cancellation). The cap applies to the next step dispatch.

### Workflow Variable: Accumulated Budget

The Temporal workflow carries a `budget_state` workflow variable:

```json
{
  "accumulated_tokens": 45312,
  "accumulated_cost_usd": 1.23,
  "last_checked_after_step": "research_step_3",
  "cap_exceeded_at_step": null
}
```

This variable is queryable via Temporal's query API, enabling operators to inspect a running workflow's budget consumption in real time.

### `on_exceeded: cancel_with_partial_output` Contract

When this action fires:

1. The `workflow_runs` record is updated: `status = "cancelled_budget"`, `cancellation_reason = "BudgetExceededError"`.
2. All completed step outputs are preserved in `workflow_step_runs` (no deletion).
3. A synthetic `partial_output` field is populated on the `workflow_runs` record: a JSON object containing all step outputs that completed successfully before the cap was reached.
4. The Temporal workflow is cancelled gracefully (same mechanism as operator cancel, `07-v6` Epic 2 Story 2.3).

---

## Price Table Configuration

The platform maintains a configurable price table used for cost estimation:

```json
{
  "price_table": [
    {
      "provider": "claude",
      "model_name": "claude-3-7-sonnet-20250219",
      "input_price_per_1k_tokens_usd": 0.003,
      "output_price_per_1k_tokens_usd": 0.015
    },
    {
      "provider": "openai",
      "model_name": "gpt-4o",
      "input_price_per_1k_tokens_usd": 0.0025,
      "output_price_per_1k_tokens_usd": 0.010
    }
  ]
}
```

Platform Admins manage this table via:

```
POST /admin/price-table
GET  /admin/price-table
```

Prices are estimates. Actual provider invoices may differ (batch discounts, promotional pricing). The platform makes no billing guarantees based on its estimates — this is a safety mechanism, not a billing system.

### Unknown Model Handling

When a model is not in the price table:

- `max_total_tokens` enforcement continues (no price calculation needed).
- `max_estimated_cost_usd` enforcement is skipped with a warning metric: `agent_platform_unknown_model_price_total{provider, model}`.

---

## Validation Service Rules

When `budget` is declared:

1. **Warning**: If `budget.max_total_tokens` is less than the sum of `max_tokens` declared in all `llm_call` steps (i.e., the cap is guaranteed to be exceeded on the first run even with minimum output), emit: `BUDGET_UNREACHABLE: max_total_tokens (10000) is less than minimum possible token consumption (15000). Workflow will always cancel.`
2. **Warning**: If `budget.max_estimated_cost_usd` is declared but any `llm_call` step's `provider` or `model_config.model_name` is not in the price table, emit: `BUDGET_PRICE_UNKNOWN: Model 'claude-3-8-sonnet' not in price table. Cost cap will not be enforced for this step.`
3. **Error**: If `on_exceeded` is missing when `budget` is present.

---

## Namespace-Level Default Budget

Platform Admins can configure a namespace-level default budget that applies to all workflow runs without an explicit DSL `budget` declaration:

```json
{
  "namespace_defaults": {
    "workflow_budget": {
      "max_total_tokens": 500000,
      "max_estimated_cost_usd": 50.00,
      "on_exceeded": "cancel"
    }
  }
}
```

The namespace default is a safety net for workflows that omit the `budget` field. DSL-declared budgets override namespace defaults entirely (no merging — the DSL declaration is the operative policy).

---

## Prometheus Metrics

| Metric | Labels | Description |
|---|---|---|
| `agent_platform_budget_exceeded_total` | `namespace`, `agent`, `on_exceeded` | Workflow runs that exceeded their budget |
| `agent_platform_run_tokens_total` | `namespace`, `agent`, `provider`, `model` | Cumulative tokens per run (histogram) |
| `agent_platform_run_cost_usd_total` | `namespace`, `agent`, `provider` | Cumulative estimated cost per run (histogram) |
| `agent_platform_unknown_model_price_total` | `provider`, `model` | Calls to models not in the price table |

---

## CLI Integration

The `agent-platform dry-run` command gains a `--estimate-cost` flag (referenced in E-23 / `15-developer-tooling.md`):

```
agent-platform dry-run ./agents/research-pipeline.json \
  --input '{"topic": "quantum computing"}' \
  --mock-dir ./mocks/ \
  --estimate-cost

Budget Estimate (worst-case, based on declared max_tokens):
  Step classify    [llm_call]   Input: ~512 tokens   Output: ≤256 tokens    Cost: ~$0.006
  Step research    [llm_call]   Input: ~1024 tokens  Output: ≤4096 tokens   Cost: ~$0.064
  Step synthesize  [llm_call]   Input: ~4096 tokens  Output: ≤4096 tokens   Cost: ~$0.073
  ─────────────────────────────────────────────────────────────────────────
  Worst-case total:             ~9,600 tokens                               ~$0.143
  Declared budget:              max_total_tokens: 200000   max_cost: $5.00
  Status: ✓ Within declared budget (6.7x headroom on tokens, 35x on cost)
```

When `budget` is not declared in the DSL, `--estimate-cost` still prints the estimate but notes: `No budget declared — namespace default applies (max_total_tokens: 500000, max_cost: $50.00)`.

---

## Interaction with Chatbot Agents

The `budget` field applies to **workflow agents only**. Chatbot agents (`agent_type: chatbot`) do not support per-run cost caps because:

1. Chatbot turns are short, synchronous, and individually low-cost.
2. The "run" concept does not apply to conversational sessions.

For chatbots, cost control is namespace-level daily quotas (E-18) and per-turn `model_config.max_tokens` limits.

---

## Implementation Phase

| Feature | Phase |
|---|---|
| `budget` field in DSL spec | v4 (multi-agent orchestration adds complex cost scenarios) |
| Enforcement in Temporal workflow | v4 (alongside `while`/loop enforcement) |
| Price table API | v6 (alongside cost tracking from E-18) |
| Validation service budget rules | v4 |
| Namespace-level default budget | v6 |
| `--estimate-cost` CLI flag | v3 (alongside `dry-run` specification E-23) |

---

## What to Read Next

The per-run cost cap works alongside the namespace-level daily quota and rate limiting. Read together:

→ **[05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md)** — Loops, fan-out, and parallel execution — the workflow constructs that create the cost risk this document guards against.

Or explore the cost control stack:
- [18-enhancements.md — E-18](./18-enhancements.md) — Namespace-level daily token quota: the outer limit that complements the per-run cap
- [19-provider-rate-limiting.md](./19-provider-rate-limiting.md) — Rate limiting: the third layer of provider cost protection (token bucket + pre-flight check)
- [15-developer-tooling.md](./15-developer-tooling.md) — The `dry-run --estimate-cost` flag for pre-deployment budget validation
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace-level default budget configuration

---

## Related Documents

- [14-dsl-spec.md](./14-dsl-spec.md) — DSL field specification (budget block)
- [18-enhancements.md](./18-enhancements.md) — E-18 (namespace token quota), E-23 (dry-run spec)
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace quota enforcement
- [15-developer-tooling.md](./15-developer-tooling.md) — `dry-run --estimate-cost` flag
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Prometheus metrics (Story 4.2)
- [05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md) — Loops and fan-out that create cost risk
