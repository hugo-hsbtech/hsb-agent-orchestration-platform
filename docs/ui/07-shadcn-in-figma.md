# 07 — Getting shadcn/ui components into Figma (no code)

> Decision for this track: **Figma only, no local code.** We want to assemble screens from *real* shadcn/ui components inside Figma (faithful, not hand-rolled), themed to [Paper & Signal](./02-design-system.md). This note records exactly what's required to make that possible, because it needs a one-time manual step in the Figma app.

## The constraint

The Figma MCP can only use libraries **already accessible to the file**. Running `get_libraries` on the Conductor file (`wgCEx2MmeF3tGDeulGmVqI`) returns:
- **Attached/available:** Material 3 Design Kit, Simple Design System (SDS), iOS/iPadOS/macOS/watchOS/visionOS kits.
- **`libraries_available_to_add`: empty** — the MCP cannot pull an arbitrary Community file (including any shadcn/ui kit) on its own.

So shadcn/ui is **not reachable** until a shadcn Figma library exists in the user's team and is published.

## What's needed — one-time setup (done by the user in the Figma app)

1. **Obtain a shadcn/ui Figma kit.** shadcn's own list lives at **https://ui.shadcn.com/docs/figma** — there is **no official kit**; all are community-made. Prefer a **variables-based** kit (Figma variables for colors/radius/typography) — that is what makes retheming to Paper & Signal clean.
   - **Free Community files (recommended):**
     - **"shadcn/ui components"** by Sitsiilia Bergmann — well-structured, regularly maintained — https://www.figma.com/community/file/1342715840824755935 *(first choice)*
     - **"shadcn/ui design system"** by Pietro Schirano — crafted to match the code — https://www.figma.com/community/file/1203061493325953101 *(fallback)*
   - **Paid** alternatives (more complete, optional): shadcndesign.com · shadcnstudio.com/figma · shadcn.obra.studio · shadcnblocks.com
   - **Duplicate** the chosen file into your team ("Open in Figma" / Duplicate).
2. **Publish it as a team library.**
   - Move the duplicated file into your team (Pro plan → library publishing is available).
   - Assets panel → **Publish library**.
3. **Hand back to the agent.** Once published, it becomes addable to the Conductor file. Then the agent can:
   - `get_libraries` → confirm it now appears in `libraries_available_to_add`, and add it.
   - `search_design_system` (scoped to its `libraryKey`) → list its components.
   - `importComponentByKeyAsync` → drop its components onto the screens.

### Fastest hand-off
If you've duplicated a kit, just paste its **file link** here. The agent can read its components directly (`get_design_context` / `search_design_system`) and import by key.

## Theming the kit to Paper & Signal

shadcn/ui is **variable-driven**, which is the whole reason it retheme's cleanly. The kit will expose variables like:
`--background, --foreground, --card, --primary, --secondary, --muted, --accent, --destructive, --border, --input, --ring, --radius, --font-sans, --font-mono`.

Reskinning to Paper & Signal = repointing those variables at our tokens:

| shadcn variable | Paper & Signal value |
|---|---|
| `--background` / `--card` | `surface/canvas` / `surface/card` |
| `--foreground` | `text/ink` |
| `--muted-foreground` | `text/muted` |
| `--primary` | `brand/tide` |
| `--destructive` | `state/error` |
| `--border` / `--input` | `border/hairline` |
| `--ring` | `brand/tide-bright` |
| `--radius` | `radius/md` (8) |
| `--font-sans` | Hanken Grotesk |
| `--font-mono` | JetBrains Mono |
| (display) | Bricolage Grotesque |

Two ways to apply: (a) overwrite the kit collection's variable values, or (b) add a **"Paper & Signal" mode** to the kit's color collection and switch frames to it. The platform's **semantic state palette** (draft/staging/canary/production/running/error/paused) and **provider chips** stay as our own tokens — shadcn has no equivalent.

## Component coverage (what we'd map)

