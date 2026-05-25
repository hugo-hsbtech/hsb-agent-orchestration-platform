# Agent Platform — Roadmap & Implementation Plan

## What We're Building

An **Agent Provisioning and Orchestration Platform**: declarative AI agents defined in a DSL, executed durably via Temporal, composed into multi-agent orchestrations, and exposed to users through a streaming chatbot layer.

The defining property is **separation of concerns** — the Agent Core stays minimal (receives a prepared bundle, runs inference), while all infrastructure complexity lives around it as composable layers. Operators can provision a complete customer-support chatbot ecosystem (routing → specialist chatbots → background workflow agents → tools/MCPs → vector KB) without writing orchestration code.

> Full context, architectural decisions, and technology stack: **[01-PRD.md](./01-PRD.md)**

---

## Delivery Philosophy

Each version is a **vertical, shippable slice** of the platform. No phase is a "setup" phase — each one delivers real, usable value. Each phase:

- Produces a working system that can be validated with real users.
- Builds directly on the previous phase without invalidating prior work.
- Keeps the scope tight enough to ship without accumulating debt.

---

## Roadmap

| Phase | Theme | Quick-Win Delivered |
|---|---|---|
| **v1** | Agent Provisioning Foundation | A single agent can be defined via DSL and executed against a live LLM. End-to-end, from API call to response. |
| **v2** | Workflow Orchestration | Multi-step workflows run durably on Temporal. State is persisted. Steps can be retried without re-running from scratch. |
| **v3** | Tools & MCPs | Agents can call registered tools and MCP connectors. Real-world actions (APIs, databases, Jira, Linear) become available. |
| **v4** | Multi-Agent Orchestration | Agents can invoke other agents as sub-workflows. Parallel execution, branching, and loops work. Complex compositions are now possible. |
| **v5** | Chatbot Layer | End users interact via a streaming chat interface. A routing chatbot dispatches to specialists. Background tasks fold results back into the conversation. |
| **v6** | Versioning & Hardening | Immutable agent versions, pause/resume/debug, distributed tracing, secrets vault. The platform is production-ready. |

---

## Phase Documents

| Doc | Phase | What It Covers |
|---|---|---|
| [01-PRD](./01-PRD.md) | Foundation | Requirements, decisions, technology stack, personas, out-of-scope |
| [02-v1](./02-v1-agent-provisioning-foundation.md) | **v1** | DSL v1 schema, provisioning API, provider-agnostic Agent Core, single-step execution |
| [03-v2](./03-v2-workflow-orchestration.md) | **v2** | Temporal integration, multi-step workflow interpreter, DB state machine, retry logic |
| [04-v3](./04-v3-tools-and-mcps.md) | **v3** | Tool registry, MCP registry, credential pipeline, tool-calling per provider |
| [05-v4](./05-v4-multi-agent-orchestration.md) | **v4** | Sub-agent invocation, parallel/sequential composition, branching, loops, fan-out/fan-in |
| [06-v5](./06-v5-chatbot-layer.md) | **v5** | Streaming endpoints, routing chatbot, specialist delegation, async task surfacing |
| [07-v6](./07-v6-versioning-and-hardening.md) | **v6** | Version immutability, canary promotions, debug mode, tracing, vault migration |

---

## Personas

| Persona | Role |
|---|---|
| **Platform Admin** | Registers tools, MCPs, and global settings. |
| **Agent Builder** | Defines agents via DSL: prompts, model, tools, workflow shape. |
| **End User** | Interacts with chatbots and agent-driven experiences. |
| **Operator** | Monitors workflows, debugs failures, manages versions. |

---

## Out of Scope (intentionally deferred)

- **Custom DSL parser/compiler** — JSON for now; DSL syntax tooling comes after real-world usage informs the shape.
- **LangChain / LangFuse** — LLM-level observability deferred post-v6.
- **Agent provisioning UI** — early phases are API-driven; UI is downstream. A minimal Operator CLI ships in v6.
- **Encrypted secrets vault** — plain-text DB through v5; vault migration is part of v6.
- **Event-driven orchestrator** — polling is sufficient for now; event bus is a later optimization.
- **Full ticketing handoff** — v5 ships `human_handoff` with `message`/`webhook` strategies; `ticket` (Jira, Zendesk) is deferred.

---

## Deep-Dive Reference

Supporting specs, conceptual docs, enhancement proposals, and future-roadmap material live in **[deep-dive/](./deep-dive/)**. Read them when implementing a specific component — they are not required to understand the phased plan above.

| Category | What's Inside |
|---|---|
| Conceptual | Performance/scaling, security model, deployment, testing strategy, upgrade paths, chatbot UX edge cases |
| Specifications | DSL full spec, developer tooling, multi-tenancy, operator CLI, enhancement proposals |
| Gap-closure | Provider rate limiting, agent version promotion, conversation memory, chatbot degradation, cost caps, DSL migration tooling, shared agent catalogue |
| Future (v7+) | Agent evaluation, content safety, BI analytics, workflow scheduling, intelligent model selection, multi-agent collaboration, developer experience, compliance, disaster recovery |
