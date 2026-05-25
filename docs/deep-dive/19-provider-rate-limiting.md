# Provider Rate Limiting and Throttle Management

## Purpose

This document specifies how the Agent Platform handles LLM provider rate limits. It is distinct from provider *failover* (E-27 / `01-PRD.md`) — failover addresses provider *unavailability*; this document addresses provider *throttling*, which is far more common in production and requires its own detection, queuing, and back-pressure strategy.

This fills a gap identified in review: `08-performance-and-scaling.md` lists `llm_call` latency budgets and retry strategies but contains no rate-limit-specific policy. `14-dsl-spec.md`'s retry block does not distinguish rate-limit errors from transient errors. Without an explicit model, every team implements ad-hoc 429 handling, leading to thundering-herd retries that make the problem worse.

---

## Problem Statement

Each LLM provider enforces rate limits along two dimensions:

| Dimension | Description | Common Limit Shape |
|---|---|---|
| **Requests per minute (RPM)** | Number of API calls within a rolling minute window | e.g., 1,000 RPM for Claude Sonnet |
| **Tokens per minute (TPM)** | Total input + output tokens within a rolling minute | e.g., 800,000 TPM for Claude Sonnet |

When a limit is breached, the provider returns a `429 Too Many Requests` response, often with a `Retry-After` header indicating when the client may retry.

### Why Generic Retry Is Insufficient

The platform's existing `retry` block (AD-14) applies uniform exponential backoff to all retryable errors. This is wrong for rate limits because:

1. **Thundering herd**: If 50 `llm_call` activities all hit a rate limit simultaneously and each backs off independently with jitter, they reconverge and re-trigger the limit in the next window.
2. **TPM budget blindness**: Retrying a 4,000-token request three times consumes 12,000 tokens — which may itself exceed the TPM budget, causing further 429s.
3. **`Retry-After` is ignored**: Providers signal exactly when the client should retry. Generic backoff ignores this signal, either waiting too long (waste) or not long enough (immediate re-limit).

---

## Architecture

### Rate Limit Handling Layers

```
┌──────────────────────────────────────────────────┐
│              DSL / Agent Builder                  │
│  Declares: provider, model, max_tokens per step  │
└──────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│          Pre-Flight Token Budget Check            │
│  Before workflow start: estimate token demand     │
│  vs. namespace's remaining TPM allowance.         │
│  Defer start if demand exceeds current window.    │
└──────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│         Provider Adapter (per-provider)           │
│  Detects 429 responses and Retry-After headers.   │
│  Classifies: RateLimitError (retryable) vs        │
│  QuotaExceededError (non-retryable, daily limit). │
└──────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│       Provider Rate Limit Token Bucket            │
│  Per-provider, per-namespace in-process bucket.   │
│  Drains at the provider's published RPM/TPM rate. │
│  Activities acquire a token before calling the    │
│  provider; block (with timeout) if exhausted.     │
└──────────────────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────┐
│            Temporal Activity Retry                │
│  RateLimitError → retry with Retry-After wait.   │
│  QuotaExceededError → non-retryable, fail step.  │
└──────────────────────────────────────────────────┘
```

---

## Provider Adapter Contract: Rate Limit Error Classification

Every provider adapter must distinguish the following error classes on a `429` response:

| Error Class | Meaning | Retry Behavior |
|---|---|---|
| `RateLimitError` | Transient per-minute window exceeded | Retryable; respect `Retry-After` header |
| `QuotaExceededError` | Daily or monthly hard quota exhausted | Non-retryable; alert Platform Admin |
| `ConcurrencyLimitError` | Too many simultaneous requests (provider-specific) | Retryable with reduced concurrency |

The adapter inspects the response body and headers to classify. If classification is ambiguous, default to `RateLimitError` (safer).

### `Retry-After` Handling

When the provider returns a `Retry-After` header:

1. The adapter extracts the wait duration (seconds or HTTP-date format).
2. It returns a `RateLimitError` with `retry_after_seconds` populated.
3. The Temporal activity sets its next retry delay to `max(retry_after_seconds, initial_interval_seconds)`, overriding the DSL's backoff for this attempt only.
4. The step run record stores `rate_limited_at` and `retry_after_seconds` for observability.

