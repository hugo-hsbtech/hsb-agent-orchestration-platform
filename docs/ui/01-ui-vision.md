# 01 — UI Vision

## The one-sentence vision

**Conductor** is the operator-facing surface of the Agent Platform: a single, calm, data-dense workspace where an Agent Builder defines and proves agents, an Operator watches and steers them in production, a Platform Admin governs the tenancy, and an End User talks to the result — all sharing one design language.

("Conductor" is a working name — orchestration → conductor. Trivial to rename; it lives in one design token and one nav header.)

## The product thesis

The platform's core principle is **separation of concerns**: a lean Agent Core wrapped in composable infrastructure. The UI must express the same idea. It is not one monolithic dashboard; it is **six coherent surfaces** sharing a shell, a token set, and a component library. You can stand in any one of them and understand it without reading the others — the same property the architecture demands of the code.

## Personas → jobs the UI must serve

| Persona | The job they hire the UI to do | Primary surfaces |
|---|---|---|
| **Agent Builder** | "Define an agent, prove it works, and ship it safely." | Studio (builder, validation, playground, evaluation, promotion) |
| **Operator** | "Know what's running, why it broke, and stop the bleeding." | Console (runs, step debug, traces, health, cost) |
| **Platform Admin** | "Govern tools, tenants, budgets, and compliance." | Admin / Catalog (registry, namespaces, scheduling, KB, audit) |
| **End User** | "Get my answer; don't make me wait or repeat myself." | Chat (streaming, routing, proactive task surfacing) |

These four personas map to the platform's `PD-03`. A fifth implicit audience — **the buyer / stakeholder** — is served by Analytics/BI (cost, ROI, deflection, quality).

## The showcase concept

One navigable application shell:

- **Left sidebar** — namespace switcher at top (multi-tenancy is first-class, `AD-26`), then nav grouped by *job*: **Build**, **Operate**, **Chat**, **Catalog**, **Analytics**, **Admin**.
- **Top bar** — environment/namespace pill, global command palette (⌘K), docs, user.
- **Body** — the active surface.

The shell is the spine that turns ~20 screens into one believable product. It is itself a reusable component (see [02-design-system.md](./02-design-system.md)).

## Fidelity: static design vision

This trial produces a **static design vision**: every screen is polished and populated with realistic mock data, but it is not wired to a backend (there is none yet). Interaction is limited to what sells the vision without faking logic:

- Navigation works; the command palette opens; tabs, hovers, and disclosure work.
- "Live" behaviors (streaming, validation, running workflows) are shown as **designed states** with mock fixtures, not real engines.
- Where a single behavior is the whole point of a surface (the chat's proactive background-task surfacing; the workflow's running state), we show it as **two or three frames / states** rather than animating a fake engine.

## What "first-class" means here

Three non-negotiables, drawn from the platform's own values:

1. **The machine layer is legible.** Agent IDs, DSL, trace IDs, durations, token counts, and costs are everywhere. They get a monospace treatment and never get truncated into uselessness. Operators live in this detail.
2. **State is honest.** The version lifecycle (draft → staging → canary → production) and run lifecycle (queued → running → paused → succeeded/failed/cancelled) are encoded in a disciplined semantic color set used *only* for state, never decoration. A glance tells you what's safe to touch.
3. **It is calm under density.** This is an operations tool. It favors generous structure, hairline rules, and restraint over dashboard maximalism. Elegance comes from execution, not ornament.

→ Next: [02-design-system.md](./02-design-system.md)