The shadcn primitives we need (all standard in any kit) map to the [02-design-system.md](./02-design-system.md) list: Button, Input, Textarea, Select, Combobox/Command, Checkbox, Switch, Badge, Card, Table, Tabs, Dialog, Sheet, Popover, Tooltip, Dropdown, Avatar, Breadcrumb, Toast, Skeleton, Progress, Separator, Sidebar, Chart. Our **custom composites** (Workflow Node, Promotion Lane, Step Timeline row, Chat bubble + routing badge, Background-task card, Version-State Badge, Provider Chip, Scorecard cell, etc.) are not in shadcn and stay bespoke — built *from* shadcn primitives + Paper & Signal tokens.

## Status / next action

- ⛔ Blocked on the one-time setup above (user action in Figma app).
- ✅ Ready on the agent side: import + retheme + swap the hand-built primitives (Badge, Provider Chip, Tabs/segmented — already built to shadcn spec) for the real kit components, then continue the screens.
- 🔗 Alternative if no kit is brought in: keep rebuilding shadcn primitives to spec in-file (already done for Badge / Provider Chip / Tabs) — fully in-Figma and on-theme, lower fidelity to "real shadcn".

> Cross-ref: [06-learnings.md](./06-learnings.md) §11 (shadcn not importable; SDS is the only code-backed kit currently attached).

## shadcn pilot findings (2026-05-25)

The one-time setup above has happened: the Conductor file (`wgCEx2MmeF3tGDeulGmVqI`) now subscribes to **"shadcn/ui components with variables & Tailwind classes — Updated January 2026 (Community)"** (`libraryKey lk-1865af9c…f294ea`, source `team`). This pilot tested swapping the Agent Catalogue's hand-built primitives for *real* imported shadcn components, rethemed to Paper & Signal. Work was done on an isolated **clone** — the canonical `23:3` was verified untouched after.

**Status: DONE_WITH_CONCERNS.** Import works and the retheme is faithful at screen scale, but the kit's published component keys point at *example/documentation frames*, not atomic primitives, and the kit's variables are **remote/read-only** — so a clean variable remap is impossible in-file; every swap is a per-instance override.

### Import mechanics — WORKS
- `importComponentByKeyAsync` / `importComponentSetByKeyAsync` succeed from the subscribed kit. No auth or availability errors. `createInstance()` on imported variants works.
- Kit component keys imported in the pilot: Button set `3d180da66113aba935827ccafb0cb5245aa3ea7b`; Button (example) `32f61cc2142d340523b7146f658e83c5a3450ca3`; Badge set `6dd09bda467636231718db7abae5b752a76082af`; Input `e595971f3dde8ccc2b78eb90ac7e60de9937e368`; Card (example) `969665763ffec8b06e08524bb3a5537e20d21f83`; Tabs (example) `c1b03efc4dc493d1e27472b5a3d60e4b3f6200e8`; Select set `d9b648b7f45f9abda1b72f0b6450c9e0cc32d986`.

