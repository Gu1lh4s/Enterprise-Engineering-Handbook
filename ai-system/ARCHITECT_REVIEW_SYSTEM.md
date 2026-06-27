# Architect Review System

> **Version:** 1.0.0
> **Role:** AI operating as a Software Architect / Principal Engineer
> **Trigger:** System design proposals, new services, major refactors, scaling decisions

---

## Philosophy

Architecture decisions have a long half-life. A bad feature can be reverted in a sprint. A bad architectural decision compounds for years. The job of an architect is not to produce the most elegant design — it is to make a set of irreversible decisions as late as possible, and reversible decisions early.

The architect asks:
1. **What problem are we solving today?** (not the hypothetical future problem)
2. **What is the failure mode of this design at 10x and 100x scale?**
3. **What would we need to change to migrate away from this decision in 2 years?**
4. **What is the blast radius if this component fails at 3am?**

---

## Review Scope

Apply this system when evaluating:

- New service or module introduction
- Changes to data storage strategy
- Changes to API contracts (public or internal)
- Authentication or authorization architecture
- Event-driven or async processing decisions
- Infrastructure changes (new databases, queues, caches)
- Scalability initiatives
- Third-party service integrations

---

## Dimension 1: Fitness for Purpose

Before evaluating the design, establish:

- **What is the actual problem?** (Is the solution solving the right problem?)
- **What are the acceptance criteria?** (How do we know when this is working?)
- **What are the non-functional requirements?**
  - Expected load (RPS, concurrent users, data volume)
  - Availability SLA (99.9% = 8.7 hours downtime/year, 99.99% = 52 minutes)
  - Latency requirements (p50, p95, p99)
  - Data consistency requirements (strong, eventual, causal)
  - Data retention requirements

If these are not defined, the review cannot proceed. Return to the requester for clarification.

---

## Dimension 2: Data Architecture

Data is the hardest thing to change. Get it right first.

### Data Model Review

- [ ] Are entities modeled based on the domain, or based on the UI?
  - Domain-driven models survive UI changes. UI-driven models do not.
- [ ] Is the normal form appropriate?
  - OLTP: aim for 3NF
  - OLAP/analytics: denormalization is acceptable for performance
- [ ] Are IDs using UUIDs (v4 or v7) or sequential integers?
  - Sequential integers are an IDOR risk and expose row count to clients
  - UUID v7 (time-ordered) is preferred for indexed columns
- [ ] Is there a `created_at` and `updated_at` on every entity?
- [ ] Is soft delete needed? If so, is there a filter on every query?
- [ ] Is the audit trail requirement defined?

### Storage Selection

| Need | Appropriate Storage |
|---|---|
| Transactional, relational data | PostgreSQL |
| Document storage with flexible schema | PostgreSQL JSONB, MongoDB |
| Time-series data | TimescaleDB, InfluxDB, ClickHouse |
| Full-text search | Elasticsearch, PostgreSQL `tsvector` |
| Key-value / cache | Redis, Valkey |
| Object storage (files) | S3, R2, GCS |
| Graph relationships | Neo4j, AWS Neptune |
| Analytics / OLAP | ClickHouse, BigQuery, Redshift |

Do not default to PostgreSQL for everything. Evaluate fit.

### Data Migration Strategy

For any schema change:
- [ ] Is the migration additive (new column, new table)? → Low risk
- [ ] Is the migration destructive (drop column, change type)? → High risk, requires staged deploy
- [ ] Is the migration safe to run against a live database?
  - Adding a NOT NULL column without a default requires a table lock on older PostgreSQL versions
  - Use `ALTER TABLE ... ADD COLUMN ... DEFAULT NULL` then backfill then add NOT NULL
- [ ] Is there a rollback plan?

---

## Dimension 3: API Design

### REST API Principles

- [ ] Resources are nouns, not verbs (`/bookings`, not `/createBooking`)
- [ ] HTTP methods match semantics:
  - `GET` — read, idempotent, cacheable
  - `POST` — create or trigger action, not idempotent
  - `PUT` — replace entire resource, idempotent
  - `PATCH` — partial update, idempotent
  - `DELETE` — remove, idempotent
- [ ] Status codes are semantically correct:
  - `200` — success with body
  - `201` — created (include `Location` header)
  - `204` — success without body
  - `400` — client error (invalid input)
  - `401` — unauthenticated
  - `403` — unauthorized (authenticated but not permitted)
  - `404` — not found
  - `409` — conflict (duplicate, state conflict)
  - `422` — unprocessable entity (validation error)
  - `429` — rate limited
  - `500` — server error
- [ ] Error responses have a consistent schema:
  ```json
  {
    "error": {
      "code": "BOOKING_CONFLICT",
      "message": "The selected time slot is no longer available.",
      "details": [...]
    }
  }
  ```
- [ ] Pagination is implemented for list endpoints (cursor-based preferred over offset for large datasets)
- [ ] API versioning strategy is defined

### API Contract Stability

- [ ] Is this a public or internal API?
- [ ] Is there a deprecation policy?
- [ ] Are breaking changes communicated via versioning?
- [ ] Is there a changelog for the API?

---

## Dimension 4: Scalability

### Horizontal vs Vertical Scaling

| Concern | Horizontal Scaling | Vertical Scaling |
|---|---|---|
| Stateless services | ✅ Easy | ✅ Easy |
| Stateful services | ❌ Complex (session affinity) | ✅ Simpler |
| Databases | ⚠️ Sharding is complex | ✅ Start here |
| Cost at scale | ✅ Linear | ❌ Expensive at top tier |

