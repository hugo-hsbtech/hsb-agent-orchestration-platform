# v3 — Tools and MCPs Registry

## Version Goal

Give agents the ability to **act on the world**: invoke internally-built tools (functions) and external MCP connectors (own + third-party like Jira, Linear, Figma), through a unified abstraction. Both are catalogued in registries, authenticated through a clean credentials pipeline, and surfaced to the Agent Core without it having to know whether a capability is a local function or a remote MCP.

## Business Value

- Agents become genuinely useful: they can fetch real data, trigger actions, and orchestrate operations across external systems.
- Builders compose agent capabilities from a curated catalog instead of writing integration code per agent.
- The provider's native tool-calling format (Claude tool use, OpenAI function calling, Gemini, etc.) is leveraged — no reinvention.

---

## Epic 1 — Tool Registry

**Business Value**: A central catalogue of internally-built tools, available to any agent via configuration, eliminates per-agent integration work.

### Story 1.1 — Tool definition contract
**As a** Platform Engineer
**I want** a clear contract for what a tool is
**So that** any team can build tools that work consistently with the platform.

**Description**: Define a Tool interface: `name`, `description`, `input_schema` (JSON Schema), `output_schema`, `handler` (callable). The handler receives validated inputs and the workflow context, returns a structured output.

**Acceptance Criteria**:
- The interface is documented with examples.
- A starter set of 2–3 tools is implemented as reference.
- Tools are independently unit-testable.

### Story 1.2 — Tool registry service
**As a** Platform Admin
**I want** a registry that catalogues all available tools
**So that** agent builders can discover and reference them.

**Description**: A registry holding all tools, populated at startup. Exposes lookup by name, listing, and metadata query. Backed by either code-side registration (decorators) or database registration; v3 starts with code-side registration to keep it simple.

**Acceptance Criteria**:
- Tools register themselves on worker startup.
- A `GET /tools` API returns the full catalogue with schemas.
- An unknown tool reference in a DSL fails validation with a clear error.

### Story 1.3 — Tool invocation as a workflow step type
**As an** Agent Builder
**I want** to call any registered tool from my workflow
**So that** I can compose agent behavior from cataloged capabilities.

**Description**: Add a `tool_call` step type to the DSL. The step references a tool by name and provides inputs mapped from prior steps or workflow input. The activity validates inputs against the tool's schema, invokes the handler, persists results.

**Acceptance Criteria**:
- A workflow can include a `tool_call` step that successfully invokes a registered tool.
- Invalid inputs fail before the handler runs, with structured error output.
- Tool outputs are available to downstream steps via the data flow declared in v2.

---

## Epic 2 — MCP Registry

**Business Value**: Brings the MCP ecosystem (own + third-party) into the platform under a single, consistent interface.

### Story 2.1 — MCP definition contract
**As a** Platform Engineer
**I want** a clear contract for registering an MCP connector
**So that** both internally-built MCPs and third-party connectors are integrated identically.

**Description**: An MCP entry records: `name`, `transport` (HTTP/WS/stdio), `endpoint`, `auth_strategy` (api_key, oauth, basic, none, etc.), `tool_discovery_mode` (manifest URL or runtime introspection). The platform connects on demand, lists available tools from the MCP, and surfaces them as agent-callable capabilities.

**Acceptance Criteria**:
- The contract is documented with at least one own MCP and one third-party example (e.g. Linear).
- An MCP can be registered via configuration without code changes.

### Story 2.2 — MCP authentication abstraction
**As an** Agent Builder
**I want** the platform to handle MCP authentication for me
**So that** my agent definitions don't carry credentials.

**Description**: Each MCP registration declares its auth strategy. The platform stores credentials (still plain text in v3, vault in v6) and injects them into MCP calls. The DSL references the MCP by name; credentials are resolved through the workflow context built in v2.

**Acceptance Criteria**:
- Agent definitions contain no raw credentials, only MCP references.
- Credentials are loaded once per workflow run via the context mechanism.
- A misconfigured MCP fails fast at provisioning time, not at execution.

### Story 2.3 — Discover MCP tools at runtime
**As a** Platform Engineer
**I want** to list available tools/methods of an MCP at runtime
**So that** agent definitions can validate references to MCP capabilities without hardcoding lists.

**Description**: For each MCP, the platform queries it (at startup, periodically, or on demand) for its tool/capability manifest. The manifest is cached and used during DSL validation.

**Acceptance Criteria**:
- The platform retrieves and caches manifests for each registered MCP.
- A DSL referencing an unknown MCP capability fails validation with a clear message.
- The cache invalidates appropriately when an MCP's manifest changes.

### Story 2.4 — MCP invocation as a workflow step type
**As an** Agent Builder
**I want** to call any MCP capability from my workflow
**So that** I can act on external systems declaratively.

**Description**: Add an `mcp_call` step type to the DSL. The step references the MCP by name and the capability by name, with inputs mapped per the v2 data flow rules. The activity invokes the MCP, persists the result, propagates errors.

**Acceptance Criteria**:
- A workflow can call at least one own MCP and one third-party MCP successfully.
- Auth failures, timeouts, and protocol errors produce structured, persisted errors.

---

## Epic 3 — Native Tool-Calling Integration in the LLM Step

