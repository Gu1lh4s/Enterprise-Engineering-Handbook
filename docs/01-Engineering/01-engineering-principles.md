# Engineering Principles

> **Category:** Software Engineering
> **Version:** 1.0.0
> **Level:** All Engineers → Staff Engineer

---

## Table of Contents

1. [SOLID Principles](#1-solid-principles)
2. [Clean Code](#2-clean-code)
3. [Domain-Driven Design (DDD)](#3-domain-driven-design-ddd)
4. [Clean Architecture](#4-clean-architecture)
5. [YAGNI, DRY, KISS](#5-yagni-dry-kiss)
6. [Design Patterns](#6-design-patterns)
7. [Functional Programming Concepts](#7-functional-programming-concepts)
8. [Error Handling Philosophy](#8-error-handling-philosophy)
9. [Code Review Philosophy](#9-code-review-philosophy)
10. [Technical Debt Management](#10-technical-debt-management)

---

## 1. SOLID Principles

### S — Single Responsibility Principle

```
A class/module should have ONE reason to change.
"Reason to change" = one actor whose requirements it serves.

Not: "do one thing" (too vague)
Yes: "serve one stakeholder group" (actionable)
```

```typescript
// BAD: UserService serves multiple actors (business rules + persistence + email)
class UserService {
  async register(email: string, password: string) {
    const hashedPassword = await bcrypt.hash(password, 12)         // Persistence concern
    const user = await this.db.insert({ email, hashedPassword })   // Persistence concern
    await this.sendWelcomeEmail(user.email)                        // Notification concern
    await this.updateMarketingList(user.email)                     // Marketing concern
    return user
  }
}

// GOOD: Each concern in its own module
class UserRegistrationService {         // Business rule: orchestrates registration
  constructor(
    private userRepo: UserRepository,   // Persistence concern
    private emailService: EmailService, // Notification concern
    private eventBus: EventBus,         // Async decoupling
  ) {}
  
  async register(email: string, password: string): Promise<User> {
    const hashedPassword = await bcrypt.hash(password, 12)
    const user = await this.userRepo.create({ email, hashedPassword })
    
    // Publish event — let other concerns react independently
    await this.eventBus.publish(new UserRegisteredEvent(user.id, user.email))
    
    return user
  }
}

class UserRegisteredEmailHandler {     // Notification concern
  async handle(event: UserRegisteredEvent) {
    await this.emailService.sendWelcomeEmail(event.email)
  }
}
```

### O — Open/Closed Principle

```
Open for extension, closed for modification.
Add new behavior by adding new code, not changing existing code.
```

```typescript
// BAD: Adding new payment method requires modifying existing class
class PaymentProcessor {
  async process(method: string, amount: number) {
    if (method === 'stripe') {
      return await stripe.charge(amount)
    } else if (method === 'paypal') {    // Modification every time
      return await paypal.pay(amount)
    } else if (method === 'pix') {       // Another modification
      return await pix.transfer(amount)
    }
  }
}

// GOOD: Add new method by adding new class
interface PaymentGateway {
  charge(amount: number, currency: string): Promise<PaymentResult>
}

class StripeGateway implements PaymentGateway {
  async charge(amount: number, currency: string) { /* ... */ }
}

class PayPalGateway implements PaymentGateway {
  async charge(amount: number, currency: string) { /* ... */ }
}

class PixGateway implements PaymentGateway {   // New: no modification of existing code
  async charge(amount: number, currency: string) { /* ... */ }
}

class PaymentProcessor {
  constructor(private gateway: PaymentGateway) {}  // Open for extension via DI
  
  async process(amount: number, currency: string) {
    return this.gateway.charge(amount, currency)
  }
}
```

### L — Liskov Substitution Principle

```
Subtypes must be substitutable for their base types.
If S extends T, code using T must work correctly with S.

Violations: subclass throws where parent doesn't, subclass narrows preconditions,
subclass weakens postconditions.
```

```typescript
// BAD: Square violates LSP when extending Rectangle
class Rectangle {
  constructor(public width: number, public height: number) {}
  
  setWidth(w: number) { this.width = w }
  setHeight(h: number) { this.height = h }
  area() { return this.width * this.height }
}

class Square extends Rectangle {
  setWidth(w: number) { 
    this.width = w
    this.height = w   // Side effect not in parent contract
  }
  setHeight(h: number) {
    this.width = h    // Violates: caller expects only height to change
    this.height = h
  }
}

// Code that works for Rectangle breaks for Square:
function doubleWidth(rect: Rectangle) {
  const originalHeight = rect.height
  rect.setWidth(rect.width * 2)
  // Assumes: height unchanged. Square breaks this assumption.
  console.assert(rect.height === originalHeight)  // FAILS for Square
}

// GOOD: Model correctly (no inheritance, or abstract shape)
interface Shape { area(): number }
class Rectangle implements Shape { area() { return this.width * this.height } }
class Square implements Shape { area() { return this.side ** 2 } }
```

### I — Interface Segregation Principle

```
Clients should not be forced to depend on interfaces they don't use.
Split fat interfaces into smaller, focused ones.
```

```typescript
// BAD: UserRepository forces all implementors to implement everything
interface UserRepository {
  findById(id: string): Promise<User>
  findAll(): Promise<User[]>
  create(user: User): Promise<User>
  update(id: string, data: Partial<User>): Promise<User>
  delete(id: string): Promise<void>
  findByEmail(email: string): Promise<User | null>
  exportToCsv(): Promise<string>          // Not all repos need this
  sendMarketingEmail(id: string): void    // Not a repo concern at all
}

// GOOD: Segregated interfaces
interface UserReader {
  findById(id: string): Promise<User | null>
  findByEmail(email: string): Promise<User | null>
  findAll(filters?: UserFilters): Promise<User[]>
}

interface UserWriter {
  create(data: CreateUserInput): Promise<User>
  update(id: string, data: UpdateUserInput): Promise<User>
  delete(id: string): Promise<void>
}

// Read-only service only depends on what it needs
class UserQueryService {
  constructor(private reader: UserReader) {}
}

// Admin service gets write access
class UserAdminService {
  constructor(private reader: UserReader, private writer: UserWriter) {}
}
```

### D — Dependency Inversion Principle

```
High-level modules should not depend on low-level modules.
Both should depend on abstractions.
Abstractions should not depend on details — details depend on abstractions.
```

```typescript
// BAD: High-level business logic depends on low-level detail (PostgreSQL)
import { Pool } from 'pg'

class BookingService {
  private pool = new Pool({ connectionString: process.env.DATABASE_URL })  // Direct coupling
  
  async createBooking(input: CreateBookingInput) {
    const result = await this.pool.query(
      'INSERT INTO bookings (...) VALUES (...)',
      [input.clientId, input.serviceId]
    )
    return result.rows[0]
  }
}

// GOOD: Business logic depends on abstraction; concrete implementation injected
// 1. Define the abstraction (in domain layer)
interface BookingRepository {
  create(input: CreateBookingInput): Promise<Booking>
  findById(id: string): Promise<Booking | null>
  findByTenantId(tenantId: string): Promise<Booking[]>
}

// 2. High-level module uses abstraction
class BookingService {
  constructor(private repository: BookingRepository) {}   // Depends on abstraction
  
  async createBooking(input: CreateBookingInput): Promise<Booking> {
    // Business logic only — no SQL, no HTTP
    if (input.scheduledAt < new Date()) throw new ValidationError('Must be future date')
    return this.repository.create(input)
  }
}

// 3. Low-level detail implements abstraction (in infrastructure layer)
class PostgresBookingRepository implements BookingRepository {
  constructor(private db: Database) {}
  
  async create(input: CreateBookingInput): Promise<Booking> {
    const [booking] = await this.db.insert(bookings).values(input).returning()
    return booking
  }
  // ...
}

// 4. Wired together in composition root (app.ts)
const repository = new PostgresBookingRepository(db)
const service = new BookingService(repository)
```

---

## 2. Clean Code

### Naming

```typescript
// Names should reveal intent — read code like a sentence

// BAD
const d = new Date()
const u = await getU(id)
const list = await query(filter, 20)
function calc(x: number, y: number) { return x * 0.07 }

// GOOD
const registeredAt = new Date()
const user = await getUserById(id)
const recentBookings = await findBookings({ status: 'confirmed' }, { limit: 20 })
const TAX_RATE = 0.07
function calculateTax(subtotal: number) { return subtotal * TAX_RATE }

// Functions: verbs (actions)
// GOOD: createUser, sendEmail, validateInput, fetchBookings, isSlotAvailable
// BAD:  user, email, input, bookings, slotCheck

// Boolean: is/has/can/should
// GOOD: isAuthenticated, hasPermission, canCancelBooking, shouldSendReminder
// BAD:  authenticated, permission, cancel, reminder

// Avoid abbreviations except established conventions
// BAD:  usr, bkng, svc, err
// GOOD: user, booking, service, error
// OK:   id, db, ctx, req, res, err (established in Node.js ecosystem)
```

### Functions

```typescript
// Rules for functions:
// 1. Small (fits on one screen)
// 2. Do ONE thing
// 3. Same level of abstraction throughout
// 4. Max 3 parameters (use object if more needed)
// 5. No side effects in functions named as queries (Command-Query Separation)

// BAD: function does too many things at different levels of abstraction
async function handleBooking(req: Request) {
  // Validation (low level)
  if (!req.body.serviceId || typeof req.body.serviceId !== 'string') {
    return res.status(400).json({ error: 'Invalid serviceId' })
  }
  if (!req.body.scheduledAt) {
    return res.status(400).json({ error: 'Missing scheduledAt' })
  }
  
  // Business logic (high level)
  const isAvailable = await db.query(
    'SELECT COUNT(*) FROM bookings WHERE service_id = $1 AND scheduled_at = $2',
    [req.body.serviceId, req.body.scheduledAt]
  )
  
  if (parseInt(isAvailable.rows[0].count) > 0) {
    return res.status(409).json({ error: 'Slot taken' })
  }
  
  // Persistence (low level again)
  const booking = await db.query(...)
  
  // Email (different concern entirely)
  await fetch('https://api.sendgrid.com/...', { ... })
}

// GOOD: each function at consistent level of abstraction
async function handleCreateBooking(req: Request, res: Response) {
  const input = parseCreateBookingInput(req.body)   // Validation
  const booking = await bookingService.create(input, req.user)  // Business
  res.status(201).json({ data: booking })           // Response
}

async function createBooking(input: ValidatedInput, actor: User): Promise<Booking> {
  await ensureSlotAvailable(input.serviceId, input.scheduledAt)
  const booking = await bookingRepository.create({ ...input, clientId: actor.id })
  await notifications.sendBookingConfirmation(booking)
  return booking
}
```

### Comments

```typescript
// Write NO comments by default.
// Write a comment ONLY when the WHY is non-obvious.

// BAD: explains WHAT (code already says that)
// Loop through all users
for (const user of users) {
  // Send email to each user
  await sendEmail(user.email)
}

// BAD: translates code to English
// Check if count is greater than zero
if (count > 0) { ... }

// GOOD: explains WHY (non-obvious constraint)
// PCI-DSS 3.4: cardholder data must be truncated before logging
const truncatedPan = `${pan.slice(0, 6)}******${pan.slice(-4)}`

// GOOD: explains workaround for known bug
// Node.js 18 has a bug where AbortSignal.timeout() doesn't work with fetch
// in some environments. Using Promise.race() as a workaround.
const result = await Promise.race([
  fetch(url),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000))
])

// GOOD: explains non-obvious business rule
// Slots are locked 2h before to allow professional preparation time
const lockoutTime = new Date(scheduledAt.getTime() - 2 * 60 * 60 * 1000)
if (now > lockoutTime) throw new BookingLockedError()
```

---

## 3. Domain-Driven Design (DDD)

### Core Concepts

```
Ubiquitous Language:
  A shared vocabulary between developers and domain experts.
  The same words used in: code, tests, documentation, conversation.
  
  Example — Booking domain language:
    "appointment" vs "booking" vs "reservation" → pick ONE, use everywhere
    "cancellation" → has meaning (refund policy, notification rules)
    "slot" → time + service + professional
    "no-show" → distinct from "cancellation" (different fee)

Bounded Context:
  A semantic boundary where terms have specific meaning.
  "User" in auth context ≠ "User" in booking context ≠ "User" in billing context.
  
  Booking context:   Client (person who books), Professional (person who serves)
  Auth context:      Principal (authenticated identity)
  Billing context:   Customer (Stripe customer), Account
  
  Each context has its own models — don't share domain models across contexts.

Context Mapping (how bounded contexts relate):
  Partnership:     Two teams collaborate, align frequently
  Shared Kernel:   Shared library/model (high coupling, use carefully)
  Customer/Supplier: One team consumes another's API
  Anticorruption Layer: Translation layer to prevent upstream corruption of your model
  Open Host:       Provider publishes protocol for others to consume
```

### Building Blocks

```typescript
// Entity: has identity (ID), mutable state over time
class Booking {
  constructor(
    public readonly id: BookingId,       // Identity (immutable)
    private status: BookingStatus,       // State (mutable)
    public readonly clientId: UserId,
    public readonly serviceId: ServiceId,
    public readonly scheduledAt: Date,
  ) {}
  
  // Behavior encapsulated in entity (not in service)
  cancel(reason: string, cancelledBy: UserId): void {
    if (this.status === 'completed') throw new BookingAlreadyCompletedError()
    if (this.status === 'cancelled') throw new BookingAlreadyCancelledError()
    
    this.status = 'cancelled'
    this.addDomainEvent(new BookingCancelledEvent(this.id, cancelledBy, reason))
  }
  
  confirm(): void {
    if (this.status !== 'pending') throw new InvalidBookingStateError()
    this.status = 'confirmed'
    this.addDomainEvent(new BookingConfirmedEvent(this.id))
  }
  
  isOwnedBy(userId: UserId): boolean {
    return this.clientId === userId
  }
}

// Value Object: no identity, defined by its values, immutable
class Money {
  constructor(
    public readonly amount: number,   // In centavos (no float!)
    public readonly currency: 'BRL' | 'USD' | 'EUR',
  ) {
    if (amount < 0) throw new Error('Amount cannot be negative')
  }
  
  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch')
    return new Money(this.amount + other.amount, this.currency)
  }
  
  multiply(factor: number): Money {
    return new Money(Math.round(this.amount * factor), this.currency)
  }
  
  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency
  }
  
  // Value objects are equal by value, not reference
  toString() { return `${(this.amount / 100).toFixed(2)} ${this.currency}` }
}

// Aggregate: cluster of entities/VOs with consistency boundary
// ONE aggregate root controls all access to the cluster
class Order {  // Aggregate root
  private items: OrderItem[] = []
  
  addItem(product: Product, quantity: number): void {
    const existing = this.items.find(i => i.productId === product.id)
    if (existing) {
      existing.increaseQuantity(quantity)
    } else {
      this.items.push(new OrderItem(product.id, product.price, quantity))
    }
    // Enforce invariant: total cannot exceed budget
    if (this.total().amount > this.budget.amount) {
      throw new BudgetExceededError()
    }
  }
  
  total(): Money {
    return this.items.reduce(
      (sum, item) => sum.add(item.subtotal()),
      new Money(0, 'BRL')
    )
  }
}

// Repository: collection-like abstraction for aggregates
interface BookingRepository {
  findById(id: BookingId): Promise<Booking | null>
  save(booking: Booking): Promise<void>
  findByClientId(clientId: UserId): Promise<Booking[]>
}

// Domain Service: operations that don't naturally belong to an entity
class SlotAvailabilityService {
  async isAvailable(serviceId: ServiceId, at: Date): Promise<boolean> {
    // Cross-entity logic: check bookings + blocked times + professional schedule
    const [bookingCount, isBlocked] = await Promise.all([
      this.bookingRepo.countBySlot(serviceId, at),
      this.blockedTimeRepo.isBlocked(serviceId, at),
    ])
    return bookingCount === 0 && !isBlocked
  }
}
```

### Domain Events

```typescript
// Domain events: something that happened in the domain
// Published by aggregate, handled by other bounded contexts

abstract class DomainEvent {
  public readonly occurredAt: Date = new Date()
  public readonly eventId: string = randomUUID()
  abstract readonly eventType: string
}

class BookingConfirmedEvent extends DomainEvent {
  readonly eventType = 'booking.confirmed'
  constructor(
    public readonly bookingId: string,
    public readonly clientId: string,
    public readonly serviceId: string,
    public readonly scheduledAt: Date,
  ) { super() }
}

// Handlers in other bounded contexts react to events
class NotificationContext {
  async onBookingConfirmed(event: BookingConfirmedEvent) {
    const [client, service] = await Promise.all([
      this.clientRepo.findById(event.clientId),
      this.serviceRepo.findById(event.serviceId),
    ])
    await this.emailService.sendConfirmation({ client, service, scheduledAt: event.scheduledAt })
    await this.smsService.sendConfirmation({ client, service, scheduledAt: event.scheduledAt })
  }
}
```

---

## 4. Clean Architecture

```
Dependency Rule: source code dependencies point INWARD only.
Inner layers know nothing about outer layers.

                    ┌─────────────────────────────────┐
                    │         Infrastructure           │  ← Outer (details)
                    │   (DB, HTTP, email, queues)      │
                    │  ┌────────────────────────────┐  │
                    │  │      Interface Adapters     │  │  ← Controllers, Repos, Presenters
                    │  │  ┌──────────────────────┐  │  │
                    │  │  │   Application Core   │  │  │  ← Use Cases
                    │  │  │  ┌────────────────┐  │  │  │
                    │  │  │  │    Domain      │  │  │  │  ← Inner (policy)
                    │  │  │  │ Entities, VOs  │  │  │  │
                    │  │  │  └────────────────┘  │  │  │
                    │  │  └──────────────────────┘  │  │
                    │  └────────────────────────────┘  │
                    └─────────────────────────────────┘

Flows inward (allowed): Infrastructure → Adapters → Application → Domain
Flows outward (forbidden): Domain → Application layer (never)
```

```typescript
// Domain layer — no imports from outer layers
// domain/booking.ts
export interface Booking {
  id: string
  clientId: string
  serviceId: string
  scheduledAt: Date
  status: 'pending' | 'confirmed' | 'cancelled' | 'completed'
}

export interface BookingRepository {
  findById(id: string): Promise<Booking | null>
  save(booking: Booking): Promise<void>
}

// Application layer — depends only on domain
// application/create-booking.usecase.ts
import type { BookingRepository } from '../domain/booking'

export class CreateBookingUseCase {
  constructor(
    private readonly bookingRepo: BookingRepository,  // Depends on abstraction
    private readonly notificationPort: NotificationPort,
  ) {}
  
  async execute(input: CreateBookingInput): Promise<Booking> {
    // Pure business logic
    const booking = createNewBooking(input)
    await this.bookingRepo.save(booking)
    await this.notificationPort.notifyBookingCreated(booking)
    return booking
  }
}

// Infrastructure layer — implements domain interfaces
// infrastructure/postgres-booking.repository.ts
import { BookingRepository } from '../../domain/booking'
import { db } from '../database'

export class PostgresBookingRepository implements BookingRepository {
  async findById(id: string) {
    return db.query.bookings.findFirst({ where: eq(bookings.id, id) })
  }
  
  async save(booking: Booking) {
    await db.insert(bookings).values(booking).onConflictDoUpdate({ ... })
  }
}

// Interface adapter (controller) — translates HTTP to use case
// adapters/http/bookings.controller.ts
export class BookingsController {
  constructor(private useCase: CreateBookingUseCase) {}
  
  async create(req: Request, res: Response) {
    const booking = await this.useCase.execute({
      clientId: req.user.id,
      serviceId: req.body.serviceId,
      scheduledAt: new Date(req.body.scheduledAt),
    })
    res.status(201).json({ data: booking })
  }
}
```

---

## 5. YAGNI, DRY, KISS

```
YAGNI — You Aren't Gonna Need It
  Don't build for imaginary future requirements.
  "We might need multi-currency later" → don't build it until you do.
  Cost: code that exists must be maintained, understood, tested.
  
  Signs of YAGNI violation:
    - Abstract base classes with one concrete subclass
    - Configuration options no one uses
    - "Plugin architecture" for a monolith
    - Generic framework built before the second use case exists

DRY — Don't Repeat Yourself
  Not just "no duplicate code" — it's "no duplicate knowledge".
  The same business rule in two places = two places to update when it changes.
  
  DRY violation example:
    Booking validation: in API controller AND in background job AND in cron
    → One change to rule needs 3 code changes
  
  DRY does NOT mean: extract any 2 similar-looking code snippets
    Three similar lines ≠ violated DRY (may be coincidentally similar)
    Wait until you understand the pattern before abstracting

KISS — Keep It Simple, Stupid
  Simple code is code that's easy to understand, change, and debug at 3am.
  
  Complexity budget: every feature has limited complexity you can add.
    Spend it on: solving the actual user problem
    Don't spend it on: clever abstractions, premature optimization
  
  Signs of unnecessary complexity:
    - Multiple levels of indirection for a simple operation
    - Generic solution for a specific problem
    - Configuration where hardcoding would be simpler
    - "Strategy pattern" for one strategy
```

---

## 6. Design Patterns

### Creational

```typescript
// Factory: create objects without exposing creation logic
class NotificationFactory {
  static create(type: 'email' | 'sms' | 'push', config: Config): Notification {
    switch (type) {
      case 'email': return new EmailNotification(config.smtp)
      case 'sms':   return new SmsNotification(config.twilio)
      case 'push':  return new PushNotification(config.fcm)
    }
  }
}

// Builder: construct complex objects step by step
class QueryBuilder {
  private conditions: string[] = []
  private orderBy?: string
  private limit?: number
  
  where(condition: string) { this.conditions.push(condition); return this }
  order(field: string) { this.orderBy = field; return this }
  take(n: number) { this.limit = n; return this }
  
  build(): string {
    let query = 'SELECT * FROM bookings'
    if (this.conditions.length) query += ` WHERE ${this.conditions.join(' AND ')}`
    if (this.orderBy) query += ` ORDER BY ${this.orderBy}`
    if (this.limit) query += ` LIMIT ${this.limit}`
    return query
  }
}

const query = new QueryBuilder()
  .where("status = 'pending'")
  .where("scheduled_at > NOW()")
  .order('scheduled_at ASC')
  .take(20)
  .build()
```

### Structural

```typescript
// Adapter: bridge incompatible interfaces
// Wraps legacy/external system to match your interface
class LegacyPaymentGatewayAdapter implements PaymentGateway {
  constructor(private legacy: LegacyPaymentSystem) {}
  
  async charge(amount: number, currency: string): Promise<PaymentResult> {
    // Translate: your interface → legacy interface
    const legacyResult = await this.legacy.processPayment({
      amount_cents: amount,
      curr: currency.toLowerCase(),
      type: 'CHARGE',
    })
    
    // Translate: legacy response → your interface
    return {
      success: legacyResult.status_code === '00',
      transactionId: legacyResult.txn_ref,
      amount,
      currency,
    }
  }
}

// Decorator: add behavior without modifying original
function withRetry(fn: () => Promise<any>, retries = 3) {
  return async (...args: any[]) => {
    for (let i = 0; i < retries; i++) {
      try { return await fn(...args) }
      catch (err) { if (i === retries - 1) throw err }
    }
  }
}

function withCache(fn: (id: string) => Promise<any>, ttl: number) {
  const cache = new Map<string, { value: any; expiresAt: number }>()
  
  return async (id: string) => {
    const cached = cache.get(id)
    if (cached && cached.expiresAt > Date.now()) return cached.value
    
    const value = await fn(id)
    cache.set(id, { value, expiresAt: Date.now() + ttl })
    return value
  }
}
```

### Behavioral

```typescript
// Observer / Event Bus: decouple producers from consumers
class EventBus {
  private handlers = new Map<string, Set<Function>>()
  
  on(event: string, handler: Function) {
    if (!this.handlers.has(event)) this.handlers.set(event, new Set())
    this.handlers.get(event)!.add(handler)
    return () => this.handlers.get(event)?.delete(handler)  // Returns unsubscribe
  }
  
  async emit(event: string, payload: unknown) {
    const handlers = this.handlers.get(event) ?? []
    await Promise.allSettled([...handlers].map(h => h(payload)))
  }
}

// Strategy: swap algorithms at runtime
interface PricingStrategy {
  calculatePrice(service: Service, date: Date): Money
}

class StandardPricing implements PricingStrategy {
  calculatePrice(service: Service) { return service.basePrice }
}

class WeekendPricing implements PricingStrategy {
  calculatePrice(service: Service, date: Date) {
    const isWeekend = [0, 6].includes(date.getDay())
    return isWeekend ? service.basePrice.multiply(1.3) : service.basePrice
  }
}

class DynamicPricing implements PricingStrategy {
  async calculatePrice(service: Service, date: Date) {
    const demand = await this.getDemandScore(service.id, date)
    return service.basePrice.multiply(1 + demand * 0.5)
  }
}

class BookingPriceCalculator {
  constructor(private strategy: PricingStrategy) {}
  
  setStrategy(strategy: PricingStrategy) { this.strategy = strategy }
  calculate(service: Service, date: Date) { return this.strategy.calculatePrice(service, date) }
}
```

---

## 7. Functional Programming Concepts

```typescript
// Pure functions: same input → same output, no side effects
// Good for: data transformation, calculations, validation

// Impure (side effects)
let totalBookings = 0
function countBooking() { totalBookings++ }  // Modifies external state

// Pure
function count(bookings: Booking[]): number { return bookings.length }

// Immutability: don't mutate, create new
// Bad
function updateStatus(booking: Booking, status: string) {
  booking.status = status   // Mutation — changes original
  return booking
}

// Good
function updateStatus(booking: Booking, status: string): Booking {
  return { ...booking, status }  // New object — original unchanged
}

// Composition: build complex functions from simple ones
const pipe = <T>(...fns: Array<(x: T) => T>) => (x: T) => fns.reduce((v, f) => f(v), x)

const processBooking = pipe(
  validateBooking,
  enrichWithServiceData,
  calculatePrice,
  applyDiscount,
)

// Option/Maybe: represent absence without null
type Option<T> = { some: true; value: T } | { some: false }

function findBooking(id: string): Option<Booking> {
  const booking = db.find(id)
  return booking ? { some: true, value: booking } : { some: false }
}

function getBookingPrice(id: string): Option<Money> {
  const result = findBooking(id)
  if (!result.some) return { some: false }
  return { some: true, value: calculatePrice(result.value) }
}
```

---

## 8. Error Handling Philosophy

```typescript
// Two types of errors:
// 1. Operational: expected failures (invalid input, not found, conflict)
//    → Handle gracefully, return to user, log at WARN level
// 2. Programmer: unexpected bugs (null reference, type error, assertion)
//    → Crash the process, let process manager restart, log at ERROR/FATAL

// Operational errors: use typed error classes
class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} ${id} not found`, 404)
  }
}

class ValidationError extends AppError {
  constructor(message: string, public fields: Record<string, string>) {
    super(message, 422)
  }
}

class ConflictError extends AppError {
  constructor(message: string) { super(message, 409) }
}

// Don't throw for expected failures in internal code
// Use Result type instead:
type Result<T, E = AppError> =
  | { ok: true; value: T }
  | { ok: false; error: E }

async function findUser(id: string): Promise<Result<User, NotFoundError>> {
  const user = await db.query.users.findFirst({ where: eq(users.id, id) })
  if (!user) return { ok: false, error: new NotFoundError('User', id) }
  return { ok: true, value: user }
}

// Caller handles both cases explicitly
const result = await findUser(id)
if (!result.ok) {
  return res.status(404).json({ error: result.error.message })
}
const user = result.value  // TypeScript knows this is User here
```

---

## 9. Code Review Philosophy

```
What code review is for:
  ✓ Catching bugs and security issues (most valuable)
  ✓ Sharing knowledge (reviewer learns the change; author gets fresh eyes)
  ✓ Maintaining consistency (patterns, naming, structure)
  ✗ Enforcing personal style preferences (use linters/formatters instead)
  ✗ Redesigning the whole approach (discuss before PR, not after)
  ✗ Slowing down shipping (if it's slow, the process is broken)

Reviewer mindset:
  Be kind. Be specific. Be constructive.
  
  BAD: "This is wrong"
  GOOD: "This will fail if serviceId is null — consider adding a null check here
         because [reason], or use the optional chaining approach: service?.id"
  
  Separate blocking from non-blocking:
  "Must fix (blocks merge): security issue"
  "Should fix: performance concern, easy to address"
  "Nit/suggestion: could be improved, but not blocking"
  
  Acknowledge good work:
  "Nice — using allSettled here means one failure doesn't kill the rest. Good call."

Author mindset:
  PR description = why this change, not what changed (git diff shows what)
  Small PRs: < 400 lines of logic changes (large PRs get rubber-stamped)
  Self-review before requesting: read your own diff as if someone else wrote it
  Don't get defensive: every comment is a learning opportunity or a clarification needed
```

---

## 10. Technical Debt Management

```
Definition: technical debt = cost of rework caused by choosing fast over right.
Like financial debt: sometimes intentional (startup speed), sometimes accidental.

Types:
  Deliberate (intentional): "We'll hardcode this for now, ticket to fix it"
  Inadvertent: "We didn't know the right way when we wrote it"
  Reckless: "We don't have time for tests" (always wrong)
  Prudent: "We need to ship, but we know this is suboptimal" (sometimes correct)

Identifying debt:
  → Where are the parts of the code everyone is afraid to touch?
  → What takes 3x longer to change than it should?
  → Where do bugs keep coming back?
  → What requires the most explanation to new engineers?

Managing debt:
  Track it: put it in your backlog with impact estimate
  Prioritize by pain: fix debt when it slows you down most
  "Boy Scout Rule": leave code cleaner than you found it (small improvements each PR)
  Scheduled debt sprints: dedicate 20% of capacity per quarter to debt reduction
  Don't accumulate deliberately unless there's a clear plan to repay

When to incur intentional debt:
  ✓ Proving a market hypothesis (MVP)
  ✓ Under a hard regulatory or customer deadline
  ✓ Exploration code (thrown away after learning)
  
  Always: write the ticket, estimate the repayment cost, get agreement from the team.
  If no one will write the ticket: don't take the debt.
```
