# Operator CLI

## Purpose

This document specifies the minimal Operator CLI — a focused command-line interface for Operators who monitor running workflows, step through debug mode, and send control signals (pause, resume, cancel) without needing a full provisioning UI.

The CLI is scoped deliberately: it covers the Operator persona's needs from v6 only. It is not a general-purpose UI. The full provisioning UI (deferred to post-v6, per AD-25) will be built against the same stable API endpoints this CLI uses.

---

## Why a CLI Before a Full UI

The v6 debug mode (step-by-step workflow approval, input override) is API-driven. Without at least a minimal CLI, operators must craft raw HTTP requests against a stateful, multi-step workflow system — a serious operational friction point. The CLI provides:

- **Discoverability**: `run list`, `run inspect` make the system's current state legible without a dashboard.
- **Debug workflow**: `run debug approve` / `run debug override` make step-through development practical.
- **Incident response**: `run pause`, `run resume`, `run cancel` are the primary tools during production incidents.
- **Low build cost**: The CLI is a thin wrapper over the existing v6 API. No new backend work is required.
- **UI design input**: Real operator usage of the CLI will reveal exactly what a full UI needs to show.

---

## Command Reference

All commands are under the `agent-platform` binary (same binary as the developer CLI documented in `15-developer-tooling.md`).

### `run list`

```
agent-platform run list [options]

Options:
  --namespace <ns>     Filter by namespace (default: from config)
  --agent <name>       Filter by agent name
  --status <s>         Filter by status: pending | running | paused | debug | succeeded | failed | cancelled
  --since <duration>   Show runs started within duration (e.g. 1h, 24h, 7d). Default: 24h
  --limit <n>          Max results (default: 50)
  --json               Machine-readable output
```

**Example**:

```
$ agent-platform run list --status running --namespace acme-corp

Active Workflow Runs (acme-corp)
─────────────────────────────────────────────────────────────────────────
  RUN ID           AGENT                  STATUS    STEP              ELAPSED
  wfr-a1b2c3       support-triage         running   gather_context    0:00:12
  wfr-d4e5f6       billing-specialist     running   call_payment_api  0:00:45
  wfr-g7h8i9       technical-specialist   paused    generate_answer   0:05:02 ⏸
  wfr-j0k1l2       support-triage         debug     classify          0:01:18 🔍

4 active runs.
```

---

### `run inspect`

Shows the full state of a specific workflow run including step timeline.

```
agent-platform run inspect <run-id> [--namespace <ns>] [--json]
```

**Example**:

```
$ agent-platform run inspect wfr-a1b2c3

Workflow Run: wfr-a1b2c3
  Agent:     support-triage (v7)
  Namespace: acme-corp
  Status:    running
  Started:   2026-05-25T14:31:02Z (12s ago)
  Trace ID:  trace-xyz789

Steps:
  ✓  classify          llm_call     0:00:01   succeeded
  ▶  gather_context    parallel     0:00:11   running
       kb/kb_search    tool_call    0:00:03   succeeded
       history/fetch_history  mcp_call  running  (9s)
  ○  route             switch       —         pending
  ○  call_billing_agent  agent_call  —         pending
  ○  log_outcome       mcp_call     —         pending

Current step inputs (gather_context):
  {
    "query": "Duplicate charge inquiry",
    "customer_id": "C-1234"
  }
```

---

### `run pause`

```
agent-platform run pause <run-id> [--reason "text"] [--namespace <ns>]
```

**Example**:

```
$ agent-platform run pause wfr-d4e5f6 --reason "Investigating API rate limit spike"

Pausing wfr-d4e5f6... The run will pause after the current step completes.
Run wfr-d4e5f6 is now paused at step: call_payment_api
```

---

### `run resume`

```
agent-platform run resume <run-id> [--namespace <ns>]
```

**Example**:

```
$ agent-platform run resume wfr-g7h8i9

Resuming wfr-g7h8i9 from step: generate_answer
Run wfr-g7h8i9 is now running.
```

---

### `run cancel`

```
agent-platform run cancel <run-id> [--reason "text"] [--cascade | --no-cascade] [--namespace <ns>]
```

`--cascade` (default) cancels child workflows. `--no-cascade` orphans them.

**Example**:

```
$ agent-platform run cancel wfr-a1b2c3 --reason "Test run, no longer needed"

Cancelling wfr-a1b2c3...
  Child wfr-x1y2z3 (item-enricher) → cancelled
Run wfr-a1b2c3 cancelled. Reason: "Test run, no longer needed"
```

---

### `run debug approve`

In a workflow running in debug mode, approves the next step to execute using its resolved inputs.

```
agent-platform run debug approve <run-id> [--namespace <ns>]
```

**Example**:

```
$ agent-platform run debug approve wfr-j0k1l2

Debug mode: wfr-j0k1l2
Current step: classify [llm_call]

Resolved inputs:
  {
    "request": "I was charged twice",
    "customer_id": "C-1234"
  }

Approve this step? [y/N] y

Step classify approved. Executing...
Step classify completed.
  Output:
    {
      "intent": "billing",
      "confidence": 0.97,
      "summary": "Duplicate charge inquiry"
    }

Next step: gather_context [parallel]
Run paused at next step. Use 'run debug approve' or 'run debug override' to continue.
```

---

### `run debug override`

Approves the next step with a manually supplied input override, replacing the resolved inputs.

