# DSL Specification

## Purpose

This document is the canonical reference for the Agent Platform DSL (Domain Specific Language). It specifies the complete structure, semantics, and valid values for every field in every version of the DSL, with annotated examples for all step types through v4.

This document is intended for:
- **Platform Engineers** building the validation service and workflow interpreter.
- **Agent Builders** writing DSL documents.
- **The CLI tooling** (`agent-platform validate`, `agent-platform dry-run`).

---

## DSL Versioning

The DSL evolves additively across platform versions. Each version adds fields and step types; it never removes or renames them. A v1 document is valid on a v6 platform.

| DSL Version | Introduced In | New Constructs |
|---|---|---|
| **v1** | v1 platform | `name`, `description`, `system_prompt`, `user_prompt_template`, `provider`, `model_config` |
| **v2** | v2 platform | `workflow.steps`, step types: `llm_call`, `transform`, `fetch`, `noop`, `retry`, data flow `inputs`/`outputs` |
| **v3** | v3 platform | Step types: `tool_call`, `mcp_call`; `llm_call.tools`, `llm_call.mcps` |
| **v4** | v4 platform | Step types: `agent_call`, `parallel`, `for_each`, `branch`, `switch`, `while`, `until`; inline mapping expressions |
| **v5** | v5 platform | `agent_type: chatbot`; chatbot-specific fields |

---

## Top-Level Document Structure

```json
{
  "dsl_version": "4",
  "agent_type": "workflow | chatbot",
  "name": "string (required, unique per namespace)",
  "description": "string (required)",
  "namespace": "string (required, see 16-multi-tenancy.md)",
  "system_prompt": "string (required)",
  "user_prompt_template": "string with {{variable}} placeholders",
  "provider": "claude | openai | gemini | grok | other",
  "model_config": { ... },
  "workflow": { ... }
}
```

### Field Reference: Top Level

| Field | Type | Required | Description |
|---|---|---|---|
| `dsl_version` | string | Yes | DSL version this document conforms to. Must be a known version string. |
| `agent_type` | enum | Yes | `workflow` (async, Temporal-backed) or `chatbot` (sync, streaming). |
| `name` | string | Yes | Human-readable identifier. Unique within a namespace. Max 128 chars. |
| `description` | string | Yes | Purpose of this agent. Shown in registry listings. |
| `namespace` | string | Yes | Tenant/namespace identifier. See `16-multi-tenancy.md`. |
| `tags` | array | No | Flat string array for categorisation. e.g. `["billing", "team-payments"]`. Indexed; supports API and CLI filtering. See E-37. |
| `system_prompt` | string | Yes | The LLM's system instruction. Not interpolated at runtime. |
| `user_prompt_template` | string | No | Template for the user turn. Supports `{{input.field}}` interpolation. |
| `provider` | enum | Yes | LLM provider. Must be configured in the platform. |
| `model_config` | object | Yes | Provider-specific model parameters. See below. |
| `input_schema` | object | No (recommended) | Declares the expected shape of the workflow invocation payload. Used by the validation service to verify `$.workflow.input.*` references and by the invocation API to validate request bodies at runtime. See §Input Schema below. |
| `budget` | object | No | Per-run token/cost cap. See `23-per-run-cost-cap.md`. |
| `workflow` | object | Conditional | Required when `agent_type: workflow`. Not used for `chatbot` type at the step level. |

### `model_config`

```json
{
  "model_name": "claude-3-7-sonnet-20250219",
  "temperature": 0.7,
  "max_tokens": 4096,
  "top_p": 1.0,
  "stop_sequences": [],
  "fallback_provider": {
    "provider": "openai",
    "model_name": "gpt-4o",
    "trigger_on": ["ProviderUnavailableError", "RateLimitError"],
    "max_attempts": 1
  }
}
```

| Field | Type | Required | Notes |
|---|---|---|---|
| `model_name` | string | Yes | Provider-specific model identifier. |
| `temperature` | float | No | Default: `1.0`. Range: `0.0–2.0`. |
| `max_tokens` | integer | Yes | Maximum tokens to generate. |
| `top_p` | float | No | Default: `1.0`. |
| `stop_sequences` | array | No | Default: `[]`. |
| `fallback_provider` | object | No | Optional fallback when primary provider fails. See §Provider Failover below (E-27). |

### `fallback_provider` (E-27)

Declares a backup provider to use when the primary fails. Defined inside `model_config`.

| Field | Type | Required | Notes |
|---|---|---|---|
| `provider` | enum | Yes | Fallback provider slug. Must be configured in the platform. |
| `model_name` | string | Yes | Provider-specific model identifier on the fallback. |
| `trigger_on` | array | Yes | Error class names that activate failover: `ProviderUnavailableError`, `RateLimitError`. |
| `max_attempts` | integer | No | Default: `1`. Failover is attempted this many times before the step fails. |

**Failover contract**:
- Transient errors retry on the primary first (per the step `retry` policy). Only after exhausting primary retries does the fallback engage.
- The step run record stores `provider_used` to record which provider actually served the request.
- Mid-stream failover (provider drops connection during streaming) is **not** supported. Failover applies only at the activity invocation boundary.
- The validation service checks that `fallback_provider.provider` is configured in the platform (same static check as the primary, E-06).

### `input_schema` (E-04)

Declares the expected shape of the workflow's invocation payload. Optional but recommended.

```json
{
  "input_schema": {
    "user_question": { "type": "string", "required": true },
    "customer_id":   { "type": "string", "required": false },
    "language":      { "type": "string", "required": false, "default": "en" }
  }
}
```

| Sub-field | Type | Notes |
|---|---|---|
| `type` | enum | `string`, `number`, `boolean`, `object`, `array`. |
| `required` | boolean | Default: `false`. |
| `default` | any | Injected when field is absent from the invocation payload. |

**Validation behaviour**: The validation service uses `input_schema` to verify all `$.workflow.input.*` references point to declared fields. The invocation API validates the request body against it at runtime, rejecting missing required fields. For `agent_call` steps, the validation service cross-references the called agent's `input_schema` against the calling step's `inputs` mapping.

---

## Canonical Step Output Envelope

Every step — regardless of type — produces output wrapped in a canonical envelope. This is the shape referenced by downstream data flow expressions (`$.steps.step_id.output.*`).

```json
{
  "output": { ... },
  "metadata": {
    "step_id": "string",
    "step_type": "llm_call | tool_call | ...",
    "status": "succeeded | failed | skipped",
    "started_at": "ISO 8601",
    "completed_at": "ISO 8601",
    "attempt_number": 1,
    "provider": "claude | null",
    "tokens": {
      "input": 512,
      "output": 128,
      "total": 640
    },
    "error": null
  }
}
```

The `output` object's shape is **step-type-specific** and documented per step type below. The `metadata` block is always present and always has the same shape.

---

## Data Flow Expression Syntax

Step `inputs` fields use dot-path expressions to reference values from the workflow context.

