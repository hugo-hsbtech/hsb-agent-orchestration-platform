# J5 — Trigger Creation

> **Entry:** `New trigger` on Scheduling (`67:56`, button `67:284`). **Figma worked examples:** nightly `kb-refresh` cron (`0 2 * * *`) + `ticket-created` webhook (`https://api.acme.dev/hooks/tj_9f3a2c`) + Linear `issue.created` event — frames `239:62` → `244:81`. **Grounds:** [29-workflow-scheduling-and-events](../../deep-dive/29-workflow-scheduling-and-events.md) (trigger types, config, delivery, observability), [09-security-model](../../deep-dive/09-security-model.md) (webhook auth), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (namespace scope), [AD-16](../../01-PRD.md) (version pinning).

Triggers are **first-class resources, separate from agents** ([29](../../deep-dive/29-workflow-scheduling-and-events.md) §Trigger Configuration) — they invoke a target agent + pinned version with an input mapping. A trigger run creates a `workflow_runs` row with `triggered_by` metadata and the same observability as API runs. The 3-step wizard: **Type → Configure → Review & Activate**.

---

## Step 1 · Choose type

The **trigger-type branch** (3 cards). Each leads to a different Configure form. The platform docs also list Database CDC and File/storage trigger types ([29](../../deep-dive/29-workflow-scheduling-and-events.md) §Trigger Types); the wizard surfaces the three primary ones (Event subsumes message-bus/CDC/file via source selection — *design assumption that CDC/file are modelled as Event sources*).

| Type | Pattern | `trigger_type` |
|---|---|---|
| **Cron** | time-based, cron expression | `scheduled` |
| **Webhook** | HTTP POST to a generated endpoint | `webhook` |
| **Event** | message-queue / event-bus subscription | `event_subscription` |

Common to all: **target agent** + **version pin** (`latest` or a specific version, [AD-16](../../01-PRD.md)) + **input mapping** (`{{trigger.…}}` → agent input).

## Step 2 · Configure (per-branch)

### Branch A — Cron

| Field | Values |
|---|---|
| Preset chips | Hourly / Daily / Weekly / Monthly (or custom) |
| Expression | 5-field Unix cron (mono, e.g. `0 2 * * *`) |
| Next-run preview | computed (e.g. "tomorrow 02:00 UTC") |
| Timezone | e.g. `UTC`, `America/New_York` |
| Start / end dates *(optional)* | limited-run window |
| Target agent + version | e.g. `kb-refresh` + Production badge `v3` |
| Input payload | JSON (e.g. `{}`) |
| Missed-execution policy | `run_once` / `skip` / `run_all` (what to do for executions missed during downtime) |

- **Necessary actions:** the cron expression is validated at creation; the platform's distributed scheduler (Temporal scheduled workflows) must be running and HA (clustered, leader election) for schedules to fire ([29](../../deep-dive/29-workflow-scheduling-and-events.md) §Scheduler Implementation). The target agent + pinned version must exist.
- **Validation:** invalid cron rejected at creation; overlap/conflict handled by the **conflict policy** (Review step) — `skip` (default) / `queue` / `parallel` / `terminate`.

### Branch B — Webhook

| Field | Values |
|---|---|
| Generated endpoint URL | `https://platform.example.com/webhooks/{namespace}/{trigger_id}` — stable across redeploys (e.g. `https://api.acme.dev/hooks/tj_9f3a2c`) |
| Signing secret | masked (e.g. `whsec_••••••••`), reveal/copy |
| Auth type | `hmac_signature` (header + algorithm + secret ref) / `api_key` / `mtls` |
| Target agent | e.g. `triage` |
| Payload mapping | `$.issue.id → input.ticket_id`, etc. (transform: jsonata/jq/javascript) |
| Payload schema *(optional)* | JSON Schema for validation after transform |

