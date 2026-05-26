# Conductor Setup Journeys — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add six creation/setup journeys (Agent, MCP, Namespace, KB source + ingestion, Schedule/Trigger, Credentials) as polished static frames to the existing "Conductor — Paper & Signal" Figma showcase, plus the docs that record them.

**Architecture:** Figma-only (no code). A new `Journeys` page holds the flows as horizontal bands (one row per journey). Agent creation is a full-viewport **takeover** wizard; the other five are **right-drawer steppers** over a clone of their parent screen. Three new reusable component sets (Stepper, Drawer shell, Takeover shell) plus three small primitives are built first. All frames reuse the Paper & Signal variables and fonts; verification is visual via `get_screenshot`.

**Tech Stack:** Figma + Figma MCP (`use_figma`, `get_screenshot`, `get_metadata`); Paper & Signal Figma variables/styles; Bricolage Grotesque / Hanken Grotesk / JetBrains Mono. Git tracks only the `docs/` deltas (the Figma file lives in Figma cloud).

---

## Spec

Design spec: `docs/superpowers/specs/2026-05-25-conductor-setup-journeys-design.md`. Read it before starting.

## Working facts (from `docs/ui/08-build-status.md` and `02-design-system.md`)

- **Figma file:** `fileKey wgCEx2MmeF3tGDeulGmVqI`. Pages: `Cover` (0:1), `Foundations` (15:2), `Components` (15:3), `Screens` (23:2). **New:** add `Journeys`.
- **Parent screen node IDs** (Screens page) to clone for drawer flows:
  - Agent Catalogue #01 = `23:3` (entry point for Agent creation — gets a `New agent` button; Agent flow itself is a takeover, not a clone)
  - Tool & MCP Catalogue #10 = `56:56` (MCP)
  - Admin #11 = `57:56` (Namespace + Credentials)
  - Scheduling #15 = `67:56` (Schedule/Trigger)
  - KB Management #16 = `68:56` (KB source)
- **Existing component sets:** Version State Badge `20:23`, Provider Chip `21:14` (Components page `15:3`).
- **Color variables:** `surface/canvas #FAF9F5`, `surface/card #FFFFFF`, `surface/sunken #F4F2EC`, `border/hairline #E7E4DB`, `text/ink #1C1B18`, `text/muted #6B675E`, `text/faint #9C978C`, `brand/tide #0E7C86`, `brand/tide-bright #13A4B0`, `brand/tide-wash #E6F3F4`, and `state/{draft,staging,canary,production,running,error,paused}`, `provider/*`.
- **Text styles:** Display/XL, Display/L, Heading/H1, Heading/H2, Body/L, Body/M, Body/S, Strong/M, Label/M, Label/S, Mono/M, Mono/S, Mono/Medium.
- **Effect styles:** Shadow/Card, Shadow/Raised, Shadow/Overlay.
- **Fonts:** Bricolage Grotesque (display), Hanken Grotesk (body/UI), JetBrains Mono (machine layer: IDs, cron exprs, URLs, keys).

## Figma gotchas (from `06-learnings.md` §13–14 — apply on every `use_figma` write)

- Use `figma.createAutoLayout()` (hugs both axes). Do NOT `createFrame` then set `layoutMode` — it stays 100×100.
- Append children **before** setting `layoutSizing='FILL'/'HUG'`; `FILL`/`HUG` only apply to children of auto-layout parents.
- Never call `resize()` **after** setting `FILL` (it resets sizing to fixed).
- `setSharedPluginData` is unavailable on Variable/Style objects — use **name-based** variable/style lookups.
- `use_figma` writes are **strictly sequential** per file — never parallelize mutations to this file.

## Layout grid for the `Journeys` page

