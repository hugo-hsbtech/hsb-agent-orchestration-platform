# 08 — Build Status & Handoff

> **Snapshot: 2026-05-25.** The single source of truth for *what actually exists right now*, with the exact Figma IDs needed to resume. Track is **Figma-only, no code** (see [README](./README.md), [05](./05-build-workflow.md)). The plan docs (01–05) describe intent; this file records reality.

## Figma file

| | |
|---|---|
| **File** | "Conductor — Paper & Signal" |
| **fileKey** | `wgCEx2MmeF3tGDeulGmVqI` |
| **URL** | https://www.figma.com/design/wgCEx2MmeF3tGDeulGmVqI |
| **Pages** | `Cover` (0:1) · `Foundations` (15:2) · `Components` (15:3) · `Screens` (23:2) · `Journeys` (82:2) |

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

## Journeys — IN PROGRESS (Journeys page 82:2)

Multi-step takeover/drawer flows that compose the reusable creation-flow primitives (Stepper, Takeover shell, Connector card, Result chip) over the same **acme** customer-support story. Laid out left→right at x = index × 1540, each frame 1440×1024. Band label **"Agent creation"** (Display/L, `text/ink`) sits above frame 1.

### Agent creation — takeover wizard ✅

Worked example: create top-level **chatbot `acme-concierge`** (router, draft `v0.1`) routing to the existing `payments` / `technical` / `knowledge` specialists. The 7-step chatbot spine plus one diverging workflow-tail frame. Each frame is a (detached) instance of the **Takeover shell** (`88:26`), flow name **New agent**, with its Takeover/Horizontal Stepper rebuilt to the right step count/position.

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

### MCP creation — drawer flow ✅

Worked example: **add the Linear MCP via OAuth**. Five standalone snapshots, each a **clone of the Tool & MCP Catalogue screen (`56:56`)** dimmed by a full-frame `text/ink` scrim at 40% opacity, with a right-side **Drawer shell** (Size=560, instanced from `87:14` then detached) anchored full-height. Band label **"MCP creation"** (Display/L, `text/ink`, node `128:56`) sits above frame 1. Laid out at x = i×1540, y = 1240. Drawer title across all frames: "Add MCP". Each drawer's stepper slot is rebuilt as a 5-step compact stepper (`① Connect ② Auth ③ Discover ④ Test ⑤ Register`) advanced to the right step. Drawer applies Shadow/Overlay; all fills/text bind Paper & Signal variables/styles by name (machine register — URLs, env keys, tool IDs, latency, name/version — uses JetBrains Mono).

| Frame | Step | Node ID |
|---|---|---|
| 1 | Connect — Transport (SSE selected) · Server URL `https://mcp.linear.app/sse` · Env `LINEAR_WORKSPACE=acme` · Namespace `acme` · Cancel/Connect → | `128:57` |
| 2 | Auth — OAuth handoff "Connect to Linear" · brand/tide Authorize · Result chip `pending` "Waiting for authorization…" · Back/Continue (disabled) | `142:56` |
| 3 | Discover — Result chip `success` "Connected as acme" · checklists Tools (`create_issue`,`search_issues`,`add_comment`) / Resources (`issue://`,`team://`) / Prompts (`triage_issue`), all checked | `143:68` |
| 4 | Test — sample call `search_issues({"query":"refund"})` in a `surface/sunken` well · Result chip `success` + latency `412ms` | `144:80` |
| 5 | Register / Done — summary Name `linear` · Version `1.0` · Scope `acme` · Auth `state/production` Connected chip · primary Register MCP | `145:92` |

**Entry point:** Tool & MCP Catalogue `56:56` already carries the primary `Add MCP` button in its content header (`brand/tide` fill at node **`56:284`**, label `56:288`) → launches this drawer flow. The button **pre-existed** from the screen build; it was *not* duplicated (additive check satisfied). The dimmed catalogue behind each frame keeps its existing cards (the Linear MCP card `56:369` is already present), so no new card is injected.

### Namespace creation — drawer flow ✅

