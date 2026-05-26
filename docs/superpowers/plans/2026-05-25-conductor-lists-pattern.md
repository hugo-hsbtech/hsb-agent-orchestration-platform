# Conductor Lists Pattern — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a reusable **List** pattern (faceted filter bar + Blocks⇄Datatable view toggle + dual pagination) to the Conductor Figma showcase, apply it to 3 representative screens, and document the shared pagination contract.

**Architecture:** Figma-only (no code). Build four reusable constructs (FilterBar, ViewToggle, BlockGrid-with-sentinel, DataTable-with-pager) on the Components page, themed to Paper & Signal. Apply the **default** view in place on three screens (Agent Catalogue, Tool & MCP, Runs) and show each screen's **toggled** view + a shared **states gallery** on a new "Lists pattern" band of the Journeys page. The Blocks⇄Datatable toggle couples layout + pagination (Blocks⇒infinite scroll, Datatable⇒numbered). Verification is visual via `get_screenshot`; git commits cover only `docs/` deltas.

**Tech Stack:** Figma + Figma MCP (`use_figma`, `get_design_context`, `get_screenshot`, `get_metadata`, `get_variable_defs`); Paper & Signal Figma variables/styles; fonts Bricolage Grotesque / Hanken Grotesk / JetBrains Mono.

---

## Spec

Design spec: `docs/superpowers/specs/2026-05-25-conductor-lists-pattern-design.md`. Read it before starting.

## Working facts (from `docs/ui/08-build-status.md`, `02-design-system.md`)

- **Figma file:** `fileKey wgCEx2MmeF3tGDeulGmVqI`. Pages: `Cover` (0:1), `Foundations` (15:2), `Components` (15:3), `Screens` (23:2), `Journeys` (82:2).
- **Screens to apply to (Screens page):** Agent Catalogue `23:3` (card blocks today), Tool & MCP Catalogue `56:56` (card blocks today), Runs `51:56` (table today).
- **Existing components/primitives:** Version State Badge `20:23`, Provider Chip `21:14`, Connector card `91:49`, Result chip `89:46`. The Runs (`51:56`) and Admin Namespaces (`57:56`) screens already contain a dense table built in-place — reference their table structure for the DataTable construct.
- **Color variables:** `surface/canvas` `surface/card` `surface/sunken` `border/hairline` `text/ink` `text/muted` `text/faint` `brand/tide` `brand/tide-bright` `brand/tide-wash` `state/{draft,staging,canary,production,running,error,paused}` `provider/*`.
- **Text styles:** Display/XL, Display/L, Heading/H1, Heading/H2, Body/L, Body/M, Body/S, Strong/M, Label/M, Label/S, Mono/M, Mono/S, Mono/Medium.
- **Effects:** Shadow/Card, Shadow/Raised, Shadow/Overlay.

## Figma gotchas (apply on every `use_figma` write)

- Use `figma.createAutoLayout()` (hugs both axes). Do NOT `createFrame`+`layoutMode` (stays 100×100).
- Append children to an auto-layout parent BEFORE setting `layoutSizing='FILL'/'HUG'`.
- Never `resize()` AFTER setting FILL.
- Name-based variable/style lookups (no `setSharedPluginData` on those).
- `use_figma` writes are strictly sequential per file — never parallelize mutations.
- Invoke the `/figma-use` skill before any `use_figma` write.

## Layout for new frames (dedicated `Lists pattern` Figma page)

Per the page-per-flow structure, the Lists pattern lives on its **own Figma page** named `Lists pattern` (created in Task L0). It MUST conform to `docs/ui/09-consistency-contract.md` (shared tokens/fonts/branding, shared component instances, naming + 1440×1024 frame conventions, the shared `acme` story).
- Toggled-view frames (1440×1024) at y=0: Agent-Catalogue-as-Datatable `x=0`; Tool&MCP-as-Datatable `x=1540`; Runs-as-Blocks `x=3080`.
- States gallery at y=1240: smaller frames (≈ 680×460) left→right — Empty, Filtered-empty, Loading (blocks), Loading (table), Error, End-of-list.

## Verification model (no code/tests)

Each task verified by `get_screenshot` on the relevant node IDs → download via `curl` → view the PNG → confirm it matches the spec. Git commits cover only `docs/` updates (the Figma file lives in Figma cloud). Each task ends by recording node IDs in `docs/ui/08-build-status.md` and committing that delta.

---

### Task L0: Reusable List constructs (Components page `15:3`)

