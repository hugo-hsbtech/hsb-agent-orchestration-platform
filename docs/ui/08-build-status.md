# 08 — Build Status & Handoff

> **Snapshot: 2026-05-25.** The single source of truth for *what actually exists right now*, with the exact Figma IDs needed to resume. Track is **Figma-only, no code** (see [README](./README.md), [05](./05-build-workflow.md)). The plan docs (01–05) describe intent; this file records reality.

## Figma file

| | |
|---|---|
| **File** | "Conductor — Paper & Signal" |
| **fileKey** | `wgCEx2MmeF3tGDeulGmVqI` |
| **URL** | https://www.figma.com/design/wgCEx2MmeF3tGDeulGmVqI |
| **Pages** | `Cover` (0:1) · `Foundations` (15:2) · `Components` (15:3) · `Screens` (23:2) · `Agent creation` (265:2) · `MCP creation` (266:2) · `Namespace creation` (267:2) · `KB source` (268:2) · `Trigger creation` (270:2) · `Credentials` (271:2) · `Lists pattern` (278:2) |

## Design system — DONE ✅ (Foundations page 15:2)

- **82 variables** across 4 collections: `Primitives` (34, mode *Value*) · `Color` (32, mode *Light* — dark mode is a structured future add) · `Spacing` (10) · `Radius` (6). Naming: `surface/*`, `text/*`, `border/*`, `brand/tide*`, `state/*` (draft·staging·canary·production·running·error·paused), `provider/*`.
- **13 text styles**: Display/XL, Display/L, Heading/H1, Heading/H2, Body/L, Body/M, Body/S, Strong/M, Label/M, Label/S, Mono/M, Mono/S, Mono/Medium.
- **3 effect styles**: Shadow/Card, Shadow/Raised, Shadow/Overlay.
- **Fonts**: Bricolage Grotesque (display) · Hanken Grotesk (body) · JetBrains Mono (machine layer).
- **Idempotency note:** variables/styles are *name-based* (the sandbox has no `setSharedPluginData` on Variable/Style objects — see [06](./06-learnings.md) §14).

## Components — DONE ✅ (Components page 15:3)

Built as proper Figma **component sets with variants** (themed to Paper & Signal):

| Component set | Node ID | Variants |
|---|---|---|
| **Version State Badge** | `20:23` | Draft · Staging · Canary · Production · Running · Failed · Paused |
| **Provider Chip** | `21:14` | Claude · OpenAI · Gemini · Grok |
| **Stepper** | `84:15` | Type=Takeover/Horizontal (`83:2`) · Type=Drawer/Compact (`84:2`) |
| **Drawer shell** | `87:30` | Size=480 (`86:2`) · Size=560 (`87:14`) |
| **Takeover shell** | `88:26` | single variant (1440×1024) |
| **Result chip** | `89:46` | success (`89:37`) · error (`89:40`) · pending (`89:43`) |
| **Connector card** | `91:49` | Selected=false (`91:37`) · Selected=true (`91:43`) |
| **Progress-step row** | `92:48` | done (`92:37`) · active (`92:41`) · pending (`92:45`) |
| **FilterBar** | `280:37` | single — vertical on `surface/card`, hairline bottom border, full-width. Row 1 (space-between): LEFT = Search field (`surface/sunken`, radius 8) + facet dropdowns Status (`brand/tide-wash` count badge `2`)/Provider/Owner (`▾`, Label/S, radius 8); RIGHT = Sort control (`⇅ Sort: Updated`) + **ViewToggle instance** slot (`284:41`). Row 2 (wraps): removable `brand/tide-wash` chips ("Status: Production ✕", "Provider: Claude ✕") + "Clear all" `brand/tide` text button |
| **ViewToggle** | `279:67` | View=Blocks (`279:65`) · View=Table (`279:66`) — segmented control; active segment `brand/tide` fill + `text/on-brand`, inactive `text/muted` on `surface/card`; grid / rows icons |
| **BlockGrid sentinel** | `285:70` | State=loading-more (`285:68`, 3 `surface/sunken` skeleton cards radius 12 + "Loading more…" Label/S `text/muted`) · State=end (`285:69`, centered hairline rule + "End of list · 240 items" Label/S `text/faint`). Companion **count** text `Showing 24 of 240` (Mono/S `text/muted`) at `285:71` |
| **DataTable + Pager** | `286:50` | single — dense table matching Runs/Namespaces proportions. Header `surface/sunken`, sortable Label/S column labels (active AGENT = `brand/tide-text` + `▾` caret); 6 dense rows (Body/S; Mono/S machine values) each with a **Version State Badge `20:23`** + **Provider Chip `21:14`** instance. Footer (`289:74`, space-between): left `Rows per page [20 ▾]`; center `1–20 of 240` (Mono/S `text/muted`); right numbered pager `‹ 1 2 3 … 12 ›` (current page `brand/tide` fill + `text/on-brand`, others `text/muted`, chevrons hairline buttons) |

