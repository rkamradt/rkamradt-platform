# Task: [Task Name]

**Created**: YYYY-MM-DD
**Last Updated**: YYYY-MM-DD
**Status**: PLANNING | IN PROGRESS | COMPLETE | BLOCKED | CANCELLED

## Overview

Brief description of what this task aims to accomplish and why it's important.

## Repos In Scope

List all repositories that will be modified or are relevant to this task:

- [ ] `~/github/manufacturing-service` - [Brief description of changes needed]
- [ ] `~/github/vehicleevent-api` - [Brief description of changes needed]
- [ ] `~/github/inventory-service` - [Brief description of changes needed]

## Context & Background

### Problem Statement

What problem are we solving? What's the current state that needs to change?

### Requirements

- Functional requirement 1
- Functional requirement 2
- Non-functional requirement 1

### Constraints

- Technical constraints
- Time constraints
- Resource constraints

### Dependencies

- External dependencies (libraries, services, APIs)
- Internal dependencies (other tasks, services, features)

## Key Decisions

Document important architectural and implementation decisions:

### Decision 1: [Title]

**Context**: Why did this decision need to be made?

**Options Considered**:
1. Option A - pros/cons
2. Option B - pros/cons
3. Option C - pros/cons

**Decision**: Which option was chosen and why

**Consequences**: What are the implications of this decision?

---

### Decision 2: [Title]

...

## Architecture & Design

### Event Flows

Describe how events flow between services for this feature:

```
Service A                    Service B                    Service C
    │                            │                            │
    ├─ event.created ────────────▶│                            │
    │                            ├─ process event             │
    │                            ├─ event.processed ─────────▶│
    │                            │                            ├─ handle event
```

### API Changes

Document any new or modified API endpoints:

**New Endpoints**:
- `POST /api/resource` - Create new resource
- `GET /api/resource/{id}` - Retrieve resource

**Modified Endpoints**:
- `PUT /api/resource/{id}` - Updated to support new fields

### Database Changes

**New Tables**:
```sql
CREATE TABLE new_table (
    id UUID PRIMARY KEY,
    ...
);
```

**Schema Migrations**:
- Migration V5__add_new_column.sql

### Kafka Topics

**New Topics**:
| Topic Name | Producer | Consumer(s) | Schema |
|------------|----------|-------------|--------|
| `domain.entity.event` | service-a | service-b, service-c | [Link](../architecture/event-schemas.md) |

**Modified Topics**: _(list any schema changes)_

## Implementation Phases

Break down the work into concrete, actionable phases:

### Phase 1: [Phase Name]

**Goal**: What does this phase accomplish?

**Tasks**:
- [ ] Task 1 description
  - Repo: `manufacturing-service`
  - Files: `src/main/java/...`
- [ ] Task 2 description
  - Repo: `vehicleevent-api`
  - Files: `src/main/java/...`

**Testing**:
- [ ] Unit tests for new functionality
- [ ] Integration tests
- [ ] Manual testing steps

**Deliverables**:
- Working feature X
- Updated documentation

---

### Phase 2: [Phase Name]

**Goal**: What does this phase accomplish?

**Tasks**:
- [ ] Task 1 description
- [ ] Task 2 description

**Testing**:
- [ ] Test 1
- [ ] Test 2

**Deliverables**:
- Item 1
- Item 2

---

### Phase 3: [Phase Name]

...

## Testing Strategy

### Unit Tests

- Coverage goals
- Key test cases to write

### Integration Tests

- Service integration scenarios
- Database integration scenarios
- Kafka integration scenarios

### End-to-End Tests

- User workflows to test
- Expected outcomes

### Performance Tests

- Load testing requirements
- Performance benchmarks
- Scalability considerations

## Rollout Plan

### Development Environment

1. Deploy to dev
2. Smoke test
3. Integration testing

### Staging Environment

1. Deploy to staging
2. Full regression testing
3. Performance testing
4. Stakeholder demo/approval

### Production Environment

1. Database migration (if needed)
2. Deploy service A
3. Monitor and verify
4. Deploy service B
5. Monitor and verify
6. Full system validation

### Rollback Plan

Steps to rollback if issues are discovered:
1. Revert deployment
2. Restore database (if needed)
3. Clear affected Kafka topics (if needed)

## Risks & Mitigation

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Database migration fails | Low | High | Test migration on staging, have rollback script ready |
| Service downtime during deployment | Medium | Medium | Use blue-green deployment, rolling updates |
| Breaking change to event schema | Low | High | Use schema versioning, maintain backward compatibility |

## Success Criteria

How do we know this task is complete and successful?

- [ ] All functional requirements met
- [ ] All tests passing (unit, integration, e2e)
- [ ] Documentation updated
- [ ] Code reviewed and approved
- [ ] Deployed to production
- [ ] Monitoring and alerts configured
- [ ] No P0/P1 bugs in production for 1 week

## Notes

### YYYY-MM-DD

- Note about progress, decision, blocker, etc.

### YYYY-MM-DD

- Another note

## References

- Link to related tasks
- Link to design documents
- Link to Slack/email discussions
- Link to relevant architecture docs
