# Agent Platform — UI Vision ("Conductor")

> **Status:** Design exploration / trial. No production UI is committed by the platform roadmap — AD-25 explicitly defers the UI to "Post-v6". This folder captures **what a first-class UI for the platform could be**: the full surface map, a design system with a real point of view, a tiered screen inventory, and the build workflow. It is a vision artifact, not a roadmap commitment.

---

## Why this folder exists

The PRD and roadmap describe a deep, multi-persona platform but intentionally ship **no UI** through v6 (API-first; a minimal Operator CLI lands in v6). The phrase that started this trial: *"create a first-class UI experience of what is possible from this project."*

Answering that honestly required reading the **entire** doc set — not just the PRD and roadmap, but all 30+ deep-dives — because the v7+ documents (evaluation, scheduling, KB management, developer playground, marketplace, compliance) each imply whole UI surfaces that the headline roadmap never names. The [surface map](./03-surface-map.md) is the result.

## What's here

| Doc | Purpose |
|---|---|
| [01-ui-vision.md](./01-ui-vision.md) | Product framing, personas → jobs, the showcase concept, fidelity decisions, working name. |
| [02-design-system.md](./02-design-system.md) | The "Paper & Signal" design language — typography, color, motion, components. Written under the `frontend-design` skill (distinctive, not generic). |
| [03-surface-map.md](./03-surface-map.md) | The **complete** capability → UI surface map across six areas. The "nothing forgotten" reference. |
| [04-screen-inventory.md](./04-screen-inventory.md) | The curated, tiered build set (Core 12 + Extended 8), each screen mapped to the capabilities and decisions it demonstrates. |
| [05-build-workflow.md](./05-build-workflow.md) | The Figma-only build workflow: design system → screens → shadcn-in-Figma, and "done" criteria. No code. |
| [06-learnings.md](./06-learnings.md) | What this trial taught us — about the platform, the process, and the gaps. |
| [07-shadcn-in-figma.md](./07-shadcn-in-figma.md) | How to bring real shadcn/ui components into Figma (no code) and theme them to Paper & Signal. |
| **[08-build-status.md](./08-build-status.md)** | **Start here to resume.** What actually exists now (Figma file/page/node IDs, design system, all 20 screens + 6 journey pages + Lists pattern), shadcn status, open items, and cold-start instructions. |
| [09-consistency-contract.md](./09-consistency-contract.md) | The Screens↔flows consistency contract every per-flow page must satisfy + the verification checklist. |
| [journeys/](./journeys/) | Per-journey docs covering the **complete** decision tree (all branches/options) + the real actions required to make each path work. |
| [patterns/lists.md](./patterns/lists.md) | The reusable Lists pattern: filter bar + Blocks⇄Datatable toggle + dual pagination + the shared pagination contract. |

## Decisions locked for this trial

| Decision | Choice | Rationale |
|---|---|---|
| **Scope** | Full vision showcase — all surfaces in one navigable shell | "What's possible" leans to breadth; one shell makes it read as a product, not loose mockups. |
| **Fidelity** | Static design vision — polished screens, mock data, minimal interaction | Fastest path to a credible, complete picture; no backend exists to wire to. |
| **Figma role** | Design in Figma only — **no code in this track** | The Figma file is the deliverable; a code implementation is explicitly out of scope here. |
| **Aesthetic** | Refined SaaS (light), executed with a distinctive POV | See [02-design-system.md](./02-design-system.md). "Refined" only counts if it has character. |
| **Components** | Real shadcn/ui brought in as a Figma kit, themed to Paper & Signal | See [07-shadcn-in-figma.md](./07-shadcn-in-figma.md). No local code. |

## Relationship to the rest of `docs/`

This folder **consumes** the platform docs; it does not change them. Every screen is traced back to a decision (AD-XX / E-XX) or deep-dive in [03-surface-map.md](./03-surface-map.md). If a capability changes upstream, the surface map is where to reconcile it.
