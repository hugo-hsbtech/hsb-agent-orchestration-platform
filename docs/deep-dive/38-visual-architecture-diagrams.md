# Visual Architecture Diagrams

> **Purpose**: Visual reference for system components, data flows, and deployment topology. Complements the text specifications in `01-PRD.md` and phase documents.

---

## System Component Overview

```mermaid
flowchart TB
    subgraph External["External Systems"]
        User["End User<br/>Chat UI"]
        LLM["LLM Providers<br/>Claude / OpenAI / Gemini / Grok"]
        MCP["MCP Servers<br/>Jira / Linear / Figma / Custom"]
        KB["Knowledge Base<br/>Qdrant Vector DB"]
    end

    subgraph Platform["Agent Platform"]
        subgraph APILayer["API Layer"]
            ChatAPI["/chat<br/>Streaming Endpoint"]
            InvokeAPI["/invoke<br/>Workflow Endpoint"]
            AdminAPI["/admin<br/>Provisioning API"]
        end

        subgraph CoreServices["Core Services"]
            Router["Routing Chatbot"]
            Specialist["Specialist Chatbots"]
            AgentCore["Agent Core<br/>(Provider Abstraction)"]
            Validation["Validation Service"]
        end

        subgraph Registries["Registries"]
            ToolReg["Tool Registry"]
            MCPReg["MCP Registry"]
        end

        subgraph Execution["Execution Layer"]
            Temporal["Temporal Cluster<br/>(Workflow Orchestration)"]
            Workers["Temporal Workers<br/>(Generic, Scalable)"]
        end

        subgraph State["State & Persistence"]
            AppDB["Application Database<br/>(Postgres)"]
            StateMachine[("State Machine<br/>workflow_runs<br/>step_runs<br/>conversations")]
        end

        subgraph Observability["Observability"]
            Prometheus["Prometheus Metrics"]
            Tracing["OpenTelemetry Tracing"]
            Logs["Structured Logs"]
        end
    end

    User --> ChatAPI
    ChatAPI --> Router
    Router --> Specialist
    Specialist --> AgentCore
    AgentCore --> LLM
    Specialist --> ToolReg
    Specialist --> MCPReg
    MCPReg --> MCP
    ToolReg --> KB

    AdminAPI --> Validation
    Validation --> ToolReg
    Validation --> MCPReg

    InvokeAPI --> Temporal
    Temporal --> Workers
    Workers --> AgentCore
    Workers --> ToolReg
    Workers --> MCPReg

    Temporal <--> StateMachine
    Workers --> StateMachine
    ChatAPI --> StateMachine

    CoreServices --> Prometheus
    CoreServices --> Tracing
    CoreServices --> Logs
    Execution --> Prometheus
    Execution --> Tracing
```

---

## DSL to Execution Flow

```mermaid
sequenceDiagram
    autonumber
    participant AB as Agent Builder
    participant API as Provisioning API
    participant VS as Validation Service
    participant TR as Tool Registry
    participant DB as Application DB
    participant TW as Temporal Worker
    participant AC as Agent Core
    participant LLM as LLM Provider

    AB->>API: POST /agents<br/>{DSL Document}
    API->>VS: validate(dsl)
    VS->>TR: Check tool references
    TR-->>VS: Tool schemas
    VS->>VS: Validate data flow<br/>type compatibility
    VS-->>API: ValidationResult<br/>{ok} or {errors: []}

    alt Validation Failed
        API-->>AB: 400 Bad Request<br/>{errors: [...]}
    else Validation Passed
        API->>DB: INSERT agent_versions<br/>version = v{N+1}, state = draft
        DB-->>API: version_id
        API-->>AB: 201 Created<br/>{version_id, state: draft}
    end

    Note over AB,LLM: Runtime Execution
    AB->>API: POST /invoke?version=v{N+1}<br/>{input}
    API->>DB: Fetch DSL for version
    DB-->>API: DSL Document
    API->>Temporal: StartWorkflow<br/>(dsl, input, run_id)
    Temporal->>TW: Dispatch Workflow
    TW->>DB: CREATE workflow_runs<br/>status: running

    loop For Each Step
        TW->>TW: Load step config from DSL
        TW->>AC: Execute(provider, prompt, tools)
        AC->>LLM: Native API Call
        LLM-->>AC: Response + token usage
        AC-->>TW: StepOutput{result, tokens}
        TW->>DB: UPDATE workflow_step_runs<br/>status, output, tokens
    end

    TW->>DB: UPDATE workflow_runs<br/>status: completed
    TW-->>Temporal: Workflow Complete
```

---

## Chatbot Routing Flow

