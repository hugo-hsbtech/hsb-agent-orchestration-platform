# HSB Agent Orchestration Platform

A platform for provisioning and orchestrating specialized AI agents declaratively — no orchestration code required.

## What it does

You write a JSON configuration (DSL). The platform runs it: validates it, versions it, executes it durably through Temporal workflows, and exposes it to end users through a streaming chatbot layer with intelligent routing.

The target outcome: a website operator can provision a complete multi-agent customer-support system — routing chatbot, specialist chatbots, background workflow agents, knowledge base — entirely through configuration.

## Core idea

**Agents are configuration artifacts, not code.**

A single generic Temporal worker reads the DSL at runtime and interprets it. No per-agent deployments. No code generation. Change an agent's model, prompt, or tools by updating its configuration and promoting the new version.

```
DSL definition → Validation → Versioned storage → Temporal workflow execution → Streaming chatbot UX
```

## Key properties

- **Provider-agnostic** — Claude, OpenAI, Gemini, Grok, or others, selected per agent via config
- **DSL-driven** — all agent behavior declared in JSON (custom domain language planned later)
- **Durable** — every workflow run, every step, every input/output persisted in the database
- **Composable** — agents call other agents (sync or fire-and-forget), with parallel/branching/loop constructs
- **Immutably versioned** — every change creates a new version; in-flight workflows pin to their original
- **Multi-tenant** — namespace isolation from day one; every resource scoped to a namespace

## How it is built (phases)

| Version | Theme |
|---|---|
| v1 | Agent Provisioning Foundation — DSL, validation, single LLM call |
| v2 | Workflow Orchestration — multi-step, Temporal, state machine |
| v3 | Tools & MCPs — tool registry, MCP connectors, credentials |
| v4 | Multi-Agent Orchestration — sub-agents, parallel, branches, loops |
| v5 | Chatbot Layer — streaming, routing, specialist delegation, background tasks |
| v6 | Versioning & Hardening — promotion lifecycle, debug mode, observability, vault |
| v7+ | Evaluation, content safety, intelligent model routing, enterprise compliance |

## Who uses it

| Persona | What they do |
|---|---|
| **Platform Admin** | Registers tools, MCP connectors, global settings |
| **Agent Builder** | Writes DSL definitions and promotes versions |
| **Operator** | Monitors runs, debugs failures, manages traffic |
| **End User** | Talks to chatbots backed by the platform |

## Deeper reading

→ **[DEEPER.md](./DEEPER.md)** — architecture internals, data flows, DSL mechanics, diagrams

→ **[docs/](./docs/)** — full specification set (PRD, phase documents, DSL spec, enhancement proposals)