### Retheme mechanism — per-instance override ONLY
The kit is variable-based, but its variables live in **remote library collections** (`mode` with only `light mode`/`dark mode`; plus `tw/font`, `tw/padding`, `tw/border-width`, `tw/stroke-width`). In the consuming file these are read-only:
- (a) **Overwrite the kit's variable values** — BLOCKED. `setValueForMode` on a kit variable throws *"Cannot write to internal and read-only node. Are you trying to modify a remote style or component?"*
- (b) **Alias the kit's variables to ours** — BLOCKED for the same reason (can't rebind a remote variable in this file).
- (c) **Per-instance overrides** — FEASIBLE and what the pilot used. After placing an instance, re-bind each paint to our P&S variables with `figma.variables.setBoundVariableForPaint(paint, 'color', psVar)` (e.g. button bg → `brand/tide`, label → `text/on-brand`, badge → `state/*` + `state/*-wash`, input → `surface/card`/`border/hairline`/`text/faint`). Token-driven, not hardcoded hex — good.
- (d) **Edit + republish the source kit file** — the only route to a true variable remap (add a "Paper & Signal" mode, or repoint `mode` values, in the kit's own file). Out of scope for in-file work; this is the clean rollout path if a full swap is pursued.

### The real blocker — published keys are example frames, not atoms
Every "primitive" except Input turned out to be a **multi-element showcase frame**:
- **Button** set variants are *rows of 3 buttons* (default/outline/ghost, ~267px, 3 text nodes). The standalone "Button" component key is an *Accept ⏎ / Cancel Esc* button-group example. The usable atomic button is a nested `FRAME "Button"` inside — reachable only by `createInstance()` → `detachInstance()` → extract the inner frame.
- **Card** key is a 368×394 form composite (header + labelled inputs + button rows) — not a container card. Not usable as the AgentCard shell without rebuilding.
- **Tabs** key is a 398×406 tab-strip-plus-demo-card; the inner `FRAME "Tabs" 160×36` is a clean atomic segmented control (extractable the same detach-and-lift way).
- **Input** `e595971f…` is genuinely atomic (231×36, placeholder text + lucide search icon) — the one clean drop-in.
- **Badge** set is a proper variant set (`Type` = default/secondary/destructive/outline/…number/icon) — atomic and usable via `createInstance()` on a variant; needs hug-fix (kit padding wraps long labels) + state-color override.

### What was swapped on the clone (all rethemed to P&S)
- **New agent button** → atomic shadcn Button (extracted from example), bg `brand/tide`, label `text/on-brand`. Clean.
- **Topbar search** → shadcn Input, `surface/card` + `border/hairline` + faint placeholder. Clean (icon sits right, vs left in the original — minor).
- **4 version-state badges** (3× Production, 1× Canary) → shadcn Badge `secondary`, fill `state/*-wash`, text `state/*`, hugged to single line.
- **ViewToggle (Blocks/Table)** → atomic shadcn Tabs strip (extracted), sunken track + white active pill + ink/muted labels.
- **Agent card shells** — NOT swapped (kit Card is a form composite; a faithful swap would mean detach + rebuild, no fidelity gain).

### Fidelity — good at screen scale
Side-by-side, the rethemed shadcn components are visually indistinguishable from the bespoke ones (warm paper, Tide accent, correct state colors). Two caveats: (1) the kit's text uses its own `family/sans`, **not Hanken Grotesk** — a font override per text node is required for type fidelity (the pilot set characters/colors but the family stays the kit's until explicitly overridden); (2) kit Badge padding is looser than our spec and needs a hug-fix.

### Effort to roll out (~26 screens + 6 journeys + Lists)
**Per-component override grind, not a clean remap.** Each instance needs: detach-and-extract (for Button/Card/Tabs/anything composite) + N paint re-binds + font-family override per text node + hug/size fixes. Estimate ~5–15 min per distinct component *type* to script a reusable helper, then bulk-apply — but composites (Card especially) approach the cost of just keeping our bespoke build. Realistic: a few days to swap the genuinely-atomic primitives (Input, Badge, the extracted Button/Tabs atoms) across all screens; the composites are not worth swapping.

### Recommendation — PARTIAL swap
- **Worth swapping:** Input, Badge (with hug-fix), and the *extracted* atomic Button and Tabs/segmented control. These are real shadcn, retheme cleanly, and are faithful.
- **Not worth swapping:** Card and any composite — the kit ships them as documentation examples, so the swap is a detach-and-rebuild with zero fidelity gain over our bespoke components. Our signature composites (Agent Card, Version-State Badge semantics, Provider Chip, Workflow Node, etc.) stay bespoke regardless.
- **If a full real-shadcn mandate is required:** do it the (d) way — fork the kit file, add a Paper & Signal variable mode there, republish — so instances retheme by *mode switch* instead of thousands of per-instance overrides. That is the only path that scales across 26 screens without an override grind.

### Pilot artifacts
- Pilot page: **`shadcn pilot`** = node `364:2`.
- Clone of Agent Catalogue: `364:3` (original `23:3` confirmed untouched).
- Swapped instance IDs: New agent button `377:73`; topbar Input `380:67`; badges `383:111`/`383:114`/`383:117`/`383:120`; ViewToggle Tabs `386:280`.

## P1 — rethemed shadcn primitives (2026-05-26)

P1 of the **partial swap**: formalized the 4 worth-swapping atoms into **canonical local components** on the Components page (`15:3`), in a new **"shadcn × Paper & Signal"** band (below the existing component shelf, starting y≈3600). Each wraps the pilot's real-shadcn atom (cloned from the pilot page `364:2`, then detached/componentized) and is fully rethemed to Paper & Signal. The bespoke Version State Badge (`20:23`) and all canonical Screens/journey frames were left untouched (P2 owns those).

