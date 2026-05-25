# Enhanced Multi-Agent Collaboration Patterns

> **Status**: Enhancement proposal (post-v6). Advanced agent-to-agent interaction beyond hierarchical calls.
>
> **Depends on**: v4 multi-agent orchestration, v5 chatbot layer, v6 observability.

---

## Purpose

Extend beyond the current hierarchical `agent_call` pattern to support peer-to-peer agent collaboration, shared working memory, consensus mechanisms, and agent swarms. Enable complex multi-agent scenarios where agents negotiate, collaborate, and reach collective decisions.

---

## Business Value

- **Complex Problem Solving**: Multiple agents collaborate on hard problems
- **Collective Intelligence**: Consensus and voting for high-stakes decisions
- **Scalable Processing**: Agent swarms for parallel task distribution
- **Dynamic Adaptation**: Self-organizing agent teams for emerging situations

---

## Core Concepts

### Collaboration Patterns

| Pattern | Description | Use Case |
|---|---|---|
| **Hierarchical** (v4) | Parent calls child, waits for result | Current implementation |
| **Peer-to-Peer** | Agents message each other directly | Collaborative problem solving |
| **Shared Workspace** | Common memory all agents contribute to | Brainstorming, planning |
| **Consensus** | Multiple agents vote on decision | High-stakes approvals |
| **Swarm** | Decentralized task distribution | Load balancing, exploration |
| **Market** | Auction/bidding for task assignment | Resource optimization |

### Agent Roles

| Role | Responsibility | Example |
|---|---|---|
| **Coordinator** | Orchestrates collaboration session | Meeting facilitator |
| **Specialist** | Contributes domain expertise | Legal reviewer, fact checker |
| **Critic** | Identifies flaws and risks | Red team, devil's advocate |
| **Synthesizer** | Combines outputs into coherent whole | Report writer |
| **Executor** | Carries out agreed actions | Task completer |

---

## Epic 1 — Agent Message Bus

### Story 1.1 — Pub/sub messaging infrastructure
**As an** Agent Builder  
**I want** agents to publish and subscribe to topics  
**So that** they can communicate asynchronously.

**Message Bus Features**:
- **Topics**: Hierarchical channels (e.g., `finance.approvals.urgent`)
- **Pub/Sub**: Agents publish messages, subscribed agents receive
- **Persistence**: Messages durably stored, replayable
- **Ordering**: Per-topic ordering guarantees
- **TTL**: Automatic message expiration

**Message Schema**:
```json
{
  "message_id": "uuid",
  "timestamp": "ISO8601",
  "topic": "finance.approvals",
  "sender": "agent://compliance-checker/1.2",
  "correlation_id": "workflow_run_123",
  "payload": {
    "event_type": "approval_needed",
    "data": { ... }
  },
  "priority": "high|normal|low"
}
```

**Acceptance Criteria**:
- Agents can publish to any topic
- Subscription patterns support wildcards
- Message delivery reliable (at-least-once)
- Message history queryable

### Story 1.2 — Request-response patterns over bus
**As an** Agent Builder  
**I want** synchronous request-response over the message bus  
**So that** agents can query each other.

**Pattern**:
1. Agent A publishes request to topic with `reply_to` header
2. Agent B (subscriber) receives request, processes
3. Agent B publishes response to reply topic
4. Agent A receives response, correlation matched

**Timeout Handling**:
- Request timeout configurable
- No response → fallback or error
- Partial responses (multiple agents can reply)

**Acceptance Criteria**:
- Request-response pattern reliable
- Correlation tracking automatic
- Timeout and retry handling
- Response aggregation when multiple responders

### Story 1.3 — Agent presence and discovery
**As an** Agent Builder  
**I want** to discover which agents are available  
**So that** I know who to collaborate with.

**Presence Features**:
- **Heartbeat**: Agents announce availability periodically
- **Capability advertisement**: Agents publish what they can do
- **Discovery query**: "Find agents that can do X"
- **Load indication**: Busy agents can indicate capacity

**Registry Integration**:
- Integrate with existing Tool/MCP Registry
- Dynamic registration/unregistration
- Health check integration

**Acceptance Criteria**:
- Available agents discoverable
- Capability-based search functional
- Stale entries cleaned up automatically

---

## Epic 2 — Shared Working Memory

### Story 2.1 — Collaborative workspace
**As an** Agent Builder  
**I want** multiple agents to read and write to shared memory  
**So that** they can build collective understanding.

**Workspace Concept**:
- **Shared context**: Common memory space for collaborating agents
- **Structured entries**: Typed facts, hypotheses, questions, conclusions
- **Attribution**: Who contributed what, when
- **Versioning**: History of how shared understanding evolved

