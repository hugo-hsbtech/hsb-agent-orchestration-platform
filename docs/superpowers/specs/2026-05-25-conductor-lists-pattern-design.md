# Conductor — Lists Pattern (Filter + Blocks⇄Datatable + Dual Pagination) — Design Spec

> **Date:** 2026-05-25 · **Track:** Figma design + docs (no running code), extending the "Conductor — Paper & Signal" showcase (`fileKey wgCEx2MmeF3tGDeulGmVqI`).
> **Problem:** The list-bearing screens render inconsistently (some card **blocks**, some **tables**) with weak filtering. We want one reusable **List** pattern: a faceted filter bar, a **Blocks⇄Datatable** view toggle, and **dual pagination** (blocks = infinite scroll, datatable = numbered) — both views driven by **one shared paginated datasource signature**.
> **Goal:** Design the List pattern once as reusable Figma constructs, apply it to 3 representative screens, document the shared pagination contract and facet model, and specify how it maps to the remaining list screens.

---

## Decisions locked (from brainstorming)

| Decision | Choice |
|---|---|
| **Scope** | Build the pattern ONCE as reusable Figma constructs; apply to **3 representative screens** (Agent Catalogue, Tool & MCP Catalogue, Runs). Document mapping to the rest; the global shadcn pass rolls it out everywhere. |
| **Pagination contract** | Document a **concrete shared signature** (request params + response shape) supporting both consumption styles; design both views to honor it visually (same fields + total count). Design + docs, **no code**. |
| **Filtering** | **Faceted filter bar** (search + per-facet dropdowns + sort) with **active-filter chips** (removable + "Clear all"). Facets double as datatable column filters. |
| **Toggle coupling** | The view toggle couples **layout + pagination**: Blocks ⇒ infinite scroll; Datatable ⇒ numbered pages. No mixing. |
| **Fidelity** | Static designed states, mock data, Figma-only — consistent with the existing screens + journeys. Each applied screen shown in **2 states** (default view + toggled view). |

---

## A. Pattern anatomy

A **List** region sits directly below a screen's page header. Composed of four reusable constructs (built on the Components page `15:3`, themed to Paper & Signal; a later shadcn pass swaps primitives):

1. **FilterBar** — auto-layout row on `surface/card` with a hairline bottom border:
   - **Search** field (left; `🔍` + placeholder, Body/S).
   - **Facet dropdowns** (per-screen; e.g. Type, Provider, State, Namespace, Owner) — each a `Select`/dropdown with a count badge when active.
   - **Sort** control (field + direction).
   - **ViewToggle** (right-aligned).
   - **Active-filter chips** row beneath: removable chips (`brand/tide-wash` bg, `✕`) + a "Clear all" text button. Hidden when no filters active.