### Component node IDs
| Component | Node ID | Notes |
|---|---|---|
| **Button (shadcn·P&S)** | set `392:77` | variants: `Variant=primary` `391:79` · `Variant=secondary` `392:74` |
| **Input (shadcn·P&S)** | `393:94` | single component, generic text field (search icon removed) |
| **Badge (shadcn·P&S)** | set `396:74` | 7 variants — see below |
| **Segmented (shadcn·P&S)** | `398:81` | 2-segment (Blocks/Table) from the kit Tabs atom |

Badge variant IDs: `State=draft` `395:74` · `State=staging` `395:78` · `State=canary` `395:82` · `State=production` `394:83` · `State=running` `395:86` · `State=error` `395:90` · `State=paused` `395:94`.

### Retheme method — same per-instance approach the pilot proved, now made canonical
Confirmed again: `importComponentByKeyAsync` works, but the kit's variables stay **remote/read-only**, so retheming is **per-instance paint re-binding** to our P&S variables (`figma.variables.setBoundVariableForPaint`), **plus** the two fixes the pilot flagged:
1. **Paint re-bind** — every fill/stroke/text fill bound by name to a P&S variable (`brand/tide`, `text/on-brand`, `text/ink`, `text/muted`, `text/faint`, `surface/card`, `surface/sunken`, `border/hairline`, `state/*` + `state/*-wash`, `radius/md`). No bare hex, no leftover kit colors.
2. **Font override** — every text node forced to **Hanken Grotesk** (SemiBold for button/badge/segment labels, Regular for the input placeholder); band title uses **Bricolage Grotesque SemiBold** (display register). The pilot's leftover **Inter** is gone.
3. **Hug / sizing fix** — Button height normalized to 36 + radius 8 (pilot was 32); Input 280×36 radius 8; Segmented track radius 8 (kit shipped 10); Badge built to hug (pad `[4,10,4,8]`, gap 6, radius full) with a **dot ellipse** added so it mirrors the bespoke Version State Badge (the kit Badge had no dot).

Rather than re-import-and-extract from keys (fragile detach-and-lift), P1 **cloned the pilot's already-extracted atoms** as the starting point (permitted reuse), then `detachInstance()` / `createComponentFromNode()` to turn them into clean **local** COMPONENT/COMPONENT_SET nodes — so they no longer point at the remote kit and can't drift when the kit updates.

