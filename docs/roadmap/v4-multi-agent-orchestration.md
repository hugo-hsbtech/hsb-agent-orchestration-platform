# v4 — Multi-Agent Orchestration

## Version Goal

Enable **agents to call other agents** as sub-workflows, and add the workflow control-flow constructs needed for genuinely complex agent compositions: parallel execution, conditional branching, loops, fan-out/fan-in, and explicit data mapping between steps.

By the end of this phase, the platform can model arbitrary orchestrations: a parent agent that fans out to multiple specialist agents in parallel, collects their results, branches based on a classifier, and loops a refinement step until a condition is met.

## Business Value

- Specialist agents become composable building blocks: a "router" agent can dispatch to "payments", "support", "knowledge" agents, each in turn able to call further sub-agents.
- Workflows that would otherwise be hard-coded become DSL-configured.
- Real-world product scenarios (customer support routing, multi-stage research, multi-modal pipelines) become directly expressible.

---

## Epic 1 — Sub-Agent Invocation

**Business Value**: The atomic primitive for compositional agent design. Without this, every multi-agent system is monolithic.

### Story 1.1 — `agent_call` step type
**As an** Agent Builder
**I want** to invoke another agent as a step in my workflow
**So that** I can build hierarchies of specialized agents.

**Description**: Add an `agent_call` step type referencing another agent by ID (or name + version). At runtime, this triggers a child Temporal workflow for the called agent, waits for its completion, and exposes its output to subsequent steps.

**Acceptance Criteria**:
- A parent agent can invoke a child agent successfully.
- The child workflow is a separate, queryable workflow run linked to the parent via a `parent_run_id`.
- The child's output flows into the parent's step outputs per the v2 data flow rules.
- Failures propagate correctly per the retry/error policy.

### Story 1.2 — Fire-and-forget sub-agent invocation
**As an** Agent Builder
**I want** to optionally kick off a sub-agent without waiting for its result
**So that** I can fan work out asynchronously for background processing.

**Description**: The `agent_call` step accepts a `mode` field: `wait` (default) or `fire_and_forget`. The latter starts the child workflow and immediately proceeds.

**Acceptance Criteria**:
- A `fire_and_forget` call returns immediately with the child run ID.
- The child still appears in the workflow runs database with its parent linkage.
- The parent's overall completion is independent of the child's outcome.

### Story 1.3 — Version pinning for sub-agents
**As an** Agent Builder
**I want** to pin sub-agent calls to specific agent versions
**So that** parent agents are not silently broken when child agents change.

**Description**: The `agent_call` step accepts `agent_id` plus optional `version`. If version is omitted, the latest at workflow start time is used and recorded; if specified, that exact version is invoked. v6 introduces the full versioning system; v4 prepares for it.

**Acceptance Criteria**:
- The actual version used is persisted on the parent step's record.
- An unknown version fails validation at provisioning time.

---

## Epic 2 — Parallel Execution

**Business Value**: Performance and scalability. Sequential-only workflows do not scale to real-world multi-agent scenarios.

### Story 2.1 — Parallel step blocks
**As an** Agent Builder
**I want** to declare that multiple steps run in parallel
**So that** independent work happens concurrently instead of sequentially.

**Description**: Add a `parallel` step type containing an array of child steps. All children execute concurrently; the parent step completes when all children complete. Outputs are aggregated under the parent's output as a map keyed by child step IDs.

**Acceptance Criteria**:
- A `parallel` block with N independent children executes them concurrently.
- The block's output exposes each child's output by ID.
- Failure of any child triggers the block's failure policy (`fail_fast` or `collect_all`).

### Story 2.2 — Fan-out over a collection
**As an** Agent Builder
**I want** to apply a step (or sub-agent) to each item in a collection in parallel
**So that** batch operations on lists are first-class.

**Description**: Add a `for_each` step type that iterates over an array input, running a declared sub-step for each item in parallel (with a required concurrency limit). Outputs collect into an ordered array. The body may be any step type — including `agent_call`.

**`for_each` with `agent_call` body (fully supported)**:

When the body step is an `agent_call`, each array item spawns an independent child Temporal workflow. This is the idiomatic pattern for batch agent processing:
- `mode: wait` (default) — the `for_each` step blocks until all child workflows complete. The concurrency limit controls how many child workflows run simultaneously.
- `mode: fire_and_forget` — the `for_each` step completes immediately after dispatching all child workflows. Each item's result in the output array contains only the child run ID, not the child's output. This is appropriate for bulk background dispatch.

The body step's `inputs` use `$.item.*` to access the current array element.

**Acceptance Criteria**:
- A collection of N items is processed concurrently with a required `max_concurrency` value (the validation service rejects `for_each` steps without it).
- The output preserves input order regardless of execution order.
- Per-item failure modes are configurable: `fail_fast`, `skip`, `collect_errors`.
- A `for_each` with an `agent_call (wait)` body creates N independently queryable child workflow runs, all linked to the parent `for_each` step via `parent_run_id`.
- A `for_each` with an `agent_call (fire_and_forget)` body dispatches all child workflows and returns immediately with child run IDs.
- The full specification and output envelope for `for_each` is in `14-dsl-spec.md`.

---

## Epic 3 — Conditional Branching