2. **ViewToggle** — segmented control, two options with icons: **Blocks** (grid icon) / **Table** (rows icon). Active segment `brand/tide` on `surface/card`. Switching swaps layout **and** pagination mode.
3. **BlockGrid (infinite scroll)** — responsive card grid (reuses each screen's existing card). Bottom affordance: a **sentinel** state — a row of 2–3 skeleton cards + "Loading more…" (Label/S, `text/muted`); plus a quiet `Showing 24 of 240` count (Mono/S) above or below the grid. The "end of list" state shows a subtle end rule.
4. **DataTable (numbered pagination)** — dense sortable table (reuses the design system **Table** primitive already used by Runs `51:56` / Namespaces `57:56`): sortable column headers (with a sort caret), dense rows (Body/S, machine values Mono/S), per-row state via Version State Badge / Result chips where relevant. **Footer**: `Rows per page [20 ▾]` · `1–20 of 240` (Mono/S) · numbered pager `‹ 1 2 3 … 12 ›` (current page `brand/tide`).

## B. The two views + pagination behavior

The same filtered + sorted result set renders either way; the toggle chooses presentation + pagination:

| View | Layout | Pagination | Consumes contract via |
|---|---|---|---|
| **Blocks** | card grid | **infinite scroll** (append on scroll; sentinel → end rule) | `cursor + limit`, follows `nextCursor` |
| **Datatable** | dense table | **numbered pages** (pager + rows-per-page) | `offset + limit`, uses `total` for page count |

Facets selected in the FilterBar apply identically to both views (in the table they also surface as column-header filter affordances). Sort applies to both. Changing filters/sort resets pagination.

## C. Shared pagination contract (the "signature")

One query shape both views honor — documented as design intent (no backend in this track):

```
Request:
  {
    filter: { <facetKey>: <value | value[]>, … },   // e.g. { type: "chatbot", state: ["production","canary"] }
    sort:   { field: string, dir: "asc" | "desc" },
    page:   { cursor?: string, offset?: number, limit: number }   // cursor (blocks) OR offset (table)
  }

Response:
  {
    items:      [ … ],          // same item shape regardless of view
    total:      number,         // used by datatable to compute page numbers
    nextCursor: string | null   // used by blocks to fetch the next page; null = end
  }
```

- **Blocks** send `page.cursor` (omit on first load) + `limit`; append `items`; stop when `nextCursor` is `null`.
- **Datatable** sends `page.offset` + `limit`; render `total / limit` pages; jump by setting `offset`.
- **One endpoint, two consumption styles.** The item shape is identical, so toggling views never changes *what* a row/card represents — only how it's laid out and paged. The Figma designs show the **same fields and the same `total` count** in both views to make the shared source legible.

## D. Filtering — facet model

The FilterBar is configured per-screen with a facet set. Facets for the 3 representative screens:

| Screen | Search | Facets | Default sort |
|---|---|---|---|
| **Agent Catalogue** | name / id | Type (chatbot/workflow), Provider, Version state, Namespace, Owner | Last run desc |
| **Tool & MCP Catalogue** | name | Kind (tool/MCP), Auth status, Namespace, Capability | Name asc |
| **Runs** | run id / agent | Status, Agent, Namespace, Started-by, Time range | Started desc |

Active facets render as removable chips. "Clear all" resets to the default (unfiltered) view. Facet dropdowns show multi-select where the field is multi-valued (e.g. Version state).

## E. Application — reusable constructs + 3 representative screens

**Build (Components page `15:3`):** `FilterBar`, `ViewToggle`, `BlockGrid sentinel` (loading-more + end states), `DataTable + Pager` (footer with rows-per-page + numbered pager). Themed to Paper & Signal, tokens bound by name.

**Apply (Screens page `23:2`), each in 2 states:**

| Screen | Node | Default view | Toggled view shown |
|---|---|---|---|
| Agent Catalogue | `23:3` | **Blocks** (infinite scroll, sentinel) | Datatable (numbered) |
| Tool & MCP Catalogue | `56:56` | **Blocks** | Datatable |
| Runs | `51:56` | **Datatable** (numbered) | Blocks |

The two states per screen are realized as: the existing screen updated in place with the FilterBar + default view, plus **one alternate-state frame** (a clone showing the toggled view) placed on a "Lists" band on the Journeys page or adjacent to the screen — exact placement decided in the plan. Mapping notes for the remaining list screens (KB, Marketplace, Scheduling, Compliance audit, Namespaces, Memory, Evaluations) are documented, not built.

## F. States (designed once on the pattern)

- **Empty (no data):** illustrative empty state + primary "New …" CTA.
- **Filtered-to-nothing:** "No results for these filters" + "Clear all".
- **Loading:** blocks → skeleton cards + sentinel; table → skeleton rows.
- **Error:** inline error row + retry.
- **End of list (blocks):** subtle end rule + final count.

## G. Documentation deliverable

- A `docs/ui/` doc (e.g. `docs/ui/patterns/lists.md` or a section) capturing: the pattern anatomy, the **pagination contract** (§C verbatim), the **facet model** (§D), the toggle coupling rule, the per-screen application, and the mapping to remaining screens.
- Update `08-build-status.md` with the new component sets + applied-screen states.

## Out of scope

- Working code / a real data layer (Figma + docs track).
- Rolling the pattern onto every list screen now (only 3 representative; the rest mapped in docs + handled by the shadcn pass).
- Saved/named filter views and an advanced query builder (faceted bar chosen over the query-builder option).
- Dark mode for the new constructs (inherits the single-mode decision).

## Sequencing

Per the agreed order: this Lists pattern comes **after** the journeys (done) and **before** the global shadcn swap (so the swap standardizes the new list constructs too).