---

## Token Bucket: Per-Provider, Per-Namespace Rate Limiter

### Purpose

Before any activity calls an LLM provider, it acquires a permit from a token bucket that enforces the provider's published rate limits. This prevents thundering-herd retries by serializing concurrent requests at the platform level.

### Configuration

Rate limit budgets are configured in the platform's provider registry (not per-agent DSL):

```json
{
  "provider": "claude",
  "rate_limits": {
    "requests_per_minute": 1000,
    "tokens_per_minute": 800000,
    "burst_multiplier": 1.2
  }
}
```

`burst_multiplier` allows short bursts above the sustained rate (matches provider token bucket behavior). Default: `1.0` (no burst).

Namespace-level overrides allow a single namespace to be capped below the platform total:

```json
{
  "namespace_id": "acme-corp",
  "provider": "claude",
  "max_rpm_share": 200,
  "max_tpm_share": 150000
}
```

### Behavior Under Exhaustion

When the bucket is exhausted:

1. The activity waits (blocks) for up to `rate_limit_wait_timeout_seconds` (default: `60`, configurable in platform settings).
2. If the bucket refills within the timeout, the call proceeds — no 429 is ever sent to the provider.
3. If the timeout expires without refill, the activity returns a `RateLimitError` and Temporal retries per the step's retry policy.
4. This wait is transparent to the DSL — the step's `timeout_seconds` (E-07) counts total elapsed time including the wait.

### Prometheus Metrics

| Metric | Labels | Description |
|---|---|---|
| `agent_platform_rate_limit_waits_total` | `namespace`, `provider`, `model` | Number of times an activity waited for the token bucket |
| `agent_platform_rate_limit_wait_duration_seconds` | `namespace`, `provider` | Duration of bucket wait (histogram) |
| `agent_platform_provider_429s_total` | `namespace`, `provider`, `error_class` | 429 responses received after bucket miss |

---

## Pre-Flight Token Budget Check

### Motivation

A workflow with ten `llm_call` steps, each declaring `max_tokens: 4096`, has a worst-case demand of 40,960 tokens per run. If a namespace's TPM share is 30,000, launching this workflow guarantees mid-run rate limiting.

### Mechanism

At workflow invocation time (before the first Temporal activity), the API performs a pre-flight check:

1. Sum the `max_tokens` declared on all `llm_call` steps in the workflow.
2. Query the namespace's current TPM window consumption from the rate limit counter.
3. If `current_consumption + worst_case_demand > tpm_share`, one of two behaviors applies (namespace-configurable):

| Mode | Behavior |
|---|---|
| `defer` | Reject the invocation with `HTTP 429` and a `Retry-After` indicating the next window. |
| `warn_only` | Allow the invocation but emit a `PotentialRateLimitRisk` warning metric. |

Default mode: `warn_only` (conservative; production namespaces should configure `defer`).

### DSL Declaration

No DSL changes are required. The check uses the existing `model_config.max_tokens` field per `llm_call` step. When E-04's `input_schema` is implemented, the pre-flight check can use declared output schemas to refine the token estimate.

---

## DSL: Rate Limit Policy per Step

The step-level `retry` block gains two optional rate-limit-specific fields:

```json
{
  "retry": {
    "max_attempts": 5,
    "initial_interval_seconds": 2,
    "backoff_coefficient": 2.0,
    "non_retryable_errors": ["QuotaExceededError", "ValidationError"],
    "rate_limit_max_attempts": 3,
    "rate_limit_honor_retry_after": true
  }
}
```

| Field | Type | Default | Description |
|---|---|---|---|
| `rate_limit_max_attempts` | integer | same as `max_attempts` | Separate retry cap specifically for `RateLimitError`. Allows more retries for rate limits than for other errors. |
| `rate_limit_honor_retry_after` | boolean | `true` | When `true`, the activity uses the provider's `Retry-After` header to set the retry delay. When `false`, uses the standard backoff. |

