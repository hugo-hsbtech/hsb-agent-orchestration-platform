# v7 Platform Sketch

> **Status**: Post-v6 horizon sketch. Nothing here is committed. This document captures directional thinking to prevent architecture decisions in v1–v6 from accidentally closing off these possibilities. It should be revisited and formalized when v6 is in production.

---

## Purpose

v7 is the first release built on top of a complete, hardened platform (v6). Its goal is to extend the platform beyond its initial scope in directions that compound the value already delivered.

---

## Candidate Capability Areas

### C-01: Visual Workflow Builder (UI-first DSL editing)

The platform API and CLI are the only provisioning surfaces through v6. A visual builder would allow Agent Builders to construct and modify agent DSL documents using a graph-based UI instead of raw JSON.

**Design constraints that v1–v6 must not violate**:
- The DSL remains the canonical source of truth. The UI produces and consumes DSL documents; it does not have a parallel data model.
- Any new DSL field must have a human-readable label and description registered in the schema, to enable UI form generation.
- The validation service API must remain a first-class client — the UI calls it on every edit, identical to the CLI.

---

### C-02: Marketplace / Shared Catalogue (public tier)

`25-shared-agent-catalogue.md` covers namespace-scoped sharing. v7 could introduce a platform-wide public catalogue where agent definitions are published, versioned, and consumed across organizations.

**Design constraints that v1–v6 must not violate**:
- The `catalogue://slug` URI scheme introduced for internal sharing (see `14-dsl-spec.md` `agent_call` notes) must remain stable. Public catalogue entries would use the same URI scheme with a `public://` or `marketplace://` prefix.
- Agent definitions must be serializable as standalone, portable JSON documents with no implicit platform state baked in.
- Namespace isolation must remain the isolation primitive — a public catalogue entry is adopted into a namespace, not shared raw.

---

### C-03: Streaming Workflow Agents

v5 introduces synchronous streaming for chatbots. v7 could extend streaming to workflow agents, enabling long-running workflow steps to push incremental results to the caller rather than blocking until completion.

**Design constraints that v1–v6 must not violate**:
- The step output envelope (`$.steps.<id>.output.*`) must remain the canonical data flow reference. Streaming is a delivery mechanism, not a data model change.
- The `agent_call (wait)` mode must remain semantically synchronous through v6. v7 may introduce a `mode: stream` that changes delivery but not the workflow graph contract.
- The SSE/WebSocket infrastructure introduced in v5 for chatbot streaming should be reusable, not duplicated.

---

### C-04: Multi-Region Workflow Execution

v6 is single-region conceptually. v7 could introduce workflow pinning to execution regions based on data residency requirements or latency optimization.

**Design constraints that v1–v6 must not violate**:
- The `namespace` field is the tenancy unit. Region assignment maps to namespace, not to individual agent definitions or workflow runs — this keeps the DSL region-agnostic.
- Temporal's multi-cluster support is the mechanism. The platform must not build region routing at the application layer.
- Cross-region `agent_call` must remain prohibited by default (same as cross-namespace today) unless explicitly allowed.

---

### C-05: Agent Evaluation Framework

Agent Builders currently test via `dry-run` and integration tests. v7 could introduce a structured evaluation framework: define test cases with expected outputs, run them against an agent version, get a scored report.

**Design constraints that v1–v6 must not violate**:
- Evaluation test cases must be plain DSL-aligned JSON. No proprietary format.
- The cassette format from `11-testing-strategy.md` is the natural precursor. Evaluation builds on cassettes, not a parallel recording mechanism.
- Evaluation results must be stored per agent version and queryable via the existing API, not a separate service.

---

## Design Gates

Before v7 planning begins in earnest, the following must be decided:

| Gate | Depends On | Notes |
|---|---|---|
| **AD-27 (backend language)** | v1 planning | Language choice affects UI SDK, streaming ergonomics, and evaluation library options. Must be decided before v1. |
| **AD-28 resolved (multi-device conflict)** | v5 production | Multi-device conflict resolution (E-28) must be decided before v7 conversation features are designed. |
| **v6 production data** | v6 shipped | Actual usage data (agent graph depth, fan-out sizes, catalogue sharing patterns) should inform v7 priorities. |

---

## What to Read Next

- **[25-shared-agent-catalogue.md](./25-shared-agent-catalogue.md)** — Internal catalogue, the precursor to v7's marketplace tier
- **[14-dsl-spec.md](./14-dsl-spec.md)** — DSL contract that v7 must remain compatible with
- **[18-enhancements.md](./18-enhancements.md)** — Enhancement proposals that include post-v6 items deferred to this horizon
- **[07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md)** — The foundation v7 builds on

---

## Related Documents

- [01-PRD](./01-PRD.md) — AD-25 (no UI in early phases), AD-26 (multi-tenancy foundation)
- [18-enhancements.md](./18-enhancements.md) — E-26 (this sketch)
- [25-shared-agent-catalogue.md](./25-shared-agent-catalogue.md) — Internal catalogue (C-02 precursor)