- Frame size: **1440 × 1024** for all frames (takeover and parent-clone-with-drawer alike).
- **Horizontal** within a band: `x = stateIndex × 1540` (matches the Screens page rhythm).
- **Vertical** per band: `y = bandIndex × 1240`. Bands: 0 Agent (`y=0`), 1 MCP (`y=1240`), 2 Namespace (`y=2480`), 3 KB (`y=3720`), 4 Schedule (`y=4960`), 5 Credentials (`y=6200`).
- Each band gets a label above its first frame (Display/L, `text/ink`): the journey name.

## Verification model (no code/tests)

Each frame is verified by screenshot, not unit test:
- Run: `get_screenshot` on the frame's node ID → download the returned URL via `curl` → view the image.
- Expected: the frame matches the spec below (correct content, Paper & Signal tokens, no 100×100 auto-layout artifacts, mono register on IDs/URLs/keys).

Git commits cover only `docs/` updates. Each journey task ends by appending its node IDs to `docs/ui/08-build-status.md` and committing that delta — that is the "frequent commit" rhythm here.

---

### Task 0: Foundation — Journeys page + reusable components

**Files:**
- Figma: new page `Journeys`; Components page `15:3` (new component sets).
- Modify: `docs/ui/08-build-status.md` (record new page + component node IDs).

- [ ] **Step 1: Open the Figma session**

Invoke the `/figma-use` skill (MANDATORY before any `use_figma` call), then `use_figma` with `fileKey: wgCEx2MmeF3tGDeulGmVqI`. Keep this session for all subsequent tasks. Re-read the gotchas above before each write batch.

- [ ] **Step 2: Create the `Journeys` page**

Create a new page named `Journeys`. Record its node ID.

- [ ] **Step 3: Build the `Stepper` component set (Components page `15:3`)**

Component set name `Stepper`, two variants:
- `Type=Takeover/Horizontal`: a horizontal row of segments `●━━●━━○━━○━━○`; active/done segments `brand/tide`, upcoming `border/hairline`; a `Step N of M` label (Label/S, `text/muted`) at the right. Build with `createAutoLayout()`, horizontal, 12px gap.
- `Type=Drawer/Compact`: inline numbered chips `① ② ③` with the active number filled `brand/tide` on `brand/tide-wash`, others `text/faint`; a one-word label per step (Label/S).

Verify: `get_screenshot` on the component set → both variants render, no 100×100 artifacts.

- [ ] **Step 4: Build the `Drawer shell` component set**

Component set `Drawer shell`, variants `Size=480` and `Size=560`. Structure (auto-layout, vertical): header row (title — Heading/H2, `text/ink`; spacer; `✕` close icon button), `border/hairline` rule, `Drawer/Compact` Stepper slot, scrollable body slot (`surface/card`), sticky footer (auto-layout horizontal, right-aligned: secondary `Back` button + primary action button `brand/tide`). Apply `Shadow/Overlay`. The panel is right-anchored; pair it on screens with a full-frame `#1C1B18` scrim at ~40% opacity behind it.

Verify: `get_screenshot` → drawer renders with header/stepper/body/footer regions.

- [ ] **Step 5: Build the `Takeover shell` component set**

Component set `Takeover shell`, single variant. Structure (auto-layout, vertical, fills 1440×1024 on `surface/canvas`): top bar (auto-layout horizontal: `Conductor · {flow name}` left — Strong/M; spacer; `Exit ✕` ghost button right), `Takeover/Horizontal` Stepper centered below the top bar, a centered ~720px content column slot (`surface/card`, `Shadow/Card`, radius 12), sticky footer (right-aligned `Back` + primary `Continue →`).

Verify: `get_screenshot` → centered column on warm canvas, stepper on top, footer at bottom.

- [ ] **Step 6: Build the three small primitives**

- `Result chip` component set, variants `State={success,error,pending}`: dot + label, using `state/production` (success), `state/error` (error), `state/running` (pending). Label/S.
- `Connector card` component: icon slot + title (Strong/M) + one-line descriptor (Body/S, `text/muted`), `surface/card`, `Shadow/Card`, radius 12; hover/selected state uses `brand/tide-wash` bg + `brand/tide` border.
- `Progress-step row` component set, variants `State={done,active,pending}`: status dot (`state/production`/`state/running`/`text/faint`) + stage label (Body/M) + optional count (Mono/S, `text/muted`).

