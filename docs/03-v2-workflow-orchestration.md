# v2 — Workflow Orchestration

## Version Goal

Introduce **Temporal-backed multi-step workflows** dynamically interpreted from the DSL. Agents are no longer one-shot prompts — they can declare a sequence of steps with inputs, outputs, and retry behavior, all executed durably with Temporal handling parallelization, retries, and state.

The Agent Core from v1 remains untouched. The infrastructure layer wraps it with workflow orchestration.

## Business Value

- Agents can now execute complex, multi-step logic: prepare data, call the LLM, transform output, persist results, all reliably.
- Failures are recoverable and observable — Temporal gives durability and retry semantics that would otherwise take months to build.
- Each step is independently scalable on Kubernetes via separate worker processes.
- This phase delivers the architectural pattern needed by every real-world use case described in the discovery.

---

## Epic 1 — DSL Workflow Schema

**Business Value**: Extends the declarative language so agent builders can describe complex orchestration without writing code.

### Story 1.1 — Add a `workflow` section to the DSL
**As an** Agent Builder
**I want** to declare an ordered list of steps in my agent definition
**So that** the platform can execute multi-step logic without me writing workflow code.

**Description**: Extend the v1 DSL with a `workflow` field containing an array of `steps`. Each step has: `id`, `type` (e.g. `llm_call`, `transform`, `fetch`, `noop`), `inputs` (mapping from prior step outputs or workflow input), `outputs` (declared output shape), `next` (id of the next step), and `retry` (retry policy).

**Acceptance Criteria**:
- The v2 schema is published, backward-compatible with v1 single-step agents (workflow section is optional).
- Example workflows are included in docs covering sequential execution and one branching scenario.

### Story 1.2 — Step-level retry policy
**As an** Agent Builder
**I want** to declare retry behavior per step
**So that** transient failures recover automatically without dropping the whole workflow.

**Description**: Each step accepts a `retry` block: `max_attempts`, `initial_interval`, `backoff_coefficient`, `max_interval`, `non_retryable_errors`. These are passed directly to Temporal's retry policy.

**Acceptance Criteria**:
- A step with a transient error retries according to its policy.
- A non-retryable error fails the step immediately.
- The policy defaults are documented.

### Story 1.3 — Data flow declaration between steps
**As an** Agent Builder
**I want** to declare how each step's outputs feed the next step's inputs
**So that** complex data transformations are visible and auditable in the DSL.

**Description**: A step's `inputs` field maps named inputs to expressions referencing the workflow input or prior step outputs (e.g. `{"question": "$.workflow.input.user_question", "context": "$.steps.fetch.output.docs"}`). Validation checks all references resolve.

**Acceptance Criteria**:
- References to non-existent steps or fields fail validation.
- At runtime, the input resolution produces the expected payload for the step.

---

## Epic 2 — Temporal Integration

**Business Value**: Brings industrial-grade durability, retries, and orchestration to the platform. Operators no longer need to build these primitives from scratch.

### Story 2.1 — Temporal worker process
**As a** Platform Engineer
**I want** a worker process that connects to Temporal and registers the dynamic workflow handler
**So that** workflows defined in the DSL can be executed.

**Description**: A worker service that connects to the Temporal cluster, polls a task queue, and executes activities. The workflow definition is generic: it loads the agent's DSL config at the start of the workflow run, then dispatches each step to the appropriate activity based on its `type`.

**Acceptance Criteria**:
- A worker can be started, registers cleanly with Temporal, and shuts down gracefully.
- Worker logs include the workflow run ID and step ID for every action.
- A simple two-step DSL workflow runs to completion via the worker.

### Story 2.2 — Activity implementations per step type
**As a** Platform Engineer
**I want** a clear, finite set of step-type activities
**So that** the dispatch logic is simple and step behaviors are well-isolated.

**Description**: Implement activities for the initial step types: `llm_call` (wraps the Agent Core), `transform` (applies a declarative transformation to inputs), `fetch` (a placeholder for HTTP/DB calls — fully fleshed out in v3 once tools land), `noop` (control-flow only). Each activity is independently testable.

