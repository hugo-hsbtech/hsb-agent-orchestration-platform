# Content Safety and PII Protection Layer

> **Status**: Enhancement proposal (post-v6). Addresses enterprise security and compliance requirements.
>
> **Depends on**: v6 observability infrastructure, workflow run persistence, chatbot layer.

---

## Purpose

Provide comprehensive safety guardrails for agent inputs and outputs, including content moderation, PII detection and handling, and audit logging for compliance. This ensures the platform meets security baselines for production deployment in regulated environments.

---

## Business Value

- **Risk Mitigation**: Prevent harmful, toxic, or inappropriate content generation
- **Compliance**: Meet regulatory requirements (GDPR, CCPA, HIPAA, SOC2)
- **Trust**: Users and customers trust agents with their data
- **Operational Safety**: Automated safeguards reduce incident response burden

---

## Core Concepts

### Safety Layers

| Layer | When | Purpose |
|---|---|---|
| **Input Guardrails** | Before LLM call | Block harmful requests, detect prompt injection |
| **Output Guardrails** | After LLM call | Filter toxic or inappropriate responses |
| **PII Detection** | Both directions | Identify and handle sensitive personal information |
| **Audit Logging** | Continuous | Immutable record for compliance investigations |

### PII Categories

| Category | Examples | Handling Options |
|---|---|---|
| **Personal Identifiers** | SSN, passport, driver's license | Redact, block, encrypt |
| **Financial** | Credit card, bank account | Redact, tokenize |
| **Contact** | Email, phone, address | Mask, pseudonymize |
| **Health** | Medical records, diagnoses | Block, encrypt, strict audit |
| **Biometric** | Fingerprints, facial data | Block, strictest controls |

---

## Epic 1 — Content Moderation

### Story 1.1 — Input content moderation
**As a** Platform Admin  
**I want** to screen user inputs for harmful content before they reach agents  
**So that** agents don't process toxic, hateful, or dangerous requests.

**Description**: Analyze user messages through moderation APIs (Perspective API, Azure Content Safety, AWS Comprehend). Block or flag based on configurable thresholds.

**Moderation Categories**:
- Toxicity
- Severe toxicity
- Threats
- Insults
- Profanity
- Sexual content
- Violence
- Self-harm

**Actions on Violation**:
- `block`: Return error, don't process
- `flag`: Allow but mark for review, log at higher level
- `sanitize`: Attempt to clean (e.g., profanity filter)

**Acceptance Criteria**:
- All user inputs pass through moderation check
- Configurable thresholds per category per namespace
- Violations logged with severity and category
- Response to user explains why input was blocked (optional)

### Story 1.2 — Output content moderation
**As a** Platform Admin  
**I want** to screen agent responses for inappropriate content  
**So that** harmful outputs never reach users.

**Description**: Post-process LLM responses through same moderation pipeline. Filter or regenerate if violations detected.

**Violation Actions**:
- `filter`: Replace with fallback message
- `regenerate`: Retry with adjusted prompt (max attempts configurable)
- `escalate`: Route to human review

**Acceptance Criteria**:
- All agent responses moderated before delivery
- Regeneration attempts tracked to prevent loops
- Escalation triggers human handoff workflow

### Story 1.3 — Prompt injection detection
**As a** Platform Admin  
**I want** to detect and block prompt injection attempts  
**So that** attackers cannot override system instructions.

**Detection Methods**:
- Pattern matching (e.g., "ignore previous instructions")
- Delimiter analysis (unclosed quotes, code blocks)
- Semantic analysis (intent classification)
- Known attack signature database

**Acceptance Criteria**:
- Common prompt injection patterns detected
- Indirect injection (via retrieved content) flagged
- Detection confidence score, block at high confidence
- False positive rate monitored, model tuned

---

## Epic 2 — PII Detection and Handling

### Story 2.1 — PII detection in inputs
**As a** Platform Admin  
**I want** to detect PII in user messages  
**So that** I can apply appropriate handling policies.

**Detection Engine**:
- Regex patterns for known formats (SSN, CC, phone)
- Named Entity Recognition (NER) for names, locations
- ML-based PII classifiers
- Custom patterns per namespace

**Detection Output**:
```json
{
  "pii_detected": [
    {
      "type": "credit_card",
      "value": "4111111111111111",
      "position": [45, 61],
      "confidence": 0.98
    },
    {
      "type": "email",
      "value": "john@example.com",
      "position": [72, 88],
      "confidence": 0.95
    }
  ]
}
```

