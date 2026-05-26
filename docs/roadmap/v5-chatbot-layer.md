# v5 — Chatbot Layer

## Version Goal

Deliver the **real-time, user-facing conversational layer** that sits on top of the agent platform: a unified chat surface backed by multiple specialized streaming chatbots. A **routing chatbot** classifies each user turn and delegates to the appropriate specialist (payments, support, technical, administrative, etc.). Specialists never run heavy work inline — they delegate to backend agents (from v1–v4) and report results back through the conversation.

This phase also integrates the existing **Qdrant knowledge base** as a first-class capability and introduces the conversational pattern for handling asynchronous background work without breaking the user experience.

## Business Value

- The platform now has a complete end-to-end product story: a website operator provisions a chatbot, configures specialist behaviors, connects a knowledge base, and gets a production-ready conversational system that delegates complex work behind the scenes.
- Users get a single, unified chat experience with intelligent routing — they never see the underlying complexity.
- Long-running background work (workflows, multi-agent calls) doesn't block conversation: the chatbot keeps talking while jobs run, then proactively surfaces results.

---

## Epic 1 — Streaming Chatbot Infrastructure

**Business Value**: Synchronous streaming conversation is fundamentally different from asynchronous workflows. This epic builds the infrastructure for real-time interaction.

### Story 1.1 — Streaming chat endpoint
**As an** End User<br/>
**I want** to send a message and receive a token-by-token streaming response<br/>
**So that** the experience feels immediate and responsive.

**Description**: Implement a streaming endpoint (Server-Sent Events or WebSocket) that accepts a user message and conversation context, then streams the LLM response back. Uses the v1 provider abstraction with streaming-enabled methods added to each adapter.

**Acceptance Criteria**:
- The endpoint streams tokens with low time-to-first-token.
- Streaming works for at least two LLM providers (Claude + one other).
- Connection drops are handled gracefully (partial responses persisted).

### Story 1.2 — Conversation persistence
**As an** Operator<br/>
**I want** every conversation turn persisted with full context<br/>
**So that** we can audit, analyze, and resume conversations.

**Description**: A `conversations` table and `conversation_turns` table. Each turn stores: user message, assistant response, the specialist chatbot that handled it, references to any background workflows triggered, timing.

**Acceptance Criteria**:
- Every message exchange persists.
- A conversation can be fully reconstructed from the database.
- Background workflow references are linked so we can show pending tasks.

### Story 1.3 — Chatbot definition as a special agent type
**As an** Agent Builder<br/>
**I want** to define a chatbot using the same DSL machinery as workflow agents<br/>
**So that** the platform stays consistent and chatbots inherit validation, versioning, and provisioning.

**Description**: Add a chatbot agent type to the DSL. Differences from workflow agents: synchronous streaming execution, no Temporal workflow per turn (the conversation itself is the long-running unit), can delegate to workflow agents via the `agent_call` step semantics adapted for async-with-callback.

**Acceptance Criteria**:
- A chatbot is provisioned through the same admin API as workflow agents.
- It is versioned identically.
- It uses tools/MCPs through the same registries.

---

## Epic 2 — Routing Chatbot

**Business Value**: The single entry point that gives users a unified interface and intelligently dispatches to the right specialist.

### Story 2.1 — Intent classification
**As an** End User<br/>
**I want** my message to be understood and handled by the right specialist automatically<br/>
**So that** I don't have to know which department to talk to.

**Description**: The routing chatbot inspects each user turn (with conversation history context) and decides which specialist chatbot should handle it. Classification can be a quick LLM call with a strict, schema-constrained output or a dedicated classifier. The decision is recorded per turn.

**Acceptance Criteria**:
- A new user turn routes to a specialist with measurable accuracy on a test set.
- The routing decision and rationale are persisted.
- Misclassification is correctable via user clarification.

### Story 2.2 — Per-turn re-routing
**As an** End User<br/>
**I want** the system to follow my conversation as the topic shifts<br/>
**So that** I can move from one subject to another without restarting.