**Acceptance Criteria**:
- Each activity is implemented behind a clear contract.
- Each activity has unit tests with mocked dependencies.
- The dispatcher correctly routes step types to their activities.

### Story 2.3 — Workflow context with pre-loaded credentials
**As a** Platform Engineer
**I want** credentials and runtime metadata loaded into the workflow context at the entry point
**So that** subsequent steps avoid repeated database calls and the codebase is ready for vault-based loading in v6.

**Description**: When a workflow starts, the entry-point activity (or workflow code) loads all credentials/secrets relevant to the agent's tools and MCPs from the database, packages them into an in-memory context object, and passes that context through each step. v6 swaps the data source for the vault without changing the contract.

**Acceptance Criteria**:
- Credentials are fetched once per workflow run.
- Each step receives the context as an input argument (not via global state).
- The interface is abstracted such that the data source is swappable.

---

## Epic 3 — State Machine Persistence

**Business Value**: Provides full auditability, recovery from failure, and the basis for the monitoring/orchestrator described in discovery.

### Story 3.1 — Per-step persistence
**As an** Operator
**I want** every step execution recorded in the database with its input, output, status, and timing
**So that** I can audit, debug, and analyze workflow behavior after the fact.

**Description**: Add a `workflow_step_runs` table (or sub-document on the workflow run). Each step gets a record with `workflow_run_id`, `step_id`, `attempt_number`, `status`, `input`, `output`, `error`, `started_at`, `completed_at`.

**Acceptance Criteria**:
- Every step execution writes a record at start, updates on completion.
- Retries produce separate records per attempt.
- A single query can retrieve the full timeline of a workflow run.

### Story 3.2 — Workflow run state advancement
**As an** Operator
**I want** the workflow run record to reflect the current step and overall progress
**So that** the UI and upstream consumers can show meaningful progress without querying every step.

**Description**: Extend `workflow_runs` with `current_step_id`, `total_steps`, `progress_percent`. These are updated by the Temporal worker as it advances. The overall workflow status (`pending`, `running`, `succeeded`, `failed`) is maintained at the workflow run level.

**Acceptance Criteria**:
- The workflow run record always reflects the latest known state.
- The transition timing is captured in `started_at` / `completed_at`.

---

## Epic 4 — Orchestrator / Monitor

**Business Value**: The polling-based monitor described in discovery — provides the loose coupling between callers and workflows without introducing a message bus.

### Story 4.1 — Background monitor service
**As an** Operator (or upstream service)
**I want** a background process that watches workflow runs and surfaces status changes
**So that** the platform can react to completions without coupling callers tightly to Temporal.

**Description**: A polling service (initially polling the database) that watches for state transitions on workflow runs and emits internal events. v2 uses simple periodic polling with configurable interval; v6 may optimize.

**Acceptance Criteria**:
- The monitor detects status transitions reliably with configurable latency.
- It does not lose events under restart (idempotent transitions).
- It exposes an internal subscription API for downstream consumers.

### Story 4.2 — Workflow run query API
**As an** Operator
**I want** to query workflow runs by status, agent, or time range
**So that** I can build dashboards and debug failures.

**Description**: An HTTP API exposing workflow run records: list with filters, fetch by ID, fetch step timeline. Read-only.

**Acceptance Criteria**:
- The API supports the listed filters with pagination.
- The endpoints are documented (OpenAPI or equivalent).

---

## Epic 5 — End-to-End Workflow Execution Path

**Business Value**: Proves the whole stack works end to end before moving to richer capabilities in v3+.

### Story 5.1 — Multi-step example agent
**As an** Agent Builder
**I want** a documented example agent with at least three steps
**So that** I have a reference for building my own.

**Description**: Build a reference agent — e.g. "Summarize and Translate": step 1 LLM summarize, step 2 transform/clean, step 3 LLM translate. Document its DSL fully.

**Acceptance Criteria**:
- The example runs successfully end to end.
- The DSL is in the repository.
- Documentation walks through each step.