### Creation-flow component sets (Task 0 — Journeys foundation)

The `Stepper`, `Drawer shell`, `Takeover shell`, `Result chip`, `Connector card`, and `Progress-step row` sets above are the reusable primitives the multi-step creation/onboarding journeys depend on (the new **`Journeys` page**, `82:2`). All bind Paper & Signal variables/styles by name (no hardcoded hexes); the machine register (counts) uses Mono/S, shells apply Shadow/Overlay (drawer) and Shadow/Card (takeover content column).

Other composites (App Shell sidebar/topbar, agent card, workflow node, step-timeline row, chat bubble, etc.) are currently built **in-place within screens** to shadcn spec, not yet extracted into standalone component sets. Extracting them is a future enhancement (and the target for the shadcn-kit swap below).

## Screens — 20 built ✅ (Screens page 23:2)

Laid out left→right at x = index × 1540. One shared **acme-support** customer-support story runs through all of them.

| # | Screen | Node ID | Nav (active) | Demonstrates |
|---|---|---|---|---|
| 01 | Agent Catalogue | `23:3` | Build › Agents | catalogue, version states, providers |
| 02 | Agent Editor — Workflow | `35:34` | Build › Agents | visual node graph + DSL + inspector |
| 03 | Validation | `41:36` | Build › Agents | collect-all errors (AD-13) |
| 04 | Playground | `42:36` | Build › Playground | test run, routing, step trace |
| 05 | Evaluation | `44:40` | Build › Evaluations | scorecards, A/B, LLM-as-judge (27) |
| 06 | Version Promotion | `48:40` | Build › Agents | draft→staging→canary→prod + canary split (AD-29) |
| 07 | Runs | `51:56` | Operate › Runs | run table, statuses |
| 08 | Run Detail | `52:56` | Operate › Runs | step timeline + step-debug (AD-17) |
| 09 | Chat | `54:56` | Engage › Chat | routing, background-task surfacing, citations, degradation, memory |
| 10 | Tool & MCP Catalogue | `56:56` | Catalog › Tools & MCPs | tools + MCPs, auth (AD-11/18) |
| 11 | Admin | `57:56` | Admin › Namespaces | tenancy, quotas, price table, vault |
| 12 | Analytics | `59:56` | Insights › Analytics | volume, routing dist, cost vs budget, KB hit rate (30) |
| 13 | Trace | `63:56` | Operate › Traces | OTel span waterfall + attributes (AD-24) |
| 14 | Health | `65:56` | Operate › Health | circuit breakers (AD-31) — open `kb_lookup` ties to Chat degradation |
| 15 | Scheduling | `67:56` | Catalog › Scheduling | cron / webhook / event triggers (29) |
| 16 | KB Management | `68:56` | Catalog › Knowledge Base | sources, ingestion, retrieval bench (33) |
| 17 | Marketplace | `71:56` | Catalog › Marketplace | shared catalogue, ratings, import (25/26) |
| 18 | Compliance | `72:56` | Admin › Compliance | audit log, PII, consent, residency (35) |
| 19 | Model Routing | `73:56` | Admin › Model routing | complexity cascade + cache (31) |
| 20 | Memory Inspector | `74:56` | Operate › Memory | working / episodic / semantic tiers + injection trace (AD-30) |

