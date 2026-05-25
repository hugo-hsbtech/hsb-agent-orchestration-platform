# Agent Platform — Enhancement Proposals

> **Purpose of this document**: A comprehensive review of the full documentation set (docs 01–17) with specific, actionable enhancements to features, capabilities, and gaps. Each enhancement is anchored to a specific document, decision, or story and includes a clear rationale and proposed resolution. This document does not override any existing architectural decision — enhancements that conflict with an existing AD are explicitly flagged as proposals requiring a decision update.

---

## How to Read This Document

Enhancements are grouped by the platform layer they affect. Each entry includes:

- **Anchored to**: the document(s) and section it relates to
- **Gap**: what is missing, ambiguous, or underspecified
- **Proposal**: the concrete enhancement
- **Priority**: `pre-build` (must resolve before implementation begins), `in-phase` (resolve during the relevant phase), `post-v6` (future planning)
- **Impacts**: downstream documents or decisions affected

---

## Layer 1: DSL and Expression Language

### E-01 — Settle the expression language (condition and mapping operators)

**Anchored to**: `14-dsl-spec.md` §Data Flow Expression Syntax, §Condition expressions; `05-v4-multi-agent-orchestration.md` Epic 3 Story 3.1  
**Priority**: `pre-build` (before v4)

**Gap**: The DSL spec defines a useful table of condition operators (`==`, `!=`, `>=`, `&&`, `||`, `.length`) and mapping operators (field selection, default fallback, ternary, string concat, `_each`). However, the specification stops at a table. There is no formal grammar, no named language, no specification of evaluation order, no definition of type coercion rules, and no guidance on what happens when an expression references a field that is `null` vs. missing vs. the wrong type. "The validation service rejects expressions that use constructs outside this set" is a behavioral claim without a machine-checkable definition.

**Proposal**: Publish a formal mini-spec within `14-dsl-spec.md` that includes:
1. An unambiguous EBNF or PEG grammar for both condition expressions and mapping expressions (they share 80% of their grammar and should be unified).
2. Type system: the three scalar types (`string`, `number`, `boolean`), `null`, `array`, and `object`. Explicit rules for what `==` means across mixed types (strict, no implicit coercion).
3. Missing-field vs. null semantics: `$.steps.X.output.field` when `field` does not exist in the output — does it produce `null`, throw a `ReferenceError`, or fall through to the `|` default operator?
4. Evaluation short-circuit rules for `&&` and `||`.
5. A named reference to the grammar in the validation service story (`02-v1` Epic 3 Story 3.1) so the grammar is the validation contract, not a secondary document.
6. The `_each` directive for `transform` steps should be formally specified alongside the rest — currently it appears only in the `transform` step example with a one-line note.

**Impacts**: `14-dsl-spec.md`, `05-v4`, `11-testing-strategy.md` (validation service unit tests), `15-developer-tooling.md` (CLI validator).

---

### E-02 — DSL v5 (chatbot type) is underspecified in the spec

**Anchored to**: `14-dsl-spec.md` §DSL Versioning table; `06-v5-chatbot-layer.md` Epic 1 Story 1.3  
**Priority**: `in-phase` (during v5)

**Gap**: The DSL versioning table in `14-dsl-spec.md` notes that v5 introduces `agent_type: chatbot` and "chatbot-specific fields," but the document contains no chatbot step types, no chatbot-specific top-level fields, and no chatbot DSL examples. The v5 phase document describes the chatbot's capabilities at the story level but never produces a canonical DSL definition. An Agent Builder reading `14-dsl-spec.md` to write a chatbot agent will find nothing.

**Proposal**: Add a `## Chatbot Agent Type (DSL v5)` section to `14-dsl-spec.md` that includes:
1. The complete top-level chatbot DSL shape (all fields present for `agent_type: chatbot`, including `context_window`, `human_handoff`, `routing` configuration, and `knowledge_base` scoping).
2. Clarification that chatbot agents have no `workflow.steps` block — the conversation loop is the runtime, not the DSL.
3. A fully worked chatbot DSL example (routing chatbot + one specialist chatbot) mirroring the v4 multi-agent workflow example that already appears at the bottom of `14-dsl-spec.md`.
4. Specification of which top-level fields from workflow agents are shared (`system_prompt`, `provider`, `model_config`, `tools`, `mcps`, `namespace`) and which are chatbot-only (`context_window`, `human_handoff`, `routing`).

**Impacts**: `14-dsl-spec.md`, `06-v5-chatbot-layer.md`, `15-developer-tooling.md` (CLI must validate chatbot DSL).

---

### E-03 — The `fetch` step type should be deprecated in favor of `tool_call`/`mcp_call`

**Anchored to**: `14-dsl-spec.md` §`fetch`; `03-v2-workflow-orchestration.md` Epic 2 Story 2.2  
**Priority**: `in-phase` (during v3)

**Gap**: `fetch` was introduced in v2 as a "placeholder for HTTP/DB calls — fully fleshed out in v3 once tools land." v3 landed `tool_call` and `mcp_call`. The `14-dsl-spec.md` notes: "In v3+, prefer `tool_call` or `mcp_call` for named integrations." But `fetch` is neither formally deprecated nor is there guidance for what remains a valid use case for it (ad-hoc HTTP calls not worth registering as a tool). This leaves an ambiguous, unsupported step type in the registry that every implementer must decide how to handle.

**Proposal**: Make an explicit architectural decision (AD-28) in `01-PRD.md` — one of:
- **Option A** (preferred): Keep `fetch` as a first-class step for ad-hoc HTTP calls not registered in the Tool Registry. Add full field specifications (auth headers, response parsing, timeout, content-type) and a note that it carries no idempotency guarantees (unlike `tool_call`). Add `credentials.*` resolution rules.
- **Option B**: Formally deprecate `fetch` in v3. Require all HTTP calls go through the Tool Registry. Provide a `generic_http` built-in tool as a migration target.

Whichever option is chosen, resolve it before v3 build begins and update `14-dsl-spec.md` accordingly.

**Impacts**: `14-dsl-spec.md`, `04-v3-tools-and-mcps.md`, `01-PRD.md` (new AD-28).

---

### E-04 — No input schema declaration at the workflow level

**Anchored to**: `14-dsl-spec.md` §Top-Level Document Structure; `03-v2-workflow-orchestration.md` Epic 1 Story 1.3  
**Priority**: `pre-build` (before v2)

**Gap**: Workflow steps reference `$.workflow.input.*` fields extensively, but there is no way to declare the expected shape of the workflow's input payload in the DSL. This means: (1) the validation service cannot check that `$.workflow.input.user_question` will be present at runtime, (2) the invocation API (`POST /namespaces/{ns}/agents/{id}/invoke`) has no machine-readable schema to validate the request body against, (3) Agent Builders have no contract to document for callers of their agent.

**Proposal**: Add an optional (recommended) `input_schema` top-level field to the DSL:

```json
{
  "input_schema": {
    "user_question": { "type": "string", "required": true },
    "customer_id":   { "type": "string", "required": false },
    "language":      { "type": "string", "required": false, "default": "en" }
  }
}
```

The validation service uses this to verify all `$.workflow.input.*` references. The invocation API validates the request body against it at runtime. Agent Builders publishing agents to callers can share this schema. For `agent_call` steps, the validation service cross-references the called agent's `input_schema` against the calling step's `inputs` mapping.

**Impacts**: `14-dsl-spec.md`, `03-v2`, `05-v4` (agent_call cross-schema validation), `11-testing-strategy.md`.

---

## Layer 2: Validation Service

### E-05 — Validation service has no cross-agent graph validation path documented

**Anchored to**: `01-PRD.md` AD-12, AD-13; `05-v4-multi-agent-orchestration.md` Technical Considerations  
**Priority**: `in-phase` (during v4)

**Gap**: The validation service collects all errors per-document and returns them together (AD-13). This works for single-agent validation. But v4 introduces `agent_call`, which means a DSL document's correctness now depends on another agent's definition (its input schema, its output schema, its existence, its version, its cycle membership). Cross-document validation is fundamentally different from single-document validation: it requires resolving a graph of agent references, detecting cycles across that graph, and validating type compatibility across agent boundaries. None of this cross-graph validation path is specified — neither how it is triggered, nor what errors it produces, nor how it interacts with the "collect all, present once" error model when one of the referenced agents itself has errors.

**Proposal**: Add a new section to `01-PRD.md` under AD-12 or as a companion AD-28 covering the validation service's graph validation mode. Specify:
1. `validate --graph` mode: resolves all `agent_call` references recursively, validates the full call graph in one pass, returns errors attributed to their source document.
2. Cycle detection algorithm: depth-first traversal, maximum call depth enforcement (default 10, per the existing note in `14-dsl-spec.md`).
3. Cross-agent type compatibility: if agent B declares `input_schema`, any `agent_call` step targeting B must supply all required fields, and the validation service checks type compatibility.
4. Error attribution: errors found in a transitively referenced agent are reported at the referencing step with a chain: `agent A → step run_payment → agent B: missing required input 'currency'`.
5. How this integrates with the provisioning API: does saving agent A trigger validation of all agents that call A? Or is graph validation opt-in?