Verify: `get_screenshot` on each → renders per spec.

- [ ] **Step 7: Record foundation IDs and commit**

In `docs/ui/08-build-status.md`, under the Components section, add rows for `Stepper`, `Drawer shell`, `Takeover shell`, `Result chip`, `Connector card`, `Progress-step row` with their node IDs; add the `Journeys` page to the Pages row. Then:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record Journeys page + creation-flow component sets"
```

---

### Task 1: Agent creation — takeover wizard (8 frames, band y=0)

**Files:**
- Figma: `Journeys` page (8 frames) + `Screens` page `23:3` (add entry button).
- Modify: `docs/ui/08-build-status.md`.

Worked example: create a new top-level chatbot **`acme-concierge`** (router, draft `v0.1`) that routes to the existing `payments` / `technical` / `knowledge` specialists.

Each frame instantiates `Takeover shell` (flow name `New agent`), sets the `Takeover/Horizontal` Stepper to the right step, and fills the content column. Positions: `x = i×1540, y = 0`.

- [ ] **Step 1: Frame 1 — `Type` (x=0)**

Content column: heading "What are you building?" (Display/L). Two `Connector card`s side by side: **Chatbot** (icon, "Conversational agent — routes to specialists or answers directly. Best for support, Q&A.") selected (tide-wash); **Workflow** (icon, "Deterministic multi-step pipeline of LLM/tool/MCP calls. Best for automation."). Footer: `Cancel` + `Continue →`. Stepper step 1 of, chatbot path, 7.

- [ ] **Step 2: Frame 2 — `Identity` (x=1540)**

Fields (stacked, FILL width): Name `acme-concierge` (mono hint of slug below), Namespace select `acme` (prefilled), Description "Front-door concierge that triages and routes customer messages.", Owner `support-platform`, Tags chips `support`, `router`. Stepper step 2.

- [ ] **Step 3: Frame 3 — `Model & Provider` (x=3080)**

Provider Chip select row (`21:14`) with `Claude` selected; Model dropdown `claude-opus-4-7`; Temperature slider `0.3`; Max tokens `4096` (Mono/M); System/router prompt textarea (Body/S, mono content) with a routing-classifier prompt. Stepper step 3.

- [ ] **Step 4: Frame 4 — `Routing & Specialists` (x=4620)**

Strategy segmented control: `LLM classifier` (selected) / `Rule-based`. Confidence threshold slider `0.72`. Fallback specialist select `knowledge`. Multi-select list of specialists to route to, each a row with Provider Chip + Version State Badge: `payments` (Production), `technical` (Production), `knowledge` (Production) — all checked. Helper text cites routing behavior. Stepper step 4.

- [ ] **Step 5: Frame 5 — `Knowledge & Memory` (x=6160)**

KB attach: collection picker showing `acme` Qdrant KB collection checked, with a hit-count stat (Mono/S). Memory tiers: three toggles — `Working` (on), `Episodic` (on, retention `30d`), `Semantic` (on, "customer profile"). Each toggle row has a one-line descriptor (Body/S, `text/muted`). Stepper step 5.

- [ ] **Step 6: Frame 6 — `Safety` (x=7700)**

Moderation level segmented `Off / Standard / Strict` (Standard selected). PII detection/redaction toggle (on) with a `state/production` Result chip "redacts card numbers, emails". Blocked topics tag input (`legal advice`, `competitor pricing`). Refusal-message textarea with sample copy. Stepper step 6.

- [ ] **Step 7: Frame 7 — `Review & Create` (x=9240)**

Two-column summary card of every prior choice (labels Label/S `text/muted`, values Body/M / mono where machine). Right rail: a draft preview chip (Version State Badge `Draft`). Footer primary `Create as draft`; secondary `Test in Playground`. Stepper step 7 (final, all dots filled).

- [ ] **Step 8: Frame 8 — Workflow tail: `Triggers → Open builder` (x=10780)**

Single frame showing the diverging workflow path: heading "Workflow scaffold ready". Trigger picker (cards): `Manual` (selected) / `Cron` / `Webhook` / `Event`. Note "Steps are defined in the Visual Builder." Footer primary `Create draft & open builder` (implies handoff to `35:34`). Stepper shows the shorter workflow path (Type · Identity · Model · Triggers).

- [ ] **Step 9: Add the entry button to Agent Catalogue (`23:3`)**

On screen `23:3`, add a primary `New agent` button (`brand/tide`) into the page header toolbar, right-aligned. Additive only — do not relayout existing content.

- [ ] **Step 10: Verify**

Run: `get_screenshot` on each of the 8 frame node IDs and on `23:3`. Download via `curl`, view. Expected: takeover shell consistent, stepper advances 1→7 (chatbot) and the workflow tail frame reads correctly; `New agent` button present on the catalogue without layout regression.

- [ ] **Step 11: Record IDs and commit**

Append an "Agent creation" sub-table (frame # → node ID) to the new Journeys section of `docs/ui/08-build-status.md`. Then:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record Agent-creation takeover journey frames"
```