**Files:** Figma Components page `15:3`; Modify `docs/ui/08-build-status.md`.

- [ ] **Step 1: Open session + create the `Lists pattern` page.** Invoke `/figma-use`; `use_figma` on fileKey `wgCEx2MmeF3tGDeulGmVqI`. Create a new Figma page named `Lists pattern` (for the applied/state frames in L1–L4). Record its page ID.

- [ ] **Step 2: Build `FilterBar` component.** Auto-layout (vertical) on `surface/card`, hairline bottom border, full-width:
  - Row 1 (horizontal, space-between): left group = Search field (`🔍` + placeholder "Search…", Body/S, `surface/sunken` input, radius 8) + up to 5 **facet dropdowns** (each: label + `▾`, Label/S, radius 8, hairline border; show a `brand/tide-wash` count badge when active); right group = **Sort** control (`⇅ Sort: {field}`) + a **ViewToggle** instance slot.
  - Row 2 (horizontal, wraps): **active-filter chips** — removable chip component (`brand/tide-wash` bg, `text/ink`, `✕`) + "Clear all" text button (`brand/tide`). This row hides when empty (build it visible for the showcase).
  Record the FilterBar node ID. Verify with `get_screenshot`.

- [ ] **Step 3: Build `ViewToggle` component set.** Segmented control, variants `View=Blocks` / `View=Table`, each with an icon (grid / rows) + label. Active segment `brand/tide` fill + `text/on-brand`; inactive `text/muted` on `surface/card`. Verify screenshot.

- [ ] **Step 4: Build `BlockGrid sentinel` component set.** Variants `State=loading-more` (a row of 3 skeleton cards — `surface/sunken` blocks, radius 12 — + "Loading more…" Label/S `text/muted` centered) and `State=end` (a centered hairline rule + "End of list · 240 items" Label/S `text/faint`). Also a small **count** text element `Showing 24 of 240` (Mono/S, `text/muted`). Verify screenshot.

- [ ] **Step 5: Build `DataTable + Pager` construct.** A dense table: header row (sortable column labels Strong/S + a sort caret on the active column, `surface/sunken` header bg, hairline rules), 6–8 dense body rows (Body/S; machine values Mono/S; per-row Version State Badge / Provider Chip instances where natural). **Footer** (horizontal, space-between): left `Rows per page [20 ▾]`; center `1–20 of 240` (Mono/S `text/muted`); right numbered pager `‹ 1 2 3 … 12 ›` (current page `brand/tide` filled, others `text/muted`, chevrons hairline buttons). Reference the existing Runs `51:56` / Namespaces `57:56` table for proportions. Verify screenshot.

