# 03 — Complete Surface Map

> The exhaustive reference: every user- or operator-facing capability the platform docs describe, mapped to the UI surface it implies, the persona who uses it, and its source decision (AD-XX / E-XX) or deep-dive doc. This is the "nothing forgotten" inventory. The curated build set is in [04-screen-inventory.md](./04-screen-inventory.md).

## Capability → surface map

### 1. Agent Builder Studio

| Capability | Persona | UI surface | Source |
|------------|---------|------------|--------|
| Create/edit agent DSL (visual + JSON split) | Agent Builder | Visual Workflow Builder | 26, 34 |
| Multi-step workflows with retries | Agent Builder | Workflow Canvas | v2, 14 |
| Model/provider/temperature config | Agent Builder | Model Config Panel | v1, 31 |
| Tools & MCPs for an agent | Agent Builder | Tool/MCP Selector | v3, 25 |
| Webhook/cron/event triggers | Agent Builder | Trigger Configuration | 29 |
| Multi-agent calls (sync/async, version pin) | Agent Builder | Agent Call Designer | v4, 25 |
| Chatbot routing rules & specialists | Agent Builder | Routing Configuration | v5, 13, AD-33 |
| Memory tiers (episodic, semantic) | Agent Builder | Memory Configuration | 21, AD-30 |
| Safety policies (moderation, PII) | Agent Builder | Safety Policy Editor | 28 |
| Validate DSL before save | Agent Builder | Real-time Validation Panel | AD-13, 14, 34 |
| Test agent interactively | Agent Builder | Interactive Playground | 34 |
| Compare agent versions | Agent Builder | Version Diff Viewer | 34 |
| Define eval test cases & golden responses | Agent Builder | Test Case Editor | 27 |

### 2. Operator Console

| Capability | Persona | UI surface | Source |
|------------|---------|------------|--------|
| List/inspect workflow runs | Operator | Runs Dashboard | AD-04, v6, 17 |
| Pause/resume/cancel | Operator | Run Control Panel | AD-17, 17 |
| Step-through debug | Operator | Debug Stepper | AD-17, 34 |
| Step-level data flow (I/O) | Operator | Data Inspector | v6, 34 |
| Stream logs | Operator | Log Viewer | v6, 17 |
| OpenTelemetry traces | Operator | Trace Browser | AD-24 |
| Version status & deprecation | Operator | Version Registry | AD-16, 17 |
| Run health & errors | Operator | Health Monitor | v6, AD-31 |
| Cost/token budgets | Operator | Budget Config | AD-32, 23, 31 |

### 3. End-User Chat

| Capability | Persona | UI surface | Source |
|------------|---------|------------|--------|
| Stream chatbot responses | End User | Chat UI | AD-19, v5 |
| Routing to specialists (badge, confidence) | End User | Routing Indicator | AD-20, AD-33 |
| Proactive background-task surfacing | End User | Background-Task Card | AD-21 |
| Conversation history | End User | Chat History | v5, 21 |
| Graceful degradation messaging | End User | Degradation Banner | AD-31, 22 |
| KB citations & feedback | End User | Citation/Feedback Widget | 30, 33 |
| Satisfaction rating | End User | Rating Widget | 30 |
| Consent preferences | End User | Consent Manager | 35 |
| Data deletion request | End User | Erasure Request | 35 |

### 4. Platform Admin / Catalog

| Capability | Persona | UI surface | Source |
|------------|---------|------------|--------|
| Create/manage namespaces | Platform Admin | Namespace Manager | AD-26, 16 |
| Global tools & MCPs registry | Platform Admin | Tool/MCP Registry | AD-11, v3 |
| Provider rate limiting | Platform Admin | Provider Config | AD-28, 19 |
| Publish to shared catalogue | Platform Admin | Catalogue Publisher | 25 |
| Browse/search shared catalogue | Agent Builder | Catalogue Browser | 25 |
| Promotion gate config | Platform Admin | Promotion Gate Config | AD-29, 20 |
| Namespace safety defaults | Platform Admin | Safety Defaults | 28 |
| Cross-namespace allowlist | Platform Admin | Cross-Namespace Policy | 16 |
| Data residency regions | Platform Admin | Data Residency Config | 35, 36 |
| Platform quotas & usage | Platform Admin | Quota Dashboard | 16 |
| Price table / cost estimation | Platform Admin | Price Table | 23, 40 |
| Credentials / vault status | Platform Admin | Credentials Manager | AD-07, AD-08 |
| Bulk profile import (CRM) | Platform Admin | Profile Importer | 21 |