---

### Task 2: MCP creation — drawer stepper (5 frames, band y=1240)

**Files:** Figma `Journeys` page (5 frames, each a clone of `56:56` + scrim + `Drawer shell` Size=560); `Screens` `56:56` (add `Add MCP` button); `docs/ui/08-build-status.md`.

Worked example: **add the Linear MCP via OAuth**. Each frame = clone of Tool & MCP Catalogue `56:56`, dimmed with a 40% `#1C1B18` scrim, with the drawer on the right. Positions `x=i×1540, y=1240`.

- [ ] **Step 1: Frame 1 — `Connect` (x=0)**

Drawer title "Add MCP". Stepper `① Connect ② Auth ③ Discover ④ Test ⑤ Register`. Fields: Transport select `stdio / SSE / streamable-http` (`SSE` selected for Linear), Server URL `https://mcp.linear.app/sse` (Mono/M), Env/secrets key-value rows (one: `LINEAR_WORKSPACE` = `acme`), Namespace scope `acme`. Footer `Cancel` / `Connect →`.

- [ ] **Step 2: Frame 2 — `OAuth consent` (x=1540)**

Drawer body shows an OAuth handoff state: provider lockup "Connect to Linear", a `brand/tide` "Authorize" button, and below it a `Result chip` `pending` "Waiting for authorization…". Footer `Back` / `Continue →` (disabled look until connected). Stepper step 2.

- [ ] **Step 3: Frame 3 — `Discover capabilities` (x=3080)**

`Result chip` `success` "Connected as acme". Checklist of discovered capabilities grouped Tools / Resources / Prompts: e.g. Tools `create_issue`, `search_issues`, `add_comment`; Resources `issue://`, `team://`; each a checkbox row (all checked) with a Mono/S identifier. Stepper step 3.

- [ ] **Step 4: Frame 4 — `Test` (x=4620)**

A test-call panel: sample invocation `search_issues({"query":"refund"})` (Mono/M in a `surface/sunken` well), `Result chip` `success` + latency `412ms` (Mono/S). Footer `Back` / `Continue →`. Stepper step 4.

- [ ] **Step 5: Frame 5 — `Register / Done` (x=6160)**

Summary: Name `linear`, Version `1.0`, Scope `acme`, Auth `Connected` (`state/production` chip). The catalogue behind shows a new `linear` MCP card with `Connected` auth status. Footer primary `Register MCP`. Stepper step 5 (complete).

- [ ] **Step 6: Add entry button + verify + commit**

