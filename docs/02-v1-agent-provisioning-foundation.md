# v1 — Agent Provisioning Foundation

## Version Goal

Deliver the minimum end-to-end slice: the ability to **define an agent declaratively**, persist it in the database, validate its configuration, and **execute a single-step LLM call** through a clean, provider-agnostic core. No workflow orchestration yet — just one prompt, one model call, one response.

This phase establishes the foundational contracts that every subsequent phase depends on: the DSL shape, the agent core abstraction, and the provider interface.

## Business Value

- The team can prove the core architecture works end to end before investing in orchestration complexity.
- Product stakeholders can already create and call agents with custom system prompts against multiple LLM providers — a useful capability in itself.
- All future phases (workflows, tools, MCPs, chatbots) plug into contracts proven in this phase.

---

## Epic 1 — Database Schema and Persistence Layer

**Business Value**: Establishes the durable source of truth for every agent in the system. Without this, no other capability can be built.

### Story 1.1 — Agent record persistence
**As an** Agent Builder
**I want to** persist an agent definition into the database with a unique identifier
**So that** the platform can later retrieve and execute it.

**Description**: Define and migrate the database schema for the `agents` table (or document collection). Minimum fields: `id`, `namespace_id` (FK to `namespaces`), `name`, `description`, `system_prompt`, `model_provider`, `model_config` (JSON), `created_at`, `updated_at`. The `namespaces` table is also created in v1 with a seeded `DEFAULT` namespace (see `16-multi-tenancy.md`, AD-26). Plain-text storage is acceptable for credentials in v1 — vault migration is in v6.

**Acceptance Criteria**:
- An agent record can be inserted via a service-layer call.
- The record can be fetched by ID with all fields intact.
- The schema is documented and migration-controlled.

### Story 1.2 — Workflow run persistence skeleton
**As an** Operator
**I want** every agent invocation to leave a durable trace in the database
**So that** we can audit, monitor, and analyze every execution.

**Description**: Create the `workflow_runs` table (even though v1 has no workflow yet, the contract starts here). Fields: `id`, `namespace_id` (FK to `namespaces`), `agent_id`, `status` (enum: pending, running, succeeded, failed), `input` (JSON), `output` (JSON nullable), `started_at`, `completed_at`, `error` (nullable). This table will hold every invocation, including single-step ones in v1.

**Acceptance Criteria**:
- Each agent invocation in v1 creates exactly one `workflow_runs` row.
- Status transitions correctly across the lifecycle.
- The full input and output are queryable.
- Rows are always scoped to a `namespace_id`; cross-namespace queries are blocked at the service layer.

---

## Epic 2 — DSL Schema v1 (Agent Definition)

**Business Value**: A clear, declarative schema is what makes the platform configurable without code changes. v1 establishes the minimum vocabulary; later phases extend it.

### Story 2.1 — Define the v1 JSON schema for an agent
**As an** Agent Builder
**I want** a documented JSON schema for defining an agent
**So that** I know exactly which fields are required and how they will be interpreted.

**Description**: The v1 DSL is minimal. Required fields: `name`, `description`, `system_prompt`, `provider` (enum: `claude`, `openai`, `gemini`, `grok`, `other`), `model_config` (object with at least `model_name`, `temperature`, `max_tokens`). No `workflow` section yet — that arrives in v2.

**Acceptance Criteria**:
- A JSON Schema document is published and version-tagged.
- Example agent definitions are included in the docs.
- The schema validates with a standard JSON Schema validator.

### Story 2.2 — Accept user prompt template in DSL
**As an** Agent Builder
**I want** to declare how runtime input is structured into the user prompt
**So that** the agent can be invoked with structured data and produce contextual outputs.

**Description**: Add a `user_prompt_template` field to the DSL — a string with variable interpolation placeholders (e.g. `{{input.question}}`). When the agent runs, the runtime input is interpolated into this template before being sent to the LLM.

**Acceptance Criteria**:
- Templates support simple variable substitution.
- Missing variables produce a clear validation error.
- The interpolated prompt is stored in the workflow run record for audit.

---

## Epic 3 — Validation Service

**Business Value**: Catches misconfiguration at provisioning time rather than at runtime, dramatically reducing operational headaches and giving builders fast feedback.

### Story 3.1 — Schema-level validation
**As an** Agent Builder
**I want** my agent definition to be validated against the schema at provisioning time
**So that** I get immediate, specific feedback on what is wrong with my configuration.

**Description**: A standalone validation service that takes a candidate DSL document and returns a list of structured errors. v1 validation: schema compliance, required fields, allowed enum values, valid JSON.

**Acceptance Criteria**:
- The service is callable both at provisioning time and as a standalone API endpoint.
- Errors include the field path, the expected value, the received value, and a human-readable message.
- Multiple errors are collected and returned together (no fail-fast).

### Story 3.2 — Provider availability validation (E-06)
**As an** Agent Builder
**I want** the platform to reject agent definitions that reference unavailable model providers
**So that** I cannot accidentally provision a broken agent.

**Description**: The provider check is split into two distinct stages to avoid coupling provisioning to provider availability:

