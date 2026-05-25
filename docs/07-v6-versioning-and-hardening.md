# v6 — Versioning, Observability, and Hardening

## Version Goal

Take the platform from "working end to end" to **production-grade**: immutable agent versioning, pause/resume capabilities, step-by-step debug mode for development, full observability (tracing, metrics, dashboards), and migration of secrets storage to a proper vault.

This phase doesn't add user-facing features; it adds the operational properties that make the platform safe to run at scale, debug under pressure, and evolve without breaking in-flight work.

## Business Value

- **Safety**: in-flight workflows never break when agents are updated.
- **Debuggability**: operators can step through workflows manually during development, dramatically shortening agent iteration cycles.
- **Operability**: real-time visibility into the platform's health, throughput, and latency.
- **Security**: secrets are no longer plain-text. The platform meets enterprise security baselines.

---

## Epic 1 — Agent Versioning

**Business Value**: Decouples agent evolution from agent execution. Builders can iterate freely while running workflows continue undisturbed.

### Story 1.1 — Immutable agent versions
**As an** Agent Builder
**I want** every change to an agent to produce a new version
**So that** I can iterate without disrupting active workflows.

**Description**: When an agent is updated, a new version row is inserted (never an update-in-place). Each version has its own unique ID. Workflow runs reference the agent version they started with, and the worker loads that exact version's configuration.

**Acceptance Criteria**:
- Updating an agent produces a new version; the old version remains intact.
- Workflow runs persist the version they were launched with.
- A workflow run that resumes after restart uses its original version.

### Story 1.2 — "Latest" alias
**As an** Agent Builder
**I want** new invocations to use the latest version by default
**So that** improvements propagate to new traffic automatically.

**Description**: Maintain a `latest` pointer per agent name. New invocations without an explicit version use `latest`. Versioned references (`v3`, `v17`, etc.) are honored exactly.

**Acceptance Criteria**:
- Calling an agent without a version uses the current `latest`.
- The actual version used is recorded on the workflow run.

### Story 1.3 — Version deprecation
**As a** Platform Admin
**I want** to deprecate old versions
**So that** I can prevent new traffic from using them while existing runs finish.

**Description**: A version can be marked `deprecated`. Deprecated versions can be invoked only by referencing them explicitly (with a warning) or not at all (configurable). They cannot be set as `latest`.

**Acceptance Criteria**:
- Deprecation marks a version without deleting it.
- The behavior of deprecated versions (warn vs reject) is configurable per deployment.

### Story 1.4 — Version diff and history view
**As an** Agent Builder
**I want** to see how an agent has evolved over its versions
**So that** I can debug regressions and audit changes.

**Description**: An API/UI endpoint returns a version list with timestamps, authors, and structured diffs between versions.

**Acceptance Criteria**:
- The full history of an agent is queryable.
- Diffs show added/removed/changed fields in a human-readable form.

---

## Epic 2 — Pause and Resume

**Business Value**: Enables operator intervention without restarting runs. Critical for production support.

### Story 2.1 — Pause a running workflow
**As an** Operator
**I want** to pause an active workflow at the next safe checkpoint
**So that** I can intervene, inspect, or wait for an external dependency.

**Description**: Use Temporal's signal/query primitives to implement pause. A signal pauses the workflow before its next step transition; state is persisted; the workflow waits for a resume signal.

**Acceptance Criteria**:
- A pause request leaves the workflow in a `paused` state with no further progress.
- The current step completes before pause takes effect (no mid-step pause in v6).
- The workflow run record reflects the paused state.

### Story 2.2 — Resume a paused workflow
**As an** Operator
**I want** to resume a paused workflow exactly where it left off
**So that** I can continue execution after intervention.

**Description**: A resume signal continues the workflow from the next step. Optionally, the operator may override the input to the next step at resume time.

**Acceptance Criteria**:
- Resume continues correctly.
- Optional input override is applied and recorded.
- Resume failures (e.g. agent definition deprecated mid-pause) produce clear errors.

### Story 2.3 — Cancel a workflow
**As an** Operator
**I want** to cancel a workflow that should not continue
**So that** I can stop runaway or stuck work.

**Description**: A cancel signal terminates the workflow gracefully, records the cancellation reason and operator, and triggers any defined cleanup steps.

**Acceptance Criteria**:
- Cancellation produces a `cancelled` status with a recorded reason.
- Cleanup hooks (if any) are invoked.
- Child workflows are cancelled appropriately according to policy.

---

