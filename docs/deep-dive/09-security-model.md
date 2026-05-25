# Security Model

## Purpose

This document defines the conceptual security architecture of the agent platform. It establishes trust boundaries, threat models, and security principles without prescribing specific implementation mechanisms (e.g., particular vault technologies or encryption algorithms).

---

## Security Principles

### 1. Defense in Depth

Security is not a single boundary but layered:
- **Edge**: API authentication and rate limiting
- **Orchestration**: Workflow isolation and step sandboxing
- **Execution**: Tool/MCP invocation constraints
- **Data**: Credential abstraction and minimal persistence

### 2. Least Privilege

Each component operates with the minimum access required:
- Workers access only the workflow runs they process
- Tools execute with scoped credentials (per-MCP, per-tool)
- Chatbots access only their designated knowledge base filters
- Operators see only the operational data relevant to their role

### 3. Fail Closed

Security failures default to denial:
- Unvalidated DSL documents are rejected, not partially executed
- Missing credentials halt workflow start, not mid-execution
- Suspicious MCP responses terminate the connection
- Unrecognized tool calls fail rather than fallback

---

## Trust Boundaries

```
┌─────────────────────────────────────────────────────────┐
│                    UNTRUSTED ZONE                        │
│  (End User browsers, third-party chat integrations)      │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                    EDGE ZONE                             │
│  (API Gateway, Authentication, Rate Limiting)            │
│  Trust assumption: Validated identity, bounded requests  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                  ORCHESTRATION ZONE                      │
│  (Temporal workflows, Agent Core, Tool Registry)       │
│  Trust assumption: Internal services, authenticated    │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 CREDENTIAL ZONE (Vault)                │
│  (Secrets, API keys, OAuth tokens)                     │
│  Trust assumption: Vault is the sole credential source   │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                   EXTERNAL ZONE                          │
│  (LLM providers, MCP servers, third-party APIs)          │
│  Trust assumption: No trust; validate all responses      │
└─────────────────────────────────────────────────────────┘
```

---

## Threat Model

### Threat: Credential Extraction via Prompt Injection

**Scenario**: A malicious user crafts input that causes an agent to reveal its system prompt or tool credentials.

**Mitigation**:
- System prompts are immutable and not exposed in `user_prompt_template` interpolation
- Tool credentials never appear in LLM context; the Agent Core injects them at invocation time
- Input sanitization layer validates user input against known injection patterns

### Threat: Sub-Agent Escape

**Scenario**: A compromised or misconfigured sub-agent escalates privileges by calling agents outside its intended scope.

**Mitigation**:
- Agent call graphs are validated at provisioning time (cycle detection, depth limits)
- Per-agent access control lists define which other agents may be invoked
- Fire-and-forget calls are logged and auditable; no implicit trust inheritance

### Threat: MCP Impersonation

**Scenario**: An attacker registers a malicious MCP or intercepts MCP communication.

**Mitigation**:
- MCP registrations require Platform Admin approval
- MCP manifest caching prevents runtime substitution
- Tool discovery results are validated against cached schemas
- MCP calls carry idempotency keys derived from step run IDs

### Threat: Workflow Run Enumeration

**Scenario**: An attacker scans workflow run IDs to discover other users' agent executions.

**Mitigation**:
- Workflow run IDs are non-sequential UUIDs
- Query endpoints enforce tenant isolation
- Operator access is logged and alerted

### Threat: Resource Exhaustion

**Scenario**: An attacker submits expensive workflows (deep recursion, massive fan-out) to degrade platform performance.

**Mitigation**:
- Per-agent concurrency quotas
- Depth limits on sub-agent calls
- Fan-out concurrency bounds in DSL
- Circuit breakers on MCP calls

---

## Input Validation and Sanitization

### DSL Validation (Provisioning Time)

All agent definitions undergo validation before persistence:
- **Schema compliance**: Required fields, type correctness
- **Reference integrity**: Tools, MCPs, and sub-agents exist and are accessible
- **Data flow validation**: Step inputs reference available outputs
- **Security constraints**: No credential literals in DSL; only credential references

### Runtime Input Validation

User-provided inputs to agent invocations:
- Size limits (prevent prompt bomb attacks)
- Structure validation against declared input schemas
- Content sanitization (strip or escape known injection patterns)

### LLM Output Handling

Responses from LLM providers:
- Schema validation when output format is declared
- Tool call validation against registered tool schemas
- Max token enforcement to prevent unbounded response processing

---

## Authentication and Authorization

### Persona-Based Access Control

