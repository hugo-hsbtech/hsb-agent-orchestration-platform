# Lists Pattern — Filter · Blocks⇄Datatable · Dual Pagination

> The reusable pattern for every list-bearing surface in Conductor: a faceted **filter bar**, a **Blocks⇄Datatable** view toggle, and **dual pagination** (Blocks = infinite scroll, Datatable = numbered pages) — both views driven by **one shared paginated datasource signature**. Design + docs only (no running code). Spec: [`../superpowers/specs/2026-05-25-conductor-lists-pattern-design.md`](../../superpowers/specs/2026-05-25-conductor-lists-pattern-design.md).

## A. Anatomy (reusable constructs)

Built on the Components page (`15:3`), themed to Paper & Signal; live on the dedicated **`Lists pattern`** Figma page (`278:2`) where applied/state frames are assembled.

| Construct | Node ID | What it is |
|---|---|---|
| **FilterBar** | `280:37` | Search + per-facet dropdowns (count badge when active) + Sort + a ViewToggle slot; an active-filter **chips** row beneath (removable chips + "Clear all"). |
| **ViewToggle** | `279:67` (Blocks `279:65` / Table `279:66`) | Segmented control; switching it swaps **layout + pagination together**. |
| **BlockGrid sentinel** | `285:70` (loading-more `285:68` / end `285:69`) + count `285:71` | Infinite-scroll affordances for the Blocks view. |
| **DataTable + Pager** | `286:50` | Dense sortable table + footer pager (`Rows per page` · `1–20 of N` · `‹ 1 2 3 … ›`). |

All bind Paper & Signal variables/styles by name (no bare hex); machine values (ids, counts, durations, costs, versions) use JetBrains Mono.

## B. Toggle coupling rule

The view toggle is **not** orthogonal to pagination — it couples them:

| View | Layout | Pagination | Affordances |
|---|---|---|---|
| **Blocks** | card grid | **infinite scroll** | BlockGrid sentinel (`loading-more` → `end`) + a `Showing N of M` count |
| **Datatable** | dense table | **numbered pages** | footer pager only — **no** sentinel, **no** top "Showing N of M" count |

Changing filters or sort resets pagination. The sentinel/count belong to the Blocks view **only**.

## C. Shared pagination contract (the "signature")

One query shape both views honor (design intent; no backend in this track):

```
Request:
  {
    filter: { <facetKey>: <value | value[]>, … },   // e.g. { type: "chatbot", state: ["production","canary"] }
    sort:   { field: string, dir: "asc" | "desc" },
    page:   { cursor?: string, offset?: number, limit: number }   // cursor (blocks) OR offset (table)
  }

Response:
  { items: [ … ], total: number, nextCursor: string | null }
```

- **Blocks** send `page.cursor` (omit on first load) + `limit`; append `items`; stop when `nextCursor` is `null`.
- **Datatable** send `page.offset` + `limit`; render `total / limit` pages; jump by setting `offset`.
- **One endpoint, two consumption styles.** The `items` shape is identical, so toggling never changes *what* a row/card represents — only how it's laid out and paged. Both Figma views display the **same fields + the same `total`** to make the shared source legible.

## D. Facet model (per applied screen)

The FilterBar is configured per-screen with a facet set:

| Screen | Search | Facets | Default sort |
|---|---|---|---|
| **Agent Catalogue** | name / id | Type · Provider · Version state · Namespace · Owner | Last run desc |
| **Tool & MCP Catalogue** | name | Kind · Auth status · Namespace · Capability | Name asc |
| **Runs** | run id / agent | Status · Agent · Namespace · Started-by · Time range | Started desc |

Active facets render as removable chips; "Clear all" resets to the default unfiltered view. Multi-valued fields (e.g. Version state) use multi-select. In the Datatable view the same facets surface as column-header filters.

## E. Applied screens (each shown in 2 states)

| Screen | Default (in place) | Toggled frame (`Lists pattern` page `278:2`) |
|---|---|---|
| **Agent Catalogue** `23:3` | **Blocks** + FilterBar + infinite-scroll sentinel | Datatable — `302:56` |
| **Tool & MCP Catalogue** `56:56` | **Blocks** + FilterBar + sentinel | Datatable — `319:2` |
| **Runs** `51:56` | **Datatable** + FilterBar + numbered pager | Blocks (run-cards) — `335:2` |

Runs is table-first (its default is the Datatable); the catalogues are blocks-first. Each toggled frame is a 1440×1024 clone showing the alternate view so the toggle reads.

**States gallery** (`278:2`, y=1240): Empty `343:2` · Filtered-to-nothing `345:2` · Loading-blocks `346:2` · Loading-table `347:2` · Error `348:2` · End-of-list `349:2`.

## F. Mapping to the remaining list screens

Not built (designed on the 3 representative screens above); the global shadcn pass rolls the pattern out everywhere. Suggested default view + facets:

| Screen | Node | Default view | Facets |
|---|---|---|---|
| KB Management (sources) | `68:56` | Blocks | Source type · Status · Collection · Freshness |
| Marketplace | `71:56` | Blocks | Category · Rating · Provider · Publisher |
| Scheduling (triggers) | `67:56` | Datatable | Trigger type · Status · Target agent · Namespace |
| Compliance (audit log) | `72:56` | Datatable | Event type · Actor · Severity · Time range |
| Admin (namespaces) | `57:56` | Datatable | Region · Tier · Status |
| Memory Inspector | `74:56` | Datatable | Tier (working/episodic/semantic) · User · Recency |
| Evaluations | `44:40` | Datatable | Suite · Result · Agent version · Time range |

> Log/run-like surfaces default to **Datatable** (numbered pages); catalogue-like surfaces default to **Blocks** (infinite scroll). Every adopting screen reuses the same constructs (§A) and honors the same pagination contract (§C), per the [consistency contract](../09-consistency-contract.md).
