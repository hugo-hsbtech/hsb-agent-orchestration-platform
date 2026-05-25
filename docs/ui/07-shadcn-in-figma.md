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