1. **At provisioning time (mandatory, static)**: verify that the declared provider has credentials configured in the platform (a local DB lookup — no network call). Fail provisioning if not. This keeps provisioning fast and independent of provider availability.
2. **At invocation time (first execution, live)**: run a `probe()` call as the first activity in the workflow. If the probe fails, the workflow fails immediately with `ProviderUnavailableError` (which is retryable per the step retry policy). This decouples agent registration from provider uptime — a provider outage does not block Agent Builders from provisioning new agents.

If `fallback_provider` is declared in `model_config`, the same static credential check applies to the fallback provider at provisioning time.

**Acceptance Criteria**:
- An agent referencing a provider with no configured credentials fails provisioning.
- An agent referencing a provider that is temporarily unreachable still provisions successfully.
- The invocation-time probe failure surfaces as `ProviderUnavailableError` on the workflow run record.
- The error message names the missing provider configuration explicitly.

---

## Epic 4 — Provider-Agnostic LLM Interface

**Business Value**: Locks in the abstraction that makes the platform genuinely provider-agnostic from day one. Avoiding this contract early would force a painful refactor later.

### Story 4.1 — Define the LLM provider interface
**As a** Platform Engineer
**I want** a single interface that every LLM provider implementation conforms to
**So that** swapping providers is purely configuration, not code change.

**Description**: A provider interface with methods to invoke a single completion: takes (system prompt, user prompt, model config), returns (text response, usage metadata, raw response). Stub implementations for each supported provider; only one fully implemented in v1 (recommend Claude).

**Acceptance Criteria**:
- The interface is documented.
- At least one provider (Claude) is fully implemented and tested against the live API.
- Other providers have stub implementations that raise a clear "not yet implemented" error.

### Story 4.2 — Second provider implementation
**As an** Agent Builder
**I want** to point my agent at a different LLM provider via config
**So that** I can compare results or use the best provider for the task without code changes.

**Description**: Implement one additional provider (OpenAI recommended for broad coverage) end-to-end. This proves the abstraction holds across two distinct API shapes.

**Acceptance Criteria**:
- An agent configured with `provider: openai` runs successfully.
- The same agent definition (other than `provider`) produces a response from both providers.
- The provider differences are entirely encapsulated in their adapter implementations.

---

## Epic 5 — Agent Core Execution

**Business Value**: This is the lean, focused execution kernel that every subsequent phase will reuse. Keeping it clean now means workflow complexity in later phases will not pollute it.

### Story 5.1 — Agent Core abstraction
**As a** Platform Engineer
**I want** a clean function/class that receives a fully-prepared agent context and executes the LLM call
**So that** the orchestration layer never reaches into model-specific logic and the core stays lean.

**Description**: The Agent Core receives a pre-assembled bundle: provider adapter, system prompt, interpolated user prompt, model config (and in later versions, tools and MCPs). It executes the inference and returns the result. It does not know about workflows, state machines, retries, or persistence — those are infrastructure concerns layered around it.

**Acceptance Criteria**:
- The Agent Core has zero dependencies on the database, Temporal, the orchestrator, or HTTP frameworks.
- It is unit-testable in isolation with mock providers.
- All side effects (logging, metrics) are passed in as injected interfaces.

### Story 5.2 — Single-step invocation endpoint
**As an** End User (or upstream service)
**I want** to invoke an agent by ID with an input payload and receive its response
**So that** I can actually use the agents the platform makes available.

**Description**: An HTTP endpoint (e.g. `POST /agents/{id}/invoke`) that: loads the agent definition, creates a `workflow_runs` row, interpolates the user prompt, calls the Agent Core, persists the result to the run, returns the response synchronously. v1 is synchronous; async streaming arrives in v5.

**Acceptance Criteria**:
- The endpoint returns the agent response on success.
- A `workflow_runs` row is created and updated through its full lifecycle on every call.
- Failures return a structured error and persist the failure reason in the run record.

---

## Technical Considerations (high level)

- **Repository structure**: separate modules for `dsl-schema`, `validation`, `providers`, `agent-core`, `persistence`, `api`. The Agent Core must not import from `persistence` or `api`.
- **Provider adapters**: each provider lives behind a small adapter implementing the unified interface. SDK-level differences are encapsulated there.
- **Plain-text secrets**: acceptable in v1, but isolate credential reads through an interface so v6 vault migration is a swap, not a rewrite.
- **No Temporal yet**: v1 runs synchronously in-process. The `workflow_runs` table exists so v2 can layer Temporal on top without schema changes.
- **Logging**: introduce structured logging from day one with a `run_id` field on every log line — this becomes the trace anchor in v6.

## Definition of Done for v1

- An agent can be created via a JSON definition.
- The definition is validated end-to-end.
- The agent can be invoked through the API and returns a real LLM response.
- The invocation is persisted in `workflow_runs` with full input and output.
- At least two LLM providers are working end-to-end.
- The Agent Core is provider-agnostic and unit-tested.

---

## What to Read Next

With v1 complete, the foundation is in place. The next phase adds workflow orchestration:

→ **[03-v2 — Workflow Orchestration](./03-v2-workflow-orchestration.md)** — Multi-step workflows, Temporal integration, and the state machine.

Or explore supporting topics:
- [01-PRD](./01-PRD.md) — Return to the full decision log and requirements
- [08-Performance and Scaling](./08-performance-and-scaling.md) — Understand latency expectations
- [11-Testing Strategy](./11-testing-strategy.md) — Unit testing the Agent Core