| Prefix | Resolves To | Example |
|---|---|---|
| `$.workflow.input.*` | The workflow's initial invocation payload | `$.workflow.input.user_question` |
| `$.steps.<id>.output.*` | A named prior step's output field | `$.steps.fetch_docs.output.documents` |
| `$.steps.<id>.metadata.*` | A named prior step's metadata | `$.steps.classify.metadata.tokens.total` |
| `$.env.*` | Workflow-level context (namespace, run_id, etc.) | `$.env.workflow_run_id` |

**Literal values** — strings, numbers, booleans — can be used directly as input values without the `$` prefix.

**Transformation operators** (available in v4 inline mapping):

| Operator | Syntax | Example |
|---|---|---|
| Field selection | `$.steps.X.output.field` | Direct path traversal |
| Array index | `$.steps.X.output.items[0]` | First element |
| Default value | `$.steps.X.output.field \| "default"` | Fallback if null/missing |
| String concat | `"prefix_" + $.steps.X.output.id` | Concatenation |
| Conditional | `$.steps.X.output.flag ? "yes" : "no"` | Ternary |

**Condition expressions** (used in `branch.condition`, `while.condition`, `until.condition`):

| Operator | Example |
|---|---|
| Equality | `$.steps.classify.output.intent == "billing"` |
| Inequality | `$.steps.classify.output.score != 0` |
| Comparison | `$.steps.loop_body.output.quality_score >= 0.8` |
| Null check | `$.steps.fetch.output.result != null` |
| Boolean AND | `$.steps.A.output.ok == true && $.steps.B.output.ok == true` |
| Boolean OR | `$.steps.A.output.label == "yes" \|\| $.steps.B.output.label == "yes"` |
| Array length | `$.steps.search.output.results.length > 0` |

**Constraint**: Condition expressions are a closed, sandboxed language. They have no side effects and no access to system functions. The validation service rejects expressions that use constructs outside this set.

### Expression Language Formal Specification (E-01)

Both condition expressions and mapping expressions share a single unified grammar. The following EBNF defines the complete expression language:

```ebnf
expr        ::= ternary
ternary     ::= or_expr ( "?" expr ":" expr )?
or_expr     ::= and_expr ( "||" and_expr )*
and_expr    ::= compare ( "&&" compare )*
compare     ::= add_expr ( ( "==" | "!=" | ">" | ">=" | "<" | "<=" ) add_expr )?
add_expr    ::= primary ( "+" primary )*
primary     ::= path | literal | "(" expr ")"
path        ::= "$" "." segment ( "." segment )* ( ".length" )?
segment     ::= identifier | identifier "[" integer "]"
literal     ::= string | number | boolean | "null"
string      ::= '"' [^"]* '"'
number      ::= "-"? [0-9]+ ( "." [0-9]+ )?
boolean     ::= "true" | "false"
identifier  ::= [a-zA-Z_][a-zA-Z0-9_]*
integer     ::= [0-9]+
```

**Type system**: Three scalar types (`string`, `number`, `boolean`), plus `null`, `array`, and `object`.
- `==` and `!=` use **strict equality** — no implicit type coercion. `"1" == 1` is `false`.
- Arithmetic `+` is defined only for `number + number` and `string + string` (concatenation). Mixed types produce a `TypeError` at validation time.
- Comparison operators (`>`, `>=`, `<`, `<=`) are defined only for `number`. Applying to other types produces a `TypeError`.

**Missing-field semantics**: `$.steps.X.output.field` when `field` does not exist in the output produces `null`. The `|` default operator (`$.steps.X.output.field | "default"`) returns the right-hand side when the left-hand side evaluates to `null`.

**Short-circuit evaluation**: `&&` short-circuits on the first `false` operand; `||` short-circuits on the first `true` operand.

**`_each` directive** (mapping expressions only): Used inside a `transform` step `mapping` block to apply a mapping spec to each element of an array. `$.item.*` is bound to the current element within the `_each` body. `_each` is not available in condition expressions.

---

## Workflow Structure

