# Agent Platform — Project Requirements Document (PRD)

> **Purpose of this document**: This is the canonical, self-contained brief for the Agent Platform project. It captures the project's mission, every architectural and product decision made during discovery, the rationale behind each decision, the personas involved, the technology stack, what is explicitly out of scope, and a roadmap reference to the phased implementation documents. Drop this document into any new context (a new LLM session, a new team member, a new design discussion) and the reader will have everything they need to engage productively without re-litigating settled decisions.

---

## 1. Executive Summary

We are building an **Agent Provisioning and Orchestration Platform**: a system that allows the declarative creation, configuration, and execution of specialized AI agents. Agents are defined via a structured configuration (DSL), executed durably through Temporal workflows, composed into multi-agent orchestrations, and exposed to end users through a streaming chatbot layer with intelligent routing.

The platform's defining property is **separation of concerns**: the actual LLM inference (the "Agent Core") stays clean and focused, while all infrastructure complexity (workflow orchestration, state management, secrets, versioning, observability) sits around it as composable layers. The DSL is the single contract that ties everything together.

The target outcome is to enable, for example, a website operator to provision a complete customer-support chatbot system in configuration — a unified chat UI backed by a routing chatbot that dispatches to specialist sub-chatbots (payments, support, technical), which in turn delegate complex work to background agent workflows that may call further sub-agents, fetch real-time data via tools and MCP connectors, query a vector knowledge base, and surface results back into the conversation asynchronously — all without writing orchestration code.

---

## 2. Problem Statement

Building production-grade multi-agent AI systems today requires combining many heterogeneous pieces: LLM SDKs, workflow engines, vector databases, MCP connectors, secret management, observability, queueing, retries. Teams end up writing custom orchestration code per use case, hardcoding agent behavior, and reinventing infrastructure primitives. This makes iteration slow, the resulting systems brittle, and the agents themselves indistinguishable from one another.

The problem is that **agent definition and agent execution are entangled with infrastructure plumbing**. A small change to an agent's behavior (a new tool, a different model, a new routing rule) requires code changes, deployments, and regression testing. Multi-agent scenarios amplify this pain because each agent's orchestration is bespoke.

---

## 3. Vision and Value Proposition

A platform where:

- **Agents are declarative artifacts**, defined in a structured configuration that lives in a database and is independent of platform code.
- **Workflow orchestration is dynamic**: a single generic Temporal worker reads agent configurations at runtime and executes them — no code generation, no per-agent deployments.
- **Tools and MCPs are catalogued**: builders compose agent capabilities from registries rather than writing integration code.
- **The Agent Core stays minimal**: receives a fully-prepared bundle (model, prompt, tools, MCPs) and executes inference. All complexity lives outside it.
- **Provider-agnostic by design**: Claude, OpenAI, Gemini, Grok, or any other LLM is selected via configuration. Native tool-calling per provider is preserved.
- **State is durable and auditable**: every workflow run, step, input, and output is persisted in the database. Workers are stateless and resumable.
- **Versioning is immutable**: every agent change produces a new version; in-flight workflows pin to their original version. Operators can iterate freely without breaking production traffic.
- **Operability is first-class**: workflows can be paused, resumed, cancelled, and stepped through manually during development.
- **The chatbot layer delivers the user-facing experience**: streaming, routing, specialist delegation, knowledge base lookup, async background tasks folded back into the conversation.

---

## 4. Target Use Cases

The discovery conversation centered on a representative scenario, but the platform is general:

### Primary scenario (customer support chatbot for a website)

A site operator provisions:

- A **routing chatbot** exposed to end users via a single chat UI.
- Multiple **specialist chatbots** — payments, technical support, administrative, knowledge — each with a focused system prompt, scoped knowledge base filters, and a curated set of tools/MCPs.
- **Backend workflow agents** that handle heavy or long-running work: querying real-time APIs (e.g. hotel room availability), running multi-step processes, calling further sub-agents.
- **Tools and MCPs**: internally-built tools plus third-party MCPs like Linear, Jira, Figma.
- A **vector knowledge base** (Qdrant, already in place) connected per specialist with relevant filters.

A user opens the chat, asks a question. The routing chatbot classifies the intent, dispatches to the right specialist, the specialist may delegate to a background agent for a complex task while the conversation continues, and when the background work completes, the chatbot proactively surfaces the result.

### General applicability

The same platform supports any scenario where structured, multi-step, multi-agent workflows are needed: document processing pipelines, research assistants, ops automation, data enrichment, etc.

---

## 5. Architectural Components

