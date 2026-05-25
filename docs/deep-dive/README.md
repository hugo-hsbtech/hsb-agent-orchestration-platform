# Deep-Dive Reference Documents

These documents are **not required to follow the implementation roadmap**. They exist as reference material — read them when building or designing a specific component.

> For the phased plan and delivery roadmap, start at **[../README.md](../README.md)**.

---

## Conceptual & Supporting

| Document | When to Read |
|---|---|
| [08-performance-and-scaling](./08-performance-and-scaling.md) | Planning capacity, latency budgets, worker scaling |
| [09-security-model](./09-security-model.md) | Auth/authz design, threat model, tenant isolation |
| [10-deployment-concepts](./10-deployment-concepts.md) | Infrastructure topology, Kubernetes layout, ops |
| [11-testing-strategy](./11-testing-strategy.md) | Quality gates, test pyramid, integration test approach |
| [12-upgrades-and-migrations](./12-upgrades-and-migrations.md) | Schema migrations, backward compatibility, rollout strategy |
| [13-chatbot-ux-edge-cases](./13-chatbot-ux-edge-cases.md) | Streaming failure modes, reconnect, partial responses |

## Specifications

| Document | When to Read |
|---|---|
| [14-dsl-spec](./14-dsl-spec.md) | Building the validation service, workflow interpreter, or authoring DSL configs |
| [15-developer-tooling](./15-developer-tooling.md) | Building the `agent-platform` CLI (local validation, dry-run) |
| [16-multi-tenancy](./16-multi-tenancy.md) | DB schema design, API auth, multi-operator deployment — read before v1 |
| [17-operator-cli](./17-operator-cli.md) | Building the v6 Operator CLI or designing incident-response flows |
| [18-enhancements](./18-enhancements.md) | 39 enhancement proposals (E-01–E-39) across all phases and docs |

## Gap-Closure Documents

Detailed specs that fill gaps identified during review. Each anchored to a specific phase.

| Document | What It Closes |
|---|---|
| [19-provider-rate-limiting](./19-provider-rate-limiting.md) | LLM 429 handling, token bucket, `Retry-After` — v2/v3 gap |
| [20-agent-version-promotion](./20-agent-version-promotion.md) | Draft→staging→canary→production lifecycle, auto-rollback — v6 gap |
| [21-conversation-memory](./21-conversation-memory.md) | Episodic + semantic memory (Qdrant + DB) — v5 gap |
| [22-chatbot-degradation](./22-chatbot-degradation.md) | Circuit breakers, fallback routing, static degraded mode — v5 gap |
| [23-per-run-cost-cap](./23-per-run-cost-cap.md) | `budget` DSL field, mid-run enforcement, `cancel_with_partial_output` — v4 gap |
| [24-dsl-migration-tooling](./24-dsl-migration-tooling.md) | `agent-platform migrate`, migration catalog, compatibility shims |
| [25-shared-agent-catalogue](./25-shared-agent-catalogue.md) | Platform-level agent registry, `catalogue://` DSL references |

## Future Roadmap (v7+)

Capability areas for post-v6 development, grouped by theme.

| Document | Theme | What It Delivers |
|---|---|---|
| [26-v7-sketch](./26-v7-sketch.md) | Horizon | Post-v6 candidate areas and design constraints |
| [27-agent-evaluation-framework](./27-agent-evaluation-framework.md) | Quality | Test case management, LLM-as-judge, A/B evaluation |
| [28-content-safety-and-pii](./28-content-safety-and-pii.md) | Quality | Content moderation, PII detection, compliance audit trails |
| [29-workflow-scheduling-and-events](./29-workflow-scheduling-and-events.md) | Scale | Cron triggers, webhooks, event bus integration |
| [30-business-intelligence-analytics](./30-business-intelligence-analytics.md) | Quality | Conversation analytics, agent performance, ROI dashboards |
| [31-intelligent-model-selection](./31-intelligent-model-selection.md) | Scale | Dynamic model routing, response caching, cost reduction |
| [32-multi-agent-collaboration](./32-multi-agent-collaboration.md) | Advanced | Agent message bus, shared memory, consensus, swarms |
| [33-data-pipeline-kb-management](./33-data-pipeline-kb-management.md) | Scale | Document ingestion, KB sync, continuous learning |
| [34-developer-experience](./34-developer-experience.md) | Advanced | VS Code extension, interactive playground, visual debugging |
| [35-compliance-governance](./35-compliance-governance.md) | Enterprise | GDPR/CCPA/HIPAA, data residency, consent management |
| [36-disaster-recovery](./36-disaster-recovery.md) | Enterprise | Multi-region, backup/restore, RPO/RTO guarantees |
| [37-enhancement-summary](./37-enhancement-summary.md) | Summary | Full roadmap summary, sequencing, dependency map |
| [38-visual-architecture-diagrams](./38-visual-architecture-diagrams.md) | Reference | Architecture diagrams across all phases |
| [39-llm-provider-sdk-matrix](./39-llm-provider-sdk-matrix.md) | Reference | Provider SDK comparison and integration notes |
| [40-cost-estimation-and-pricing](./40-cost-estimation-and-pricing.md) | Reference | Cost model, pricing assumptions, estimation tooling |
