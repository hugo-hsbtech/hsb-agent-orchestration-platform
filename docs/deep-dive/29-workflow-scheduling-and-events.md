# Workflow Scheduling and Event Triggers

> **Status**: Enhancement proposal (post-v6). Extends agent invocation beyond API calls.
>
> **Depends on**: v4 multi-agent orchestration, v6 workflow versioning, event infrastructure.

---

## Purpose

Extend agent invocation beyond manual API calls to include time-based scheduling and event-driven triggers. This enables automated recurring workflows (reports, maintenance, batch processing) and reactive workflows that respond to external system events.

---

## Business Value

- **Automation**: Agents run on schedule without human intervention
- **Reactiveness**: Respond to events in real-time (new tickets, data changes, alerts)
- **Batch Processing**: Process data collections on schedule (nightly ETL, weekly reports)
- **Integration**: Connect to external systems without polling

---

## Core Concepts

### Trigger Types

| Type | Pattern | Example |
|---|---|---|
| **Scheduled** | Cron expression | "Every Monday at 9 AM" |
| **Webhook** | HTTP POST to endpoint | GitHub push, Jira ticket created |
| **Event Bus** | Message queue subscription | Kafka, RabbitMQ, SQS events |
| **Database** | CDC (Change Data Capture) | Row inserted/updated |
| **File** | Storage event | S3 file uploaded |

### Trigger Configuration
Triggers are first-class resources, separate from agents:

```json
{
  "trigger_id": "daily-report-generation",
  "trigger_type": "scheduled",
  "enabled": true,
  "namespace_id": "acme-prod",
  "target_agent": {
    "agent_id": "sales-report-generator",
    "version": "latest",
    "input_mapping": {
      "date_range": "{{trigger.date_range}}",
      "report_type": "daily"
    }
  },
  "schedule": {
    "cron": "0 9 * * MON-FRI",
    "timezone": "America/New_York",
    "missed_execution_policy": "run_once|skip|run_all"
  },
  "execution_policy": {
    "max_concurrent_runs": 1,
    "timeout_seconds": 3600,
    "retry_on_failure": true
  }
}
```

---

## Epic 1 — Scheduled Workflows

### Story 1.1 — Cron-based scheduling
**As an** Agent Builder  
**I want** to schedule agents to run at specific times  
**So that** I can automate recurring tasks.

**Schedule Features**:
- Standard cron expressions (5-field Unix cron)
- Named presets ("hourly", "daily", "weekly", "monthly")
- Timezone specification per schedule
- Start/end dates for limited runs

**Acceptance Criteria**:
- Cron expressions validated at trigger creation
- Agent runs initiated at scheduled times
- Missed executions handled per policy
- Schedule changes take effect immediately

### Story 1.2 — Schedule management API
**As an** Agent Builder  
**I want** to create, update, pause, and delete schedules  
**So that** I can manage automated workflows.

**API Operations**:
- `POST /triggers/scheduled` — Create schedule
- `GET /triggers/scheduled/{id}` — Get schedule details
- `PUT /triggers/scheduled/{id}` — Update schedule
- `POST /triggers/scheduled/{id}/pause` — Pause schedule
- `POST /triggers/scheduled/{id}/resume` — Resume schedule
- `DELETE /triggers/scheduled/{id}` — Delete schedule
- `GET /triggers/scheduled/{id}/runs` — History of triggered runs

**Acceptance Criteria**:
- All CRUD operations functional
- Pause/resume prevents/frees execution
- History shows all triggered runs with status

### Story 1.3 — Schedule conflict and overlap handling
**As an** Operator  
**I want** predictable behavior when schedules overlap or run long  
**So that** resource usage stays controlled.

**Conflict Policies**:

| Policy | Behavior |
|---|---|
| `skip` | If previous still running, skip this occurrence |
| `queue` | Queue for execution after current completes |
| `parallel` | Allow multiple concurrent runs (respect `max_concurrent`) |
| `terminate` | Kill previous, start new |

**Acceptance Criteria**:
- Policy configurable per trigger
- Default is `skip` for safety
- Alerts when runs are skipped/terminated