Worked example: **create namespace `acme-eu`** (EU data residency). Four standalone snapshots, each a **clone of the Admin screen (`57:56`)** dimmed by a full-frame `text/ink` scrim at 40% opacity, with a right-side **Drawer** (560 wide, 1024 tall) anchored full-height at x=880. Band label **"Namespace creation"** (Display/L, `text/ink`, node `163:62`) sits above frame 1. Laid out at x = i×1540, y = 2480. Drawer title across all frames: "New namespace". Each drawer has a 4-step compact stepper (`① Identity ② Quotas ③ Defaults ④ Review`) advanced to the right step. Drawer applies `DROP_SHADOW` overlay effect; all fills/text bind Paper & Signal variables by name (slug/numeric caps use JetBrains Mono).

| Frame # | Step | Node ID |
|---|---|---|
| 1 | Identity — Name `acme-eu` · Slug `acme-eu` (Mono/M) · Description "EU-resident tenant for acme support." · Region `eu-west-1 (Frankfurt)` · Cancel / Continue → | `162:62` |
| 2 | Quotas & Budget — Monthly token cap `50M` (Mono) · Cost cap `$2,000` · Provider rate limit `60 req/min` (Mono) · Max agents `25` (Mono) · Back / Continue → | `163:63` |
| 3 | Defaults — Default provider Chip `Claude` · Allowed providers multi-select (Claude ✓, OpenAI ✓, Gemini ☐, Grok ☐) · Safety defaults `Standard moderation` + `PII redaction on` · Back / Continue → | `164:62` |
| 4 | Review & Create — NAMESPACE SUMMARY card with all 11 fields (label Label/S `text/muted`, value Body/M right-aligned; slug/caps/numeric in Mono) · Back / Create namespace | `165:64` |

**Entry point:** Admin screen `57:56` received an additive primary `New namespace` button (`brand/tide` fill, white label, radius 8, ~36px height) inserted between Search and Avatar in the Topbar, at node **`160:2`**. The button was not present before this task; no regression to existing topbar elements.

### Knowledge source — drawer flow ✅

Worked example: **add `acme-helpcenter` web crawl** (ties to the `kb_lookup` story thread). Five standalone snapshots, each a **clone of the KB Management screen (`68:56`)** dimmed by a full-frame `text/ink` scrim at 40% opacity, with a right-side **Drawer** (560 wide, 1024 tall) anchored full-height at x=880. Band label **"Knowledge source"** (Display/L, `text/ink`, node `182:64`) sits above frame 1 at y=3600. Laid out at x = i×1540, y = 3720. Drawer title across all frames: "Add knowledge source". Each drawer has a hand-built 5-step compact stepper (`① Source ② Configure ③ Test ④ Ingest ⑤ Done`) advanced to the right step. All stepper chip colors and connector bars bind Paper & Signal variables by name (`brand/tide` for done/active chips and bars, `border/hairline` for upcoming — no bare hex literals). Drawer applies `DROP_SHADOW` overlay effect; fills/text bind variables by name (URLs, chunking config, counts use JetBrains Mono).

| Frame # | Step | Node ID |
|---|---|---|
| 1 | Choose source — 2-col grid of 6 Connector cards: Web crawl (SELECTED), S3, Google Drive, Notion, Upload, API · Cancel / Continue → | `176:64` |
| 2 | Configure — Fields: Start URL `https://help.acme.com` (Mono/M) · Target collection `acme` (select) · Chunking `recursive · 800 tok · 80 overlap` (Mono/S) · Embedding model `text-embedding-3-large` (select) · Sync schedule `daily 02:00 UTC` (select) · Back / Continue → | `178:64` |
| 3 | Test / preview — Result chip `success` "Reached help.acme.com — 1,204 pages discovered" · 3 sample doc rows (title + Mono/S URL) · Back / Start ingestion → | `179:64` |
| 4 | Ingestion running — 4 Progress-step rows: Fetch (done, `1,204 pages`) · Chunk (done, `9,841 chunks`) · Embed (active `state/running`, `6,002 / 9,841`) · Index (pending) · Running status chip "Embedding in progress · 61% complete" · Cancel | `180:64` |
| 5 | Indexed ✓ — Success icon · "Indexed ✓" heading · Result chip `success` "Indexed 9,841 chunks · freshness just now" · Stats: 1,204 pages · 9,841 chunks · acme collection · Done (primary) | `181:64` |

