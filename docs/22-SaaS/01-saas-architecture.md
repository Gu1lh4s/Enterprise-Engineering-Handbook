# SaaS Architecture

> **Category:** Product Engineering
> **Version:** 1.0.0
> **Level:** Staff/Principal Engineer

---

## Table of Contents

1. [Multi-Tenancy Models](#1-multi-tenancy-models)
2. [Tenant Isolation Strategies](#2-tenant-isolation-strategies)
3. [Identity and Access Management](#3-identity-and-access-management)
4. [Billing and Subscription Management](#4-billing-and-subscription-management)
5. [Onboarding and Provisioning](#5-onboarding-and-provisioning)
6. [Webhook System](#6-webhook-system)
7. [Feature Flags](#7-feature-flags)
8. [Audit Logging](#8-audit-logging)
9. [Usage Metering](#9-usage-metering)
10. [SaaS Security Checklist](#10-saas-security-checklist)

---

## 1. Multi-Tenancy Models

```
Three main architectures — choose based on isolation requirements and operational cost:

1. Shared Database, Shared Schema ("Row-Level Tenancy")
   └── Single DB, single tables, tenant_id column on every table
   ✓ Easiest to operate (one DB to manage)
   ✓ Lowest per-tenant cost (no provisioning overhead)
   ✗ Risk of tenant data leakage (developer mistakes)
   ✗ "Noisy neighbor" problem (one tenant's query slows others)
   ✗ Hard to export or migrate one tenant's data
   
   Use when: SMB SaaS, < $1M ARR, security requirements are moderate

2. Shared Database, Separate Schema ("Schema-per-tenant")
   └── Single DB, separate PostgreSQL schemas per tenant
   ✓ Better isolation (RLS not needed; schema boundary)
   ✓ Easy tenant-level migrations
   ✗ Schema sprawl at scale (> 1000 tenants = complex)
   ✗ Connection multiplexing needed (can't have 1 pool per schema)
   
   Use when: regulated mid-market, moderate isolation needs

3. Separate Database Per Tenant ("Database-per-tenant")
   └── Each tenant has own RDS/Aurora instance
   ✓ Maximum isolation (breach of one DB doesn't expose others)
   ✓ Tenant-specific backups and compliance
   ✗ Operationally expensive (> $50/mo per tenant DB)
   ✗ Requires database provisioning automation
   
   Use when: enterprise/regulated customers, HIPAA/FedRAMP requirements

Hybrid: single DB for small tenants; dedicated DB for enterprise tenants
```

---

## 2. Tenant Isolation Strategies

### Row-Level Security (Shared Schema)

```sql
-- Every table has tenant_id
CREATE TABLE bookings (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id   UUID NOT NULL REFERENCES organizations(id),
  client_id   UUID NOT NULL REFERENCES users(id),
  service_id  UUID NOT NULL REFERENCES services(id),
  scheduled_at TIMESTAMPTZ NOT NULL,
  status      VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Index for tenant scoping (always include tenant_id in queries)
CREATE INDEX idx_bookings_tenant ON bookings(tenant_id, created_at DESC);

-- Row Level Security
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;
ALTER TABLE bookings FORCE ROW LEVEL SECURITY;

-- Policy: each row visible only to matching tenant
CREATE POLICY tenant_isolation ON bookings
  USING (tenant_id = current_setting('app.tenant_id')::UUID)
  WITH CHECK (tenant_id = current_setting('app.tenant_id')::UUID);

-- Application: set tenant context before every query
SET app.tenant_id = 'tenant_abc123';
```

```typescript
// Middleware: set tenant context from JWT
async function setTenantContext(req: Request, res: Response, next: NextFunction) {
  const tenantId = req.user?.tenantId
  if (!tenantId) return res.status(401).json({ error: 'No tenant context' })
  
  // Set on database connection for RLS
  await db.execute(sql`SET LOCAL app.tenant_id = ${tenantId}`)
  
  // Also keep in AsyncLocalStorage for logging
  tenantContext.run(tenantId, next)
}

// Repository: always include tenant_id for defense in depth
// Even though RLS enforces it, being explicit makes code reviewable
class BookingRepository {
  private readonly tenantId: string
  
  constructor(private db: Database, tenantId: string) {
    this.tenantId = tenantId
  }
  
  async findAll(): Promise<Booking[]> {
    // RLS would protect even without WHERE, but be explicit
    return this.db.query.bookings.findMany({
      where: eq(bookings.tenantId, this.tenantId),
      orderBy: [desc(bookings.createdAt)],
    })
  }
}
```

---

## 3. Identity and Access Management

### Organization and User Model

```sql
-- Tenant = Organization
CREATE TABLE organizations (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name            VARCHAR(255) NOT NULL,
  slug            VARCHAR(100) NOT NULL UNIQUE,  -- URL-safe identifier
  plan            VARCHAR(50) NOT NULL DEFAULT 'free',  -- free, starter, professional, enterprise
  billing_email   VARCHAR(255) NOT NULL,
  stripe_customer_id VARCHAR(100) UNIQUE,
  
  -- Limits per plan (enforce in application)
  max_seats       INTEGER NOT NULL DEFAULT 5,
  max_bookings_per_month INTEGER NOT NULL DEFAULT 100,
  
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE users (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email           VARCHAR(255) NOT NULL UNIQUE,
  name            VARCHAR(255) NOT NULL,
  email_verified  BOOLEAN NOT NULL DEFAULT FALSE,
  password_hash   VARCHAR(255),  -- NULL if SSO-only
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- M:M between users and organizations (user can belong to multiple orgs)
CREATE TABLE organization_members (
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id         UUID REFERENCES users(id) ON DELETE CASCADE,
  role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- owner, admin, member, viewer
  invited_by      UUID REFERENCES users(id),
  joined_at       TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (organization_id, user_id)
);

CREATE INDEX idx_org_members_user ON organization_members(user_id);
```

### Role-Based Access Control

```typescript
// RBAC: Permissions matrix per role
const PERMISSIONS = {
  // Owners can do everything
  owner: [
    'org:settings:read', 'org:settings:write', 'org:billing:read', 'org:billing:write',
    'org:members:read', 'org:members:invite', 'org:members:remove',
    'bookings:create', 'bookings:read', 'bookings:update', 'bookings:cancel',
    'services:create', 'services:update', 'services:delete',
    'reports:read',
  ],
  
  admin: [
    'org:settings:read', 'org:members:read', 'org:members:invite',
    'bookings:create', 'bookings:read', 'bookings:update', 'bookings:cancel',
    'services:create', 'services:update',
    'reports:read',
  ],
  
  member: [
    'bookings:create', 'bookings:read', 'bookings:update', 'bookings:cancel',
    'services:read',
  ],
  
  viewer: ['bookings:read', 'services:read', 'reports:read'],
} as const satisfies Record<string, string[]>

type Role = keyof typeof PERMISSIONS
type Permission = typeof PERMISSIONS[Role][number]

function hasPermission(role: Role, permission: Permission): boolean {
  return (PERMISSIONS[role] as readonly string[]).includes(permission)
}

// Middleware
function requirePermission(permission: Permission) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!hasPermission(req.user.role as Role, permission)) {
      return res.status(403).json({
        error: { code: 'FORBIDDEN', message: `Requires permission: ${permission}` }
      })
    }
    next()
  }
}

// Route protection
router.post('/services', requirePermission('services:create'), createServiceController)
router.delete('/members/:id', requirePermission('org:members:remove'), removeMemberController)
```

### Invitation Flow

```typescript
async function inviteMember(inviterId: string, orgId: string, email: string, role: string) {
  // Check inviter has permission
  const inviter = await getMemberRole(inviterId, orgId)
  if (!hasPermission(inviter.role, 'org:members:invite')) throw new ForbiddenError()
  
  // Check seat limit
  const org = await getOrganization(orgId)
  const currentSeats = await countMembers(orgId)
  if (currentSeats >= org.maxSeats) throw new SeatLimitExceededError()
  
  // Create invitation token (JWT with 72h expiry)
  const token = jwt.sign({ orgId, email, role, type: 'invitation' }, process.env.JWT_SECRET!, { expiresIn: '72h' })
  
  await db.insert(invitations).values({
    organizationId: orgId,
    email,
    role,
    token,
    invitedById: inviterId,
    expiresAt: new Date(Date.now() + 72 * 60 * 60 * 1000),
  })
  
  await emailService.sendInvitation({ email, inviterName: inviter.name, orgName: org.name, token })
}

async function acceptInvitation(token: string, userId: string) {
  const payload = jwt.verify(token, process.env.JWT_SECRET!) as any
  
  const invitation = await db.query.invitations.findFirst({
    where: and(
      eq(invitations.token, token),
      eq(invitations.acceptedAt, null),
      gt(invitations.expiresAt, new Date())
    )
  })
  
  if (!invitation) throw new InvalidInvitationError()
  
  await db.transaction(async (tx) => {
    await tx.insert(organizationMembers).values({
      organizationId: invitation.organizationId,
      userId,
      role: invitation.role,
      invitedBy: invitation.invitedById,
    })
    
    await tx.update(invitations)
      .set({ acceptedAt: new Date() })
      .where(eq(invitations.token, token))
  })
}
```

---

## 4. Billing and Subscription Management

### Stripe Integration

```typescript
// stripe.service.ts
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, { apiVersion: '2024-06-20' })

// Plan → Stripe Price IDs
const PLANS = {
  starter:      { priceId: 'price_starter_monthly',      seats: 5,   bookingsPerMonth: 200 },
  professional: { priceId: 'price_professional_monthly', seats: 25,  bookingsPerMonth: 1000 },
  enterprise:   { priceId: 'price_enterprise_monthly',   seats: 999, bookingsPerMonth: 99999 },
} as const

// Create Stripe Customer when organization is created
async function createStripeCustomer(org: Organization): Promise<string> {
  const customer = await stripe.customers.create({
    email: org.billingEmail,
    name: org.name,
    metadata: { organizationId: org.id },
  })
  
  await db.update(organizations)
    .set({ stripeCustomerId: customer.id })
    .where(eq(organizations.id, org.id))
  
  return customer.id
}

// Start subscription
async function startSubscription(orgId: string, plan: keyof typeof PLANS): Promise<string> {
  const org = await getOrganization(orgId)
  const planConfig = PLANS[plan]
  
  const subscription = await stripe.subscriptions.create({
    customer: org.stripeCustomerId!,
    items: [{ price: planConfig.priceId }],
    payment_behavior: 'default_incomplete',
    payment_settings: { save_default_payment_method: 'on_subscription' },
    expand: ['latest_invoice.payment_intent'],
  })
  
  const invoice = subscription.latest_invoice as Stripe.Invoice
  const paymentIntent = invoice.payment_intent as Stripe.PaymentIntent
  
  return paymentIntent.client_secret!  // Return to client for Stripe Elements
}

// Stripe Webhooks handler (events drive subscription state)
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), async (req, res) => {
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      req.body,
      req.headers['stripe-signature'] as string,
      process.env.STRIPE_WEBHOOK_SECRET!
    )
  } catch {
    return res.status(400).send('Invalid webhook signature')
  }
  
  switch (event.type) {
    case 'customer.subscription.created':
    case 'customer.subscription.updated': {
      const subscription = event.data.object as Stripe.Subscription
      const orgId = subscription.metadata.organizationId
      const planName = getPlanFromPriceId(subscription.items.data[0].price.id)
      const planConfig = PLANS[planName]
      
      await db.update(organizations)
        .set({
          plan: planName,
          maxSeats: planConfig.seats,
          maxBookingsPerMonth: planConfig.bookingsPerMonth,
          stripeSubscriptionId: subscription.id,
          subscriptionStatus: subscription.status,
        })
        .where(eq(organizations.id, orgId))
      break
    }
    
    case 'customer.subscription.deleted': {
      const subscription = event.data.object as Stripe.Subscription
      const orgId = subscription.metadata.organizationId
      
      await db.update(organizations)
        .set({ plan: 'free', maxSeats: 5, subscriptionStatus: 'canceled' })
        .where(eq(organizations.id, orgId))
      
      await notifyOrgOwner(orgId, 'subscription_cancelled')
      break
    }
    
    case 'invoice.payment_failed': {
      const invoice = event.data.object as Stripe.Invoice
      const orgId = (await stripe.customers.retrieve(invoice.customer as string)).metadata.organizationId
      await notifyOrgOwner(orgId, 'payment_failed')
      break
    }
  }
  
  res.json({ received: true })
})
```

### Customer Portal

```typescript
// Stripe Customer Portal: self-service subscription management (no code needed)
async function createPortalSession(orgId: string, returnUrl: string): Promise<string> {
  const org = await getOrganization(orgId)
  
  const session = await stripe.billingPortal.sessions.create({
    customer: org.stripeCustomerId!,
    return_url: returnUrl,
  })
  
  return session.url  // Redirect user here
}
```

---

## 5. Onboarding and Provisioning

```typescript
// Automated onboarding: create everything needed in one transaction
async function provisionNewOrganization(input: ProvisionOrganizationInput): Promise<{ org: Organization; owner: User }> {
  return db.transaction(async (tx) => {
    // 1. Create organization
    const [org] = await tx.insert(organizations).values({
      name: input.orgName,
      slug: generateSlug(input.orgName),
      billingEmail: input.ownerEmail,
      plan: 'free',
    }).returning()
    
    // 2. Create owner user (or find existing)
    let user = await tx.query.users.findFirst({ where: eq(users.email, input.ownerEmail) })
    if (!user) {
      [user] = await tx.insert(users).values({
        email: input.ownerEmail,
        name: input.ownerName,
        passwordHash: await bcrypt.hash(input.password, 12),
      }).returning()
    }
    
    // 3. Link user to org as owner
    await tx.insert(organizationMembers).values({
      organizationId: org.id,
      userId: user.id,
      role: 'owner',
    })
    
    // 4. Create default service catalog
    await tx.insert(services).values([
      { tenantId: org.id, name: 'Massagem Relaxante', price: 15000, durationMin: 60 },
      { tenantId: org.id, name: 'Massagem Terapêutica', price: 20000, durationMin: 60 },
    ])
    
    // 5. Send verification email (async — outside transaction)
    setImmediate(() => emailService.sendVerification(user!.email))
    
    // 6. Create Stripe customer (async — don't block signup on billing)
    setImmediate(() => createStripeCustomer(org))
    
    return { org, owner: user! }
  })
}
```

---

## 6. Webhook System

```typescript
// Allow customers to subscribe to events in your SaaS

// Schema
CREATE TABLE webhook_endpoints (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL REFERENCES organizations(id),
  url             VARCHAR(500) NOT NULL,
  secret          VARCHAR(100) NOT NULL,  -- HMAC signing secret
  events          TEXT[] NOT NULL,         -- ['booking.created', 'booking.cancelled']
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,
  failure_count   INTEGER NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  endpoint_id     UUID REFERENCES webhook_endpoints(id),
  event_type      VARCHAR(100) NOT NULL,
  payload         JSONB NOT NULL,
  status          VARCHAR(20) NOT NULL,  -- pending, delivered, failed
  attempt_count   INTEGER NOT NULL DEFAULT 0,
  last_attempt_at TIMESTAMPTZ,
  next_attempt_at TIMESTAMPTZ,
  response_status INTEGER,
  response_body   TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

// Event dispatch
async function dispatchEvent(orgId: string, eventType: string, data: unknown) {
  const endpoints = await db.query.webhookEndpoints.findMany({
    where: and(
      eq(webhookEndpoints.organizationId, orgId),
      eq(webhookEndpoints.isActive, true),
      sql`${eventType} = ANY(${webhookEndpoints.events})`
    )
  })
  
  const payload = {
    id: randomUUID(),
    type: eventType,
    created: Math.floor(Date.now() / 1000),
    data,
  }
  
  // Queue delivery (non-blocking)
  await Promise.all(
    endpoints.map(endpoint =>
      webhookQueue.add('deliver', { endpointId: endpoint.id, payload }, {
        attempts: 5,
        backoff: { type: 'exponential', delay: 1000 },
      })
    )
  )
}
```

---

## 7. Feature Flags

```typescript
// Feature flags: ship code before features are active
// Decouple deployment from release; enable per-plan/per-org/per-user

// Simple: database-backed feature flags
CREATE TABLE feature_flags (
  key             VARCHAR(100) PRIMARY KEY,
  enabled_for_all BOOLEAN NOT NULL DEFAULT FALSE,
  enabled_for_plans TEXT[] DEFAULT '{}',     -- e.g., ['professional', 'enterprise']
  enabled_for_orgs  UUID[] DEFAULT '{}',     -- Specific orgs (for beta)
  rollout_percentage INTEGER DEFAULT 0,      -- 0-100 gradual rollout
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

// Feature flag service
class FeatureFlagService {
  async isEnabled(key: string, context: { orgId: string; plan: string }): Promise<boolean> {
    const flag = await this.getFlag(key)
    if (!flag) return false
    if (flag.enabledForAll) return true
    if (flag.enabledForPlans.includes(context.plan)) return true
    if (flag.enabledForOrgs.includes(context.orgId)) return true
    
    // Percentage rollout (deterministic: same org always gets same result)
    if (flag.rolloutPercentage > 0) {
      const hash = hashCode(context.orgId + key) % 100
      if (hash < flag.rolloutPercentage) return true
    }
    
    return false
  }
}

// Usage in API
async function getBookings(req: Request) {
  const canUseCalendarSync = await featureFlags.isEnabled('calendar_sync', {
    orgId: req.user.orgId,
    plan: req.user.plan,
  })
  
  const bookings = await bookingService.findAll(req.user.tenantId)
  
  return {
    data: bookings,
    features: { calendarSync: canUseCalendarSync },
  }
}
```

---

## 8. Audit Logging

```sql
-- Immutable audit log (never UPDATE or DELETE)
CREATE TABLE audit_logs (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID NOT NULL,
  actor_id        UUID NOT NULL,       -- Who performed the action
  actor_email     VARCHAR(255) NOT NULL, -- Denormalized (user might be deleted later)
  action          VARCHAR(100) NOT NULL, -- 'booking.created', 'member.removed', etc.
  resource_type   VARCHAR(50) NOT NULL,  -- 'booking', 'user', 'service'
  resource_id     UUID NOT NULL,
  before_state    JSONB,               -- Previous value (for updates)
  after_state     JSONB,               -- New value
  ip_address      INET,
  user_agent      VARCHAR(500),
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Partition by month for query performance
CREATE INDEX idx_audit_logs_org_time ON audit_logs(organization_id, created_at DESC);
```

```typescript
// Audit middleware
async function auditAction(actor: User, action: string, resource: { type: string; id: string }, changes?: { before?: object; after?: object }) {
  await db.insert(auditLogs).values({
    organizationId: actor.orgId,
    actorId: actor.id,
    actorEmail: actor.email,
    action,
    resourceType: resource.type,
    resourceId: resource.id,
    beforeState: changes?.before ?? null,
    afterState: changes?.after ?? null,
    ipAddress: actor.ip,
    userAgent: actor.userAgent,
  })
}

// Usage
await auditAction(req.user, 'booking.cancelled', { type: 'booking', id: booking.id }, {
  before: { status: 'confirmed' },
  after: { status: 'cancelled', cancelledAt: new Date() }
})
```

---

## 9. Usage Metering

```typescript
// Track usage for billing and plan limit enforcement
async function trackUsage(orgId: string, metric: string, quantity: number = 1) {
  const now = new Date()
  const month = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`
  
  // Upsert monthly usage counter
  await db.execute(sql`
    INSERT INTO usage_metrics (organization_id, metric, month, quantity)
    VALUES (${orgId}, ${metric}, ${month}, ${quantity})
    ON CONFLICT (organization_id, metric, month)
    DO UPDATE SET quantity = usage_metrics.quantity + ${quantity}, updated_at = NOW()
  `)
  
  // Check plan limits
  const [usage] = await db.execute(sql`
    SELECT quantity FROM usage_metrics
    WHERE organization_id = ${orgId} AND metric = ${metric} AND month = ${month}
  `)
  
  const org = await getOrganization(orgId)
  
  if (metric === 'bookings' && usage.quantity > org.maxBookingsPerMonth) {
    throw new PlanLimitExceededError(`Monthly booking limit reached. Upgrade to create more bookings.`)
  }
}

// Call before creating resources
async function createBooking(input: CreateBookingInput) {
  await trackUsage(input.tenantId, 'bookings', 1)
  // ... create booking
}
```

---

## 10. SaaS Security Checklist

```
Authentication:
  [ ] MFA available for all plans, enforced for Enterprise/Admin roles
  [ ] SSO (SAML/OIDC) available for Enterprise plan
  [ ] Session timeout (15 min idle for admin, 8h for users)
  [ ] Secure password reset (time-limited tokens, invalidate on use)
  [ ] Rate limit login attempts (5 attempts → lockout)

Authorization:
  [ ] Tenant isolation enforced at DB level (RLS or schema-per-tenant)
  [ ] RBAC enforced at API layer (not just UI)
  [ ] Audit log for all sensitive operations
  [ ] API keys scoped to specific permissions
  [ ] JWT audience claim validates tenant context

Data:
  [ ] Data encrypted at rest (AES-256)
  [ ] Data encrypted in transit (TLS 1.2+)
  [ ] Customer data never in logs (use IDs, not names/emails)
  [ ] GDPR/LGPD compliant: data export, erasure, DPA
  [ ] Backup encryption and restore tested

Multi-tenancy:
  [ ] Impossible to list other tenants' resources (test with OWASP IDOR)
  [ ] Slugs/subdomains validated against org ownership
  [ ] Stripe webhooks verified (no spoofed subscription upgrades)
  [ ] Billing limits enforced server-side (not just client-side)

Operations:
  [ ] Dependency scanning (Snyk/Dependabot)
  [ ] DAST scan against staging environment
  [ ] SOC 2 Type II (or equivalent) on roadmap
  [ ] Penetration test annually
```