**Description**: Routing happens on every user turn, not once per conversation. The router considers prior context but is not locked in. Handover between specialists is seamless; the user does not perceive a switch beyond perhaps a brief acknowledging line.

**Acceptance Criteria**:
- Mid-conversation topic changes route to the new specialist.
- Conversation context is shared appropriately (the specialist sees prior turns).
- The user-visible experience remains continuous.

### Story 2.3 — Fallback and graceful escalation
**As an** End User<br/>
**I want** a clear path when no specialist can help<br/>
**So that** I am not stuck in a loop.

**Description**: When the router cannot confidently classify, or when a specialist explicitly cannot help, the conversation falls back to a generic helper or surfaces a human-handoff path (initially a documented response; integration with ticketing is later).

**Acceptance Criteria**:
- Low-confidence classifications trigger fallback behavior.
- Specialists can signal "I cannot help" and route back.
- The fallback is configurable per deployment.

---

## Epic 3 — Specialized Chatbots

**Business Value**: Each domain (payments, support, technical, etc.) gets a chatbot tuned to its language, knowledge base filters, and tool access — quality goes up because the chatbot is focused.

### Story 3.1 — Specialist chatbot template
**As an** Agent Builder<br/>
**I want** a clear template for building specialists<br/>
**So that** every domain chatbot is consistent and predictable.

**Description**: Document and codify the pattern: specialist chatbots have a focused system prompt, a curated set of tools/MCPs, a knowledge base scope (filters in Qdrant), and a defined handoff protocol. Provide at least one fully worked specialist example.

**Acceptance Criteria**:
- A reference specialist (e.g. customer support) is fully implemented.
- The specialist is documented end to end as a build pattern.

### Story 3.2 — Knowledge base scoping per specialist
**As an** Agent Builder<br/>
**I want** to restrict each specialist's knowledge base lookups to a relevant subset<br/>
**So that** answers stay accurate and on-topic.

**Description**: Specialists declare Qdrant filters (collection, metadata filters, etc.) in their definitions. Knowledge base lookups during conversation automatically apply these filters.

**Acceptance Criteria**:
- A specialist's lookups never return out-of-scope results.
- Filter definitions are validated against the Qdrant schema at provisioning.

### Story 3.3 — Delegate to backend agent for heavy work
**As a** Specialist Chatbot<br/>
**I want** to delegate long-running or complex tasks to backend agents<br/>
**So that** the conversation stays snappy and uses the full power of the platform.

**Description**: Specialists invoke backend (workflow) agents via a clean API: send a task, optionally subscribe to its completion. The chatbot does not block waiting; the conversation continues.

**Acceptance Criteria**:
- Delegations create a workflow run linked to the conversation turn.
- The chatbot receives a callback or polls the run state to know when results are ready.
- The user can continue chatting while the work runs.

---

## Epic 4 — Knowledge Base Integration

**Business Value**: Connects the existing Qdrant infrastructure into agents and chatbots as a registered, configurable capability.

### Story 4.1 — Knowledge base as a registered tool/MCP
**As an** Agent Builder<br/>
**I want** to reference the knowledge base from any agent or chatbot via standard mechanisms<br/>
**So that** I don't write Qdrant integration code per agent.

**Description**: Register the knowledge base as a tool (or an MCP if it fits the MCP model) in the v3 registry. The tool accepts query parameters (text, filters, retrieval strategy: semantic, hybrid, etc.) and returns structured results.

**Acceptance Criteria**:
- Any agent or chatbot can reference the KB tool by name.
- Retrieval strategy (semantic vs hybrid) is configurable per call.
- The tool exposes the existing test approaches as configuration options.

### Story 4.2 — KB lookup inside chatbot turns
**As an** End User<br/>
**I want** the chatbot to ground its answers in the knowledge base when relevant<br/>
**So that** I get accurate, source-backed answers.