---

## Epic 2 — Webhook Triggers

### Story 2.1 — Webhook endpoint provisioning
**As an** Agent Builder  
**I want** a unique URL to receive webhook events  
**So that** external systems can trigger my agents.

**Webhook Configuration**:
```json
{
  "trigger_type": "webhook",
  "webhook_config": {
    "endpoint_path": "github-pr-events",
    "auth_type": "hmac_signature",
    "auth_config": {
      "header": "X-Hub-Signature-256",
      "algorithm": "sha256",
      "secret_ref": "github_webhook_secret"
    },
    "payload_schema": {
      "type": "object",
      "required": ["action", "pull_request"]
    }
  }
}
```

**Generated URL**: `https://platform.example.com/webhooks/{namespace}/{trigger_id}`

**Acceptance Criteria**:
- Unique URL generated per webhook trigger
- URL stable across redeploys
- Auth methods: HMAC signature, API key, mTLS
- Invalid auth returns 401, valid triggers workflow

### Story 2.2 — Webhook payload transformation
**As an** Agent Builder  
**I want** to transform webhook payloads before they reach the agent  
**So that** the agent receives clean, expected input.

**Transformation DSL**:
```json
{
  "payload_transform": {
    "type": "jsonata|jq|javascript",
    "expression": "{
      'event_type': action,
      'pr_title': pull_request.title,
      'author': pull_request.user.login,
      'branch': pull_request.head.ref
    }"
  }
}
```

**Acceptance Criteria**:
- Transform applied before agent input
- Transform failures logged, trigger fails gracefully
- Input validation after transform

### Story 2.3 — Webhook delivery verification
**As an** Operator  
**I want** to verify webhook delivery and retry on failure  
**So that** no events are lost.

**Features**:
- Idempotency key support (prevent duplicate processing)
- Delivery logs (timestamp, status, payload hash)
- Automatic retry with backoff (configurable)
- Dead letter queue for repeated failures

**Acceptance Criteria**:
- Each delivery logged with unique ID
- Retries follow exponential backoff
- After max retries, event moved to DLQ
- Manual replay from DLQ supported

---

## Epic 3 — Event Bus Integration

### Story 3.1 — Message queue subscriptions
**As a** Platform Admin  
**I want** agents to subscribe to message queue topics  
**So that** we can react to events from Kafka, RabbitMQ, SQS.

**Supported Brokers**:
- Apache Kafka
- RabbitMQ (AMQP)
- AWS SQS/SNS
- Google Pub/Sub
- Azure Event Hub

**Subscription Config**:
```json
{
  "trigger_type": "event_subscription",
  "subscription_config": {
    "broker_type": "kafka",
    "connection": {
      "bootstrap_servers": "kafka:9092",
      "sasl_config_ref": "kafka_sasl"
    },
    "topic": "customer-events",
    "consumer_group": "agent-platform-prod",
    "filter": {
      "event_type": ["customer.created", "customer.updated"]
    }
  }
}
```

**Acceptance Criteria**:
- Connection to supported brokers
- Message filtering before agent trigger
- Ack/Nack handling (retry on failure)
- Consumer group coordination

### Story 3.2 — Event schema validation
**As an** Agent Builder  
**I want** to validate event schemas before triggering agents  
**So that** agents only receive expected event shapes.

**Features**:
- JSON Schema validation per event type
- Schema registry integration (Confluent, AWS Glue)
- Invalid events logged to DLQ, not processed
- Schema evolution handling (backward compatibility)

**Acceptance Criteria**:
- Schema validation at ingestion
- Clear error for schema mismatch
- Schema versions tracked

### Story 3.3 — Event ordering and exactly-once processing
**As a** Platform Admin  
**I want** exactly-once processing guarantees  
**So that** events don't trigger duplicate workflows.

**Mechanisms**:
- Idempotency keys from event metadata
- Deduplication window (e.g., 24 hours)
- Checkpoints for consumer offset management
- Transactional outbox pattern