```json
{
  "workflow": {
    "entry": "step_id_of_first_step",
    "steps": [
      { ... },
      { ... }
    ]
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `entry` | string | Yes | The `id` of the first step to execute. |
| `steps` | array | Yes | Ordered list of step definitions. All steps must be reachable from `entry`. |

The validation service checks:
- `entry` references a valid step id.
- No unreachable steps (all steps reachable from `entry` via `next` chains).
- No cycles (except inside `while`/`until` bodies, which are explicitly loop constructs).
- All `$.steps.<id>.*` references point to steps that precede the referencing step in execution order.

---

## Retry Policy

Every step accepts an optional `retry` block and an optional `timeout_seconds` field. Defaults apply when omitted.

```json
{
  "timeout_seconds": 30,
  "retry": {
    "max_attempts": 3,
    "initial_interval_seconds": 1,
    "backoff_coefficient": 2.0,
    "max_interval_seconds": 60,
    "non_retryable_errors": ["ValidationError", "AuthenticationError"],
    "rate_limit_max_attempts": 2,
    "rate_limit_honor_retry_after": true
  }
}
```

| Field | Type | Default | Description |
|---|---|---|---|
| `timeout_seconds` | integer | See table | Maximum wall-clock time for a single attempt. Temporal cancels the activity if exceeded. |
| `max_attempts` | integer | `3` | Total attempts including the first. `1` means no retries. |
| `initial_interval_seconds` | float | `1.0` | Wait before first retry. |
| `backoff_coefficient` | float | `2.0` | Multiplier applied to interval each retry. |
| `max_interval_seconds` | float | `60.0` | Cap on wait interval. |
| `non_retryable_errors` | array | `[]` | Error class names that fail the step immediately without retry. |
| `rate_limit_max_attempts` | integer | `3` | Attempts specifically for `RateLimitError` (E-30). Separate from `max_attempts`. |
| `rate_limit_honor_retry_after` | boolean | `true` | When `true`, the platform waits the duration from the provider's `Retry-After` header before retrying. |

**Default `timeout_seconds` by step type** (E-07):

| Step Type | Default timeout | Notes |
|---|---|---|
| `llm_call` | `120` | Covers streaming + multi-turn tool loops. |
| `tool_call` | `30` | Most tools complete in <5s; 30s covers slow KB lookups. |
| `mcp_call` | `60` | MCPs vary widely; 60s is conservative. The validation service **warns** when `mcp_call` or `agent_call (wait)` steps are missing an explicit `timeout_seconds`. |
| `transform` | `5` | Deterministic; should never take long. |
| `fetch` | `30` | HTTP network timeout. |
| `agent_call (wait)` | `600` | Child workflows may be long-running. |
| `agent_call (fire_and_forget)` | `10` | Only covers the dispatch; child runs independently. |
| `parallel` | `600` | Covers slowest branch. |
| `for_each` | `600` | Covers slowest item. |

**Default by step type**:

| Step Type | Default `max_attempts` | Notes |
|---|---|---|
| `llm_call` | `3` | Transient provider errors are retryable. |
| `tool_call` | `3` | Depends on tool implementation; tools declare non-retryable errors. |
| `mcp_call` | `3` | MCP transport errors are retryable; MCP logic errors are not. |
| `transform` | `1` | Deterministic; retrying is pointless unless there's a bug. |
| `fetch` | `3` | Network errors retryable. |
| `agent_call` | `1` | Child workflow manages its own retries. |
| `parallel` | `1` | Individual children declare their own retry policies. |
| `for_each` | `1` | Per-item step declares its own retry policy. |
| `branch` / `switch` | `1` | Routing decisions are deterministic. |
| `while` / `until` | `1` | Loop control is deterministic; body step declares its own policy. |

---

## Step Types (Complete Reference)

### `llm_call`

Invokes the Agent Core with a prepared prompt bundle. The model may call tools/MCPs if declared.

```json
{
  "id": "summarize",
  "type": "llm_call",
  "inputs": {
    "user_prompt": "$.workflow.input.raw_text",
    "context_docs": "$.steps.fetch_context.output.documents"
  },
  "prompt_template": "Summarize the following text concisely:\n\n{{inputs.user_prompt}}\n\nContext:\n{{inputs.context_docs}}",
  "tools": ["knowledge_base_lookup", "web_search"],
  "mcps": ["linear", "jira"],
  "model_config_override": {
    "temperature": 0.3
  },
  "outputs": {
    "summary": "string",
    "tool_calls_made": "array"
  },
  "next": "translate",
  "retry": { "max_attempts": 3 }
}
```

**Output envelope `output` shape**:

```json
{
  "text": "The generated response text as a string.",
  "tool_calls": [
    {
      "tool_name": "knowledge_base_lookup",
      "inputs": { "query": "..." },
      "output": { ... },
      "started_at": "ISO 8601",
      "completed_at": "ISO 8601"
    }
  ],
  "finish_reason": "stop | tool_use | max_tokens | error",
  "declared_outputs": {
    "summary": "...",
    "tool_calls_made": [...]
  }
}
```

**Notes**:
- `prompt_template` uses `{{inputs.*}}` to reference the step's resolved inputs (not raw `$.steps.*` paths). This is distinct from the top-level `user_prompt_template`.
- `tools` and `mcps` are names from the registry. The model may call them zero or many times within this step (multi-turn tool-use loops are fully supported within a single step).
- `model_config_override` overrides specific fields from the agent's top-level `model_config` for this step only.
- `declared_outputs` in the envelope contains only the fields named in the step's `outputs` block, extracted from the model's response. The raw `text` is always available regardless.

---

### `transform`

Applies declarative data mapping. No LLM call. No side effects. Deterministic.

```json
{
  "id": "reshape_results",
  "type": "transform",
  "inputs": {
    "raw_items": "$.steps.fetch_data.output.items",
    "label": "$.steps.classify.output.label"
  },
  "mapping": {
    "tagged_items": {
      "_each": "$.inputs.raw_items",
      "id": "$.item.id",
      "name": "$.item.title",
      "category": "$.inputs.label"
    },
    "count": "$.inputs.raw_items.length",
    "processed_by": "$.env.workflow_run_id"
  },
  "outputs": {
    "tagged_items": "array",
    "count": "integer",
    "processed_by": "string"
  },
  "next": "summarize"
}
```

**Output envelope `output` shape**:

```json
{
  "tagged_items": [...],
  "count": 5,
  "processed_by": "wfr-abc123"
}
```

**Notes**:
- `_each` is a mapping directive that applies the sibling mapping spec to each element in the referenced array, with `$.item.*` bound to the current element.
- `transform` has no `retry` by default (deterministic). Validation errors fail immediately as `non_retryable`.

---

### `fetch` ⚠️ Soft-Deprecated (E-03)

> **Decision (E-03 / AD-28 Option A)**: `fetch` is retained as a first-class step type for ad-hoc HTTP calls not worth registering in the Tool Registry. It is **soft-deprecated** from v3 onward: existing `fetch` steps continue to work, but new agent definitions should prefer `tool_call` or `mcp_call` for any named integration. A future version may promote `fetch` to `generic_http_tool` and formally remove this step type — `agent-platform migrate` will handle that transition automatically. The validation service emits a `warning` (not an error) for `fetch` steps in DSL v3+.

Makes an HTTP or DB call. Retained for ad-hoc HTTP calls not registered in the Tool Registry.

```json
{
  "id": "get_pricing",
  "type": "fetch",
  "inputs": {
    "product_id": "$.workflow.input.product_id"
  },
  "request": {
    "url": "https://api.internal/pricing/{{inputs.product_id}}",
    "method": "GET",
    "headers": {
      "Authorization": "Bearer {{credentials.pricing_api_key}}"
    },
    "timeout_seconds": 10
  },
  "outputs": {
    "price": "number",
    "currency": "string"
  },
  "next": "summarize",
  "retry": { "max_attempts": 3 }
}
```

**Output envelope `output` shape**:

```json
{
  "status_code": 200,
  "body": { ... },
  "declared_outputs": {
    "price": 29.99,
    "currency": "USD"
  }
}
```

**Notes**:
- `{{credentials.*}}` references are resolved through the `CredentialsProvider` at runtime. Raw values must never appear in the DSL.
- The validation service checks that `credentials.*` references exist in the configured credential store.

---

### `noop`

No-op step. Used as a convergence point after branches or as a named placeholder during development.

```json
{
  "id": "converge",
  "type": "noop",
  "next": "finalize"
}
```

**Output envelope `output` shape**: `{}`

---

### `tool_call`

Invokes a named tool from the Tool Registry directly, without going through the LLM.

```json
{
  "id": "search_kb",
  "type": "tool_call",
  "tool": "knowledge_base_lookup",
  "inputs": {
    "query": "$.workflow.input.question",
    "filters": {
      "collection": "product_docs",
      "metadata.language": "en"
    },
    "strategy": "hybrid",
    "top_k": 5
  },
  "outputs": {
    "documents": "array",
    "scores": "array"
  },
  "next": "generate_answer",
  "retry": { "max_attempts": 2, "initial_interval_seconds": 0.5 }
}
```

**Output envelope `output` shape**:

```json
{
  "tool_name": "knowledge_base_lookup",
  "raw_output": { ... },
  "declared_outputs": {
    "documents": [...],
    "scores": [...]
  }
}
```

**Notes**:
- `tool` must be a name registered in the Tool Registry. Unknown names fail validation at provisioning time.
- `inputs` are validated against the tool's declared `input_schema` before the handler runs.
- Tools declare `idempotent: true` in their registry entry (E-12). When `true`, the platform injects the step run ID as an idempotency key on every call. The validation service **warns** when a `tool_call` step has `retry.max_attempts > 1` and the tool does not declare `idempotent: true`.

---

### `mcp_call`

Invokes a capability on a named MCP connector from the MCP Registry.

```json
{
  "id": "create_ticket",
  "type": "mcp_call",
  "mcp": "linear",
  "capability": "issues.create",
  "inputs": {
    "title": "$.steps.classify.output.issue_title",
    "description": "$.steps.summarize.output.text",
    "priority": "medium",
    "team_id": "TEAM-ABC"
  },
  "outputs": {
    "issue_id": "string",
    "issue_url": "string"
  },
  "next": "notify_user",
  "retry": {
    "max_attempts": 3,
    "non_retryable_errors": ["AuthenticationError", "ValidationError"]
  }
}
```

**Output envelope `output` shape**:

```json
{
  "mcp_name": "linear",
  "capability": "issues.create",
  "raw_output": { ... },
  "declared_outputs": {
    "issue_id": "LIN-4521",
    "issue_url": "https://linear.app/team/issue/LIN-4521"
  }
}
```

**Notes**:
- `mcp` must be a name registered in the MCP Registry. `capability` must be present in the MCP's cached manifest.
- Authentication is injected from the workflow context; no credentials appear in the DSL.
- Idempotency keys derived from the step run ID are automatically injected for MCP calls that declare `idempotent: true` in their manifest (E-12).
- MCP manifest cache TTL is 60 minutes by default; force-refresh with `agent-platform mcps refresh <name>` (E-13).

---

### `agent_call`

Invokes another agent as a sub-workflow. The most powerful composition primitive.

```json
{
  "id": "run_payment_agent",
  "type": "agent_call",
  "agent": "payment-processor",
  "version": "v3",
  "mode": "wait",
  "inputs": {
    "transaction_id": "$.workflow.input.transaction_id",
    "amount": "$.steps.validate.output.amount",
    "currency": "$.steps.validate.output.currency"
  },
  "outputs": {
    "confirmation_code": "string",
    "status": "string"
  },
  "next": "notify_user",
  "retry": { "max_attempts": 1 }
}
```

**`mode` values**:

| Mode | Behaviour |
|---|---|
| `wait` (default) | Parent workflow blocks until child completes. Child output is available to downstream steps. |
| `fire_and_forget` | Parent immediately proceeds. Child output is NOT available to downstream steps. The child run ID is available as `$.steps.<id>.output.child_run_id`. |

**Output envelope `output` shape (mode: wait)**:

```json
{
  "child_run_id": "wfr-xyz789",
  "child_agent": "payment-processor",
  "child_version": "v3",
  "child_status": "succeeded",
  "declared_outputs": {
    "confirmation_code": "CC-8821",
    "status": "approved"
  }
}
```

**Output envelope `output` shape (mode: fire_and_forget)**:

```json
{
  "child_run_id": "wfr-xyz789",
  "child_agent": "payment-processor",
  "child_version": "v3"
}
```

**Notes**:
- `version` is optional. If omitted, the latest version of the named agent at workflow start time is used and recorded on the step run.
- The child workflow is a fully independent Temporal workflow linked to the parent via `parent_run_id` in the database.
- `agent_call` inside a `for_each` body is fully supported. Each item's call becomes an independent child workflow. See `for_each` section.
- Cycle detection: the validation service traverses the full agent call graph and rejects configurations that create direct or transitive cycles. Maximum call depth is configurable (default: `10`).
- `agent` may use a `catalogue://slug` URI to reference a Shared Agent Catalogue entry. See `25-shared-agent-catalogue.md`.