**Description**: Specialist chatbots use the KB tool either implicitly (via the LLM's native tool-calling) or explicitly (the chatbot's prompt always queries KB before responding). Per-specialist policy.

**Acceptance Criteria**:
- KB lookups happen seamlessly within streaming responses.
- Retrieved snippets are referenceable in the response.
- The KB hits per turn are recorded for analysis.

---

## Epic 5 — Asynchronous Background Tasks in Conversation

**Business Value**: The signature UX feature: users keep talking while heavy work runs in the background, then results are folded back in naturally.

### Story 5.1 — Track pending tasks per conversation
**As an** Operator<br/>
**I want** every conversation to maintain a list of pending background workflows<br/>
**So that** the chatbot can reason about them and surface results.

**Description**: A `conversation_pending_tasks` association: each task has a workflow run ID, a description, a status, and a "recall context" — the conversational snippet to inject when the task completes.

**Acceptance Criteria**:
- A delegation creates a pending task entry.
- The conversation can query its outstanding tasks.
- Completion updates the entry and queues a follow-up surfacing.

### Story 5.2 — Proactive result surfacing
**As an** End User<br/>
**I want** the chatbot to tell me when a background task completes<br/>
**So that** I don't have to ask "is it ready yet?"

**Description**: When a pending task completes, the next chatbot turn (or a push, if streaming is open) surfaces the result with the recall context: "Earlier you asked about X — here is the result …". The mechanism uses the v2 orchestrator's status change events.

**Acceptance Criteria**:
- A completed background task is surfaced in the conversation at the next opportunity.
- The surfacing is contextual, not jarring.
- The user can ask follow-up questions about the result.

**Polling latency and perceived UX**:

Background task surfacing depends on the orchestrator detecting workflow completion (AD-05: polling-based). The poll interval directly determines the maximum delay between a task finishing and the user seeing the result. This latency must be explicitly budgeted:

| Poll Interval | Max Surfacing Delay | Acceptable For |
|---|---|---|
| 1s | ~1s | High-priority conversations (payment confirmations) |
| 5s | ~5s | Standard support workflows |
| 30s | ~30s | Low-priority background batch work |

The orchestrator subscription API (introduced in v2, Epic 4, Story 4.1) should support a **shorter poll interval specifically for conversation-linked tasks** — decoupled from the global workflow polling interval. A configurable `conversation_task_poll_interval_seconds` setting (default: `2`) allows this to be tuned independently.

When the event-bus optimization is implemented in a future release (see AD-05), conversation-linked task completions become push events, eliminating this latency entirely. The subscription API interface is designed to accommodate both polling and push without contract changes.

### Story 5.3 — Explicit wait flow for tasks that block
**As an** End User<br/>
**I want** the chatbot to be honest when something must block the conversation<br/>
**So that** I understand what's happening and can decide to wait.

**Description**: Some tasks fundamentally block (e.g. confirming a payment). In those cases, the chatbot explicitly tells the user what's happening, shows progress where possible, and resumes when ready.

**Acceptance Criteria**:
- Blocking tasks are clearly communicated.
- Progress updates appear at reasonable intervals.
- Cancellation is supported where the underlying task supports it.

---

## Epic 6 — Conversation Context Window Management

**Business Value**: Long conversations and multi-specialist routing push against LLM token limits. Without an explicit strategy, context silently truncates in unpredictable ways — damaging response quality and creating hard-to-diagnose bugs.

### Story 6.1 — Context window budget per specialist
**As an** Agent Builder<br/>
**I want** to declare how much conversation history each specialist uses<br/>
**So that** I can tune the accuracy/cost trade-off per domain.

**Description**: Each specialist chatbot DSL definition declares a `context_window` policy:

```json
{
  "context_window": {
    "strategy": "sliding_window | summarize | semantic",
    "max_turns": 20,
    "max_tokens": 8192,
    "summary_agent": "conversation-summarizer"
  }
}
```