| Persona | Create Agents | Invoke Agents | View Runs | Manage Credentials | Debug Workflows |
|---|---|---|---|---|---|
| **Platform Admin** | Yes | Yes | All | All | Yes |
| **Agent Builder** | Own agents | Test own | Own agents | Own agent scopes | Own agents |
| **End User** | No | Via chatbot | Own conversations | No | No |
| **Operator** | No | No | All | No | Yes |

**Note**: This is a conceptual model. Implementation may map to RBAC, ABAC, or custom authorization systems.

### Workflow Run Isolation

- Workflow runs are isolated by tenant/namespace
- Cross-tenant agent calls are prohibited (or explicitly allowlisted)
- Sub-agent calls inherit the parent run's security context

### MCP Authentication Strategies

MCPs declare their authentication model:
- **API Key**: Key provided by Platform Admin, injected in headers
- **OAuth 2.0**: Refresh token stored; access token fetched at workflow start
- **Basic Auth**: Credentials stored encrypted, injected at invocation
- **None**: Public MCPs with no authentication

**Principle**: Agent definitions contain no raw credentials. Only references to credential entries managed by Platform Admins.

### OAuth Token Lifecycle for MCPs (E-11)

Two OAuth modes are supported:

**Service-account OAuth** (v3+): A refresh token is stored in the credential store by the Platform Admin via an out-of-band flow. The `CredentialsProvider` exchanges it for an access token at workflow start and stores it in the workflow context snapshot. If the access token TTL is less than the workflow's estimated runtime, the provider pre-fetches a refreshed token and stores both.

