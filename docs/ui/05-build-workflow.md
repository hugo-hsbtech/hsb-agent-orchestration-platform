# 05 — Build Workflow (Figma only)

> **This track is Figma-only — no code.** The deliverable is the Figma file: the **Paper & Signal** design system plus the screen showcase. A code implementation (Next.js / shadcn app / Storybook) is **explicitly out of scope** for this thread. shadcn/ui still features here — but as *real shadcn components brought into Figma*, not as code (see [07-shadcn-in-figma.md](./07-shadcn-in-figma.md)).

The build runs **foundation-first, then fan out** across independent screens.

## Phase A — Design system in Figma

Foundation, built once, used everywhere. "Paper & Signal" ([02-design-system.md](./02-design-system.md)) made real as Figma **variables** and **component sets**. Uses the `figma-generate-library` skill.

1. **Variables / tokens** — color (warm paper neutrals + Tide brand + the 7-state semantic set + provider colors), type scale, spacing, radius, effects. Built in a **light mode**, structured so a dark mode drops in later as a second mode.
2. **Component library** — shadcn primitives + the platform's custom composites, as Figma component sets with variants, every visual property bound to a variable.

Phase A is mostly serial (components depend on tokens; composites depend on primitives).

## Phase B — Assemble screens in Figma

With the library in place, assemble the screens ([04-screen-inventory.md](./04-screen-inventory.md)) as Figma frames built section-by-section from the components, always referencing design tokens. Uses the `figma-generate-design` skill.

1. **Tier-1 Core (12), first** — the spine of "what's possible".
2. **Tier-2 Extended (8), second** — the enterprise breadth.

All screens populate from the **one shared customer-support demo namespace** so the showcase reads as a single believable product. Independent screens assemble in **parallel** once the Phase A library exists.

## Phase C — Real shadcn/ui components, in Figma (enhancement)

Once a shadcn/ui Figma kit is published to the team, bring its components in and theme them to Paper & Signal — full procedure in [07-shadcn-in-figma.md](./07-shadcn-in-figma.md). Until then, the primitives that needed shadcn fidelity are built **to shadcn spec in-file** (already done: Version-State Badge, Provider Chip, the Tabs/segmented control). **No code at any point.**

## Parallelization

| Stage | Mode | What runs |
|---|---|---|
| **1. Foundation** | Serial | Phase A tokens → primitives → composites |
| **2. Fan-out** | Parallel | Phase B screen frames (independent screens by parallel agents) |
| **3. Integration** | Serial | Shell consistency pass across screens |

Note: **Figma writes (`use_figma`) are strictly sequential per file** — parallelism is across agents/screens at the planning level, not concurrent mutations.

## Tools

| Layer | Choice | Notes |
|---|---|---|
| **Design tool** | Figma (variables + component sets) | The single deliverable |
| **Components** | shadcn/ui brought in as a Figma kit, themed to Paper & Signal | See [07](./07-shadcn-in-figma.md); spec-built in-file until a kit is published |
| **Fonts** | Bricolage Grotesque / Hanken Grotesk / JetBrains Mono | Google Fonts, available in Figma |
| **Demo data** | One coherent customer-support namespace story | Mock content placed directly in the canvas |

## Definition of done

- [ ] **Figma file** with the full Paper & Signal design system (variables/tokens + component sets) plus the Foundations and Components pages.
- [ ] **All Tier-1 Core screens** assembled from the system; Tier-2 Extended is a stretch goal.
- [ ] **Every screen traceable** to a capability/decision in [03-surface-map.md](./03-surface-map.md).
- [ ] **Desktop-first**, with **Chat also shown at mobile width**.
- [ ] *(If a shadcn kit is published)* primitives swapped for the real shadcn components, themed to Paper & Signal.

**Out of scope (this track):** any local code — no Next.js, no shadcn CLI/app, no Storybook, no lint/typecheck/build. The Figma file is the artifact.