### Theme-correctness — verified clean
Screenshotted each component (and the whole band). Confirmed: warm-paper surfaces, Tide accent on the primary button + active segment, Hanken type throughout, correct per-state badge colors. A programmatic audit of all 4 components returned **zero issues** — no non-P&S fonts, no unbound text fills, no bare solid fills. No kit-blue, no Inter, no wrapping, no 100×100 artifacts. The Badge set is visually identical to the bespoke Version State Badge (`20:23`) — same dot+label, wash fill, state-colored text; only the label string differs (`Error` here vs the bespoke `Failed`, per the task's variant naming).

### Effort signal for P2 rollout
Each atom took ~1 short scripted pass (clone → detach → componentize → re-bind paints → font override → sizing fix → verify). The genuinely-atomic primitives (Input, Badge, extracted Button/Tabs) are now **canonical and reusable** — P2 can instance these instead of re-rethemeing per screen, which collapses the per-instance grind for these four. The standing caveats hold: composites (Card etc.) are still not worth swapping, and a true variable remap still requires the (d) fork-and-republish path. Net: P1 removes the override cost for the four atoms across the screen set; P2's remaining work is swapping screen instances to these local components.

## P2 — reback + pilot-screen rollout (2026-05-26)

P2 did two things: (1) **rebacked** the shared bespoke **Version State Badge** set (`20:23`) so every variant is now shadcn-backed *without touching any instance*, and (2) **finalized** the Agent Catalogue (`23:3`) by swapping its three remaining bespoke primitives for instances of the P1 local components. Figma-only; git commit is this doc delta.

### Reback of the shared badge set (`20:23`) — DONE, all instances preserved

The bespoke set has exactly one component property — `State` (VARIANT) with the 7 options Draft·Staging·Canary·Production·Running·Failed·Paused — and **no** text/boolean/instance-swap props. A blast-radius scan found ~29 live instances across 4 pages (Screens 17, Components 6, Agent creation 4, Trigger creation 2; none on MCP / Namespace / KB / Lists / shadcn pilot). All sampled instances had **zero overrides** (default text matching their variant).

**Reback mechanic (no detach, no rename, no id change):** for each of the 7 variant COMPONENT nodes inside `20:23`, the inner hand-built `ellipse + text` was replaced with a nested **instance of the matching P1 Badge variant** (`396:74` set): Draft→`395:74`, Staging→`395:78`, Canary→`395:82`, Production→`394:83`, Running→`395:86`, **Failed→`395:90` (error) with the nested label overridden back to "Failed"**, Paused→`395:94`. To avoid a double-pill / layout shift, each wrapper variant's own wash fill, stroke, padding and gap were zeroed so the **nested shadcn instance defines the appearance**; the wrapper stays HUG. Result: every variant kept its exact size (Draft 59×22, Staging 70, Canary 70, Production 89, Running 73, Failed 63, Paused 69 — all unchanged), name, id, and the `State` variant property. Because ids/names/props are untouched, **all ~29 existing instances keep resolving** and simply inherit the shadcn-backed content — no detach, no broken instances.

> Why not a literal "swap-in-place over the bespoke fill": the bespoke variant is itself a complete wash pill (fill + padding + dot + text). Nesting a shadcn pill *inside* it unchanged would double the background and padding. Zeroing the wrapper's fill/padding/gap (so the inner instance is the only visible pill) was the only way to keep the outward size and look identical. If that hadn't held, the plan was to STOP and revert — it held cleanly on the first verified variant (Draft) before the other six were applied.

**Badge verification — screenshotted + viewed, all PASS:**
- **Agent Catalogue `23:3`** — card badges (Production green, Canary orange) render with correct dot+label, wash fills, no empty/broken instances, no layout shift.
- **Runs `51:56`** — STATUS column shows Running (teal), Failed (red), Paused (orange) from the rebacked set, all correct. (Succeeded badges are a separate component — the set has no Succeeded variant — and were unaffected.)
- **Version Promotion `48:40`** — Draft/Staging/Canary/Production render correctly both in the promotion-lane cards and the version-history table.
- The rebacked set itself (`20:23`) screenshots pixel-identical to the original bespoke set, including the "Failed" label (not "Error").

**Other screens get it for free:** because the reback edits the shared variant components in place (ids/names/props unchanged), every other screen's badge instances (Components, Agent creation, Trigger creation, and the rest of Screens) are now shadcn-backed automatically — no per-screen work, no per-instance overrides.

### Agent Catalogue (`23:3`) primitive swaps — DONE

Swapped three in-place bespoke primitives on the **real** `23:3` for instances of the P1 local components (referenced the pilot clone `364:3` for which nodes to target, but instanced the clean P1 locals rather than the pilot's detached kit frames):

| Bespoke (removed) | → P1 component | New instance id |
|---|---|---|
| `Button/New agent` `25:30` (Catalogue Header `25:18`) | Button primary `391:79` | **`425:105`** |
| FilterBar `Search field` `301:58` | Input `393:94` | **`426:107`** (resized to 190w, placeholder "Search agents…") |
| `ViewToggle` `301:79` (Blocks/Table) | Segmented `398:81` (Blocks active) | **`427:110`** |

Each instance was inserted at the original node's index in its auto-layout parent, then the bespoke node removed — position/order preserved, tokens stay bound (the P1 components carry the P&S bindings). Sizes follow the P1 spec (button HUG 95×36; input 190×36 vs old 28h; segmented 160×36 vs old 148×31) — minor height bumps from the P1 36px normalization, no layout break. `get_screenshot` of `23:3` (full + FilterBar `301:56` + Catalogue Header `25:18` crops) confirms: Tide-filled "New agent" button, shadcn Input with hairline border, Segmented with Tide active "Blocks" / muted "Table" — the active-pill being Tide (not a white sunken pill) is the canonical P1 Segmented appearance, verified against `398:81` itself, not a regression.

**Status: DONE.** Reback preserved all instances across all 4 pages; the three `23:3` swaps are clean. The Version State Badge is now shadcn-backed everywhere it appears.