```mermaid
sequenceDiagram
    autonumber
    participant User as End User
    participant UI as Chat UI
    participant ChatAPI as Chat API
    participant ConvDB as Conversation Store
    participant Router as Routing Chatbot
    participant Specialist as Specialist Chatbot
    participant Backend as Backend Agent
    participant Temporal as Temporal

    User->>UI: "I need a refund"
    UI->>ChatAPI: POST /chat/stream<br/>{conversation_id, message}
    ChatAPI->>ConvDB: Load conversation history
    ConvDB-->>ChatAPI: Turns [n-5 .. n-1]

    ChatAPI->>Router: classify(message + history)
    Router->>Router: LLM inference<br/>intent = "billing.refund"
    Router->>Router: Match specialist<br/>"billing" → billing_specialist_v3
    Router-->>ChatAPI: {specialist: billing, confidence: 0.94}

    ChatAPI->>Specialist: handle(message, context)
    Specialist->>Specialist: Check KB<br/>refund policy lookup
    Specialist->>Specialist: Build response stream
    Specialist-->>ChatAPI: Streaming chunks
    ChatAPI-->>UI: SSE chunks
    UI-->>User: "I can help with refunds..."

    alt Needs Background Work
        Specialist->>ChatAPI: Delegate to backend
        ChatAPI->>Backend: POST /invoke (fire_and_forget)
        Backend->>Temporal: Start workflow
        Temporal-->>ChatAPI: {workflow_run_id}
        ChatAPI->>ConvDB: Link pending_task<br/>{run_id, recall_context}
        ChatAPI-->>Specialist: {accepted, task_id}
        Specialist-->>ChatAPI: "...I'll check your<br/>eligibility now."
        ChatAPI-->>UI: [continue stream]
    end

    Note over Temporal,ChatAPI: Later: Workflow Completes
    Temporal->>ChatAPI: Callback / push notification
    ChatAPI->>ConvDB: Mark task complete<br/>Store result
    ChatAPI->>User: [Push notification]<br/>"Your refund eligibility: ..."
```

---

## Namespace-Based Multi-Tenancy

```mermaid
flowchart TB
    subgraph TenantA["Namespace: acme-corp"]
        A_Agents["Agent Definitions<br/>v1, v2, v3..."]
        A_Runs["Workflow Runs"]
        A_Conv["Conversations"]
        A_Creds["Credentials<br/>Stripe API Key<br/>Zendesk Token"]
        A_Quota["Daily Quota<br/>$500/day"]
    end

    subgraph TenantB["Namespace: globex-inc"]
        B_Agents["Agent Definitions"]
        B_Runs["Workflow Runs"]
        B_Conv["Conversations"]
        B_Creds["Credentials<br/>Jira API Key"]
        B_Quota["Daily Quota<br/>$200/day"]
    end

    subgraph TenantC["Namespace: default"]
        C_Agents["Agent Definitions"]
        C_Runs["Workflow Runs"]
    end

    subgraph PlatformShared["Platform Shared Resources"]
        ToolReg["Tool Registry<br/>(read-only to tenants)"]
        MCPReg["MCP Registry<br/>(read-only to tenants)"]
        Providers["Provider Configs<br/>(endpoints, rate limits)"]
        TemporalCluster["Temporal Cluster<br/>(namespaced workflow isolation)"]
    end

    TenantA -.->|Uses| ToolReg
    TenantA -.->|Uses| MCPReg
    TenantA -.->|Uses| Providers
    TenantA -.->|Runs on| TemporalCluster

    TenantB -.->|Uses| ToolReg
    TenantB -.->|Uses| MCPReg
    TenantB -.->|Uses| Providers
    TenantB -.->|Runs on| TemporalCluster

    TenantC -.->|Uses| ToolReg
    TenantC -.->|Uses| MCPReg

    Note over TenantA,TenantC: Complete isolation:<br/>• No cross-namespace queries<br/>• Separate credential scopes<br/>• Separate quota tracking
```

---

## State Machine: Workflow Run Lifecycle

```mermaid
stateDiagram-v2
    [*] --> pending: API receives invoke request
    pending --> running: Temporal workflow starts

    running --> running: Step completes<br/>next step dispatched
    running --> paused: Operator pause signal
    running --> waiting_input: Need user clarification
    running --> pending_task: Delegated to background agent

    paused --> running: Operator resume signal
    paused --> cancelled: Operator cancel

    waiting_input --> running: Input received
    waiting_input --> cancelled: Timeout / Cancel

    pending_task --> running: Background task completes
    pending_task --> failed: Background task fails

    running --> succeeded: Final step completes
    running --> failed: Uncaught error / max retries
    running --> cancelled: Budget exceeded<br/>Operator cancel

    succeeded --> [*]
    failed --> [*]: Error recorded
    cancelled --> [*]: Cleanup complete

    running --> canary_rollback: Auto-rollback trigger
    canary_rollback --> running: Return to previous version
```

