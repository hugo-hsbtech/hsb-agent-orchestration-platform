# Conversation Memory Architecture

## Purpose

This document specifies the platform's approach to **conversational memory** beyond the immediate context window. It closes a critical gap: `06-v5-chatbot-layer.md` Epic 6 defines context window management (what fits in a single LLM call), but says nothing about how a chatbot accesses information from past conversations, recurring users, or structured user-level facts. Without this, every conversation starts from zero — a chatbot that forgets who you are each time is not a production support system.

This document defines three distinct memory tiers, their storage strategies, lifecycle policies, and integration points with the existing context window and knowledge base infrastructure.

---

## Memory Tier Model

Conflating all memory into "context" is the root of the gap. Three tiers serve different purposes:

| Tier | Time Horizon | What It Stores | Who Generates It | Storage |
|---|---|---|---|---|
| **Working memory** | Current conversation | Active turns, pending tasks | Platform (automatic) | `conversations` + `conversation_turns` tables |
| **Episodic memory** | Past conversations (days–weeks) | Summaries of prior sessions | Platform (on session end) | Qdrant (vector), keyed by user + namespace |
| **Semantic memory** | Persistent structured facts | User profile: name, preferences, account tier, open issues | Agent writes, admin imports | Application database, `user_profiles` table |

Working memory is already implemented via `06-v5-chatbot-layer.md` Epic 6. This document specifies episodic and semantic memory.

---

## Tier 1 — Working Memory (Existing, for Reference)

Managed by context window strategies (`sliding_window`, `summarize`, `semantic`) as specified in `06-v5-chatbot-layer.md` Story 6.1. No new work required.

**Important**: Working memory ends when the conversation ends. It does not persist across sessions without episodic memory.

---

## Tier 2 — Episodic Memory

### Concept

When a conversation ends (user closes the chat, session times out, or specialist signals conversation complete), the platform creates a **session summary** and writes it to the Qdrant knowledge base, tagged to the user and namespace. On subsequent conversations, the specialist chatbot retrieves relevant past sessions and injects them as context.

### Session Summary Generation

At conversation end, the platform invokes the `summary_agent` (reused from the context window `summarize` strategy, per E-15) with the full conversation transcript. The summary is:

- Maximum 500 tokens (configurable per namespace: `episodic_memory.max_summary_tokens`).
- Stored as a Qdrant point in a dedicated episodic memory collection per namespace.
- Tagged with: `user_id`, `conversation_id`, `ended_at`, `specialist_ids` (which specialists were active), `topics` (a short keyword list extracted by the summary agent).

### Qdrant Collection Schema

```
Collection: agent_platform_episodic_{namespace_slug}

Point metadata:
{
  "user_id":          "string",
  "conversation_id":  "uuid",
  "ended_at":         "ISO 8601",
  "specialist_ids":   ["billing-specialist", "technical-specialist"],
  "topics":           ["duplicate charge", "refund request"],
  "summary_tokens":   412
}

Vector: embedding of the summary text (same embedding model as KB)
```

### Retrieval at Conversation Start

When a new conversation turn arrives (first turn of a new session, after user identification):

1. The routing chatbot queries the episodic collection: `user_id == current_user AND ended_at >= (now - episodic_lookback_days)` with the current message as the semantic query.
2. Returns the top-K most relevant past session summaries (default K=3, configurable per namespace: `episodic_memory.top_k`).
3. Summaries are injected as a system-level preamble into the specialist's first turn: *"This user has interacted with the platform before. Relevant past sessions: [summaries]."*

### DSL Configuration

Episodic memory is enabled per specialist in the chatbot DSL:

```json
{
  "agent_type": "chatbot",
  "episodic_memory": {
    "enabled": true,
    "lookback_days": 30,
    "top_k": 3,
    "inject_on": "first_turn_per_session"
  }
}
```

| Field | Default | Description |
|---|---|---|
| `enabled` | `false` | Opt-in. Requires `user_id` to be present in the conversation context. |
| `lookback_days` | `30` | How far back to search for past sessions. |
| `top_k` | `3` | Maximum past session summaries to inject. |
| `inject_on` | `first_turn_per_session` | When to inject: `first_turn_per_session` or `every_specialist_handoff`. |

### User Identity Requirement

Episodic memory requires a stable `user_id` in the conversation's invocation context. Anonymous conversations (no `user_id`) do not generate or retrieve episodic memories. Authenticated user ID strategies are namespace-configurable: `session_token`, `external_user_id`, `email_hash`.

---

## Tier 3 — Semantic Memory (User Profiles)

### Concept

Episodic memory stores *what happened* in past conversations. Semantic memory stores *structured facts about the user* that do not change per conversation: their name, account tier, open support tickets, stated preferences, recent purchases. These are structured records, not text embeddings.

### `user_profiles` Table

```sql
CREATE TABLE user_profiles (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  namespace_id    UUID NOT NULL REFERENCES namespaces(id),
  user_id         TEXT NOT NULL,
  facts           JSONB NOT NULL DEFAULT '{}',
  source          TEXT NOT NULL,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE (namespace_id, user_id)
);

CREATE INDEX idx_user_profiles_ns_user ON user_profiles (namespace_id, user_id);
```

The `facts` JSONB object is schema-free per namespace. Example:

```json
{
  "name": "Alice Rodriguez",
  "account_tier": "premium",
  "language": "en",
  "open_ticket_count": 2,
  "last_purchase_date": "2026-03-12",
  "preferred_contact": "chat",
  "do_not_call": true
}
```

