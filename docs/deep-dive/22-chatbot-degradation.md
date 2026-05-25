# Chatbot Graceful Degradation and Circuit Breakers

## Purpose

This document specifies how the chatbot layer (v5) behaves when components fail partially. It closes a gap in `06-v5-chatbot-layer.md` and `13-chatbot-ux-edge-cases.md`: both documents handle individual edge cases (connection drops, task failures, human handoff), but neither specifies a **system-level degraded mode** — what the user experience looks like when the routing chatbot, a specialist, or the knowledge base is unavailable, and how the system protects itself from cascading failures.

Without this, a single failing specialist can stall the routing chatbot indefinitely, a flapping KB lookup can block every response, and there is no circuit that opens to protect the platform from a downstream dependency in freefall.

---

## Degradation Model

The chatbot system degrades along two axes:

| Axis | Healthy | Degraded | Failed |
|---|---|---|---|
| **Routing** | Classification succeeds in <200ms | Classification slow (>500ms) or low-confidence | Classification completely fails |
| **Specialist** | Responds and streams correctly | Slow or partial stream | Times out or errors repeatedly |
| **Knowledge base** | Returns relevant results in <500ms | Returns stale/empty results | Unavailable |
| **Background agent** | Starts and completes asynchronously | Delayed or retrying | Fails; orphan risk (E-10) |

The system must handle each degradation state without breaking the others.

---

## Routing Chatbot Degradation

### Fallback Routing

If the routing chatbot's classification LLM call fails or times out:

1. **First failure**: Retry the classification call once with a `max_tokens: 50` constrained prompt (faster, cheaper).
2. **Second failure**: Route to the **default specialist** — a configurable fallback defined in the routing chatbot's DSL:

```json
{
  "routing": {
    "default_specialist": "general-support",
    "classification_timeout_ms": 400,
    "classification_retry_on_timeout": true
  }
}
```

The default specialist is a catch-all that handles any topic. If no `default_specialist` is configured, the routing chatbot falls back to a static message (see Static Fallback below).

3. The classification failure is logged as a `routing_fallback` event in the conversation record and emits a Prometheus metric: `chatbot_routing_fallbacks_total{namespace, reason}`.

### Low-Confidence Routing

