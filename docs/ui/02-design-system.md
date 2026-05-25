# 02 — Design System: "Paper & Signal"

> Written under the `frontend-design` skill. That skill's mandate: commit to a distinctive, intentional direction and avoid generic AI aesthetics (no Inter/Roboto, no purple-on-white gradients, no Space Grotesk default). "Refined SaaS, light" is the brief — but refined only counts if it has a point of view. This is that point of view.

## The concept

**Paper & Signal.** Three ideas, one system:

1. **Paper** — the human layer. Warm off-white surfaces, ink-black text, hairline rules, soft layered shadows. It reads like a well-set document, not a glass dashboard. Calm under density.
2. **Signal** — orchestration is motion and state. A single confident accent (**Tide**, a deep teal-cyan) marks every *action* and *live* thing. State gets its own disciplined semantic palette, used **only** for state.
3. **Machine** — IDs, DSL, trace IDs, durations, token counts, costs. These get a monospace treatment so the operator's eye locks onto them instantly. The machine layer is a first-class typographic register, not an afterthought.

The result is enterprise-credible and warm, with a clear identity that is *not* "another blue SaaS dashboard."

## Typography

Distinctive, free (Google Fonts), available in Figma.

| Role | Family | Why |
|---|---|---|
| **Display** (page titles, hero metrics, empty-state headlines) | **Bricolage Grotesque** | Characterful, slightly humanist grotesque with real personality. Memorable without being loud. |
| **Body / UI** (everything functional) | **Hanken Grotesk** | Clean, warm, highly legible at small sizes and in dense tables. The workhorse. |
| **Mono** (IDs, DSL, JSON, metrics, trace IDs, code) | **JetBrains Mono** | Built for code legibility; clear `0/O`, `1/l/I`. Defines the machine register. |

Type scale: a modular scale on an 8px rhythm (12 / 14 / 16 / 20 / 24 / 32 / 44). Body default 14px (operations tools live at 14, not 16). Tight, deliberate line-height in tables (1.4); roomier in prose (1.6).

## Color

Built as **Figma variables** (light mode now; structured so a dark mode is a second mode later), named consistently (`surface/canvas`, `brand/tide`, …). Hexes below are intent; exact values are tuned in Figma.

### Neutrals — warm "paper"

Not cold gray. A warm stone/taupe ramp gives the "paper" warmth.

| Token | Intent | Approx |
|---|---|---|
| `surface/canvas` | App background | warm off-white `#FAF9F5` |
| `surface/card` | Cards, panels | `#FFFFFF` |
| `surface/sunken` | Wells, code blocks | `#F4F2EC` |
| `border/hairline` | 1px rules | `#E7E4DB` |
| `text/ink` | Primary text | warm near-black `#1C1B18` |
| `text/muted` | Secondary | `#6B675E` |
| `text/faint` | Tertiary / placeholders | `#9C978C` |

### Signal — the brand accent

| Token | Intent | Approx |
|---|---|---|
| `brand/tide` | Primary actions, active nav, links, "live" motifs | deep teal-cyan `#0E7C86` |
| `brand/tide-bright` | Accents, focus rings, running pulses | `#13A4B0` |
| `brand/tide-wash` | Tinted backgrounds, selected rows | `#E6F3F4` |

Teal is the deliberate choice: distinctive against the sea of SaaS blue, professional, and — critically — it does **not** collide with the semantic state ramp below (which owns blue/green/amber/violet/red).

### Semantic state — reserved, never decorative

The disciplined heart of the system. These colors appear **only** to encode lifecycle state, so a glance is always trustworthy.

| State | Use | Color family |
|---|---|---|
| `state/draft` | Unpromoted versions, drafts | slate gray |
| `state/staging` | Staging versions | violet |
| `state/canary` | Canary versions / partial traffic | amber |
| `state/production` | Live production versions | green |
| `state/running` | In-flight runs/steps (animated pulse) | Tide brand |
| `state/error` | Failed runs/steps, validation errors | red |
| `state/paused` | Paused runs, circuit-open | desaturated amber |

