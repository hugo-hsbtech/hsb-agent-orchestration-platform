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

---

## Verification results (2026-05-25)

Consistency verification pass run against fileKey `wgCEx2MmeF3tGDeulGmVqI`. Method: screenshot + `get_metadata` spot-checks on representative frames for each page; entry-point buttons confirmed via Screens page screenshots.

### Per-page conformance

| Page | Node | Frames checked | Checklist result | Notes |
|---|---|---|---|---|
| Agent creation | `265:2` | `106:221` (Step 1), `106:317` (Step 7) | **PASS** | Top bar: `Conductor · New agent` + `Exit ✕`. Stepper: dots+bars, `Step N of 7`, no per-step labels. `acme-concierge` / `acme` namespace / specialists `payments·technical·knowledge` / `acme (Qdrant)` KB all consistent with Screens page story. 1440×1024 frames confirmed. |
| MCP creation | `266:2` | `198:58` (Step 1), `204:64` (Step 5) | **PASS** | Top bar: `Conductor · Add MCP`. Stepper: dots+bars, `Step N of 5`. Linear MCP with `https://mcp.linear.app/sse`, namespace scope `acme` — story-consistent. |
| Namespace creation | `267:2` | `211:62` (Step 1), `212:110` (Step 4) | **PASS** | Top bar: `Conductor · New namespace`. Stepper: dots+bars, `Step N of 4`. `acme-eu` / `eu-west-1` — secondary namespace consistent with `acme` story. |
| KB source | `268:2` | `223:62` (Step 1), `227:62` (Step 5) | **PASS** | Top bar: `Conductor · Add knowledge source`. Stepper: dots+bars, `Step N of 5`. Collection `acme`, Qdrant, 9,841 chunks — consistent with Screens KB screen. |
| Trigger creation | `270:2` | `239:62` (Step 1), `243:79` (Step 3) | **PASS** | Top bar: `Conductor · New trigger`. Stepper: dots+bars, `Step N of 3`. Cron `0 2 * * *` targeting `kb-refresh` at Production v3 — story-consistent. |
| Credentials | `271:2` | `247:82` (Step 1), `249:81` (Step 3) | **PASS** | Top bar: `Conductor · Add credential`. Stepper: dots+bars, `Step N of 3`. Provider API key for Claude, namespace `acme`, vault path `vault://acme/anthropic` — story-consistent. |
| Lists pattern | `278:2` | `302:56` (Datatable) | **fixed-1** | See fix below. All other checklist items pass: full app shell (no takeover), filter bar, view toggle, state colors used for state only (`Production`/`Canary`/`Staging` badges), `acme` story agents present. |

### Entry-point buttons (Screens page)

| Screen | Frame | Button | Result |
|---|---|---|---|
| Agent Catalogue | `23:3` | `+ New agent` (brand/tide, content header) | **PASS** |
| Tools & MCPs | `56:56` | `+ Add MCP` (brand/tide, content header) | **PASS** |
| Admin / Namespaces | `57:56` | `New namespace` (brand/tide, content header) + `+ Add credential` (Credentials card) | **PASS** |
| Knowledge Base | `68:56` | `+ Add source` (brand/tide, content header) | **PASS** |
| Scheduling | `67:56` | `+ New trigger` (brand/tide, content header) | **PASS** |

### Fix applied — `302:56` sentinel + count removal

**Problem:** Frame `302:56` (`Lists · Agent Catalogue (Datatable)`) contained two elements inconsistent with the Datatable/numbered-pager pattern:
- Node `302:246` — `Count row` frame, containing text `Showing 24 of 240` (BlockGrid-only UI element).
- Node `302:337` — `BlockGrid sentinel — loading-more` frame with animated dots + `Loading more agents…` text (belongs exclusively to infinite-scroll BlockGrid view, not the numbered pager).

Both elements coexisted with a fully-formed numbered pager (`305:156`) showing `1-20 of 240` and pages `1 2 3 … 12`, creating a contradictory mixed datatable+BlockGrid state.

**Fix:** Both nodes deleted from the Figma file via `use_figma` (Plugin API). Confirmed by post-fix screenshot: the frame now shows only the DataTable with numbered pager footer. No other elements removed.

**Screenshot confirmation:** Post-fix render of `302:56` shows DataTable with rows, `Rows per page 20`, range `1–20 of 240`, and numbered pager `‹ 1 2 3 … 12 ›`. Sentinel and count row absent.

### Items flagged for follow-up (not fixed)

- **BlockGrid sentinel on Screens page BlockGrid views** (`23:3`, `56:56`): "Showing N of N" and "Loading more…" sentinels are correctly present on the Blocks-view Screens frames — these are appropriate for BlockGrid and were not modified.
- **Component instancing debt** (pre-existing, tracked in Known Consistency Debts above): hand-built chips and steppers on flow pages are visually conformant but not live component instances. Will be resolved in the global shadcn swap pass.
- **Token binding depth**: spot-checks via `get_metadata` on `302:56` confirm named variable references in the design system are used; a deeper `get_variable_defs` audit per-frame is deferred to the shadcn swap pass, which will re-bind all fills systematically.

### Summary

All seven flow pages pass the contract checklist on the categories verifiable via screenshot + metadata (top bar, stepper style, frame dimensions, story consistency, entry-point buttons, state color semantics). One targeted fix was applied to the Lists pattern Datatable frame (`302:56`) to remove the contradictory BlockGrid sentinel and count row. No broader drift requiring immediate intervention was found.
