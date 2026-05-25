# DSL Migration Tooling and Breaking Change Strategy

## Purpose

This document specifies how the platform handles DSL changes that cannot be expressed as purely additive extensions. It addresses a gap in `14-dsl-spec.md` and `12-upgrades-and-migrations.md`: both assert that DSL evolution is "additive only — never removes or renames," but neither specifies what happens when that constraint cannot be maintained, nor provides tooling for Agent Builders to migrate existing DSL documents when deprecations or structural changes occur.

The "additive only" contract is a strong and correct default. This document does not weaken it. Instead, it defines:

1. The classification of changes as additive, soft-breaking, or hard-breaking.
2. The process for managing soft-breaking changes under the additive contract.
3. The `agent-platform migrate` CLI command for mechanical DSL transformations.
4. The compatibility shim strategy for hard-breaking changes that cannot be avoided.

---

## Change Classification

| Class | Definition | Examples | Resolution |
|---|---|---|---|
| **Additive** | New field or step type. Old documents remain valid. | `budget` field (doc 23), `episodic_memory` (doc 21) | No migration needed |
| **Soft-breaking** | A field or step type is deprecated; old syntax still works but emits warnings. | `fetch` step deprecation (E-03), `grok` provider slug rename | Deprecation period + migration tooling |
| **Hard-breaking** | Old syntax cannot be supported alongside new semantics. | Expression grammar formalization (E-01) invalidating existing expressions | Compatibility shim or mandatory migration with version gate |

Hard-breaking changes are expected to be rare (zero expected through v6). The mechanism is documented here to prevent permanent deferral.

---

## Deprecation Lifecycle for Soft-Breaking Changes

### Phase 1 — Deprecation Warning (2 platform releases minimum)

The deprecated field/step type continues to function identically. The validation service emits a `DEPRECATION_WARNING`:

```
WARNING [workflow.steps[1].type]
        DEPRECATED_STEP_TYPE: Step type 'fetch' is deprecated as of platform v3.
        Use 'tool_call' or 'mcp_call' instead.
        Run 'agent-platform migrate <file> --deprecated-to-replacement' to auto-migrate.
        This warning will become an error in platform v5.
```

The `agent-platform validate --strict` flag treats deprecation warnings as errors, enabling CI pipelines to enforce migration before deadlines.

### Phase 2 — Deprecation Error (1 platform release before removal)

The deprecated syntax is rejected at validation time. Existing workflow runs that started before this gate continue executing (Temporal pinning ensures they use the agent version they started with — AD-16). New invocations of agents with deprecated syntax are blocked.

### Phase 3 — Removal

The syntax is removed from the validation service and interpreter. Agents using it cannot be provisioned or invoked.

---

## `agent-platform migrate` Command

### Overview

```
agent-platform migrate <file> [options]

Options:
  --from-dsl-version <N>    Source DSL version (default: auto-detect from dsl_version field)
  --to-dsl-version <N>      Target DSL version (default: latest)
  --dry-run                 Show proposed changes without writing the file
  --in-place                Write changes to the source file (default: write to <file>.migrated.json)
  --namespace <ns>          Namespace context (for registry lookups during migration)
  --json                    Output a machine-readable migration report
```

### Migration Report Output

```
$ agent-platform migrate ./agents/support-triage.json --dry-run

Analyzing support-triage.json (DSL v2 → v4)...

Migration: fetch → tool_call
  Location: workflow.steps[3] (id: "fetch_tickets")
  Change: Step type 'fetch' deprecated. Replacing with 'tool_call' referencing tool 'generic_http'.
  Before:
    { "type": "fetch", "request": { "url": "https://...", "method": "GET" } }
  After:
    { "type": "tool_call", "tool": "generic_http", "inputs": { "url": "...", "method": "GET" } }
  Risk: LOW — mechanical replacement; semantics preserved.

Migration: input_schema (recommended addition — E-04)
  Location: top-level
  Change: Workflow references $.workflow.input.user_question but no input_schema is declared.
  Proposed addition:
    { "input_schema": { "user_question": { "type": "string", "required": true } } }
  Risk: NONE — additive; validation service will now verify input references.
  Note: This migration is advisory. Run with --apply-advisory to include it.

─────────────────────────────────────────────────────────────────
Summary:
  1 required migration (fetch → tool_call)
  1 advisory migration (input_schema addition)
  0 manual migrations required

Run without --dry-run to apply required migrations.
```

### Migration Risk Levels

| Level | Meaning | Requires |
|---|---|---|
| `NONE` | Purely additive change; no behavioral difference | Auto-applied |
| `LOW` | Mechanical replacement; semantics identical | Auto-applied with confirmation |
| `MEDIUM` | Transformation with minor behavioral difference (e.g., default value change) | Applied with detailed explanation; requires `--confirm-medium` |
| `HIGH` | Semantic change that may affect outputs; requires human review | Not auto-applied; generates a TODO comment in output file |
| `MANUAL` | No automated migration possible; must be rewritten by Agent Builder | Not applied; generates detailed instructions |

### Migration Catalog

The migration catalog documents every known migration. It is versioned alongside the platform. Each entry specifies:

- **Trigger**: the deprecated syntax pattern.
- **Replacement**: the new syntax.
- **Risk level**: from the table above.
- **Automated**: whether the migration tool can apply it.