**Nav (final, consistent across all 20 sidebars):**
`BUILD` Agents · Playground · Evaluations — `OPERATE` Runs · Traces · Health · Memory — `ENGAGE` Chat — `CATALOG` Tools & MCPs · Knowledge Base · Scheduling · Marketplace — `INSIGHTS` Analytics — `ADMIN` Namespaces · Compliance · Model routing · Settings. *(Settings has no standalone screen — folds into the Admin/Namespaces view.)*

## Journeys — DONE ✅ (split into per-flow pages, 2026-05-25)

> **Page structure update (2026-05-25):** The single `Journeys` page (`82:2`) has been split into **one Figma page per flow**. Each flow's band label and frames were moved to a dedicated page and normalized so the band label sits at (0, 0) and frames run left→right at y=120, x = i×1540. The `Journeys` page was then emptied and deleted. The per-flow pages are: `Agent creation` (265:2) · `MCP creation` (266:2) · `Namespace creation` (267:2) · `KB source` (268:2) · `Trigger creation` (270:2) · `Credentials` (271:2).

Six multi-step **takeover-wizard** creation/onboarding flows, all unified on one style (the Agent-creation wizard): full-screen `surface/canvas` frame, top bar `Conductor · {flow}` + `Exit ✕`, centered horizontal stepper (`Step N of M`, no per-step text labels), centered ~720px `surface/card` content column (Shadow/Card, radius 12), sticky footer (ghost Back/Cancel + primary `brand/tide`). No side-drawers, scrims, or parent-screen clones — an earlier side-drawer approach for five of the journeys was **reworked to takeover** for consistency (per user direction, 2026-05-25). They compose the reusable creation-flow primitives (Stepper/Takeover shell, Connector card, Result chip, Progress-step row) plus Version State Badge / Provider Chip, over the same **acme** customer-support story. Each frame is 1440×1024 with a Display/L band label. *(The `Drawer shell` `87:30` and the Stepper `Drawer/Compact` variant `84:2` from Task 0 are now unused — retained for possible future use.)*

### Agent creation — takeover wizard ✅ (page 265:2)

Worked example: create top-level **chatbot `acme-concierge`** (router, draft `v0.1`) routing to the existing `payments` / `technical` / `knowledge` specialists. The 7-step chatbot spine plus one diverging workflow-tail frame. Each frame is a (detached) instance of the **Takeover shell** (`88:26`), top-bar flow name **Conductor · New agent**, with its Takeover/Horizontal Stepper rebuilt to the right step count/position.

| Frame | Step | Node ID |
|---|---|---|
| 1 | Type (Chatbot selected / Workflow) | `106:221` |
| 2 | Identity (`acme-concierge`, ns `acme`, owner, tags) | `106:237` |
| 3 | Model & Provider (Claude · `claude-opus-4-7` · temp 0.3 · 4096 · router prompt) | `106:253` |
| 4 | Routing & Specialists (LLM classifier · 0.72 · payments/technical/knowledge Production) | `106:269` |
| 5 | Knowledge & Memory (acme Qdrant KB · Working/Episodic 30d/Semantic) | `106:285` |
| 6 | Safety (Standard moderation · PII redaction · blocked topics · refusal) | `106:301` |
| 7 | Review & Create (two-col summary · Draft `v0.1` rail · Create as draft) | `106:317` |
| 8 | Workflow tail — Triggers → Open builder (Manual selected; 4-step path) | `106:333` |

**Entry point:** Agent Catalogue `23:3` carries the primary `New agent` button (`brand/tide` fill, white label bound to `text/on-brand`, right-aligned in the Catalogue Header toolbar) at node **`25:30`** → launches this wizard. The workflow tail's `Create draft & open builder` implies handoff to the existing Workflow editor (`35:34`).

Steppers: chatbot frames show 1→7 of 7; the workflow tail shows the shorter 4-step path (Type · Identity · Model · Triggers, step 4 of 4). All text binds Paper & Signal styles by name; the machine register (IDs, slug, numbers, model name, router prompt) uses JetBrains Mono (Mono/M, Mono/S).

