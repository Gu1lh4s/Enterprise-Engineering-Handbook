# Master Engineering Checklists

> **Category:** Quality Gates
> **Version:** 1.0.0
> **Level:** All Engineers

---

## Table of Contents

1. [Pre-Deploy Checklist](#1-pre-deploy-checklist)
2. [Code Review Checklist](#2-code-review-checklist)
3. [Security Review Checklist](#3-security-review-checklist)
4. [API Design Checklist](#4-api-design-checklist)
5. [Database Migration Checklist](#5-database-migration-checklist)
6. [New Service Launch Checklist](#6-new-service-launch-checklist)
7. [Incident Readiness Checklist](#7-incident-readiness-checklist)
8. [Data Privacy Checklist (GDPR/LGPD)](#8-data-privacy-checklist-gdprlgpd)
9. [Frontend Launch Checklist](#9-frontend-launch-checklist)
10. [On-Call Handoff Checklist](#10-on-call-handoff-checklist)

---

## 1. Pre-Deploy Checklist

**Must pass ALL before merging to main and deploying to production.**

### Code Quality
- [ ] All CI checks passing (lint, typecheck, unit tests, integration tests)
- [ ] No new TypeScript `any` types without justification
- [ ] No `console.log` / debug code left in
- [ ] No commented-out code blocks
- [ ] No TODO comments without a linked ticket

### Testing
- [ ] New code paths have unit tests
- [ ] API changes have integration tests
- [ ] Happy path + error cases covered
- [ ] No test snapshots updated without intentional change

### Security
- [ ] No secrets in code or comments (run: `git log -p | grep -iE "password|secret|key|token"`)
- [ ] No `eval()`, `innerHTML` without sanitization, `dangerouslySetInnerHTML` unguarded
- [ ] Input validation at all API boundaries (Zod/Joi/class-validator)
- [ ] SQL queries use parameterized form (not string concatenation)
- [ ] New endpoints behind authentication middleware

### Database
- [ ] Migrations tested on staging with production-size data
- [ ] No `DROP COLUMN` without deprecation period (or confirmed zero usage)
- [ ] No destructive migrations in same deploy as code change (run migration first)
- [ ] New columns are nullable or have a DEFAULT (for zero-downtime migration)
- [ ] New indexes created CONCURRENTLY (doesn't lock table)

### Observability
- [ ] New endpoints emit metrics (request count, duration)
- [ ] New error conditions are logged with context (not just `console.error`)
- [ ] Key business events have structured log entries
- [ ] Alerts updated if behavior changes

### Configuration
- [ ] New environment variables added to `.env.example` and deployment config
- [ ] Kubernetes secrets/ConfigMaps updated for new env vars
- [ ] Feature flag created for risky changes (can turn off without deploy)

### Deployment
- [ ] Staged rollout: deploy to staging → smoke test → production
- [ ] Rollback plan documented (how to revert if this breaks)
- [ ] Deployment window: prefer off-peak hours for risky changes
- [ ] On-call engineer notified of the deploy

---

## 2. Code Review Checklist

**For reviewers — what to look for beyond correctness.**

### Correctness
- [ ] Logic handles all edge cases (null, empty list, zero, negative numbers)
- [ ] Race conditions considered (parallel requests, concurrent mutations)
- [ ] Error handling: all exceptions caught or explicitly propagated
- [ ] Async/await used correctly (no floating promises)
- [ ] Correct HTTP status codes returned

### Security
- [ ] Authentication enforced on protected endpoints
- [ ] Authorization checked (user can only access their own resources)
- [ ] Input validated and sanitized before use
- [ ] No sensitive data logged (passwords, tokens, PII)
- [ ] Cryptographic operations use standard library (not home-grown)
- [ ] IDOR prevented: `WHERE id = $1 AND tenant_id = $2`

### Performance
- [ ] No N+1 queries (check for loops with DB calls inside)
- [ ] Pagination on list endpoints (no `SELECT *` without LIMIT)
- [ ] Expensive operations cached where appropriate
- [ ] No synchronous operations in async request handlers
- [ ] Database indexes exist for new WHERE/JOIN conditions

### Maintainability
- [ ] Functions do one thing (single responsibility)
- [ ] Names are self-explanatory (no `data`, `thing`, `foo`)
- [ ] Magic numbers extracted to named constants
- [ ] Complex logic has explanatory comment (not what, but why)
- [ ] No duplication that should be extracted to shared utility

### API Design
- [ ] Breaking changes versioned (new endpoint, not modified existing)
- [ ] Error responses use standard format
- [ ] OpenAPI spec updated for new/changed endpoints
- [ ] Idempotent where appropriate (PUT, DELETE, retry-safe POST)

---

## 3. Security Review Checklist

**Run for any feature touching: auth, payments, PII, file uploads, webhooks, admin functions.**

### Authentication and Authorization
- [ ] All endpoints protected appropriately (auth/no-auth documented)
- [ ] JWT: algorithm pinned (`algorithms: ['HS256']`), expiry checked
- [ ] Session tokens: HttpOnly, Secure, SameSite=Strict
- [ ] Password reset: time-limited tokens, invalidated on use, no user enumeration
- [ ] MFA enforced for high-privilege operations
- [ ] RBAC: tested with user having lowest expected privilege

### Injection Prevention
- [ ] SQL: all queries parameterized (zero string concatenation)
- [ ] NoSQL: input validated before use in query operators
- [ ] XSS: innerHTML never used with untrusted input; React JSX auto-escapes
- [ ] Command injection: no `exec(userInput)`, `spawn(userInput)`
- [ ] Path traversal: file paths validated/normalized before use

### Data Handling
- [ ] PII: logged only when necessary, masked in logs (`email: *@example.com`)
- [ ] Payment card data: never stored in database (tokenize via Stripe)
- [ ] Passwords: never logged, always hashed with bcrypt/Argon2
- [ ] Sensitive data: not in URL params (appear in access logs)
- [ ] Data minimization: only collect what's needed for the stated purpose

### Cryptography
- [ ] Secrets not hardcoded (use env vars or secret manager)
- [ ] Random: `crypto.randomBytes()` or `crypto.getRandomValues()` (not `Math.random()`)
- [ ] Tokens: min 32 bytes of entropy
- [ ] Comparison of secrets: `crypto.timingSafeEqual()` (prevent timing attacks)
- [ ] TLS: enforced, certificate validation not disabled

### Infrastructure
- [ ] New ports not exposed unnecessarily
- [ ] New external HTTP calls: validate domain against allowlist (prevent SSRF)
- [ ] File uploads: type validated by content (not extension), virus scanned
- [ ] Rate limiting on new user-facing endpoints
- [ ] Error messages don't leak internal details (stack traces, SQL, paths)

### Third-Party / Supply Chain
- [ ] New dependencies checked: `npm audit`, Snyk, license check
- [ ] No new dependency for a task that can be done with existing deps
- [ ] Webhook handlers verify signature before processing payload
- [ ] OAuth: state parameter validated, PKCE for mobile/SPA

---

## 4. API Design Checklist

### Resource Design
- [ ] Resources are nouns (not verbs)
- [ ] Plural for collections: `/bookings`, not `/booking`
- [ ] Nesting max 2 levels: `/users/{id}/bookings`
- [ ] Actions as sub-resources: `POST /bookings/{id}/cancel`

### HTTP Semantics
- [ ] GET: safe and idempotent (no side effects)
- [ ] POST: for creating resources (returns 201 with Location header)
- [ ] PUT: full replacement (idempotent)
- [ ] PATCH: partial update
- [ ] DELETE: idempotent (second delete returns 404 or 204)

### Response Format
- [ ] Success: `{ data: ... }` envelope
- [ ] Collection: `{ data: [...], meta: { hasMore, nextCursor } }`
- [ ] Error: `{ error: { code, message, status, details, requestId } }`
- [ ] Error codes: SCREAMING_SNAKE_CASE application codes (not just HTTP status)
- [ ] Timestamps: ISO 8601 UTC (`2025-06-27T10:00:00Z`)
- [ ] IDs: UUIDs or opaque strings (not auto-increment integers in public API)
- [ ] Money: integers in smallest unit (centavos/cents, not floats)

### Versioning and Compatibility
- [ ] New required fields behind version bump
- [ ] Adding optional fields to response is backward-compatible (safe)
- [ ] Removing fields: deprecation header first, removal after sunset date
- [ ] `Deprecation:` and `Sunset:` headers on deprecated endpoints

### Documentation
- [ ] OpenAPI spec updated with new endpoints, schemas, and examples
- [ ] Request/response examples in spec are realistic
- [ ] Error cases documented in spec
- [ ] Authentication documented

### Security
- [ ] Rate limiting applied
- [ ] Authentication required (or explicitly marked as public)
- [ ] CORS configured (no wildcard `*` for authenticated APIs)
- [ ] Request size limit configured (default Express: 100KB; adjust per use case)

---

## 5. Database Migration Checklist

**Every migration that touches production must pass this checklist.**

### Safety
- [ ] Migration is idempotent (can run twice without error)
- [ ] No `DROP TABLE`, `DROP COLUMN`, `TRUNCATE` without explicit approval
- [ ] No column rename (= DROP + ADD for running application)
- [ ] No `NOT NULL` without default on existing tables (use 3-step migration)
- [ ] No EXCLUSIVE LOCK operations (most DDL) during peak hours

### Zero-Downtime Compatibility (for rolling deploys)
- [ ] Backward compatible with the current code version (old code + new schema works)
- [ ] Forward compatible with the new code version (new code + old schema fallback)
- [ ] New indexes: `CREATE INDEX CONCURRENTLY` (non-blocking)
- [ ] New columns: nullable or with default (code handles both old and new state)

### Three-Step NOT NULL Migration (mandatory pattern)

```sql
-- Step 1: Add nullable column (deploy with code that writes it)
ALTER TABLE bookings ADD COLUMN cancelled_reason VARCHAR(500);

-- Step 2: Backfill existing rows (after code deployed and writing)
UPDATE bookings SET cancelled_reason = '' WHERE cancelled_reason IS NULL;

-- Step 3: Add NOT NULL constraint (after all rows have values)
ALTER TABLE bookings ALTER COLUMN cancelled_reason SET NOT NULL;
```

### Testing
- [ ] Tested on a copy of production schema (not just dev database)
- [ ] Estimated duration at production data volume (`EXPLAIN` + row count estimate)
- [ ] Rollback plan exists and tested (migration can be reversed)
- [ ] Application works during migration (zero-downtime verified)

### Review
- [ ] DBA or senior engineer reviewed for performance impact
- [ ] Scheduled during low-traffic window if > 1 minute estimated duration
- [ ] Team notified of migration timing

---

## 6. New Service Launch Checklist

**Any new microservice, API, or major feature going to production.**

### Operational Readiness
- [ ] Health check endpoints: `/health` (liveness) and `/health/ready` (readiness)
- [ ] Metrics emitted: request rate, error rate, duration (RED)
- [ ] Structured logging (JSON, with traceId)
- [ ] Distributed tracing instrumented (OpenTelemetry)
- [ ] Alerts configured: error rate, latency, saturation
- [ ] Dashboard created (Grafana) for key metrics
- [ ] On-call runbook written and linked from alert

### Infrastructure
- [ ] Kubernetes Deployment with resource requests/limits
- [ ] HPA configured (min/max replicas, CPU/memory targets)
- [ ] PodDisruptionBudget configured (min available during node maintenance)
- [ ] Network Policy (restrict ingress/egress to required only)
- [ ] Pod security context (non-root, read-only rootfs, no privilege escalation)
- [ ] Image from private registry, pinned to SHA (not `:latest`)

### Security
- [ ] Threat model completed (STRIDE)
- [ ] Security review passed (checklist §3)
- [ ] Penetration test scheduled (within 3 months of launch)
- [ ] Secrets in Kubernetes Secrets or External Secrets (not ConfigMaps)
- [ ] TLS enforced for all external communication
- [ ] Rate limiting configured

### Reliability
- [ ] SLO defined (target availability, latency)
- [ ] Error budget policy documented
- [ ] Graceful shutdown implemented (handles SIGTERM)
- [ ] Circuit breaker for downstream dependencies
- [ ] Retry logic with exponential backoff
- [ ] DR plan: how to recover if this service is completely lost

### Compliance (if handling PII/payment data)
- [ ] Data flow documented (what PII enters and exits this service)
- [ ] RoPA (Record of Processing Activities) updated
- [ ] DPIA completed if high-risk processing
- [ ] Retention policy implemented
- [ ] Data erasure capability verified (GDPR/LGPD right to erasure)

---

## 7. Incident Readiness Checklist

**Run quarterly — verify your incident response capability is current.**

### Detection
- [ ] All production services have alerting configured
- [ ] Alerts fire within 5 minutes of issue onset (verify with test)
- [ ] PagerDuty rotation current (no stale on-call schedules)
- [ ] Status page (status.example.com) can be updated independently of application

### Communication
- [ ] Incident channel in Slack exists and all engineers are in it
- [ ] Escalation matrix is current (who to page at what hour)
- [ ] Customer notification templates drafted and reviewed
- [ ] Regulator notification templates drafted (GDPR 72h template ready)
- [ ] PR/communications team knows the breach notification process

### Recovery
- [ ] Last backup restore was tested within 90 days
- [ ] RTO/RPO have been measured and documented
- [ ] Rollback procedure for deployments is tested and documented
- [ ] DR runbook is current and accessible (not just in production wiki)

### Team Readiness
- [ ] Incident playbooks reviewed by current team (may have changed since written)
- [ ] New engineers have been through at least one tabletop exercise
- [ ] Chaos engineering exercises planned (Gremlin, LitmusChaos)
- [ ] Post-mortems for all P0/P1 incidents are completed and reviewed

---

## 8. Data Privacy Checklist (GDPR/LGPD)

**For any feature that collects, processes, or stores personal data.**

### Lawful Basis
- [ ] Identified lawful basis for processing (consent, contract, legal obligation, legitimate interest)
- [ ] If consent: it is specific, informed, freely given, and withdrawable
- [ ] If legitimate interest: LIA (Legitimate Interest Assessment) completed

### Data Minimization
- [ ] Only collecting data actually needed for the stated purpose
- [ ] Not storing data longer than necessary (retention policy set)
- [ ] Not sharing data with more parties than necessary

### Data Subject Rights
- [ ] Data export (DSAR) capability: user can export their data
- [ ] Data deletion capability: user can request erasure
- [ ] Data correction capability: user can correct inaccurate data
- [ ] Portability: export in machine-readable format (JSON/CSV)
- [ ] Rights honored within statutory timeframes (GDPR: 1 month; LGPD: 15 days)

### Records and Documentation
- [ ] RoPA (Record of Processing Activities) entry created
- [ ] DPIA completed if high-risk (biometrics, health, large-scale monitoring)
- [ ] Privacy notice updated to reflect new processing

### Technical Controls
- [ ] Data encrypted at rest
- [ ] Data encrypted in transit
- [ ] Access logged for sensitive data
- [ ] No PII in logs (or masked)
- [ ] Cross-border transfer: standard contractual clauses in place for non-adequate countries

---

## 9. Frontend Launch Checklist

### Performance
- [ ] Lighthouse score > 90 (Performance, Accessibility, Best Practices, SEO)
- [ ] LCP < 2.5s on mobile
- [ ] CLS < 0.1
- [ ] Bundle size < 200KB (initial JS, compressed)
- [ ] Images: WebP/AVIF format, lazy loaded, correct sizes attribute
- [ ] Critical CSS inlined, non-critical deferred

### Accessibility (WCAG 2.1 AA)
- [ ] All images have meaningful alt text (decorative: `alt=""`)
- [ ] Color contrast ≥ 4.5:1 for normal text, 3:1 for large
- [ ] All interactive elements reachable via keyboard
- [ ] Focus indicators visible
- [ ] Forms: labels associated, errors associated with aria-describedby
- [ ] Modals: focus trapped, closes on Escape
- [ ] Screen reader tested (NVDA/VoiceOver spot check)

### Security
- [ ] CSP (Content Security Policy) configured
- [ ] No sensitive data in URL params
- [ ] Authentication tokens not in localStorage
- [ ] HTTPS enforced (HSTS header set)
- [ ] Third-party scripts minimized and reviewed

### Cross-Browser / Device
- [ ] Tested on Chrome, Firefox, Safari, Edge
- [ ] Tested on iPhone (Safari) and Android (Chrome)
- [ ] Responsive at 320px, 768px, 1024px, 1440px breakpoints
- [ ] Tested with slow network (Chrome DevTools: Slow 3G)

### SEO (for public pages)
- [ ] Unique, descriptive `<title>` (< 60 chars)
- [ ] Meta description (< 160 chars)
- [ ] Open Graph tags for sharing
- [ ] Canonical URLs set
- [ ] robots.txt and sitemap.xml present

---

## 10. On-Call Handoff Checklist

**Complete before transferring on-call responsibility.**

### Current State
- [ ] Active incidents documented (none, or full status of each)
- [ ] Degraded services documented (any known issues, workarounds)
- [ ] Scheduled maintenance in next 24h documented
- [ ] Recent deploys in last 48h documented (watch for late-breaking issues)

### Access Verification
- [ ] Incoming engineer has PagerDuty access and is on schedule
- [ ] Incoming engineer can access production Kubernetes cluster
- [ ] Incoming engineer can access production database (read-only at minimum)
- [ ] Incoming engineer has Slack access to incident channels
- [ ] Incoming engineer knows how to update status page

### Knowledge Transfer
- [ ] Any known intermittent issues briefed (flaky alerts, known anomalies)
- [ ] Any customer escalations in progress briefed
- [ ] Any risky operations planned (e.g., migration at 3am) communicated
- [ ] Runbook links shared for any non-standard conditions

### Emergency Contacts
- [ ] Security escalation contact (CISO/security team)
- [ ] Engineering leadership contact
- [ ] Infrastructure/cloud provider contact (AWS support case link)
- [ ] DBA contact (if DB issues require specialist)
```