When classification succeeds but confidence is below the threshold (E-14's `<0.5` band):

- The routing chatbot acknowledges uncertainty and asks a clarifying question (configurable: `routing.low_confidence_behavior: clarify | default_specialist`).
- If `default_specialist` is chosen, the routing explanation is included in the handoff context so the specialist knows the routing was uncertain.

### Static Fallback Response

When both the classification and the default specialist are unavailable:

```json
{
  "routing": {
    "static_fallback_message": "We're experiencing technical difficulties. Please try again in a moment or contact us at support@acme.com."
  }
}
```

This is the last line of defense — a configurable plain-text message served with `HTTP 200` (not an error) so the client renders it normally.

---

## Specialist Circuit Breaker

### Problem

A specialist that fails repeatedly consumes retries, delays the user, and puts load on the routing chatbot. Without a circuit breaker, a flapping specialist causes a thundering herd of failing requests.

### Circuit Breaker Specification

Each specialist has a per-namespace circuit breaker. State machine:

```
CLOSED ──(threshold exceeded)──► HALF_OPEN ──(probe succeeds)──► CLOSED
                                      │
                               (probe fails)
                                      │
                                      ▼
                                    OPEN
```

| State | Behavior |
|---|---|
| `CLOSED` | Normal operation. Requests routed to specialist. |
| `OPEN` | Specialist is bypassed. All requests go to fallback. Counter resets on timer. |
| `HALF_OPEN` | One probe request sent to specialist. If succeeds, returns to `CLOSED`. If fails, returns to `OPEN`. |

### Circuit Breaker Configuration

Declared at the namespace level (not per-agent DSL, since it is an operational concern):

```json
{
  "circuit_breaker": {
    "failure_threshold": 5,
    "failure_window_seconds": 60,
    "open_duration_seconds": 30,
    "success_threshold_to_close": 2
  }
}
```

| Field | Description |
|---|---|
| `failure_threshold` | Number of failures in `failure_window_seconds` that trips the breaker. |
| `failure_window_seconds` | Rolling window for counting failures. |
| `open_duration_seconds` | How long the breaker stays open before moving to `HALF_OPEN`. |
| `success_threshold_to_close` | Consecutive successes in `HALF_OPEN` needed to close. |

A "failure" counts as: LLM call timeout, unhandled exception in the specialist, or response with HTTP 5xx from the chatbot endpoint.

### Fallback When Open

When the circuit is `OPEN` for a specialist:

1. The routing chatbot is notified that the specialist is unavailable (via an internal availability registry).
2. The routing chatbot routes to the `default_specialist` (if configured and not also open) or returns a specialist-specific degraded message:

```json
{
  "degraded_message": "Our billing team's assistant is temporarily unavailable. I'll take note of your question and connect you with them when they're back online — usually within a few minutes."
}
```

3. A `chatbot_circuit_open_total{namespace, specialist}` metric is emitted.
4. The Operator is alerted via the Prometheus alerting rule.

### Circuit Breaker State API

```
GET /namespaces/{ns}/chatbot/circuit-breakers

{
  "specialists": {
    "billing-specialist": { "state": "open", "open_since": "...", "failure_count": 7 },
    "technical-specialist": { "state": "closed", "failure_count": 0 },
    "general-support": { "state": "closed", "failure_count": 1 }
  }
}
```

The Operator CLI gains a `chatbot circuit-breakers` command that shows this table.

---

## Knowledge Base Degradation

### KB Circuit Breaker

The knowledge base tool (`knowledge_base_lookup`) uses the same circuit breaker pattern as specialists, configurable separately:

```json
{
  "knowledge_base_circuit_breaker": {
    "failure_threshold": 3,
    "failure_window_seconds": 30,
    "open_duration_seconds": 60
  }
}
```

### KB Fallback Behavior

When the KB circuit is `OPEN`:

| Strategy | Behavior | Configured Where |
|---|---|---|
| `skip` | Omit KB lookup; specialist responds from model knowledge only. | Specialist DSL: `knowledge_base.degraded_mode: skip` |
| `cached` | Use the most recent cached result for a semantically similar query (TTL-based, stored in Redis/memory). | Specialist DSL: `knowledge_base.degraded_mode: cached` |
| `block` | Block the specialist from responding; surface a degraded message. | Specialist DSL: `knowledge_base.degraded_mode: block` |

Default: `skip`. The `block` strategy is appropriate for specialists where responses without KB grounding are unacceptable (e.g., a legal-compliance specialist).

The KB degraded state is logged per turn for analysis: `kb_lookup_skipped: true` in the conversation turn record.

---

## Full System Degraded Mode

When both the routing chatbot and the default specialist are unavailable (both circuit breakers open), the platform enters **full degraded mode**:

1. All new chat sessions receive the `static_fallback_message` immediately (no LLM call attempted).
2. In-progress sessions receive: *"We're experiencing temporary issues. Your conversation is saved. We'll be back shortly."*
3. A `chatbot_full_degraded` Prometheus alert fires (severity: critical).
4. All human handoff endpoints are enabled automatically regardless of individual specialist configuration (because if the chatbot system is down, every conversation needs a human path).

Full degraded mode exits automatically when at least one specialist circuit closes.

---

## DSL Additions

The routing chatbot and specialist DSL definitions gain degradation configuration fields:

### Routing Chatbot

```json
{
  "agent_type": "chatbot",
  "routing": {
    "default_specialist": "general-support",
    "classification_timeout_ms": 400,
    "classification_retry_on_timeout": true,
    "static_fallback_message": "We're experiencing technical difficulties. Please try again shortly."
  }
}
```

### Specialist Chatbot

```json
{
  "agent_type": "chatbot",
  "degraded_message": "Our billing assistant is temporarily unavailable.",
  "knowledge_base": {
    "degraded_mode": "skip"
  }
}
```

These fields are optional. Specialists without `degraded_message` use the namespace-level default degraded message.

---

## Prometheus Metrics

| Metric | Labels | Description |
|---|---|---|
| `chatbot_routing_fallbacks_total` | `namespace`, `reason` | Routing fallbacks (classification failure, timeout) |
| `chatbot_circuit_open_total` | `namespace`, `specialist` | Times a circuit breaker opened |
| `chatbot_circuit_state` | `namespace`, `specialist`, `state` | Current state (gauge: 0=closed, 1=half_open, 2=open) |
| `chatbot_kb_degraded_turns_total` | `namespace`, `specialist` | Turns served without KB lookup due to KB circuit open |
| `chatbot_full_degraded_seconds_total` | `namespace` | Total seconds the system was in full degraded mode |

---

## Interaction with Human Handoff (v5 Epic 7)

Graceful degradation and human handoff are complementary. The cascade is:

```
Specialist circuit open ──► degraded_message displayed
                              │
                              ▼
              User signals they need help
                              │
                              ▼
              human_handoff triggered (webhook/message)
                              │
                              ▼
              If webhook fails (reliability spec E-17):
                              │
                              ▼
              static_fallback_message with contact info
```

The circuit breaker and human handoff are independent. A specialist can trigger human handoff even when its circuit is `CLOSED` (normal escalation). The circuit being `OPEN` is a *platform* signal — it causes the routing chatbot to bypass the specialist before the specialist even sees the request.

---

## Implementation Phase

| Feature | Phase |
|---|---|
| Static fallback message on routing failure | v5 (new story in Epic 2) |
| Default specialist routing fallback | v5 (new story in Epic 2) |
| Specialist circuit breaker | v5 (new story in Epic 3) |
| KB circuit breaker | v5 (new story in Epic 4) |
| Full degraded mode flag + alert | v5 (new story in Epic 1) |
| Circuit breaker Prometheus metrics | v6 (alongside other metrics — Story 4.2) |
| `chatbot circuit-breakers` CLI command | v6 (alongside CLI — Story 6.1) |

---

## What to Read Next

Graceful degradation sits at the intersection of chatbot design and platform reliability. Start with:

→ **[06-v5-chatbot-layer.md](./06-v5-chatbot-layer.md)** — The full chatbot layer specification: routing, specialists, knowledge base, human handoff, and the healthy-path behavior this document handles failures for.

Or explore related concerns:
- [13-chatbot-ux-edge-cases.md](./13-chatbot-ux-edge-cases.md) — Individual turn-level edge cases that complement the system-level degradation model here
- [21-conversation-memory.md](./21-conversation-memory.md) — Memory retrieval failures fall under the KB circuit breaker defined in this document
- [08-performance-and-scaling.md](./08-performance-and-scaling.md) — Failure mode impact table and latency budgets under degraded conditions
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Prometheus metrics and alerting (Story 4.2) that surface circuit breaker state

---

## Related Documents

- [06-v5-chatbot-layer.md](./06-v5-chatbot-layer.md) — Routing chatbot, specialist chatbots, human handoff
- [13-chatbot-ux-edge-cases.md](./13-chatbot-ux-edge-cases.md) — Individual UX edge cases and failure modes
- [08-performance-and-scaling.md](./08-performance-and-scaling.md) — Failure mode impact table
- [18-enhancements.md](./18-enhancements.md) — E-10 (fire-and-forget orphan), E-14 (routing confidence), E-17 (webhook reliability)
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Prometheus metrics (Story 4.2)