## Epic 3 — Step-by-Step Debug Mode

**Business Value**: Drastically shortens the development cycle for complex agents. Operators iterate on workflow design without rebuilding test fixtures.

### Story 3.1 — Launch a workflow in debug mode
**As an** Operator
**I want** to start a workflow in debug mode
**So that** I control when each step executes.

**Description**: A workflow started with `debug: true` begins paused before the first step. The operator approves each step transition manually via an API call. Step inputs can be inspected and optionally overridden before each step runs.

**Acceptance Criteria**:
- A debug-mode workflow runs no steps until approved.
- Each step approval can pass through or override the input.
- Step outputs are presented for inspection before the next approval.

### Story 3.2 — Inspect intermediate state in debug mode
**As an** Operator
**I want** to view the full workflow state between steps
**So that** I can verify the run is behaving as expected.

**Description**: At each pause point, the operator can query workflow state: prior step outputs, accumulated context, the upcoming step's resolved inputs.

**Acceptance Criteria**:
- A state query API returns the current pause context.
- The response includes all data a step would receive.

### Story 3.3 — Switch a workflow between debug and normal mode
**As an** Operator
**I want** to flip a workflow into or out of debug mode at runtime
**So that** I can deepen scrutiny only when needed.

**Description**: Signals can switch a workflow between debug (manual step-through) and normal execution. Useful when something starts going wrong mid-run.

**Acceptance Criteria**:
- A normal-running workflow can be switched into debug mode at the next safe point.
- A debug workflow can be switched back to normal execution.
- All mode transitions are recorded.

---

## Epic 4 — Observability

**Business Value**: Real-time visibility into the platform's health, plus the data needed to diagnose problems quickly.

### Story 4.1 — Structured trace IDs end-to-end
**As an** Operator
**I want** every action across the platform to share a correlatable trace ID
**So that** I can follow a single request through every layer.

**Description**: Generate a trace ID at the API edge. Propagate it through Temporal workflows (as a workflow attribute), into activities, into provider calls, into tool/MCP invocations, into the database via log records.

**Acceptance Criteria**:
- A single request can be reconstructed from logs by trace ID across all services.
- Trace IDs appear in every workflow run, step run, and log line.

### Story 4.2 — Prometheus metrics
**As an** Operator
**I want** standard metrics exported to Prometheus
**So that** I can build dashboards and alerts on platform health.

**Description**: Export metrics: workflow throughput, latency per step type, error rates per provider, retry counts, queue depth, active workers, KB lookup latency, tool/MCP call latency. Sensible default labels and cardinality controls.

**Acceptance Criteria**:
- Metrics are available at a `/metrics` endpoint.
- A starter Grafana dashboard is provided.
- Alerts on critical conditions (error spike, worker death, queue backlog) are documented.

### Story 4.3 — Distributed tracing with OpenTelemetry
**As an** Operator
**I want** distributed traces across the platform's services
**So that** I can visualize latency contributions and find bottlenecks.

**Description**: Instrument the API, the orchestrator, the Temporal workers, and the chatbot streaming layer with OpenTelemetry. Spans cover step execution, tool/MCP calls, provider calls, KB lookups.

**Acceptance Criteria**:
- A request produces a complete trace in the configured backend (Jaeger/Tempo/etc.).
- Spans include relevant attributes (agent ID, step ID, provider).
- Tracing has a configurable sampling rate to control overhead.

### Story 4.4 — LLM observability hook (LangFuse or equivalent — design only) (E-19)
**As an** Operator
**I want** the platform ready to plug in LLM-specific observability later
**So that** v6 leaves a clean integration point.

**Description**: Define an `LLMObservabilityClient` interface that the provider adapters call on each LLM invocation. v6 ships with a no-op default and documentation for swapping in LangFuse or similar. Full implementation deferred.

The interface must carry both the stable prompt template (from the agent version) and the per-invocation interpolated prompt, so observability backends can segment observations by template version:

```
interface LLMObservabilityClient {
  record(event: {
    agent_id:            string,
    agent_version:       string,
    workflow_run_id:     string,
    step_id:             string,
    trace_id:            string,
    provider:            string,
    model_name:          string,
    system_prompt:       string,          // from agent version (stable)
    prompt_template:     string,          // user_prompt_template (from agent version)
    interpolated_prompt: string,          // final user prompt sent to provider (per-invocation)
    response:            string,
    tokens:              TokenCounts,
    latency_ms:          number,
    estimated_cost_usd:  number,
    finish_reason:       string,
    provider_used:       string           // may differ from agent's declared provider (failover)
  }): void
}
```

