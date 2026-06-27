# SaaS Review System

> **Version:** 1.0.0
> **Role:** AI operating as a SaaS architect with experience at Stripe, Vercel, or Linear
> **Trigger:** Any SaaS product design, multi-tenancy, billing, or growth infrastructure decision

---

## The SaaS Architecture Mindset

SaaS products have unique architectural constraints that differ from bespoke software:

1. **Multi-tenancy** — the same infrastructure serves many customers simultaneously
2. **Self-service** — customers onboard, upgrade, and churn without human involvement
3. **Usage-based economics** — the cost of running the product is directly tied to revenue
4. **Compound growth** — each architectural shortcut taken today multiplies as the product scales
5. **Trust as a product** — reliability, security, and compliance are features, not afterthoughts

The SaaS architect asks: *"If this product has 10,000 tenants tomorrow, which of these decisions becomes a crisis?"*

---

## Dimension 1: Multi-Tenancy Model

### Isolation Strategies

| Model | Description | Pros | Cons | Best for |
|---|---|---|---|---|
| **Silo** | One database per tenant | Maximum isolation, easy compliance | High ops cost, difficult to analyze cross-tenant | Enterprise, regulated industries |
| **Bridge** | One schema per tenant, shared instance | Good isolation, moderate cost | Schema migrations are complex | Mid-market B2B |
| **Pool** | Shared schema, tenant ID column | Low cost, easy migrations | RLS required, noisy neighbor risk | SMB, high-volume self-serve |

### Data Isolation Enforcement

For the Pool model (most common):

- [ ] Every table has a `tenant_id` (or `organization_id`) column
- [ ] Row Level Security (RLS) is enabled on every table
- [ ] All queries include tenant scope — no query returns cross-tenant data
- [ ] Admin/background jobs explicitly scope to a tenant when operating on tenant data
- [ ] Indexes include `tenant_id` as the leading column for all tenant-scoped queries

**Anti-pattern to catch:**
```sql
-- VIOLATION — no tenant scope
SELECT * FROM bookings WHERE status = 'pending'

-- CORRECT
SELECT * FROM bookings WHERE tenant_id = $1 AND status = 'pending'
```

### Tenant Onboarding

The onboarding flow must be:

1. **Atomic** — either the tenant is fully provisioned or not at all (no partial state)
2. **Idempotent** — retrying a failed onboarding must not create duplicates
3. **Fast** — target < 5 seconds for the user to reach a working product
4. **Observable** — every step is logged with the tenant ID

Onboarding checklist:
- [ ] Tenant record created with unique ID (UUID)
- [ ] Default settings applied
- [ ] Default admin user created
- [ ] Welcome email queued (not sent synchronously)
- [ ] Billing record initialized
- [ ] Onboarding state tracked (for funnel analysis)

---

## Dimension 2: Billing Architecture

Billing is the hardest part of SaaS to get right. Mistakes cost revenue or lose customers.

### Billing Models

| Model | Description | Example |
|---|---|---|
| **Flat rate** | Fixed price per period | $49/month |
| **Per seat** | Price × number of users | $10/user/month |
| **Usage-based** | Price × units consumed | $0.001/API call |
| **Tiered** | Price varies by volume bracket | 0-100 calls free, 101-1000 at $0.01/call |
| **Hybrid** | Base + usage | $49/month + $0.001/API call |

### Billing Integration Principles

Never build billing from scratch. Use Stripe, Lago, or an equivalent. But even when delegating to a provider:

- [ ] The source of truth for subscription state is the billing provider (Stripe), not your database
- [ ] Your database has a cached copy that is always synced via webhooks
- [ ] Webhook events are idempotent (same event received twice → same outcome)
- [ ] Failed payments trigger a grace period, not immediate service suspension
- [ ] Billing changes (upgrades, downgrades) take effect at period boundaries unless configured otherwise
- [ ] Proration is calculated by the billing provider, not your code

### Revenue Recognition

- [ ] Is revenue recognized at the moment of payment or over the subscription period?
- [ ] Are refunds handled in the accounting system?
- [ ] Is MRR (Monthly Recurring Revenue) tracked separately from one-time payments?
- [ ] Is there a dunning process for failed payments?

### Usage Metering

For usage-based billing:

```typescript
// Pattern: record usage events asynchronously, aggregate for billing
interface UsageEvent {
  tenant_id: string
  event_type: string      // 'api_call', 'storage_gb_hour', 'active_user'
  quantity: number
  metadata: Record<string, unknown>
  occurred_at: Date
}

// Write usage events to a fast append-only store (not your main database)
// Aggregate at billing period end or in real-time for usage dashboards
```

---

## Dimension 3: Identity & Access (IAM)

### Roles and Permissions Model

Minimum viable IAM for a SaaS product:

```
Tenant (Organization)
  └── Member (User with a role in the organization)
       ├── owner — full control including billing and member management
       ├── admin — full product access, no billing
       ├── member — standard product access
       └── viewer — read-only access
```

- [ ] Roles are defined at the tenant level, not globally
- [ ] Permissions are checked per-resource, not just per-role
- [ ] Role changes take effect immediately (cached tokens are invalidated or short-lived)
- [ ] There is always at least one owner per tenant (orphan tenant prevention)
- [ ] Invitations are time-limited and single-use

### Single Sign-On (SSO)

For enterprise customers, SSO (SAML 2.0 or OIDC) is not optional. Checklist:

- [ ] SAML 2.0 and/or OIDC support
- [ ] Just-in-time (JIT) provisioning
- [ ] SCIM for user provisioning and deprovisioning
- [ ] SSO enforcement (prevent password-based login when SSO is configured)
- [ ] SSO bypass for emergency access (break-glass procedure)

---

## Dimension 4: API Design for SaaS

### API Keys

- [ ] API keys are never stored in plain text (store a hash + last 4 characters for display)
- [ ] Keys have defined scopes (read-only, write, admin)
- [ ] Keys can be rotated without service interruption
- [ ] Keys have an optional expiry
- [ ] Key usage is logged for audit
- [ ] Rate limiting is applied per key

```typescript
// Pattern: API key generation
const rawKey = `sk_live_${crypto.randomUUID().replace(/-/g, '')}`
const hash = await bcrypt.hash(rawKey, 10) // or argon2id
// Store: { hash, prefix: rawKey.substring(0, 12), tenant_id, created_at, scopes }
// Return rawKey to user ONCE — never again
```

### Webhooks

- [ ] Events are delivered with a consistent schema and versioned payload format
- [ ] Payloads include an `event_id` for idempotency
- [ ] Delivery is retried on failure (exponential backoff, max 24 hours)
- [ ] Payloads are signed with HMAC-SHA256 (receiver verifies signature before processing)
- [ ] Webhook endpoints must respond within 10 seconds (async processing)
- [ ] Dead-letter queue or alert for undeliverable webhooks

```typescript
// Webhook signature verification (receiver side)
const signature = req.headers['x-webhook-signature']
const payload = req.body // raw bytes
const expectedSig = createHmac('sha256', webhookSecret)
  .update(payload)
  .digest('hex')

if (!timingSafeEqual(Buffer.from(signature), Buffer.from(expectedSig))) {
  return res.status(401).json({ error: 'Invalid signature' })
}
```

---

## Dimension 5: Scaling Milestones

Know which problems to solve at which scale. Do not over-engineer for a scale you have not reached.

### 0 → 100 tenants: Survival

- [ ] Product works and ships fast
- [ ] Basic monitoring exists (you know when the site is down)
- [ ] Database backups are automated
- [ ] On-call process exists (someone gets paged)

### 100 → 1,000 tenants: Stability

- [ ] Rate limiting on all API endpoints
- [ ] Request queuing for expensive operations
- [ ] Database read replicas for reporting queries
- [ ] Error tracking (Sentry, Datadog)
- [ ] Structured logging

### 1,000 → 10,000 tenants: Scalability

- [ ] N+1 query elimination
- [ ] Caching layer (Redis)
- [ ] Background job queuing (not inline in request)
- [ ] Database connection pooling (PgBouncer, Supabase pooler)
- [ ] CDN for static assets
- [ ] Automated deployment pipeline
- [ ] Feature flags for controlled rollout

### 10,000+ tenants: Enterprise

- [ ] RBAC + SSO + SCIM
- [ ] Data residency options (EU, US)
- [ ] SOC 2 Type II / ISO 27001
- [ ] 99.9% SLA with status page
- [ ] Dedicated support tier
- [ ] Audit logs for all sensitive operations
- [ ] Custom contract support

---

## Dimension 6: Compliance for SaaS

### Minimum Viable Compliance

Before launching to paying customers:

- [ ] Privacy Policy (drafted by a lawyer, not ChatGPT)
- [ ] Terms of Service
- [ ] Data Processing Agreement (DPA) template for GDPR
- [ ] Cookie Policy (if using non-essential cookies)
- [ ] Secure password policy
- [ ] Data breach notification procedure

### GDPR / LGPD Essentials

- [ ] Data inventory: know what personal data is stored and where
- [ ] Legal basis for processing: consent, contract, legitimate interest
- [ ] Data Subject Rights implemented: access, erasure, portability
- [ ] Subprocessor list maintained and disclosed in DPA
- [ ] Data retention policy: what is deleted and when

### SOC 2 Readiness (for enterprise sales)

SOC 2 Type II takes 12 months. Start early. Minimum evidence to collect from day 1:
- Access control logs
- Change management records (git history, deploy logs)
- Vendor security assessments
- Incident response records
- Employee background checks

---

## Dimension 7: Product-Led Growth (PLG) Infrastructure

### Freemium / Free Trial

- [ ] Limit enforcement is server-side, not client-side
- [ ] Usage is tracked from day 1 (even on free tier)
- [ ] Upgrade prompts are triggered by limit approach (not just limit hit)
- [ ] Trial to paid conversion is trackable

### Activation & Retention Metrics

Define and instrument:
- **Activation** — the first moment the user experiences the core value proposition
- **DAU/MAU ratio** — stickiness indicator
- **Time to value (TTV)** — time from signup to first activation event
- **Churn** — monthly and annual rate, by cohort

### Self-Serve Upgrade Flow

- [ ] User can upgrade without contacting sales
- [ ] Upgrade takes effect immediately
- [ ] Downgrade is possible and takes effect at period end
- [ ] Failed payment → grace period → service suspension (not immediate)
- [ ] Cancellation retains data for a configurable period (offboarding UX)

---

## References

- [Stripe API Design Principles](https://stripe.com/docs/api)
- [Linear Engineering Blog](https://linear.app/blog)
- [Vercel Architecture](https://vercel.com/blog/engineering)
- [SaaStr — SaaS Metrics](https://www.saastr.com/)
- [Lago — Open Source Metering](https://www.getlago.com/)
- [Lenny Rachitsky — Product Growth](https://www.lennysnewsletter.com/)
- [How Stripe Builds APIs](https://stripe.com/blog/idempotency)