- **Necessary actions:** the **source system must be pointed at the generated URL** and configured with the **same signing secret** (the secret must be stored as a credential/`secret_ref`, J6); the platform must be internet-reachable at the webhook host. HMAC verification is the standard; invalid auth returns `401`, valid fires the workflow ([29](../../deep-dive/29-workflow-scheduling-and-events.md) §Story 2.1).
- **Validation / failure:** transform failures are logged and the trigger fails gracefully; idempotency keys prevent duplicate processing; repeated delivery failures move events to a **dead-letter queue** (manual replay supported). Rate limiting + payload-size limits apply per endpoint ([29](../../deep-dive/29-workflow-scheduling-and-events.md) §Webhook Security).

### Branch C — Event subscription

| Field | Values |
|---|---|
| Event source / broker | `kafka` / `rabbitmq` / `sqs` / `pubsub` / Azure Event Hub (worked example uses source `linear`) |
| Connection | broker connection + auth ref (e.g. `sasl_config_ref`); for SaaS sources, the provider connection |
| Topic | e.g. `issue.created` |
| Consumer group | for queue brokers |
| Filter | event-type / attribute filter (e.g. `team = support`) |
| Target agent | e.g. `triage` |

- **Necessary actions:** broker connectivity + credentials must be configured; the consumer group must be coordinated; for SaaS event sources the upstream integration (e.g. Linear) must be emitting the topic. Optional schema-registry integration (Confluent/Glue) validates event shape before triggering.
- **Validation / failure:** invalid events go to DLQ, not processed; exactly-once via idempotency keys + dedup window (default 24h); ordering preserved per partition/key ([29](../../deep-dive/29-workflow-scheduling-and-events.md) §Story 3.3).

## Step 3 · Review & Activate

Summary card (type-specific) + **Enabled** toggle + **Retry/delivery policy** + conflict policy. Primary: **Activate trigger**.

| Policy field | Branch / values |
|---|---|
| Enabled | on/off (a trigger can be created paused) |
| Retry on failure | e.g. `3× exponential backoff`; webhooks retry with backoff then DLQ; cron `retry_on_failure` |
| `max_concurrent_runs` | execution-policy cap |
| `timeout_seconds` | per-run cap |
| Conflict/overlap (cron) | `skip` / `queue` / `parallel` / `terminate` |
| Missed-execution (cron) | `run_once` / `skip` / `run_all` |

- **What this enables:** the trigger appears in the trigger registry/catalog (health status, last execution, error rate) and execution history is queryable with linked workflow run IDs ([29](../../deep-dive/29-workflow-scheduling-and-events.md) Epic 4). Trigger alerting fires on failure-rate thresholds, missed schedules, webhook delivery failures, or queue backlog; a trigger may auto-disable (circuit breaker).

---

## Per-branch "make it work" checklist

| Branch | The decisive external requirement |
|---|---|
| Cron | scheduler service running + HA; valid cron; target agent/version exists |
| Webhook | source system configured to POST the generated URL **with the matching signing secret**; platform reachable |
| Event | broker reachable + credentialed; upstream emitting the topic; consumer group set |

## Failure-state summary

| Step | Failure | Fix |
|---|---|---|
| Type | (none — selection only) | — |
| Cron | invalid expression / scheduler down / agent missing | fix cron; ensure scheduler HA; pin a real agent+version |
| Webhook | signature mismatch (`401`) / source not pointed at URL / transform error | sync the secret; configure the source; fix the mapping |
| Event | broker unreachable / bad credentials / schema mismatch (→ DLQ) | fix connection/creds; align event schema |
| Review | trigger left disabled | enable the toggle to activate |

## Cross-references

- Decisions: [AD-16](../../01-PRD.md) (version pinning on triggered runs).
- Deep-dives: [29-workflow-scheduling-and-events](../../deep-dive/29-workflow-scheduling-and-events.md) (all trigger types, delivery, observability), [09-security-model](../../deep-dive/09-security-model.md) (webhook HMAC auth), [16-multi-tenancy](../../deep-dive/16-multi-tenancy.md) (namespace-scoped endpoint URL).
- Figma (worked examples): `239:62` (Type) · `241:77` (Cron) · `242:79` (Webhook) · `243:79` (Review) · `244:81` (Event). Parent screen: Scheduling `67:56`.