### Story 5.2 — Failure scenarios validated
**As an** Operator
**I want** known failure scenarios to behave predictably
**So that** I can trust the platform under adverse conditions.

**Description**: Validate behavior under: provider timeout, malformed step output, non-retryable error, retry exhaustion, worker restart mid-run. Document expected outcomes for each.

**Acceptance Criteria**:
- Each scenario has an automated or documented test.
- The system recovers gracefully or fails cleanly with persisted error details.

---

## Canonical Step Output Envelope

Every step — regardless of type — writes its result to `workflow_step_runs.output` using a consistent envelope shape. This contract must be locked down in v2 so that v3+ data flow expressions (`$.steps.<id>.output.*`) are built on a stable foundation.

```json
{
  "output": { },
  "metadata": {
    "step_id": "string",
    "step_type": "llm_call | transform | fetch | noop | ...",
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

The `metadata` block is always present and always has this shape. The `output` object's shape is step-type-specific:

| Step Type | `output` Shape |
|---|---|
| `llm_call` | `{ "text": "...", "finish_reason": "stop\|tool_use\|max_tokens", "declared_outputs": { ... }, "tool_calls": [...] }` |
| `transform` | The declared output fields directly (e.g. `{ "count": 5, "items": [...] }`) |
| `fetch` | `{ "status_code": 200, "body": { ... }, "declared_outputs": { ... } }` |
| `noop` | `{}` |

Steps added in v3+ extend this table. The full per-step shape is specified in `14-dsl-spec.md`.

**Data flow references** use `$.steps.<step_id>.output.<field>` to reference the `output` portion, and `$.steps.<step_id>.metadata.<field>` for metadata. The envelope wrapping is transparent to DSL expressions.

---

## Technical Considerations (high level)

- **Workflow definition is dynamic, not generated code**: a single generic Temporal workflow handler reads the DSL and dispatches activities. No per-agent code generation.
- **Activities are pure**: they receive everything they need as arguments (no global state, no implicit DB reads beyond the context they're given). This keeps them retry-safe.
- **Persistence is separate from Temporal state**: Temporal manages workflow execution state; the database is the application's source of truth for product-visible data. They coexist; do not try to read Temporal history as a product API.
- **Worker deployment**: each worker is a Kubernetes deployment with its own task queue if isolation is needed. v2 can ship with one worker; segmentation is a v3+ concern.
- **Idempotency**: every activity must be safe to retry. Persist effects via idempotency keys (e.g. step run ID).
- **Output envelope is immutable once written**: a completed step's output record is never updated. If a step is retried, a new `workflow_step_runs` row is inserted with the new `attempt_number`. The canonical output envelope must be written atomically with the step's `succeeded` status.
- **Monitor scaling trigger (E-38)**: when the monitor detects that the `workflow_runs` table has more than `MONITOR_SCALE_THRESHOLD` (default: `1000`) rows in `RUNNING` state, it emits a `high_pending_runs_total` Prometheus counter. This metric is the designated auto-scaling trigger for the worker pool — infrastructure teams wire this to a HPA or Keda ScaledObject. The threshold is configurable via environment variable; the monitor does not perform the scaling itself.

## Definition of Done for v2

- An agent with a multi-step DSL workflow executes end to end through Temporal.
- All step executions are durably persisted with full input/output.
- The monitor detects state changes and exposes them via API.
- At least one documented multi-step reference agent works.
- Failure modes are documented and behave correctly.

---

## What to Read Next

With v2, agents can execute multi-step workflows. The next phase adds tools and external integrations:

→ **[04-v3 — Tools and MCPs](./04-v3-tools-and-mcps.md)** — Tool registry, MCP connectors, and credentials pipeline.

Or explore supporting topics:
- [02-v1](./02-v1-agent-provisioning-foundation.md) — Review the foundation (single-step agents)
- [08-Performance and Scaling](./08-performance-and-scaling.md) — Workflow latency and fan-out limits
- [12-Upgrades and Migrations](./12-upgrades-and-migrations.md) — Database evolution strategies
