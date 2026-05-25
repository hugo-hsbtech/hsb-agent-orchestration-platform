# Testing Strategy

## Purpose

This document defines the conceptual testing framework for the agent platform. It establishes testing layers, quality gates, and validation principles without prescribing specific test frameworks or commands.

---

## Testing Philosophy

### Four-Layer Testing Model

| Layer | What It Validates | When It Runs |
|---|---|---|
| **Unit** | Individual functions, adapters, transformers | Every code change |
| **Integration** | Component interactions (DB, Temporal, providers) | Pre-merge, in CI |
| **Contract** | DSL schema compliance, API contracts | Pre-merge, in CI |
| **E2E (Evolution)** | Full agent behaviors, conversation flows | Nightly, pre-release |

**Principle**: Lower layers catch regressions fast; upper layers validate user-facing behavior.

---

## Unit Testing

### Agent Core Isolation

The Agent Core (provider-agnostic LLM execution) is designed for testability:
- **Mock providers**: Simulate Claude, OpenAI, etc. without network calls
- **Deterministic prompts**: Input → output mapping is reproducible
- **No side effects**: Core has no database, HTTP, or logging dependencies

**Test Categories**:
- Provider adapter translation (internal format ↔ provider format)
- Prompt template interpolation (variable substitution, missing variable handling)
- Tool call response parsing (provider-specific formats)
- Error classification (retryable vs. non-retryable)

### Validation Service Testing

The standalone validation service is exhaustively unit-tested:
- Schema compliance (valid/invalid DSL documents)
- Cross-reference validation (tool names, MCP names, sub-agent references)
- Data flow validation (input/output type compatibility)
- Error accumulation (multiple errors returned, not fail-fast)

---

## Integration Testing

### Database Persistence

- **Schema migrations**: Up and down migrations produce consistent state
- **Transaction boundaries**: Workflow run updates are atomic
- **Query performance**: Index usage on large run tables
- **Connection handling**: Pool behavior under load

### Temporal Workflow Execution

- **Worker startup**: Registration, task queue binding
- **Activity execution**: Success, failure, retry, timeout paths
- **Workflow signals**: Pause, resume, cancel propagation
- **Child workflows**: Parent-child linkage, fire-and-forget semantics

### Provider Integration

- **Live API testing** (limited, with mocks preferred):
  - Rate limit handling
  - Error response parsing
  - Streaming response handling
- **Stub verification**: Each provider has stub that fails with clear message

### MCP and Tool Registry

- **Tool discovery**: Registry populates correctly at startup
- **MCP manifest caching**: Fetch, cache, invalidate cycles
- **Authentication injection**: Credentials flow through to invocations
- **Schema validation**: Tool inputs validated before handler execution

---

## Provider Test Doubles

LLM provider calls are non-deterministic and have real cost. This section establishes the policy for when to use live providers vs. test doubles, and how to manage recorded responses across model versions.

### Decision Matrix: Live vs. Double

| Test Layer | Default | When to Use Live |
|---|---|---|
| **Unit** | Always use mock | Never — Agent Core must be testable in isolation (AD-10) |
| **Integration (CI)** | Recorded cassette | Never in CI; only when cassette needs refresh |
| **Integration (local)** | Developer's choice | When debugging provider-specific behaviour |
| **E2E / Evolution (staging)** | Live providers | Always — evolution tests require real outputs |
| **Nightly** | Live providers | Full test set against staging with real credentials |

### Record/Replay (Cassette) Strategy

Provider responses are recorded once and replayed in CI. This gives determinism without network calls.

**Cassette format**: A cassette file records the request (prompt, model config) and the full provider response (text, token counts, finish reason, any tool calls). Secrets are stripped before storage.

**Cassette directory structure**:
```
tests/
  cassettes/
    claude/
      v3-7-sonnet/
        test_summarize_basic.json
        test_tool_call_kb_lookup.json
        test_streaming_response.json
    openai/
      gpt-4o/
        test_summarize_basic.json
```