**Business Value**: Lets the LLM itself decide which tools/MCPs to invoke during inference, leveraging each provider's native tool-calling capability rather than orchestrating tool calls only via explicit DSL steps.

### Story 3.1 — Expose tools/MCPs to the LLM provider
**As an** Agent Builder
**I want** my `llm_call` step to declare which tools/MCPs the LLM may invoke
**So that** the model can autonomously orchestrate them within a single step.

**Description**: Extend the `llm_call` step with a `tools` and `mcps` field listing names from the registries. The provider adapter translates these into the provider's native tool-calling format (Claude tool_use blocks, OpenAI function calling, etc.).

**Acceptance Criteria**:
- An `llm_call` step with declared tools makes them available to the model.
- The model's tool-call requests are routed to the registry handlers.
- Multi-turn tool-use loops (model calls tool, gets result, decides next step) work within a single step.

### Story 3.2 — Tool-call results in workflow run records
**As an** Operator
**I want** every tool call (whether explicit step or model-invoked) to be persisted
**So that** I can fully audit agent behavior.

**Description**: Tool/MCP invocations made by the model during an `llm_call` step are recorded as sub-entries under that step, with their inputs, outputs, and timing.

**Acceptance Criteria**:
- The workflow run record exposes a complete sequence of every tool invocation, whether DSL-explicit or model-driven.
- Failed tool calls are recorded with their error context.

---

## Epic 4 — Credentials and Context Pipeline Hardening

**Business Value**: Solidifies the credentials handling introduced in v2 so v6's vault migration is a clean swap.

### Story 4.1 — Credentials interface abstraction
**As a** Platform Engineer
**I want** all credential reads to go through a single abstraction
**So that** v6 can swap plain-text storage for a secrets vault without rewriting the platform.

**Description**: A `CredentialsProvider` interface with one method: `get(agent_id, scope)` returning a typed credentials bundle. v3 implementation reads from the database in plain text; v6 swaps in a vault-backed implementation. The Agent Core, activities, and MCP adapters consume this interface only.

**Acceptance Criteria**:
- No code outside the credentials module accesses the credentials table directly.
- Unit tests cover both happy path and missing-credentials scenarios.

### Story 4.2 — Per-workflow credentials snapshot
**As an** Operator
**I want** credentials to be loaded once per workflow run and reused across steps
**So that** the database is not hammered and the security boundary is well-defined.

**Description**: At workflow start, the entry activity calls `CredentialsProvider.get(...)` once, packages the result into the workflow context, and passes it through. Steps receive the snapshot, not the database connection.

**Acceptance Criteria**:
- A workflow that uses 10 steps reads credentials once.
- The snapshot is not logged or persisted beyond the workflow lifetime.

---

## Technical Considerations (high level)

- **Tools live in code; MCPs live in configuration**: tools are versioned with the platform release; MCPs are registered dynamically and may be added without redeployment.
- **MCP transport adapters**: implement HTTP and stdio first; add WebSocket if needed by specific MCPs.
- **OAuth flows**: for MCPs requiring OAuth (e.g. Google, Atlassian), the full lifecycle is specified in `09-security-model.md` §OAuth Token Lifecycle for MCPs (E-11). Service-account OAuth in v3; user-delegated OAuth in v5.
- **Schema validation everywhere**: every tool and MCP capability has a JSON Schema for inputs and outputs. Validate aggressively.
- **Native provider tool-calling**: each provider adapter handles its own format. Do not invent a cross-provider tool format — adapt at the edges.
- **Idempotency contract (E-12)**: Every tool registered in the Tool Registry declares `idempotent: boolean`. When `true`, the platform injects the step run ID as an idempotency key on every call. The Tool interface contract: `{ name, description, input_schema, output_schema, handler, idempotent }`. The validation service warns when a `tool_call` step has `retry.max_attempts > 1` and the tool does not declare `idempotent: true`.
- **MCP manifest cache invalidation (E-13)**: Manifests are cached with a default TTL of 60 minutes. Three invalidation modes: (1) **TTL-based** (default, 60 min configurable); (2) **On-demand** via `agent-platform mcps refresh <name>` CLI command; (3) **Version-pinned**: MCP registry entries may declare a `manifest_version` string — if the live manifest version changes, a Platform Admin alert is raised rather than silently switching. When a cached manifest capability is removed, the validation service warns on next provisioning of any DSL referencing that capability.

## Definition of Done for v3

- A starter catalogue of internal tools is registered and callable from DSL.
- At least one own MCP and one third-party MCP (Linear, Jira, or Figma) are registered and callable.
- The LLM step exposes tools/MCPs to the model via native tool-calling.
- Every tool and MCP invocation is persisted in workflow runs.
- The credentials pipeline is fully abstracted; vault swap is a single-implementation change.

---

## What to Read Next

With v3, agents can interact with the world via tools and MCPs. The next phase enables agent composition:

→ **[05-v4 — Multi-Agent Orchestration](./05-v4-multi-agent-orchestration.md)** — Sub-agent calls, parallel execution, branching, and loops.

Or explore supporting topics:
- [09-Security Model](./09-security-model.md) — MCP authentication and trust boundaries
- [10-Deployment Concepts](./10-deployment-concepts.md) — Worker topology for tool/MCP calls
- [03-v2](./03-v2-workflow-orchestration.md) — Review workflow foundations
