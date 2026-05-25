# Developer Tooling

## Purpose

This document specifies the local developer tooling for Agent Builders working with the platform DSL. The primary surface is a CLI (`agent-platform`) that provides fast, local feedback without requiring a running platform instance for basic validation tasks.

This addresses the gap identified during design review: Agent Builders need faster iteration than HTTP round-trips to the provisioning API allow, particularly when building complex v4 workflows with sub-agents, loops, and branches.

---

## Design Principles

- **Zero-dependency local execution**: `validate` and `dry-run` should not require a running Temporal cluster, live LLM providers, or a database connection.
- **Same validation logic**: The CLI runs the same validation service used by the provisioning API. There is one validator; the CLI is a local transport layer over it.
- **Progressive disclosure**: The simplest command (`validate`) is one line. Advanced features (`dry-run`, `trace`) are opt-in.
- **Composable**: CLI output is machine-readable (JSON with `--json` flag) so it can integrate with editors, CI pipelines, and pre-commit hooks.

---

## Command Reference

### `agent-platform validate`

Validates a DSL document against the platform's validation service locally.

```
agent-platform validate <file>

Options:
  --namespace <ns>    Override the namespace field for testing in a different context
  --json              Output errors as JSON (default: human-readable)
  --strict            Treat warnings as errors
  --version <v>       Force validation against a specific DSL version (default: auto-detect)
```

**Example — human-readable output**:

```
$ agent-platform validate ./agents/support-triage.json

Validating support-triage.json (DSL v4)...

ERROR  [workflow.steps[2].tool]
       UNKNOWN_TOOL: Tool 'web_scraper' is not registered in the Tool Registry.
       Value: "web_scraper"
       Tip: Run 'agent-platform tools list' to see registered tools.

ERROR  [workflow.steps[3].inputs.documents]
       UNRESOLVABLE_REFERENCE: Step 'search' does not declare output field 'docs'.
       Declared outputs: ['documents', 'scores']
       Value: "$.steps.search.output.docs"

WARNING [provider]
        DEPRECATED_PROVIDER: Provider 'grok' is deprecated.
        It will continue to work but should be migrated.

2 error(s), 1 warning(s). Validation failed.
```

**Example — JSON output**:

```
$ agent-platform validate ./agents/support-triage.json --json

{
  "valid": false,
  "errors": [...],
  "warnings": [...],
  "dsl_version": "4",
  "agent_name": "support-triage"
}
```

**Example — passing validation**:

```
$ agent-platform validate ./agents/payment-processor.json

Validating payment-processor.json (DSL v4)...
✓ Valid. 0 errors, 0 warnings.
```

**Exit codes**:
- `0`: No errors (warnings may be present unless `--strict`).
- `1`: One or more errors.
- `2`: File not found or unparseable JSON.

---

### `agent-platform dry-run`

Simulates a workflow execution locally without invoking real providers, tools, or MCPs. Uses mock responses to exercise the data flow, branching logic, and step sequencing.

```
agent-platform dry-run <file> --input <json-file-or-inline-json>

Options:
  --input <json>         Workflow input payload (required). File path or inline JSON string.
  --mock-dir <path>      Directory of mock response files (see Mock Resolution below)
  --provider-mock <file> Single mock file applied to all llm_call steps
  --tool-mock <file>     Single mock file applied to all tool_call steps
  --cassette <dir>       Use recorded cassettes from the testing cassette directory (11-testing-strategy.md)
  --live-llm             Use real LLM providers (requires credentials configured via config or env)
  --estimate-cost        Print token and cost estimates per step and a total run estimate
  --graph                Validate the full agent call graph before dry-running (requires platform connection)
  --step <id>            Run only up to (and including) this step, then stop
  --verbose              Show full input/output at each step (including metadata envelope)
  --json                 Output execution trace as JSON
  --timeout <seconds>    Maximum dry-run duration (default: 30)
```

**Example**:

```
$ agent-platform dry-run ./agents/support-triage.json \
    --input '{"user_message": "I was charged twice", "customer_id": "C-1234"}' \
    --mock-dir ./mocks/support-triage/

Dry-running support-triage (DSL v4)...

Step 1: classify [llm_call]
  Input:  { "request": "I was charged twice", "customer_id": "C-1234" }
  Mock:   ./mocks/support-triage/classify.json
  Output: { "intent": "billing", "confidence": 0.97, "summary": "Duplicate charge inquiry" }
  ✓ OK  (12ms)

Step 2: gather_context [parallel — 2 branches]
  Branch kb/kb_search [tool_call: knowledge_base_lookup]
    Input:  { "query": "Duplicate charge inquiry", "filters": {...}, "top_k": 3 }
    Mock:   ./mocks/support-triage/kb_search.json
    Output: { "documents": [...] }
    ✓ OK  (3ms)
  Branch history/fetch_history [mcp_call: jira/issues.search]
    Input:  { "customer_id": "C-1234", "limit": 5 }
    Mock:   ./mocks/support-triage/fetch_history.json
    Output: { "issues": [...] }
    ✓ OK  (5ms)
  ✓ Both branches succeeded (5ms)

Step 3: route [switch]
  Value:  "billing"
  Matched: "billing" → call_billing_agent
  ✓ OK

Step 4: call_billing_agent [agent_call]
  Agent:  billing-specialist (mode: wait)
  Input:  { "customer_id": "C-1234", ... }
  Mock:   ./mocks/support-triage/call_billing_agent.json
  Output: { "resolution": "Refund initiated", "actions_taken": ["refund_issued"] }
  ✓ OK  (8ms)

Step 5: log_outcome [mcp_call: jira/issues.create]
  Input:  { "title": "Duplicate charge inquiry", ... }
  Mock:   ./mocks/support-triage/log_outcome.json
  Output: { "issue_id": "LIN-4521" }
  ✓ OK  (4ms)

─────────────────────────────────────────
Dry-run complete. 5/5 steps succeeded.
Total simulated time: 32ms
Final output: { "issue_id": "LIN-4521" }
```

---

### Mock Resolution

The dry-run command resolves mocks in the following order (first match wins):

1. **Named mock file**: `<mock-dir>/<step-id>.json` — exact match by step ID.
2. **Type mock file**: `<mock-dir>/<step-type>.json` — fallback for all steps of a type (e.g., `llm_call.json` for all `llm_call` steps without a named mock).
3. **Default mock**: A built-in minimal mock response for each step type (see below).

**Default mock shapes** (used when no mock file is provided):

| Step Type | Default Mock Output |
|---|---|
| `llm_call` | `{ "text": "[MOCK LLM RESPONSE]", "finish_reason": "stop", "declared_outputs": {} }` |
| `tool_call` | `{ "raw_output": {}, "declared_outputs": {} }` |
| `mcp_call` | `{ "raw_output": {}, "declared_outputs": {} }` |
| `agent_call (wait)` | `{ "child_run_id": "mock-run-id", "child_status": "succeeded", "declared_outputs": {} }` |
| `agent_call (fire_and_forget)` | `{ "child_run_id": "mock-run-id" }` |
| `transform` | *(computed from the actual mapping expression — no mock needed)* |
| `branch` / `switch` | *(evaluated from prior step outputs — no mock needed)* |
| `while` / `until` | *(condition evaluated from actual loop state — body steps use their own mocks)* |

**Mock file format** — the `output` portion of the canonical step output envelope:

```json
{
  "text": "The charge appears to be a duplicate. Initiating a refund.",
  "finish_reason": "stop",
  "declared_outputs": {
    "intent": "billing",
    "confidence": 0.97,
    "summary": "Duplicate charge inquiry"
  },
  "tool_calls": []
}
```

---

### `agent-platform mcps refresh` (E-13)

Force-refreshes an MCP connector's cached manifest from the live MCP server.

```
agent-platform mcps refresh <name> [--namespace <ns>]

$ agent-platform mcps refresh jira

Refreshing MCP manifest: jira...
  Previous manifest version: 2025-01-15
  New manifest version:      2025-05-20
  Capabilities added: ["projects.create"]
  Capabilities removed: ["issues.bulk_delete"]  ← WARNING

WARNING: 1 capability removed. The following agents reference removed capabilities:
  - support-triage (step: log_outcome → jira/issues.bulk_delete)
  Run 'agent-platform validate --graph support-triage.json' to check impact.

Manifest updated. New version cached.
```