| Component | Responsibility |
|---|---|
| **Provisioning Layer** | Accepts agent DSL definitions, validates, stores versioned in the database. |
| **DSL Schema** | The declarative configuration language. JSON-based in v1; custom domain language planned for future. Describes agent metadata, model provider, tools/MCPs, and workflow steps with full data flow declarations. |
| **Validation Service** | Standalone service. Verifies schema compliance, referenced tools/MCPs exist, data types flow correctly between steps, credentials are configured. Collects all errors, returns them together. |
| **Workflow Interpreter (Temporal worker)** | A single generic Temporal workflow that loads the agent's DSL config at runtime and dispatches each step to an appropriate activity. No code generation; the worker interprets. |
| **Agent Core** | The lean execution kernel. Receives a fully-prepared bundle (provider adapter, system prompt, interpolated user prompt, tools, MCPs). Executes inference, returns the result. Knows nothing about workflows, state, persistence, or retries. |
| **Provider Adapters** | One adapter per LLM provider (Claude, OpenAI, Gemini, Grok, etc.), each conforming to a unified interface. Native tool-calling is preserved per provider; differences encapsulated in the adapter. |
| **Tool Registry** | Catalog of internally-built callable functions. Tools are versioned with platform releases. Looked up by name at runtime. |
| **MCP Registry** | Catalog of MCP connectors (own + third-party). Dynamically registered via configuration. Includes auth strategies and capability discovery. |
| **Credentials Pipeline** | Single abstracted `CredentialsProvider` interface. v1–v5 read plain text from DB; v6 swaps in a vault-backed implementation. Credentials loaded once per workflow run and passed via in-memory context. |
| **State Machine (Database)** | The durable source of truth. `agents` (versioned), `workflow_runs`, `workflow_step_runs`, `conversations`, `conversation_turns`, `conversation_pending_tasks`. Every input/output persisted. |
| **Orchestrator / Monitor** | Background polling service watching workflow run state transitions. Surfaces progress to consumers. Polling chosen over event bus for simplicity (event bus is a later optimization). |
| **Chatbot Layer** | Streaming-capable, real-time conversational entry points. Multiple specialized chatbots behind a unified UI. The routing chatbot classifies each user turn and dispatches to a specialist. |
| **Knowledge Base Integration** | Qdrant-backed vector database, already in place. Exposed as a registered tool with per-specialist scoping (filters, retrieval strategy). |

---

## 6. Architectural Decisions Log

Every decision below is a settled choice from the discovery conversation. The rationale is captured so future readers understand why — not just what.

### AD-01: Temporal as the workflow orchestration backbone
**Decision**: Use Temporal for durable workflow orchestration.
**Rationale**: Temporal provides durability, automatic retries, parallelization, and state management out of the box. Building these from scratch would take months and would still be inferior. LangChain was considered but rejected for orchestration because it runs everything in-process and does not natively support distributed step execution across workers.
**Implication**: Every multi-step agent is a Temporal workflow. Each step is a Temporal activity.

### AD-02: DSL is JSON for MVP; custom domain language later
**Decision**: Configuration is JSON for now. A custom DSL will be designed when real-world usage reveals the right shape.
**Rationale**: JSON gives immediate validation, tooling, and parsing support. Building a custom parser upfront is premature optimization. Once we have real workflows in production, we will know what the DSL needs to express ergonomically.
**Implication**: All agent definitions are JSON documents validated against a JSON Schema.

### AD-03: Dynamic interpretation, not code generation
**Decision**: A single generic Temporal workflow handler reads the DSL configuration at runtime and dispatches each step to the appropriate activity. No per-agent code is generated, compiled, or deployed.
**Rationale**: Code generation adds a deploy step to every agent change, breaks the "agent as configuration" mental model, and complicates versioning. Dynamic interpretation keeps the platform agile.
**Implication**: Workers are generic. They load configurations from the database at workflow start.

### AD-04: Database is the source of truth for state machine
**Decision**: Every workflow run, every step execution, every input/output is persisted in the application database. Temporal manages workflow execution state internally, but the application's product-visible state lives in our DB.
**Rationale**: This decouples the orchestrator from Temporal's internal storage, enables straightforward querying, gives full audit history, and makes the state machine readable independent of Temporal. The Operator clarified that this also gives easy async behavior, retries, and analytical data for the future.
**Implication**: All step boundaries write to the database. The two systems (Temporal + app DB) coexist; do not treat Temporal history as a product API.

### AD-05: Polling-based orchestrator (no event bus yet)
**Decision**: A background process polls the database periodically to detect workflow state transitions. No message bus (RabbitMQ, Kafka, etc.) in MVP.
**Rationale**: The Operator clarified that polling is sufficient and simpler. The DB-as-state approach explicitly avoids dependencies on a separate broker. Polling latency is acceptable; if event-driven becomes necessary later, it is an additive change.
**Implication**: Latency between step completion and orchestrator action is bounded by the polling interval (configurable).

### AD-06: Job-based execution per Temporal workflow run (not standing pods)
**Decision**: Workflows are spawned on demand and terminate when complete. No persistent per-agent pods.
**Rationale**: The Operator initially asked about controller-watched persistent pods vs. job-based execution and chose job-based for resource efficiency. Temporal then refines this: workers are deployed as Kubernetes services, but workflow execution is ephemeral — a workflow starts, runs its steps across available workers, and ends.
**Implication**: Workers can be horizontally scaled per workload. No per-agent infrastructure to provision.

