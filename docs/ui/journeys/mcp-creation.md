# J2 — MCP Creation

> **Entry:** `Add MCP` on Tool & MCP Catalogue (`56:56`, button `56:284`). **Figma worked example:** add the Linear MCP via OAuth (SSE transport) — frames `198:58` → `204:64`. **Grounds:** [AD-11](../../01-PRD.md) (MCPs are config-registered), [AD-18](../../01-PRD.md), [v3-tools-and-mcps](../../roadmap/v3-tools-and-mcps.md) (MCP registry, transport, auth, discovery), [09-security-model](../../deep-dive/09-security-model.md) §MCP Authentication / OAuth Token Lifecycle (E-11), [39-llm-provider-sdk-matrix](../../deep-dive/39-llm-provider-sdk-matrix.md), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (registry scoping).

MCPs are registered **dynamically via configuration** (no redeploy, [AD-11](../../01-PRD.md)) — unlike tools, which live in code. The 5-step wizard: **Connect → Auth → Discover → Test → Register**. An MCP entry records `name`, `transport`, `endpoint`, `auth_strategy`, `tool_discovery_mode` ([v3](../../roadmap/v3-tools-and-mcps.md) Story 2.1).

---

## Step 1 · Connect — Transport

The **transport branch** is the primary fork; each transport needs different connection inputs.

| Transport | What it is | Required inputs | Notes |
|---|---|---|---|
| **stdio** | local subprocess speaking MCP over stdin/stdout | `command` + `args[]`, env vars / secrets, namespace scope | the binary/process must be installed and runnable on the worker host; no URL. Implemented in v3 ([v3](../../roadmap/v3-tools-and-mcps.md) §Technical — "implement HTTP and stdio first") |
| **SSE** | remote server, Server-Sent-Events transport | Server `URL` (e.g. `https://mcp.linear.app/sse`), env vars, namespace scope | the URL must be network-reachable from the platform; the Figma example uses this |
| **streamable-http** | remote server, HTTP streamable transport | Server `URL`, env vars, namespace scope | HTTP transport variant; same reachability requirement |

Common to all: env vars / secrets (e.g. `LINEAR_WORKSPACE=acme`) and **namespace scope** (`acme`). Registration level branch ([16-multi-tenancy](../../deep-dive/16-multi-tenancy.md)):

| Level | Scope | Managed by |
|---|---|---|
| Platform-level | all namespaces | Platform Admin |
| Namespace-level | one namespace; may shadow a platform-level entry of the same name | Namespace Admin |

- **Necessary actions:** for **stdio**, the command/binary must be deployed to the worker image and any required env present. For **SSE / streamable-http**, the endpoint must be reachable from the orchestration zone and resolve TLS; secrets referenced (e.g. workspace IDs) must be supplied. Registration requires Platform Admin approval ([09-security-model](../../deep-dive/09-security-model.md) §MCP Impersonation: "MCP registrations require Platform Admin approval").
- **Validation:** unreachable URL / unrunnable command surfaces at Connect (no manifest); a misconfigured MCP must **fail fast at provisioning, not at execution** ([v3](../../roadmap/v3-tools-and-mcps.md) Story 2.2).

## Step 2 · Auth

The **auth-mode branch** maps to the declared `auth_strategy`:

| Auth mode | What the UI does | Necessary action |
|---|---|---|
| **OAuth 2.0** | "Connect to {provider}" handoff → consent → `Connected ✓` (pending Result chip while awaiting) | a registered OAuth app (client id/secret + **redirect URI**) must exist provider-side; provider must be reachable. Service-account OAuth in v3 (refresh token stored out-of-band by Platform Admin); user-delegated OAuth in v5 (per-user token via chatbot session). Access token is exchanged at workflow start; refresh is atomic per workflow ([09-security-model](../../deep-dive/09-security-model.md) E-11) |
| **API key** | masked key field | a valid provider API key; injected in headers |
| **Basic auth** | username + password | stored encrypted, injected at invocation |
| **None** | nothing | public MCP, no credentials |

- **Necessary actions (OAuth):** the OAuth client/app must be registered with the provider and its redirect URI allow-listed; the refresh token (service-account mode) is stored in the credential store by a Platform Admin via an out-of-band flow. Credentials never live in the DSL — only references ([AD-08](../../01-PRD.md), [09-security-model](../../deep-dive/09-security-model.md) §MCP Authentication).
- **Validation / failure:** consent denied or token exchange failure leaves the Result chip `pending`/`error` and blocks Continue; mid-workflow token expiry surfaces as `CredentialExpiredError` (non-retryable) at runtime ([09-security-model](../../deep-dive/09-security-model.md) E-11).