Example catalog entries:

| From | To | Risk | Automated |
|---|---|---|---|
| `step.type: fetch` | `step.type: tool_call` with `tool: generic_http` | LOW | Yes |
| `provider: grok` (if renamed) | `provider: xai` | LOW | Yes |
| `retry.initial_interval` (old naming) | `retry.initial_interval_seconds` | LOW | Yes |
| Expression using unsupported operator `!` (E-01 grammar change) | Rewrite using `== false` | HIGH | No — MANUAL |

---

## Compatibility Shims for Hard-Breaking Changes

When a hard-breaking change is unavoidable (exclusively anticipated for the expression grammar formalization, E-01), the platform provides a compatibility shim.

### Shim Mechanism

A compatibility shim is a DSL-version-aware interpretation layer in the workflow interpreter:

1. When the interpreter loads a DSL document, it reads `dsl_version`.
2. If `dsl_version < current_version` and a shim is registered for that version, the shim pre-processes the DSL before interpretation.
3. The pre-processed form is interpreted by the current interpreter.

Shims are registered in the platform code as version-specific transformers. They are not exposed to Agent Builders.

### Shim Lifecycle

- Shims are introduced when a hard-breaking change ships.
- Shims are maintained for 2 major platform versions minimum.
- When a shim is removed, agents pinned to the older DSL version will fail validation. Before removal, the platform enforces migration via the `agent-platform migrate` tool.

---

## Batch Migration (Fleet-Wide)

For Platform Admins managing many agents across a namespace:

```
agent-platform migrate --all-agents \
  --namespace acme-corp \
  --to-dsl-version 4 \
  --dry-run \
  --output-report ./migration-report.json
```

The batch migration:
1. Fetches all agent definitions from the API.
2. Runs the migration analyzer on each.
3. Produces a fleet-wide report: how many agents need migration, risk distribution, agents requiring manual review.
4. Does not write back to the platform without explicit `--apply` (even with `--in-place`).

Fleet-wide apply:

```
agent-platform migrate --all-agents \
  --namespace acme-corp \
  --to-dsl-version 4 \
  --apply \
  --low-risk-only \
  --create-draft-versions
```

`--create-draft-versions`: instead of overwriting the current version, creates new `draft` versions (per `20-agent-version-promotion.md`) for each migrated agent. Agent Builders can review before promoting.

---

## DSL Version Compatibility Matrix (Extended)

The existing table in `12-upgrades-and-migrations.md` is extended with migration tooling availability:

| Platform Version | DSL Versions Supported | Migration Tool Supports |
|---|---|---|
| v1 | v1 | — |
| v2 | v1, v2 | v1 → v2 |
| v3 | v1–v3 | v1 → v3, v2 → v3 |
| v4 | v1–v4 | v1 → v4, v2 → v4, v3 → v4 |
| v5 | v1–v5 | v1 → v5, ..., v4 → v5 |
| v6 | v1–v6 | Full backward migration from any version |

---

## Validation Service Additions

The `agent-platform validate` command gains migration awareness:

- When deprecated syntax is detected, the validation output includes: *"1 deprecated construct found. Run `agent-platform migrate <file>` to auto-migrate."*
- When a hard-breaking change is pending (shim expiry within 2 versions): *"This document uses DSL constructs that will require migration before platform v7. Run `agent-platform migrate --check-upcoming <file>` for details."*

---

## Implementation Phase

| Feature | Phase |
|---|---|
| `agent-platform migrate` command (initial, covering E-03 fetch deprecation) | v3 |
| Migration catalog (versioned, extensible) | v3 |
| Batch migration (`--all-agents`) | v5 |
| Compatibility shim framework | v6 (in anticipation of E-01 expression grammar formalization) |
| `--create-draft-versions` integration with v20 promotion | v6 |

---

## What to Read Next

DSL migration is the bridge between the "additive only" guarantee and real-world deprecations. Read:

→ **[14-dsl-spec.md](./14-dsl-spec.md)** — The canonical DSL reference: every field, step type, and version table that `agent-platform migrate` works against.

Or explore related tooling and lifecycle:
- [12-upgrades-and-migrations.md](./12-upgrades-and-migrations.md) — Platform-level upgrade and rollback procedures that migrations plug into
- [15-developer-tooling.md](./15-developer-tooling.md) — Full `agent-platform` CLI reference, including the `migrate` command added by this document
- [20-agent-version-promotion.md](./20-agent-version-promotion.md) — How `--create-draft-versions` integrates migration output with the promotion lifecycle
- [18-enhancements.md](./18-enhancements.md) — E-01 (expression grammar formalization), E-03 (`fetch` deprecation) — the two primary triggers for migration tooling

---

## Related Documents

- [14-dsl-spec.md](./14-dsl-spec.md) — DSL versioning table, additive evolution guarantee
- [12-upgrades-and-migrations.md](./12-upgrades-and-migrations.md) — Agent definition migration utilities
- [15-developer-tooling.md](./15-developer-tooling.md) — `agent-platform` CLI (new `migrate` command)
- [18-enhancements.md](./18-enhancements.md) — E-01 (expression grammar), E-03 (`fetch` deprecation), E-04 (input_schema)
- [20-agent-version-promotion.md](./20-agent-version-promotion.md) — Draft version creation for batch migration