### AD-07: Plain-text credentials in v1–v5; vault migration in v6
**Decision**: For the MVP, secrets and API keys are stored as plain text in the database. A proper secrets vault (HashiCorp Vault, AWS Secrets Manager, or equivalent) is introduced in v6.
**Rationale**: The Operator explicitly chose to defer vault integration to keep early phases lean. The risk is acknowledged. The mitigation: design the `CredentialsProvider` interface from v3 so the v6 migration is a single-implementation swap.
**Implication**: All credential reads go through an abstracted interface from day one. No direct DB credential access from outside the credentials module.

### AD-08: Credentials loaded once per workflow run via in-memory context
**Decision**: When a workflow starts, the entry point fetches all credentials needed for that agent's tools and MCPs, packages them into an in-memory context object, and passes that context through every step. No per-step DB lookups for credentials.
**Rationale**: Avoids hammering the DB across multi-step workflows and creates a clean point for the v6 vault swap. The Operator described this as one of the explicit early design choices.
**Implication**: Steps receive the credentials context as input. The context is not persisted beyond the workflow's lifetime.

### AD-09: Provider-agnostic LLM interface with native tool-calling preserved
**Decision**: All LLM providers (Claude, OpenAI, Gemini, Grok, others) conform to a single internal interface. Tool-calling uses each provider's native format (Claude's tool_use blocks, OpenAI's function calling, etc.) translated at the adapter layer.
**Rationale**: The Operator explicitly rejected reinventing a cross-provider tool-calling format. Native formats give the best per-provider behavior and let us leverage upstream improvements directly.
**Implication**: Each provider adapter encapsulates its own tool format. The Agent Core sees a uniform tool/MCP interface.

### AD-10: Agent Core stays minimal — infrastructure does not leak in
**Decision**: The Agent Core's only responsibility is to invoke the LLM with a prepared bundle and return the result. It has zero dependencies on Temporal, the database, the HTTP layer, or the orchestrator.
**Rationale**: The Operator was emphatic on this. The infrastructure around the Agent Core may evolve significantly; the Core must stay simple, testable in isolation, and free of orchestration concerns. This is the design principle that keeps complexity manageable.
**Implication**: The Agent Core is unit-testable with mock providers. All side effects (logging, metrics, persistence) are injected as interfaces.

### AD-11: Tools are code-registered; MCPs are configuration-registered
**Decision**: Tools live in the platform's codebase and self-register on startup. MCPs are registered dynamically via configuration (so new MCPs can be added without redeployment).
**Rationale**: Tools are tightly coupled to platform releases. MCPs are by nature external integrations that need to be added and removed flexibly.
**Implication**: Different lifecycles for the two. Both consumed identically by agents.