**`on_orphan_failure` for `fire_and_forget` (E-10)**

Adds an optional field to `agent_call` steps with `mode: fire_and_forget`:

| Value | Behaviour |
|---|---|
| `log_only` (default) | Failure is logged in the orchestrator audit trail only. |
| `notify_conversation` | For chatbot-linked background tasks: surfaces a failure notice in the linked conversation. |
| `webhook` | POSTs a failure payload to the namespace's configured `orphan_failure_webhook_url`. |
| `create_pending_alert` | Creates an operator-visible alert in the platform's alert queue. |

The Orchestrator / Monitor detects `fire_and_forget` child workflows that have reached a terminal failure state and dispatches the configured action.

---

### `parallel`

Executes multiple independent steps or sub-blocks concurrently. Completes when all branches complete.

```json
{
  "id": "gather_context",
  "type": "parallel",
  "failure_policy": "fail_fast",
  "max_concurrency": 5,
  "branches": [
    {
      "id": "kb_results",
      "steps": [
        {
          "id": "search",
          "type": "tool_call",
          "tool": "knowledge_base_lookup",
          "inputs": { "query": "$.workflow.input.question" },
          "outputs": { "documents": "array" }
        }
      ]
    },
    {
      "id": "ticket_history",
      "steps": [
        {
          "id": "fetch_tickets",
          "type": "mcp_call",
          "mcp": "jira",
          "capability": "issues.search",
          "inputs": { "customer_id": "$.workflow.input.customer_id" },
          "outputs": { "issues": "array" }
        }
      ]
    }
  ],
  "next": "synthesize"
}
```

**`failure_policy` values**:

| Value | Behaviour |
|---|---|
| `fail_fast` | First branch failure cancels all others and fails the parallel block. |
| `collect_all` | All branches run to completion (or failure). The block succeeds if all succeed; fails with aggregated errors if any fail. |

**Output envelope `output` shape**:

```json
{
  "kb_results": {
    "search": {
      "documents": [...]
    }
  },
  "ticket_history": {
    "fetch_tickets": {
      "issues": [...]
    }
  }
}
```

The output is a map keyed by branch `id`, each containing a map of that branch's step outputs keyed by step `id`.

**Reference from downstream steps**: `$.steps.gather_context.output.kb_results.search.documents`

---

### `for_each`

Applies a step (or multi-step sub-block) to each element of an array input, in parallel with a concurrency limit.

```json
{
  "id": "process_items",
  "type": "for_each",
  "over": "$.steps.fetch_list.output.items",
  "item_variable": "item",
  "max_concurrency": 10,
  "item_failure_policy": "collect_errors",
  "body": {
    "steps": [
      {
        "id": "enrich",
        "type": "agent_call",
        "agent": "item-enricher",
        "mode": "wait",
        "inputs": {
          "item_id": "$.item.id",
          "item_name": "$.item.name"
        },
        "outputs": {
          "enriched_data": "object"
        }
      }
    ]
  },
  "outputs": {
    "results": "array",
    "errors": "array"
  },
  "next": "aggregate"
}
```

**`item_failure_policy` values**:

| Value | Behaviour |
|---|---|
| `fail_fast` | First item failure cancels all remaining and fails the step. |
| `skip` | Failed items are excluded from results. A `skipped` count is recorded. |
| `collect_errors` | All items run. Results include successes; errors are collected into `errors` array. |

**Output envelope `output` shape**:

```json
{
  "results": [
    {
      "item_index": 0,
      "item": { "id": "abc", "name": "Widget A" },
      "output": { "enriched_data": { ... } }
    },
    ...
  ],
  "errors": [
    {
      "item_index": 2,
      "item": { "id": "def", "name": "Widget C" },
      "error": { "type": "TimeoutError", "message": "..." }
    }
  ],
  "total": 5,
  "succeeded": 4,
  "failed": 1,
  "skipped": 0
}
```

**Notes**:
- `$.item.*` in body step inputs refers to the current iteration's element.
- `max_concurrency` is **required**. The validation service rejects `for_each` steps without it.
- Output order matches input order regardless of execution order.
- When body is a single `agent_call` with `mode: fire_and_forget`, each item spawns a background child workflow. The `for_each` step completes immediately with child run IDs in results.

**`parallel` block `max_concurrency`**: Also **required** on `parallel` blocks (E-09), consistent with `for_each`. The validation service rejects `parallel` blocks without `max_concurrency`. A platform-configurable global cap (default: `50`) is enforced even when `max_concurrency` is explicitly declared — the lower of the two values applies.

---

### `branch`

Conditional routing: evaluates an expression and routes to one of two downstream steps.

```json
{
  "id": "route_by_intent",
  "type": "branch",
  "condition": "$.steps.classify.output.intent == \"billing\"",
  "then": "billing_handler",
  "else": "general_handler",
  "next": null
}
```

**Notes**:
- `then` and `else` reference step IDs defined elsewhere in the `steps` array.
- `else` is optional. If the condition is false and no `else` is defined, the branch falls through to a `noop` convergence point (which must be wired as the implicit next for both paths).
- `next` should be `null` when `then`/`else` define different continuations. Set `next` only when both branches converge to the same step.
- The chosen branch (`then` or `else`) is persisted on the step run for audit.

**Output envelope `output` shape**:

```json
{
  "condition": "$.steps.classify.output.intent == \"billing\"",
  "result": true,
  "branch_taken": "then"
}
```

---

### `switch`

Multi-way routing: matches a value against a list of cases.

```json
{
  "id": "dispatch_specialist",
  "type": "switch",
  "value": "$.steps.classify.output.department",
  "cases": [
    { "match": "billing",   "next": "billing_agent" },
    { "match": "technical", "next": "technical_agent" },
    { "match": "admin",     "next": "admin_agent" }
  ],
  "default": "general_fallback"
}
```

**Notes**:
- `value` is an expression resolving to a scalar (string, integer, boolean).
- `match` values are literal scalars compared with strict equality.
- `default` is required if unmatched values are possible. The validation service warns (not errors) if `default` is absent, since exhaustive coverage may be provable from prior steps.
- The matched case and `value` result are persisted on the step run.

**Output envelope `output` shape**:

```json
{
  "value": "billing",
  "matched_case": "billing",
  "next": "billing_agent"
}
```

---

### `while`

Repeats a body block while a condition holds. Evaluates condition **before** each iteration (pre-condition loop).

```json
{
  "id": "refine_until_quality",
  "type": "while",
  "condition": "$.steps.refine_until_quality.loop_state.quality_score < 0.85",
  "max_iterations": 5,
  "body": {
    "steps": [
      {
        "id": "refine",
        "type": "llm_call",
        "inputs": {
          "draft": "$.steps.refine_until_quality.loop_state.current_draft | $.steps.initial_draft.output.text"
        },
        "prompt_template": "Improve this draft for clarity and accuracy:\n\n{{inputs.draft}}",
        "outputs": {
          "improved_draft": "string"
        }
      },
      {
        "id": "evaluate",
        "type": "tool_call",
        "tool": "quality_scorer",
        "inputs": {
          "text": "$.steps.refine.output.declared_outputs.improved_draft"
        },
        "outputs": {
          "quality_score": "number"
        }
      }
    ],
    "loop_state_update": {
      "current_draft": "$.steps.evaluate.output.declared_outputs.improved_draft | $.steps.refine.output.declared_outputs.improved_draft",
      "quality_score": "$.steps.evaluate.output.declared_outputs.quality_score"
    }
  },
  "next": "publish"
}
```

**Loop state**: `$.steps.<while_id>.loop_state.*` holds the mutable state that persists between iterations. Updated by `loop_state_update` at the end of each iteration. The loop condition references it via `$.steps.<while_id>.loop_state.*`.

**`max_iterations`**: Required. Default: `10`. Reaching the limit fails the step with `MaxIterationsError` (which is non-retryable).

**Output envelope `output` shape**:

```json
{
  "iterations_completed": 3,
  "terminated_by": "condition_false | max_iterations",
  "final_loop_state": {
    "current_draft": "...",
    "quality_score": 0.91
  },
  "last_body_outputs": {
    "refine": { ... },
    "evaluate": { ... }
  }
}
```

---

### `until`

Identical to `while` except the body executes **at least once** before the condition is evaluated (post-condition loop).

```json
{
  "id": "poll_until_ready",
  "type": "until",
  "condition": "$.steps.poll_until_ready.loop_state.status == \"complete\"",
  "max_iterations": 20,
  "body": {
    "steps": [
      {
        "id": "poll",
        "type": "tool_call",
        "tool": "job_status_checker",
        "inputs": {
          "job_id": "$.workflow.input.job_id"
        },
        "outputs": {
          "status": "string",
          "result": "object"
        }
      }
    ],
    "loop_state_update": {
      "status": "$.steps.poll.output.declared_outputs.status"
    }
  },
  "next": "process_result"
}
```

**Notes**: All semantics identical to `while` except execution order: body runs first, then condition is evaluated.

---

## Complete Worked Example: Multi-Step Multi-Agent Workflow (v4)

This example shows a customer support triage workflow with classification, parallel context gathering, conditional routing to specialists, and fan-out processing.