**Acceptance Criteria**:
- All major PII types detected with high accuracy
- False positive rate < 5% for common patterns
- Detection latency < 100ms

### Story 2.2 — PII handling policies
**As a** Platform Admin  
**I want** to configure how detected PII is handled  
**So that** I can balance utility and privacy per data type.

**Handling Modes**:

| Mode | Description | Use Case |
|---|---|---|
| `block` | Reject input containing PII | Public chatbots, strict compliance |
| `redact` | Replace with `[REDACTED]` | Log storage, analytics |
| `tokenize` | Replace with reversible token | Internal processing, need to recover |
| `encrypt` | Encrypt with namespace key | Long-term storage, audit logs |
| `mask` | Partial hide (e.g., `***-**-6789`) | Display, support workflows |
| `passthrough` | Allow with audit logging | Trusted environments, logged consent |

**Acceptance Criteria**:
- Policy configurable per PII category per namespace
- Policy enforced at detection point
- Handling action recorded in audit log

### Story 2.3 — PII detection in outputs
**As a** Platform Admin  
**I want** to detect if agents inadvertently include PII in responses  
**So that** we don't leak sensitive data.

**Description**: Post-process agent outputs for PII. This catches cases where the agent echoes input PII or retrieves PII from knowledge base.

**Acceptance Criteria**:
- Output PII detected with same engine as input
- Policy can differ from input (e.g., mask in output vs block input)
- Violations logged as potential data leak

### Story 2.4 — Conversation PII audit trail
**As a** Compliance Officer  
**I want** an immutable record of all PII detected and handled  
**So that** I can demonstrate compliance during audits.

**Audit Record**:
```json
{
  "audit_id": "uuid",
  "timestamp": "2024-01-15T10:30:00Z",
  "conversation_id": "conv_123",
  "turn_id": "turn_456",
  "direction": "input|output",
  "pii_findings": [
    {
      "type": "ssn",
      "action": "redacted",
      "policy_applied": "strict_financial"
    }
  ],
  "hash": "sha256...",
  "integrity_proof": "..."
}
```

**Acceptance Criteria**:
- Every PII detection recorded immutably
- Tamper-evident logging (hash chain)
- Exportable reports for auditors
- Retention policy enforcement (auto-delete after TTL)

---

## Epic 3 — Safety Configuration and Governance

### Story 3.1 — Safety policy DSL
**As a** Platform Admin  
**I want** to declare safety policies in agent/agent-type configuration  
**So that** safety is configurable per use case.

**Policy Schema**:
```json
{
  "safety_policy": {
    "content_moderation": {
      "provider": "azure_content_safety",
      "input_action": "block",
      "output_action": "filter",
      "thresholds": {
        "toxicity": 0.7,
        "violence": 0.5,
        "self_harm": 0.1
      }
    },
    "pii_handling": {
      "input_policy": {
        "credit_card": "block",
        "ssn": "block",
        "email": "mask"
      },
      "output_policy": {
        "credit_card": "redact",
        "ssn": "redact",
        "email": "mask"
      }
    },
    "prompt_injection": {
      "detection_enabled": true,
      "action": "block",
      "min_confidence": 0.8
    },
    "audit_logging": {
      "level": "all_pii",
      "retention_days": 2555
    }
  }
}
```

**Acceptance Criteria**:
- Policies validated at agent provisioning time
- Unknown providers/configs fail validation
- Policy changes require explicit deployment

### Story 3.2 — Namespace-level safety defaults
**As a** Platform Admin  
**I want** to set default safety policies at the namespace level  
**So that** all agents in my organization inherit appropriate baselines.

**Description**: Namespace configuration specifies default safety policies. Individual agents can override with stricter settings but cannot relax below namespace minimum.

**Acceptance Criteria**:
- Namespace settings apply to all agents by default
- Agent can override to be stricter
- Attempt to relax below namespace minimum is rejected

### Story 3.3 — Safety bypass and emergency procedures
**As an** Operator  
**I want** emergency procedures to bypass safety controls when necessary  
**So that** critical operations aren't blocked by false positives.

**Emergency Actions**:
- Temporarily disable moderation for specific conversation
- Whitelist specific user/account from PII blocking
- Emergency access for incident response