**Impacts**: `01-PRD.md`, `05-v4`, `14-dsl-spec.md`, `15-developer-tooling.md` (the `agent-platform validate --graph` command is already mentioned but not specified).

---

### E-06 — Provider availability check is not idempotent-safe for provisioning

**Anchored to**: `02-v1-agent-provisioning-foundation.md` Epic 3 Story 3.2; `11-testing-strategy.md` §Provider Availability Check  
**Priority**: `pre-build` (before v1)

**Gap**: Story 3.2 says the provisioning API checks that the declared provider "is configured (has credentials, is reachable)." `11-testing-strategy.md` correctly notes this should use a minimal `probe()` call. The gap: if the provisioning API issues a live provider probe on every `POST /agents`, a provider outage will make all agent provisioning fail — even for providers the agent doesn't use yet (e.g., registering an agent that will use OpenAI, but Claude is down). The current spec does not distinguish between "provider credentials configured" and "provider currently reachable." The former is static and fast; the latter is dynamic and fragile.

**Proposal**: Split the provider check into two:
1. **At provisioning time (mandatory)**: verify provider credentials are configured in the platform (a local DB lookup, no network call). Fail if not.
2. **At invocation time (first execution)**: run the `probe()` call as the first activity in the workflow. If the probe fails, the workflow fails immediately with `ProviderUnavailableError` (which is retryable). This keeps provisioning fast and decoupled from provider availability.

Document this in `02-v1` Story 3.2 and update `11-testing-strategy.md` accordingly.

**Impacts**: `02-v1`, `11-testing-strategy.md`.

---

## Layer 3: Workflow Execution and Orchestration

### E-07 — No timeout declaration at the step level

**Anchored to**: `14-dsl-spec.md` §Retry Policy; `08-performance-and-scaling.md` §Latency Budgets by Step Type  
**Priority**: `pre-build` (before v2)

**Gap**: Every step has a `retry` block but no `timeout` field. `08-performance-and-scaling.md` notes that `mcp_call` steps have latency of 100ms–30s and "need explicit timeout declarations in the DSL." Yet neither `14-dsl-spec.md` nor the retry policy spec includes a `timeout_seconds` field. Without a declared timeout, Temporal will apply its default activity timeout (which may be minutes), meaning a hung `mcp_call` can block a workflow for far longer than intended — and the LLM context window may expire, retries may stack up, and the Operator has no declared SLA to enforce. The `fetch` step already has `timeout_seconds` on its `request` block, but only for HTTP — this is inconsistent.

**Proposal**: Add `timeout_seconds` to the step-level retry block (or as a sibling field):

```json
{
  "timeout_seconds": 30,
  "retry": {
    "max_attempts": 3,
    "initial_interval_seconds": 2,
    "non_retryable_errors": ["ValidationError"]
  }
}
```

Document per-step-type recommended timeout ranges in `14-dsl-spec.md`, consistent with the latency budget table in `08-performance-and-scaling.md`. The validation service warns when a step type known to be high-latency (`mcp_call`, `agent_call (wait)`) is missing a `timeout_seconds`.

**Impacts**: `14-dsl-spec.md`, `08-performance-and-scaling.md`, `03-v2`, `04-v3`.

---

### E-08 — Application DB and Temporal reconciliation after a partial outage is undocumented

**Anchored to**: `01-PRD.md` AD-04; `10-deployment-concepts.md` §Failure Modes; `12-upgrades-and-migrations.md`  
**Priority**: `pre-build` (before v2)

**Gap**: AD-04 states that both Temporal and the application database coexist as state stores for different purposes: Temporal manages execution state; the app DB is the product-visible state machine. The deployment doc's failure mode table lists "Database primary failure → brief unavailability" but does not address the split-brain scenario: **Temporal progresses a workflow step while the app DB is unavailable to record it**. When the DB recovers, the Temporal history and the app DB diverge. There is no documented reconciliation strategy, no specification of which system is authoritative, and no recovery procedure.

**Proposal**: Add a dedicated `## Temporal/DB Reconciliation` section to `10-deployment-concepts.md` (and a pointer from `12-upgrades-and-migrations.md`) that specifies:
1. **The contract**: every Temporal activity that writes a step result to the app DB must be idempotent. A step run row's primary key is derived from `(workflow_run_id, step_id, attempt_number)` — always idempotent to replay.
2. **Recovery procedure**: on DB recovery, a reconciliation job queries Temporal for workflow runs that are in `running` state in Temporal but in `running` or `unknown` state in the app DB, and replays their activity outputs from Temporal history into the app DB.
3. **Acceptable divergence window**: the time between DB recovery and reconciliation job completion is bounded by the reconciliation polling interval.
4. Specify that Temporal is the execution truth and the app DB is the projection. Never update Temporal to match the app DB; always update the app DB to match Temporal.

**Impacts**: `10-deployment-concepts.md`, `12-upgrades-and-migrations.md`, `03-v2` Technical Considerations.

---

### E-09 — `parallel` block's `max_concurrency` is optional but should be required

**Anchored to**: `14-dsl-spec.md` §`parallel`; `05-v4-multi-agent-orchestration.md` Epic 2 Story 2.1  
**Priority**: `in-phase` (during v4)

**Gap**: `for_each` correctly makes `max_concurrency` required (validation rejects without it). But the `parallel` block's `max_concurrency` field appears in the example but the spec does not state whether it is required or optional. If it is optional with no default, a `parallel` block with many branches will fan out unboundedly — the exact behavior the platform explicitly warns against in `08-performance-and-scaling.md` §Fan-Out Limits.

**Proposal**: Make `max_concurrency` required on `parallel` blocks, consistent with `for_each`. The validation service rejects `parallel` blocks without it. Add a default recommendation (e.g., "document expected fan-out limits" — as stated in `05-v4` Technical Considerations — and set a platform-configurable global cap, e.g., `50`, that the validation service enforces even when `max_concurrency` is declared).

**Impacts**: `14-dsl-spec.md`, `05-v4`, `08-performance-and-scaling.md`.

---

### E-10 — Fire-and-forget orphan risk has no documented mitigation

**Anchored to**: `14-dsl-spec.md` §`agent_call`; `05-v4-multi-agent-orchestration.md` Epic 1 Story 1.2; `08-performance-and-scaling.md` §Latency Budgets  
**Priority**: `in-phase` (during v4)

**Gap**: `08-performance-and-scaling.md` notes for `agent_call (fire_and_forget)`: "No retry (orphan risk)." This is the only mention of the orphan risk. A `fire_and_forget` child workflow that fails will have no parent to propagate the error to, no automatic cleanup, no user-visible failure notification, and no documented cleanup path. In the chatbot layer (v5), `fire_and_forget` children are how background tasks are kicked off — and their failure mode is exactly the "permanent failure" scenario in `13-chatbot-ux-edge-cases.md` §12. The two documents do not reference each other.

**Proposal**: 
1. Add an `on_orphan_failure` field to the `agent_call` DSL for `fire_and_forget` mode, with options: `log_only` (default), `notify_conversation` (for chatbot-linked tasks), `webhook` (call a configured URL), `create_pending_alert`.
2. The Orchestrator / Monitor (v2 Epic 4) should detect `fire_and_forget` child workflows that have failed and, based on the `on_orphan_failure` policy, take action.
3. Cross-reference `13-chatbot-ux-edge-cases.md` §12 with this mechanism, so the chatbot layer's task failure surfacing is built on the same foundation.

**Impacts**: `14-dsl-spec.md`, `05-v4`, `06-v5-chatbot-layer.md`, `08-performance-and-scaling.md`, `13-chatbot-ux-edge-cases.md`.

---

## Layer 4: Tools, MCPs, and Credentials

### E-11 — OAuth token lifecycle for MCPs is unresolved before v5

**Anchored to**: `04-v3-tools-and-mcps.md` Technical Considerations; `09-security-model.md` §MCP Authentication Strategies; `01-PRD.md` Open Questions  
**Priority**: `pre-build` (before v5)

**Gap**: The OAuth flow for MCPs is mentioned three times across three documents with slightly different wording: "stub UI in v3," "OAuth 2.0: Refresh token stored; access token fetched at workflow start," and "OAuth flow UX for MCP authentication: stub in v3, refine in v5 if chatbot UX intersects with auth." None of these define: who authorizes the OAuth flow (service account vs. user-delegated), how the refresh token is obtained initially (out-of-band admin configuration vs. OAuth redirect flow), how token expiry is handled mid-workflow (the credentials snapshot is loaded once at workflow start — what if the access token expires during a long workflow?), and what happens when a refresh fails.