**Entry point:** KB Management screen `68:56` already carries the primary `Add source` button in the content header at node **`68:284`** (`brand/tide` fill, white label, placed next to the "Knowledge Base" page heading — NOT the global topbar). The button **pre-existed** from the original screen build; it was not duplicated (additive check satisfied).

## Cross-screen story threads (intentional coherence)
- Open `kb_lookup` breaker (14 Health) → degradation banner (09 Chat) → `Error` source (16 KB).
- `wf_run_8c3a2f` flows 07 Runs → 08 Run Detail → 13 Trace.
- usr_88213's duplicate-charge refund appears in 09 Chat, 18 Compliance (PII redaction), 20 Memory.

## shadcn/ui-in-Figma — IN PROGRESS / BLOCKED ⛔

Goal: import *real* shadcn components into Figma and retheme to Paper & Signal (full procedure in [07](./07-shadcn-in-figma.md)).

- **Kit chosen:** Pietro Schirano "shadcn/ui Design System (Community)", duplicated by the user → **fileKey `B0cErNvwoThSCicLU3x0Fj`**.
- **Blocker:** not yet **published as a team library** — `get_libraries` on Conductor shows it absent and `libraries_available_to_add` empty, so `importComponentByKeyAsync` can't reach it. The kit file must be in a **team project** (HSB) **and** explicitly **Published** (moving ≠ publishing).
- **Caveat (important):** this kit is **style-based — 0 Figma variables** (uses Inter text styles + color styles + 877 icons). So even after publishing, a clean Tide/paper/Bricolage retheme is hard (per-component overrides). A **variable-based** kit (e.g. Bergmann "shadcn/ui components", `figma.com/community/file/1342715840824755935`) would retheme cleanly — recommended if fidelity-to-theme matters.
- **Next action (user):** publish the kit (or a variable-based alternative) → paste the link → agent runs `get_libraries` → add → `search_design_system` → `importComponentByKeyAsync` → retheme → swap the in-place primitives for real components.

## Open items / backlog
- [ ] Publish a shadcn kit → import + retheme → swap primitives (blocked on user).
- [ ] Extract in-place composites (App Shell, agent card, workflow node, chat bubble…) into standalone Figma component sets.
- [ ] Optional: standalone **Settings** screen (currently folded into Admin).
- [ ] Optional: **dark mode** (Color collection is structured for a second mode).
- [ ] Commit `docs/ui/` to git (not yet committed this session).

## How to resume (cold start)
1. Read this file + [README](./README.md). Figma MCP is connected as Hugo Seabra (Pro).
2. To edit screens: `use_figma` with `fileKey: wgCEx2MmeF3tGDeulGmVqI`; screens live on the **Screens** page (`23:2`) at the node IDs above.
3. **New screen pattern:** clone an existing screen frame (`s = src.clone()`), set `x = 31×1540`+, retarget nav (deactivate `nav Agents`, activate the target `nav X`: fill `brand/tide-wash`, recolor icon vectors to `#0E7C86`, set its text to `brand/tide-text` + Hanken SemiBold), rebuild the `Breadcrumb` frame, remove the old `Content`, build the new body.
4. **Gotchas (from [06](./06-learnings.md)):** use `figma.createAutoLayout()` (not `createFrame`+`layoutMode`, which stays 100×100); append children *before* setting `FILL`/`HUG`; never `resize()` after `FILL`; `setSharedPluginData` unavailable on variables/styles → use name lookups; `use_figma` writes are strictly sequential.
5. Verify visually: `get_screenshot` on the screen node, download via `curl`, view.