### 5. Analytics / BI

| Capability | Persona | UI surface | Source |
|------------|---------|------------|--------|
| Conversation volume & trends | Operator | Conversation Analytics | 30 |
| Routing & specialist performance | Agent Builder | Routing Dashboard | 30 |
| Agent quality / eval scores | Agent Builder | Quality Scorecard | 27, 30 |
| Cost per conversation & ROI | Stakeholder | Cost & ROI Dashboard | 30, 31 |
| Token consumption by provider/model | Stakeholder | Token Analytics | 30, 31 |
| Workflow execution bottlenecks | Operator | Execution Dashboard | 30 |
| Satisfaction & sentiment trends | Stakeholder | UX Dashboard | 30 |
| KB coverage & gaps | Knowledge Manager | KB Analytics | 30, 33 |
| PII detection & handling audit | Compliance | Compliance Dashboard | 28, 35 |
| Data retention compliance | Compliance | Retention Status | 35 |
| Custom metrics & alerts | Analyst | Custom Metric Builder | 30 |
| Pre-built dashboard templates | All | Dashboard Library | 30 |

### 6. Standalone surfaces (beyond the five)

| Capability | Persona | UI surface | Source |
|------------|---------|------------|--------|
| Evaluation Framework (suite runner, scoring, comparison) | Agent Builder | Evaluation Dashboard | 27 |
| Scheduling & Triggers (cron, webhook, events) | Agent Builder | Trigger Manager | 29 |
| Knowledge Base ingestion/lifecycle | Knowledge Manager | KB Pipeline Monitor | 33 |
| Model routing & response cache | Agent Builder | Model Router | 31 |
| Multi-agent collaboration (swarms, consensus) | Agent Builder | Collaboration Workspace | 32 |
| Conversation memory inspector | Operator | Memory Dashboard | 21, AD-30 |
| Marketplace / public catalogue | Agent Builder | Marketplace | 25, 26 |
| Visual debugger / execution replay | Operator | Execution Visualizer | 34 |
| Compliance & audit console | Compliance | Audit & Governance Hub | 35 |
| Provider SDK compatibility matrix | Platform Admin | SDK Matrix | 39 |
| Disaster recovery & backups | Platform Admin | DR Dashboard | 36 |
| IDE extension & CLI wizard | Agent Builder | VS Code Extension | 34 |

## Coverage verdict

The platform docs imply **six surface areas** and **50+ distinct capabilities** spanning seven personas (Agent Builder, Operator, End User, Platform Admin, Stakeholder, Knowledge Manager, Compliance). Taken together they describe a far larger product than any single demo can hold.

A naive "core flow" showcase — catalogue, builder, validation, promotion, runs, debug, chat, tool catalogue, admin, analytics — covers the **primary value flow** well. It tells the end-to-end story of building an agent, validating it, promoting it, running it, debugging it, and serving it to end users, with admin and analytics framing the lifecycle. For demonstrating the platform's central promise, that set is sufficient.

But the core flow **misses three enterprise pillars** that the docs treat as first-class:

1. **Quality & Reliability** — systematic evaluation suites, quality gates, golden-response comparison, and A/B / version comparison. Without these the platform looks like it ships agents on faith.
2. **Compliance & Risk** — audit trails, consent management, data residency, PII detection/handling, retention, and disaster recovery. These are the prerequisites for regulated and enterprise deployment.
3. **Sophisticated Automation** — scheduling and event triggers, KB ingestion/lifecycle, multi-agent collaboration (swarms, consensus), and conversation memory. These are what make agents proactive, collaborative, and self-maintaining rather than purely request/response.

Concretely, a core-flow-only build leaves these screens on the floor:

- Evaluation Dashboard, Quality Scorecard, Test Case Editor, Version Diff Viewer (Quality & Reliability)
- Audit & Governance Hub, Compliance Dashboard, Consent Manager, Erasure Request, Data Residency Config, Retention Status, DR Dashboard (Compliance & Risk)
- Trigger Manager, KB Pipeline Monitor, Collaboration Workspace, Memory Dashboard, Model Router (Sophisticated Automation)

For this reason the curated build set in [04-screen-inventory.md](./04-screen-inventory.md) promotes the **Interactive Playground** and **Evaluation** surfaces into **Tier-1 Core** — closing the most visible Quality gap inside the primary flow — and gathers the remaining enterprise surfaces into **Tier-2 Extended**, so the full inventory above stays accounted for without bloating the headline demo.