**Proposal**: Add an `18-oauth-for-mcps.md` document (or a dedicated section in `09-security-model.md`) that specifies:
1. **Service-account OAuth**: a refresh token is stored in the credential store by the Platform Admin via an out-of-band flow. The `CredentialsProvider` exchanges it for an access token at workflow start and stores it in the workflow context. Access token TTL is known; if TTL < workflow estimated runtime, the provider pre-fetches a new access token and stores both.
2. **User-delegated OAuth**: deferred until v5 introduces user sessions. At v5, the chatbot session can carry a per-user OAuth token that is injected into the workflow context when a specialist chatbot delegates to a backend workflow agent.
3. **Mid-workflow token expiry**: if a step receives an auth error from an MCP, the activity retries with a token refresh before propagating the error. Token refresh is an atomic operation on the workflow context (one refresh per workflow, not per step).
4. **Failure mode**: refresh failure is a `non_retryable_error` that fails the step and surfaces a `CredentialExpiredError`.

Update `04-v3` to reference this document and remove the vague "stub" language.

**Impacts**: `04-v3`, `09-security-model.md`, `06-v5-chatbot-layer.md`, `14-dsl-spec.md` (MCP auth field in MCP registry entry).

---

### E-12 — Tool idempotency is assumed but not enforced

**Anchored to**: `04-v3-tools-and-mcps.md` Technical Considerations; `03-v2-workflow-orchestration.md` Technical Considerations  
**Priority**: `in-phase` (during v3)

**Gap**: `04-v3` states "Tool/MCP calls that mutate state must carry idempotency keys based on the step run ID." `14-dsl-spec.md` notes for `mcp_call` that idempotency keys are "automatically injected for MCP calls that declare `idempotent: true` in their manifest." But:
1. There is no `idempotent` field documented in the MCP manifest schema.
2. For `tool_call`, the tool's `input_schema` has no `idempotent` declaration — so the platform cannot know which tool calls are safe to retry.
3. For mutating tools (`tool_call` steps that write to a database), retries without idempotency can produce duplicate writes. The current spec's only protection is "Temporal handles retries safely" — but Temporal retries activities, and if the activity itself is not idempotent, duplicates occur.

**Proposal**:
1. Add an `idempotent: boolean` field to the Tool interface definition and the MCP manifest. When `true`, the platform injects the step run ID as an idempotency key header/field on every call.
2. The validation service **warns** (not errors) when a `tool_call` or `mcp_call` step has `retry.max_attempts > 1` and the referenced tool/MCP does not declare `idempotent: true`. The warning message: "Tool 'X' does not declare idempotency. Retries may produce duplicate side effects."
3. Add a `Tools contract` section to `04-v3` Epic 1 that formalizes what a tool must declare: `name`, `description`, `input_schema`, `output_schema`, `handler`, and `idempotent`.

**Impacts**: `04-v3`, `14-dsl-spec.md`, `02-v1` (agent core tool interface is set in v1/v3).

---

### E-13 — MCP manifest cache invalidation strategy is underspecified

**Anchored to**: `04-v3-tools-and-mcps.md` Epic 2 Story 2.3  
**Priority**: `in-phase` (during v3)

**Gap**: Story 2.3 says the MCP manifest "cache invalidates appropriately when an MCP's manifest changes." This is the entire specification of cache invalidation. There is no TTL, no push invalidation mechanism, no versioning of manifests, and no definition of what "changes" means (new capabilities added? existing capabilities removed? auth requirements changed?). If a cached manifest drifts from the live MCP's capabilities, `mcp_call` steps will either fail at runtime (MCP rejects an unknown capability) or, worse, call a capability that has been renamed without the platform knowing.

**Proposal**: Specify three cache invalidation modes in `04-v3`:
1. **TTL-based** (default): manifest is refreshed every N minutes (configurable, default 60). Stale manifests serve until TTL expires.
2. **On-demand**: `agent-platform mcps refresh <name>` (CLI command) forces a re-fetch and re-validates all DSL documents referencing this MCP against the new manifest.
3. **Version-pinned**: the MCP entry in the registry can declare a `manifest_version` string. If the live MCP's manifest version changes, alert the Platform Admin rather than silently switching.

Add a `manifest_version` field to the MCP registry schema. Document what happens when a capability referenced in an existing DSL is removed from an updated manifest (warning at next validation; no automatic workflow disruption).

**Impacts**: `04-v3`, `14-dsl-spec.md` (MCP registry schema), `15-developer-tooling.md` (CLI `mcps refresh` command).

---

## Layer 5: Chatbot Layer

### E-14 — Routing chatbot's confidence model must be a settled decision before v5 build

**Anchored to**: `06-v5-chatbot-layer.md` Epic 2 Story 2.1; `13-chatbot-ux-edge-cases.md` §7; `08-performance-and-scaling.md` §TTFT  
**Priority**: `pre-build` (before v5)

**Gap**: Story 2.1 says routing "can be a quick LLM call with a strict, schema-constrained output or a dedicated classifier." The edge case document (§7) adds confidence bands (>0.8, 0.5–0.8, <0.5) without specifying how those scores are produced. The performance spec targets routing classification at <200ms TTFT. These are three inconsistent, unconnected specifications. "Quick LLM call" and "<200ms" are difficult to reconcile: a full LLM round-trip (even constrained) to Claude or OpenAI typically takes 300ms–1s. A dedicated classifier (embedding similarity against specialist scope definitions) could achieve <50ms but requires building and maintaining a separate model or embedding index.

**Proposal**: Make an explicit architectural decision (AD-28 or update AD-19) on the routing mechanism — one of:
- **Option A** (recommended): Constrained LLM call with `max_tokens: 50` and JSON schema output `{ "specialist": "billing|technical|admin|knowledge|fallback", "confidence": 0.0–1.0 }`. Accept that TTFT for the first token of the specialist's response is 400–800ms (classification + handoff). Revise the <200ms target to be "classification must complete within 500ms."
- **Option B**: Embedding-based classifier: encode the user message and compare cosine similarity against pre-computed centroid embeddings for each specialist's scope. Sub-50ms, but requires maintaining embeddings and a separate inference step.
- **Option C**: Hybrid: embedding-based pre-filter (fast), LLM confirmation only for low-confidence cases.

The decision must be in `01-PRD.md` (new AD) and reflected in the DSL: does the routing chatbot's DSL declare its classification mechanism, or is it a platform-level configuration?

**Impacts**: `06-v5-chatbot-layer.md`, `13-chatbot-ux-edge-cases.md`, `08-performance-and-scaling.md`, `01-PRD.md`, `14-dsl-spec.md`.

---

### E-15 — Context window strategy `summarize` creates an unspecified agent call loop

**Anchored to**: `06-v5-chatbot-layer.md` Epic 6 Story 6.1; `13-chatbot-ux-edge-cases.md` §Multi-turn  
**Priority**: `in-phase` (during v5)

**Gap**: The `summarize` context window strategy "invokes `summary_agent` as a synchronous agent call and injects the summary into context." This creates three unspecified behaviors:
1. **Who triggers the summary**: is it triggered before the specialist sends its next response (adding latency) or lazily (the next turn may exceed the token budget before the summary is ready)?
2. **What does `summary_agent` receive**: the full conversation history? The turns being dropped? Both?
3. **Recursive summarization risk**: if the conversation is extremely long and summary itself consumes tokens, does the summary get summarized? There is no maximum summary length, no budget for the summary block within the overall token budget.

Additionally, the `cross-specialist context handoff` (Story 6.2) specifies a "summary of the conversation so far, injected as a system-level note." This is a second, different summarization path that is also unspecified — who generates it? The routing chatbot? A designated summary agent? The outgoing specialist?

**Proposal**: 
1. Specify that summarization is **eager and proactive**: when the context window manager determines on turn N that turn N+1 will exceed the budget, it triggers summarization before sending turn N's response, so the summary is ready before the user replies.
2. Add a `max_summary_tokens` field to the `context_window` policy (default: 25% of `max_tokens`). The summary agent is given this constraint in its invocation.
3. Unify cross-specialist handoff summaries with the `summary_agent` mechanism: the routing chatbot uses the same registered `summary_agent` for handoff summaries. The handoff summary length is configurable separately (`handoff_summary_tokens`, default: 500).
4. Add an acceptance criterion to Story 6.1: "The context actually sent to the provider never exceeds `max_tokens` even accounting for the summary block's size."

**Impacts**: `06-v5-chatbot-layer.md`, `13-chatbot-ux-edge-cases.md`, `14-dsl-spec.md` (chatbot DSL fields).

---

### E-16 — PII handling in conversation persistence needs a concrete policy

**Anchored to**: `09-security-model.md` §Compliance Considerations; `13-chatbot-ux-edge-cases.md` §14; `06-v5-chatbot-layer.md` Epic 1 Story 1.2  
**Priority**: `in-phase` (during v5)