---

### `agent-platform tools list`

Lists all tools registered in a configured platform instance (requires connection).

```
agent-platform tools list [--namespace <ns>] [--json]

$ agent-platform tools list

Registered Tools (namespace: acme-corp)
─────────────────────────────────────────
  knowledge_base_lookup    Query the Qdrant knowledge base (semantic, hybrid, reranked)
  web_search               Search the public web for recent information
  quality_scorer           Score text quality against a rubric
  job_status_checker       Poll an internal job for completion status

4 tool(s) registered.
```

---

### `agent-platform mcps list`

Lists all MCP connectors registered in a configured platform instance.

```
agent-platform mcps list [--namespace <ns>] [--json]

$ agent-platform mcps list

Registered MCPs (namespace: acme-corp)
─────────────────────────────────────────
  linear      HTTP    Authenticated (api_key)    15 capabilities
  jira        HTTP    Authenticated (oauth)      28 capabilities
  figma       HTTP    Authenticated (oauth)      11 capabilities

3 MCP(s) registered. Run 'agent-platform mcps capabilities <name>' to inspect.
```

---

### `agent-platform agents list`

Lists agents in a namespace on a configured platform instance.

```
agent-platform agents list [--namespace <ns>] [--json]

$ agent-platform agents list --namespace acme-corp

Agents (namespace: acme-corp)
─────────────────────────────────────────
  support-triage        v4   workflow   latest: v7    active runs: 3
  billing-specialist    v4   workflow   latest: v2    active runs: 0
  technical-specialist  v4   workflow   latest: v4    active runs: 1
  general-support       v4   workflow   latest: v1    active runs: 0
  support-chatbot       v5   chatbot    latest: v3    active runs: —

5 agent(s).
```

---

### `agent-platform agents diff`

Shows the diff between two versions of an agent.

```
agent-platform agents diff <agent-name> <version-a> <version-b> [--namespace <ns>]

$ agent-platform agents diff support-triage v6 v7

Diff: support-triage v6 → v7
─────────────────────────────────────────
  system_prompt:
    - "You are a precise support triage assistant. Classify requests accurately."
    + "You are a precise support triage assistant. Classify requests accurately and route to the right specialist."

  workflow.steps[0].model_config_override.temperature:
    - 0.1
    + 0.0

  workflow.steps[2].cases[1].match:
    (unchanged: "technical")
  workflow.steps[2].cases added:
    + { "match": "admin", "next": "call_admin_agent" }

3 change(s).
```

---

## Editor Integration

### JSON Schema Autocomplete

The platform publishes JSON Schema documents for each DSL version at a well-known URL:

```
GET /schemas/dsl/v{N}.json
```

Adding this to your editor's JSON schema mapping (VS Code, JetBrains, Neovim LSP) enables:
- Field autocomplete.
- Inline validation with underlines.
- Hover documentation for each field.

### Pre-Commit Hook

Add to `.pre-commit-config.yaml`:

```yaml
- repo: local
  hooks:
    - id: agent-platform-validate
      name: Validate agent DSL files
      entry: agent-platform validate
      language: system
      files: \.agent\.json$
      pass_filenames: true
```

---

## CI Pipeline Integration

```yaml
# Example GitHub Actions step
- name: Validate agent definitions
  run: |
    find ./agents -name "*.agent.json" | xargs -I{} agent-platform validate {} --json --strict
```

Exit code `1` from any validation failure causes the CI step to fail.

---

## Configuration

The CLI reads platform connection details from (in priority order):

1. CLI flags (`--platform-url`, `--api-key`).
2. Environment variables: `AGENT_PLATFORM_URL`, `AGENT_PLATFORM_API_KEY`.
3. Config file: `~/.agent-platform/config.json` or `.agent-platform.json` in the current directory.

```json
{
  "platform_url": "https://platform.acme-internal.com",
  "api_key": "sk-...",
  "default_namespace": "acme-corp"
}
```

Commands that require a live platform (`tools list`, `mcps list`, `agents list`, `agents diff`) fail cleanly with a clear error if not configured. `validate` and `dry-run` work offline (using a bundled copy of the validation service).

---

## Implementation Notes

