# 09 — Screens ↔ Flows Consistency Contract

> As the Figma file grows to **one page per flow**, the flow pages must stay consistent with the `Screens` page and with each other. This document is the contract every flow page (the six creation journeys, the Lists pattern, and any future flow) must satisfy — plus the verification checklist used to enforce it. Snapshot: 2026-05-25.

## Page structure (target)

| Figma page | Holds |
|---|---|
| `Cover` (0:1) | Cover |
| `Foundations` (15:2) | Variables, text/effect styles — the **shared** design system |
| `Components` (15:3) | Shared component sets (badges, chips, shells, steppers, List constructs) |
| `Screens` (23:2) | The 20 product-tour screens — app shell (sidebar + topbar + body) |
| **One page per flow** | `Agent creation` · `MCP creation` · `Namespace creation` · `KB source` · `Trigger creation` · `Credentials` · `Lists pattern` |

*(The single `Journeys` page `82:2` is retired once its bands move to per-flow pages.)*

## The contract — every flow page MUST

1. **Tokens.** All color/spacing/radius come from Paper & Signal **variables** (no bare hex): `surface/*`, `text/*`, `border/hairline`, `brand/tide*`, `state/*`, `provider/*`.
2. **Typography.** Bricolage Grotesque (display/headings), Hanken Grotesk (body/UI), JetBrains Mono (machine layer: IDs, URLs, cron, keys, counts). Use the shared text styles (Display/L, Heading/H1·H2, Body/L·M·S, Strong/M, Label/M·S, Mono/M·S).
3. **Branding.** Takeover flows: top bar reads **`Conductor · {flow name}`** + `Exit ✕`. The Screens page carries the full app shell (Conductor brand + namespace switcher + nav).
4. **Shared components (reuse as instances where practical).** Version State Badge `20:23`, Provider Chip `21:14`, Takeover shell `88:26`, Stepper `84:15`, Result chip `89:46`, Connector card `91:49`, Progress-step row `92:48`, and the List constructs (FilterBar / ViewToggle / DataTable+Pager / BlockGrid sentinel). Hand-built primitives are tolerated **only if visually identical** to the component (the global shadcn pass standardizes them).
5. **Frame conventions.** Full-screen frames are **1440×1024**, laid left→right at `x = i×1540` within a page/band. Each page/band carries a **Display/L** label. Frame names follow `{Flow} · {n} — {step}`.
6. **State semantics.** The semantic palette (draft·staging·canary·production·running·error·paused) is used **only** to encode state, never decoration — identical to the Screens page.
7. **Entry-point linkage.** Every flow launches from a primary `brand/tide` button in the **content header** (NOT the global topbar) of its parent screen on the Screens page:
   - Agent → Agent Catalogue `23:3` (`New agent`)
   - MCP → Tool & MCP Catalogue `56:56` (`Add MCP`)
   - Namespace → Admin `57:56` (`New namespace`, content header)
   - Credentials → Admin `57:56` Credentials card (`Add credential`)
   - KB source → KB Management `68:56` (`Add source`)
   - Trigger → Scheduling `67:56` (`New trigger`)
8. **One story.** All flows use the same `acme` customer-support mock story and consistent identifiers, referencing artifacts that exist on the Screens page (specialists `payments`/`technical`/`knowledge`, the `acme` Qdrant KB, the Linear MCP, namespaces, etc.).

## Verification checklist (run per flow page)

- [ ] Tokens bound (spot-check fills/text via `get_variable_defs`); no bare hex.
- [ ] Fonts correct (Bricolage / Hanken / JetBrains Mono in the right registers).
- [ ] Takeover top bar = `Conductor · {flow}`; `Exit ✕` present.
- [ ] Stepper style matches: dots/bars + `Step N of M`, **no** per-step text labels.
- [ ] Shared components used as instances where practical.
- [ ] Frames 1440×1024; `x = i×1540`; Display/L page/band label; frame names conform.
- [ ] State colors used only for state.
- [ ] Entry-point button present on the parent screen's **content header** (brand/tide, not topbar).
- [ ] `acme` story identifiers consistent; referenced artifacts exist on the Screens page.
- [ ] No 100×100 auto-layout artifacts; no orphaned/old frames left behind.

## Known consistency debts (tracked)

- **Component instancing:** several flow primitives (some chips/cards/steppers) are hand-built — visually identical, but not live instances. The **global shadcn swap** will standardize them. Not worth paying down before that pass.

> This contract is enforced by the **consistency verification pass** (see `08-build-status.md` backlog) and should be re-checked whenever a new flow page is added.