**Gap**: `13-chatbot-ux-edge-cases.md` §14 describes PII detection and masking at the input layer: "PII detection masks before persistence. Specialist receives redacted version." `09-security-model.md` mentions "Conversation logs: Pseudonymization options, user deletion requests." `06-v5` Story 1.2 says each turn stores "user message, assistant response." These three descriptions do not cohere: if PII is masked before persistence, then the `conversation_turns` table stores the masked version — but the specialist's LLM receives the masked version too, which may impede its ability to help. There is no defined PII detection mechanism, no specification of which PII categories are detected (CREDIT_CARD, SSN, PII_EMAIL, etc.), no policy on what the LLM receives vs. what is stored, and no GDPR-right-to-erasure path.

**Proposal**: Add a `## PII Handling` section to `09-security-model.md` with:
1. **Detection layer**: PII scanning runs on user input before it reaches the routing chatbot's LLM call. Detected PII categories: CREDIT_CARD, SSN, FULL_NAME (optional, configurable per namespace), EMAIL, PHONE, IBAN.
2. **Two-track persistence**: the raw (unredacted) message is stored in an encrypted `conversation_turns.raw_message` column (key managed per namespace). The redacted version (`conversation_turns.message`) is what the LLM and downstream systems see. Raw storage is configurable: operators can opt-out to store only redacted.
3. **Right-to-erasure**: a `DELETE /namespaces/{ns}/conversations/{id}/user-data` endpoint zeroes the `raw_message` columns and replaces `message` with `[USER_DATA_DELETED]` for all turns in the conversation. The conversation record (ID, timestamps, metadata) is retained for audit.
4. **Namespace-level PII policy**: namespaces declare their PII detection scope in `settings.pii_detection_categories`.

**Impacts**: `09-security-model.md`, `06-v5-chatbot-layer.md`, `13-chatbot-ux-edge-cases.md`, `16-multi-tenancy.md` (namespace settings).

---

### E-17 — Human handoff webhook reliability is one-sided

**Anchored to**: `06-v5-chatbot-layer.md` Epic 7 Story 7.1 Technical Considerations  
**Priority**: `in-phase` (during v5)

**Gap**: The technical considerations note: "webhook deliveries should retry on failure (configurable, default 3 attempts). Failed deliveries fall back to the `message` strategy and alert the Operator." What is missing: (1) There is no acknowledgement protocol — the spec says "waits for a `200` acknowledgement" but does not define what the receiving system must return in the body, how to distinguish a successful 200 from a 200 that indicates a downstream error, or how to handle 2xx vs. 5xx vs. timeout. (2) There is no delivery receipt persisted in the conversation record beyond "escalated_at" — meaning an Operator cannot tell whether the webhook actually delivered or silently fell back to `message`. (3) The Prometheus metric `chatbot_escalations_total` (Story 7.2) does not distinguish between successful webhook delivery and fallback-to-message escalations — two very different operational outcomes.

**Proposal**: 
1. Add `escalation_outcome` to the conversation record: `webhook_delivered`, `webhook_failed_fallback`, `message_displayed`, `ticket_created`.
2. Split the Prometheus metric into `chatbot_escalations_total{strategy, outcome}` where `outcome` is one of the above.
3. Add a `webhook_delivery_log` entry per escalation attempt: attempt number, timestamp, HTTP status, response body (truncated), outcome.
4. Define the webhook acknowledgement contract: the receiving system must return `200` with `{"received": true}` or any 2xx (body ignored). Non-2xx or timeout triggers retry. Document this contract for operators configuring the webhook URL.

**Impacts**: `06-v5-chatbot-layer.md`, `07-v6-versioning-and-hardening.md` (metrics spec).

---

## Layer 6: Versioning, Observability, and Hardening

### E-18 — Cost tracking and token budget enforcement is absent

**Anchored to**: `14-dsl-spec.md` §Canonical Step Output Envelope (`tokens` field); `16-multi-tenancy.md` §Quota Enforcement; `08-performance-and-scaling.md`  
**Priority**: `in-phase` (during v6, but schema from v1)

**Gap**: Every step's output envelope already includes `metadata.tokens.input`, `metadata.tokens.output`, `metadata.tokens.total`. This data is persisted for every `llm_call` step. Yet there is no document, no API, and no quota that aggregates this data into: per-workflow-run cost, per-agent cost, per-namespace cost, or per-day cost. For a multi-tenant SaaS platform, this is a critical operational and revenue capability. The quota system in `16-multi-tenancy.md` covers workflow run counts and session counts but has no token or cost dimension.

**Proposal**: Add a cost tracking section to `16-multi-tenancy.md` and `07-v6-versioning-and-hardening.md`:
1. **Per-run cost summary**: the `workflow_runs` table gains a `total_tokens` and `estimated_cost_usd` column (estimated based on published provider pricing, configurable rate table per provider/model). Populated by the orchestrator on run completion.
2. **Namespace quota**: add `max_tokens_per_day` and `max_estimated_cost_per_day_usd` to the namespace quota schema. Enforced at the API level (reject new invocations when daily token budget is exhausted) or as alerting-only (configurable per namespace).
3. **Prometheus metrics**: `agent_platform_tokens_total{namespace, agent, provider, model, step_type}` and `agent_platform_estimated_cost_usd_total{namespace, provider, model}`.
4. **Cost API**: `GET /namespaces/{ns}/cost-summary?from=&to=` returns aggregated token and cost data, useful for billing reconciliation.
5. **At-validation cost estimate**: `agent-platform dry-run` already exists in `15-developer-tooling.md`. Extend it to include a `--estimate-cost` flag that uses the declared `max_tokens` per `llm_call` step and the configured provider pricing to produce a worst-case cost estimate per workflow run.

**Impacts**: `16-multi-tenancy.md`, `07-v6`, `08-performance-and-scaling.md`, `15-developer-tooling.md`, `14-dsl-spec.md`.

---

### E-19 — LLM observability hook (LangFuse) should include a prompt versioning contract

**Anchored to**: `07-v6-versioning-and-hardening.md` Epic 4 Story 4.4; `01-PRD.md` AD-23  
**Priority**: `in-phase` (during v6)

**Gap**: Story 4.4 specifies an `LLMObservabilityClient` interface called on every LLM invocation with "prompt, response, tokens, latency, cost estimate." The gap: agent versions are immutable (AD-16), but the `LLMObservabilityClient` interface as described would receive the raw interpolated prompt — which changes per invocation (different user inputs). LangFuse and similar tools track **prompt templates** as versioned artifacts (the system prompt + template), distinct from the interpolated runtime prompt. If the interface only receives the final interpolated prompt, it cannot correlate observations back to a specific agent version's prompt template, making the observability data hard to aggregate by prompt version.

**Proposal**: Extend the `LLMObservabilityClient` interface signature to include:
```
interface LLMObservabilityClient {
  record(event: {
    agent_id:          string,
    agent_version:     string,
    workflow_run_id:   string,
    step_id:           string,
    trace_id:          string,
    provider:          string,
    model_name:        string,
    system_prompt:     string,        // template (from agent version)
    prompt_template:   string,        // user_prompt_template (from agent version)
    interpolated_prompt: string,      // final user prompt sent to provider
    response:          string,
    tokens:            TokenCounts,
    latency_ms:        number,
    estimated_cost_usd: number,
    finish_reason:     string
  }): void
}
```

With both `prompt_template` (stable, from the agent version) and `interpolated_prompt` (per-invocation), LangFuse can segment observations by template and detect regressions at the template level.

**Impacts**: `07-v6`, `01-PRD.md` AD-23.

---

### E-20 — Debug mode step override has no type-safety or schema validation

**Anchored to**: `07-v6-versioning-and-hardening.md` Epic 3 Story 3.1; `17-operator-cli.md` §`run debug override`  
**Priority**: `in-phase` (during v6)

**Gap**: Debug mode allows an Operator to override a step's inputs before it runs. This is a powerful capability for testing edge cases. But the spec does not address: (1) whether the override is validated against the step's declared `inputs` schema or the tool's `input_schema` before the step runs, (2) what happens if the operator supplies malformed JSON or a field of the wrong type — does the step fail with a confusing internal error, or does the debug session catch it cleanly? (3) whether overrides are persisted to the step run record so the audit log reflects what actually ran vs. what the workflow would have passed.

**Proposal**:
1. Override inputs are validated against the step's `inputs` mapping and (for `tool_call`/`mcp_call`) the tool/MCP's `input_schema` before the step executes. Invalid overrides are rejected with a structured error at the CLI — not passed through to the step.
2. The `workflow_step_runs` record gains an `input_override: JSON | null` column. When an override is applied, `input` stores what the step actually received, and `input_override` stores the original resolved input (for diffing).
3. The `run debug override` CLI command displays the validation result of the proposed override before asking for confirmation.

**Impacts**: `07-v6`, `17-operator-cli.md`, `14-dsl-spec.md` (step run schema), `11-testing-strategy.md`.

---

## Layer 7: Multi-Tenancy and Security

### E-21 — Inter-agent data poisoning threat (prompt injection across agent boundaries)

**Anchored to**: `09-security-model.md` §Threat: Credential Extraction via Prompt Injection; §Sub-Agent Escape  
**Priority**: `pre-build` (before v4)

