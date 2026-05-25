# 04 — Screen Inventory & Build Tiers

The [surface map](./03-surface-map.md) catalogues *everything the platform could expose*. This document is the **curated build set** — the screens we actually design in Figma — organized into two tiers so the showcase can ship Core-first and extend into full breadth.

Every screen lists the **capabilities and decisions** it demonstrates, so the showcase is traceable back to the platform docs rather than invented.

## How the tiers work

- **Tier 1 — Core (12 screens):** the spine of "what's possible." Covers the full primary value flow (define → validate → prove → promote → run → observe → talk) plus the highest-impact surfaces the first 10-screen pass missed (playground, evaluation). Build these first.
- **Tier 2 — Extended (8 screens):** the enterprise-grade breadth (trace/replay, health, scheduling, KB lifecycle, marketplace, compliance, model routing, memory). Build to complete the full "what's possible" picture.

A "screen" may include 2–3 designed **states** (e.g. the builder's empty / valid / error states) where the state is the point.

---

## Tier 1 — Core (12)

### Build — Agent Builder Studio

**1. Agent Catalogue** *(home)*
Grid/list of agents: name, provider chip, type (workflow / chatbot), version-state badge, last run, owner. Filters by state, provider, type, namespace.
→ Demonstrates: catalogue, multi-tenancy (`AD-26`), versioning surface (`AD-16`).

**2. Visual Workflow Builder**
Node-graph canvas (step types: `llm_call`, `tool_call`, `mcp_call`, `transform`, `branch`, `switch`, `for_each`, `parallel`, `agent_call`) + **DSL JSON split-view** + node inspector panel (model config, retry policy, I/O mapping).
→ Demonstrates: `AD-02`, `AD-03`, `AD-14`, `AD-15`, v2/v4, `14-dsl-spec`, v7 visual builder (`26`).

**3. Validation (builder state)**
The "collect-all-errors" panel: every error at once, inline node markers, jump-to-error.
→ Demonstrates: `AD-13`, `PD-06`, `AD-12`.

**4. Interactive Playground**
Test an agent before shipping: input panel, live execution view, per-step output, step-through. Side-by-side version compare.
→ Demonstrates: developer experience playground (`34`).

**5. Evaluation Dashboard**
Test-suite runner, golden responses, result heatmap (pass/fail/score per assertion), LLM-as-judge rubric, A/B comparison across versions.
→ Demonstrates: agent evaluation framework (`27`).

**6. Version Promotion**
Draft → staging → canary → production lane, canary traffic-split control, promotion gates, rollback trigger, version history.
→ Demonstrates: `AD-29`, `AD-16`, `PD-07`.

### Operate — Operator Console

**7. Runs Dashboard**
`workflow_runs` table: status, agent + pinned version, duration, token/cost, namespace, started-by. Filters, search, status timeline.
→ Demonstrates: `AD-04`, state machine, run lifecycle.

**8. Run Detail / Step Timeline + Debug**
Per-run step timeline: each step's status, duration, retries, expandable input/output. Pause / resume / cancel controls and step-by-step debug ("approve next step", input override).
→ Demonstrates: `AD-17`, `AD-04`, `AD-14`, per-run cost cap (`AD-32`).

### Chat — End User

**9. Unified Chat**
Streaming conversation with a **routing badge** (which specialist handled each turn + confidence), a **background-task card** (pending → proactively surfaced result), KB **citation chips**, and a graceful-**degradation banner**.
→ Demonstrates: `AD-19`, `AD-20`, `AD-21`, `AD-31`, `AD-33`, conversation memory (`21`).

### Catalog — Platform Admin

**10. Tool & MCP Catalogue**
Code-registered tools + config-registered MCPs: capability descriptions, versions, auth status, "add MCP" + OAuth flow, knowledge-base tool entry.
→ Demonstrates: `AD-11`, `AD-18`, v3, shared catalogue (`25`).

**11. Admin & Namespaces**
Namespace management, quotas, **price table** (cost estimation), provider rate-limit config, credentials/vault status.
→ Demonstrates: `AD-26`, `AD-07`, `AD-28`, `16`, `19`, `23`, `40`.

### Analytics — BI

**12. Analytics Dashboard**
Conversation volume, routing distribution across specialists, agent performance, cost trend vs. budget caps, deflection/ROI, KB hit rate.
→ Demonstrates: business intelligence (`30`), cost (`31`, `40`).

---

## Tier 2 — Extended (8)

**13. Trace / Execution Visualizer** — OTel span waterfall + flow-graph replay with status colors and state snapshots. *(`AD-24`, `34`)*

**14. Health & Circuit-Breaker Monitor** — per-specialist and KB circuit state, error rates, fallback config. *(`AD-31`, `22`)*

**15. Scheduling & Triggers** — cron wizard, webhook endpoints, event subscriptions, delivery history. *(`29`)*

**16. Knowledge Base Management** — source connectors, ingestion pipeline status, collections, freshness, retrieval test bench. *(`AD-18`, `33`)*

**17. Marketplace / Shared Catalogue** — public agent browsing, ratings, import-to-namespace, `marketplace://` URIs. *(`25`, `26`)*

**18. Compliance & Audit Console** — audit log search, PII detection trail, consent registry, data-residency map, right-to-erasure queue. *(`28`, `35`)*

**19. Model Routing & Cost Optimization** — complexity classifier config, tier selection, cascading fallback, response-cache analytics. *(`31`)*

**20. Memory Inspector** — episodic summary browser, semantic profile viewer, memory-injection traces. *(`21`, `AD-30`)*

---

## Mock-data fixtures (shared across screens)

One coherent demo namespace tells a single story so the showcase feels real: a **customer-support deployment** (the PRD's primary scenario, `PD-02`) — a routing chatbot, specialists (payments / technical / knowledge), a few background workflow agents, a Qdrant KB, and a handful of tools/MCPs (Linear, Jira, an internal hotel-availability API). The demo data (placed directly in the Figma canvas) provides: ~8 agents across version states, ~20 workflow runs across statuses, one rich chat transcript exercising routing + background surfacing + citations + a degradation moment, eval suites with results, and analytics time-series.

→ Build sequence and tech: [05-build-workflow.md](./05-build-workflow.md)