```json
{
  "dsl_version": "4",
  "agent_type": "workflow",
  "name": "support-triage",
  "namespace": "acme-corp",
  "description": "Classifies a support request, gathers context in parallel, routes to appropriate specialist agent, and logs to issue tracker.",
  "system_prompt": "You are a precise support triage assistant. Classify requests accurately and route to the right specialist.",
  "provider": "claude",
  "model_config": {
    "model_name": "claude-3-7-sonnet-20250219",
    "temperature": 0.2,
    "max_tokens": 1024
  },
  "workflow": {
    "entry": "classify",
    "steps": [
      {
        "id": "classify",
        "type": "llm_call",
        "inputs": {
          "request": "$.workflow.input.user_message",
          "customer_id": "$.workflow.input.customer_id"
        },
        "prompt_template": "Classify this support request into one of: billing, technical, admin, general.\n\nRequest: {{inputs.request}}\nCustomer ID: {{inputs.customer_id}}\n\nRespond with JSON: {\"intent\": \"...\", \"confidence\": 0.0, \"summary\": \"...\"}",
        "model_config_override": { "temperature": 0.0 },
        "outputs": {
          "intent": "string",
          "confidence": "number",
          "summary": "string"
        },
        "next": "gather_context",
        "retry": { "max_attempts": 2 }
      },
      {
        "id": "gather_context",
        "type": "parallel",
        "failure_policy": "collect_all",
        "branches": [
          {
            "id": "kb",
            "steps": [
              {
                "id": "kb_search",
                "type": "tool_call",
                "tool": "knowledge_base_lookup",
                "inputs": {
                  "query": "$.steps.classify.output.declared_outputs.summary",
                  "filters": { "collection": "support_docs" },
                  "top_k": 3
                },
                "outputs": { "documents": "array" }
              }
            ]
          },
          {
            "id": "history",
            "steps": [
              {
                "id": "fetch_history",
                "type": "mcp_call",
                "mcp": "jira",
                "capability": "issues.search",
                "inputs": {
                  "customer_id": "$.workflow.input.customer_id",
                  "limit": 5
                },
                "outputs": { "issues": "array" }
              }
            ]
          }
        ],
        "next": "route"
      },
      {
        "id": "route",
        "type": "switch",
        "value": "$.steps.classify.output.declared_outputs.intent",
        "cases": [
          { "match": "billing",   "next": "call_billing_agent" },
          { "match": "technical", "next": "call_technical_agent" },
          { "match": "admin",     "next": "call_admin_agent" }
        ],
        "default": "call_general_agent"
      },
      {
        "id": "call_billing_agent",
        "type": "agent_call",
        "agent": "billing-specialist",
        "mode": "wait",
        "inputs": {
          "customer_id": "$.workflow.input.customer_id",
          "summary": "$.steps.classify.output.declared_outputs.summary",
          "kb_docs": "$.steps.gather_context.output.kb.kb_search.documents",
          "prior_issues": "$.steps.gather_context.output.history.fetch_history.issues"
        },
        "outputs": { "resolution": "string", "actions_taken": "array" },
        "next": "log_outcome"
      },
      {
        "id": "call_technical_agent",
        "type": "agent_call",
        "agent": "technical-specialist",
        "mode": "wait",
        "inputs": {
          "customer_id": "$.workflow.input.customer_id",
          "summary": "$.steps.classify.output.declared_outputs.summary",
          "kb_docs": "$.steps.gather_context.output.kb.kb_search.documents"
        },
        "outputs": { "resolution": "string", "actions_taken": "array" },
        "next": "log_outcome"
      },
      {
        "id": "call_admin_agent",
        "type": "agent_call",
        "agent": "admin-specialist",
        "mode": "wait",
        "inputs": {
          "customer_id": "$.workflow.input.customer_id",
          "summary": "$.steps.classify.output.declared_outputs.summary"
        },
        "outputs": { "resolution": "string", "actions_taken": "array" },
        "next": "log_outcome"
      },
      {
        "id": "call_general_agent",
        "type": "agent_call",
        "agent": "general-support",
        "mode": "wait",
        "inputs": {
          "customer_id": "$.workflow.input.customer_id",
          "summary": "$.steps.classify.output.declared_outputs.summary"
        },
        "outputs": { "resolution": "string", "actions_taken": "array" },
        "next": "log_outcome"
      },
      {
        "id": "log_outcome",
        "type": "mcp_call",
        "mcp": "jira",
        "capability": "issues.create",
        "inputs": {
          "title": "$.steps.classify.output.declared_outputs.summary",
          "description": "$.steps.call_billing_agent.output.declared_outputs.resolution | $.steps.call_technical_agent.output.declared_outputs.resolution | $.steps.call_admin_agent.output.declared_outputs.resolution | $.steps.call_general_agent.output.declared_outputs.resolution | \"No resolution recorded\"",
          "customer_id": "$.workflow.input.customer_id",
          "intent": "$.steps.classify.output.declared_outputs.intent"
        },
        "outputs": { "issue_id": "string" },
        "next": null
      }
    ]
  }
}
```

---

## Cross-Agent Graph Validation (E-05)

When `agent_call` steps are present, the validation service supports a **graph validation mode** triggered by:
- `agent-platform validate --graph <file>` (CLI)
- Provisioning API: always runs graph validation when saving an agent with `agent_call` steps

**Graph validation process**:
1. Resolve all `agent_call` references recursively, loading each referenced agent's DSL from the platform.
2. Run depth-first traversal to detect cycles (maximum depth: `10`, configurable). Reject on cycle detected.
3. Cross-reference type compatibility: if agent B declares `input_schema`, every `agent_call` step targeting B must supply all `required` fields. Type compatibility is checked where types can be statically inferred.
4. Attribute errors to their source: errors in a transitively referenced agent are reported at the referencing step with a chain — e.g. `agent A → step run_payment → agent B: missing required input 'currency'`.
5. Graph validation is always online (requires platform API access). The CLI warns when running `--graph` without platform connectivity.

---

## Chatbot Agent Type (DSL v5) (E-02)

Chatbot agents use `agent_type: chatbot`. They have **no `workflow.steps` block** — the conversation loop is the runtime, not the DSL. The following fields replace `workflow`:

```json
{
  "dsl_version": "5",
  "agent_type": "chatbot",
  "name": "billing-specialist",
  "namespace": "acme-corp",
  "description": "Handles billing and payment queries.",
  "system_prompt": "You are a billing specialist for Acme Corp...",
  "provider": "claude",
  "model_config": { "model_name": "claude-3-7-sonnet-20250219", "max_tokens": 2048 },
  "tools": ["knowledge_base_lookup"],
  "mcps": [],
  "context_window": {
    "strategy": "sliding | summarize | semantic",
    "max_tokens": 8000,
    "summary_agent": "conversation-summarizer",
    "max_summary_tokens": 2000,
    "handoff_summary_tokens": 500
  },
  "episodic_memory": {
    "enabled": true,
    "top_k": 3
  },
  "semantic_memory": {
    "enabled": true,
    "auto_inject": true
  },
  "human_handoff": {
    "strategy": "message | webhook | ticket",
    "message": "Our team will be in touch shortly.",
    "webhook_url": "https://hooks.internal/handoff"
  },
  "degraded_message": "I'm temporarily unavailable. Please try again shortly."
}
```

**Chatbot-only fields**:

| Field | Type | Required | Notes |
|---|---|---|---|
| `context_window` | object | Yes | Working memory management strategy for this session. |
| `context_window.strategy` | enum | Yes | `sliding` (drop oldest turns), `summarize` (invoke `summary_agent`), `semantic` (retain by relevance). |
| `context_window.max_tokens` | integer | Yes | Token budget for the context window passed to the LLM. |
| `context_window.summary_agent` | string | Conditional | Required when `strategy: summarize`. Agent name for producing summaries. |
| `context_window.max_summary_tokens` | integer | No | Default: 25% of `max_tokens`. Token budget for the summary block. |
| `context_window.handoff_summary_tokens` | integer | No | Default: `500`. Token budget for cross-specialist handoff summaries. |
| `episodic_memory` | object | No | Enables retrieval of past session summaries from Qdrant. See `21-conversation-memory.md`. |
| `semantic_memory` | object | No | Enables user profile fact injection from the `user_profiles` table. See `21-conversation-memory.md`. |
| `human_handoff` | object | No | Escalation configuration. See `06-v5-chatbot-layer.md` Epic 7. |
| `degraded_message` | string | No | Message displayed when this specialist's circuit breaker is OPEN. See `22-chatbot-degradation.md`. |