- The CLI bundles the validation service as a library dependency — it does not make HTTP calls for `validate` or `dry-run`.
- The validation service's library interface is the same one the provisioning API uses. A single implementation, two transports.
- `dry-run` evaluates `transform`, `branch`, `switch`, `while`, and `until` steps using actual expression evaluation (not mocks), because these are deterministic. Only LLM/tool/MCP/agent steps use mocks.
- The `--verbose` flag on `dry-run` prints the full envelope (including `metadata`) at each step; default output shows only the `output` portion.
- `--estimate-cost`: uses `max_tokens` declared in `llm_call` steps and the provider pricing table configured in the platform (or a bundled default table) to produce a worst-case cost estimate per step and total.
- `--cassette`: cassette files follow the format from `11-testing-strategy.md`. When a cassette is available for a step, it is used in preference to mock files.

---

## Adding a New Provider (E-24)

This runbook documents how a Platform Engineer adds support for a new LLM provider (e.g., Mistral, Grok, Gemini).

### 1. Implement the Provider Interface

Every provider adapter must implement the following interface (pseudocode):

```
interface LLMProvider {
  complete(request: CompletionRequest): CompletionResponse
  stream(request: CompletionRequest): AsyncIterable<CompletionChunk>
  probe(): ProbeResult                     // lightweight reachability check
  supportedModels(): string[]              // list of model_name values this adapter accepts
  formatToolCall(tool: Tool): ProviderToolFormat  // translate platform tool to provider format
  parseToolCallResponse(raw: any): ToolCallResult[]
}

type CompletionRequest = {
  system_prompt: string
  user_prompt: string
  model_config: ModelConfig
  tools?: ToolDefinition[]
  credentials: CredentialsBundle
}

type CompletionResponse = {
  text: string
  tool_calls: ToolCallResult[]
  tokens: TokenCounts
  finish_reason: "stop" | "tool_use" | "max_tokens" | "error"
  raw_response: any   // stored for observability hook
}
```

### 2. Required Unit Tests

Every provider adapter must pass:
- `test_completion_success` — happy path with a real or cassette response
- `test_completion_rate_limit_error` — `RateLimitError` is correctly surfaced
- `test_completion_timeout` — timeout produces a retryable `TimeoutError`
- `test_tool_call_round_trip` — a tool call is formatted, sent, and parsed correctly
- `test_streaming_mode` — `stream()` produces well-formed chunks
- `probe_returns_available` — `probe()` returns a `ProbeResult` within 5s
- `probe_returns_unavailable` — `probe()` returns a non-success result gracefully when offline (use a cassette)

### 3. Register the Provider Slug

1. Add the slug to the provider enum in the DSL schema (e.g., `"mistral"`). The validation service will then accept it in agent definitions.
2. Add the adapter to the provider registry (the factory/map that `ProviderFactory.get(slug)` uses).
3. Add a row to the platform's `providers` configuration table (or config file) documenting its pricing for `--estimate-cost`.

### 4. Cassette Directory

Create `test/cassettes/<provider_slug>/` and add at minimum:
- `completion_success.json`
- `completion_rate_limit.json`
- `tool_call_round_trip.json`

These cassettes are used in CI to test the adapter without live credentials.

### 5. Example

```
# Verify the new adapter works end-to-end:
agent-platform dry-run ./agents/hello-world.agent.json \
  --input '{"message": "Hello"}' \
  --live-llm \
  --estimate-cost
```

The `provider` field in `hello-world.agent.json` should be set to the new slug.

---

## What to Read Next

- **[14-dsl-spec.md](./14-dsl-spec.md)** — The complete DSL reference the CLI validates against
- **[11-testing-strategy.md](./11-testing-strategy.md)** — Using dry-run mocks in CI and test suites
- **[02-v1](./02-v1-agent-provisioning-foundation.md)** — The provisioning API the CLI wraps

---

## Related Documents

- [14-dsl-spec.md](./14-dsl-spec.md) — DSL specification and validation error reference
- [11-testing-strategy.md](./11-testing-strategy.md) — Testing strategy including contract and evolution tests
- [01-PRD](./01-PRD.md) — AD-12 (validation as a separate service), AD-13 (collect-all errors)
