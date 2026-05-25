# 06 — Learnings from this trial

What this UI-vision exercise taught us — about the platform, the design, and the process. Captured so the next person (or the next session) starts ahead of where we did.

## About the platform

**1. The UI surface is far larger than the roadmap implies.**
The PRD defers UI explicitly (`AD-25`, "No UI in early phases") and the headline roadmap names only a "Visual Workflow Builder" for v7. But reading the *entire* doc set surfaced **six distinct surface areas and 50+ user-facing capabilities** — most of them buried in the v7+ deep-dives (evaluation `27`, scheduling `29`, KB management `33`, developer playground `34`, marketplace `25/26`, compliance/audit `35`). The "UI is deferred" framing hides how much UI the platform actually demands. A real product would need a deliberate IA across all six areas, not one dashboard.

**2. There are more than four personas in practice.**
The PRD names four (`PD-03`: Platform Admin, Agent Builder, End User, Operator). But the analytics and compliance docs implicitly serve a **buyer/stakeholder** (cost, ROI, deflection) and a **compliance officer** (audit, consent, residency). The UI has to speak to people the architecture docs never named.

**3. State is the spine of this product.**
Two lifecycles dominate everything an operator sees: the **version lifecycle** (draft → staging → canary → production, `AD-16`/`AD-29`) and the **run lifecycle** (queued → running → paused → succeeded/failed/cancelled, `AD-17`). The single most valuable design decision was reserving a disciplined semantic color palette *exclusively* for state. Get that right and the whole product reads at a glance.

**4. The "machine layer" deserves typographic status.**
Agent IDs, DSL, trace IDs, durations, token counts, costs — these are not metadata, they are the operator's primary working material. Treating monospace as a first-class register (not a code-block afterthought) is what separates a credible ops tool from a generic SaaS skin.

## About the design

**5. "Refined" is not "safe."**
The brief was "Refined SaaS (light)," which is exactly where generic AI aesthetics live (Inter, cobalt-on-white, soft purple gradients). The `frontend-design` skill forced the harder question: what is the *point of view*? The answer — "Paper & Signal" (warm paper + a single Tide-teal signal + a monospace machine register) — is refined **and** distinctive. Teal was a deliberate dodge of the SaaS-blue cliché, chosen partly because it doesn't collide with the semantic state ramp.

**6. One demo story beats twenty disconnected mockups.**
Anchoring every screen to a single coherent fixture — the PRD's customer-support deployment (`PD-02`) — is what will make a static design vision feel like a real product instead of a component gallery.

## About the process

**7. Breadth-checking needs a fan-out read, not a skim.**
The first screen list (10 screens) was drawn from the PRD + roadmap and missed ~15 surfaces. A dedicated survey agent reading **all 30+ docs** caught them. Lesson: when "did we cover everything?" matters, dispatch an exhaustive read rather than trusting a from-memory pass. The gap analysis is preserved in [03-surface-map.md](./03-surface-map.md).

**8. Tier the build; document the whole.**
Trying to build all 20 screens at equal depth would dilute the showcase. The resolution: **document the complete surface map** (nothing forgotten) but **build in tiers** — Core 12 first, Extended 8 to complete breadth. The map is the contract; the tiers are the schedule.

**9. Parallelize the fan-out, serialize the foundation.**
The design-system foundation (tokens + components) and the mock fixtures must exist before screens can be built — that part is serial. Once they exist, independent screens (Figma assembly and shadcn implementation) are embarrassingly parallel and suit multiple agents. See the parallelization strategy in [05-build-workflow.md](./05-build-workflow.md).

**10. Decide direction before opening Figma.**
"Design in Figma first" only pays off if the aesthetic POV, the screen inventory, and the demo story are settled *before* the first frame. This doc set is that settling. Figma work starts from a decided position, not a blank exploration.

## Learnings from the live Figma build

**11. shadcn/ui isn't importable as a Figma library — Figma's Simple Design System (SDS) is the closest analog.**
`get_libraries` on a fresh file exposes only Figma/Apple/Google community kits; there's no way to pull an arbitrary community shadcn file via the MCP. SDS (Figma's own *code-backed*, Radix-adjacent kit) is the realistic substitute. **Decision:** rebuild the components that looked hand-rolled to shadcn/SDS *specs* (proportions, structure) while keeping Paper & Signal theming — rather than importing SDS's foreign theme (gray/blue/Inter) and fighting it. Spec-faithful, theme-cohesive.

**12. "Workflows" is not a separate destination — a workflow is an agent *type*.**
The first IA pass added a "Workflows" nav item and a standalone "Workflow Builder", which conflicts with the PRD model (`AD-19`): agents are the artifacts; workflow vs. chatbot is a *type*. The visual builder is therefore the **editor you open for a workflow-type agent**, reached from the catalogue — not a peer of "Agents". Tells that confirmed it: "Refund Workflow" is already a row in the Agent Catalogue, and the catalogue already filters `All / Chatbots / Workflows`. **Fix:** builder breadcrumb is `Build / Agents / Refund Workflow`, active nav is Agents, and the redundant "Workflows" nav item was removed. Lesson: derive IA from the domain model, not from screen names.

**13. Figma `createFrame` + `layoutMode` does NOT auto-hug — it stays 100×100.**
Setting `layoutMode` on a plain frame leaves its sizing modes fixed at the default 100×100, producing giant circles/boxes (a `cornerRadius:999` tag became a 100px circle). Use `figma.createAutoLayout()` (hugs both axes by default), or explicitly set `primaryAxisSizingMode='AUTO'` + `counterAxisSizingMode='AUTO'`. Also: `layoutSizing='FILL'/'HUG'` only applies to children of auto-layout parents — setting it on a child of a plain frame silently no-ops. And never call `resize()` *after* setting `FILL` (resize resets it to fixed).

**14. `setSharedPluginData` isn't available on `VariableCollection`/`Variable`/`Style` objects in this sandbox.**
The design-system skill's idempotency tagging assumes it is. Fall back to **name-based idempotency** (look up by name within collection) and track IDs via return values — which works fine.

## Open questions to resolve before/with the build

- **Exact hexes & font weights** for the Paper & Signal tokens — finalized as Figma variables in Phase A.
- **Dark mode** — structured for (a second Figma mode) but out of scope for this static light vision unless promoted.
- **Responsive scope** — desktop-first for builder/operator surfaces; Chat is the one surface shown at mobile width. Other surfaces' mobile treatment is deferred.
- **Tier-2 depth** — whether the 8 Extended screens are built fully or left as Figma-only frames depends on time/appetite after Core ships.
- **Name** — "Conductor" is a placeholder; confirm or replace before it hardens into assets.