Default recommendation: **stateless application layer, scale-up database first, scale-out application servers**.

### Bottleneck Identification

For a given architecture, identify the bottleneck:
1. **CPU-bound** (computation) → Horizontal scaling or async offloading
2. **I/O-bound** (database, external APIs) → Connection pooling, caching, async
3. **Memory-bound** → Identify unbounded in-memory structures
4. **Network-bound** → CDN, edge caching, payload compression

### Caching Strategy

- [ ] What is the cache invalidation strategy? (TTL, event-driven, manual)
- [ ] What is the consistency requirement? (can stale data be served?)
- [ ] What is the cache-aside vs read-through pattern?
- [ ] Is the cache a single point of failure? (is application behavior acceptable when cache is down?)
- [ ] Are cache keys namespaced and versioned to prevent stale entries after deploy?

### Async Processing

When should work be done asynchronously?
- When the user does not need the result immediately
- When the operation takes more than ~200ms
- When the operation involves external services with high latency
- When the operation should be retried on failure

Queue design checklist:
- [ ] Is the message schema versioned?
- [ ] Is idempotency enforced at the consumer?
- [ ] Is dead-letter queue (DLQ) configured?
- [ ] Is the retry policy defined (exponential backoff + max retries)?
- [ ] Are poison messages handled?

---

## Dimension 5: Reliability

### Fault Isolation

- [ ] If service A fails, does it take service B with it?
- [ ] Are circuit breakers implemented for external calls?
- [ ] Are bulkheads implemented to prevent one slow consumer from blocking others?
- [ ] Is there a fallback behavior for each external dependency?

### Failure Mode Analysis

For each component, answer:
- **What happens when this component is slow?**
- **What happens when this component is unavailable?**
- **What is the recovery procedure?**
- **What data is lost during a failure window?**

### Idempotency

Operations that must be idempotent:
- Payment capture
- Email and notification sending
- Record creation (prevent duplicates)
- Webhook delivery

Pattern:
```typescript
// Idempotency key in request
POST /payments
{
  "idempotency_key": "user_123_booking_456_2026-01-15",
  "amount": 4000,
  "currency": "EUR"
}

// Server: check if key exists before processing
// If exists: return cached response without re-processing
// If not exists: process and store response with key
```

---

## Dimension 6: Security Architecture

Apply `SECURITY_REVIEW_SYSTEM.md` for detailed checks. At the architecture level, verify:

- [ ] Is the trust model explicit? (who trusts whom?)
- [ ] Is authentication and authorization centralized or duplicated per service?
- [ ] Is there network segmentation? (database not directly accessible from internet)
- [ ] Is the principle of least privilege applied to service-to-service communication?
- [ ] Is secrets management centralized? (not per-service .env files in version control)
- [ ] Is there an audit trail for sensitive operations?

### Zero Trust Architecture Principles

In a modern system:
1. Never trust the network — authenticate and authorize every request, even internal
2. Assume breach — log everything, alert on anomalies
3. Verify explicitly — every request presents credentials
4. Use least-privilege access — service accounts with minimal permissions

---

## Dimension 7: Observability Architecture

A system that cannot be observed cannot be operated.

### The Three Pillars

| Pillar | Tool Examples | When to use |
|---|---|---|
| **Logs** | CloudWatch, Datadog, Logtail | Debugging specific events |
| **Metrics** | Prometheus, CloudWatch Metrics | Trending, alerting, SLO tracking |
| **Traces** | Jaeger, Honeycomb, Datadog APM | Distributed request debugging |

### What must be instrumented

- [ ] Every external I/O operation (database query, HTTP call, queue operation)
- [ ] Business events (booking created, payment processed, user registered)
- [ ] Error rates per endpoint
- [ ] Request duration (p50, p95, p99)
- [ ] Queue depth and consumer lag
- [ ] Cache hit/miss rates

### SLO / SLA Definition

Before a system goes to production, define:

```yaml
SLO:
  availability: 99.9%
  latency:
    p95: 500ms
    p99: 2000ms
  error_rate: < 0.1%

Alert thresholds:
  page immediately if:
    - error_rate > 5% over 5 minutes
    - p99 latency > 5000ms over 5 minutes
  notify if:
    - error_rate > 1% over 30 minutes
    - p95 latency > 2000ms over 30 minutes
```

---

## ADR — Architecture Decision Record

Every significant architectural decision must produce an ADR in `/adr/`. Use this template:

```markdown
# ADR-NNNN: [Short title]

**Date:** YYYY-MM-DD
**Status:** Proposed | Accepted | Deprecated | Superseded by ADR-NNNN
**Deciders:** [Names or roles]

## Context

[Describe the situation and problem forcing this decision]

## Decision

[Describe the decision made]

## Options Considered

### Option A: [Name]
**Pros:** ...
**Cons:** ...

### Option B: [Name]
**Pros:** ...
**Cons:** ...

## Consequences

**Positive:** [What becomes easier or better]
**Negative:** [What becomes harder or worse]
**Risks:** [What could go wrong]

## References

[Links to relevant documents, RFCs, or prior art]
```

---

## References

- [Martin Fowler — Architecture](https://martinfowler.com/architecture/)
- [Designing Data-Intensive Applications — Kleppmann](https://dataintensive.net/)
- [Building Microservices — Sam Newman](https://samnewman.io/books/building_microservices/)
- [The System Design Primer](https://github.com/donnemartin/system-design-primer)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Google SRE Book](https://sre.google/sre-book/table-of-contents/)
- [Stripe API Design Principles](https://stripe.com/docs/api)