**Versioning cassettes**: Cassettes are namespaced by provider AND model version. When a model version is updated (e.g. `claude-3-7-sonnet-20250219` → `claude-3-8-sonnet-20260101`), the affected cassettes must be re-recorded against the new model before CI is updated to use it. The cassette file name includes the model version slug to make this explicit.

**Cassette refresh procedure**:
1. Set `AGENT_PLATFORM_TEST_MODE=live` in local environment.
2. Run the affected test suite — calls go to the real provider and responses are written to cassette files.
3. Review diffs: confirm output quality is acceptable (responses may change between models).
4. Commit updated cassette files.
5. CI reverts to cassette replay mode automatically.

**When cassettes are stale**: If a test fails because the prompt changed but the cassette was not refreshed, the error message must say `STALE_CASSETTE: Request hash mismatch` with instructions to re-record. This is preferable to silently replaying a cassette that no longer matches the request.

### Provider Availability Check (v1 Story 3.2)

The provisioning-time provider availability check (verifying credentials are configured and the provider is reachable) should use a **minimal, cheap probe call** — not a full inference. Each provider adapter implements a `probe()` method that makes the smallest valid API call (e.g. a completion with `max_tokens: 1`). In CI, `probe()` is mocked to return `available: true`. In staging, real probes run.

### Stub Implementations

Each provider adapter ships with a stub implementation used in unit tests:

```
StubClaudeAdapter:
  - Returns a configured mock response synchronously.
  - Records every call made to it (for assertion: "was the prompt correct?").
  - Supports injecting errors (rate limit, timeout, auth failure) for error-path tests.
  - Supports streaming mode (yields tokens character-by-character with configurable delay).
```

The stub is the canonical tool for Agent Core unit tests and validation service tests that don't need real LLM output.

---

## Contract Testing

### DSL Schema Compliance

Every DSL version has a contract test suite:
- Valid documents pass validation
- Invalid documents fail with expected error categories
- Edge cases (empty strings, max-length values, unicode)
- Cross-version compatibility (v1 agents valid in v6 platform)

### API Contract Stability

Public endpoints have contract tests:
- Request/response schema validation
- HTTP status code expectations
- Header requirements (auth, content-type)
- Backward compatibility (old clients work with new API)

---

## E2E Testing (Evolution Testing)

### The Evolution Concept

Traditional E2E tests verify a fixed scenario. **Evolution tests** verify that agents improve correctly:
- **Baseline**: Agent version N produces output O for input I
- **Change**: Agent Builder updates prompt/tools
- **Evolution**: Agent version N+1 produces output O' that is "better" than O

**Challenge**: "Better" is often subjective. Evolution tests use:
- Reference outputs (human-approved gold standards)
- Evaluation rubrics (accuracy, relevance, safety)
- A/B comparison (N vs. N+1 on same test set)

### Conversation Flow Testing

Chatbot E2E tests simulate complete user interactions:
- **Single-turn**: User asks, bot responds, assertion on response
- **Multi-turn**: Context carries across turns
- **Routing**: User message routes to correct specialist
- **Delegation**: Chatbot triggers background task, later surfaces result
- **Interruption**: User changes topic mid-conversation

### Workflow Orchestration E2E

