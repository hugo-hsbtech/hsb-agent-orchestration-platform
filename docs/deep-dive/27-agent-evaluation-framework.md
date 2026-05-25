# Agent Evaluation and Quality Framework

> **Status**: Enhancement proposal (post-v6). Implements E-05 evaluation infrastructure referenced in `18-enhancements.md`.
>
> **Depends on**: v6 versioning system, workflow run persistence, DSL specification.

---

## Purpose

Provide a systematic, automated framework for measuring agent performance, detecting quality regressions, and ensuring agents meet defined quality standards before promotion to production. This closes the gap between "agent works" and "agent works well."

---

## Business Value

- **Regression Prevention**: Catch quality degradation before it reaches users
- **Objective Comparison**: Compare agent versions with metrics, not guesswork  
- **Continuous Improvement**: Track quality trends over time
- **Confidence in Deployment**: Automated gates prevent bad deployments

---

## Core Concepts

### Test Case
A single evaluation scenario with defined inputs and expected outcomes:

```json
{
  "id": "payment-intent-001",
  "name": "Payment status inquiry",
  "description": "User asks about a pending payment",
  "input": {
    "user_message": "Where is my payment? I sent it yesterday.",
    "conversation_context": [],
    "user_profile": { "tier": "premium" }
  },
  "expected_behavior": {
    "routing_classification": "payments",
    "response_must_contain": ["payment", "status", "track"],
    "response_must_not_contain": ["I don't know", "contact support"],
    "max_response_tokens": 200,
    "tool_calls_expected": ["lookup_payment_status"]
  },
  "difficulty": "easy",
  "tags": ["payments", "status-inquiry"]
}
```

### Evaluation Suite
A collection of test cases covering a domain:

| Suite Type | Purpose | Size |
|---|---|---|
| **Smoke** | Quick validation, core functionality | 10-20 cases |
| **Regression** | Previously failed scenarios | 50-100 cases |
| **Comprehensive** | Full coverage, pre-release gating | 500+ cases |
| **Adversarial** | Edge cases, adversarial inputs | 50-100 cases |

### Scoring Dimensions

| Dimension | Metric | Weight |
|---|---|---|
| **Correctness** | Output matches expected (exact/semantic) | 40% |
| **Completeness** | All required elements present | 20% |
| **Conciseness** | Token efficiency vs. gold standard | 15% |
| **Safety** | No disallowed content | 15% |
| **Latency** | Response time within budget | 10% |

---

## Epic 1 — Test Case Management

### Story 1.1 — Test case DSL
**As an** Agent Builder  
**I want** to define test cases in a structured format  
**So that** I can version and share evaluation criteria alongside agent definitions.

**Description**: A JSON/YAML format for test cases stored in version control. Cases reference agents by ID and declare assertions about expected behavior.

**Acceptance Criteria**:
- Test cases can be loaded from JSON files or defined via API
- Cases support multiple assertion types: exact match, contains, regex, JSON path, semantic similarity
- Cross-references to agent definitions are validated at load time
- Suite files can import and compose other suites

### Story 1.2 — Golden response management
**As an** Agent Builder  
**I want** to maintain "golden" reference responses that represent ideal output  
**So that** I have a benchmark for semantic comparison.

**Description**: Store reference responses per test case. Support multiple golden variants for responses that have equivalent meaning but different phrasing.

**Acceptance Criteria**:
- Golden responses are stored per test case version
- Semantic similarity scoring using embeddings (configurable threshold)
- Golden set can be updated based on human-reviewed good responses
- Version history of golden responses is maintained

### Story 1.3 — Test case versioning and lineage
**As an** Operator  
**I want** to track which test cases covered which agent versions  
**So that** I can audit what was tested when.

**Description**: Test cases are versioned independently of agents. Linkage table records which suite ran against which agent version.

**Acceptance Criteria**:
- Test case changes produce new versions
- Test runs record exact suite version used
- Diff view shows how test coverage evolved

---

## Epic 2 — Automated Evaluation Execution

### Story 2.1 — Evaluation run execution
**As an** Agent Builder  
**I want** to run evaluation suites against agent versions automatically  
**So that** I get quantitative quality metrics.

**Description**: Execute test cases against a target agent version. Each case produces a scored result. Aggregate metrics computed across the suite.