### AD-12: Validation is a separate service
**Decision**: The validation logic is its own service, callable both at provisioning time and as a standalone endpoint.
**Rationale**: The Operator chose separation so validation can be reused (e.g. by a future UI's "validate before save" feature) and remains testable in isolation.
**Implication**: The validation service is consumed by the provisioning API and any future tooling.

### AD-13: Validation collects errors; does not fail fast
**Decision**: When a DSL document is validated, all errors are gathered and returned together rather than aborting at the first problem.
**Rationale**: The Operator wants the experience of "try, accept the errors, fix what's wrong" — fast feedback on the full set of problems, not iterative one-error-at-a-time discovery.
**Implication**: Validators are designed to accumulate errors and continue scanning.

### AD-14: Retry policy declared per step in the DSL
**Decision**: Each workflow step declares its own retry policy (max attempts, backoff, non-retryable errors). No global retry rule.
**Rationale**: Different steps have different reliability profiles. LLM calls might benefit from exponential backoff; transient HTTP calls might need fast retry; transformations should not retry at all.
**Implication**: The DSL includes a `retry` block per step. Defaults are documented for steps that omit it.

### AD-15: Sub-agent invocation can be sync or fire-and-forget (declared per call)
**Decision**: The `agent_call` step type accepts a `mode` field: `wait` (parent blocks until child completes) or `fire_and_forget` (parent immediately proceeds).
**Rationale**: Both patterns are needed. Some compositions require the child's output; others kick off background work that the parent does not need to wait on.
**Implication**: The DSL supports both modes. Each child workflow is independently tracked in the DB with a parent linkage.

### AD-16: Versioning is immutable
**Decision**: Every change to an agent produces a new immutable version. Workflow runs persist the exact version they were launched with. Old versions remain available; deprecated versions can be marked but not deleted while runs reference them.
**Rationale**: The Operator explicitly chose this over in-place updates. It prevents in-flight workflows from breaking when an agent definition changes and gives full audit history.
**Implication**: An `agent_versions` table stores every version. A `latest` pointer per agent serves new traffic without explicit versioning.

### AD-17: Workflows can pause, resume, cancel, and step-through-debug
**Decision**: Operators can pause running workflows at safe checkpoints (between steps), resume them with optional input override, cancel them with cleanup, and run them in step-by-step debug mode where each step transition requires manual approval.
**Rationale**: The Operator wants two distinct capabilities: pause/resume for production intervention, and step-by-step manual control for development iteration. Both are built on Temporal's native signal/query primitives.
**Implication**: Debug mode is a flag on workflow start. Pause/resume are signals delivered at runtime.

### AD-18: Knowledge base is Qdrant, already in place, exposed as a registered tool
**Decision**: The Qdrant vector database is already operational with multiple retrieval strategies tested (LlamaIndex semantic, hybrid semantic). It will be exposed to agents as a registered tool/capability, with per-specialist filter scoping.
**Rationale**: The Operator confirmed this infrastructure already exists. The platform's job is to integrate it cleanly, not rebuild it.
**Implication**: Knowledge base lookups are first-class capabilities available to any agent or chatbot via the tool registry.

### AD-19: Chatbots are a distinct agent type with synchronous streaming
**Decision**: Chatbots run synchronously and stream responses; they are NOT executed as Temporal workflows per turn. The conversation itself is the long-lived unit. When chatbots need heavy work, they delegate to backend (workflow) agents.
**Rationale**: Real-time streaming and Temporal workflow execution have fundamentally different latency profiles. Forcing chatbots through Temporal would harm the user experience. The Operator confirmed chatbots are an exception to the workflow pattern.
**Implication**: Two execution paths: synchronous (chatbot) and asynchronous (workflow agent). Chatbots are still defined via DSL, still versioned, still use the registries — they are a distinct agent type within the same platform.

### AD-20: Multiple specialized chatbots behind a unified UI
**Decision**: The end user sees one chat window. Behind it, a routing chatbot classifies each user turn and dispatches to a specialist chatbot (payments, technical support, administrative, etc.). Specialists may delegate to backend workflow agents.
**Rationale**: The Operator's explicit design — gives a unified user experience while allowing each specialist to be tuned (system prompt, KB filters, tool access) for its domain. Per-turn routing means topic changes are handled seamlessly.
**Implication**: Multiple internal endpoints, one external surface. The conversation context flows across specialists.

### AD-21: Background task surfacing is proactive
**Decision**: When a background workflow that was kicked off during a conversation completes, the chatbot proactively surfaces the result on the next turn (or via push if streaming is connected), referencing the original request.
**Rationale**: Avoids the "is it ready yet?" UX. The Operator described this as a signature feature.
**Implication**: Conversations track pending tasks. The orchestrator notifies the conversation layer on completion. Each pending task has a "recall context" — the conversational snippet to inject when surfacing.

### AD-22: LangChain may be used inside Temporal activities, but not for orchestration
**Decision**: If LangChain (or LangGraph, LlamaIndex, etc.) is useful for specific agent logic inside a single step, it can be used there. It is NOT used as the workflow orchestration layer.
**Rationale**: LangChain is in-process and does not distribute step execution. Temporal does. The Operator agreed they are complementary at the right layer, not substitutable.
**Implication**: Activity implementations are free to use any framework internally as long as they conform to the activity contract.

### AD-23: LangFuse-style LLM observability deferred
**Decision**: Prompt and LLM-call observability (LangFuse or equivalent) is deferred to a later phase. v6 lands an interface in place but ships a no-op default.
**Rationale**: The Operator wants foundational observability (Prometheus, tracing) before specialized LLM observability. Keeping the hook in place from v6 means the integration is a swap, not a refactor.
**Implication**: Provider adapters call an `LLMObservabilityClient` interface on every invocation. The default does nothing; a real implementation can be added later.

### AD-24: Standard observability stack from v6
**Decision**: Prometheus metrics, OpenTelemetry distributed tracing, structured logging with trace IDs end to end.
**Rationale**: These are the operational primitives. The Operator confirmed Prometheus and trace IDs are the right starting point.
**Implication**: Every workflow run, step, and external call carries a trace ID. Metrics are exported at `/metrics`.

### AD-25: No UI in early phases
**Decision**: Provisioning is API-driven through v5. UI work is deferred.
**Rationale**: The Operator wants to focus on the platform itself first. The UI can be built later against the stable provisioning API.
**Implication**: All capabilities are documented as APIs. UI mocks and design are not part of the early roadmap. A minimal Operator CLI ships in v6 (see `17-operator-cli.md`) to cover the Operator persona's operational needs before the full UI is built.

### AD-26: Namespace-based multi-tenancy from v1
**Decision**: The platform is multi-tenant by design. Every resource (agents, workflow runs, credentials, conversations) belongs to a `namespace`. All database tables include `namespace_id` from v1. All API endpoints enforce namespace-scoped authorization. Cross-namespace access is prohibited by default.
**Rationale**: The platform's value proposition is enabling multiple operators to provision independent agent ecosystems. Retrofitting multi-tenancy after v6 would require breaking schema migrations and API changes. Adding `namespace_id` to v1's schema costs almost nothing but prevents a catastrophic migration later. A `DEFAULT` namespace allows single-tenant deployments to use the full platform transparently.
**Implication**: All DSL documents include a required `namespace` field. API tokens are scoped to a namespace. Platform Admins hold a super-token spanning all namespaces. Full multi-tenancy specification is in `16-multi-tenancy.md`.

### AD-27: Backend implementation language — TypeScript or Python, decided before end of v1
**Decision**: The backend implementation language was not chosen during discovery. It must be decided before v1 implementation begins. Two viable options are TypeScript (Node.js) and Python. The decision gate is end of v1 planning.
**Rationale**: The language choice affects: Temporal client SDK maturity, JSON Schema validation library ecosystem, streaming SSE/WebSocket ergonomics, LLM provider SDK availability, and the `agent-platform` CLI implementation. Deferring past v1 planning creates speculative work and diverging prototypes.
**Implication**: The decision will be recorded here as an update to this document. Both TypeScript and Python have mature Temporal SDKs and LLM provider SDKs. The choice should be made based on team proficiency and existing codebase conventions.

### AD-28: Provider rate limiting handled by the platform, not individual activities
**Decision**: LLM provider rate limits (429 responses, `Retry-After` headers) are managed by a platform-level per-provider token bucket and error classifier, not by each activity independently.
**Rationale**: Independent per-activity retry of rate limit errors causes thundering-herd re-triggering. A shared token bucket serializes concurrent requests before they reach the provider, preventing the thundering herd at the source. `Retry-After` header values are honored as the authoritative retry signal.
**Implication**: Provider adapters must classify `RateLimitError` vs. `QuotaExceededError`. A token bucket is deployed per-provider per-namespace. The DSL retry block gains `rate_limit_max_attempts` and `rate_limit_honor_retry_after` fields. Full specification in `19-provider-rate-limiting.md`.

### AD-29: Agent version promotion is gated (draft → staging → canary → production)
**Decision**: Creating a new agent version does not immediately update the `latest` pointer. Versions move through explicit promotion states before serving production traffic.
**Rationale**: The existing model (create version → auto-update `latest`) has no staging gate, no canary safety, and no rollback trigger. For production platforms this is unsafe. The promotion model adds safety without changing the immutable versioning contract (AD-16).
**Implication**: The `agent_versions` table gains a `state` column. The provisioning API no longer auto-promotes on create. Canary traffic splitting and auto-rollback are built on namespace-level configuration. Full specification in `20-agent-version-promotion.md`.

### AD-30: Conversation memory is three tiers: working, episodic, semantic
**Decision**: Conversation memory is explicitly modeled as three distinct tiers with separate storage, retrieval, and lifecycle policies: working memory (active conversation turns), episodic memory (past session summaries in Qdrant), and semantic memory (structured user profile facts in the application database).
**Rationale**: Conflating all memory into "context" leads to gaps: working memory is already handled by the context window strategy (v5 Epic 6), but users expect the system to recall prior conversations and know their account tier. These are fundamentally different storage and retrieval problems. Modeling them separately gives Agent Builders precise control over which memory tier each specialist uses.
**Implication**: A `user_profiles` table is added to the schema. An episodic collection is created in Qdrant per namespace. Two new registered tools: `user_profile_lookup`, `user_profile_update`. Full specification in `21-conversation-memory.md`.

### AD-31: Chatbot layer uses circuit breakers for specialist and KB availability
**Decision**: Each specialist chatbot and the knowledge base tool have independent circuit breakers that open on repeated failures, preventing cascading failure across the chatbot system.
**Rationale**: Without circuit breakers, a flapping specialist causes the routing chatbot to exhaust retries on every affected request, starving healthy specialists of capacity. A circuit breaker decouples specialist health from routing chatbot health. The routing chatbot has a configurable `default_specialist` fallback and `static_fallback_message` for full-degraded scenarios.
**Implication**: Circuit breaker state is tracked in an in-process registry per namespace, queryable via API and CLI. New Prometheus metrics track circuit state. Full specification in `22-chatbot-degradation.md`.

### AD-32: Per-run token/cost cap enforced at the Temporal workflow level
**Decision**: Workflow DSL definitions may declare a `budget` block specifying `max_total_tokens` and/or `max_estimated_cost_usd`. The Temporal workflow enforces this budget after each `llm_call` step completes, cancelling the run if exceeded.
**Rationale**: Existing guards (loop `max_iterations`, parallel `max_concurrency`) prevent structural runaway but not token runaway within structurally valid bounds. A single while loop iterating within its limit can still consume unbounded tokens if each iteration runs a large LLM call. A cost cap is the correct safety valve at the run level, orthogonal to the namespace-level daily quota (E-18).
**Implication**: New `budget` top-level DSL field. Platform price table configurable by Platform Admins. `on_exceeded: cancel | warn_and_continue | cancel_with_partial_output` actions. Namespace-level default budget applies to agents without explicit `budget` declaration. Full specification in `23-per-run-cost-cap.md`.

### AD-33: Routing chatbot confidence threshold and low-confidence action (E-14)
**Decision**: The routing chatbot's classification is LLM-based. When the classification `confidence` score falls below a configurable threshold (default: `0.5`), the routing chatbot applies a declared `low_confidence_action`:
- `ask_clarification`: sends a clarifying question back to the user and awaits the next turn.
- `route_to_default`: immediately delegates to the configured `default_specialist`.

The threshold and action are declared in the routing chatbot's DSL `routing` block. There is **no hard-coded fallback model** — the routing chatbot is the only classification layer.

**Rationale**: A single LLM routing layer is sufficient for the described use case. A secondary classifier or ML-based scoring system would add complexity without clear benefit at this scale. The clarification loop prevents silent misrouting at the cost of an extra user interaction.

**Implication**: The `routing` block in the chatbot DSL (see `14-dsl-spec.md` §Chatbot Agent Type) requires `confidence_threshold` and `low_confidence_action`. The routing chatbot's `outputs` declaration must include a `confidence: number` field. The routing activity validates that `confidence` is present in the classification output before applying the threshold check.

---

## 7. Product Decisions Log

### PD-01: The platform's primary deliverable is "agent provisioning" — not bespoke agents
**Decision**: The platform is a meta-product. Its users (Agent Builders, Platform Admins) configure agents declaratively. The platform itself does not ship pre-built agents (beyond reference examples).
**Rationale**: This is what makes the platform reusable across use cases and customer scenarios. Building bespoke agents would tie the platform to specific verticals.

### PD-02: Customer support is the first concrete use case
**Decision**: The first real-world scenario the platform is validated against is a customer-support chatbot for a website, with routing across specialist domains and a knowledge base.
**Rationale**: This was the use case the Operator described in discovery as the value proposition.

### PD-03: Persona scope
**Decision**: Four personas are recognized:
- **Platform Admin**: configures the platform itself (tools, MCPs, global settings).
- **Agent Builder**: defines new agents via DSL.
- **End User**: interacts with chatbots and agent-driven experiences.
- **Operator**: monitors, debugs, manages versions, runs step-by-step debug.

### PD-04: Phasing prioritizes infrastructure foundation before user-facing features
**Decision**: v1–v4 build the agent provisioning, workflow orchestration, tools/MCPs, and multi-agent layers. v5 delivers the chatbot layer. v6 hardens for production.
**Rationale**: The chatbot layer (the user-facing piece) consumes the agent platform. Building chatbots before the platform would mean inventing throwaway infrastructure.

### PD-05: Build over buy at the architectural level; assemble at the component level
**Decision**: The platform itself is bespoke (no off-the-shelf solution combines DSL-driven dynamic workflows, multi-provider, multi-agent, chatbot routing, KB scoping, versioning, and debug). The components beneath it are battle-tested off-the-shelf (Temporal, Qdrant, Prometheus, vault, etc.).
**Rationale**: The Operator asked "am I reinventing the wheel?" — the answer was no, because the combination is bespoke, but the pieces beneath are not.

### PD-06: Validation feedback is "collect all, present once"
**Decision**: When an Agent Builder submits a DSL definition, every error is collected and returned in one response.
**Rationale**: Maximizes the speed of iteration. The Operator described this as the desired experience.

### PD-07: Versions evolve forward; deprecated versions retire gradually
**Decision**: Old versions are not deleted while runs reference them. Deprecation is a soft signal first, then a hard block once callers have migrated.
**Rationale**: Safety. Operators decide when to retire versions based on actual usage.

---

## 8. Technology Stack

| Layer | Technology | Status |
|---|---|---|
| Workflow orchestration | Temporal | Chosen (AD-01) |
| Vector database | Qdrant | Already in place (AD-18) |
| Application database | TBD (Postgres recommended) | To be confirmed |
| LLM providers | Claude, OpenAI, Gemini, Grok, others | All supported (AD-09) |
| Container orchestration | Kubernetes / Docker | Standard deployment target |
| Secrets management | Plain DB (v1–v5) → Vault (v6) | Migration planned (AD-07) |
| Observability — metrics | Prometheus | Chosen (AD-24) |
| Observability — tracing | OpenTelemetry | Chosen (AD-24) |
| Observability — LLM | LangFuse or equivalent | Interface in v6; integration deferred (AD-23) |
| Inter-agent comms | DB state machine (polling) | Chosen (AD-04, AD-05) |
| MCP transport | HTTP, stdio (WebSocket if needed) | Standard MCP protocol |
| Backend implementation language | TBD | Open |
| Frontend (chatbot UI) | TBD | Out of scope for early phases (AD-25) |

---

## 9. DSL Concept (high-level)

The DSL evolves across phases. The full schema lives in the version documents; the conceptual structure is summarized here.

### Agent definition (v1)
```
{
  "name": "string",
  "description": "string",
  "system_prompt": "string",
  "user_prompt_template": "string with {{interpolation}}",
  "provider": "claude | openai | gemini | grok | other",
  "model_config": { "model_name": "...", "temperature": ..., "max_tokens": ... }
}
```

### Workflow extension (v2)
```
{
  ...v1 fields...,
  "workflow": {
    "steps": [
      {
        "id": "step_name",
        "type": "llm_call | transform | fetch | noop",
        "inputs": { ...mapping from prior outputs or workflow input... },
        "outputs": { ...declared output shape... },
        "next": "next_step_id",
        "retry": { "max_attempts": N, "initial_interval": ..., "backoff_coefficient": ..., "non_retryable_errors": [...] }
      }
    ]
  }
}
```

### Tools and MCPs (v3)
- New step types: `tool_call`, `mcp_call`.
- `llm_call` step extended with `tools` and `mcps` arrays referencing registry entries by name, enabling native provider tool-calling.

### Multi-agent and control flow (v4)
- New step types: `agent_call` (with `mode: wait | fire_and_forget`, optional `version`), `parallel`, `for_each` (with concurrency limit), `branch` (if/else), `switch`, `while`, `until`, `transform`.
- Inline input mapping expressions for declarative data flow without LLM transformations.

### Chatbot type (v5)
- Chatbots are a distinct agent type. They use the same DSL machinery but run synchronously, stream responses, and delegate heavy work to workflow agents via `agent_call`.

### Versioning (v6)
- Versions are recorded server-side. The DSL itself does not change; the lifecycle around it does.

---

## 10. Out of Scope (intentionally deferred)

| Item | Reason | When |
|---|---|---|
| Custom DSL parser/compiler | JSON sufficient until real usage reveals friction | Future |
| LangFuse-style LLM observability integration | Foundational observability first; interface in place v6 | Post-v6 |
| Provisioning UI | API-first; UI design depends on stable APIs | Post-v6 |
| Encrypted secrets vault | Plain text acceptable for MVP; interface abstracted | v6 |
| Event-driven orchestrator (message bus) | Polling is sufficient | Future optimization |
| Human-handoff ticketing integration | Fallback documented; integration deferred | Future |
| WebSocket MCP transport (if no MCP needs it) | HTTP and stdio cover known MCPs | As needed |
| Cross-provider unified tool-calling format | Use each provider's native format | Permanent decision |
| Per-agent persistent pods | Job-based execution chosen | Permanent decision |

---

## 11. Open Questions and Deferred Details

These were noted during discovery and explicitly parked for later:

- **Conversation state tracking for background tasks**: how exactly the conversation's pending-tasks list integrates with the state machine. Outlined in v5; details to refine during implementation. Note: the poll interval for conversation-linked task surfacing is separately configurable (see v5 Epic 5).
- **Specific Kubernetes deployment topology**: number of worker pools, task queue segmentation, autoscaling triggers. Defer to v3+ once load patterns are understood.
- **Backend implementation language**: not chosen during discovery. Must be decided before end of v1 planning (AD-27).
- **Conversation resume across sessions**: persistence is in scope (v5); the exact reconnect protocol is implementation detail.
- **OAuth flow UX for MCP authentication**: stub in v3, refine in v5 if chatbot UX intersects with auth.
- **Cycle detection for sub-agent recursion**: addressed in the DSL specification (`14-dsl-spec.md`). Default maximum call depth: 10. Enforced by the validation service.
- **`for_each` with `agent_call` body**: fully supported and specified. Each item spawns an independent child workflow. `mode: wait` blocks until all complete; `mode: fire_and_forget` dispatches all and proceeds. Full semantics in `14-dsl-spec.md` and `05-v4`.
- **Conversation context window management**: addressed in v5 Epic 6. Three strategies: `sliding_window`, `summarize`, `semantic`. Cross-specialist handoff context is a summary, not the full transcript.
- **Human handoff integration**: addressed in v5 Epic 7 as a configurable interface (`message`, `webhook`, `ticket` strategies). Full ticketing MCP integration remains deferred but the interface is defined and the webhook stub is production-usable.

---

## 12. Phased Roadmap

Each phase produces a working subset of the platform that delivers value end to end. Each phase builds on prior phases without invalidating earlier work.

| Version | Theme | What lands | Document |
|---|---|---|---|
| **v1** | Agent Provisioning Foundation | DSL v1 (single agent, single LLM call), validation, provider-agnostic interface, lean Agent Core, sync invocation endpoint, state machine persistence skeleton | [02-v1-agent-provisioning-foundation.md](./02-v1-agent-provisioning-foundation.md) |
| **v2** | Workflow Orchestration | DSL v2 (multi-step workflows with data flow, retries), Temporal worker, per-step persistence, polling orchestrator/monitor | [03-v2-workflow-orchestration.md](./03-v2-workflow-orchestration.md) |
| **v3** | Tools and MCPs Registry | Tool registry (code-registered), MCP registry (config-registered, own + third-party), credentials pipeline abstracted, native provider tool-calling in `llm_call` | [04-v3-tools-and-mcps.md](./04-v3-tools-and-mcps.md) |
| **v4** | Multi-Agent Orchestration | `agent_call` (sync/fire-and-forget, versioned), parallel and fan-out, branching (if/else, switch), loops (while, until), explicit data mapping | [05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md) |
| **v5** | Chatbot Layer | Streaming endpoints, routing chatbot, specialist chatbots, KB integration via tool registry, async background tasks with proactive surfacing | [06-v5-chatbot-layer.md](./06-v5-chatbot-layer.md) |
| **v6** | Versioning, Observability, Hardening | Immutable agent versioning, pause/resume/cancel, step-by-step debug mode, trace IDs end-to-end, Prometheus + OTel, LLM observability hook, vault migration | [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) |

The [README.md](./README.md) document gives a single-page summary of the phasing. Supporting documents: [Performance](./08-performance-and-scaling.md), [Security](./09-security-model.md), [Deployment](./10-deployment-concepts.md), [Testing](./11-testing-strategy.md), [Upgrades](./12-upgrades-and-migrations.md), [Chatbot UX](./13-chatbot-ux-edge-cases.md).

Enhancement and gap-closure documents: [Provider Rate Limiting](./19-provider-rate-limiting.md), [Version Promotion](./20-agent-version-promotion.md), [Conversation Memory](./21-conversation-memory.md), [Chatbot Degradation](./22-chatbot-degradation.md), [Per-Run Cost Cap](./23-per-run-cost-cap.md), [DSL Migration Tooling](./24-dsl-migration-tooling.md), [Shared Agent Catalogue](./25-shared-agent-catalogue.md).

---

## 13. How to Use This Document

### For a new LLM context window
Paste this PRD at the start of the conversation. The LLM now has:
- The mission and value proposition.
- Every architectural decision and its rationale.
- Every product decision and its rationale.
- The technology stack and what is open.
- The phased roadmap with links to detail.

The LLM can then engage productively on any specific question — designing a particular feature, debugging an architectural choice, drafting code for a specific component — without you re-explaining the project.

### For a new team member
This is the canonical onboarding read. Follow with the version documents for the specific phase they will work on.

### For design discussions
Reference decision IDs (AD-XX, PD-XX) to anchor the conversation. If a discussion is about revisiting a settled decision, that is fine — but explicit, and the decision log gets updated.

### For roadmap planning
The phased structure is the unit of planning. Each version has a clear "Definition of Done" in its document. Estimates and resourcing happen at the version level.

---

## 14. Decision Log Index (quick reference)

**Architectural (AD)**
- AD-01 Temporal as workflow backbone
- AD-02 JSON DSL for MVP, custom domain later
- AD-03 Dynamic interpretation, no code generation
- AD-04 Database as state machine source of truth
- AD-05 Polling-based orchestrator, no event bus
- AD-06 Job-based workflow execution, no standing pods
- AD-07 Plain-text credentials v1–5, vault in v6
- AD-08 Credentials loaded once per workflow via in-memory context
- AD-09 Provider-agnostic LLM interface with native tool-calling preserved
- AD-10 Agent Core stays minimal
- AD-11 Tools are code-registered; MCPs are config-registered
- AD-12 Validation is a separate service
- AD-13 Validation collects all errors, no fail-fast
- AD-14 Retry policy declared per step in DSL
- AD-15 Sub-agent invocation: sync or fire-and-forget
- AD-16 Versioning is immutable
- AD-17 Workflows: pause/resume/cancel/step-debug
- AD-18 Qdrant KB integrated as registered tool
- AD-19 Chatbots: synchronous streaming, distinct agent type
- AD-20 Multiple specialist chatbots behind unified UI
- AD-21 Background task surfacing is proactive
- AD-22 LangChain usable inside activities, not for orchestration
- AD-23 LangFuse-style observability deferred (interface in v6)
- AD-24 Prometheus + OpenTelemetry observability stack
- AD-25 No UI in early phases; Operator CLI ships in v6
- AD-26 Namespace-based multi-tenancy from v1
- AD-27 Backend language: TypeScript or Python; decided before end of v1 planning
- AD-28 Provider rate limiting: platform-level token bucket, not per-activity retry
- AD-29 Version promotion gated: draft → staging → canary → production
- AD-30 Conversation memory: three tiers (working, episodic, semantic)
- AD-31 Chatbot layer uses circuit breakers for specialist and KB availability
- AD-32 Per-run token/cost cap enforced at Temporal workflow level

**Product (PD)**
- PD-01 Platform's deliverable is provisioning, not bespoke agents
- PD-02 Customer support is the first validation use case
- PD-03 Four personas: Platform Admin, Agent Builder, End User, Operator
- PD-04 Phasing: infrastructure before user-facing features
- PD-05 Build at architectural level; assemble at component level
- PD-06 Validation feedback: collect all, present once
- PD-07 Versions retire gradually via soft deprecation

---

## What to Read Next

After the PRD, proceed to the phased implementation documents:

→ **[02-v1 — Agent Provisioning Foundation](./02-v1-agent-provisioning-foundation.md)** — The first implementation phase covering single-step agents, the DSL foundation, and the provider-agnostic Agent Core.

Or jump to a specific topic:
- [08-Performance and Scaling](./08-performance-and-scaling.md) — Latency budgets and capacity planning
- [09-Security Model](./09-security-model.md) — Trust boundaries and threat model
- [11-Testing Strategy](./11-testing-strategy.md) — Quality gates and testing layers
- [14-DSL Specification](./14-dsl-spec.md) — Complete DSL reference with annotated examples
- [15-Developer Tooling](./15-developer-tooling.md) — Local CLI for Agent Builders
- [16-Multi-Tenancy](./16-multi-tenancy.md) — Namespace model, schema impact, API design
- [17-Operator CLI](./17-operator-cli.md) — Terminal interface for Operators (v6)