- [ ] **Step 6: Record IDs + commit.** In `docs/ui/08-build-status.md` Components section, add rows for `FilterBar`, `ViewToggle`, `BlockGrid sentinel`, `DataTable + Pager` with node IDs. Commit only that file:
```
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record reusable List-pattern constructs"
```
(Footer convention: `Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>`. Don't stage other files.)

---

### Task L1: Apply to Agent Catalogue (`23:3`) — default Blocks + toggled Datatable

**Files:** Figma `23:3` (in place) + Journeys page `82:2` (alt frame); Modify `docs/ui/08-build-status.md`.

Facets (per spec §D): Type (chatbot/workflow), Provider, Version state, Namespace, Owner. Default sort: Last run desc. Active chips example: `(Chatbot ✕)(Production ✕)`.

- [ ] **Step 1: Inspect.** `get_design_context` on `23:3` to locate the existing header (title + segmented `All/Chatbots/Workflows` + Search + `New agent` button `25:30`) and the card grid.

- [ ] **Step 2: Add the FilterBar (default Blocks state) in place.** Insert a `FilterBar` instance directly below the page header, configured with the 5 facets above + Search + Sort + a `ViewToggle` set to **Blocks**. Show two active-filter chips (`Chatbot`, `Production`). The existing `All/Chatbots/Workflows` segmented control is superseded by the Type facet — remove it (additive-minus: remove only that control). Keep the `New agent` button.

- [ ] **Step 3: Add the infinite-scroll affordance.** Below the card grid, add a `BlockGrid sentinel` instance in `State=loading-more` + the `Showing 24 of 240` count above the grid (right-aligned). This is the designed infinite-scroll state.

- [ ] **Step 4: Verify default state.** `get_screenshot` `23:3`; download + view. Expect: FilterBar with chips + Blocks toggle active, card grid, loading-more sentinel, count. No layout regression to cards.

- [ ] **Step 5: Build the toggled Datatable frame.** On the `Lists pattern` page at `x=0, y=0`, clone `23:3` and replace the card grid with a `DataTable + Pager` instance: columns Name · Type · Provider · State · Namespace · Owner · Last run; rows = the same ~8 agents from the catalogue (Version State Badge + Provider Chip instances); footer pager `1–20 of 240`, page 1 active. Set the FilterBar's ViewToggle to **Table**. Name the frame "Lists · Agent Catalogue (Datatable)".

- [ ] **Step 6: Verify toggled state.** `get_screenshot` the new frame; download + view. Expect: same FilterBar + chips, Table toggle active, dense table with the same agents, numbered pager.

- [ ] **Step 7: Record + commit.** Add a "Lists — Agent Catalogue" sub-entry to `08-build-status.md` (default `23:3`, toggled frame node ID). Commit only that file:
```
git add docs/ui/08-build-status.md
git commit -m "docs(ui): apply List pattern to Agent Catalogue (blocks default + datatable)"
```

---

### Task L2: Apply to Tool & MCP Catalogue (`56:56`) — default Blocks + toggled Datatable

**Files:** Figma `56:56` (in place) + Journeys page `82:2` (alt frame); Modify `docs/ui/08-build-status.md`.

Facets (per spec §D): Kind (tool/MCP), Auth status, Namespace, Capability. Default sort: Name asc.

- [ ] **Step 1: Inspect.** `get_design_context` on `56:56` to locate the header (incl. the `Add MCP` button `56:284`) and the card list.

- [ ] **Step 2: Add the FilterBar (default Blocks) in place.** Insert a `FilterBar` instance below the header with the 4 facets + Search + Sort + ViewToggle=**Blocks**. Show one active chip (e.g. `Connected ✕`). Keep the `Add MCP` button.

- [ ] **Step 3: Add infinite-scroll affordance.** `BlockGrid sentinel` `State=loading-more` below the grid + `Showing 18 of 64` count.

- [ ] **Step 4: Verify default.** `get_screenshot` `56:56`; download + view. Expect FilterBar + Blocks + sentinel, no regression.

- [ ] **Step 5: Build toggled Datatable frame.** On the `Lists pattern` page at `x=1540, y=0`, clone `56:56` and replace the card list with a `DataTable + Pager`: columns Name · Kind · Capability · Auth · Namespace · Version; rows = the same tools/MCPs (Auth status via Result chip; Kind tag); pager `1–20 of 64`. ViewToggle=**Table**. Name "Lists · Tool & MCP (Datatable)".

- [ ] **Step 6: Verify toggled.** `get_screenshot` the frame; download + view.

- [ ] **Step 7: Record + commit.**
```
git add docs/ui/08-build-status.md
git commit -m "docs(ui): apply List pattern to Tool & MCP Catalogue (blocks default + datatable)"
```

---

### Task L3: Apply to Runs (`51:56`) — default Datatable + toggled Blocks

**Files:** Figma `51:56` (in place) + Journeys page `82:2` (alt frame); Modify `docs/ui/08-build-status.md`.

Facets (per spec §D): Status, Agent, Namespace, Started-by, Time range. Default sort: Started desc. Runs is table-first, so the **default** here is the Datatable with numbered pagination.

- [ ] **Step 1: Inspect.** `get_design_context` on `51:56` to locate its existing run table + header.

- [ ] **Step 2: Add the FilterBar (default Datatable) in place.** Insert a `FilterBar` instance below the header with the 5 facets + Search + Sort + ViewToggle=**Table**. Show active chips (e.g. `Failed ✕`, `Last 24h ✕`). Ensure the existing run table reads as the `DataTable + Pager` construct: add/confirm the numbered pager footer (`1–20 of 312`, page 1 active) if not already present.

- [ ] **Step 3: Verify default.** `get_screenshot` `51:56`; download + view. Expect FilterBar (Table toggle active) + dense run table + numbered pager.

- [ ] **Step 4: Build toggled Blocks frame.** On the `Lists pattern` page at `x=3080, y=0`, clone `51:56` and replace the table with a **run-card** BlockGrid: each card = run id (Mono) + status badge + agent + pinned version + duration + cost (Mono); add a `BlockGrid sentinel` `State=loading-more` + `Showing 24 of 312` count. ViewToggle=**Blocks**. Name "Lists · Runs (Blocks)". (Design a simple run-card if none exists — small card mirroring the table row's fields.)

- [ ] **Step 5: Verify toggled.** `get_screenshot` the frame; download + view.

- [ ] **Step 6: Record + commit.**
```
git add docs/ui/08-build-status.md
git commit -m "docs(ui): apply List pattern to Runs (datatable default + blocks)"
```

---

### Task L4: Pattern states gallery (Journeys page `82:2`)

**Files:** Figma `82:2` (states gallery); Modify `docs/ui/08-build-status.md`.

- [ ] **Step 1: Build the band + 6 state frames.** On the `Lists pattern` page: band label "Lists · states" (Display/L) at y=1160. Six frames (≈680×460) at y=1240, left→right at `x = i×760`:
  1. **Empty (no data)** — centered illustration placeholder + "No agents yet" (Heading/H2) + a `brand/tide` primary "New agent" button.
  2. **Filtered-to-nothing** — "No results for these filters" (Heading/H2, `text/muted`) + "Clear all" button; the FilterBar (with chips) shown above.
  3. **Loading (blocks)** — 6 skeleton cards (`surface/sunken`, radius 12) + the `loading-more` sentinel.
  4. **Loading (table)** — table header + 6 skeleton rows (`surface/sunken` bars).
  5. **Error** — inline error row (`state/error` accent) "Couldn't load — retry" + a Retry button.
  6. **End of list (blocks)** — last 2 cards + the `BlockGrid sentinel` `State=end` ("End of list · 240 items").

- [ ] **Step 2: Verify.** `get_screenshot` each state frame (or the band); download + view. Expect the 6 states reading clearly with tokens bound.

- [ ] **Step 3: Record + commit.**
```
git add docs/ui/08-build-status.md
git commit -m "docs(ui): add List-pattern states gallery (empty/loading/error/end)"
```

---

### Task L5: Documentation

**Files:** Create `docs/ui/patterns/lists.md`; Modify `docs/ui/08-build-status.md`, `docs/ui/README.md`.

- [ ] **Step 1: Write `docs/ui/patterns/lists.md`.** Sections: (a) Pattern anatomy (FilterBar / ViewToggle / BlockGrid / DataTable) with the component node IDs from Task L0; (b) **Pagination contract** — paste the request/response signature from spec §C verbatim, with the Blocks-cursor vs Table-offset consumption notes; (c) **Facet model** — the per-screen facet table from spec §D; (d) the **toggle coupling** rule (Blocks⇒infinite, Table⇒numbered, no mixing); (e) **Applied screens** — the 3 screens, their default view, and the toggled-frame node IDs; (f) **Mapping to remaining list screens** — a table assigning a default view + facet set to KB Management, Marketplace, Scheduling, Compliance audit, Namespaces, Memory, Evaluations (default view = Table for log/run-like screens, Blocks for catalogue-like screens); (g) note the global shadcn pass will standardize the constructs.

- [ ] **Step 2: Update `08-build-status.md`.** Add a "Lists pattern — DONE" section summarizing constructs + applied screens + states gallery + the doc link; tick the backlog item.

- [ ] **Step 3: Update `docs/ui/README.md`.** Add a row/link for `patterns/lists.md` in the docs table.

- [ ] **Step 4: Commit.**
```
git add docs/ui/patterns/lists.md docs/ui/08-build-status.md docs/ui/README.md
git commit -m "docs(ui): document the Lists pattern (anatomy, pagination contract, facet model, mapping)"
```

---

## Self-review notes

- **Spec coverage:** anatomy (§A) → Task L0; two views + pagination (§B) → L1–L3 (each screen shows both); pagination contract (§C) → documented in L5 step 1(b); facet model (§D) → applied in L1–L3, documented L5 step 1(c); application to 3 screens (§E) → L1/L2/L3; states (§F) → L4; documentation deliverable (§G) → L5.
- **Toggle coupling** (Blocks⇒infinite, Table⇒numbered) is realized by pairing the ViewToggle state with the matching pagination affordance in every applied frame.
- **No placeholders:** every construct + applied frame has concrete fields, facets, mock counts, and node-ID placements; verification is `get_screenshot`; commits are doc deltas.
- **Naming consistency:** construct names (`FilterBar`, `ViewToggle`, `BlockGrid sentinel`, `DataTable + Pager`) and screen node IDs (`23:3`, `56:56`, `51:56`) used consistently across tasks.
- **Sequencing:** runs after journeys (done), before the global shadcn swap (which will standardize these constructs).