**Acceptance Criteria**:
- The interface is in place with the full signature above and called on every LLM invocation.
- A no-op default is shipped.
- Documentation describes how to wire in a real backend (e.g., LangFuse `traceId` mapping).

---

## Epic 5 — Secrets Vault Migration

**Business Value**: Removes the plain-text storage tech debt. Brings the platform to a security baseline acceptable for production and enterprise use.

### Story 5.1 — Vault-backed CredentialsProvider implementation
**As a** Platform Engineer
**I want** to load credentials from a real secrets vault
**So that** plain-text storage is eliminated.

**Description**: Implement a vault-backed `CredentialsProvider` (HashiCorp Vault, AWS Secrets Manager, GCP Secret Manager — configurable). The interface defined in v3 means no code outside the credentials module changes.

**Acceptance Criteria**:
- A vault-backed implementation passes the same tests as the plain-text implementation.
- Switching between implementations is a configuration change.
- Failures in vault access produce clear, distinguishable errors.

### Story 5.2 — Migration of existing credentials
**As a** Platform Admin
**I want** a one-time migration that moves credentials from the database to the vault
**So that** the cutover is safe and reversible.

**Description**: A migration tool reads existing plain-text credentials, writes them to the vault, verifies retrieval, then optionally clears them from the database. Dry-run mode for verification.

**Acceptance Criteria**:
- The tool can be run in dry-run mode and reports what would change.
- A real run successfully migrates all credentials with verification.
- Rollback procedure is documented.

### Story 5.3 — Secret rotation support
**As a** Platform Admin
**I want** to rotate secrets without affecting in-flight workflows
**So that** rotations are routine, not risky.