Add a primary `Add MCP` button to `56:56` header. Run `get_screenshot` on the 5 frames + `56:56`; download/view; expect coherent drawer-over-dimmed-catalogue with OAuth and test states. Append the MCP sub-table to `08-build-status.md`:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record MCP-creation drawer journey frames"
```

---

### Task 3: Namespace creation — drawer stepper (4 frames, band y=2480)

**Files:** Figma `Journeys` page (4 frames = clone of `57:56` + scrim + `Drawer shell` Size=480); `Screens` `57:56` (add `New namespace` button); `docs/ui/08-build-status.md`.

Worked example: **`acme-eu`** (EU data residency). Positions `x=i×1540, y=2480`.

- [ ] **Step 1: Frame 1 — `Identity` (x=0)**

Drawer title "New namespace". Stepper `① Identity ② Quotas ③ Defaults ④ Review`. Fields: Name `acme-eu`, Slug `acme-eu` (Mono/M), Description "EU-resident tenant for acme support.", Region/data-residency select `eu-west-1` (Frankfurt). Footer `Cancel` / `Continue →`.

- [ ] **Step 2: Frame 2 — `Quotas & Budget` (x=1540)**

Fields: Monthly token cap `50M` (Mono/M), Cost cap `$2,000` , Provider rate limit `60 req/min`, Max agents `25`. Use `Quota Meter` style if present. Stepper step 2.

- [ ] **Step 3: Frame 3 — `Defaults` (x=3080)**

Default provider Provider Chip `Claude`; Allowed providers multi-select (`Claude`, `OpenAI` checked; `Gemini`, `Grok` unchecked); Safety defaults `Standard moderation` + `PII redaction on`. Stepper step 3.

- [ ] **Step 4: Frame 4 — `Review & Create` (x=4620) + entry + verify + commit**

Summary card of all fields; primary `Create namespace`. Add `New namespace` button to `57:56` header. `get_screenshot` the 4 frames + `57:56`; download/view. Append Namespace sub-table to `08-build-status.md`:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record Namespace-creation drawer journey frames"
```

---

### Task 4: KB source + first ingestion — drawer→inline (5 frames, band y=3720)

**Files:** Figma `Journeys` page (5 frames = clone of `68:56` + scrim + `Drawer shell` Size=560); `Screens` `68:56` (add `Add source` button); `docs/ui/08-build-status.md`.

Worked example: **`acme-helpcenter`** web crawl (ties to the `kb_lookup` breaker thread). Positions `x=i×1540, y=3720`.

- [ ] **Step 1: Frame 1 — `Choose source` (x=0)**

Drawer title "Add knowledge source". Stepper `① Source ② Configure ③ Test ④ Ingest ⑤ Done`. Six `Connector card`s in a 2-col grid: Web crawl (selected), S3, Google Drive, Notion, Upload, API. Footer `Cancel` / `Continue →`.

- [ ] **Step 2: Frame 2 — `Configure` (x=1540)**

Fields: Start URL `https://help.acme.com` (Mono/M), Target collection `acme` (select), Chunking `recursive · 800 tok · 80 overlap` (Mono/S), Embedding model `text-embedding-3-large`, Sync schedule `daily 02:00 UTC`. Stepper step 2.

- [ ] **Step 3: Frame 3 — `Test / preview` (x=3080)**

`Result chip` `success` "Reached help.acme.com — 1,204 pages discovered". Sample-docs preview list (3 rows: title + Mono/S URL). Footer `Back` / `Start ingestion →`. Stepper step 3.

- [ ] **Step 4: Frame 4 — `Ingestion running` (x=4620)**

Four `Progress-step row`s: `Fetch` (done, `1,204 pages`), `Chunk` (done, `9,841 chunks`), `Embed` (active, `6,002 / 9,841`), `Index` (pending). A `state/running` pulse motif on the active row. Stepper step 4. (Designed running state.)

- [ ] **Step 5: Frame 5 — `Indexed ✓` (x=6160) + entry + verify + commit**

