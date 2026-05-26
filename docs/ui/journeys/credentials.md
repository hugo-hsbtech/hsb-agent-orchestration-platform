# J6 — Credentials

> **Entry:** `+ Add credential` on the Admin CREDENTIALS card (`57:56` → card `58:83`, button `250:56`). **Figma worked example:** Anthropic production key (`sk-ant-••••••••` → `sk-ant-…3f9`, vault `vault://acme/anthropic`) — frames `247:82` → `249:81`. **Grounds:** [AD-07](../../01-PRD.md) (plain-text in v1–v5, vault in v6), [AD-08](../../01-PRD.md) (credentials snapshot per run), [09-security-model](../../deep-dive/09-security-model.md) (credential zone, MCP auth, OAuth lifecycle E-11, rotation, scopes E-29), [v3-tools-and-mcps](../../roadmap/v3-tools-and-mcps.md) (CredentialsProvider abstraction), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (credentials are namespace-scoped).

Credentials are the **only** place secrets live — agent DSL contains **references**, never raw values ([AD-08](../../01-PRD.md), [09-security-model](../../deep-dive/09-security-model.md) §MCP Authentication: "Agent definitions contain no raw credentials"). All reads go through the `CredentialsProvider` interface ([v3](../../roadmap/v3-tools-and-mcps.md) Story 4.1) so the v6 vault swap is a single-implementation change. Only Platform Admins manage credentials (`credentials:read`/`credentials:write` scopes, [09-security-model](../../deep-dive/09-security-model.md) E-29). The 3-step wizard: **Enter → Verify → Stored**.

---

## Step 1 · Enter — type branch

| Type | What it stores | Required fields | Necessary action |
|---|---|---|---|
| **Provider API key** | an LLM provider key | provider chip (`Claude`/`OpenAI`/`Gemini`/`Grok`), masked key, label, namespace scope, optional rotation reminder | the key must be valid for the chosen provider; the provider must be configured + in the namespace allow-list (J3) for agents to use it |
| **OAuth app** | an OAuth client + refresh token for an MCP/connector | client id/secret, **redirect URI**, token endpoint/scopes; the obtained refresh token | the OAuth app must be **registered with the provider** and its redirect URI allow-listed; service-account OAuth stores a refresh token out-of-band (v3), user-delegated OAuth comes from the chatbot session (v5) — [09-security-model](../../deep-dive/09-security-model.md) E-11 |
| **Secret** | an arbitrary secret (webhook signing secret, API token, basic-auth pair, S3 key) | name/ref, value(s), namespace scope | referenced by `secret_ref` from webhook triggers (J5), `fetch`/`mcp_call` steps (`{{credentials.*}}`), and KB connector `auth_ref` (J4) |

Shared fields: **label** (e.g. "Anthropic — production"), **namespace scope** (credentials are namespace-isolated — namespace A's `jira_api_key` is invisible to B, [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md)), **rotation reminder** toggle (e.g. 90 days).

- **Necessary action:** caller must hold `credentials:write` (Platform Admin, [09-security-model](../../deep-dive/09-security-model.md) E-29); the namespace must exist (J3).
- **Validation:** the value is masked on entry and never logged ([09-security-model](../../deep-dive/09-security-model.md) §Audit — "Credential access (read, not value)").

## Step 2 · Verify — branch by type

A summary row + a live verification → **success** / **error** Result chip.

| Type | Verification | Success / failure |
|---|---|---|
| Provider API key | call the provider (list models / cheap probe) | success: "Key valid · models reachable"; failure: error chip "key rejected" → fix and re-enter |
| OAuth app | complete the consent/token exchange and a probe call | success: token obtained + scope OK; failure: consent denied / redirect mismatch / scope insufficient |
| Secret | format/echo check (or a target-system probe where applicable) | success: stored-shape valid; (a signing secret has no remote verify — *design assumption: verified by use at first trigger delivery*) |

- **Necessary action:** verification proves reachability + validity **before** storage so failures surface here, not at an agent's first invocation ([v3](../../roadmap/v3-tools-and-mcps.md) Story 2.2 "fail fast at provisioning"). Provider keys are subject to the rate-limit token bucket like any other call.
- **Failure:** an `error` chip blocks Save; user returns to Enter. Auth errors map to `ConfigurationError` (no retry — alert operator, [39-llm-provider-sdk-matrix](../../deep-dive/39-llm-provider-sdk-matrix.md) §Error Classification).

## Step 3 · Stored — vault

Success banner + masked value (`sk-ant-…3f9`) + **vault location** (`vault://acme/anthropic`) + last-verified timestamp + status chip **Active** (`state/production`).

- **Storage reality ([AD-07](../../01-PRD.md)):** in **v1–v5** credentials are stored **plain-text in the database** (risk explicitly accepted); the **vault** (`vault://…` URI) is introduced in **v6**. The `vault://` location shown is the v6 end-state; behind the `CredentialsProvider` interface the swap is transparent to agents.
- **Runtime use ([AD-08](../../01-PRD.md)):** at workflow start the entry activity loads all needed credentials **once** into the in-memory context snapshot, passed through every step; the snapshot is not persisted beyond the run. For OAuth, the access token is exchanged at start; refresh is atomic (one attempt per workflow) — refresh failure is `CredentialExpiredError` (non-retryable).
- **Rotation ([09-security-model](../../deep-dive/09-security-model.md) §Secret Rotation):** new secrets are added; in-flight workflows continue on their snapshot; new workflows fetch the updated credential; the old secret is revoked after confirming no active references. The rotation reminder (Step 1) prompts this cycle.

---

## What each credential type unblocks downstream

| Type | Enables |
|---|---|
| Provider API key | a provider in the namespace allow-list → models callable by agents (J1 Step 3); fallback providers |
| OAuth app | MCP OAuth auth (J2 Step 2); KB connectors using OAuth (Google Drive, J4) |
| Secret | webhook signing secret (J5 Branch B); `fetch`/`mcp_call` credential refs; KB connector `auth_ref` (S3/Notion/API, J4) |

## Failure-state summary

| Step | Failure | Fix |
|---|---|---|
| Enter | not Platform Admin / namespace missing | use a `credentials:write` token; create the namespace (J3) |
| Verify | provider key rejected / OAuth redirect mismatch / scope insufficient | re-issue the key; fix the registered redirect URI; grant required scopes |
| Stored | (v1–v5) plain-text storage risk | accepted per [AD-07](../../01-PRD.md); vault hardening lands in v6 |
| Runtime (later) | token expired mid-workflow | rotate/refresh; `CredentialExpiredError` surfaces to the Operator |

## Cross-references

- Decisions: [AD-07](../../01-PRD.md), [AD-08](../../01-PRD.md), [AD-09](../../01-PRD.md) (provider interface).
- Deep-dives: [09-security-model](../../deep-dive/09-security-model.md) (credential zone, MCP auth strategies, OAuth lifecycle E-11, rotation, scopes E-29, audit), [v3-tools-and-mcps](../../roadmap/v3-tools-and-mcps.md) (CredentialsProvider abstraction, per-run snapshot), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (namespace credential isolation), [39-llm-provider-sdk-matrix](../../deep-dive/39-llm-provider-sdk-matrix.md) (auth error mapping).
- Figma (worked example): `247:82` (Enter) · `248:81` (Verify) · `249:81` (Stored). Parent screen: Admin `57:56`, CREDENTIALS card `58:83`.