**User-delegated OAuth** (v5+): The chatbot session carries a per-user OAuth token (obtained via the chatbot layer's auth flow). When a specialist chatbot delegates to a backend workflow agent, the token is injected into the workflow context.

**Mid-workflow token expiry**: If a step receives an auth error from an MCP, the activity retries with a token refresh before propagating the error. Token refresh is an atomic operation on the workflow context (one refresh attempt per workflow, not per step). Refresh failure is classified as `CredentialExpiredError` which is `non_retryable` — it fails the step immediately and surfaces to the Operator.

### Authorization Scope Contract (E-29)

Every API token carries at minimum `{ namespace_id, persona, scopes[] }`. The `scopes` array is the enforcement primitive. Canonical scope list:

| Scope | Description | Persona(s) |
|---|---|---|
| `agents:read` | Read agent definitions and versions | Agent Builder, Operator, Platform Admin |
| `agents:write` | Create/update agent definitions | Agent Builder, Platform Admin |
| `runs:read` | Read workflow run records and step outputs | Agent Builder (own), Operator (all), Platform Admin |
| `runs:write` | Invoke workflow runs | Agent Builder, End User (via chatbot), Platform Admin |
| `runs:control` | Pause, resume, cancel runs | Operator, Platform Admin |
| `credentials:read` | List credential entries (no values) | Platform Admin |
| `credentials:write` | Create/update/delete credential entries | Platform Admin |
| `namespaces:admin` | Cross-namespace super-scope | Platform Admin only |

**Enforcement**: Authorization is enforced in a single middleware layer at the API edge. Per-handler permission checks are not permitted — all authorization logic lives in one auditable place.

**Namespace super-token**: Platform Admin tokens carry `namespace_id: "*"` (wildcard). All namespace isolation checks treat `"*"` as matching any namespace.

**Technology decision gate**: The specific implementation (JWT with scope claims, opaque tokens with lookup table, OPA, etc.) is decided before v1 API implementation begins and recorded as an update to AD-27 in `01-PRD.md`. The scope list above is the contract regardless of implementation.

---

## Audit and Non-Repudiation

### Immutable Audit Trail

The following events are persistently logged:
- Agent definition create, update, deprecate (v6)
- Workflow run start, step completion, failure, cancellation
- Credential access (read, not value)
- MCP manifest fetch and cache invalidation
- Chatbot conversation turns (content may be pseudonymized per policy)

### Tamper Evidence

- Workflow run records are append-only; no updates to completed steps
- Agent versions are immutable
- Audit logs are write-once, with integrity checks (hash chain or external log aggregation)

---

## Incident Response

### Workflow Cancellation (Operator Override)

Operators may cancel workflows that exhibit suspicious behavior:
- Cancellation is immediate (where safe) or at next checkpoint
- Cancellation reason is logged
- Child workflows are cancelled according to policy (cascade or orphan)

### Version Rollback

When a security issue is discovered in an agent version:
- The version is marked deprecated (blocks new invocations)
- Existing runs continue (immutable version guarantee)
- Platform Admin notifies affected Agent Builders

### Secret Rotation

Credential rotation strategy:
- New secrets are added to the vault
- Existing workflows continue with their snapshot (no interruption)
- New workflows fetch updated credentials
- Old secrets are revoked after confirmation of no active references

---

## PII Handling Policy (E-16)

### Detection Layer

PII scanning runs on user input **before** it reaches the routing chatbot's LLM call. Detected PII categories (configurable per namespace):
- `CREDIT_CARD`, `SSN`, `EMAIL`, `PHONE`, `IBAN`, `FULL_NAME` (optional)

Namespace-level PII policy: `settings.pii_detection_categories` declares which categories are active.

### Two-Track Persistence

The `conversation_turns` table stores two message forms:
- `raw_message` (encrypted column, key managed per namespace): the unredacted original input.
- `message`: the redacted version — what the LLM and all downstream systems see.

Raw storage is configurable: operators may opt out and store only redacted turns.

### Right to Erasure

The endpoint `DELETE /namespaces/{ns}/users/{user_id}/data` covers all memory tiers (E-32):
1. Zeroes `raw_message` columns in `conversation_turns`.
2. Replaces `message` with `[USER_DATA_DELETED]` for all affected turns.
3. Deletes all Qdrant episodic memory points for the user.
4. Deletes the `user_profiles` row.

The conversation record (ID, timestamps, metadata) is retained for audit. Audit log entries are never erased.

## Inter-Agent Data Propagation Threat (E-21)

**Threat**: A malicious content artifact (fetched via `tool_call`/`mcp_call`/`fetch` or produced by a compromised sub-agent) propagates through the workflow graph as trusted step output and is injected into a downstream agent's LLM context. This is the multi-agent equivalent of second-order prompt injection.

**Mitigations**:

1. **`trust_level` step output annotation**: Steps producing data from external sources are tagged `external` by default. Downstream `llm_call` steps receiving `external`-tagged data are given an injected system note instructing the model to treat that content as untrusted.

   | Step type | Default `trust_level` |
   |---|---|
   | `tool_call`, `mcp_call`, `fetch` | `external` |
   | `llm_call`, `transform`, `noop` | `internal` |
   | `agent_call` | `internal` (platform-trusted artifact) |

2. **Agent boundary control**: A namespace-level setting can mark specific external-facing agents as `trust_level: external`, causing their `agent_call` output to be treated as untrusted by receivers.

3. **Validation warning**: The validation service warns when an `llm_call` step receives data from a chain that includes `external` sources without an intervening `transform` (sanitization) step.

## Compliance Considerations

### Data Residency

Workflow run data and conversation logs may have residency requirements:
- Conceptual support for region-specific persistence
- MCP calls respect data residency (MCP endpoint selection)

### Retention Policies

- Workflow runs: Configurable retention with automatic archival
- Conversation logs: Pseudonymization options, user deletion requests
- Audit logs: Extended retention, immutable storage

### LLM Provider Data Handling

Different providers have different data usage policies:
- Agent definitions may specify provider data handling tier
- Platform respects provider-specific retention and training opt-outs

---

## Out of Scope (Implementation Detail)

The following are deferred to implementation phases:
- Specific encryption algorithms and key management
- Vault technology selection (HashiCorp Vault, AWS Secrets Manager, etc.)
- OAuth implementation specifics (authorization code flow, PKCE)
- Specific authorization implementation (JWT, opaque tokens, OPA) — scope contract is defined above (E-29); technology selected before v1 API build
- Network topology (VPCs, private endpoints, mTLS details)
- SIEM integration specifics

---

## What to Read Next

Security spans all phases of the platform:

- **[07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md)** — Vault migration and production security
- **[04-v3 — Tools and MCPs](./04-v3-tools-and-mcps.md)** — MCP authentication and credential pipeline
- **[10-Deployment Concepts](./10-deployment-concepts.md)** — Network segmentation and secret injection
- **[13-Chatbot UX Edge Cases](./13-chatbot-ux-edge-cases.md)** — PII handling and content safety

Or explore:
- [01-PRD](./01-PRD.md) — Security-related ADs (AD-07, AD-08, AD-16)

---

## Related Documents

- [01-PRD](./01-PRD.md) — AD-07 (vault migration), AD-08 (credentials context), AD-16 (immutable versioning)
- [04-v3 — Tools and MCPs](./04-v3-tools-and-mcps.md) — MCP authentication abstraction
- [07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md) — secrets vault migration