**Workspace Schema**:
```json
{
  "workspace_id": "uuid",
  "session_id": "workflow_run_123",
  "participants": ["agent://researcher/1.0", "agent://fact-checker/1.2"],
  "entries": [
    {
      "entry_id": "uuid",
      "type": "fact|hypothesis|question|conclusion|action",
      "content": "string",
      "author": "agent://researcher/1.0",
      "timestamp": "ISO8601",
      "confidence": 0.85,
      "evidence": ["citation_1", "citation_2"],
      "status": "proposed|confirmed|rejected"
    }
  ],
  "lock_status": "open|locked_by_agent"
}
```

**Acceptance Criteria**:
- Multiple agents can read/write workspace
- Concurrent access handling (optimistic locking)
- Entry history maintained
- Workspace persists across agent restarts

### Story 2.2 — Structured deliberation
**As an** Agent Builder  
**I want** structured formats for agent deliberation  
**So that** discussions are productive and trackable.

**Deliberation Patterns**:
- **Round-robin**: Each agent contributes in turn
- **Devil's advocate**: Designated critic challenges proposals
- **Delphi method**: Anonymous estimation, iteration to consensus
- **Red team/Blue team**: Adversarial analysis

**Acceptance Criteria**:
- Deliberation templates available
- Progress trackable through structured phases
- Contributions attributed and timestamped
- Convergence detection (consensus reached or not)

### Story 2.3 — Conflict resolution
**As an** Agent Builder  
**I want** agents to resolve conflicting conclusions  
**So that** contradictions are reconciled.

**Resolution Strategies**:
- **Evidence comparison**: Weight of evidence decides
- **Seniority**: More authoritative agent wins
- **Voting**: Majority rules
- **Escalation**: Human review for unresolvable conflicts
- **Confidence threshold**: Higher confidence wins

**Acceptance Criteria**:
- Conflict detection automated
- Resolution strategy configurable
- Unresolved conflicts escalated appropriately
- Resolution process auditable

---

## Epic 3 — Consensus and Voting

### Story 3.1 — Multi-agent voting
**As an** Agent Builder  
**I want** agents to vote on decisions  
**So that** we have democratic or weighted decision making.

**Voting Types**:
- **Simple majority**: >50% wins
- **Supermajority**: >2/3 or >3/4 required
- **Unanimous**: All must agree
- **Weighted**: Different agents have different vote weights
- **Quadratic**: Vote strength costs quadratically

**Voting Process**:
1. Proposal submitted
2. Voting period opens
3. Agents cast votes (with optional reasoning)
4. Voting closes, results tallied
5. Decision enacted or escalated

**Acceptance Criteria**:
- Voting processes complete correctly
- Vote tallies accurate and auditable
- Tie-breaking rules defined
- Abstention handling clear

### Story 3.2 — Confidence-weighted consensus
**As an** Agent Builder  
**I want** consensus based on confidence scores  
**So that** more certain agents have more influence.

**Mechanism**:
- Each agent provides conclusion + confidence (0-1)
- Weighted average or weighted voting
- Minimum confidence threshold for participation
- Consensus declared when weighted agreement > threshold

**Acceptance Criteria**:
- Confidence scores incorporated correctly
- Weighting formulas documented
- Edge cases handled (all low confidence, split high confidence)

### Story 3.3 — Human escalation for contested decisions
**As an** Operator  
**I want** close or contested votes escalated to humans  
**So that** important decisions get human oversight.

**Escalation Triggers**:
- Vote margin < 5%
- All participants have confidence < 0.7
- Explicit objection raised
- High-stakes decision (based on topic classification)

**Escalation Package**:
- Full deliberation transcript
- Vote breakdown with reasoning
- Confidence scores
- Recommended actions

**Acceptance Criteria**:
- Escalation triggers work reliably
- Escalation package comprehensive
- Human decision integrated back into workflow

---

## Epic 4 — Agent Swarms

### Story 4.1 — Swarm formation and task distribution
**As an** Agent Builder  
**I want** to spawn multiple agents for parallel exploration  
**So that** we can solve problems faster through parallelism.

**Swarm Concepts**:
- **Swarm coordinator**: Orchestrates the swarm
- **Worker agents**: Multiple instances of same or different agents
- **Task decomposition**: Break problem into subtasks
- **Result aggregation**: Combine worker outputs

**Distribution Strategies**:
- **Partition**: Divide data/work among agents
- **Replication**: Same task to multiple agents, pick best
- **Exploration**: Different approaches in parallel
- **Hierarchical**: Tree of subtasks

**Acceptance Criteria**:
- Swarms spawn correctly via DSL
- Task distribution balanced
- Worker failures handled gracefully
- Results aggregated appropriately

### Story 4.2 — Self-organizing task allocation
**As an** Agent Builder  
**I want** agents to self-organize task allocation  
**So that** we don't need central coordination.