**Gap**: `09-security-model.md` addresses prompt injection from user input and sub-agent privilege escalation. It does not address **inter-agent data poisoning**: when agent A passes the output of a tool call or LLM response to agent B via `agent_call`, agent B's LLM receives that data as part of its prompt. If the tool's output or agent A's response contains injected content (e.g., a malicious document was fetched via `tool_call` that contains "Ignore previous instructions and..."), agent B inherits the injection without any sanitization boundary.

This is the multi-agent equivalent of second-order SQL injection — the injection payload is not in the original user input but in a data artifact processed along the way.

**Proposal**: Add a `## Inter-Agent Data Propagation Threat` section to `09-security-model.md`:
1. **Threat description**: a malicious content artifact (fetched via `fetch`/`tool_call`/`mcp_call` or produced by a compromised sub-agent) propagates through the workflow graph as trusted step output and is injected into a downstream agent's LLM context.
2. **Mitigations at the DSL level**: introduce an optional `trust_level: external | internal` annotation on step outputs. Data from `tool_call`/`mcp_call`/`fetch` steps is tagged `external` by default. Downstream `llm_call` steps receiving `external` data are prompted (via an injected system note) to treat that content as untrusted.
3. **Mitigations at the agent boundary**: `agent_call` output is treated as `internal` by default (the called agent is a trusted platform artifact). But a namespace-level setting can mark specific external-facing agents as `external` trust level.
4. **Mitigation at provisioning**: the validation service warns when an `llm_call` step receives data from a chain that includes `external` sources without an intervening `transform` (sanitization) step.

**Impacts**: `09-security-model.md`, `14-dsl-spec.md`, `05-v4`, `01-PRD.md` (new threat entry).

---

### E-22 — Namespace quota enforcement has a race condition under concurrent invocations

**Anchored to**: `16-multi-tenancy.md` §Quota Enforcement  
**Priority**: `in-phase` (during v1)

**Gap**: The quota enforcement spec states: "`max_concurrent_runs`: Invocation API: reject if active run count ≥ limit." And: "Quota counters are maintained in the database (not in-memory) so they survive service restarts and work correctly across API instances." The gap: under concurrent invocations, two API instances can simultaneously read `active_run_count = 49` (against a limit of 50), both allow a new run, and the count becomes 51 — a classic TOCTOU (time-of-check, time-of-use) race condition. Database-level quota counters do not automatically prevent this without an atomic increment-and-check operation.

**Proposal**: Specify that quota enforcement uses database-level atomic operations:
1. `max_concurrent_runs` is enforced via a `SELECT ... FOR UPDATE` on the quota counter row, or an `UPDATE quotas SET concurrent_runs = concurrent_runs + 1 WHERE concurrent_runs < max_concurrent_runs RETURNING concurrent_runs` pattern that atomically increments and checks in one statement.
2. If the atomic update affects 0 rows (the quota was already at the limit), the API returns `429 Too Many Requests`.
3. On workflow completion, the counter is decremented atomically.
4. Document this pattern in `16-multi-tenancy.md` so implementers choose the correct SQL primitive.

**Impacts**: `16-multi-tenancy.md`, `02-v1-agent-provisioning-foundation.md` (schema established in v1).

---

## Layer 8: Developer Experience and Tooling

### E-23 — The `dry-run` command's output is not specified

**Anchored to**: `15-developer-tooling.md` §`agent-platform dry-run`  
**Priority**: `in-phase` (during v3, when dry-run is built)

**Gap**: `15-developer-tooling.md` documents `agent-platform validate` exhaustively (full output format, JSON mode, error structure). The `dry-run` command is mentioned and its existence is implied (referenced in the context of "without requiring a running platform instance") but it is never specified: what does it output? Does it simulate the data flow step-by-step? Does it mock LLM calls? Does it use the cassette mechanism from `11-testing-strategy.md`? Is the output a trace? A summary? A step-by-step breakdown?

**Proposal**: Add a full `agent-platform dry-run` specification to `15-developer-tooling.md` matching the depth of the `validate` command:

```
agent-platform dry-run <file> --input <json-file> [options]

Options:
  --namespace <ns>      Namespace context for registry lookups
  --mock-llm <file>     JSON file of mock LLM responses per step (matched by step ID)
  --cassette <dir>      Use recorded cassettes from the testing cassette directory
  --live-llm            Use real LLM providers (requires credentials configured)
  --estimate-cost       Print token and cost estimates per step
  --json                Machine-readable output
```

Output: a step-by-step execution trace showing resolved inputs, mock/estimated outputs, data flow between steps, and cost estimate. Each step shows its `type`, resolved `inputs`, simulated `outputs`, and any validation warnings.

**Impacts**: `15-developer-tooling.md`, `11-testing-strategy.md` (cassette integration).

---

### E-24 — No documented workflow for adding a new LLM provider

**Anchored to**: `02-v1-agent-provisioning-foundation.md` Epic 4; `01-PRD.md` AD-09; `04-v3-tools-and-mcps.md`  
**Priority**: `in-phase` (during v1)

**Gap**: The PRD and v1 documents specify that new providers are added by implementing a provider adapter. But there is no documented procedure: what interface must the adapter implement? What tests must it pass? How is it registered in the platform? How does the validation service know a new provider slug is valid? How does the `probe()` method work for a new provider? A team member adding a Grok or Gemini adapter has no reference beyond "implement the interface" — which is described in prose, not as a code contract.

**Proposal**: Add a `## Adding a New Provider` runbook section to `15-developer-tooling.md` (or a standalone `18-provider-integration-guide.md`):
1. The exact interface contract (methods, return types, error types) in pseudocode.
2. The required unit tests: `test_completion_success`, `test_completion_rate_limit_error`, `test_completion_timeout`, `test_tool_call_round_trip`, `test_streaming_mode`, `probe_returns_available`.
3. How to register the new provider slug in the platform's provider enum (DSL validation will then accept it).
4. How to add the provider's cassette directory for CI.
5. An example: "Adding the `mistral` provider looks like this."

**Impacts**: `15-developer-tooling.md`, `02-v1`, `14-dsl-spec.md` (provider enum), `11-testing-strategy.md`.

---

## Layer 9: Roadmap and Planning

### E-25 — AD-27 (backend language choice) must be resolved and recorded

**Anchored to**: `01-PRD.md` AD-27; §8 Technology Stack  
**Priority**: `pre-build` (immediately, before v1)

**Gap**: AD-27 states the backend language "must be decided before end of v1 planning" and is still marked open. This is not just an implementation detail: the language choice directly determines the Temporal SDK (TypeScript's `@temporalio/client` vs Python's `temporalio`), the JSON Schema validation library, the streaming SSE/WebSocket approach, the `agent-platform` CLI implementation, and all code examples in the spec documents. Every phase document's "Technical Considerations" section is language-agnostic by necessity — but implementation teams will need language-specific guidance from v1 onward. Leaving this open creates speculative work and, in a team setting, parallel diverging prototypes.

**Proposal**: The decision must be made and recorded as an update to `01-PRD.md` AD-27 and §8 Technology Stack before v1 implementation begins. Once decided, update:
1. `15-developer-tooling.md`: the CLI installation method and language runtime requirements.
2. `11-testing-strategy.md`: specific test framework (jest/vitest for TypeScript; pytest for Python).
3. `02-v1` Technical Considerations: repository structure references (module names, package manager).

This is the single highest-priority gap in the entire documentation set.

**Impacts**: `01-PRD.md`, all phase documents, `15-developer-tooling.md`, `11-testing-strategy.md`.

---

### E-26 — A v7 sketch is needed to prevent "out of scope" items from being permanently deferred

**Anchored to**: `01-PRD.md` §10 Out of Scope; `README.md` §Out of Scope  
**Priority**: `post-v6`

**Gap**: Six items are explicitly parked as "out of scope" or "future": custom DSL parser/compiler, LangFuse integration, provisioning UI, event-driven orchestrator (message bus), full ticketing integration for human handoff, and per-agent provider cost tracking. None of these have a planned home. They are not backlog items in any roadmap document; they are simply deferred to "Future" or "Post-v6." Without a home, they will remain permanently deferred or will be added ad-hoc without architectural consideration.

**Proposal**: Create `19-v7-sketch.md` — a one-page planning stub that:
1. Lists the deferred items with a short description of what activating each would require.
2. Proposes a v7 theme ("Developer Experience, Observability Depth, and Event Infrastructure") that naturally encompasses the custom DSL, LangFuse integration, and event bus.
3. Notes which deferred items have hooks already in place (LangFuse interface in v6, `CredentialsProvider` abstraction) vs. which would require new design work (custom DSL grammar, UI).
4. Is explicitly labeled as a sketch, not a committed plan.

This converts "permanently deferred" into "planned but not yet scheduled."

**Impacts**: `01-PRD.md` (update Out of Scope table with v7 references), `README.md`.

---

## Layer 10: Remaining Residual Gaps

### E-27 — Provider failover has no DSL field or policy declaration