### Validation Service Rules

- The validation service **warns** when `rate_limit_max_attempts > 10` (excessive retries amplify cost).
- The validation service **errors** when `non_retryable_errors` includes `RateLimitError` (prevents retry of a retryable condition, which would silently drop work).

---

## Observability

### Structured Log Fields

Every `llm_call` activity adds the following fields to its structured log output:

| Field | Description |
|---|---|
| `rate_limited` | `true` if the call received a 429 |
| `rate_limit_wait_ms` | Time spent waiting in the token bucket |
| `retry_after_seconds` | Value from provider `Retry-After` header (null if absent) |
| `tokens_requested` | Declared `max_tokens` for this call |
| `tokens_used` | Actual token consumption (from provider response) |

### Grafana Dashboard Additions

The v6 starter Grafana dashboard (`07-v6-versioning-and-hardening.md` Story 4.2) should include a **Rate Limit Health** panel showing:
- RPM/TPM consumption vs. configured limits per provider per namespace (gauge)
- Rate limit wait duration p95 over time (line chart)
- 429 rate by provider (bar chart)

---

## Interaction with Provider Failover (E-27)

Rate limiting and failover interact at one point: when the primary provider is rate-limiting and a `fallback_provider` is declared (E-27), the platform **should attempt the fallback** rather than waiting in the primary's rate limit queue, if:

1. `RateLimitError` is listed in `fallback_provider.trigger_on`, AND
2. The fallback provider's token bucket has capacity.

This is the recommended configuration for production deployments with cost-sensitive SLAs. When the fallback is used due to rate limiting, this is recorded in the step run record with `provider_used: "openai"` and `failover_reason: "rate_limit"`.

---

## Implementation Phase

| Component | Phase | Story |
|---|---|---|
| Error classification in provider adapters | v3 | Alongside tool-calling (same provider adapter work) |
| Token bucket per-provider per-namespace | v3 | New story in Epic 4 |
| Pre-flight TPM check | v4 | Alongside multi-step workflow complexity |
| Prometheus metrics | v6 | Alongside other observability work (Story 4.2) |
| Rate-limit-specific DSL retry fields | v3 | Extend `14-dsl-spec.md` retry block |

---

## Out of Scope

- **Per-model rate limit configuration**: The platform treats all models within a provider as sharing the provider's limit. Per-model limits (where providers differ between models) are a future refinement.
- **Distributed token bucket across API instances**: v1–v6 uses a single in-process bucket per worker. For deployments with many workers, a Redis-backed distributed bucket is a future optimization (same interface, different backend).
- **User-level rate limiting**: This document covers provider-facing limits. User-facing API rate limiting (throttling End Users or Agent Builders) is addressed in `09-security-model.md`.

---

## What to Read Next

Rate limiting is part of the broader provider reliability story. Continue with:

→ **[18-enhancements.md — E-27](./18-enhancements.md)** — Provider failover policy: when rate limiting triggers a switch to a fallback provider.

Or explore related concerns:
- [14-dsl-spec.md](./14-dsl-spec.md) — The `retry` block where `rate_limit_max_attempts` and `rate_limit_honor_retry_after` are declared
- [08-performance-and-scaling.md](./08-performance-and-scaling.md) — Latency budgets per step type and how rate-limit wait affects them
- [23-per-run-cost-cap.md](./23-per-run-cost-cap.md) — Per-run cost cap: the complementary safety valve for token consumption
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Prometheus metrics and Grafana dashboards (Story 4.2)

---

## Related Documents

- [01-PRD](./01-PRD.md) — AD-09 (provider-agnostic interface), AD-14 (retry policy per step), AD-28 (this decision)
- [08-performance-and-scaling.md](./08-performance-and-scaling.md) — Latency budgets by step type
- [14-dsl-spec.md](./14-dsl-spec.md) — Retry block specification
- [18-enhancements.md](./18-enhancements.md) — E-27 (provider failover), E-07 (step timeout), E-06 (provider check)
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Prometheus metrics (Story 4.2)
