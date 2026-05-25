# Upgrades and Migrations

## Purpose

This document defines the conceptual framework for evolving the platform across versions. It establishes migration principles, compatibility guarantees, and rollback strategies without prescribing specific tooling or commands.

---

## Migration Philosophy

### Progressive Enhancement, Not Breaking Change

Each version builds on the previous without invalidating earlier work:
- v2 adds workflows to v1's agent definitions; v1 agents remain valid
- v3 adds tools/MCPs; v2 workflows without tools continue working
- v4 adds sub-agent calls; v3 agents without sub-agents are unaffected
- v5 adds chatbots; workflow agents from v1–v4 remain unchanged
- v6 adds versioning; all prior agents gain immutability retroactively

**Principle**: Upgrades are additive. Existing capabilities continue working with their existing behavior.

### Immutable Version Guarantees

Once a version ships, its behavior is frozen:
- Agent definitions at version N behave identically on platform version N and N+1
- Workflow runs started on version N complete with version N semantics
- DSL schema version N validates identically before and after upgrade

---

## Database Evolution

### Schema Migration Strategy

The database schema evolves through additive changes:
- New tables for new concepts (e.g., `agent_versions` in v6)
- New columns on existing tables with sensible defaults
- No destructive changes (drops, renames) within a major platform version
- Index additions for performance; no index removals that could break queries

### Data Migration Patterns

| Scenario | Approach | Example |
|---|---|---|
| **New optional field** | Default value, nullable | Adding `retry_policy` to steps (defaults to standard) |
| **New required field** | Backfill with sensible default | Adding `version` to workflow runs (backfill to "initial") |
| **Structural change** | New table, migrate on read | Splitting `agents` into `agents` + `agent_versions` |
| **Data archival** | Separate tablespace, query union | Moving completed runs > 90 days to cold storage |

### Rollback Considerations

Database changes must be reversible:
- Additive changes are inherently rollback-safe
- Data backfills are idempotent (re-running produces same result)
- New tables can be dropped if upgrade fails
- Cross-table migrations use two-phase commit or idempotent patterns

---

## Worker Compatibility

### Versioned Worker Deployment

Workers are deployed with platform version awareness:
- Workers at version N can execute workflows defined at versions ≤ N
- Workers at version N+1 can execute workflows defined at versions ≤ N+1
- Mixed worker pools (N and N+1) during rolling upgrades are supported

### DSL Version Negotiation

When a workflow starts:
1. Platform loads the agent definition
2. Determines the DSL version of that definition
3. Dispatches to workers capable of executing that DSL version
4. Workers interpret the DSL according to its declared version

**Implication**: Old agents continue running on new platforms without code changes.

---

## API Compatibility

### Endpoint Evolution

API endpoints follow additive evolution:
- New endpoints added for new capabilities
- Existing endpoints gain optional parameters
- Response schemas gain optional fields
- No required parameter additions to existing endpoints
- No response field removals or type changes

### Deprecation Policy

When an endpoint or parameter is superseded:
1. **Soft deprecation**: Mark in documentation; new code should use replacement
2. **Warning period**: Responses include deprecation notice for 2+ minor versions
3. **Hard deprecation**: Rejection with clear error message
4. **Removal**: Only after confirmed zero usage (telemetry-verified)

---

## Cross-Version Agent Interoperability

### Calling Agents Across Versions

When agent A (v3) calls agent B (v4 with sub-agents):
- The call is valid if B's interface (input/output schema) is compatible
- B executes with v4 semantics internally
- A receives the result as if B were a black box

### Breaking Interface Changes

If agent B's interface changes incompatibly:
- A new version of B is created (v6 immutability)
- Existing A → B calls continue using B's old version
- Agent Builder updates A to reference B's new version explicitly

---

## Upgrade Procedures

### Pre-Upgrade Checklist

Before any platform upgrade:
- [ ] All workflow runs are in terminal state (completed, failed, cancelled)
- [ ] Database backup verified
- [ ] Worker pool has capacity for rolling restart
- [ ] Monitoring and alerting confirmed functional
- [ ] Rollback procedure reviewed and tested (in non-production)

### Rolling Upgrade Sequence

