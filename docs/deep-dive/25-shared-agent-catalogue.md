# Shared Agent Catalogue

## Purpose

This document specifies the **Shared Agent Catalogue** — a registry of agent definitions that are published at the platform level and reusable across namespaces without per-namespace reimplementation. It closes a gap in the multi-tenancy model: `16-multi-tenancy.md` defines cross-namespace agent access via an explicit allowlist (a security primitive), but provides no discoverability or publishing mechanism. The Tool Registry and MCP Registry already have a platform-level tier (`16-multi-tenancy.md` §Tool and MCP Registry Scoping). Agents have no equivalent.

Without a catalogue, every namespace that needs a `language-detector`, `pii-scrubber`, or `document-summarizer` agent must build and maintain its own. This directly contradicts the "platform as meta-product" value proposition (PD-01).

---

## Concept: Catalogue vs. Allowlist

The existing cross-namespace allowlist and the catalogue are distinct:

| Concept | Purpose | Managed By | Discovery |
|---|---|---|---|
| Cross-namespace allowlist | Security: explicitly permit namespace A to call namespace B's private agents | Platform Admin | Not discoverable — bilateral agreement |
| Shared Catalogue | Publishing: make an agent available to all namespaces as a platform-level primitive | Platform Admin (publishing) / Agent Builders (consumption) | Discoverable via API and CLI |

The allowlist is not deprecated. It remains the mechanism for bilateral sharing of private agents. The catalogue is for agents explicitly published as platform-level utilities.

---

## Catalogue Entry

A **catalogue entry** wraps an agent version with publishing metadata:

```json
{
  "catalogue_id":    "cat-uuid",
  "agent_id":        "agent-uuid",
  "agent_version":   "v3",
  "namespace":       "platform-utilities",
  "slug":            "language-detector",
  "display_name":    "Language Detector",
  "description":     "Detects the BCP-47 language code of a given text string. Supports 50+ languages.",
  "category":        "text-processing",
  "tags":            ["language", "classification", "utility"],
  "input_schema":    { "text": { "type": "string", "required": true } },
  "output_schema":   { "language_code": { "type": "string" }, "confidence": { "type": "number" } },
  "visibility":      "public | private",
  "published_at":    "ISO 8601",
  "published_by":    "platform-admin",
  "deprecated":      false,
  "deprecation_note": null
}
```

| Field | Description |
|---|---|
| `namespace` | The owning namespace. Catalogue agents live in a dedicated `platform-utilities` namespace by convention. |
| `slug` | URL-safe identifier. Unique across the catalogue. Used in DSL `agent_call` references. |
| `category` | Browsing dimension: `text-processing`, `data-validation`, `knowledge`, `routing`, `summarization`, `integration`. |
| `input_schema` | Canonical input schema (mirrors E-04 `input_schema` field). Used for cross-agent type validation. |
| `output_schema` | Declared output shape for downstream data flow validation. |
| `visibility` | `public`: visible to all namespaces. `private`: visible only to allowlisted namespaces (a hybrid model). |

---

## Discovery API

### Browse Catalogue

```
GET /catalogue/agents
    ?category=text-processing
    &tags=language,classification
    &q=language+detector

{
  "entries": [ { ...catalogue_entry... }, ... ],
  "total": 12,
  "page": 1
}
```

### Get Entry Detail

```
GET /catalogue/agents/{slug}

{
  ...catalogue_entry...,
  "example_input":  { "text": "Bonjour tout le monde" },
  "example_output": { "language_code": "fr", "confidence": 0.99 }
}
```

The catalogue API is read-accessible to all authenticated namespaces. No namespace token scoping — browsing is universal.

### CLI

```
agent-platform catalogue list [--category <cat>] [--tag <tag>] [--search <query>]
agent-platform catalogue inspect <slug>
agent-platform catalogue call <slug> --input <json> --namespace <ns>
```

`catalogue call` invokes the shared agent from the caller's namespace context (for quick testing before embedding in a DSL).

---

## Using Catalogue Agents in DSL

Catalogue agents are referenced in `agent_call` steps by `catalogue://` URI:

```json
{
  "id": "detect_language",
  "type": "agent_call",
  "agent": "catalogue://language-detector",
  "mode": "wait",
  "inputs": {
    "text": "$.workflow.input.user_message"
  }
}
```

The `catalogue://` prefix tells the interpreter to look up the slug in the catalogue rather than the caller's namespace registry. The resolution is pinned at workflow start: the catalogue entry's `agent_version` at the time of invocation is recorded on the step run, providing the same immutability guarantee as namespace-scoped agent calls.

### Version Pinning for Catalogue Agents

```json
{
  "agent": "catalogue://language-detector@v3"
}
```

Explicit version pins are supported. Without a pin, the `agent_version` field of the current catalogue entry is used.

### Validation Service Behavior

The validation service resolves `catalogue://` references against the live catalogue. For `agent-platform validate` offline mode (CLI without platform connection), catalogue resolution is skipped and a warning is emitted: `CATALOGUE_NOT_RESOLVED: catalogue://language-detector requires platform connection to validate.`

---

## Publishing to the Catalogue

### Publishing Workflow

Only Platform Admins can publish to the catalogue:

```
POST /admin/catalogue/publish
{
  "agent_id": "uuid",
  "agent_version": "v3",
  "slug": "language-detector",
  "display_name": "Language Detector",
  "description": "...",
  "category": "text-processing",
  "tags": ["language", "classification"],
  "visibility": "public"
}
```

Before publishing, the validation service runs the agent's full validation including:
- Schema compliance.
- Input/output schema declared (required for catalogue entries).
- No references to namespace-private tools, MCPs, or credentials (catalogue agents must be self-contained or use platform-level tools/MCPs only).

The last rule — **catalogue agents may not reference namespace-private resources** — is enforced because the agent runs in any caller's namespace context. A private credential reference would be unresolvable.

### `platform-utilities` Namespace

By convention, all platform-published catalogue agents live in a dedicated `platform-utilities` namespace. This namespace:
- Is created automatically during platform bootstrap.
- Belongs to the Platform Admin.
- Does not appear in the regular namespace listing (it is an internal namespace).
- Its agents are only accessible via `catalogue://` references or direct version pinning.

Operators may also create their own namespace-level catalogue for internal organization (e.g., `acme-corp-utilities` with `visibility: private`).

---

## Reference Agent Library (Starter Catalogue)

The platform ships with a starter set of catalogue agents. These are implemented and maintained by the platform team:

| Slug | Category | Description | Uses |
|---|---|---|---|
| `language-detector` | text-processing | Detect BCP-47 language code | `knowledge_base_lookup` tool |
| `text-summarizer` | summarization | Summarize text to configurable length | `summary_agent` for E-15 |
| `pii-scrubber` | data-validation | Detect and redact PII categories | E-16 PII handling |
| `quality-scorer` | text-processing | Score text quality against a rubric | A/B evaluation (doc 20) |
| `intent-classifier` | routing | Multi-class intent classification | Routing chatbot (E-14) |
| `document-chunker` | text-processing | Split long documents into chunks with overlap | KB ingestion pipelines |
| `schema-validator` | data-validation | Validate a JSON payload against a schema | Data pipeline quality gates |

Each starter agent has:
- A full DSL definition in the platform repository.
- A cassette-based test suite (per `11-testing-strategy.md`).
- An example invocation in the catalogue `example_input` / `example_output` fields.

---

## Agent Deprecation in the Catalogue

When a catalogue agent version is updated:

1. The old entry's `deprecated: true` flag is set.
2. `deprecation_note` explains why and what to use instead: *"v2 replaced by v3 which adds support for emoji-heavy text. Callers using `@v2` pin should update to `@v3` or remove the version pin."*
3. The platform validation service warns on any DSL referencing the deprecated slug or version: `CATALOGUE_DEPRECATED: catalogue://language-detector@v2 is deprecated. Migrate to @v3.`
4. Deprecated catalogue entries are removed after a minimum 2-platform-version deprecation window, using the same deprecation lifecycle as DSL constructs (`24-dsl-migration-tooling.md`).

---

## Quota and Cost Accounting

Catalogue agent invocations are charged to the **caller's namespace**, not the `platform-utilities` namespace:

- The caller's `max_workflow_runs_per_day` quota is consumed.
- Token costs are attributed to the caller's namespace in cost tracking (E-18).
- The `platform-utilities` namespace has no cost quotas.

This ensures that heavy users of shared agents pay their fair share and that shared agents cannot be weaponised to exhaust another namespace's quota.

---

## Implementation Phase

| Feature | Phase |
|---|---|
| `platform-utilities` namespace bootstrap | v1 (no-op namespace, schema cost zero) |
| Catalogue API (browse, inspect) | v5 (after multi-agent orchestration is stable) |
| `catalogue://` DSL reference resolver | v5 |
| Starter agent library (first 3 agents) | v5 |
| CLI `catalogue` commands | v6 (alongside Operator CLI work) |
| Full starter library (all 7 agents) | v6 |
| Private namespace-level sub-catalogues | v7 |

---

## What to Read Next

The catalogue builds on top of the multi-agent and multi-tenancy infrastructure. Start with:

→ **[05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md)** — The `agent_call` step type that the `catalogue://` URI scheme extends, including version pinning, sync vs fire-and-forget, and fan-out patterns.

Or explore adjacent systems:
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace model and the cross-namespace allowlist that the catalogue complements (not replaces)
- [20-agent-version-promotion.md](./20-agent-version-promotion.md) — How catalogue agents are promoted through draft→staging→canary→production before being published
- [14-dsl-spec.md](./14-dsl-spec.md) — Where the `catalogue://` URI scheme and `agent_call` field are formally specified
- [17-operator-cli.md](./17-operator-cli.md) — The `catalogue list`, `inspect`, and `call` CLI commands

---

## Related Documents

- [01-PRD](./01-PRD.md) — PD-01 (platform as meta-product), AD-11 (tool vs. MCP registration lifecycle)
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace model, cross-namespace allowlist, tool/MCP scoping
- [05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md) — `agent_call` step type
- [14-dsl-spec.md](./14-dsl-spec.md) — `agent_call` DSL field (catalogue:// URI scheme addition)
- [17-operator-cli.md](./17-operator-cli.md) — CLI reference (new `catalogue` commands)
- [20-agent-version-promotion.md](./20-agent-version-promotion.md) — Version promotion (catalogue agents use same promotion model)