### MCP creation — takeover wizard ✅ (page 266:2)

Worked example: **add the Linear MCP via OAuth**. Reworked from the old side-drawer flow into a **full-screen takeover wizard** matching the Agent-creation journey (one consistent creation style — no drawers, no scrim, no parent-screen clone). Five standalone frames, each a hand-built copy of the **Takeover shell** (`88:26`): full 1440×1024 on `surface/canvas`, top bar (`Conductor · Add MCP` left, `Exit ✕` right), a centered horizontal 5-step stepper (`① Connect ② Auth ③ Discover ④ Test ⑤ Register` — `Step N of 5`; done/active dots+bars bind `brand/tide`, upcoming bind `border/hairline`), a centered 720px `surface/card` content column (Shadow/Card, radius 12), and a sticky footer (ghost Back/Cancel + primary `brand/tide` button). Band label **"MCP creation"** (Display/L, `text/ink`, node `128:56`) is unchanged above frame 1. Laid out at x = i×1540, y = 1240. All fills/text bind Paper & Signal variables/styles by name (machine register — URLs, env keys, tool IDs, latency, name/version — uses JetBrains Mono); the Result chip instances the shared `89:46` component.

| Frame | Step | Node ID |
|---|---|---|
| 1 | Connect — Transport (SSE selected) · Server URL `https://mcp.linear.app/sse` · Env `LINEAR_WORKSPACE=acme` · Namespace `acme` · Cancel/Continue → | `198:58` |
| 2 | Auth — OAuth handoff "Connect to Linear" · brand/tide Authorize · Result chip `pending` "Waiting for authorization…" · Back/Continue (disabled) | `201:58` |
| 3 | Discover — Result chip `success` "Connected as acme" · checklists Tools (`create_issue`,`search_issues`,`add_comment`) / Resources (`issue://`,`team://`) / Prompts (`triage_issue`), all checked | `202:60` |
| 4 | Test — sample call `search_issues({"query":"refund"})` in a `surface/sunken` well · Result chip `success` + latency `412ms` | `203:62` |
| 5 | Register — summary Name `linear` · Version `1.0` · Scope `acme` · Auth `state/production` Connected chip · stepper fully filled · primary Register MCP | `204:64` |

**Entry point:** Tool & MCP Catalogue `56:56` already carries the primary `Add MCP` button in its content header (`brand/tide` fill at node **`56:284`**, label `56:288`) → launches this takeover wizard. The button **pre-existed** from the screen build and was left untouched; it was *not* duplicated (additive check satisfied). The takeover replaces the screen, so there is no parent-screen clone or scrim behind these frames (a change from the old drawer flow). The 5 old drawer frames (`128:57`, `142:56`, `143:68`, `144:80`, `145:92`) were deleted.

### Namespace creation — takeover wizard ✅ (page 267:2)