`source` values: `crm_import`, `agent_write`, `user_self_declared`.

### Writing to Semantic Memory

Agents can write facts to a user's profile via the **`user_profile_update` registered tool**:

```json
{
  "tool": "user_profile_update",
  "inputs": {
    "user_id": "$.workflow.input.user_id",
    "facts": {
      "preferred_contact": "email",
      "last_issue_type": "billing"
    },
    "source": "agent_write"
  }
}
```

The tool merges the provided facts into the existing `facts` JSON (shallow merge; nested objects require explicit paths). Overwrites are logged in the audit trail.

**Authorization**: Only agents with the `user_profile:write` scope (granted by Platform Admin) may call `user_profile_update`. The default specialist chatbot template does not have this scope — it must be explicitly granted.

### Reading from Semantic Memory

The `user_profile_lookup` registered tool:

```json
{
  "tool": "user_profile_lookup",
  "inputs": {
    "user_id": "$.workflow.input.user_id"
  }
}
```

Returns the full `facts` object or an empty object if no profile exists. Available to all agents with `user_profile:read` scope (default: granted to all chatbots).

### Injection at Conversation Start

The routing chatbot can automatically inject user profile facts into the conversation context before routing to a specialist. This is configured in the routing chatbot's DSL:

```json
{
  "semantic_memory": {
    "auto_inject": true,
    "inject_fields": ["name", "account_tier", "language", "preferred_contact"],
    "inject_as": "system_note"
  }
}
```

When `auto_inject: true`, the routing chatbot calls `user_profile_lookup` on every new conversation and appends the selected fields as a system note: *"User profile: name=Alice Rodriguez, tier=premium, language=en."*

### CRM Import

Platform Admins can bulk-import user profiles from external CRM systems:

```
POST /namespaces/{ns}/user-profiles/import
Content-Type: application/json

[
  { "user_id": "U-1234", "facts": { "name": "Alice", "tier": "premium" }, "source": "crm_import" },
  ...
]
```

The import is idempotent: existing profiles are merged with imported facts. Source `crm_import` facts are overwritten on re-import (last-write-wins per source).

---

## Privacy and Data Lifecycle

### GDPR Right to Erasure

The existing right-to-erasure endpoint (E-16, `09-security-model.md`) is extended to cover all memory tiers:

```
DELETE /namespaces/{ns}/users/{user_id}/data
```

This operation:
1. Zeroes `raw_message` columns in `conversation_turns` (working memory).
2. Deletes all Qdrant points for `user_id` in the episodic collection.
3. Deletes the `user_profiles` row for `user_id`.
4. Retains anonymized conversation metadata (IDs, timestamps, specialist routing) for audit.

The operation is asynchronous (Qdrant deletion is eventually consistent). A `DELETE` status endpoint confirms completion.

### Retention Policies

| Memory Tier | Default Retention | Configurable |
|---|---|---|
| Working memory (conversation turns) | 90 days | Yes, per namespace |
| Episodic memory (session summaries) | 365 days | Yes, `episodic_memory.retention_days` |
| Semantic memory (user profiles) | Indefinite | Manual deletion or GDPR request |

Retention enforcement runs as a nightly background job.

### PII in Memory

User profiles frequently contain PII. The PII handling policy (E-16, `09-security-model.md`) applies:
- `facts` JSONB values are treated as potentially PII-containing.
- The `user_profile_update` tool is subject to the namespace's PII detection policy.
- The raw `facts` object is not logged; only the field names written are logged for audit.

---

## Implementation Phase

| Feature | Phase | Notes |
|---|---|---|
| Episodic memory (Qdrant, session summaries) | v5 (new story in Epic 6) | Reuses KB tool infrastructure from Story 4.1 |
| Semantic memory (`user_profiles` table, tools) | v5 (new story in Epic 6) | Schema established in v1's migration framework |
| User profile auto-injection in routing chatbot | v5 | Story in Epic 2 (routing chatbot) |
| GDPR erasure across all memory tiers | v5 (new story in Epic 1) | Extends Story 1.2 (conversation persistence) |
| CRM import API | v6 | Low priority; admin workflow |

---

## What to Read Next

Memory tiers build on the chatbot and knowledge base infrastructure. Start here:

→ **[06-v5-chatbot-layer.md](./06-v5-chatbot-layer.md)** — The full chatbot layer specification: streaming, routing, specialist configuration, and working memory (Epic 6) that this document extends.

Or explore supporting systems:
- [09-security-model.md](./09-security-model.md) — PII handling policy and the right-to-erasure flow this document extends
- [14-dsl-spec.md](./14-dsl-spec.md) — Where `episodic_memory` and `semantic_memory` DSL fields are formally specified
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace scoping for `user_profiles` and episodic Qdrant collections
- [22-chatbot-degradation.md](./22-chatbot-degradation.md) — What happens when memory retrieval fails (KB circuit breaker applies to episodic lookups)

---

## Related Documents

- [06-v5-chatbot-layer.md](./06-v5-chatbot-layer.md) — Context window management (working memory), KB integration
- [09-security-model.md](./09-security-model.md) — PII handling, GDPR right to erasure
- [14-dsl-spec.md](./14-dsl-spec.md) — Chatbot DSL fields (`episodic_memory`, `semantic_memory`)
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace scoping for user profiles
- [18-enhancements.md](./18-enhancements.md) — E-15 (context window summarize), E-16 (PII handling)