This set is the visual encoding of `AD-16`, `AD-29` (version lifecycle) and the run lifecycle (`AD-17`).

### Provider chips

Brand-approximate chips for `Claude / OpenAI / Gemini / Grok` (`AD-09`) — a small dot + label, used wherever a model provider appears.

## Spatial system, depth, motion

- **Grid:** 8px base. Page gutters, card padding, and component spacing all derive from it.
- **Radius:** medium and consistent (8px controls, 12px cards). No pill-everything.
- **Depth:** soft, layered, low-opacity shadows on warm paper — cards *float*, they don't sit in boxes. Avoid flat solid-color blocks (a frontend-design "slop" tell).
- **Texture:** a barely-there paper grain on the canvas; a faint dotted grid on the workflow builder; hairline rules instead of heavy dividers.
- **Motion (design intent):** calm and purposeful — a soft pulse on `state/running`, the routing badge sliding in when a chat turn is classified, the background-task card transitioning `pending → done` when surfaced. High-impact moments, not scattered micro-jitter. Documented as design intent; shown in Figma where it helps.

## Component library

Two layers. Built as Figma component sets with variants, mirrored to shadcn/ui.

### shadcn primitives (themed to Paper & Signal)

Button, Input, Textarea, Select, Combobox/Command (⌘K), Checkbox, Switch, Radio, Slider, Badge, Card, Table (sortable, dense), Tabs, Accordion, Dialog, Sheet/Drawer, Popover, Tooltip, DropdownMenu, Avatar, Breadcrumb, Pagination, Toast (Sonner), Skeleton, Progress, Separator, ScrollArea, Resizable panels, Sidebar, Chart.

### Custom composites (the platform's signature)

| Composite | What it is | Demonstrates |
|---|---|---|
| **App Shell** | Sidebar + namespace switcher + topbar + ⌘K palette | `AD-26` |
| **Agent Card** | Name, provider chip, type, version-state badge, last run | catalogue |
| **Version-State Badge** | The 7-state semantic pill | `AD-16`, `AD-29`, `AD-17` |
| **Provider Chip** | Dot + label per LLM provider | `AD-09` |
| **Workflow Node** + **Edge** + **Node Inspector** | DSL step types as canvas nodes | v2/v4, `14-dsl-spec` |
| **DSL Split Panel** | Highlighted read-only JSON beside the canvas | `AD-02`, DSL↔visual duality |
| **Validation Row** | One row per error in the collect-all list | `AD-13`, `PD-06` |
| **Promotion Lane** | 4-stage lane + canary traffic-split control | `AD-29` |
| **Step Timeline Row** | Status, duration, retries, expandable I/O | v2, `AD-17` |
| **Debug Controls** | Pause / resume / cancel / approve-next-step | `AD-17` |
| **Trace Span Bar** | OTel span in a waterfall | `AD-24` |
| **Chat Bubble** + **Routing Badge** + **Citation Chip** | The chat turn anatomy | `AD-19/20`, KB |
| **Background-Task Card** | pending → proactively-surfaced result | `AD-21` |
| **Degradation Banner** | Specialist/KB circuit-open notice | `AD-31` |
| **Scorecard Cell** | Eval pass/fail/score heatmap cell | `27` |
| **KB Source Card** + ingestion status pill | KB lifecycle | `33` |
| **Trigger Row** | cron / webhook / event subscription | `29` |
| **Metric Stat** + **Chart Card** | Analytics primitives | `30` |
| **Quota Meter** + **Price-Table Row** | Admin governance | `16`, `23`, `40` |
| **Audit Row** + **Consent Toggle** | Compliance console | `35`, `28` |

→ Tokens and components are built first in Figma (Phase A), see [05-build-workflow.md](./05-build-workflow.md).