**Mechanisms** (inspired by multi-agent systems):
- **Contract net**: Agents bid on tasks, best bid wins
- **Market-based**: Auction for task assignment
- **Stigmergy**: Indirect coordination via environment
- **Gossip protocols**: Information spreads organically

**Acceptance Criteria**:
- Self-organization converges on allocation
- No central bottleneck
- Adaptable to dynamic conditions
- Performance comparable to centralized

### Story 4.3 — Swarm intelligence aggregation
**As an** Agent Builder  
**I want** to extract collective intelligence from swarms  
**So that** the whole is greater than the sum of parts.

**Aggregation Methods**:
- **Best result selection**: Pick highest scoring output
- **Ensemble averaging**: Average numerical predictions
- **Majority voting**: Most common classification
- **Stacking**: Train meta-classifier on swarm outputs
- **Debate summarization**: Synthesize arguments

**Acceptance Criteria**:
- Aggregation improves over single agent
- Multiple aggregation strategies available
- Performance metrics validate swarm advantage

---

## Epic 5 — Collaboration Observability

### Story 5.1 — Collaboration session tracking
**As an** Operator  
**I want** to trace multi-agent collaborations  
**So that** I can debug and audit complex interactions.

**Tracking Data**:
- Session participants and their roles
- Message flow (who said what to whom)
- Workspace evolution (snapshots over time)
- Decision points and outcomes
- Timing and latency

**Acceptance Criteria**:
- Complete collaboration history recorded
- Visual timeline of interaction
- Searchable by participant, topic, outcome

### Story 5.2 — Collaboration metrics
**As an** Agent Builder  
**I want** metrics on collaboration effectiveness  
**So that** I can improve agent interactions.

**Metrics**:
- Messages per collaboration (efficiency)
- Time to convergence
- Conflict rate and resolution success
- Contribution distribution (did all participate?)
- Decision quality (retrospective evaluation)

**Acceptance Criteria**:
- Metrics automatically calculated
- Dashboards for collaboration analysis
- Benchmarks for healthy collaboration

### Story 5.3 — Collaboration debugging
**As an** Operator  
**I want** to replay and step through collaborations  
**So that** I can diagnose failures.

**Debugging Features**:
- Replay: Watch collaboration unfold
- Pause at any point
- Inspect workspace state
- Step forward/backward
- Modify and re-run (what-if)

**Acceptance Criteria**:
- Replay accurate to original
- State inspection at any point
- What-if scenarios runnable

---

## Technical Considerations

### Message Bus Implementation
- Apache Kafka or Redis Pub/Sub
- Topic partitioning by correlation/session
- Message durability and replay
- Exactly-once semantics for critical messages

### Shared Workspace Storage
- CRDTs (Conflict-free Replicated Data Types) for concurrent editing
- Operational Transform for text-based collaboration
- Snapshotting for history
- Real-time sync via WebSockets

### Consensus Implementation
- Byzantine fault tolerance not required (agents are trusted)
- Simple voting sufficient for most use cases
- Raft or Paxos for leader election if needed

### Scalability
- Message bus horizontally scalable
- Workspaces sharded by session
- Agent swarms limited by worker pool size

---

## DSL Extensions

```json
{
  "collaboration": {
    "mode": "hierarchical|peer_to_peer|swarm",
    
    "message_bus": {
      "subscribe_to": ["topic1", "topic2.*"],
      "publish_to": ["topic3"]
    },
    
    "workspace": {
      "shared_with": ["agent://collaborator/1.0"],
      "structure": "freeform|deliberation|voting",
      "conflict_resolution": "evidence_based|seniority|voting"
    },
    
    "consensus": {
      "participants": ["agent://expert1/1.0", "agent://expert2/1.1"],
      "voting_strategy": "simple_majority|supermajority|unanimous",
      "weights": { "agent://expert1/1.0": 2 },
      "escalation_on_tie": true
    },
    
    "swarm": {
      "worker_agent": "agent://researcher/1.0",
      "worker_count": 5,
      "task_decomposition": "partition|replicate|explore",
      "aggregation_method": "best|ensemble|voting"
    }
  }
}
```

---

## Definition of Done

- Message bus enables agent pub/sub communication
- Shared workspaces support collaborative memory
- Voting and consensus mechanisms functional
- Agent swarms distribute and aggregate work
- Collaboration fully observable and debuggable
- Metrics measure collaboration effectiveness
- Documentation includes collaboration patterns

---

## Related Documents

- [05-v4-multi-agent-orchestration.md](./05-v4-multi-agent-orchestration.md) — Hierarchical agent calls foundation
- [14-dsl-spec.md](./14-dsl-spec.md) — Agent call specification
- [07-v6-versioning-and-hardening.md](./07-v6-versioning-and-hardening.md) — Observability foundation