**Acceptance Criteria**:
- Same event ID processed only once
- Ordering preserved per partition/key
- At-least-once delivery with idempotent dedup

---

## Epic 4 — Trigger Management and Observability

### Story 4.1 — Trigger registry and catalog
**As a** Platform Admin  
**I want** a catalog of all triggers across the platform  
**So that** I can manage and audit event-driven automation.

**Registry Features**:
- List all triggers by namespace, type, target agent
- Search and filter
- Trigger health status (last execution, error rate)
- Dependency graph (which triggers fire which agents)

**Acceptance Criteria**:
- Complete trigger inventory
- Health metrics visible
- Dependency visualization

### Story 4.2 — Trigger execution history
**As an** Operator  
**I want** to see the history of trigger executions  
**So that** I can debug why an agent didn't run.

**History Data**:
- Trigger timestamp
- Event/payload summary (sanitized)
- Execution result (success/failure)
- Linked workflow run ID
- Latency metrics

**Acceptance Criteria**:
- History queryable by time range, trigger, status
- Drill-down to workflow run details
- Export for audit

### Story 4.3 — Trigger alerting
**As an** Operator  
**I want** alerts for trigger failures and anomalies  
**So that** I can respond to automation issues.

**Alert Conditions**:
- Trigger execution failure rate > threshold
- Schedule missed (platform downtime)
- Webhook delivery failures
- Queue backlog growing
- Trigger disabled automatically (circuit breaker)

**Acceptance Criteria**:
- Alerts routed to configured channels
- Alert includes context for debugging
- Self-healing where possible (auto-retry)

---

## Technical Considerations

### Scheduler Implementation
- Use distributed scheduler (Temporal scheduled workflows, or Quartz/Temporal)
- Store schedule definitions in database
- Scheduler runs as separate service, pushes to Temporal
- High availability: scheduler clustered, leader election

### Webhook Security
- HMAC signature verification standard
- IP allowlisting option
- Rate limiting per webhook endpoint
- Payload size limits

### Event Processing
- Event consumers separate from workflow workers
- Backpressure handling (slow consumer protection)
- Horizontal scaling of event consumers
- Poison message handling

### Integration Points
- Trigger runs create `workflow_runs` rows with `triggered_by` metadata
- Same observability (traces, metrics) as API-initiated runs
- Version pinning: trigger uses agent version at creation or `latest`

---

## DSL Extensions

### Trigger Definition Schema
```json
{
  "trigger_id": "string",
  "namespace_id": "string",
  "trigger_type": "scheduled|webhook|event_subscription",
  "enabled": true,
  "description": "string",
  
  "target_agent": {
    "agent_id": "string",
    "version": "latest|specific",
    "input_mapping": {
      "field": "{{trigger.payload.field}}"
    }
  },
  
  "schedule": {
    "cron": "string",
    "timezone": "string",
    "start_date": "ISO8601",
    "end_date": "ISO8601"
  },
  
  "webhook_config": {
    "auth_type": "hmac|api_key|mtls",
    "auth_config": {},
    "payload_schema": {}
  },
  
  "subscription_config": {
    "broker_type": "kafka|rabbitmq|sqs|pubsub",
    "connection": {},
    "topic": "string",
    "filter": {}
  },
  
  "execution_policy": {
    "max_concurrent_runs": 1,
    "timeout_seconds": 3600,
    "retry_on_failure": true,
    "missed_execution_policy": "skip"
  }
}
```

---

## Definition of Done

- Scheduled triggers execute agents at configured times
- Webhook triggers receive and process HTTP events
- Event subscriptions consume from message queues
- Trigger execution history maintained
- Trigger health monitored and alerted
- Security controls (auth, rate limiting) implemented
- Documentation includes trigger best practices

---

## Related Documents

- [03-v2-workflow-orchestration.md](./03-v2-workflow-orchestration.md) — Workflow execution foundation
- [05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md) — Agent invocation patterns
- [10-deployment-concepts.md](./10-deployment-concepts.md) — Infrastructure for event processing
- [09-security-model.md](./09-security-model.md) — Webhook authentication patterns