**Execution modes**:
- **Synchronous**: Run and return results immediately (small suites)
- **Asynchronous**: Queue large suites, return run ID for polling
- **CI Integration**: Trigger on agent version creation via webhook

**Acceptance Criteria**:
- Single test case execution returns pass/fail with detailed assertions
- Suite execution returns aggregate score + per-case breakdown
- Evaluation runs appear in workflow runs table with `run_type: evaluation`
- Parallel execution up to configurable concurrency limit

### Story 2.2 — Assertion engines
**As an** Agent Builder  
**I want** multiple ways to assert correctness  
**So that** I can match the assertion to the expected behavior type.

**Assertion Types**:

| Type | Description | Example |
|---|---|---|
| `exact` | String or JSON exact match | `output.text == "Hello"` |
| `contains` | Substring match | `"payment" in output.text` |
| `regex` | Pattern match | `output.text matches /\$\d+\.\d{2}/` |
| `json_path` | JSON path exists/equals | `$.tool_calls[0].name == "lookup_payment"` |
| `semantic_similarity` | Embedding cosine similarity | `sim(output.text, golden) > 0.85` |
| `llm_judge` | LLM-as-judge evaluation | `llm_eval(output, criteria) > 0.7` |
| `structured_output` | Schema validation | `output validates against schema` |
| `routing_correct` | Classification match | `actual_route == expected_route` |
| `tool_sequence` | Tool call ordering | `tools_called == ["A", "B", "C"]` |
| `latency_budget` | Response time check | `duration_ms < max_latency_ms` |

**Acceptance Criteria**:
- All assertion types are implemented and documented
- Assertions can be combined with AND/OR logic
- Custom assertion plugins can be registered

### Story 2.3 — LLM-as-judge evaluation
**As an** Agent Builder  
**I want** to use an LLM to evaluate subjective qualities  
**So that** I can assess dimensions like "helpfulness" or "tone."

**Description**: For subjective criteria, use a separate judge LLM with a rubric. Returns 0-1 score with rationale.

**Judge Configuration**:
```json
{
  "assertion_type": "llm_judge",
  "criteria": "Response is helpful and acknowledges the user's frustration",
  "rubric": {
    "1.0": "Directly addresses concern with empathy and clear next steps",
    "0.7": "Addresses concern but lacks empathy or clarity",
    "0.4": "Partially addresses concern",
    "0.0": "Does not address concern or is dismissive"
  },
  "judge_model": "claude-3-5-sonnet-20241022",
  "min_score": 0.7
}
```

**Acceptance Criteria**:
- Judge LLM calls are traced separately from agent under test
- Rubric-based scoring with human-calibrated examples
- Multiple judge LLMs can be configured for consensus scoring
- Judge prompts are versioned and auditable

---

## Epic 3 — Quality Gates and CI/CD Integration

### Story 3.1 — Pre-deployment quality gates
**As a** Platform Admin  
**I want** to require minimum evaluation scores before version promotion  
**So that** only quality agents reach production.

**Description**: Configure quality gates that block version promotion (draft → staging → canary → production) unless evaluation criteria are met.

**Gate Configuration**:
```json
{
  "quality_gate": {
    "required_suites": ["smoke", "regression"],
    "minimum_scores": {
      "overall": 0.90,
      "correctness": 0.95,
      "safety": 1.00
    },
    "max_regression_count": 0,
    "auto_promote_on_pass": false
  }
}
```

**Acceptance Criteria**:
- Gates are checked at promotion time
- Failed gates block promotion with clear error report
- Gate configuration is per-namespace with platform defaults
- Emergency override requires explicit approval and is audited

### Story 3.2 — A/B test framework for agent versions
**As an** Agent Builder  
**I want** to route a percentage of traffic to different agent versions  
**So that** I can measure real-world performance before full rollout.

**Description**: Configure traffic splitting between agent versions. Collect metrics on each variant. Statistical comparison with confidence intervals.

**Acceptance Criteria**:
- Traffic split configurable per agent (e.g., 90% v1, 10% v2)
- Automatic metric collection: completion rate, user satisfaction, cost
- Statistical significance testing (p-value, confidence interval)
- Auto-rollback on statistically significant degradation