Full workflow tests:
- **Sequential**: Step 1 → Step 2 → ... → completion
- **Parallel**: Fan-out executes concurrently, results aggregate correctly
- **Branching**: Condition evaluation routes correctly
- **Looping**: While/until loops terminate, iteration counting correct
- **Sub-agent**: Parent waits (or doesn't) for child appropriately

---

## Test Data Management

### Fixture Strategy

| Data Type | Management Approach |
|---|---|
| **Agent definitions** | Version-controlled DSL files, parameterized for variations |
| **Provider responses** | Recorded (with secrets redacted) or hand-crafted mocks |
| **Database state** | Per-test transactions, rolled back after test |
| **Workflow runs** | Isolated test tenant, cleaned up periodically |
| **Conversations** | Synthetic sessions, pseudonymized if exported |

### Sensitive Data Handling

- No real API keys in test code
- No production conversation data in test fixtures
- Mock LLM responses for deterministic tests
- Sanitized provider response recordings

---

## Quality Gates

### Pre-Merge Gates

- All unit tests pass
- Integration tests pass (database, Temporal, registries)
- Contract tests pass (DSL, API)
- Code coverage meets threshold (conceptual; implementation defines percentage)
- No test skips without justification

### Pre-Release Gates

- All E2E/evolution tests pass
- Performance regression tests pass (latency within bounds)
- Load tests pass (concurrency handling)
- Security scan pass (no new vulnerabilities)
- Documentation updated for new behaviors

### Continuous Validation

- **Nightly**: Full E2E suite against staging environment
- **Weekly**: Evolution test suite with reference outputs
- **On-demand**: Load tests for major releases

---

## Testing Environments

### Environment Hierarchy

```
Development (local)   → Unit tests, fast feedback
  ↓
CI (ephemeral)        → Pre-merge integration, contract tests
  ↓
Staging (persistent)  → E2E tests, evolution tests, performance baseline
  ↓
Production (mirror) → Smoke tests, synthetic monitoring
```

### Staging Parity

Staging should mirror production in:
- Database schema and size (synthetic data at scale)
- Worker configuration (resource allocation)
- Provider integrations (real APIs or high-fidelity mocks)
- MCP connections (sandbox MCPs where available)

Differences from production:
- No real user data
- Relaxed rate limits (for test throughput)
- Debug endpoints enabled

---

## Regression Prevention

### Known Failure Catalog

Maintain a catalog of historical bugs with regression tests:
- Bug description and root cause
- Minimal reproduction case
- Fix verification
- Test that prevents recurrence

### Canary Testing

Pre-production validation:
- Deploy to subset of workers
- Route synthetic traffic
- Compare outputs between versions
- Automatic rollback on regression detection

---

## What to Read Next

Testing validates all platform layers:

- **[02-v1](./02-v1-agent-provisioning-foundation.md)** — Agent Core unit testability
- **[07-v6](./07-v6-versioning-and-hardening.md)** — Debug mode for operator testing
- **[08-Performance and Scaling](./08-performance-and-scaling.md)** — Load testing targets
- **[12-Upgrades and Migrations](./12-upgrades-and-migrations.md)** — Canary testing and validation

Or explore:
- [01-PRD](./01-PRD.md) — AD-10 (Agent Core minimalism) and testability requirements
---

## Operator Testing (Debug Mode)

### Step-by-Step Validation

The v6 debug mode serves as a manual testing tool:
- **Inspect**: Operator views step inputs before execution
- **Override**: Operator modifies inputs to test edge cases
- **Repeat**: Operator reruns step with same or modified inputs
- **Trace**: Operator follows data flow through workflow

### Production Debugging

When a workflow fails in production:
- Re-run in debug mode with same inputs
- Step through to identify failure point
- Compare against expected behavior from test fixtures

---

## Test Metrics and Observability

### Coverage Dimensions

| Dimension | What It Measures |
|---|---|
| **Code coverage** | Lines executed by tests |
| **Feature coverage** | DSL constructs exercised |
| **Scenario coverage** | User journey paths tested |
| **Error coverage** | Failure modes validated |
| **Provider coverage** | LLM providers tested |

### Flaky Test Management

- Quarantine flaky tests (mark pending, investigate)
- Root cause analysis (timing, external dependencies, state leakage)
- Fix or remove (no permanently quarantined tests)

---

## Out of Scope (Implementation Detail)

The following are deferred to implementation:
- Specific test frameworks (pytest, jest, etc.)
- Mock libraries (responses, wiremock, etc.)
- CI/CD pipeline configuration
- Code coverage tools and thresholds
- Test data generation utilities
- Environment provisioning automation

---

## Related Documents

- [02-v1 — Agent Provisioning Foundation](./02-v1-agent-provisioning-foundation.md) — Agent Core unit testability
- [07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md) — Debug mode for operator testing
- [08-Performance and Scaling](./08-performance-and-scaling.md) — Load testing targets