**Description**: Vault-backed credentials are refetched at workflow start (already true from v2's per-workflow snapshot). Add documentation for safe rotation patterns, including handling token expiry for OAuth-style credentials.

**Acceptance Criteria**:
- Rotations affect new workflows immediately and in-flight workflows after restart.
- OAuth-style refresh tokens are handled.
- The procedure is documented.

---

## Epic 6 — Operator CLI

**Business Value**: The v6 API surface (pause/resume, debug mode, trace IDs, versioning) is powerful but unusable in practice without a terminal interface. Operators currently managing production incidents should not need to craft raw HTTP requests to send pause signals or step through a debug workflow. The CLI makes v6's operational features actually accessible.

### Story 6.1 — Core run management commands
**As an** Operator
**I want** to list, inspect, pause, resume, and cancel workflow runs from the terminal
**So that** I can respond to incidents without waiting for a UI.

**Description**: Implement the `run list`, `run inspect`, `run pause`, `run resume`, and `run cancel` commands specified in `17-operator-cli.md`. The CLI is a thin wrapper over the v6 API endpoints introduced in Epics 1–3 of this phase.

**Acceptance Criteria**:
- All five commands are functional against a live platform instance.
- `run list` supports filtering by namespace, agent, status, and time range.
- `run inspect` shows the full step timeline including in-progress steps.
- `run pause` and `run resume` correctly drive the Temporal signal mechanism from Epic 2.
- `run cancel` accepts `--cascade` and `--no-cascade` options.

### Story 6.2 — Debug mode CLI commands (E-20)
**As an** Operator
**I want** to approve, override, and monitor debug-mode workflows from the terminal
**So that** step-by-step debugging is a practical development workflow, not just an API feature.

**Description**: Implement the `run debug approve`, `run debug override`, and `run debug mode` commands. These commands use the debug-mode API from Epic 3.

**Override input validation**: Before a `run debug override` is applied, the proposed override inputs are validated against the step's declared `inputs` mapping and (for `tool_call`/`mcp_call`) the tool/MCP's `input_schema`. Invalid overrides are rejected with a structured error at the CLI — they are not passed through to the step. The CLI displays the validation result before asking for confirmation.

**`input_override` column**: The `workflow_step_runs` table gains an `input_override: JSON | null` column. When an override is applied, `input` stores what the step actually received, and `input_override` stores the original resolved input (for audit diff).

**Acceptance Criteria**:
- `run debug approve` presents the resolved inputs and requires confirmation before executing the step.
- `run debug override` validates the override against the step's schema before applying it.
- Invalid override inputs are rejected at the CLI with a clear structured error.
- The `input_override` column is populated on the step run record when an override was applied.
- `run debug mode on|off` switches a running workflow into or out of debug mode.
- The CLI displays the step output after each approval so the operator can decide whether to proceed.

### Story 6.3 — Log streaming and trace link
**As an** Operator
**I want** to stream logs for a specific run and get a direct link to its distributed trace
**So that** I can diagnose failures without navigating multiple dashboards.

**Description**: Implement `run logs --follow` and `run trace`. Log streaming connects to the structured log stream for the run (filtered by `workflow_run_id`). The trace command outputs the trace URL in the configured backend.

**Acceptance Criteria**:
- `run logs --follow` streams new log lines in real time until the run completes or Ctrl-C.
- `run trace` outputs a clickable URL to the trace in the configured Jaeger/Tempo backend.
- `run trace --open` launches the trace in the default browser.

### Story 6.4 — Agent version management commands (E-39)
**As an** Operator
**I want** to view version history, deprecate agent versions, and notify builders of deprecations from the terminal
**So that** version lifecycle management has a usable interface before a full provisioning UI ships.

**Description**: Implement `agent versions`, `agent deprecate`, and `agent notify-deprecation` commands. These wrap the versioning API from Epic 1.

**Deprecation notification channels**: When a version is deprecated, the platform notifies the Agent Builder through configured channels:
- `email`: sends a deprecation notice to the namespace's configured admin email
- `webhook`: POSTs a deprecation event to the namespace's configured `deprecation_webhook_url`
- `platform_alert`: creates an Operator-visible alert in the platform's alert queue

The deprecation event payload includes: `agent_id`, `agent_version`, `deprecated_at`, `deprecated_by`, `active_run_count`, `message` (optional human-readable reason).

Namespaces configure their preferred channel(s) in `settings.deprecation_notification_channels`.

**Acceptance Criteria**:
- `agent versions` shows all versions with status, author, timestamp, and active run count.
- `agent deprecate` marks the specified version deprecated, requires a confirmation prompt, and triggers configured deprecation notifications.
- Deprecating a version with active runs produces a clear warning with the active run count.
- `agent notify-deprecation <agent> <version>` re-sends the deprecation notification (useful for missed deliveries).

---

## Technical Considerations (high level)

- **Temporal signals and queries**: pause/resume/debug are built on Temporal's native signal/query mechanisms. They are first-class, not bolt-on.
- **Trace ID propagation**: be deliberate. Use a single header (e.g. `traceparent`) and ensure every async hop (queue, workflow, child workflow) carries it.
- **Cardinality control**: Prometheus metrics with per-agent labels can explode cardinality. Use bounded label sets; track high-cardinality data in tracing instead.
- **Vault selection**: pick a default (HashiCorp Vault is portable) but design the interface so other vaults are pluggable.
- **Versioning storage**: agent versions can grow large. Consider an `agent_versions` table separate from `agents` for clean indexing and quota management.
- **Deprecation policy**: deprecation should be a soft signal first, then a hard block. Provide tooling to find callers of a deprecated version before hard-blocking.
- **Observability cost**: tracing and metrics have cost. Provide sampling and feature flags so volume can be tuned per environment.

## Definition of Done for v6

- Every agent change produces a new immutable version; running workflows pin to their version.
- Workflows can be paused, resumed, and cancelled cleanly via operator actions.
- Workflows can run in step-by-step debug mode with manual approval and input override.
- Trace IDs propagate end to end; Prometheus metrics are exported; OpenTelemetry tracing covers the platform.
- An LLM observability hook is in place ready for LangFuse-style integration.
- All credentials live in a vault; plain-text storage is gone.
- The platform is production-ready by operational standards.

---

## What to Read Next

With v6, the platform is production-grade. Explore operational and supporting concepts:

- **[08-Performance and Scaling](./08-performance-and-scaling.md)** — Capacity planning and latency budgets
- **[09-Security Model](./09-security-model.md)** — Threat model and security principles
- **[10-Deployment Concepts](./10-deployment-concepts.md)** — Infrastructure topology and operations
- **[11-Testing Strategy](./11-testing-strategy.md)** — Quality gates and the evolution testing concept
- **[12-Upgrades and Migrations](./12-upgrades-and-migrations.md)** — Version compatibility and rollback strategies
- **[13-Chatbot UX Edge Cases](./13-chatbot-ux-edge-cases.md)** — Conversation failure modes and edge cases

Or return to the beginning:
- [README](./README.md) — The complete documentation index
- [01-PRD](./01-PRD.md) — The full decision log and requirements