### Story 3.3 — Regression detection alerts
**As an** Operator  
**I want** to be notified when evaluation scores drop  
**So that** I can investigate quality issues quickly.

**Description**: Continuous monitoring of evaluation results. Alert when scores drop below thresholds or when new failures appear in regression suites.

**Alert Conditions**:
- Overall score drops > 5% from baseline
- Any safety assertion fails
- Previously passing test now fails (regression)
- Evaluation run fails to complete (system error)

**Acceptance Criteria**:
- Alerts route to configured channels (email, Slack, PagerDuty)
- Alert includes diff of failed assertions
- Alert severity based on failure type and agent tier

---

## Epic 4 — Analytics and Reporting

### Story 4.1 — Evaluation dashboard
**As an** Agent Builder  
**I want** a visual dashboard of evaluation results  
**So that** I can track quality trends over time.

**Dashboard Views**:
- Score trends per agent version
- Pass/fail breakdown by assertion type
- Heatmap of test case results
- Comparison between versions

**Acceptance Criteria**:
- Dashboard updates in real-time as evaluations complete
- Drill-down from aggregate to individual test case
- Exportable reports (PDF, CSV)

### Story 4.2 — Failure analysis tools
**As an** Operator  
**I want** to understand why a test case failed  
**So that** I can fix the agent or the test.

**Analysis Features**:
- Side-by-side comparison: expected vs. actual
- Assertion-level failure reasons
- Full trace of agent execution (if workflow agent)
- Suggested fixes based on failure patterns

**Acceptance Criteria**:
- Failed test case shows detailed diff
- Execution trace linked for debugging
- "Debug this case" button to re-run in debug mode

---

## Technical Considerations

### Scoring Consistency
- Embedding models for semantic similarity must be version-pinned
- LLM-as-judge should use temperature=0 and fixed seed where possible
- Multiple runs of same test should produce identical scores (deterministic)

### Performance at Scale
- Evaluation runs are workflows; leverage Temporal for distribution
- Parallel execution: one activity per test case, up to `max_concurrency`
- Cache embedding computations for golden responses
- Judge LLM calls batched where provider supports it

### Storage
- Test cases stored in database: `evaluation_suites`, `evaluation_cases`
- Results stored: `evaluation_runs`, `evaluation_results`
- Golden responses in blob storage with DB references

### Integration Points
- Provisioning API: `POST /agents/{id}/evaluate` trigger
- Version promotion API: checks gates before state change
- CLI: `agent-platform evaluate --suite regression --agent my-agent`

---

## DSL Extensions

### Agent Definition Additions
```json
{
  "evaluation_config": {
    "golden_suite": "suite://customer-support-comprehensive",
    "auto_evaluate_on_save": true,
    "quality_gate": {
      "min_overall_score": 0.90,
      "required_dimensions": ["correctness", "safety"]
    }
  }
}
```

### Test Case DSL Format
```yaml
# payment-inquiry.yml
id: payment-intent-001
name: Payment status inquiry
description: User asks about a pending payment
input:
  user_message: "Where is my payment? I sent it yesterday."
  conversation_context: []
expected:
  routing_classification: payments
  assertions:
    - type: contains
      path: response.text
      value: payment
    - type: llm_judge
      criteria: "Response is helpful and provides clear next steps"
      min_score: 0.8
    - type: latency_budget
      max_ms: 2000
tags: [payments, status-inquiry]
difficulty: easy
```

---

## Definition of Done

- Test cases can be defined, versioned, and loaded
- Evaluation runs execute and produce scored results
- Multiple assertion types work including LLM-as-judge
- Quality gates block/fail version promotion appropriately
- A/B testing framework routes traffic and measures outcomes
- Dashboard shows quality trends and failure analysis
- CLI supports evaluation commands
- Documentation includes evaluation best practices

---

## Related Documents

- [14-dsl-spec.md](./14-dsl-spec.md) — DSL schema extensions for evaluation config
- [20-agent-version-promotion.md](./20-agent-version-promotion.md) — Quality gates integrate with promotion workflow
- [11-testing-strategy.md](./11-testing-strategy.md) — Cassettes as foundation for test cases
- [18-enhancements.md](./18-enhancements.md) — E-05 cross-agent validation, E-26 evaluation framework