`Result chip` `success` "Indexed 9,841 chunks · freshness just now". Source appears `Healthy` in the KB list behind. Add `Add source` button to `68:56` header. `get_screenshot` 5 frames + `68:56`; download/view; expect the ingestion progression to read clearly. Append KB sub-table to `08-build-status.md`:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record KB-source + ingestion drawer journey frames"
```

---

### Task 5: Schedule / Trigger creation — drawer stepper (5 frames, band y=4960)

**Files:** Figma `Journeys` page (5 frames = clone of `67:56` + scrim + `Drawer shell` Size=560); `Screens` `67:56` (add `New trigger` button); `docs/ui/08-build-status.md`.

Worked examples: nightly **`kb-refresh` cron** + **`ticket-created` webhook**. Positions `x=i×1540, y=4960`.

- [ ] **Step 1: Frame 1 — `Choose type` (x=0)**

Drawer title "New trigger". Stepper `① Type ② Configure ③ Configure ④ Review` (two configure states shown across frames 2–3). Three `Connector card`s: Cron (selected), Webhook, Event. Footer `Cancel` / `Continue →`.

- [ ] **Step 2: Frame 2 — `Cron builder` (x=1540)**

Preset chips (`Hourly`, `Daily`, `Weekly`, `Custom` — Daily selected); Cron expression `0 2 * * *` (Mono/M) with a `Next run: tomorrow 02:00 UTC` preview (Mono/S); Timezone `UTC`; Target agent select `kb-refresh` + Version pin `v3 (Production)` (Version State Badge); Input payload `{}` (Mono/S well). Stepper step 2.

- [ ] **Step 3: Frame 3 — `Webhook` (x=3080)**

Generated endpoint `https://api.acme.dev/hooks/tj_9f3a…` (Mono/M, copy affordance) + Signing secret `whsec_••••` (masked, Mono/M); Target agent `triage` ; Payload mapping rows (`$.issue.id → input.ticket_id`). Stepper step 3 (Webhook path).

- [ ] **Step 4: Frame 4 — `Review & Activate` (x=4620)**

Summary; Enabled toggle (on); Retry policy `3× exponential`. Primary `Activate trigger`. Stepper step 4.

- [ ] **Step 5: Frame 5 — `Event subscription` alt (x=6160) + entry + verify + commit**

Event variant: Event source `linear` / Topic `issue.created`, Filter `team = support` (Mono/S), Target agent `triage`. (Shows the third trigger branch.) Add `New trigger` button to `67:56` header. `get_screenshot` 5 frames + `67:56`; download/view. Append Schedule sub-table to `08-build-status.md`:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record Schedule/Trigger drawer journey frames"
```

---

### Task 6: Credentials / API key — drawer stepper (3 frames, band y=6200)

**Files:** Figma `Journeys` page (3 frames = clone of `57:56` + scrim + `Drawer shell` Size=480); `Screens` `57:56` (add `Add credential` button to the credentials/vault area); `docs/ui/08-build-status.md`.

Worked example: **Anthropic production key**. Positions `x=i×1540, y=6200`.

- [ ] **Step 1: Frame 1 — `Enter` (x=0)**

Drawer title "Add credential". Stepper `① Enter ② Verify ③ Stored`. Type select `Provider API key / OAuth app / Secret` (Provider API key selected); Provider Provider Chip `Claude`; Key input `sk-ant-••••••••` (masked, Mono/M); Label `Anthropic — production`; Namespace scope `acme`; Rotation reminder `90 days` (toggle). Footer `Cancel` / `Verify →`.

- [ ] **Step 2: Frame 2 — `Verify` (x=1540)**

`Result chip` `success` "Key valid · models reachable" (and note an `error` variant exists: "Invalid key"). Footer `Back` / `Save →`. Stepper step 2.

- [ ] **Step 3: Frame 3 — `Stored` (x=3080) + entry + verify + commit**

Masked value `sk-ant-…3f9` (Mono/M), Vault location `vault://acme/anthropic`, Last verified `just now`, status `state/production` chip `Active`. Add `Add credential` button to the `57:56` credentials/vault area. `get_screenshot` 3 frames + `57:56`; download/view. Append Credentials sub-table to `08-build-status.md`:

```bash
git add docs/ui/08-build-status.md
git commit -m "docs(ui): record Credentials drawer journey frames"
```

---

### Task 7: Integration pass + documentation

**Files:** Figma `Journeys` page (band labels, consistency); `docs/ui/04-screen-inventory.md`, `docs/ui/03-surface-map.md`, `docs/ui/06-learnings.md`, `docs/ui/08-build-status.md`.

- [ ] **Step 1: Band labels + consistency pass**

Add a Display/L label above the first frame of each band (Agent creation, MCP creation, Namespace creation, KB source + ingestion, Schedule & triggers, Credentials). Verify across all frames: identical drawer width per flow, consistent footer button placement, stepper alignment, mono register applied to every ID/URL/key/cron expression, and Paper & Signal tokens (no stray hexes). Fix drift.

- [ ] **Step 2: `04-screen-inventory.md` — add "Tier 3 — Setup Journeys"**

After the Tier-2 section, add a "Tier 3 — Setup Journeys" section listing the six flows, each with its states and the capability/decision it demonstrates (Agent → AD-19/20/33, v5, 28, 35, AD-30; MCP → AD-11/18, v3; Namespace → AD-26, 16; KB → AD-18, 33; Schedule → 29; Credentials → AD-07/08). Note the tiered takeover/drawer pattern.

- [ ] **Step 3: `03-surface-map.md` — realized note**

Add a short note that the previously "intended" creation surfaces (Add MCP + OAuth, namespace creation, KB connectors, trigger config, credentials, chatbot routing/memory/safety config) are now **realized** as the Tier-3 journeys, and that chatbot configuration — previously runtime-only — now has a design home in the agent-creation wizard.

- [ ] **Step 4: `06-learnings.md` — pattern entry**

Add a learning entry: the takeover-vs-drawer decision (heavy multi-step flow → focused takeover; light flow → drawer over its parent screen), and the new Stepper/Drawer shell/Takeover shell component sets. Note that the agent-creation wizard doubles as the chatbot config/edit surface.

- [ ] **Step 5: `08-build-status.md` — finalize**

Ensure the Journeys page, all six sub-tables (frame # → node ID), the six new component sets, and the entry-point button edits on `23:3 / 56:56 / 57:56 / 67:56 / 68:56` are all recorded. Update the open-items backlog (the creation-journey gap is now closed). Update the snapshot date if changed.

- [ ] **Step 6: Commit the documentation**

```bash
git add docs/ui/04-screen-inventory.md docs/ui/03-surface-map.md docs/ui/06-learnings.md docs/ui/08-build-status.md
git commit -m "docs(ui): document Tier-3 setup journeys across inventory, surface map, learnings, build status"
```

---

## Self-review notes

- **Spec coverage:** all six journeys (spec §C) → Tasks 1–6; reusable components (spec §B) → Task 0; Figma integration/structure (spec §A) → Task 0 + per-task entry buttons; story coherence (spec §D) → worked examples pinned in each task + Task 7 consistency pass; documentation (spec §E) → Task 7; frame budget (spec §F, ~30) → 8+5+4+5+5+3 = 30 frames across Tasks 1–6.
- **Lean cut:** if requested, drop Agent frames 5–6 (fold Knowledge/Memory+Safety into Review), MCP frame 4 (Test), KB frame 3 (Test/preview), Schedule frame 5 (Event alt), Credentials frame 2 (Verify) → ~20 frames. Spine preserved.
- **Naming consistency:** node IDs (`23:3`, `35:34`, `56:56`, `57:56`, `67:56`, `68:56`) and variable/style names are taken verbatim from `08-build-status.md` / `02-design-system.md`.
- **No placeholders:** every frame lists concrete mock content and tokens; verification is `get_screenshot`; commits are doc deltas.
