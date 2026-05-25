# Conductor — Setup Journeys (Creation Flows) — Design Spec

> **Date:** 2026-05-25 · **Track:** Figma-only design vision (no code), extending the existing "Conductor — Paper & Signal" showcase (`fileKey wgCEx2MmeF3tGDeulGmVqI`).
> **Problem:** The 20 built screens are almost all landing / list / detail / dashboard states — they show the *result* of things already configured. The **creation & setup journeys** (the step-by-step flows that bring an agent, MCP, namespace, KB source, schedule, or credential into existence) are missing. Chatbot configuration in particular has **no design surface anywhere** today — it lives only as runtime behavior on the Chat screen (#09).
> **Goal:** Design the "Core setup story" — six creation journeys — as polished static states in Figma, coherent with the existing `acme-support` narrative and Paper & Signal design system, and document them in `docs/ui/`.

---

## Decisions locked (from brainstorming)

| Decision | Choice |
|---|---|
| **Scope** | Core setup story — 6 journeys: Agent, MCP, Namespace, KB source + first ingestion, Schedule/Trigger, Credentials. |
| **Interaction pattern** | Tiered: **focused takeover** for Agent creation (long, multi-step); **right-side drawer stepper** for the other five (lighter). |
| **Agent structure** | **Unified wizard with a diverging tail.** Shared spine (Type → Identity → Model), then diverges. Workflow tail hands off to the existing Visual Builder (#02). Chatbot tail walks Routing & Specialists → Knowledge & Memory → Safety → Review. The wizard *is* the chatbot config surface; later edits re-enter the same stepped surface (no separate chatbot-editor screen). |
| **Fidelity** | Static designed states, mock data, Figma-only — same track as the existing 20 screens. Frame budget ≈ 28–32 (full); a lean ~18–20 cut is acceptable if requested at build time. |

---

## A. Figma integration & structure

- **New page: `Journeys`** in the Conductor file (alongside `Cover` / `Foundations` / `Components` / `Screens`). Keeps the 20-screen product tour clean.
- **Layout:** horizontal **bands — one row per journey**, each band reading left→right through its states. Bands stacked vertically with generous gutters and a band label (Bricolage display) per journey.
- **Takeover frames** (Agent): full-viewport, **no app shell**. Top bar = `Conductor · New agent` (left) + `Exit ✕` (right) + horizontal **Stepper**; centered ~720px content column; sticky `Back / Continue` footer.
- **Drawer frames** (other five): a clone of the **parent screen** sat behind at reduced opacity (dim scrim) + a **480–560px right Drawer**: header (title + ✕), compact stepper, scrollable body, sticky footer (`Back` / primary action). Anchors each flow to its launch context.
- **Entry points:** add a `New …` / `Add …` affordance to each parent screen (Agent Catalogue #01, Tool & MCP #10, Admin #11, KB Management #16, Scheduling #15). Additive edits only — a button in an existing toolbar/header; no layout regressions.
- **Tokens:** reuse Paper & Signal throughout — `state/*` for test/verify/ingest status, `provider/*` chips, `brand/tide*` for primary actions and active stepper, JetBrains Mono for the machine layer (IDs, cron expressions, endpoint URLs, keys), Bricolage (display) / Hanken (body).

## B. Reusable components (Components page, `15:3`)

Three new component sets (they repeat across journeys):

1. **Stepper** — variants: `Takeover/Horizontal` (top bar, segmented `●━━●━━○`) and `Drawer/Compact` (numbered `① ② ③` inline). State per step: done / active / upcoming.
2. **Drawer shell** — right panel container: header (title + ✕), body slot, sticky footer slot; sizes 480 / 560.
3. **Takeover shell** — full-viewport container: top bar (name + Exit), stepper slot, centered column slot, footer slot.

Smaller primitives (build if not already present from in-place screen work):
- **Result chip** — `success` / `error` / `pending` (test connection, verify key, ingestion outcome).
- **Connector / source card** — icon + label + short descriptor (source-type and trigger-type pickers).
- **Progress-step row** — labeled stage with status dot + optional count (ingestion pipeline: fetch → chunk → embed → index).

Field inputs, selects, toggles, and generic cards are reused from the existing in-place screen primitives.

## C. The six journeys (states)

### 1. Agent creation — takeover (~8 frames)
Entry: `New agent` on Agent Catalogue (#01).

Shared spine:
1. **Type** — two large cards: `Chatbot` vs `Workflow` (icon, one-line descriptor, "best for…"). Chatbot card notes its sub-modes (router vs specialist) selected later in the Routing step.
2. **Identity** — name, namespace (`acme` prefilled), description, owner, tags/labels.
3. **Model & Provider** — provider chip select (`Claude` / `OpenAI` / `Gemini` / `Grok`), model dropdown, temperature, max tokens, system prompt (for a router chatbot, the routing/classifier prompt).

Diverge — **Chatbot tail**:
4. **Routing & Specialists** — choose strategy (LLM classifier / rule-based), confidence threshold slider, fallback specialist; multi-select the **existing** specialists (`payments`, `technical`, `knowledge`) to route to. (AD-20, AD-33, v5, deep-dive 13.)
5. **Knowledge & Memory** — attach KB collections (the `acme` Qdrant KB); memory tiers (working / episodic / semantic) toggles + retention windows. (AD-30, deep-dive 21.)
6. **Safety** — moderation level, PII detection/redaction toggle, blocked topics, refusal-message copy. (deep-dives 28, 35.)
7. **Review & Create** — full config summary; primary `Create as draft` → lands in Catalogue with a `Draft` badge (and a `Test in Playground` secondary CTA).

Diverge — **Workflow tail**:
8. **Triggers (optional) → Open in Visual Builder** — manual / cron / webhook / event picker (links conceptually to journey 5), then a `Create draft & open builder` handoff into existing screen #02. Single frame.

**Worked example:** create a new chatbot that slots into `acme-support` — a top-level **`acme-concierge`** router (draft `v0.1`) that routes to the three existing specialists. This exercises the full Routing & Specialists step without contradicting the already-deployed `acme-router`. (Final identity may be adjusted at build time; this is the canonical choice.)

### 2. MCP creation — drawer (~5 frames)
Entry: `Add MCP` on Tool & MCP Catalogue (#10). Worked example: **Linear MCP via OAuth** (Linear is already in the story).
1. **Connect** — transport (`stdio` / `SSE` / `streamable-http`), command or URL, env vars / secrets, namespace scope.
2. **Authenticate (OAuth)** — `Connect to Linear` → consent handoff state → `Connected ✓`.
3. **Discover capabilities** — discovered tools / resources / prompts as a checklist (choose what to expose).
4. **Test** — run a sample call → `success` result chip + latency (machine-layer mono).
5. **Register / Done** — name, version, scope confirmation → returns to catalogue with the new MCP card showing `Connected` auth status.

### 3. Namespace creation — drawer (~4 frames)
Entry: `New namespace` on Admin (#11). Worked example: **`acme-eu`** (EU data residency — ties to Compliance #18).
1. **Identity** — name, slug, description, region / data-residency.
2. **Quotas & Budget** — token/cost caps, provider rate limits, max agents.
3. **Defaults** — default provider, allowed providers, safety defaults.
4. **Review & Create.**

### 4. KB source + first ingestion — drawer → inline (~5 frames)
Entry: `Add source` on KB Management (#16). Worked example: **`acme-helpcenter`** web crawl (ties to the `kb_lookup` breaker thread on #14/#16).
1. **Choose source** — connector cards: Web crawl / S3 / Google Drive / Notion / Upload / API.
2. **Configure** — creds / URL, target collection, chunking + embedding config, sync schedule.
3. **Test / preview** — connection test + sample documents preview.
4. **Ingestion running** — pipeline progress as progress-step rows (fetch → chunk → embed → index) with counts — a designed running state.
5. **Indexed ✓** — `N docs indexed`, freshness timestamp; source appears `Healthy` in the KB list.

### 5. Schedule / Trigger creation — drawer (~5 frames)
Entry: `New trigger` on Scheduling (#15). Worked examples: nightly **`kb-refresh` cron** + a **`ticket-created` webhook** (Linear).
1. **Choose type** — trigger-type cards: `Cron` / `Webhook` / `Event`.
2. **Cron builder** — presets + cron expression (mono) + next-run preview + timezone; target agent + version pin; input payload.
3. **Webhook** — generated endpoint URL + signing secret (mono), target agent, payload mapping. *(Event variant — source/topic + filter — may be shown as an alternate state of this frame if doing the full cut.)*
4. **Review & Activate** — enabled toggle, retry/delivery policy.

### 6. Credentials / API key — drawer (~3 frames)
Entry: `Add credential` on Admin (#11) credentials/vault area. Worked example: **Anthropic production key**.
1. **Enter** — type (Provider API key / OAuth app / Secret), masked key input, label, namespace scope, optional rotation reminder.
2. **Verify** — test → `valid ✓` / `invalid ✗` result states.
3. **Stored** — masked value, vault location, last-verified timestamp.

## D. Story coherence

All journeys extend the single `acme-support` narrative — no orphan data: `acme-concierge` chatbot, Linear MCP (OAuth), `acme-eu` namespace, `acme-helpcenter` web-crawl source, `kb-refresh` cron + `ticket-created` webhook, Anthropic production key. Each ties to an existing screen (Compliance #18, Health/KB #14/#16, Tool catalogue #10, Admin #11), reinforcing the cross-screen threads already established.

## E. Documentation updates (delivered with the build)

- `04-screen-inventory.md` — add **"Tier 3 — Setup Journeys"** section listing the 6 flows + their states.
- `08-build-status.md` — add the `Journeys` page, the new component sets, and frame node IDs as built; update the open-items backlog.
- `03-surface-map.md` — note that the previously "intended" creation surfaces (Add MCP + OAuth, namespace creation, KB connectors, trigger config, etc.) are now realized as journeys.
- `06-learnings.md` — short entry on the takeover-vs-drawer creation pattern and the Stepper/Drawer/Takeover component sets.

## F. Frame budget

| Journey | Frames |
|---|---|
| Agent creation | ~8 |
| MCP creation | ~5 |
| Namespace creation | ~4 |
| KB source + ingestion | ~5 |
| Schedule / Trigger | ~5 |
| Credentials | ~3 |
| **Total** | **~30** |

A **lean cut (~18–20)** is acceptable if requested: keep each journey's spine (Type/Identity/Model/Review for Agent; Connect/OAuth/Done for MCP; Identity/Review for Namespace; Choose/Ingest/Done for KB; Choose/Cron/Review for triggers; Enter/Stored for credentials) and drop secondary states.

## Out of scope

- Code implementation (Figma-only track, per AD-25 / docs/ui README).
- Dark mode for the new frames (inherits the existing single-mode decision).
- Eval-suite authoring, promotion-gate config, marketplace import, tool registration (these were "Full breadth" options, deferred).
- Wiring entry-point buttons to real navigation/interaction (static states only).
