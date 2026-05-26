# Setup Journeys — Complete Decision Trees

> The six **Tier-3 Setup Journeys** ([04-screen-inventory](../04-screen-inventory.md)) are the multi-step flows that bring the platform's artifacts into existence: agents, MCPs, namespaces, KB sources, triggers, and credentials. The built Figma frames ([08-build-status](../08-build-status.md) §Journeys) each show **one worked path** through a journey. These docs cover **every branch** of each journey, plus the **real backend/config/operational actions** required to make each path actually work — grounded in the platform deep-dives.

## The shared takeover-wizard pattern

All six journeys are **full-screen takeover wizards** unified on one style (an earlier side-drawer treatment for five of them was reworked to takeover for consistency, 2026-05-25):

| Element | Spec |
|---|---|
| Frame | Full-viewport `surface/canvas` (#faf9f5), 1440×1024, no app shell |
| Top bar | `Conductor · {flow}` (left, Hanken SemiBold) + `Exit ✕` (right), 1px `border/hairline` bottom |
| Stepper | Centered horizontal `Takeover/Horizontal` stepper, `Step N of M`, no per-step text labels; done/active dots+bars bind `brand/tide`, upcoming bind `border/hairline` |
| Content | Centered ~720px `surface/card` column (Shadow/Card, radius 12, ~40px padding) |
| Footer | Sticky: ghost `Back`/`Cancel` (left) + primary `brand/tide` action (right) |
| Machine layer | IDs, slugs, cron expressions, endpoint URLs, keys, model names, vault URIs use JetBrains Mono (Mono/M, Mono/S) |

Each journey launches from a primary `New …`/`Add …` button on its parent screen (additive edits only). Wizards are **static designed states** — entry-point buttons are not wired to real navigation in the Figma track ([AD-25](../../01-PRD.md), [docs/ui README](../README.md)).

## The six journeys

| # | Journey | Doc | Entry screen | Branch coverage (summary) |
|---|---------|-----|--------------|---------------------------|
| J1 | Agent creation | [agent-creation.md](./agent-creation.md) | Agent Catalogue (`23:3`, btn `25:30`) | chatbot vs workflow; router vs specialist; LLM-classifier vs rule-based routing; memory tiers; safety levels; KB attach; full Visual-Builder step-type tail |
| J2 | MCP creation | [mcp-creation.md](./mcp-creation.md) | Tool & MCP Catalogue (`56:56`, btn `56:284`) | transport stdio / SSE / streamable-http; auth OAuth / API-key / basic / none; capability selection |
| J3 | Namespace creation | [namespace-creation.md](./namespace-creation.md) | Admin (`57:56`, btn `215:92`) | region/residency; quotas & budget; provider allow-list; safety + cost-cap defaults |
| J4 | Knowledge source | [kb-source.md](./kb-source.md) | KB Management (`68:56`, btn `68:284`) | 6 connectors (Web/S3/Drive/Notion/Upload/API); chunking/embedding/sync; 4-stage ingestion pipeline |
| J5 | Trigger creation | [trigger-creation.md](./trigger-creation.md) | Scheduling (`67:56`, btn `67:284`) | cron / webhook / event; retry & delivery policy; conflict/overlap policy |
| J6 | Credentials | [credentials.md](./credentials.md) | Admin (`57:56`, CREDENTIALS card btn `250:56`) | Provider API key / OAuth app / Secret; verify; vault storage; rotation |

## How to read each doc

Each journey doc contains:

1. **Purpose & entry point** — launching screen + the wizard's steps.
2. **Decision tree / all branches** — every option at each branching step and where it leads.
3. **Necessary actions to make each path work** — the real requirements behind the UI (registrations, network reachability, prerequisite artifacts, scopes), cited to AD-XX / deep-dive numbers.
4. **Validation / failure states** — what can go wrong per branch and what the user must fix.
5. **Cross-references** — relevant decisions/deep-dives and the Figma frame IDs that illustrate the worked example.

> **Design assumptions** are called out explicitly where the platform docs leave a UI choice unspecified. The journeys are a design surface over the documented platform; they do not introduce platform capabilities that contradict the docs.

## Story coherence

All journeys extend the single `acme-support` narrative: `acme-concierge` router chatbot, Linear MCP (OAuth), `acme-eu` namespace, `acme-helpcenter` web-crawl source, `kb-refresh` cron + `ticket-created` webhook, Anthropic production key. See the cross-screen story threads in [08-build-status](../08-build-status.md).