Worked example: **create namespace `acme-eu`** (EU data residency). Four standalone full-screen **takeover wizard** frames — matching the Agent and MCP wizard style exactly (no scrim, no drawer, no parent-screen clone). Hand-built on the **Takeover shell** (`88:26`) pattern: `surface/canvas` (#faf9f5) background, 1440×1024, white topbar (`Conductor · New namespace` left, `Exit ✕` right, 1px `border/hairline` bottom), centered horizontal 4-step stepper (done/active dots+bars bind `brand/tide`, upcoming bind `border/hairline`), centered 720px `surface/card` content column (Shadow/Card, radius 12, 40px padding), sticky footer (ghost Back/Cancel + primary `brand/tide` button). Band label **"Namespace creation"** (Display/L, `text/ink`, node `163:62`) is unchanged above frame 1. Laid out at x = i×1540, y = 2480. All fills/text bind Paper & Signal variables by name (slug/numeric caps use JetBrains Mono).

Old drawer frames (`162:62`, `163:63`, `164:62`, `165:64`) were deleted and replaced by the frames below.

| Frame # | Step | Node ID (NEW) | Old Node ID (deleted) |
|---|---|---|---|
| 1 | Identity — Name `acme-eu` · Slug `acme-eu` (Mono/M) · Description "EU-resident tenant for acme support." · Region / data residency `eu-west-1 (Frankfurt)` · Cancel / Continue → | `211:62` | `162:62` |
| 2 | Quotas & Budget — Monthly token cap `50M` (Mono) · Cost cap `$2,000` · Provider rate limit `60 req/min` (Mono) · Max agents `25` (Mono) · Back / Continue → | `211:101` | `163:63` |
| 3 | Defaults — Default provider Provider Chip `Claude` (tide pill) · Allowed providers checkboxes (Claude ✓, OpenAI ✓, Gemini ☐, Grok ☐) · Safety defaults tags `Standard moderation` + `PII redaction on` · Back / Continue → | `212:62` | `164:62` |
| 4 | Review & Create — three-section summary (Identity / Quotas & budget / Defaults), Label/S muted CAPS labels + Body/M values, Mono/M for slug/caps; stepper all 4 dots filled `brand/tide` · Back / Create namespace (primary) | `212:110` | `165:64` |

**Entry point:** The `New namespace` button was **moved from the global topbar** into the **content page header** of Admin screen `57:56`, right-aligned next to the "Namespaces" heading — matching how the Agent (`23:3`, btn `25:30`) and MCP (`56:56`, btn `56:284`) journeys place their entry buttons. The content header (`57:279`) was converted to a horizontal auto-layout (`SPACE_BETWEEN`) with a left vertical stack (heading + subtitle) and the `brand/tide` button at right (node `215:92`). The button has been removed from the topbar (old node `160:2` deleted). No regression to other topbar or header content.

### Knowledge source — takeover wizard ✅ (page 268:2)

Reworked 2026-05-25: **converted from side-drawer to full-screen takeover wizard**, matching the Agent, MCP, and Namespace wizard style (no scrim, no parent-screen clone, no drawer). Five old drawer frames (`176:64`, `178:64`, `179:64`, `180:64`, `181:64`) were deleted; band label **"Knowledge source"** (Display/L, `text/ink`, node `182:64`) at y=3600 was preserved.

Worked example: **add `acme-helpcenter` web crawl** (ties to the `kb_lookup` story thread). Five standalone full-screen frames, each hand-built on the **Takeover shell** (`88:26`) pattern: full 1440×1024 on `surface/canvas` (#faf9f5), top bar ("Conductor · Add knowledge source" left in Hanken SemiBold, `Exit ✕` right), centered horizontal 5-step stepper (`① Source ② Configure ③ Test ④ Ingest ⑤ Done`, Step N of 5 — done/active dots+bars bind `brand/tide`, upcoming bind `border/hairline`), centered 720px `surface/card` content column (Shadow/Card, radius 12, 40px padding), sticky footer (ghost Cancel/Back + primary `brand/tide` button). Laid out at x = i×1540, y = 3720. All fills/text bind Paper & Signal variables by name (URLs, chunking config, counts use JetBrains Mono). No bare hex literals.

| Frame # | Step | Old Node ID (deleted) | New Node ID |
|---|---|---|---|
| 1 | Source — 2-col grid of 6 Connector cards: Web crawl (SELECTED, teal border + `brand/tide` icon), S3, Google Drive, Notion, File upload, REST API · Cancel / Continue → | `176:64` | `223:62` |
| 2 | Configure — Start URL `https://help.acme.com` (Mono) · Target collection `acme` (dropdown) · Chunking `recursive · 800 tok · 80 overlap` (Mono) · Embedding model `text-embedding-3-large` (dropdown) · Sync schedule `daily 02:00 UTC` (dropdown) · Back / Continue → | `178:64` | `224:62` |
| 3 | Test & preview — Result chip `success` "Reached help.acme.com — 1,204 pages discovered" · 3 sample doc rows (title + Mono URL) · Back / Start ingestion → | `179:64` | `225:62` |
| 4 | Ingesting — 4 Progress-step rows: Fetch (done, `1,204 pages`) · Chunk (done, `9,841 chunks`) · Embed (active/running, `6,002 / 9,841`, highlighted `surface/sunken` row) · Index (pending) · Status bar "Embedding in progress · 61%" (teal on `brand/tide-wash`) · Cancel only (running state) | `180:64` | `226:62` |
| 5 | Indexed ✓ — All 5 stepper dots filled `brand/tide` · Result chip `success` "Indexed 9,841 chunks · freshness just now" · Stats table: Pages indexed `1,204 pages` · Chunks created `9,841 chunks` · Collection `acme` (Mono) · Freshness `just now` · Done (primary, brand/tide) | `181:64` | `227:62` |

**Entry point:** KB Management screen `68:56` already carries the primary `Add source` button in the content header at node **`68:284`** (`brand/tide` fill, white label, placed next to the "Knowledge Base" page heading — NOT the global topbar). The button **pre-existed** from the original screen build and was not touched (additive check satisfied).

### Credentials — takeover wizard ✅ (page 271:2)

Built 2026-05-25 (fresh, no prior drawer version). **Full-screen takeover wizard** matching the Agent, MCP, Namespace, KB, and Trigger wizard style (no drawers, no scrim, no parent-screen clone). Three standalone full-screen frames, each hand-built on the **Takeover shell** (`88:26`) pattern: full 1440×1024 on `surface/canvas` (#faf9f5), top bar (**"Conductor · Add credential"** left in Hanken SemiBold, `Exit ✕` right, 1px `border/hairline` bottom), centered horizontal 3-step stepper (`① Enter ② Verify ③ Stored` — dots without text labels; done/active dots+bars bind `brand/tide`, upcoming bind `border/hairline`; `Step N of 3` label at right), centered 720px `surface/card` content column (Shadow/Card, radius 12), sticky footer (ghost Cancel/Back + primary `brand/tide` button). Band label **"Credentials"** (Display/L, `text/ink`, node `247:81`) at y=6080. Laid out at x = i×1540, y = 6200. Worked example: **Anthropic production key** (`sk-ant-••••••••` / `sk-ant-…3f9`). All fills/text bind Paper & Signal variables by name (key value and vault URI use JetBrains Mono); no bare hex literals.

| Frame # | Step | Node ID |
|---|---|---|
| 1 | Enter — Type select (`Provider API key` / `OAuth app` / `Secret`) · Provider Claude chip (tide border+wash, orange dot) · API Key `sk-ant-••••••••` (Mono/M) · Label `Anthropic — production` · Namespace scope `acme` · Rotation reminder toggle ON "90 days" · Cancel / **Verify →** · Stepper step 1 active | `247:82` |
| 2 | Verify — Summary row (`Anthropic — production`, Mono `sk-ant-••••••••  ·  Provider API key  ·  Claude`) · **Result chip success** "Key valid · models reachable" (green) · note: error chip shown if key rejected · Back / **Save →** · Stepper step 2 active (1 done) | `248:81` |
| 3 | Stored — Success banner (green top bar) · KEY (MASKED) `sk-ant-…3f9` (Mono/M) · VAULT LOCATION `vault://acme/anthropic` (Mono/M, tide color) · LAST VERIFIED `just now` · STATUS chip **Active** (state/production green) · Stepper all 3 filled · **Done** (primary) | `249:81` |

**Entry point:** Admin screen `57:56` → CREDENTIALS card (`58:83`). Added a small primary `+ Add credential` button (node **`250:56`**, `brand/tide` fill, white label Hanken SemiBold 11px, height 28px, radius 8, absolute-positioned top-right of the CREDENTIALS card). No regression to "New namespace" header or other content.

### Trigger creation — takeover wizard ✅ (page 270:2)

Reworked 2026-05-25: **converted from side-drawer to full-screen takeover wizard**, matching the Agent, MCP, Namespace, and KB wizard style exactly (no scrim, no parent-screen clone, no drawer). Five old drawer frames (`187:65`, `188:64`, `189:64`, `190:64`, `191:64`) were deleted; band label **"Trigger creation"** (`text/ink`, node `187:64`) at y=4840 was preserved.

Worked examples: nightly **`kb-refresh` cron** (Daily `0 2 * * *`) + **`ticket-created` webhook** (`https://api.acme.dev/hooks/tj_9f3a2c`) + **Linear `issue.created` event** alt. Five standalone full-screen frames, each hand-built on the **Takeover shell** (`88:26`) pattern: full 1440×1024 on `surface/canvas`, top bar (**"Conductor · New trigger"** left in Hanken SemiBold, `Exit ✕` right, 1px `border/hairline` bottom), centered horizontal 3-step stepper (`① Type ② Configure ③ Review`, Step N of 3 — done/active dots+bars bind `brand/tide`, upcoming bind `border/hairline`; per-step text labels were **removed in the consistency pass** to match the other five wizards), centered 720px `surface/card` content column (Shadow/Card, radius 12, 40px padding), sticky footer (ghost Back/Cancel + primary `brand/tide` button). Laid out at x = i×1540, y = 4960. Frames 2/3/5 all show step 2 active (three different Configure branches — Cron, Webhook, Event). Frame 4 shows step 3 active with steps 1–2 done. All fills/text bind Paper & Signal variables by name (cron expressions, endpoint URLs, signing secrets, topic names, filter expressions use JetBrains Mono; no bare hex literals).

| Frame # | Step / Branch | Old Node ID (deleted) | New Node ID |
|---|---|---|---|
| 1 | Choose type — 3 Connector card instances: Cron (SELECTED, `brand/tide-wash` bg + tide border), Webhook, Event · subtext "Choose how this trigger fires" · Cancel / Continue → · Stepper step 1 active | `187:65` | `239:62` |
| 2 | Cron builder — Preset chips (Daily selected, tide) · Expression `0 2 * * *` (JetBrains Mono, `surface/card` input) · Preview `Next run: tomorrow 02:00 UTC` (Mono/S) · Timezone `UTC` (dropdown) · Target agent `kb-refresh` + **Version State Badge `Production`** (node `20:23`, green) + `v3` · Input payload `{}` in `surface/sunken` monospace well · Back / Continue → · Stepper step 2 active (done 1) | `188:64` | `241:77` |
| 3 | Webhook endpoint — Generated URL `https://api.acme.dev/hooks/tj_9f3a2c` (Mono/S in `surface/sunken`, copy ⎘ affordance) · Secret `whsec_••••••••` (Mono/M, masked, reveal ○) · Target `triage` (dropdown) · Payload mapping rows Mono/S in `surface/sunken` (`$.issue.id → input.ticket_id`, title, assignee) · Back / Continue → · Stepper step 2 active (done 1) | `189:64` | `242:79` |
| 4 | Review & activate — Summary card (Type Cron, Schedule `0 2 * * *  (Daily at 02:00 UTC)` Mono, Timezone UTC, Target `kb-refresh` bold + **Production badge** + v3, Payload `{}` Mono) · Enabled toggle ON · Retry policy `3× exponential backoff` · divider · Back / **Activate trigger** (primary) · Stepper step 3 active (done 1–2, all dots teal) | `190:64` | `243:79` |
| 5 | Event subscription — Event source `linear` (dropdown) · Topic `issue.created` (Mono/S dropdown) · Filter `team = support` (Mono/S input) · Target `triage` (dropdown) · help text · Back / Continue → · Stepper step 2 active (done 1, event branch) | `191:64` | `244:81` |

**Entry point:** Scheduling screen `67:56` already carries the primary `New trigger` button in the content header at node **`67:284`** (`brand/tide` fill, white label `+ New trigger`, placed next to the "Scheduling & Triggers" page heading — NOT the global topbar). The button **pre-existed** from the original screen build and was not touched (additive check satisfied). The takeover replaces the screen, so there is no parent-screen clone or scrim behind these frames.

## Cross-screen story threads (intentional coherence)
- Open `kb_lookup` breaker (14 Health) → degradation banner (09 Chat) → `Error` source (16 KB).
- `wf_run_8c3a2f` flows 07 Runs → 08 Run Detail → 13 Trace.
- usr_88213's duplicate-charge refund appears in 09 Chat, 18 Compliance (PII redaction), 20 Memory.

## shadcn/ui-in-Figma — UNBLOCKED / SCHEDULED 🟡 (Task 8 follow-on)

Goal: swap the in-place primitives for *real* shadcn components, rethemed to Paper & Signal (procedure in [07](./07-shadcn-in-figma.md)).

- **Library added 2026-05-25 (unblocked):** the Conductor file now subscribes to **"shadcn/ui components with variables & Tailwind classes — Updated January 2026 (Community)"** (`source: team`), libraryKey `lk-1865af9ca182d391387acb06483fc2aed1f8b1676519965c9f3270f6032c7106d8f95b338fc69621d5d8ab737cf95c5e860c8ed39634af2adabf1169caf294ea`. Confirmed present via `get_libraries`.
- **This is a variable-based kit** (Tailwind-class + Figma-variable backed) — so it can be rethemed to Tide/paper/Bricolage **cleanly**, exactly the type the earlier (style-based Pietro Schirano) kit was not. The old `B0cErNvwoThSCicLU3x0Fj` plan is superseded.
- **Plan:** one **global swap pass** across **all 20 screens + 6 journeys** uniformly (kept consistent rather than piecemeal). Sequenced **after** the Lists-pattern workstream so it standardizes the new list components too.
- **Procedure:** `get_libraries` (done) → `search_design_system` scoped by the libraryKey → `importComponentByKeyAsync` → retheme to Paper & Signal variables → swap in-place primitives + creation-flow primitives for real components.

## Open items / backlog
- [x] ~~Publish/subscribe a shadcn kit~~ — DONE 2026-05-25 (variable-based kit subscribed; see shadcn section).
- [x] ~~Setup-journey creation flows~~ — DONE (6 takeover wizards: Agent, MCP, Namespace, KB source, Trigger, Credentials).
- [ ] **Lists pattern** (in progress): filter bar + Blocks⇄Datatable view toggle + dual pagination (blocks = infinite scroll, datatable = numbered) sharing **one paginated datasource signature**, applied across the list screens. **Task L0 DONE 2026-05-25** — reusable constructs built on the Components page (`FilterBar` `280:37`, `ViewToggle` `279:67`, `BlockGrid sentinel` `285:70`, `DataTable + Pager` `286:50`) + the new `Lists pattern` page (`278:2`) for later applied/state frames. Remaining: apply across list screens.
- [ ] **Global shadcn swap** (after Lists): import + retheme → swap in-place primitives across all screens + journeys.
- [ ] **Comprehensive journey docs** (docs only): one page per macro journey covering ALL option-branches/cases + the actions needed to make each path work.
- [ ] Extract remaining in-place composites (App Shell, agent card, workflow node, chat bubble…) into standalone Figma component sets.
- [ ] Optional: standalone **Settings** screen (currently folded into Admin).
- [ ] Optional: **dark mode** (Color collection is structured for a second mode).
- [x] ~~Commit `docs/ui/` to git~~ — docs committed incrementally this session.

## How to resume (cold start)
1. Read this file + [README](./README.md). Figma MCP is connected as Hugo Seabra (Pro).
2. To edit screens: `use_figma` with `fileKey: wgCEx2MmeF3tGDeulGmVqI`; screens live on the **Screens** page (`23:2`) at the node IDs above.
3. **New screen pattern:** clone an existing screen frame (`s = src.clone()`), set `x = 31×1540`+, retarget nav (deactivate `nav Agents`, activate the target `nav X`: fill `brand/tide-wash`, recolor icon vectors to `#0E7C86`, set its text to `brand/tide-text` + Hanken SemiBold), rebuild the `Breadcrumb` frame, remove the old `Content`, build the new body.
4. **Gotchas (from [06](./06-learnings.md)):** use `figma.createAutoLayout()` (not `createFrame`+`layoutMode`, which stays 100×100); append children *before* setting `FILL`/`HUG`; never `resize()` after `FILL`; `setSharedPluginData` unavailable on variables/styles → use name lookups; `use_figma` writes are strictly sequential.
5. Verify visually: `get_screenshot` on the screen node, download via `curl`, view.