## Step 3 · Discover capabilities

After auth, the platform queries the MCP for its **manifest** and lists discovered **Tools / Resources / Prompts** as a checklist (choose what to expose). `tool_discovery_mode` is a manifest URL or runtime introspection ([v3](../../roadmap/v3-tools-and-mcps.md) Story 2.3).

- **Capability-selection branch:** the user checks which discovered capabilities the namespace exposes; unchecked capabilities are not referenceable from a `mcp_call`. (The Figma example shows all checked: tools `create_issue`/`search_issues`/`add_comment`, resources `issue://`/`team://`, prompt `triage_issue`.)
- **Necessary actions:** the manifest is **cached** (default TTL 60 min); force-refresh via `agent-platform mcps refresh <name>` (E-13). MCP entries may pin a `manifest_version` — a live version change raises a Platform Admin alert rather than silently switching.
- **Validation:** a `mcp_call` referencing a capability absent from the cached manifest → validation error; when a cached capability is later removed, the validation service **warns** on next provisioning of any DSL referencing it (E-13).

## Step 4 · Test

Runs a sample call (e.g. `search_issues({"query":"refund"})`) in a sunken well → `success` Result chip + **latency** (mono, e.g. `412ms`).

- **Necessary action:** a live round-trip through the chosen transport + auth; this is the proof that Connect + Auth + Discover all hold together.
- **Failure:** auth error, timeout, or protocol error produces a structured, persisted error ([v3](../../roadmap/v3-tools-and-mcps.md) Story 2.4) and an `error` Result chip; user returns to Auth/Connect.

## Step 5 · Register

Summary: `name` (e.g. `linear`), `version` (e.g. `1.0`), `scope` (e.g. `acme`), auth status `Connected` (`state/production` chip). Primary: **Register MCP** → the MCP card appears in the catalogue with `Connected` auth status.

- **What this enables:** the MCP becomes referenceable as `mcp_call.mcp` in any agent in scope, and exposable to the LLM via `llm_call.mcps[]` for native tool-calling ([AD-09](../../01-PRD.md), [39-llm-provider-sdk-matrix](../../deep-dive/39-llm-provider-sdk-matrix.md)). Idempotency keys (step run ID) are auto-injected for capabilities declaring `idempotent: true` (E-12).

---

## What differs per transport (quick reference)

| | stdio | SSE | streamable-http |
|---|---|---|---|
| Connection input | command + args | URL | URL |
| Network reachability | n/a (local process) | required | required |
| Host prerequisite | binary deployed to worker | none | none |
| Typical use | local/dev connectors | hosted SaaS MCPs (Linear) | hosted streaming MCPs |

## Failure-state summary

| Step | Failure | Fix |
|---|---|---|
| Connect | URL unreachable / command not found | fix endpoint/network or deploy the binary; check namespace scope |
| Auth | consent denied / token exchange fails / no OAuth app | register the OAuth app + redirect URI; supply valid key/refresh token |
| Discover | manifest fetch fails / capability missing | check transport+auth; force-refresh the manifest cache |
| Test | sample call errors / times out | resolve auth/protocol; verify the capability inputs |
| Register | not approved | obtain Platform Admin approval for the registration |

## Cross-references

- Decisions: [AD-11](../../01-PRD.md), [AD-18](../../01-PRD.md), [AD-09](../../01-PRD.md).
- Deep-dives: [v3-tools-and-mcps](../../roadmap/v3-tools-and-mcps.md) (Epic 2, E-12/E-13), [09-security-model](../../deep-dive/09-security-model.md) (§MCP Authentication, E-11), [39-llm-provider-sdk-matrix](../../deep-dive/39-llm-provider-sdk-matrix.md), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (registry scoping), [25-shared-agent-catalogue](../../deep-dive/25-shared-agent-catalogue.md) (catalogue context).
- Figma (worked example): `198:58` (Connect/SSE) · `201:58` (Auth/OAuth pending) · `202:60` (Discover) · `203:62` (Test) · `204:64` (Register). Parent screen: Tool & MCP Catalogue `56:56`.