---

## Version Promotion Lifecycle

```mermaid
stateDiagram-v2
    [*] --> draft: Builder creates new version

    draft --> draft: Testing with version pin<br/>?version=draft
    draft --> staging: Promote to staging

    staging --> staging: Shadow traffic<br/>Synthetic tests
    staging --> draft: Demote (issues found)
    staging --> canary: Promote to canary

    canary --> canary: 10% traffic<br/>Metrics observed
    canary --> staging: Auto-rollback<br/>Error rate > 5%
    canary --> production: Auto-promote<br/>or Manual

    production --> production: 100% traffic<br/>latest pointer
    production --> deprecated: Mark deprecated

    deprecated --> archived: All runs complete
    deprecated --> production: Emergency rollback

    archived --> [*]: Historical reference only
```

---

## Deployment Topology (Kubernetes)

```mermaid
flowchart TB
    subgraph K8s["Kubernetes Cluster"]
        subgraph Ingress["Ingress Layer"]
            LB["Load Balancer<br/>SSL termination"]
        end

        subgraph APITier["API Tier (Deployment)"]
            API1["API Pod 1"]
            API2["API Pod 2"]
            API3["API Pod N"]
        end

        subgraph WorkerTier["Worker Tier (Deployment)"]
            W1["Temporal Worker 1"]
            W2["Temporal Worker 2"]
            W3["Temporal Worker N"]
        end

        subgraph ChatTier["Chat Tier (Deployment)"]
            C1["Chat API Pod 1<br/>WebSocket/SSE"]
            C2["Chat API Pod 2"]
        end

        subgraph Monitoring["Observability Stack"]
            Prom["Prometheus Server"]
            Grafana["Grafana Dashboards"]
            Collector["OTel Collector"]
        end

        subgraph Internal["Internal Services"]
            Validation["Validation Service"]
            TemporalSvc["Temporal Frontend"]
            TemporalMatch["Temporal Matching"]
            TemporalHistory["Temporal History"]
            TemporalWorkerMgr["Temporal Worker Service"]
        end
    end

    subgraph External["External Services"]
        Postgres["Postgres<br/>Application DB"]
        TemporalDB["Postgres/Cassandra<br/>Temporal Persistence"]
        Qdrant["Qdrant<br/>Vector DB"]
        Redis["Redis<br/>(optional: session/cache)"]
    end

    LB --> API1
    LB --> API2
    LB --> C1
    LB --> C2

    API1 --> TemporalSvc
    API2 --> TemporalSvc
    C1 --> TemporalSvc

    W1 --> TemporalWorkerMgr
    W2 --> TemporalWorkerMgr
    W3 --> TemporalWorkerMgr

    API1 --> Validation
    API2 --> Validation

    API1 --> Postgres
    W1 --> Postgres
    C1 --> Postgres

    TemporalSvc --> TemporalDB
    TemporalMatch --> TemporalDB
    TemporalHistory --> TemporalDB
    TemporalWorkerMgr --> TemporalDB

    W1 --> Qdrant
    C1 --> Qdrant

    APITier --> Prom
    WorkerTier --> Prom
    ChatTier --> Prom
    APITier --> Collector
    WorkerTier --> Collector
    ChatTier --> Collector

    Prom --> Grafana
```

---

## Circuit Breaker State Flow

```mermaid
stateDiagram-v2
    [*] --> closed: Initial state

    closed --> closed: Success
    closed --> open: Failure count >= threshold

    open --> open: Request rejected<br/>Fast fail
    open --> half_open: Timeout expires<br/>(e.g., 60s)

    half_open --> closed: Test success
    half_open --> open: Test failure

    closed --> [*]: Manual reset
    open --> [*]: Manual reset
```

---

## Document Index

| Diagram | Use Case | Reference |
|---------|----------|-----------|
| System Component Overview | Architecture reviews, onboarding | `01-PRD.md` §Architectural Components |
| DSL to Execution Flow | Understanding the provisioning → runtime pipeline | `02-v1` through `05-v4` |
| Chatbot Routing Flow | Chatbot layer design, UX flow | `06-v5-chatbot-layer.md` |
| Namespace Multi-Tenancy | Security reviews, tenant isolation design | `16-multi-tenancy.md` |
| Workflow Run Lifecycle | State machine implementation | `03-v2-workflow-orchestration.md` |
| Version Promotion | CI/CD integration, release process | `20-agent-version-promotion.md` |
| Deployment Topology | Infrastructure planning, SRE runbooks | `10-deployment-concepts.md` |
| Circuit Breaker | Chatbot degradation handling | `22-chatbot-degradation.md` |

---

> **Document History**
> - Created: Post-review enhancement
> - Purpose: Visual reference companion to text specifications