**Business Value**: Real-world workflows are not linear. Branching turns the DSL into a true workflow language.

### Story 3.1 — `if/else` step
**As an** Agent Builder
**I want** to branch workflow execution based on a condition
**So that** I can express decision logic without code.

**Description**: Add a `branch` step type with a `condition` expression (over prior step outputs and workflow input), a `then` step ID, and an optional `else` step ID. The expression language is initially a small, safe subset (equality, comparison, AND/OR, presence checks).

**Acceptance Criteria**:
- A branch evaluates its condition and dispatches to the correct downstream step.
- Invalid conditions fail at validation time, not runtime.
- The chosen branch is recorded on the step run for audit.

### Story 3.2 — `switch` / pattern match step
**As an** Agent Builder
**I want** to dispatch to one of many downstream steps based on a value
**So that** classifier-driven routing is concise.

**Description**: Add a `switch` step type matching a value against a list of `cases`, each pointing to a downstream step, with an optional `default`.

**Acceptance Criteria**:
- A switch with N cases routes correctly to the matching case.
- A non-matching value with no default fails the step explicitly.
- The matched case is persisted.

---

## Epic 4 — Loops

**Business Value**: Some patterns (retry-with-refinement, iterative critique) require explicit looping in the DSL.

### Story 4.1 — `while` loop step
**As an** Agent Builder
**I want** to repeat a sub-step until a condition is satisfied
**So that** iterative refinement workflows are expressible.

**Description**: Add a `while` step type with a `condition` evaluated before each iteration, a `body` step or block, an optional `max_iterations` (default mandatory cap to avoid runaway loops).

**Acceptance Criteria**:
- A loop runs until the condition becomes false or max iterations are reached.
- Each iteration is independently persisted as a step run with `attempt_number`.
- Reaching max iterations triggers a defined error.

### Story 4.2 — `until` loop variant
**As an** Agent Builder
**I want** a complementary post-condition loop
**So that** I can express "do … until satisfied" patterns naturally.

**Description**: Add an `until` variant that runs the body at least once before checking the condition.

**Acceptance Criteria**:
- The body runs at least one full iteration before condition evaluation.
- Otherwise behaves identically to `while`.

---

## Epic 5 — Data Mapping and Transformations

**Business Value**: Real-world workflows constantly need to reshape data between steps. Without first-class mapping, agents end up doing menial transformation in LLM calls, which is expensive and unreliable.

### Story 5.1 — Inline data mapping in step inputs
**As an** Agent Builder
**I want** to declare structured transformations on inputs
**So that** I don't need an LLM call to reshape JSON.

**Description**: Extend the step `inputs` declaration to support mapping operations: field selection, renaming, default values, simple expressions. Use a constrained, validated expression language (JSONata-style or a curated subset).

**Acceptance Criteria**:
- Inputs can be derived from arbitrary combinations of prior outputs with declared transformations.
- Invalid expressions fail validation at provisioning time.
- The transformed inputs are recorded on the step run.

### Story 5.2 — Dedicated `transform` step type
**As an** Agent Builder
**I want** an explicit transformation step
**So that** complex reshaping is its own auditable, named step rather than hidden in an input map.

**Description**: Promote the `transform` step from v2's placeholder to a fully implemented step that accepts a mapping spec and produces a structured output. Useful when transformations are non-trivial or reused across branches.

**Acceptance Criteria**:
- A transform step runs and produces declared outputs.
- The transformation expression is validated and persisted.

---

## Technical Considerations (high level)

- **Child workflow lifecycle**: use Temporal's native child workflow primitives. Parent and child workflows are first-class Temporal workflows.
- **Parallel execution cost**: parallel blocks multiply worker load. Document expected fan-out limits and provide max-concurrency knobs.
- **Expression language safety**: keep it simple, sandboxed, and free of side effects. Avoid embedding a general-purpose language at this stage.
- **DSL validation gets richer**: the validator now must verify type compatibility across branches, ensure all branches converge correctly, detect unreachable steps, and limit recursion depth in agent calls.
- **Recursion limits**: an agent calling itself (directly or transitively) needs cycle detection or explicit depth limits to avoid runaway invocations.
- **Workflow run linkage**: the database schema must support parent/child workflow run relationships, including for multi-level chains.

## Definition of Done for v4

- An agent can invoke another agent (sync or fire-and-forget) with version pinning.
- Workflows can express parallel execution and collection-based fan-out.
- Conditional branching (if/else and switch) is fully supported.
- While/until loops with safe iteration caps are working.
- Inline data mapping and explicit transform steps allow shape changes without LLM calls.
- DSL validation catches type mismatches across branches and cycle hazards.

---

## What to Read Next

With v4, complex multi-agent orchestrations are possible. The next phase adds the user-facing layer:

→ **[06-v5 — Chatbot Layer](./06-v5-chatbot-layer.md)** — Streaming chatbots, routing, specialists, and background task surfacing.

Or explore supporting topics:
- [08-Performance and Scaling](./08-performance-and-scaling.md) — Parallel execution and fan-out limits
- [13-Chatbot UX Edge Cases](./13-chatbot-ux-edge-cases.md) — Routing ambiguity and edge cases
- [04-v3](./04-v3-tools-and-mcps.md) — Review tools and MCPs