**Anchored to**: `08-performance-and-scaling.md` §Failure Mode Impact table; `02-v1-agent-provisioning-foundation.md` Epic 4; `01-PRD.md` AD-09  
**Priority**: `in-phase` (during v3, when provider abstraction is mature)

**Gap**: The failure mode table in `08-performance-and-scaling.md` states: "Provider outage → Failover to backup provider (if configured)." This is the only mention of provider failover across the entire documentation set. There is no DSL field to declare a backup provider, no platform configuration for it, no specification of what "failover" means (automatic retry on the backup, round-robin, priority order), no definition of which error types trigger failover vs. simple retry (transient rate limit vs. full provider outage), and no documented impact on in-flight `llm_call` steps mid-streaming when a failover occurs. As written, "if configured" is a feature promise with zero implementation path.

**Proposal**: Add an optional `fallback_provider` block to the agent's top-level `model_config`:

```json
{
  "model_config": {
    "model_name": "claude-3-7-sonnet-20250219",
    "temperature": 0.3,
    "max_tokens": 2048,
    "fallback_provider": {
      "provider": "openai",
      "model_name": "gpt-4o",
      "trigger_on": ["ProviderUnavailableError", "RateLimitError"],
      "max_attempts": 1
    }
  }
}
```

Specify the failover contract:
1. `trigger_on` lists the error classes that activate failover. Transient errors (e.g., a single timeout) retry on the primary first (per the step's `retry` policy); only after exhausting primary retries does the fallback engage.
2. Failover is attempted once. If the fallback also fails, the step fails with the fallback's error.
3. The step run record stores which provider ultimately served the request (`provider_used: "openai"`), separate from the agent's declared `provider`.
4. Mid-stream failover (a provider drops the connection during streaming) is **not** supported in v3. Failover applies only at the activity invocation boundary. Document this limitation explicitly.
5. The validation service checks that `fallback_provider.provider` is configured in the platform (same static check as the primary provider, per E-06).

**Impacts**: `14-dsl-spec.md`, `08-performance-and-scaling.md`, `02-v1` (provider adapter interface), `04-v3`, `01-PRD.md` (update AD-09).

---

### E-28 — Multi-device conversation conflict resolution is undecided

**Anchored to**: `13-chatbot-ux-edge-cases.md` §2 Multi-Device Conversations  
**Priority**: `in-phase` (during v5)

**Gap**: §2 describes two conflicting options for the same scenario — "last-write-wins" or "user-visible conflict" — without committing to either. This is not guidance; it is a deferred decision presented as a conceptual approach. The choice has direct implementation consequences: last-write-wins requires no locking and is simple to implement but silently drops messages; user-visible conflict requires an active-session check, a per-conversation `active_device_session_id` field, and a UI affordance. Without a decision, v5 implementers will make inconsistent choices across the streaming layer, the conversation state machine, and the notification system.

**Proposal**: Make an explicit decision (add to `01-PRD.md` as PD-08 or a note in `06-v5-chatbot-layer.md`) — **recommended: last-write-wins with active-session advisory**:

1. Each conversation tracks an `active_session_id` (set when a client opens a streaming connection). On reconnect from a different device, the new session becomes active and the old session receives a `SESSION_SUPERSEDED` event (the old client's UI shows "This conversation was opened on another device").
2. If two devices send messages within the same 500ms window (true simultaneous conflict), last-write-wins: the second message's `sent_at` timestamp determines ordering. Both messages are persisted and visible in history.
3. Notifications are delivered to the `active_session_id`'s device. Other devices receive a background push ("New message in your conversation") without the full streaming response.
4. Add `active_session_id` and `last_active_device` to the `conversations` table schema in `06-v5-chatbot-layer.md` Epic 1 Story 1.1.

This avoids complex distributed locking while giving users a clear signal when devices conflict.

**Impacts**: `13-chatbot-ux-edge-cases.md`, `06-v5-chatbot-layer.md`, `01-PRD.md`.

---

### E-29 — Authorization system selection is perpetually deferred with no decision gate

**Anchored to**: `09-security-model.md` §Authentication and Authorization, §Out of Scope; `16-multi-tenancy.md` §API Impact  
**Priority**: `pre-build` (before v1 API implementation)

**Gap**: `09-security-model.md` defines the persona × permission table clearly (Platform Admin, Agent Builder, End User, Operator), then states: "This is a conceptual model. Implementation may map to RBAC, ABAC, or custom authorization systems." The Out of Scope section lists "Specific RBAC system (RBAC, ABAC, OPA, etc.)" as deferred to implementation. This deferral is appropriate at the conceptual level, but the authorization system is not purely an implementation detail — it determines the API contract: whether tokens carry roles or scopes, whether the API enforces permissions via middleware or per-handler, and whether the multi-tenancy model (namespace-scoped tokens) is expressed as RBAC roles or as JWT claims. `16-multi-tenancy.md` §Authentication Binding mentions "API tokens scoped to a namespace" and "Platform Admins carry a super-token" — these are authorization decisions that must be made before v1 API routes are built, not after.

**Proposal**: Add a `## Authorization Model` section to `09-security-model.md` that makes the following decisions concrete (while keeping technology selection deferred):

1. **Token structure (technology-agnostic)**: every API token carries at minimum `{ namespace_id, persona, scopes[] }`. The `scopes` array is the enforcement primitive — not free-form roles. Define the canonical scope list:
   - `agents:read`, `agents:write` — read/create/update agent definitions
   - `runs:read`, `runs:write` — read/invoke workflow runs
   - `runs:control` — pause, resume, cancel (Operator-only)
   - `credentials:read`, `credentials:write` — manage credential entries
   - `namespaces:admin` — Platform Admin super-scope

2. **Enforcement layer**: authorization is enforced in a single middleware layer at the API edge. Per-handler permission checks are not permitted — all authorization logic lives in one auditable place.

3. **Namespace super-token**: Platform Admin tokens carry `namespace_id: "*"` (wildcard). All namespace isolation checks treat `"*"` as matching any namespace.

4. **Technology decision gate**: the specific implementation (JWT with scope claims, opaque tokens with a lookup table, OPA, etc.) is decided before v1 API implementation begins — recorded as AD-28 (or renumber appropriately). The scope list above is the contract regardless of implementation.

This converts "authorization system is deferred" into "authorization contract is defined; implementation technology is deferred" — which is the correct boundary.

**Impacts**: `09-security-model.md`, `16-multi-tenancy.md`, `02-v1-agent-provisioning-foundation.md`, `01-PRD.md` (new AD).

---

---

## E-30 — Provider Rate Limit Strategy

**Layer**: Providers | **Priority**: pre-build (v3)

**Gap**: `08-performance-and-scaling.md` lists `llm_call` latency budgets and `14-dsl-spec.md` defines the `retry` block, but neither addresses 429 (rate limit) responses as a distinct error class. Generic exponential backoff on rate limit errors causes thundering-herd re-triggering across concurrent activities. The `Retry-After` header — the provider's authoritative signal for when to retry — is never mentioned.

**Enhancement**: Define a platform-level per-provider, per-namespace **token bucket** that activities must acquire a permit from before calling a provider. Classify provider errors into `RateLimitError` (retryable, respect `Retry-After`) vs `QuotaExceededError` (non-retryable, daily limit exhausted). Add `rate_limit_max_attempts` and `rate_limit_honor_retry_after` fields to the DSL `retry` block. Add a pre-flight TPM check at invocation time for large workflows.

**Resolution**: Fully specified in `19-provider-rate-limiting.md`. New AD-28 added to `01-PRD.md`. Prometheus metrics and Grafana panel specified. Interaction with E-27 (failover) documented — rate-limited requests may trigger provider failover when the fallback has capacity.

**Impacts**: `19-provider-rate-limiting.md` (new), `14-dsl-spec.md` (retry block extension), `08-performance-and-scaling.md`, `07-v6-versioning-and-hardening.md` (metrics), `01-PRD.md` (AD-28).

---

## E-31 — Agent Version Promotion Lifecycle

**Layer**: Versioning | **Priority**: in-phase (v6)

**Gap**: `07-v6-versioning-and-hardening.md` Story 1.1 states that creating a new agent version produces an immutable record — but immediately and silently updates the `latest` pointer. There is no staging gate, canary traffic split, or rollback trigger. For production platforms, "save and hope" is not acceptable.

**Enhancement**: Introduce a four-state promotion lifecycle: `draft → staging → canary → production`. Creating a new version puts it in `draft`; the `latest` pointer is not updated until explicit promotion to `production`. Canary state supports configurable traffic splits, auto-rollback thresholds (error rate, latency P95), and auto-promotion on success. A/B evaluation mode supports quality-metric comparison between versions using a judge model.

**Resolution**: Fully specified in `20-agent-version-promotion.md`. New AD-29 added to `01-PRD.md`. `agent_versions` table gains a `state` column. Operator CLI gains `agents promote`, `agents canary status/abort`, and `agents ab-test` commands. Chatbot session scoping (pin version for conversation lifetime) documented.

**Impacts**: `20-agent-version-promotion.md` (new), `07-v6-versioning-and-hardening.md` (Epic 1 stories), `06-v5-chatbot-layer.md` (session-scoped version pinning), `17-operator-cli.md` (new commands), `01-PRD.md` (AD-29).

---

## E-32 — Conversation Memory: Episodic and Semantic Tiers

**Layer**: Chatbot | **Priority**: in-phase (v5)

**Gap**: `06-v5-chatbot-layer.md` Epic 6 handles working memory (the current conversation context window). It says nothing about memory across sessions. A chatbot that cannot recall that a user had a billing dispute last week, or that their account tier is "premium," is not production-grade for customer support.

**Enhancement**: Define a three-tier memory model. **Working memory** (existing). **Episodic memory**: at conversation end, generate a session summary and store it as a Qdrant vector point keyed to the user, retrieved semantically on the next session's first turn. **Semantic memory**: a `user_profiles` database table holding structured facts about users (name, account tier, preferences), writable by agents via a `user_profile_update` registered tool and readable via `user_profile_lookup`. Auto-injection of profile facts into the routing chatbot's context. GDPR right-to-erasure extended across all three tiers.

**Resolution**: Fully specified in `21-conversation-memory.md`. New AD-30 added to `01-PRD.md`. New `user_profiles` table and two registered tools specified. DSL `episodic_memory` and `semantic_memory` blocks defined.

**Impacts**: `21-conversation-memory.md` (new), `06-v5-chatbot-layer.md` (new stories in Epics 1, 2, 6), `09-security-model.md` (GDPR erasure scope), `14-dsl-spec.md` (chatbot DSL additions), `16-multi-tenancy.md` (user_profiles table), `01-PRD.md` (AD-30).

---

## E-33 — Chatbot Graceful Degradation and Circuit Breakers

**Layer**: Chatbot | **Priority**: in-phase (v5)

**Gap**: `06-v5-chatbot-layer.md` and `13-chatbot-ux-edge-cases.md` address individual turn-level failures. Neither specifies system-level degraded mode: what the user experience is when a specialist fails repeatedly, when the KB is unavailable, or when both the routing chatbot and its fallback are down. Without explicit degradation design, any sustained downstream failure cascades silently to the user as unexplained errors.

**Enhancement**: Introduce per-specialist and per-KB circuit breakers (CLOSED / HALF_OPEN / OPEN state machine) with configurable failure thresholds and open durations. When a specialist's circuit is OPEN, the routing chatbot bypasses it and uses a `degraded_message` or routes to `default_specialist`. When the KB circuit is OPEN, specialists operate in a configurable degraded mode (`skip`, `cached`, `block`). Full system degraded mode (all circuits OPEN) activates static fallback and enables all human handoff paths automatically.

**Resolution**: Fully specified in `22-chatbot-degradation.md`. New AD-31 added to `01-PRD.md`. DSL additions for routing chatbot `default_specialist` and specialist `degraded_message` fields. Prometheus metrics for circuit state and fallback counts. CLI `chatbot circuit-breakers` command.

**Impacts**: `22-chatbot-degradation.md` (new), `06-v5-chatbot-layer.md` (new stories in Epics 2, 3, 4), `13-chatbot-ux-edge-cases.md`, `08-performance-and-scaling.md` (failure mode table), `07-v6-versioning-and-hardening.md` (metrics), `01-PRD.md` (AD-31).

---

## E-34 — Per-Run Token/Cost Cap (`budget` DSL Field)

**Layer**: Workflow/Safety | **Priority**: pre-build (v4)

**Gap**: The existing `while` loop guard (`max_iterations`) and parallel block guard (`max_concurrency`) prevent structural runaway. They do not prevent token runaway: a loop within its iteration cap can still consume unbounded tokens if each iteration makes a large LLM call. There is no mid-run kill switch. Without this, a misconfigured agent can consume hundreds of dollars before any operator notices. The namespace daily quota (E-18) prevents starting new runs after the daily limit is hit — it does not stop a single run already in progress.

**Enhancement**: Add a `budget` top-level DSL field for workflow agents with `max_total_tokens`, `max_estimated_cost_usd`, and `on_exceeded` (cancel / warn_and_continue / cancel_with_partial_output). Enforcement happens in the Temporal workflow after each `llm_call` step, using a workflow variable tracking accumulated tokens and cost against a platform-maintained price table. Namespace-level default budgets apply to agents without explicit `budget` declarations. Validation service warns when the declared budget is guaranteed to be exceeded.

**Resolution**: Fully specified in `23-per-run-cost-cap.md`. New AD-32 added to `01-PRD.md`. Price table API for Platform Admins. `--estimate-cost` flag for `agent-platform dry-run`. Prometheus metrics for budget-exceeded events and run cost distribution.

**Impacts**: `23-per-run-cost-cap.md` (new), `14-dsl-spec.md` (budget field), `05-v4-multi-agent-orchestration.md` (loops/fan-out cost risk), `16-multi-tenancy.md` (namespace default budget), `15-developer-tooling.md` (dry-run flag), `07-v6-versioning-and-hardening.md` (metrics), `01-PRD.md` (AD-32).

---

## E-35 — DSL Migration Tooling

**Layer**: Tooling | **Priority**: in-phase (v3)

**Gap**: `14-dsl-spec.md` states DSL evolution is "additive only — never removes or renames." `12-upgrades-and-migrations.md` mentions "utilities assist Agent Builders" for DSL migration but provides no specification of what those utilities are, how they work, or what migration risk levels exist. When the `fetch` step is deprecated (E-03) or the expression grammar is formalized (E-01), there is no defined path for Agent Builders to migrate 50+ existing DSL files safely.

**Enhancement**: Specify an `agent-platform migrate` CLI command that: runs the migration analyzer against a DSL file (identifying deprecated syntax, required changes, and advisory improvements); produces a dry-run report with per-change risk levels (NONE / LOW / MEDIUM / HIGH / MANUAL); applies automated migrations; supports batch migration across an entire namespace (`--all-agents`). Define a versioned migration catalog. Specify a compatibility shim framework for hard-breaking changes (rare, expected only for E-01 expression grammar formalization).

**Resolution**: Fully specified in `24-dsl-migration-tooling.md`. Integration with `20-agent-version-promotion.md`'s draft version state: `--create-draft-versions` applies migrations without overwriting current production versions. Deprecation lifecycle phases (warning → error → removal) formalized.

**Impacts**: `24-dsl-migration-tooling.md` (new), `14-dsl-spec.md` (deprecation lifecycle), `15-developer-tooling.md` (new `migrate` command), `12-upgrades-and-migrations.md` (migration utilities section), `20-agent-version-promotion.md` (draft version integration).

---

## E-36 — Shared Agent Catalogue

**Layer**: Platform/Multi-tenancy | **Priority**: in-phase (v5)

**Gap**: `16-multi-tenancy.md` defines a cross-namespace allowlist for bilateral agent sharing (a security primitive, not a discoverability feature). Tool Registry and MCP Registry both have platform-level tiers. Agents have no equivalent. Every namespace that needs a `language-detector` or `pii-scrubber` must build and maintain its own, undermining PD-01 ("platform as meta-product").

**Enhancement**: Define a Shared Agent Catalogue — a platform-level registry of agent definitions published across namespaces. Catalogue entries have a `slug`, `category`, `tags`, `input_schema`, `output_schema`, and `visibility` (public/private). Agents reference catalogue entries via `catalogue://slug` URI in `agent_call` steps. A `platform-utilities` namespace holds all platform-published agents. A starter library of 7 reference agents ships with the platform (language-detector, text-summarizer, pii-scrubber, quality-scorer, intent-classifier, document-chunker, schema-validator). Token costs for catalogue agent invocations are charged to the caller's namespace.

**Resolution**: Fully specified in `25-shared-agent-catalogue.md`. CLI `catalogue list/inspect/call` commands. Catalogue API (browse, inspect). Validation service resolves `catalogue://` references online; CLI warns when offline.

**Impacts**: `25-shared-agent-catalogue.md` (new), `16-multi-tenancy.md` (catalogue alongside tool/MCP scoping), `05-v4-multi-agent-orchestration.md` (`agent_call` step URI scheme), `14-dsl-spec.md` (`catalogue://` reference syntax), `17-operator-cli.md` (catalogue commands).

---

## E-37 — `tags` / `labels` Field on Agent DSL Top Level

**Layer**: DSL/Ops | **Priority**: pre-build (v1)

**Gap**: Agent definitions have `name` and `description` but no structured metadata for organizational categorization. At scale (100+ agents per namespace), there is no way to filter agents by domain, ownership team, or lifecycle status. Cost reporting (E-18) cannot be attributed by team. Bulk deprecation operations cannot target logical groups.

**Enhancement**: Add an optional `tags` field to the top-level DSL: `"tags": ["billing", "team-payments", "production-critical"]`. Tags are a flat string array, namespace-scoped, and indexed in the agents table. The API supports `GET /namespaces/{ns}/agents?tag=billing` filtering. Workflow run queries support `tag` filtering for cost attribution by domain. The `agent-platform agents list --tag` CLI flag is added. No DSL version bump required — additive to v1 and all subsequent versions.

**Resolution**: Update `14-dsl-spec.md` top-level field table. Add `tags TEXT[]` column to the `agents` table schema. Update `15-developer-tooling.md` CLI reference. Update `16-multi-tenancy.md` API filtering section.

**Impacts**: `14-dsl-spec.md` (top-level field addition), `15-developer-tooling.md` (agents list flag), `16-multi-tenancy.md` (API filtering), `02-v1-agent-provisioning-foundation.md` (schema story).

---

## E-38 — Polling Orchestrator Scale Trigger and Event-Bus Migration Gate

**Layer**: Workflow/Ops | **Priority**: pre-build (v6)

**Gap**: AD-05 explicitly chose polling over an event bus ("an additive change later"). `08-performance-and-scaling.md` never quantifies the polling interval or its database write-amplification impact under load. `03-v2-workflow-orchestration.md` notes the polling interval is "configurable" but provides no default, no latency impact analysis, and no criterion for when to migrate to an event bus. The event bus migration floats as "future" in the out-of-scope table with no gate condition.

**Enhancement**: (1) Document the default polling interval (`5 seconds`) and its maximum latency contribution to workflow state detection. (2) Specify the DB query pattern under polling (SELECT WHERE status IN ('pending','running') ORDER BY updated_at LIMIT N) and its index requirements. (3) Define explicit gate conditions that trigger event-bus evaluation: `>500 concurrent workflow runs` or `DB polling query latency >200ms p95`. (4) Add the event-bus migration as a named item in `12-upgrades-and-migrations.md` with a migration pattern — not just "future optimization."

**Resolution**: Update `03-v2-workflow-orchestration.md` (Story 4.1) with polling interval default and DB query spec. Update `08-performance-and-scaling.md` with polling write-amplification analysis. Add event-bus migration section to `12-upgrades-and-migrations.md`. Update AD-05 in `01-PRD.md` with gate conditions.

**Impacts**: `03-v2-workflow-orchestration.md`, `08-performance-and-scaling.md`, `12-upgrades-and-migrations.md`, `01-PRD.md` (AD-05 annotation), `10-deployment-concepts.md`.

---

## E-39 — Agent Deprecation Notification Channel

**Layer**: Versioning/Ops | **Priority**: in-phase (v6)

**Gap**: `07-v6-versioning-and-hardening.md` Story 1.3 defines version deprecation as setting a flag and optionally blocking traffic. PD-07 states "soft signal first, then hard block." But who receives the soft signal, and through what channel? An Agent Builder who hasn't touched their agent in 3 months will not notice a `deprecated: true` flag in the database. There is no notification path — making the graceful deprecation model operationally hollow.

**Enhancement**: Define a `deprecation_notices` system: when a version (or catalogue agent, E-36) is deprecated, the platform emits a notification to configured channels. Configure per-namespace `deprecation_notification` settings: `webhook_url` (POST JSON with version info and grace period), `email` (if email service is wired), or `polling` (expose a `GET /namespaces/{ns}/deprecation-notices` API that operators can poll). Notices include: which agent version is deprecated, effective date of hard-block, number of runs in the past 30 days (to prioritize urgency), and a migration guide link. Operator CLI: `agent-platform agents deprecation-notices --namespace <ns>`.

**Resolution**: Add `deprecation_notices` table to schema. Add `GET /namespaces/{ns}/deprecation-notices` API endpoint. Add `deprecation_notification` block to namespace settings. Update `07-v6-versioning-and-hardening.md` Story 1.3. Update `17-operator-cli.md` with `deprecation-notices` command. Reference in `20-agent-version-promotion.md`.

**Impacts**: `07-v6-versioning-and-hardening.md` (Story 1.3 extension), `17-operator-cli.md` (new command), `16-multi-tenancy.md` (namespace settings), `20-agent-version-promotion.md` (deprecation flow), `25-shared-agent-catalogue.md` (catalogue deprecation).

---

## Summary Table

| ID | Enhancement | Layer | Priority | Documents |
|---|---|---|---|---|
| E-01 | Formal expression language grammar | DSL | pre-build (v4) | 14, 05, 11, 15 |
| E-02 | DSL v5 chatbot spec in 14-dsl-spec | DSL | in-phase (v5) | 14, 06, 15 |
| E-03 | `fetch` step fate: keep or deprecate | DSL | pre-build (v3) | 14, 04, 01 |
| E-04 | Workflow-level `input_schema` field | DSL | pre-build (v2) | 14, 03, 05 |
| E-05 | Cross-agent graph validation path | Validation | in-phase (v4) | 01, 05, 14, 15 |
| E-06 | Provider check: static vs. live split | Validation | pre-build (v1) | 02, 11 |
| E-07 | Per-step `timeout_seconds` | Workflow | pre-build (v2) | 14, 08, 03, 04 |
| E-08 | Temporal/DB reconciliation contract | Workflow | pre-build (v2) | 10, 12, 03 |
| E-09 | `parallel` `max_concurrency` required | Workflow | in-phase (v4) | 14, 05, 08 |
| E-10 | Fire-and-forget orphan failure policy | Workflow | in-phase (v4) | 14, 05, 06, 13 |
| E-11 | OAuth token lifecycle for MCPs | Tools/MCPs | pre-build (v5) | 04, 09, 06, 14 |
| E-12 | Tool idempotency declaration | Tools/MCPs | in-phase (v3) | 04, 14, 02 |
| E-13 | MCP manifest cache invalidation | Tools/MCPs | in-phase (v3) | 04, 14, 15 |
| E-14 | Routing confidence model as AD | Chatbot | pre-build (v5) | 06, 13, 08, 01 |
| E-15 | Context window `summarize` semantics | Chatbot | in-phase (v5) | 06, 13, 14 |
| E-16 | PII handling policy with right-to-erasure | Chatbot | in-phase (v5) | 09, 06, 13, 16 |
| E-17 | Webhook delivery receipt and metrics | Chatbot | in-phase (v5) | 06, 07 |
| E-18 | Cost tracking and token budget | Versioning/Ops | in-phase (v6) | 16, 07, 08, 15, 14 |
| E-19 | LLM observability includes prompt template | Versioning/Ops | in-phase (v6) | 07, 01 |
| E-20 | Debug override schema validation | Versioning/Ops | in-phase (v6) | 07, 17, 14, 11 |
| E-21 | Inter-agent data poisoning threat | Security | pre-build (v4) | 09, 14, 05, 01 |
| E-22 | Quota enforcement race condition | Multi-tenancy | in-phase (v1) | 16, 02 |
| E-23 | `dry-run` command specification | Tooling | in-phase (v3) | 15, 11 |
| E-24 | New provider integration runbook | Tooling | in-phase (v1) | 15, 02, 14, 11 |
| E-25 | Resolve AD-27 (backend language) | Roadmap | **immediately** | 01, all phases, 15, 11 |
| E-26 | v7 planning sketch | Roadmap | post-v6 | 01, README |
| E-27 | Provider failover DSL field and policy | Workflow/Providers | in-phase (v3) | 14, 08, 02, 04, 01 |
| E-28 | Multi-device conversation conflict resolution | Chatbot | in-phase (v5) | 13, 06, 01 |
| E-29 | Authorization scope contract before v1 build | Security | **immediately** | 09, 16, 02, 01 |
| E-30 | Provider rate limit strategy (token bucket, error classification) | Providers | pre-build (v3) | 19, 14, 08, 07, 01 |
| E-31 | Agent version promotion lifecycle (draft→staging→canary→production) | Versioning | in-phase (v6) | 20, 07, 06, 17, 01 |
| E-32 | Conversation memory: episodic and semantic tiers | Chatbot | in-phase (v5) | 21, 06, 09, 16, 14 |
| E-33 | Chatbot graceful degradation and circuit breakers | Chatbot | in-phase (v5) | 22, 06, 13, 08, 07 |
| E-34 | Per-run token/cost cap (`budget` DSL field) | Workflow/Safety | pre-build (v4) | 23, 14, 05, 16, 07 |
| E-35 | DSL migration tooling (`agent-platform migrate`) | Tooling | in-phase (v3) | 24, 14, 15, 12 |
| E-36 | Shared Agent Catalogue (`catalogue://` references) | Platform/Multi-tenancy | in-phase (v5) | 25, 16, 05, 14, 17 |
| E-37 | `tags` / `labels` field on agent DSL top level | DSL/Ops | pre-build (v1) | 14, 16, 01 |
| E-38 | Polling orchestrator scale trigger and event-bus migration gate | Workflow/Ops | pre-build (v6) | 03, 08, 01, 10 |
| E-39 | Agent deprecation notification channel | Versioning/Ops | in-phase (v6) | 07, 17, 20, 01 |