1. **Database migration**: Run additive schema changes
2. **Worker pool refresh**: Rolling restart to new version
3. **API deployment**: New version accepts traffic
4. **Validation**: Smoke tests on critical paths
5. **Monitoring**: Watch for error rate changes, latency shifts

### Hotfix Exception

For critical security or availability fixes:
- Database changes (if any) are minimal and additive
- Workers restart with zero-downtime deployment
- Previous version remains deployable as instant rollback

---

## Rollback Scenarios

### When Rollback Is Required

- Elevated error rate post-upgrade
- Performance regression exceeding SLA
- Data integrity concern
- Security vulnerability discovered in new version

### Rollback Procedure

1. **Halt new workflows**: Queue but do not start new runs
2. **Drain in-flight**: Wait for active workflows to reach safe checkpoint
3. **Database**: Rollback to pre-upgrade state (requires backup restoration for non-additive changes)
4. **Workers**: Restart previous version
5. **API**: Revert to previous version
6. **Resume**: Process queued workflows

### Rollback Limitations

- Workflows started on new version may not be resumable on old version (if DSL features are used)
- In-flight workflows at time of rollback may need manual intervention
- Database rollbacks lose data written between upgrade and rollback

---

## Migration Utilities (Conceptual)

### Agent Definition Migration

When DSL syntax evolves, utilities assist Agent Builders:
- **Validation**: Check if existing definitions are compatible with new version
- **Suggestion**: Recommend changes to leverage new features
- **Auto-migration**: Optional transformation for mechanical changes (e.g., field renames)

### Workflow Run Migration

Historical workflow runs:
- Remain queryable after upgrade
- Display with version-appropriate semantics
- Exportable for long-term archival

### Credential Migration (v3 → v6)

The v6 vault migration is a specific instance of the general pattern:
1. **Preparation**: Vault configured, credentials interface ready
2. **Scan**: Identify all credentials in database
3. **Migrate**: Copy to vault, verify retrievability
4. **Switch**: Platform reads from vault
5. **Cleanup**: Optionally clear plain-text after confirmation

---

## Version Compatibility Matrix

| Platform Version | DSL Versions Supported | Worker Protocol | Database Schema |
|---|---|---|---|
| v1 | v1 | v1 | v1 |
| v2 | v1, v2 | v1, v2 | v2 (additive) |
| v3 | v1, v2, v3 | v1, v2, v3 | v3 (additive) |
| v4 | v1–v4 | v1–v4 | v4 (additive) |
| v5 | v1–v5 | v1–v5 | v5 (additive) |
| v6 | v1–v6 | v1–v6 | v6 (additive + versions table) |

---

## Long-Term Archival

### Workflow Run Lifecycle

```
Active (0–30 days)     →  Queryable, modifiable (status updates)
Warm (30–90 days)      →  Queryable, read-only
Cold (90+ days)        →  Archived, batch-query only
Deleted (per policy)   →  Purged per retention policy
```

**Note**: Conversation data may have different retention rules than workflow runs.

### Archive Format

- Structured export (preserves queryability)
- Compressed for storage efficiency
- Integrity-verified (checksums)
- Restorable to warm storage if needed

---

## Out of Scope (Implementation Detail)

The following are deferred to implementation:
- Specific migration tools (Alembic, Flyway, etc.)
- Database backup technologies
- CI/CD pipeline specifics
- Canary deployment percentages
- Specific rollback timing (how long to wait before declaring success)

---

## Related Documents

- [01-PRD](./01-PRD.md) — Versioning decisions (AD-16, PD-07)
- [07-v6 — Versioning and Hardening](./07-v6-versioning-and-hardening.md) — Immutable versions, deprecation
- [09-Security Model](./09-security-model.md) — Incident response, secret rotation

---

## What to Read Next

Upgrades connect all platform lifecycle concerns:

- **[07-v6](./07-v6-versioning-and-hardening.md)** — The version system you're upgrading
- **[10-Deployment Concepts](./10-deployment-concepts.md)** — Rolling upgrade procedures
- **[11-Testing Strategy](./11-testing-strategy.md)** — Pre-upgrade validation and canary testing
- **[08-Performance and Scaling](./08-performance-and-scaling.md)** — Archive strategies for old workflow runs

Or start from the beginning:
- [README](./README.md) — Full documentation index