**Shared fields** (same semantics as workflow agents): `dsl_version`, `agent_type`, `name`, `description`, `namespace`, `tags`, `system_prompt`, `provider`, `model_config`, `tools`, `mcps`, `input_schema`, `budget`.

**Routing chatbot** (agent with `routing: true`):

```json
{
  "agent_type": "chatbot",
  "routing": {
    "specialists": ["billing-specialist", "technical-specialist"],
    "default_specialist": "general-support",
    "confidence_threshold": 0.5,
    "low_confidence_action": "ask_clarification | route_to_default",
    "static_fallback_message": "I'm unable to help right now. Please contact support."
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `routing.specialists` | array | Ordered list of specialist agent names this router can delegate to. |
| `routing.default_specialist` | string | Fallback specialist when classification confidence is below threshold or a specialist circuit is OPEN. |
| `routing.confidence_threshold` | float | Default: `0.5`. Below this score, `low_confidence_action` is applied. |
| `routing.low_confidence_action` | enum | `ask_clarification` (prompt user for more info) or `route_to_default`. |
| `routing.static_fallback_message` | string | Displayed when all specialists are unavailable (full degraded mode). |

---

## Complete Worked Example: Routing + Specialist Chatbots (v5)

This example shows a customer support chatbot system with one routing chatbot and two specialist chatbots. The routing chatbot classifies intent and delegates; specialists handle domain-specific queries and can call backend workflow agents for heavy work.

### Routing Chatbot

```json
{
  "dsl_version": "5",
  "agent_type": "chatbot",
  "name": "support-router",
  "namespace": "acme-corp",
  "description": "Classifies customer intent and routes to appropriate specialist.",
  "system_prompt": "You are a support routing assistant. Analyze the user's message and classify into exactly one intent: billing, technical, or general. Respond with JSON: {\"intent\": \"...\", \"confidence\": 0.0-1.0, \"reasoning\": \"...\"}",
  "provider": "claude",
  "model_config": {
    "model_name": "claude-3-7-sonnet-20250219",
    "temperature": 0.0,
    "max_tokens": 512,
    "fallback_provider": {
      "provider": "openai",
      "model_name": "gpt-4o",
      "trigger_on": ["ProviderUnavailableError"],
      "max_attempts": 1
    }
  },
  "context_window": {
    "strategy": "sliding",
    "max_tokens": 4000
  },
  "routing": {
    "specialists": ["billing-specialist", "technical-specialist"],
    "default_specialist": "general-support",
    "confidence_threshold": 0.6,
    "low_confidence_action": "ask_clarification",
    "static_fallback_message": "I'm unable to route your request right now. Connecting you to a general support agent..."
  },
  "human_handoff": {
    "strategy": "message",
    "message": "I'll connect you with a human agent who can help further."
  },
  "degraded_message": "Our chat system is experiencing issues. Please try again in a few moments."
}
```

### Billing Specialist Chatbot

```json
{
  "dsl_version": "5",
  "agent_type": "chatbot",
  "name": "billing-specialist",
  "namespace": "acme-corp",
  "description": "Handles billing, payments, refunds, and subscription questions.",
  "system_prompt": "You are a billing specialist for Acme Corp. Help customers with invoices, payments, refunds, and subscription changes. Always verify account details before making changes. If you need to process a refund or modify a subscription, delegate to the billing-workflow agent.",
  "provider": "claude",
  "model_config": {
    "model_name": "claude-3-7-sonnet-20250219",
    "temperature": 0.7,
    "max_tokens": 2048
  },
  "input_schema": {
    "user_message": { "type": "string", "required": true },
    "conversation_id": { "type": "string", "required": true },
    "user_id": { "type": "string", "required": true },
    "prior_summary": { "type": "string", "required": false, "default": "" }
  },
  "tools": ["knowledge_base_lookup"],
  "mcps": ["stripe"],
  "context_window": {
    "strategy": "summarize",
    "max_tokens": 8000,
    "summary_agent": "conversation-summarizer",
    "max_summary_tokens": 2000,
    "handoff_summary_tokens": 500
  },
  "episodic_memory": {
    "enabled": true,
    "top_k": 3
  },
  "semantic_memory": {
    "enabled": true,
    "auto_inject": true
  },
  "knowledge_base": {
    "collection": "billing_docs",
    "filters": {
      "category": ["refunds", "payments", "invoices", "subscriptions"]
    },
    "retrieval_strategy": "hybrid",
    "top_k": 5
  },
  "human_handoff": {
    "strategy": "webhook",
    "webhook_url": "https://hooks.acme.com/billing-escalation",
    "message": "I'm connecting you with our billing team."
  },
  "degraded_message": "Billing support is temporarily unavailable. For urgent issues, please email billing@acme.com.",
  "budget": {
    "max_estimated_cost_usd": 2.00,
    "on_exceeded": "warn_and_continue"
  }
}
```

### Technical Specialist Chatbot

```json
{
  "dsl_version": "5",
  "agent_type": "chatbot",
  "name": "technical-specialist",
  "namespace": "acme-corp",
  "description": "Handles technical support, troubleshooting, and feature questions.",
  "system_prompt": "You are a technical support specialist for Acme Corp. Help customers troubleshoot issues, understand features, and debug errors. For complex debugging requiring log analysis or multi-step investigation, delegate to the technical-investigation workflow agent.",
  "provider": "openai",
  "model_config": {
    "model_name": "gpt-4o",
    "temperature": 0.5,
    "max_tokens": 2048
  },
  "tools": ["knowledge_base_lookup", "runbook_search"],
  "mcps": ["datadog", "sentry"],
  "context_window": {
    "strategy": "semantic",
    "max_tokens": 12000
  },
  "episodic_memory": {
    "enabled": true,
    "top_k": 5
  },
  "semantic_memory": {
    "enabled": true,
    "auto_inject": true
  },
  "knowledge_base": {
    "collection": "technical_docs",
    "filters": {
      "type": ["troubleshooting", "api_reference", "faq"]
    },
    "retrieval_strategy": "semantic",
    "top_k": 3
  },
  "human_handoff": {
    "strategy": "ticket",
    "ticket_system": "jira",
    "project": "SUPPORT",
    "message": "I've created a support ticket for you. Our engineering team will investigate."
  },
  "degraded_message": "Technical support is experiencing high load. Your question has been queued for our team.",
  "budget": {
    "max_estimated_cost_usd": 3.00,
    "on_exceeded": "cancel_with_partial_output"
  }
}
```

### Backend Workflow Agent (for Delegation)

The billing specialist delegates refund processing to this workflow agent:

```json
{
  "dsl_version": "4",
  "agent_type": "workflow",
  "name": "billing-refund-processor",
  "namespace": "acme-corp",
  "description": "Processes refund requests with validation and audit logging.",
  "system_prompt": "You are a refund processing agent. Verify eligibility, calculate amounts, and process refunds through Stripe.",
  "provider": "claude",
  "model_config": {
    "model_name": "claude-3-7-sonnet-20250219",
    "temperature": 0.2,
    "max_tokens": 1024
  },
  "input_schema": {
    "user_id": { "type": "string", "required": true },
    "order_id": { "type": "string", "required": true },
    "reason": { "type": "string", "required": true },
    "requested_amount_cents": { "type": "number", "required": false }
  },
  "workflow": {
    "entry": "lookup_order",
    "steps": [
      {
        "id": "lookup_order",
        "type": "mcp_call",
        "mcp": "stripe",
        "capability": "charges.retrieve",
        "inputs": {
          "order_id": "$.workflow.input.order_id"
        },
        "outputs": { "charge": "object", "refundable": "boolean" },
        "next": "validate_eligibility",
        "retry": { "max_attempts": 3, "initial_interval_seconds": 1 }
      },
      {
        "id": "validate_eligibility",
        "type": "llm_call",
        "inputs": {
          "charge": "$.steps.lookup_order.output.declared_outputs.charge",
          "reason": "$.workflow.input.reason"
        },
        "prompt_template": "Determine if this refund request is eligible based on the charge and reason.\n\nCharge: {{inputs.charge}}\nReason: {{inputs.reason}}\n\nRespond with JSON: {\"eligible\": boolean, \"max_refund_cents\": number, \"reasoning\": string}",
        "outputs": { "eligible": "boolean", "max_refund_cents": "number" },
        "next": "branch_on_eligibility"
      },
      {
        "id": "branch_on_eligibility",
        "type": "branch",
        "condition": "$.steps.validate_eligibility.output.declared_outputs.eligible",
        "then": "process_refund",
        "else": "reject_refund"
      },
      {
        "id": "process_refund",
        "type": "mcp_call",
        "mcp": "stripe",
        "capability": "refunds.create",
        "inputs": {
          "charge_id": "$.steps.lookup_order.output.declared_outputs.charge.id",
          "amount": "$.steps.validate_eligibility.output.declared_outputs.max_refund_cents"
        },
        "outputs": { "refund_id": "string", "status": "string" },
        "next": "log_success"
      },
      {
        "id": "reject_refund",
        "type": "transform",
        "inputs": {
          "reason": "$.steps.validate_eligibility.output.declared_outputs.reasoning"
        },
        "output_mapping": {
          "status": "rejected",
          "rejection_reason": "$.inputs.reason"
        },
        "next": "log_rejection"
      },
      {
        "id": "log_success",
        "type": "mcp_call",
        "mcp": "jira",
        "capability": "issues.create",
        "inputs": {
          "project": "REFUNDS",
          "summary": "Refund processed for order {{$.workflow.input.order_id}}",
          "user_id": "$.workflow.input.user_id",
          "refund_id": "$.steps.process_refund.output.declared_outputs.refund_id"
        },
        "outputs": { "issue_id": "string" },
        "next": null
      },
      {
        "id": "log_rejection",
        "type": "mcp_call",
        "mcp": "jira",
        "capability": "issues.create",
        "inputs": {
          "project": "REFUNDS",
          "summary": "Refund rejected for order {{$.workflow.input.order_id}}",
          "rejection_reason": "$.steps.reject_refund.output.declared_outputs.rejection_reason"
        },
        "outputs": { "issue_id": "string" },
        "next": null
      }
    ]
  },
  "budget": {
    "max_estimated_cost_usd": 1.00,
    "on_exceeded": "cancel"
  }
}
```

### Usage Flow

1. **End User** sends message to chat UI
2. **Router** classifies intent with confidence score
3. If confidence >= 0.6: Route to appropriate specialist
4. If confidence < 0.6: Ask clarifying question
5. **Specialist** receives conversation context + prior summary
6. Specialist can:
   - Answer from knowledge base
   - Call MCPs (Stripe, Datadog)
   - Delegate to backend workflow agent
7. Backend workflow runs asynchronously
8. When complete, result surfaces in conversation

---

## Validation Error Reference

The validation service returns structured errors with the following fields:

```json
{
  "errors": [
    {
      "code": "UNKNOWN_TOOL",
      "field": "workflow.steps[2].tool",
      "value": "web_scraper",
      "message": "Tool 'web_scraper' is not registered in the Tool Registry.",
      "severity": "error"
    },
    {
      "code": "UNRESOLVABLE_REFERENCE",
      "field": "workflow.steps[3].inputs.documents",
      "value": "$.steps.search.output.docs",
      "message": "Step 'search' does not declare output field 'docs'. Declared outputs: ['documents', 'scores'].",
      "severity": "error"
    },
    {
      "code": "MISSING_MAX_CONCURRENCY",
      "field": "workflow.steps[5]",
      "value": null,
      "message": "for_each step 'process_items' requires a max_concurrency value.",
      "severity": "error"
    },
    {
      "code": "CYCLE_DETECTED",
      "field": "workflow.steps[1].agent",
      "value": "support-triage",
      "message": "Agent call creates a cycle: support-triage → support-triage. Maximum call depth is 10.",
      "severity": "error"
    },
    {
      "code": "DEPRECATED_PROVIDER",
      "field": "provider",
      "value": "grok",
      "message": "Provider 'grok' is deprecated. It will continue to work but should be migrated.",
      "severity": "warning"
    }
  ]
}
```

**Severity levels**:
- `error`: Provisioning is rejected. The document must be fixed.
- `warning`: Provisioning proceeds but the issue should be addressed.

---

## What to Read Next

- **[15-developer-tooling.md](./15-developer-tooling.md)** — Local CLI for validating and dry-running DSL documents
- **[05-v4 — Multi-Agent Orchestration](./05-v4-multi-agent-orchestration.md)** — Phase document for v4 step types
- **[09-Security Model](./09-security-model.md)** — Credential references and DSL security constraints
- **[11-Testing Strategy](./11-testing-strategy.md)** — Contract testing the DSL schema

---

## Related Documents

- [01-PRD](./01-PRD.md) — AD-02 (JSON DSL), AD-03 (dynamic interpretation), AD-13 (validation accumulates errors)
- [03-v2](./03-v2-workflow-orchestration.md) — Data flow and step output envelope (original introduction)
- [04-v3](./04-v3-tools-and-mcps.md) — Tool and MCP step types
- [05-v4](./05-v4-multi-agent-orchestration.md) — Composition step types
