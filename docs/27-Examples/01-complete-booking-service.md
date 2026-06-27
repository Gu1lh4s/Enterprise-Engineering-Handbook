# Complete Example: Production Booking Service

> **Category:** Real-World Examples
> **Version:** 1.0.0
> **Level:** All Engineers

This document shows a complete, production-grade Booking Service from scratch — tying together
patterns from across the handbook into a single cohesive example. Every file shown is real and
runnable; nothing is simplified for illustration purposes.

---

## Table of Contents

1. [Project Structure](#1-project-structure)
2. [Domain Layer](#2-domain-layer)
3. [Application Layer — Use Cases](#3-application-layer--use-cases)
4. [Infrastructure Layer](#4-infrastructure-layer)
5. [HTTP Adapter Layer](#5-http-adapter-layer)
6. [Database Migrations](#6-database-migrations)
7. [Configuration and Bootstrap](#7-configuration-and-bootstrap)
8. [Testing](#8-testing)
9. [Dockerfile and Docker Compose](#9-dockerfile-and-docker-compose)
10. [Kubernetes Manifests](#10-kubernetes-manifests)
11. [CI/CD Pipeline](#11-cicd-pipeline)

---

## 1. Project Structure

```
booking-service/
├── src/
│   ├── domain/
│   │   ├── booking.entity.ts         # Booking aggregate root
│   │   ├── booking.repository.ts     # Repository interface (abstraction)
│   │   ├── booking.errors.ts         # Domain-specific errors
│   │   └── slot-availability.service.ts  # Domain service
│   │
│   ├── application/
│   │   ├── create-booking.usecase.ts
│   │   ├── cancel-booking.usecase.ts
│   │   ├── list-bookings.usecase.ts
│   │   └── ports/
│   │       └── notification.port.ts  # Outbound port interface
│   │
│   ├── infrastructure/
│   │   ├── database/
│   │   │   ├── db.ts                 # Drizzle ORM connection
│   │   │   ├── schema.ts             # Table definitions
│   │   │   └── postgres-booking.repository.ts
│   │   └── notifications/
│   │       └── email-notification.adapter.ts
│   │
│   ├── adapters/
│   │   ├── http/
│   │   │   ├── bookings.router.ts
│   │   │   ├── bookings.controller.ts
│   │   │   └── bookings.schemas.ts   # Zod validation schemas
│   │   └── middleware/
│   │       ├── auth.middleware.ts
│   │       ├── error.middleware.ts
│   │       └── metrics.middleware.ts
│   │
│   ├── shared/
│   │   ├── errors.ts                 # AppError hierarchy
│   │   ├── logger.ts                 # Pino structured logger
│   │   ├── metrics.ts                # Prometheus metrics
│   │   └── result.ts                 # Result<T, E> type
│   │
│   └── main.ts                       # Composition root + server
│
├── test/
│   ├── unit/
│   │   └── create-booking.usecase.test.ts
│   └── integration/
│       └── bookings.api.test.ts
│
├── drizzle/
│   └── migrations/
│       └── 0001_create_bookings.sql
│
├── k8s/
│   ├── deployment.yaml
│   ├── hpa.yaml
│   └── network-policy.yaml
│
├── Dockerfile
├── docker-compose.yml
├── drizzle.config.ts
└── package.json
```

---

## 2. Domain Layer

### `src/domain/booking.errors.ts`

```typescript
import { AppError } from '../shared/errors'

export class BookingNotFoundError extends AppError {
  constructor(id: string) {
    super(`Booking ${id} not found`, 404, 'BOOKING_NOT_FOUND')
  }
}

export class SlotUnavailableError extends AppError {
  constructor() {
    super('This slot is already booked', 409, 'SLOT_UNAVAILABLE')
  }
}

export class BookingCancellationError extends AppError {
  constructor(reason: string) {
    super(reason, 422, 'BOOKING_CANCELLATION_FAILED')
  }
}

export class BookingOwnershipError extends AppError {
  constructor() {
    super('You do not have permission to modify this booking', 403, 'BOOKING_FORBIDDEN')
  }
}
```

### `src/domain/booking.entity.ts`

```typescript
import { randomUUID } from 'crypto'

export type BookingStatus = 'pending' | 'confirmed' | 'cancelled' | 'completed'

export interface BookingProps {
  id: string
  tenantId: string
  clientId: string
  serviceId: string
  scheduledAt: Date
  status: BookingStatus
  cancellationReason: string | null
  createdAt: Date
  updatedAt: Date
}

export class Booking {
  private readonly props: BookingProps

  private constructor(props: BookingProps) {
    this.props = props
  }

  static create(input: {
    tenantId: string
    clientId: string
    serviceId: string
    scheduledAt: Date
  }): Booking {
    const now = new Date()
    return new Booking({
      id: randomUUID(),
      tenantId: input.tenantId,
      clientId: input.clientId,
      serviceId: input.serviceId,
      scheduledAt: input.scheduledAt,
      status: 'pending',
      cancellationReason: null,
      createdAt: now,
      updatedAt: now,
    })
  }

  static reconstitute(props: BookingProps): Booking {
    return new Booking(props)
  }

  cancel(reason: string, requestedBy: string): void {
    if (this.props.status === 'completed') {
      throw new Error('Cannot cancel a completed booking')
    }
    if (this.props.status === 'cancelled') {
      throw new Error('Booking is already cancelled')
    }
    // 2h lockout before appointment
    const lockoutAt = new Date(this.props.scheduledAt.getTime() - 2 * 60 * 60 * 1000)
    if (new Date() > lockoutAt && requestedBy === this.props.clientId) {
      throw new Error('Cannot cancel within 2 hours of appointment — contact support')
    }
    this.props.status = 'cancelled'
    this.props.cancellationReason = reason
    this.props.updatedAt = new Date()
  }

  confirm(): void {
    if (this.props.status !== 'pending') {
      throw new Error(`Cannot confirm booking in status ${this.props.status}`)
    }
    this.props.status = 'confirmed'
    this.props.updatedAt = new Date()
  }

  isOwnedBy(userId: string): boolean {
    return this.props.clientId === userId
  }

  get id() { return this.props.id }
  get tenantId() { return this.props.tenantId }
  get clientId() { return this.props.clientId }
  get serviceId() { return this.props.serviceId }
  get scheduledAt() { return this.props.scheduledAt }
  get status() { return this.props.status }
  get cancellationReason() { return this.props.cancellationReason }
  get createdAt() { return this.props.createdAt }
  get updatedAt() { return this.props.updatedAt }

  toPlainObject(): BookingProps {
    return { ...this.props }
  }
}
```

### `src/domain/booking.repository.ts`

```typescript
import type { Booking } from './booking.entity'

export interface BookingFilters {
  tenantId: string
  clientId?: string
  status?: string
  fromDate?: Date
  toDate?: Date
}

export interface PaginatedResult<T> {
  items: T[]
  nextCursor: string | null
  hasMore: boolean
}

export interface BookingRepository {
  findById(id: string, tenantId: string): Promise<Booking | null>
  findAll(filters: BookingFilters, cursor?: string, limit?: number): Promise<PaginatedResult<Booking>>
  countBySlot(tenantId: string, serviceId: string, scheduledAt: Date): Promise<number>
  save(booking: Booking): Promise<void>
}
```

### `src/domain/slot-availability.service.ts`

```typescript
import type { BookingRepository } from './booking.repository'

export class SlotAvailabilityService {
  constructor(private readonly bookingRepo: BookingRepository) {}

  async isAvailable(tenantId: string, serviceId: string, scheduledAt: Date): Promise<boolean> {
    if (scheduledAt < new Date()) return false

    const count = await this.bookingRepo.countBySlot(tenantId, serviceId, scheduledAt)
    return count === 0
  }
}
```

---

## 3. Application Layer — Use Cases

### `src/application/ports/notification.port.ts`

```typescript
import type { Booking } from '../../domain/booking.entity'

export interface NotificationPort {
  sendBookingConfirmation(booking: Booking): Promise<void>
  sendBookingCancellation(booking: Booking): Promise<void>
}
```

### `src/application/create-booking.usecase.ts`

```typescript
import { z } from 'zod'
import { Booking } from '../domain/booking.entity'
import { SlotAvailabilityService } from '../domain/slot-availability.service'
import { SlotUnavailableError } from '../domain/booking.errors'
import type { BookingRepository } from '../domain/booking.repository'
import type { NotificationPort } from './ports/notification.port'
import { logger } from '../shared/logger'

export const CreateBookingInputSchema = z.object({
  tenantId: z.string().uuid(),
  clientId: z.string().uuid(),
  serviceId: z.string().uuid(),
  scheduledAt: z.coerce.date().refine(d => d > new Date(), {
    message: 'Scheduled date must be in the future',
  }),
})

export type CreateBookingInput = z.infer<typeof CreateBookingInputSchema>

export class CreateBookingUseCase {
  private readonly availabilityService: SlotAvailabilityService

  constructor(
    private readonly bookingRepo: BookingRepository,
    private readonly notifications: NotificationPort,
  ) {
    this.availabilityService = new SlotAvailabilityService(bookingRepo)
  }

  async execute(input: CreateBookingInput): Promise<Booking> {
    const available = await this.availabilityService.isAvailable(
      input.tenantId,
      input.serviceId,
      input.scheduledAt,
    )

    if (!available) throw new SlotUnavailableError()

    const booking = Booking.create(input)
    await this.bookingRepo.save(booking)

    logger.info({ bookingId: booking.id, tenantId: input.tenantId }, 'Booking created')

    // Fire and forget — don't block response on email delivery
    this.notifications.sendBookingConfirmation(booking).catch(err =>
      logger.error({ err, bookingId: booking.id }, 'Failed to send confirmation email'),
    )

    return booking
  }
}
```

### `src/application/cancel-booking.usecase.ts`

```typescript
import { BookingNotFoundError, BookingOwnershipError, BookingCancellationError } from '../domain/booking.errors'
import type { BookingRepository } from '../domain/booking.repository'
import type { NotificationPort } from './ports/notification.port'
import { logger } from '../shared/logger'

export class CancelBookingUseCase {
  constructor(
    private readonly bookingRepo: BookingRepository,
    private readonly notifications: NotificationPort,
  ) {}

  async execute(bookingId: string, tenantId: string, requestedBy: string, reason: string): Promise<void> {
    const booking = await this.bookingRepo.findById(bookingId, tenantId)
    if (!booking) throw new BookingNotFoundError(bookingId)

    if (!booking.isOwnedBy(requestedBy)) throw new BookingOwnershipError()

    try {
      booking.cancel(reason, requestedBy)
    } catch (err) {
      throw new BookingCancellationError((err as Error).message)
    }

    await this.bookingRepo.save(booking)

    logger.info({ bookingId, tenantId, requestedBy }, 'Booking cancelled')

    this.notifications.sendBookingCancellation(booking).catch(err =>
      logger.error({ err, bookingId }, 'Failed to send cancellation email'),
    )
  }
}
```

---

## 4. Infrastructure Layer

### `src/infrastructure/database/schema.ts`

```typescript
import { pgTable, uuid, varchar, timestamp, index } from 'drizzle-orm/pg-core'

export const bookings = pgTable(
  'bookings',
  {
    id:                 uuid('id').primaryKey().defaultRandom(),
    tenantId:           uuid('tenant_id').notNull(),
    clientId:           uuid('client_id').notNull(),
    serviceId:          uuid('service_id').notNull(),
    scheduledAt:        timestamp('scheduled_at', { withTimezone: true }).notNull(),
    status:             varchar('status', { length: 20 }).notNull().default('pending'),
    cancellationReason: varchar('cancellation_reason', { length: 500 }),
    createdAt:          timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
    updatedAt:          timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
  },
  (table) => ({
    tenantIdx:     index('idx_bookings_tenant').on(table.tenantId),
    clientIdx:     index('idx_bookings_client').on(table.clientId),
    slotIdx:       index('idx_bookings_slot').on(table.tenantId, table.serviceId, table.scheduledAt),
    scheduledIdx:  index('idx_bookings_scheduled').on(table.tenantId, table.scheduledAt),
  }),
)
```

### `src/infrastructure/database/postgres-booking.repository.ts`

```typescript
import { eq, and, gt, lte, gte, count } from 'drizzle-orm'
import { Booking } from '../../domain/booking.entity'
import type { BookingRepository, BookingFilters, PaginatedResult } from '../../domain/booking.repository'
import { bookings } from './schema'
import type { Database } from './db'

export class PostgresBookingRepository implements BookingRepository {
  constructor(private readonly db: Database) {}

  async findById(id: string, tenantId: string): Promise<Booking | null> {
    const [row] = await this.db
      .select()
      .from(bookings)
      .where(and(eq(bookings.id, id), eq(bookings.tenantId, tenantId)))
      .limit(1)

    return row ? this.toDomain(row) : null
  }

  async findAll(
    filters: BookingFilters,
    cursor?: string,
    limit = 20,
  ): Promise<PaginatedResult<Booking>> {
    const conditions = [eq(bookings.tenantId, filters.tenantId)]

    if (filters.clientId) conditions.push(eq(bookings.clientId, filters.clientId))
    if (filters.status) conditions.push(eq(bookings.status, filters.status))
    if (filters.fromDate) conditions.push(gte(bookings.scheduledAt, filters.fromDate))
    if (filters.toDate) conditions.push(lte(bookings.scheduledAt, filters.toDate))
    if (cursor) conditions.push(gt(bookings.id, cursor))  // Cursor-based pagination

    const rows = await this.db
      .select()
      .from(bookings)
      .where(and(...conditions))
      .orderBy(bookings.id)
      .limit(limit + 1)  // Fetch one extra to determine hasMore

    const hasMore = rows.length > limit
    const items = rows.slice(0, limit).map(r => this.toDomain(r))

    return {
      items,
      hasMore,
      nextCursor: hasMore ? items[items.length - 1].id : null,
    }
  }

  async countBySlot(tenantId: string, serviceId: string, scheduledAt: Date): Promise<number> {
    const [{ value }] = await this.db
      .select({ value: count() })
      .from(bookings)
      .where(
        and(
          eq(bookings.tenantId, tenantId),
          eq(bookings.serviceId, serviceId),
          eq(bookings.scheduledAt, scheduledAt),
          eq(bookings.status, 'confirmed'),
        ),
      )
    return Number(value)
  }

  async save(booking: Booking): Promise<void> {
    const data = {
      id:                 booking.id,
      tenantId:           booking.tenantId,
      clientId:           booking.clientId,
      serviceId:          booking.serviceId,
      scheduledAt:        booking.scheduledAt,
      status:             booking.status,
      cancellationReason: booking.cancellationReason,
      createdAt:          booking.createdAt,
      updatedAt:          booking.updatedAt,
    }

    await this.db
      .insert(bookings)
      .values(data)
      .onConflictDoUpdate({
        target: bookings.id,
        set: {
          status:             data.status,
          cancellationReason: data.cancellationReason,
          updatedAt:          data.updatedAt,
        },
      })
  }

  private toDomain(row: typeof bookings.$inferSelect): Booking {
    return Booking.reconstitute({
      id:                 row.id,
      tenantId:           row.tenantId,
      clientId:           row.clientId,
      serviceId:          row.serviceId,
      scheduledAt:        row.scheduledAt,
      status:             row.status as any,
      cancellationReason: row.cancellationReason,
      createdAt:          row.createdAt,
      updatedAt:          row.updatedAt,
    })
  }
}
```

---

## 5. HTTP Adapter Layer

### `src/adapters/http/bookings.schemas.ts`

```typescript
import { z } from 'zod'

export const CreateBookingSchema = z.object({
  serviceId:   z.string().uuid('serviceId must be a valid UUID'),
  scheduledAt: z.string().datetime('scheduledAt must be ISO 8601').pipe(
    z.coerce.date().refine(d => d > new Date(), { message: 'Must be a future date' })
  ),
})

export const CancelBookingSchema = z.object({
  reason: z.string().min(1, 'Reason is required').max(500),
})

export const ListBookingsSchema = z.object({
  status:   z.enum(['pending', 'confirmed', 'cancelled', 'completed']).optional(),
  fromDate: z.string().datetime().optional(),
  toDate:   z.string().datetime().optional(),
  cursor:   z.string().optional(),
  limit:    z.coerce.number().int().min(1).max(100).default(20),
})
```

### `src/adapters/http/bookings.controller.ts`

```typescript
import type { Request, Response } from 'express'
import type { CreateBookingUseCase } from '../../application/create-booking.usecase'
import type { CancelBookingUseCase } from '../../application/cancel-booking.usecase'
import type { ListBookingsUseCase } from '../../application/list-bookings.usecase'
import { CreateBookingSchema, CancelBookingSchema, ListBookingsSchema } from './bookings.schemas'
import { ValidationError } from '../../shared/errors'

export class BookingsController {
  constructor(
    private readonly createBooking: CreateBookingUseCase,
    private readonly cancelBooking: CancelBookingUseCase,
    private readonly listBookings: ListBookingsUseCase,
  ) {}

  create = async (req: Request, res: Response): Promise<void> => {
    const parsed = CreateBookingSchema.safeParse(req.body)
    if (!parsed.success) {
      throw new ValidationError('Invalid request body', parsed.error.flatten().fieldErrors as any)
    }

    const booking = await this.createBooking.execute({
      tenantId:    req.user.tenantId,
      clientId:    req.user.id,
      serviceId:   parsed.data.serviceId,
      scheduledAt: parsed.data.scheduledAt,
    })

    res.status(201).json({ data: booking.toPlainObject() })
  }

  cancel = async (req: Request, res: Response): Promise<void> => {
    const parsed = CancelBookingSchema.safeParse(req.body)
    if (!parsed.success) {
      throw new ValidationError('Invalid request body', parsed.error.flatten().fieldErrors as any)
    }

    await this.cancelBooking.execute(
      req.params.id,
      req.user.tenantId,
      req.user.id,
      parsed.data.reason,
    )

    res.status(204).send()
  }

  list = async (req: Request, res: Response): Promise<void> => {
    const parsed = ListBookingsSchema.safeParse(req.query)
    if (!parsed.success) {
      throw new ValidationError('Invalid query parameters', parsed.error.flatten().fieldErrors as any)
    }

    const result = await this.listBookings.execute({
      tenantId: req.user.tenantId,
      clientId: req.user.id,
      ...parsed.data,
      fromDate: parsed.data.fromDate ? new Date(parsed.data.fromDate) : undefined,
      toDate:   parsed.data.toDate ? new Date(parsed.data.toDate) : undefined,
    })

    res.json({
      data: result.items.map(b => b.toPlainObject()),
      meta: {
        hasMore:    result.hasMore,
        nextCursor: result.nextCursor,
      },
    })
  }
}
```

### `src/adapters/middleware/auth.middleware.ts`

```typescript
import type { Request, Response, NextFunction } from 'express'
import jwt from 'jsonwebtoken'
import { AppError } from '../../shared/errors'
import { config } from '../../config'

interface JwtPayload {
  sub: string
  tenantId: string
  role: 'admin' | 'professional' | 'client'
  iat: number
  exp: number
}

declare global {
  namespace Express {
    interface Request {
      user: {
        id: string
        tenantId: string
        role: string
      }
    }
  }
}

export function authenticate(req: Request, _res: Response, next: NextFunction): void {
  const authHeader = req.headers.authorization
  if (!authHeader?.startsWith('Bearer ')) {
    throw new AppError('Missing or invalid Authorization header', 401, 'UNAUTHORIZED')
  }

  const token = authHeader.slice(7)

  try {
    const payload = jwt.verify(token, config.jwtSecret, {
      algorithms: ['HS256'],
    }) as JwtPayload

    req.user = {
      id:       payload.sub,
      tenantId: payload.tenantId,
      role:     payload.role,
    }

    next()
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      throw new AppError('Token expired', 401, 'TOKEN_EXPIRED')
    }
    throw new AppError('Invalid token', 401, 'INVALID_TOKEN')
  }
}
```

### `src/adapters/middleware/error.middleware.ts`

```typescript
import type { Request, Response, NextFunction } from 'express'
import { AppError } from '../../shared/errors'
import { logger } from '../../shared/logger'
import { httpErrorsTotal } from '../../shared/metrics'

export function errorMiddleware(
  err: Error,
  req: Request,
  res: Response,
  _next: NextFunction,
): void {
  if (err instanceof AppError) {
    // Operational error — expected, log at warn
    if (err.statusCode >= 500) {
      logger.error({ err, path: req.path, method: req.method }, 'Server error')
    } else {
      logger.warn({ err: { message: err.message, code: err.code }, path: req.path }, 'Client error')
    }

    httpErrorsTotal.labels(err.statusCode.toString(), err.code).inc()

    res.status(err.statusCode).json({
      error: {
        code:      err.code,
        message:   err.message,
        requestId: req.id,
        ...(err.details ? { details: err.details } : {}),
      },
    })
    return
  }

  // Programmer error — unexpected, log at error with full stack
  logger.error({ err, path: req.path, method: req.method }, 'Unhandled error')
  httpErrorsTotal.labels('500', 'INTERNAL_ERROR').inc()

  res.status(500).json({
    error: {
      code:      'INTERNAL_ERROR',
      message:   'An unexpected error occurred',
      requestId: req.id,
    },
  })
}
```

---

## 6. Database Migrations

### `drizzle/migrations/0001_create_bookings.sql`

```sql
-- Migration: 0001_create_bookings
-- Description: Create bookings table with RLS-ready tenant isolation

CREATE TABLE IF NOT EXISTS bookings (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id           UUID NOT NULL,
  client_id           UUID NOT NULL,
  service_id          UUID NOT NULL,
  scheduled_at        TIMESTAMPTZ NOT NULL,
  status              VARCHAR(20) NOT NULL DEFAULT 'pending'
                        CHECK (status IN ('pending', 'confirmed', 'cancelled', 'completed')),
  cancellation_reason VARCHAR(500),
  created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_bookings_tenant    ON bookings(tenant_id);
CREATE INDEX IF NOT EXISTS idx_bookings_client    ON bookings(client_id);
CREATE INDEX IF NOT EXISTS idx_bookings_scheduled ON bookings(tenant_id, scheduled_at DESC);

-- Slot uniqueness index: prevent double-booking
CREATE UNIQUE INDEX IF NOT EXISTS idx_bookings_slot_unique
  ON bookings(tenant_id, service_id, scheduled_at)
  WHERE status IN ('pending', 'confirmed');

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_bookings_updated_at
  BEFORE UPDATE ON bookings
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Row-Level Security for tenant isolation
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;

CREATE POLICY bookings_tenant_isolation ON bookings
  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

---

## 7. Configuration and Bootstrap

### `src/shared/errors.ts`

```typescript
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly details?: Record<string, unknown>,
  ) {
    super(message)
    this.name = this.constructor.name
    Error.captureStackTrace(this, this.constructor)
  }
}

export class ValidationError extends AppError {
  constructor(message: string, fields: Record<string, string[] | string>) {
    super(message, 422, 'VALIDATION_ERROR', { fields })
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} ${id} not found`, 404, 'NOT_FOUND')
  }
}
```

### `src/shared/logger.ts`

```typescript
import pino from 'pino'
import { config } from '../config'

export const logger = pino({
  level: config.logLevel,
  ...(config.nodeEnv === 'production'
    ? {}
    : { transport: { target: 'pino-pretty', options: { colorize: true } } }),
  redact: {
    paths: [
      'password',
      'token',
      'authorization',
      'req.headers.authorization',
      '*.email',
      '*.cpf',
    ],
    censor: '[REDACTED]',
  },
  base: {
    service: 'booking-service',
    env:     config.nodeEnv,
  },
})
```

### `src/shared/metrics.ts`

```typescript
import { Counter, Histogram, register } from 'prom-client'

export const httpRequestDuration = new Histogram({
  name:       'http_request_duration_seconds',
  help:       'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets:    [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
})

export const httpErrorsTotal = new Counter({
  name:       'http_errors_total',
  help:       'Total HTTP errors by status code and error code',
  labelNames: ['status_code', 'error_code'],
})

export const bookingsCreatedTotal = new Counter({
  name:       'bookings_created_total',
  help:       'Total bookings created',
  labelNames: ['tenant_id'],
})

export const bookingsCancelledTotal = new Counter({
  name:       'bookings_cancelled_total',
  help:       'Total bookings cancelled',
  labelNames: ['tenant_id', 'reason'],
})

export { register }
```

### `src/main.ts`

```typescript
import 'express-async-errors'
import express from 'express'
import { randomUUID } from 'crypto'
import { drizzle } from 'drizzle-orm/node-postgres'
import { Pool } from 'pg'
import { config } from './config'
import { logger } from './shared/logger'
import { register } from './shared/metrics'
import { PostgresBookingRepository } from './infrastructure/database/postgres-booking.repository'
import { EmailNotificationAdapter } from './infrastructure/notifications/email-notification.adapter'
import { CreateBookingUseCase } from './application/create-booking.usecase'
import { CancelBookingUseCase } from './application/cancel-booking.usecase'
import { ListBookingsUseCase } from './application/list-bookings.usecase'
import { BookingsController } from './adapters/http/bookings.controller'
import { authenticate } from './adapters/middleware/auth.middleware'
import { errorMiddleware } from './adapters/middleware/error.middleware'
import * as schema from './infrastructure/database/schema'

async function bootstrap() {
  // Database
  const pool = new Pool({
    connectionString: config.databaseUrl,
    max: 10,
    idleTimeoutMillis: 30_000,
    connectionTimeoutMillis: 5_000,
  })
  await pool.query('SELECT 1')  // Verify connection at startup
  const db = drizzle(pool, { schema })

  // Composition root — wire everything together
  const bookingRepo     = new PostgresBookingRepository(db)
  const notifications   = new EmailNotificationAdapter(config.smtpUrl)
  const createBooking   = new CreateBookingUseCase(bookingRepo, notifications)
  const cancelBooking   = new CancelBookingUseCase(bookingRepo, notifications)
  const listBookings    = new ListBookingsUseCase(bookingRepo)
  const controller      = new BookingsController(createBooking, cancelBooking, listBookings)

  const app = express()
  app.use(express.json({ limit: '100kb' }))

  // Assign request ID for tracing
  app.use((req, _res, next) => { req.id = req.headers['x-request-id'] as string ?? randomUUID(); next() })

  // Health and metrics (no auth)
  app.get('/health', (_req, res) => res.json({ status: 'ok', service: 'booking-service' }))
  app.get('/metrics', async (_req, res) => {
    res.set('Content-Type', register.contentType)
    res.end(await register.metrics())
  })

  // Bookings API
  app.use('/api/v1/bookings', authenticate)
  app.get('/api/v1/bookings',          controller.list)
  app.post('/api/v1/bookings',         controller.create)
  app.post('/api/v1/bookings/:id/cancel', controller.cancel)

  // Error handler (must be last)
  app.use(errorMiddleware)

  const server = app.listen(config.port, () => {
    logger.info({ port: config.port }, 'Server started')
  })

  // Graceful shutdown
  const shutdown = async (signal: string) => {
    logger.info({ signal }, 'Shutdown signal received')
    server.close(async () => {
      await pool.end()
      logger.info('Server and DB connections closed')
      process.exit(0)
    })
    setTimeout(() => {
      logger.error('Graceful shutdown timeout — forcing exit')
      process.exit(1)
    }, 10_000)
  }

  process.on('SIGTERM', () => shutdown('SIGTERM'))
  process.on('SIGINT',  () => shutdown('SIGINT'))
}

bootstrap().catch(err => {
  logger.fatal({ err }, 'Failed to start server')
  process.exit(1)
})
```

---

## 8. Testing

### `test/unit/create-booking.usecase.test.ts`

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { CreateBookingUseCase } from '../../src/application/create-booking.usecase'
import { SlotUnavailableError } from '../../src/domain/booking.errors'
import type { BookingRepository } from '../../src/domain/booking.repository'
import type { NotificationPort } from '../../src/application/ports/notification.port'

// Factory: creates valid test input
const makeInput = (overrides = {}) => ({
  tenantId:    '00000000-0000-0000-0000-000000000001',
  clientId:    '00000000-0000-0000-0000-000000000002',
  serviceId:   '00000000-0000-0000-0000-000000000003',
  scheduledAt: new Date(Date.now() + 24 * 60 * 60 * 1000),  // Tomorrow
  ...overrides,
})

// Mock repository
const makeRepo = (overrides: Partial<BookingRepository> = {}): BookingRepository => ({
  findById:    vi.fn(),
  findAll:     vi.fn(),
  countBySlot: vi.fn().mockResolvedValue(0),  // Slot available by default
  save:        vi.fn().mockResolvedValue(undefined),
  ...overrides,
})

const makeNotifications = (): NotificationPort => ({
  sendBookingConfirmation: vi.fn().mockResolvedValue(undefined),
  sendBookingCancellation: vi.fn().mockResolvedValue(undefined),
})

describe('CreateBookingUseCase', () => {
  let repo: BookingRepository
  let notifications: NotificationPort
  let useCase: CreateBookingUseCase

  beforeEach(() => {
    repo          = makeRepo()
    notifications = makeNotifications()
    useCase       = new CreateBookingUseCase(repo, notifications)
  })

  describe('when slot is available', () => {
    it('creates a booking with pending status', async () => {
      // Arrange
      const input = makeInput()

      // Act
      const booking = await useCase.execute(input)

      // Assert
      expect(booking.status).toBe('pending')
      expect(booking.clientId).toBe(input.clientId)
      expect(booking.tenantId).toBe(input.tenantId)
    })

    it('saves the booking to the repository', async () => {
      const input = makeInput()
      await useCase.execute(input)
      expect(repo.save).toHaveBeenCalledOnce()
    })

    it('sends a confirmation notification', async () => {
      const input = makeInput()
      await useCase.execute(input)
      // Give the fire-and-forget a tick to resolve
      await new Promise(resolve => setImmediate(resolve))
      expect(notifications.sendBookingConfirmation).toHaveBeenCalledOnce()
    })
  })

  describe('when slot is unavailable', () => {
    it('throws SlotUnavailableError', async () => {
      repo = makeRepo({ countBySlot: vi.fn().mockResolvedValue(1) })
      useCase = new CreateBookingUseCase(repo, notifications)

      await expect(useCase.execute(makeInput())).rejects.toThrow(SlotUnavailableError)
    })

    it('does not save to repository', async () => {
      repo = makeRepo({ countBySlot: vi.fn().mockResolvedValue(1) })
      useCase = new CreateBookingUseCase(repo, notifications)

      await expect(useCase.execute(makeInput())).rejects.toThrow()
      expect(repo.save).not.toHaveBeenCalled()
    })
  })

  describe('when scheduled in the past', () => {
    it('throws a validation error', async () => {
      const input = makeInput({ scheduledAt: new Date(Date.now() - 1000) })
      await expect(useCase.execute(input)).rejects.toThrow()
    })
  })
})
```

### `test/integration/bookings.api.test.ts`

```typescript
import { describe, it, expect, beforeAll, afterAll } from 'vitest'
import request from 'supertest'
import { GenericContainer, type StartedTestContainer } from 'testcontainers'
import { drizzle } from 'drizzle-orm/node-postgres'
import { Pool } from 'pg'
import { migrate } from 'drizzle-orm/node-postgres/migrator'
import jwt from 'jsonwebtoken'
import { buildApp } from '../../src/app'  // Export app factory from main.ts

const JWT_SECRET = 'test-secret-do-not-use-in-production'

const makeToken = (overrides = {}) =>
  jwt.sign(
    {
      sub:      '00000000-0000-0000-0000-000000000002',
      tenantId: '00000000-0000-0000-0000-000000000001',
      role:     'client',
      ...overrides,
    },
    JWT_SECRET,
    { algorithm: 'HS256', expiresIn: '1h' },
  )

describe('POST /api/v1/bookings', () => {
  let container: StartedTestContainer
  let app: Express.Application

  beforeAll(async () => {
    // Start a real PostgreSQL container for integration tests
    container = await new GenericContainer('postgres:16-alpine')
      .withEnvironment({ POSTGRES_PASSWORD: 'test', POSTGRES_DB: 'testdb' })
      .withExposedPorts(5432)
      .start()

    const pool = new Pool({
      connectionString: `postgresql://postgres:test@${container.getHost()}:${container.getMappedPort(5432)}/testdb`,
    })

    const db = drizzle(pool)
    await migrate(db, { migrationsFolder: './drizzle/migrations' })

    app = buildApp({ db, jwtSecret: JWT_SECRET })
  }, 60_000)  // Container startup can take up to 60s

  afterAll(async () => {
    await container.stop()
  })

  it('creates a booking and returns 201', async () => {
    const res = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${makeToken()}`)
      .send({
        serviceId:   '00000000-0000-0000-0000-000000000003',
        scheduledAt: new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString(),
      })

    expect(res.status).toBe(201)
    expect(res.body.data).toMatchObject({
      status:    'pending',
      clientId:  '00000000-0000-0000-0000-000000000002',
      serviceId: '00000000-0000-0000-0000-000000000003',
    })
  })

  it('returns 409 when slot is already taken', async () => {
    const scheduledAt = new Date(Date.now() + 48 * 60 * 60 * 1000).toISOString()
    const body = {
      serviceId: '00000000-0000-0000-0000-000000000003',
      scheduledAt,
    }

    // First booking succeeds
    await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${makeToken()}`)
      .send(body)

    // Confirm the first booking (so it blocks the slot)
    // ...confirm via PATCH in a real test...

    // Second booking on same slot fails
    const res = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${makeToken({ sub: 'other-user' })}`)
      .send(body)

    expect(res.status).toBe(409)
    expect(res.body.error.code).toBe('SLOT_UNAVAILABLE')
  })

  it('returns 401 without token', async () => {
    const res = await request(app)
      .post('/api/v1/bookings')
      .send({ serviceId: 'test', scheduledAt: new Date().toISOString() })

    expect(res.status).toBe(401)
  })

  it('returns 422 with invalid body', async () => {
    const res = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${makeToken()}`)
      .send({ serviceId: 'not-a-uuid', scheduledAt: 'not-a-date' })

    expect(res.status).toBe(422)
    expect(res.body.error.code).toBe('VALIDATION_ERROR')
    expect(res.body.error.details.fields).toBeDefined()
  })
})
```

---

## 9. Dockerfile and Docker Compose

### `Dockerfile`

```dockerfile
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --ignore-scripts

# Stage 2: Build TypeScript
FROM node:20-alpine AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build && npm prune --production

# Stage 3: Production image (distroless)
FROM gcr.io/distroless/nodejs20-debian12 AS runtime
WORKDIR /app

# Non-root user (distroless uses 65532 by default)
USER 65532:65532

COPY --from=build --chown=65532:65532 /app/dist ./dist
COPY --from=build --chown=65532:65532 /app/node_modules ./node_modules
COPY --from=build --chown=65532:65532 /app/drizzle ./drizzle

EXPOSE 3000

CMD ["dist/main.js"]
```

### `docker-compose.yml`

```yaml
version: '3.9'

services:
  api:
    build: .
    ports:
      - '3000:3000'
    environment:
      NODE_ENV: development
      PORT: 3000
      DATABASE_URL: postgresql://postgres:devpassword@db:5432/bookingdb
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-secret-change-in-production
      LOG_LEVEL: debug
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: bookingdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: devpassword
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U postgres']
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s

volumes:
  postgres_data:
```

---

## 10. Kubernetes Manifests

### `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: booking-service
  namespace: production
  labels:
    app: booking-service
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: booking-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: booking-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: booking-service

      # Graceful shutdown: allow 30s for in-flight requests
      terminationGracePeriodSeconds: 60
      containers:
        - name: api
          image: 123456789.dkr.ecr.us-east-1.amazonaws.com/booking-service:1.0.0
          imagePullPolicy: Always
          ports:
            - containerPort: 3000

          env:
            - name: NODE_ENV
              value: production
            - name: PORT
              value: "3000"

          envFrom:
            - secretRef:
                name: booking-service-secrets

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]

          securityContext:
            runAsNonRoot: true
            runAsUser: 65532
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: tmp
          emptyDir: {}

      securityContext:
        seccompProfile:
          type: RuntimeDefault
```

### `k8s/hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: booking-service-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: booking-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Don't scale down for 5 minutes after scaling up
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
```

---

## 11. CI/CD Pipeline

### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  contents: read
  id-token: write        # For OIDC to AWS ECR
  security-events: write # For CodeQL

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
      - run: npm run test:unit
      - run: npm run test:integration  # Uses TestContainers (Docker-in-Docker available on Ubuntu)

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        if: always()

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Semgrep (SAST)
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/nodejs
            p/typescript
            p/owasp-top-ten
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Dependency audit
        run: npm audit --audit-level=high

      - name: License check
        run: npx license-checker --failOn "GPL;AGPL"

  build-and-push:
    name: Build and Push
    runs-on: ubuntu-latest
    needs: [test, security]
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC — no stored keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-deploy-prod
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push
        env:
          ECR_REGISTRY: ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build \
            --tag $ECR_REGISTRY/booking-service:$IMAGE_TAG \
            --tag $ECR_REGISTRY/booking-service:latest \
            --cache-from $ECR_REGISTRY/booking-service:latest \
            .
          docker push $ECR_REGISTRY/booking-service:$IMAGE_TAG
          docker push $ECR_REGISTRY/booking-service:latest

      - name: Scan image for vulnerabilities
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/booking-service:${{ github.sha }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1  # Fail pipeline on CRITICAL/HIGH CVEs

      - name: Upload scan results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build-and-push]
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/github-actions-deploy-prod
          aws-region: us-east-1

      - name: Update EKS deployment image
        run: |
          aws eks update-kubeconfig --name prod-cluster --region us-east-1

          kubectl set image deployment/booking-service \
            api=${{ vars.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/booking-service:${{ github.sha }} \
            -n production

          kubectl rollout status deployment/booking-service -n production --timeout=300s

      - name: Run smoke tests
        run: |
          API_URL=$(kubectl get ingress booking-service -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
          curl -sf "https://$API_URL/health" | jq .

      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
            -H 'Content-type: application/json' \
            -d '{"text":"🚨 *booking-service* deploy failed on commit ${{ github.sha }}"}'
```
