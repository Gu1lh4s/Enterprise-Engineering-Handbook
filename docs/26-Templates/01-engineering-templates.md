# Engineering Document Templates

> **Category:** Templates
> **Version:** 1.0.0
> **Level:** All Engineers

---

## Table of Contents

1. [Product Requirements Document (PRD)](#1-product-requirements-document-prd)
2. [RFC — Request for Comments](#2-rfc--request-for-comments)
3. [Design Document (Tech Spec)](#3-design-document-tech-spec)
4. [Post-Mortem Template](#4-post-mortem-template)
5. [On-Call Runbook Template](#5-on-call-runbook-template)
6. [New Service README Template](#6-new-service-readme-template)
7. [Engineering Interview Feedback Template](#7-engineering-interview-feedback-template)
8. [Sprint Kickoff Template](#8-sprint-kickoff-template)

---

## 1. Product Requirements Document (PRD)

```markdown
# PRD: [Feature Name]

| Field | Value |
|-------|-------|
| Author | @name |
| Status | Draft / In Review / Approved / Implemented |
| Created | YYYY-MM-DD |
| Last Updated | YYYY-MM-DD |
| Target Release | Q? YYYY |
| Stakeholders | Product, Engineering, Design, [Customer Success if relevant] |

---

## Problem Statement

[What problem are we solving? Who has this problem? How do we know it's real?
Include: user research, support tickets, metrics, quotes from customers.]

**Example:** Clients report difficulty tracking which bookings have been rescheduled more
than once. Support receives 40+ tickets/month about this. Churn correlation analysis shows
clients who use reschedule 3+ times have 2.5x higher churn in the following 90 days.

---

## Goals

**Success metrics** (measurable, with targets):
- [ ] [Metric 1]: from [current value] to [target value] by [date]
- [ ] [Metric 2]: from [current value] to [target value] by [date]

**Non-goals** (explicitly out of scope for this release):
- [Thing we considered but decided not to do, and why]
- [Feature that could logically be included but is deferred]

---

## User Stories

**Primary user:** [Who is the main actor?]

### Must Have (P0)
- As a [user type], I want to [action], so that [outcome]
- As a [user type], I want to [action], so that [outcome]

### Should Have (P1)
- As a [user type], I want to [action], so that [outcome]

### Nice to Have (P2, future consideration)
- As a [user type], I want to [action], so that [outcome]

---

## Functional Requirements

### [Section 1: e.g., "Reschedule History View"]

**FR-1.1:** [Requirement statement]
- Acceptance criteria: [Specific, testable criteria]
- Edge cases: [Non-obvious scenarios to handle]

**FR-1.2:** [Requirement statement]
- Acceptance criteria:
- Edge cases:

### [Section 2]

**FR-2.1:** ...

---

## Non-Functional Requirements

| Requirement | Target | Notes |
|-------------|--------|-------|
| Performance | [e.g., "Page loads < 2s on mobile 4G"] | Lighthouse CI gate |
| Availability | 99.9% (aligned with platform SLA) | |
| Compliance | GDPR data minimization | Don't log PII in new endpoints |
| Accessibility | WCAG 2.1 AA | All new UI components |

---

## UX / Design

[Link to Figma design, or embed screenshots]

[Note key UX decisions and their rationale — helps engineers implement correctly]

---

## Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| 1 | [Question that needs answer before implementation] | @name | Open |
| 2 | [Question] | @name | Resolved: [answer] |

---

## Out of Scope

[Explicit list of things that came up during discussion that we decided not to include.
Important: if it's not said, engineers will guess.]

---

## Appendix

### Research and References
- [Link to user research]
- [Link to competitor analysis]
- [Link to relevant data]
```

---

## 2. RFC — Request for Comments

Use for: significant technical decisions, new architecture patterns, changes affecting multiple teams.

```markdown
# RFC-NNNN: [Title]

| Field | Value |
|-------|-------|
| Author(s) | @name, @name |
| Status | Draft / In Discussion / Accepted / Rejected / Superseded |
| Created | YYYY-MM-DD |
| Discussion Deadline | YYYY-MM-DD |
| Supersedes | RFC-NNNN (if applicable) |
| Superseded by | RFC-NNNN (if applicable) |

---

## Summary

[1-3 sentences. What are you proposing? Not why, just what.]

---

## Motivation

[Why is this change needed? What problem does it solve?
What's the current situation and why is it insufficient?
Include: metrics, incidents, developer experience issues, compliance requirements.]

---

## Detailed Design

[The full technical proposal. Include:
- Architecture diagrams (Mermaid/ASCII/linked image)
- Data model changes
- API contract changes
- Migration path from current to new state
- Security considerations

Be specific — this document should be detailed enough that a different engineer
could implement the proposal from this document alone.]

### Example

\`\`\`typescript
// Concrete code examples are more useful than abstract descriptions
\`\`\`

### Data Model

\`\`\`sql
-- Schema changes
\`\`\`

---

## Drawbacks

[What are the downsides of this proposal?
Be honest — if you can't name any drawbacks, you haven't thought it through.]

---

## Alternatives Considered

### Alternative 1: [Name]

[Describe the alternative]

**Why rejected:** [Reason]

### Alternative 2: [Name]

[Describe the alternative]

**Why rejected:** [Reason]

### Do Nothing

[Why is the status quo not acceptable?]

---

## Unresolved Questions

[Questions you don't have answers to yet. Solicit feedback on these specifically.]

1. [Question]
2. [Question]

---

## Implementation Plan

**Phase 1** (YYYY-MM): [What gets done, who owns it]
**Phase 2** (YYYY-MM): [Next phase]

**Rollback plan:** [How to revert if this goes wrong]

---

## Open Discussion

[For async comments — engineers add their feedback here before the discussion deadline]

### @engineer1 (YYYY-MM-DD)
[Feedback]

### @engineer2 (YYYY-MM-DD)
[Feedback]
```

---

## 3. Design Document (Tech Spec)

Use for: Implementation plan for a single feature or system. Less formal than RFC. Focuses on HOW to build it.

```markdown
# Design: [Feature Name]

| Field | Value |
|-------|-------|
| Author | @name |
| PRD | [Link to PRD] |
| Status | Draft / Reviewed / Approved |
| Estimated Effort | [S / M / L / XL] — [person-weeks] |
| Created | YYYY-MM-DD |

---

## Overview

[2-4 sentences: what are we building and what approach are we taking?]

---

## Architecture

[System diagram showing components, data flows, external dependencies]

\`\`\`
┌──────────────────┐      ┌──────────────────┐
│   React Client   │─────▶│   API Service    │
└──────────────────┘      └────────┬─────────┘
                                   │
                          ┌────────▼─────────┐
                          │   PostgreSQL DB   │
                          └──────────────────┘
\`\`\`

---

## Data Model

\`\`\`sql
-- New tables or columns
CREATE TABLE example (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  -- ...
);

-- Migration notes: zero-downtime? table size? CONCURRENT indexes?
\`\`\`

---

## API Design

\`\`\`
GET  /api/v1/[resource]          → List
POST /api/v1/[resource]          → Create
GET  /api/v1/[resource]/:id      → Get one
PUT  /api/v1/[resource]/:id      → Update
DELETE /api/v1/[resource]/:id    → Delete
\`\`\`

**Request body (POST /api/v1/[resource]):**
\`\`\`json
{
  "field": "value"
}
\`\`\`

**Response:**
\`\`\`json
{
  "data": {
    "id": "uuid",
    "field": "value"
  }
}
\`\`\`

---

## Implementation Plan

### Phase 1: Backend (Week 1)
- [ ] Database migration
- [ ] Repository layer
- [ ] Service / use case
- [ ] API endpoints
- [ ] Unit + integration tests

### Phase 2: Frontend (Week 2)
- [ ] API client / TanStack Query hooks
- [ ] UI components
- [ ] Form validation
- [ ] E2E tests

---

## Testing Plan

| Test type | What's tested | Tool |
|-----------|--------------|------|
| Unit | Service logic | Vitest |
| Integration | API endpoint with DB | Supertest + TestContainers |
| E2E | User flow | Playwright |

---

## Security Considerations

- [ ] Authentication required on all new endpoints
- [ ] Authorization: user can only access their own data
- [ ] Input validated with Zod schema
- [ ] Audit log for sensitive operations

---

## Observability

- [ ] New metrics: [what to track]
- [ ] Logs: [key events to log]
- [ ] Alerts: [conditions to alert on]

---

## Rollout Plan

1. Deploy to staging (Week X)
2. QA and stakeholder review
3. Feature flag: enable for internal users (Week X+1)
4. Gradual rollout: 5% → 20% → 100% users
5. Monitor error rate and latency for 48h after each step
```

---

## 4. Post-Mortem Template

For full post-mortem template with 8 incident playbooks, see `docs/25-Playbooks/01-incident-response-playbooks.md`.

Brief template:

```markdown
# Post-Mortem: [Incident Name / Date]

**Incident ID:** INC-YYYY-NNNN
**Severity:** P0 / P1 / P2
**Duration:** [HH:MM] — [Start UTC] to [End UTC]
**Impact:** [Number of affected users / % of requests / revenue impact]
**Author(s):** @name
**Review date:** YYYY-MM-DD

---

## Timeline

| Time (UTC) | Event |
|------------|-------|
| HH:MM | [What happened] |
| HH:MM | [Alert fired / engineer paged] |
| HH:MM | [Investigation step / finding] |
| HH:MM | [Mitigation applied] |
| HH:MM | [Incident resolved] |

---

## Root Cause

[One paragraph: what was the actual cause?
Example: "A deploy at 14:22 UTC included a migration that added a NOT NULL column without
a default value. The migration succeeded on staging (small dataset) but failed partway through
on production's 8M-row table, leaving the table in a partially-migrated state. The new API
code assumed the column existed, causing 500 errors on all endpoints that queried this table."]

---

## Contributing Factors

[List factors that made this worse or possible — not to blame, to fix]
- [Factor 1]
- [Factor 2]

---

## Impact

- Users affected: [number or %]
- Duration: [time to detect] + [time to resolve]
- Revenue impact: [if calculable]
- Customer communications: [if sent]

---

## What Went Well

[Genuine credit — what worked?]
- [Detection was fast]
- [Team communication was clear]

---

## What Went Poorly

[What slowed us down or made it worse?]
- [Thing 1]
- [Thing 2]

---

## Action Items

| Action | Owner | Due | Status |
|--------|-------|-----|--------|
| [Specific, actionable fix] | @name | YYYY-MM-DD | Open |
| [Process improvement] | @name | YYYY-MM-DD | Open |

---

*Blameless: this post-mortem is about systems and processes, not individuals.*
```

---

## 5. On-Call Runbook Template

```markdown
# Runbook: [Service Name] — [Alert Name]

**Alert:** `[AlertName]`
**Condition:** [e.g., "P99 latency > 1000ms for 5 minutes"]
**Team:** [Owning team]
**Escalation:** [Who to call if you can't resolve in 30 minutes]
**Dashboard:** [Grafana URL]
**Logs:** [CloudWatch / Datadog query link]

---

## Alert Summary

[One sentence: what does this alert mean for users?]

Example: "API response times are severely degraded. Users experience slow page loads and
booking creation may time out."

---

## Triage Steps

### 1. Verify the alert is real

\`\`\`bash
# Check current p99 latency
kubectl exec -n production [pod-name] -- curl -s http://localhost:9090/metrics | grep http_duration_p99

# Check error rate
kubectl get hpa -n production
\`\`\`

### 2. Check recent changes

\`\`\`bash
# Recent deploys
kubectl rollout history deployment/api -n production

# Recent config changes
kubectl describe configmap api-config -n production
\`\`\`

### 3. Check downstream dependencies

\`\`\`bash
# Database
kubectl exec -n production [api-pod] -- node -e "
  const { Pool } = require('pg');
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  pool.query('SELECT NOW()').then(r => console.log('DB OK:', r.rows[0])).catch(e => console.error('DB ERROR:', e.message));
"

# Redis
kubectl exec -n production [api-pod] -- node -e "
  const redis = require('redis').createClient({ url: process.env.REDIS_URL });
  redis.connect().then(() => redis.ping()).then(r => console.log('Redis OK:', r)).catch(e => console.error('Redis ERROR:', e.message));
"
\`\`\`

---

## Decision Tree

```
P99 latency > 1000ms
  │
  ├── Recent deploy in last 2h?
  │     YES → Rollback: kubectl rollout undo deployment/api -n production
  │     NO  → Continue ↓
  │
  ├── Database slow? (check DB dashboard)
  │     YES → Check pg_stat_activity for long-running queries
  │            KILL blocking query: SELECT pg_terminate_backend(pid) WHERE ...
  │     NO  → Continue ↓
  │
  ├── High CPU on pods?
  │     YES → Scale out: kubectl scale deployment/api --replicas=6 -n production
  │     NO  → Continue ↓
  │
  └── External API dependency slow?
        YES → Enable circuit breaker / fallback (see Feature Flags)
        NO  → Escalate to [team] with traces from APM
```

---

## Common Causes and Fixes

### Cause: Database connection pool exhausted

**Symptoms:** Logs show `Error: timeout — remaining connection slots are reserved`

\`\`\`bash
# Check connection count
kubectl exec -n production [db-pod] -- psql -U postgres -c "
  SELECT count(*), wait_event_type, wait_event 
  FROM pg_stat_activity 
  GROUP BY 2,3 
  ORDER BY 1 DESC;
"

# Kill idle connections
kubectl exec -n production [db-pod] -- psql -U postgres -c "
  SELECT pg_terminate_backend(pid)
  FROM pg_stat_activity
  WHERE state = 'idle' AND state_change < NOW() - INTERVAL '5 minutes';
"
\`\`\`

### Cause: Slow query from missing index

**Symptoms:** DB CPU high, specific endpoints slow (check APM for which query)

\`\`\`sql
-- Find slow queries
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Add index CONCURRENTLY (non-blocking)
CREATE INDEX CONCURRENTLY idx_bookings_tenant_status 
ON bookings(tenant_id, status) 
WHERE status != 'completed';
\`\`\`

---

## Escalation

| Condition | Contact | Method |
|-----------|---------|--------|
| Can't resolve in 30 min | [Team lead] | PagerDuty override |
| Database data corruption | [DBA] | Phone |
| Security incident suspected | [Security team] | Security Slack channel |

---

## Post-Incident

After resolving:
- [ ] Update incident ticket with timeline and root cause
- [ ] Schedule post-mortem if P0/P1
- [ ] Update this runbook if you found a better fix
```

---

## 6. New Service README Template

```markdown
# [Service Name]

[One sentence: what does this service do?]

## Overview

[2-3 paragraph description:
1. What problem it solves and who uses it
2. How it fits in the broader architecture (what calls it, what it calls)
3. Key technical decisions / constraints]

## Architecture

\`\`\`
[ASCII diagram or link to architecture diagram]
\`\`\`

**Key dependencies:**
- PostgreSQL (primary data store)
- Redis (caching, job queues)
- [other-service] (called for [reason])

## Getting Started

### Prerequisites

- Node.js 20+
- Docker Desktop
- Access to development secrets (see [internal link])

### Local Development

\`\`\`bash
cp .env.example .env
# Edit .env with your local values

docker compose up -d        # Start PostgreSQL and Redis
npm install
npm run db:migrate
npm run dev
\`\`\`

Service runs at: `http://localhost:3000`

## API

OpenAPI spec: `openapi.yaml` or [Swagger UI link]

Key endpoints:
- `GET /health` — Health check
- `GET /api/v1/[resource]` — [Description]
- `POST /api/v1/[resource]` — [Description]

## Configuration

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `REDIS_URL` | Yes | Redis connection string |
| `JWT_SECRET` | Yes | Secret for verifying JWTs |
| `PORT` | No (default: 3000) | HTTP port |

## Testing

\`\`\`bash
npm test           # Unit tests
npm run test:int   # Integration tests (requires Docker)
npm run test:e2e   # E2E tests (requires full stack)
\`\`\`

## Deployment

Deploys via CI/CD on merge to `main`. See `docs/deployment.md`.

**Kubernetes namespace:** `production`
**Deployment:** `kubectl get deployment [service-name] -n production`

## Runbook

See `docs/runbook.md` for operational guide and common issues.

## Contributing

See [CONTRIBUTING.md] and the [Engineering Handbook].

## Owners

Team: [Team Name]
On-call: [PagerDuty schedule link]
Slack: `#[service-channel]`
```

---

## 7. Engineering Interview Feedback Template

```markdown
# Interview Feedback: [Candidate Name]

**Role:** [Job title / Level]
**Date:** YYYY-MM-DD
**Interviewer:** @name
**Interview type:** [Technical / System Design / Behavioral / Values]
**Duration:** 60 minutes

---

## Overall Recommendation

[ ] Strong Hire — Would be in the top 10% of candidates at this level
[ ] Hire — Meets the bar for this level
[ ] No Hire — Does not meet the bar for this level
[ ] Strong No Hire — Significant concerns

**Confidence level:** Low / Medium / High

---

## Technical Assessment

### Problem Solving

[How did they approach the problem? Did they ask clarifying questions?
Did they think through edge cases?]

Rating: 1-4 (1=well below bar, 2=below bar, 3=meets bar, 4=above bar)

### Code Quality

[Was the code readable? Did they handle errors? Appropriate abstractions?]

Rating: 1-4

### System Design (if applicable)

[Did they understand scale? Identify bottlenecks? Make reasonable tradeoffs?]

Rating: 1-4

### Technical Communication

[Could they explain their thinking clearly? Did they ask good questions?]

Rating: 1-4

---

## Behavioral / Values (for behavioral interviews)

### Question 1: [Topic, e.g., "Conflict resolution"]

**Question asked:** [Exact question]

**Response summary:** [STAR method: Situation, Task, Action, Result]

**Assessment:** [What this tells us about the candidate]

### Question 2: [Topic]

---

## Specific Examples

**Strength:** [Specific thing they did well, with evidence]

**Concern:** [Specific concern, with evidence — be concrete]

---

## Notes for Hiring Committee

[Anything not captured above that the committee should know.
Context: how did this compare to others at this level?
Any doubts that more information could resolve?]
```

---

## 8. Sprint Kickoff Template

```markdown
# Sprint [N] Kickoff — [Team Name]

**Sprint dates:** YYYY-MM-DD → YYYY-MM-DD (2 weeks)
**Sprint goal:** [One sentence: what will we accomplish and why does it matter?]

---

## Capacity

| Engineer | Capacity | Notes |
|----------|----------|-------|
| @name1 | 10 days | |
| @name2 | 8 days | OOO Dec 5 |
| @name3 | 10 days | |
| **Total** | **28 days** | |

---

## Sprint Backlog

| Priority | Ticket | Points | Assignee | Notes |
|----------|--------|--------|----------|-------|
| P0 | [TICKET-123] Description | 3 | @name | Blocks deploy |
| P0 | [TICKET-456] Description | 5 | @name | |
| P1 | [TICKET-789] Description | 2 | @name | |
| P2 | [TICKET-012] Description | 3 | @name | If time permits |
| **Total** | | **13** | | |

---

## Carryover from Last Sprint

| Ticket | Why carried | Plan |
|--------|-------------|------|
| [TICKET] | [Reason] | [Complete by Day X / split / descope] |

---

## Dependencies and Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| [Risk] | High/Med/Low | High/Med/Low | [Mitigation] |

---

## Definition of Done

For this sprint, a ticket is DONE when:
- [ ] Code merged to main with tests passing
- [ ] Feature tested in staging
- [ ] Documentation updated (if user-facing)
- [ ] Ticket closed in project tracker

---

## Sprint Goal Acceptance Criteria

We will declare this sprint successful if:
- [ ] [Measurable criteria 1]
- [ ] [Measurable criteria 2]
```