| Strategy | Behaviour |
|---|---|
| `sliding_window` | Include the N most recent turns. Oldest turns are dropped when the budget is exceeded. |
| `summarize` | When turn count or token budget is exceeded, invoke `summary_agent` to compress prior history into a summary block that replaces the dropped turns. |
| `semantic` | Retrieve the K most semantically relevant prior turns to the current query (using vector similarity). Always include the last 2 turns regardless. |

**Acceptance Criteria**:
- Specialists without a declared policy use `sliding_window` with a sensible default (`max_turns: 20`).
- The context actually sent to the provider never exceeds `max_tokens` (accounting for system prompt and response budget).
- `summarize` strategy invokes the declared `summary_agent` as a synchronous agent call and injects the summary into context.
- The truncation/summarization decision is logged per turn for analysis.

### Story 6.2 — Cross-specialist context handoff
**As an** End User<br/>
**I want** the specialist I'm routed to to understand what I previously discussed<br/>
**So that** I don't have to repeat myself when the topic shifts.

**Description**: When the routing chatbot hands off to a different specialist, it includes a configurable amount of prior context — not the full history. The handoff context is a summary of the conversation so far, injected as a system-level note in the specialist's first turn: "This user was previously speaking with [prior specialist] about: [summary]."

**Acceptance Criteria**:
- Specialists receive a concise handoff summary on first activation, not the full prior transcript.
- The handoff summary length is configurable in the routing chatbot's DSL definition.
- Specialists that have been active in this conversation before receive a shorter recap ("Continuing earlier conversation about billing").

---

## Epic 7 — Human Handoff Interface

**Business Value**: The customer support use case requires a path to a human agent when the chatbot cannot resolve an issue. The full ticketing integration is deferred, but the interface must be defined in v5 so the chatbot can always offer a meaningful next step — and so the integration is a configuration change, not a code change.

### Story 7.1 — Configurable human handoff action
**As an** Agent Builder<br/>
**I want** to declare what happens when a specialist cannot help and human assistance is needed<br/>
**So that** users always have a path forward and the integration point is well-defined.