```
agent-platform run debug override <run-id> --input <json-file-or-inline-json> [--namespace <ns>]
```

**Example**:

```
$ agent-platform run debug override wfr-j0k1l2 \
    --input '{"request": "I need to update my billing address", "customer_id": "C-1234"}'

Debug mode: wfr-j0k1l2
Current step: classify [llm_call]

Resolved inputs (original):
  { "request": "I was charged twice", "customer_id": "C-1234" }

Override inputs:
  { "request": "I need to update my billing address", "customer_id": "C-1234" }

Apply override and execute? [y/N] y

Step classify executed with overridden input.
  Output:
    {
      "intent": "admin",
      "confidence": 0.94,
      "summary": "Billing address update request"
    }

Override and output recorded on step run for audit.
```

---

### `run debug mode`

Switches a workflow between normal and debug mode at runtime.

```
agent-platform run debug mode <run-id> <on|off> [--namespace <ns>]
```

**Example**:

```
$ agent-platform run debug mode wfr-a1b2c3 on

Switching wfr-a1b2c3 to debug mode at next safe checkpoint.
Run will pause before the next step and await 'run debug approve'.
```

---

### `run logs`

Streams or fetches structured logs for a specific run.

```
agent-platform run logs <run-id> [--follow] [--since <duration>] [--namespace <ns>]
```

**Example**:

```
$ agent-platform run logs wfr-a1b2c3 --follow

[14:31:02.001] INFO  run=wfr-a1b2c3 step=classify    Starting llm_call activity
[14:31:02.891] INFO  run=wfr-a1b2c3 step=classify    llm_call completed (tokens: 512 in / 128 out)
[14:31:03.002] INFO  run=wfr-a1b2c3 step=gather_context  Starting parallel block (2 branches)
[14:31:03.105] INFO  run=wfr-a1b2c3 step=kb_search   Invoking tool: knowledge_base_lookup
[14:31:03.201] INFO  run=wfr-a1b2c3 step=fetch_history  Invoking mcp: jira/issues.search
[14:31:06.103] INFO  run=wfr-a1b2c3 step=fetch_history  mcp_call completed (latency: 2.9s)
[14:31:06.201] INFO  run=wfr-a1b2c3 step=gather_context  All branches completed
```

---

### `run trace`

Outputs the full OpenTelemetry trace for a run (link or exportable JSON). Requires observability stack configured.

```
agent-platform run trace <run-id> [--namespace <ns>] [--open]
```

`--open` launches the trace in the configured Jaeger/Tempo UI in the default browser.

---

### `agent versions`

```
agent-platform agent versions <agent-name> [--namespace <ns>] [--json]

$ agent-platform agent versions support-triage --namespace acme-corp

Versions: support-triage (acme-corp)
─────────────────────────────────────────
  v7   2026-05-25  hugo@acme.com   latest   active runs: 3
  v6   2026-05-20  hugo@acme.com            active runs: 0
  v5   2026-05-14  ana@acme.com             active runs: 0
  v4   2026-05-01  hugo@acme.com  ⚠ deprecated  active runs: 0
  ...
```

---

### `agent deprecate`

```
agent-platform agent deprecate <agent-name> <version> [--namespace <ns>]
```

---

## Configuration

The Operator CLI uses the same configuration as the developer CLI (see `15-developer-tooling.md`). A separate profile can be configured for production environments:

```
$ agent-platform --profile production run list
```

Profiles live in `~/.agent-platform/config.json`:

```json
{
  "profiles": {
    "default": {
      "platform_url": "https://platform-staging.acme-internal.com",
      "api_key": "sk-staging-...",
      "default_namespace": "acme-corp"
    },
    "production": {
      "platform_url": "https://platform.acme-internal.com",
      "api_key": "sk-prod-...",
      "default_namespace": "acme-corp"
    }
  }
}
```

---

## Scope and Non-Goals

**In scope for this CLI**:
- Listing and inspecting workflow runs.
- Pause, resume, cancel.
- Debug mode: approve, override, mode switch.
- Log streaming.
- Agent version management (list, deprecate).

**Not in scope** (belongs to future provisioning UI or developer CLI):
- Creating or editing agent definitions.
- Registering tools or MCPs.
- Managing namespaces.
- Managing credentials.

---

## v6 Integration

The Operator CLI is an epic within v6 (`07-v6-versioning-and-hardening.md`). Its implementation depends on:

- **Pause/resume/cancel API** (v6 Epic 2).
- **Debug mode API** (v6 Epic 3).
- **Trace IDs and structured logging** (v6 Epic 4).
- **Agent versioning API** (v6 Epic 1).

The CLI itself adds no new backend functionality; it exposes the v6 API surface in an ergonomic terminal interface.

---

## What to Read Next

- **[07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md)** — The v6 API surface this CLI wraps
- **[15-developer-tooling.md](./15-developer-tooling.md)** — The developer-facing CLI (shared binary, different commands)
- **[10-deployment-concepts.md](./10-deployment-concepts.md)** — Operations runbook that this CLI supports

---

## Related Documents

- [07-v6](./07-v6-versioning-and-hardening.md) — Pause/resume, debug mode, observability
- [09-security-model.md](./09-security-model.md) — Operator persona access control
- [15-developer-tooling.md](./15-developer-tooling.md) — Developer CLI (shared binary)