**Governance**:
- All bypasses require explicit authorization
- All bypasses heavily audited
- Automatic reversion after timeout
- Alert on any bypass usage

**Acceptance Criteria**:
- Bypass mechanism exists with strict controls
- Bypass authorization workflow defined
- Automatic alerts and audit logging

---

## Epic 4 — Compliance and Audit

### Story 4.1 — Data retention policy enforcement
**As a** Compliance Officer  
**I want** automatic enforcement of data retention policies  
**So that** we comply with GDPR/CCPA deletion requirements.

**Retention Policies**:
- Conversation data: configurable TTL (default 90 days)
- PII audit logs: longer retention (7 years for some regulations)
- Evaluation data: medium retention (1 year)
- Workflow runs: based on namespace tier

**Deletion Procedures**:
- Soft delete (mark deleted, purge after grace period)
- Hard delete (immediate, irreversible)
- Export before deletion (user data portability)

**Acceptance Criteria**:
- Retention policies configurable per namespace per data type
- Automatic purging runs on schedule
- Proof of deletion for audit
- User-initiated deletion requests (right to erasure)

### Story 4.2 — Compliance reporting exports
**As a** Compliance Officer  
**I want** to export compliance reports  
**So that** I can provide evidence to auditors.

**Report Types**:
- Data processing activities log
- PII detection and handling summary
- Safety violation incidents
- Retention policy compliance proof
- Access logs and authorization changes

**Acceptance Criteria**:
- Reports exportable in standard formats (PDF, CSV, JSON)
- Date range filtering
- Digital signatures for integrity
- API endpoint for automated compliance systems

### Story 4.3 — Data residency controls
**As a** Platform Admin  
**I want** to control where safety/PII data is processed and stored  
**So that** I meet data sovereignty requirements.

**Description**: Configure processing regions for PII detection and audit log storage. Pin to specific geographic regions.

**Acceptance Criteria**:
- Processing region configurable per namespace
- Audit logs stored in specified region
- No cross-border transfer for sensitive data
- Documentation of data flow for compliance

---

## Technical Considerations

### Integration Points
- Content moderation as pre/post-activity hooks in workflow
- PII detection as middleware in chatbot streaming layer
- Audit logging to append-only store (separate from operational DB)

### Performance
- Content moderation calls parallel to other processing where possible
- PII detection regex patterns compiled and cached
- Async audit logging (don't block response)

### Provider Support
- Azure Content Safety (primary)
- AWS Comprehend (alternative)
- Google Perspective API (alternative)
- Open-source models (fallback, self-hosted)

### Security
- Audit logs cryptographically signed
- Tamper detection via hash chain
- Access to audit logs strictly controlled
- Encryption at rest for all PII data

---

## DSL Extensions

```json
{
  "safety_policy": {
    "content_moderation": {
      "enabled": true,
      "provider": "azure_content_safety",
      "input_action": "block",
      "output_action": "filter",
      "thresholds": {
        "toxicity": 0.7,
        "severe_toxicity": 0.5,
        "threats": 0.3,
        "self_harm": 0.1
      }
    },
    "pii_detection": {
      "engine": "microsoft_presidio",
      "input_policy": {
        "credit_card": { "action": "block", "alert": true },
        "ssn": { "action": "block", "alert": true },
        "email": { "action": "mask", "alert": false }
      },
      "output_policy": {
        "credit_card": { "action": "redact", "alert": true },
        "ssn": { "action": "redact", "alert": true },
        "email": { "action": "mask", "alert": false }
      }
    },
    "prompt_injection": {
      "enabled": true,
      "min_confidence": 0.8,
      "action": "block"
    }
  }
}
```

---

## Definition of Done

- Content moderation integrated for inputs and outputs
- PII detection working for all major categories
- Handling policies configurable and enforced
- Audit logging tamper-evident and exportable
- Retention policies enforced automatically
- Emergency bypass procedures defined and audited
- Compliance reporting available
- Documentation includes security best practices

---

## Related Documents

- [09-security-model.md](./09-security-model.md) — Threat model and security principles
- [16-multi-tenancy.md](./16-multi-tenancy.md) — Namespace isolation for safety policies
- [22-chatbot-degradation.md](./22-chatbot-degradation.md) — Safety as degradation mode
- [13-chatbot-ux-edge-cases.md](./13-chatbot-ux-edge-cases.md) — Safety-related edge cases