**Description**: Each specialist (and the routing chatbot's fallback) declares a `human_handoff` block in its DSL:

```json
{
  "human_handoff": {
    "strategy": "message | webhook | ticket",
    "message": "I wasn't able to resolve this. Our support team can help — please contact us at support@acme.com or call 1-800-XXX-XXXX.",
    "webhook_url": "https://hooks.acme-internal.com/support-escalation",
    "webhook_payload_template": {
      "conversation_id": "{{conversation.id}}",
      "summary": "{{context.summary}}",
      "customer_id": "{{workflow.input.customer_id}}"
    },
    "include_conversation_summary": true
  }
}
```

| Strategy | Behaviour |
|---|---|
| `message` | Display the configured message to the user. No integration. |
| `webhook` | POST a JSON payload to the configured URL. The platform waits for a `200` acknowledgement. |
| `ticket` | Reserved for future ticketing integration (Jira, Zendesk, etc.) via MCP. Falls back to `message` until implemented. |

**Acceptance Criteria**:
- Every specialist that declares `human_handoff` presents the configured path when it cannot resolve the issue.
- `webhook` strategy fires reliably and logs the response.
- The conversation summary injected into the webhook payload uses the same summarization logic as context window management (Story 6.1).
- `include_conversation_summary: true` attaches a readable transcript excerpt to the handoff payload.
- A specialist that lacks a `human_handoff` declaration falls back to a platform-level default message (configurable per namespace).

### Story 7.2 — Handoff state in conversation record (E-17)
**As an** Operator<br/>
**I want** to see which conversations were escalated to human handoff and whether the delivery succeeded<br/>
**So that** I can measure escalation rates and identify gaps in agent coverage.

**Description**: When a human handoff occurs, the conversation record is updated with `escalated_at`, `escalation_strategy`, `escalation_specialist`, and `escalation_outcome`. Each webhook attempt is logged. This is queryable via the Operator API.

**Escalation outcome values**:
- `webhook_delivered` — the configured URL returned a 2xx response
- `webhook_failed_fallback` — all webhook retries exhausted; fell back to `message` strategy
- `message_displayed` — the `message` strategy was the declared strategy or the fallback
- `ticket_created` — reserved for future ticketing MCP integration

**Webhook delivery log**: each attempt records `attempt_number`, `timestamp`, `http_status`, `response_body` (truncated to 512 bytes), `outcome`.

**Webhook acknowledgement contract**: the receiving system returns any `2xx` status code (body is ignored). Non-2xx or timeout triggers retry (default 3 attempts, configurable).

**Acceptance Criteria**:
- Escalation events are persisted with `escalation_outcome` and queryable.
- The Prometheus metric is `chatbot_escalations_total{namespace, specialist, strategy, outcome}` — `outcome` distinguishes successful webhook delivery from fallback.
- A `webhook_delivery_log` entry exists per attempt.

---

## Technical Considerations (high level)

- **Streaming providers**: each LLM provider has its own streaming API. Adapters must support both streaming and non-streaming modes; non-streaming continues to power workflow agents.
- **Conversation as long-lived state**: conversation context lives in the database, not Temporal. Each turn is a short-lived synchronous interaction.
- **Endpoint topology**: one front-door endpoint for the unified UX; routing happens behind it. Multiple specialist endpoints exist but are internal-facing.
- **Backpressure**: streaming endpoints need backpressure handling — slow clients should not slow down the LLM provider.
- **Reconnect/resume**: long conversations may span sessions. Persist conversation IDs server-side and allow clients to resume.
- **KB integration**: take care to expose filter configuration via DSL such that misconfiguration is caught at provisioning, not on the user's first failed lookup.
- **Notification mechanism for surfacing**: SSE/WebSocket push when the user is online; otherwise persist the surfacing into the conversation so the next turn picks it up.
- **Context window enforcement**: the platform must count tokens before sending to the provider (using the provider's tokenizer or a conservative estimate). Exceeding a provider's context window is a hard error that surfaces badly to users; enforce the budget on our side.
- **Human handoff webhook reliability (E-17)**: webhook deliveries retry on failure (configurable, default 3 attempts). Failed deliveries fall back to the `message` strategy, alert the Operator, and record `escalation_outcome: webhook_failed_fallback`. See Story 7.2 for the full delivery receipt spec.
- **Context window `summarize` strategy semantics (E-15)**: summarization is **eager and proactive** — when the context window manager determines on turn N that turn N+1 will exceed `max_tokens`, it invokes `summary_agent` before sending turn N's response, so the summary is ready before the user replies. This avoids adding per-turn latency for the user. The `max_summary_tokens` field (default: 25% of `max_tokens`) constrains the summary size; the summary block is counted against the context budget. Cross-specialist handoff summaries use the same `summary_agent` with a separate `handoff_summary_tokens` budget (default: 500 tokens).

## Definition of Done for v5

- A user can chat through a unified interface with token-streaming responses.
- A routing chatbot classifies turns and delegates to specialists.
- At least two specialist chatbots are fully provisioned and working.
- The Qdrant knowledge base is integrated as a registered tool with per-specialist filtering.
- Specialists can delegate long-running work to backend agents and continue conversation.
- Background task completions are proactively surfaced in conversation.
- Conversations are fully persisted and resumable.

---

## What to Read Next

With v5, the full user-facing product is complete. The final phase adds production readiness:

→ **[07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md)** — Immutable versions, debug mode, observability, and vault migration.

Or explore supporting topics:
- [13-Chatbot UX Edge Cases](./13-chatbot-ux-edge-cases.md) — Edge cases for conversation handling
- [09-Security Model](./09-security-model.md) — Authentication and PII handling
- [10-Deployment Concepts](./10-deployment-concepts.md) — Streaming infrastructure topology
