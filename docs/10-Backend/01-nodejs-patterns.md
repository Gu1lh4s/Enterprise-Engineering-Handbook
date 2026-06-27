# Node.js Production Patterns

> **Category:** Backend Engineering
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Application Architecture](#1-application-architecture)
2. [Error Handling](#2-error-handling)
3. [Configuration Management](#3-configuration-management)
4. [Graceful Shutdown](#4-graceful-shutdown)
5. [Connection Management](#5-connection-management)
6. [Async Patterns](#6-async-patterns)
7. [Memory Management](#7-memory-management)
8. [Worker Threads](#8-worker-threads)
9. [Job Queues (BullMQ)](#9-job-queues-bullmq)
10. [TypeScript Best Practices](#10-typescript-best-practices)
11. [Security Checklist](#11-security-checklist)
12. [Performance Tuning](#12-performance-tuning)

---

## 1. Application Architecture

### Layered Architecture (Clean Architecture for APIs)

```
src/
├── domain/               ← Business logic (no external dependencies)
│   ├── booking/
│   │   ├── booking.entity.ts       # Domain model
│   │   ├── booking.repository.ts   # Interface only (no implementation)
│   │   ├── booking.service.ts      # Business rules
│   │   └── booking.errors.ts       # Domain-specific errors
│   └── user/
│       ├── user.entity.ts
│       └── user.service.ts
│
├── application/          ← Use cases (orchestrate domain)
│   ├── create-booking.usecase.ts
│   └── cancel-booking.usecase.ts
│
├── infrastructure/       ← External adapters (DB, email, queue)
│   ├── database/
│   │   ├── schema.ts               # Drizzle/Prisma schema
│   │   └── booking.repository.ts   # Implements domain interface
│   ├── email/
│   │   └── email.service.ts        # SMTP/SES implementation
│   └── queue/
│       └── job.processor.ts
│
└── presentation/         ← HTTP layer (routes, controllers)
    ├── http/
    │   ├── bookings.router.ts
    │   └── bookings.controller.ts
    └── middleware/
        ├── auth.ts
        ├── validate.ts
        └── rate-limit.ts
```

### Module Pattern

```typescript
// Each domain module exports a factory
// booking.module.ts
import { BookingRepository } from '../infrastructure/database/booking.repository'
import { BookingService } from './booking.service'
import { CreateBookingUseCase } from '../application/create-booking.usecase'
import { BookingsController } from '../presentation/http/bookings.controller'
import { BookingsRouter } from '../presentation/http/bookings.router'

export function createBookingModule(deps: { db: Database; redis: Redis; emailService: EmailService }) {
  const repository = new BookingRepository(deps.db)
  const service = new BookingService(repository, deps.emailService)
  const createBooking = new CreateBookingUseCase(service)
  const controller = new BookingsController(createBooking)
  const router = new BookingsRouter(controller)
  
  return { router: router.getRouter() }
}

// app.ts
const { router: bookingsRouter } = createBookingModule({ db, redis, emailService })
app.use('/api/v1/bookings', bookingsRouter)
```

---

## 2. Error Handling

### Error Hierarchy

```typescript
// Base application error
export class AppError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly statusCode: number,
    public readonly details: object[] = [],
    public readonly isOperational = true,  // Expected errors vs programmer errors
  ) {
    super(message)
    this.name = this.constructor.name
    Error.captureStackTrace(this, this.constructor)
  }
}

// Domain errors (inherit status code from context)
export class BookingNotFoundError extends AppError {
  constructor(id: string) {
    super('BOOKING_NOT_FOUND', `Booking ${id} not found`, 404)
  }
}

export class SlotUnavailableError extends AppError {
  constructor() {
    super('SLOT_UNAVAILABLE', 'This time slot is no longer available', 409)
  }
}

export class ValidationError extends AppError {
  constructor(message: string, details: { field: string; message: string }[]) {
    super('VALIDATION_ERROR', message, 422, details)
  }
}

export class PaymentFailedError extends AppError {
  constructor(reason: string) {
    super('PAYMENT_FAILED', `Payment failed: ${reason}`, 402)
  }
}

// Distinguish operational (expected) vs programming errors
function isOperationalError(error: unknown): boolean {
  if (error instanceof AppError) return error.isOperational
  return false
}

// Unhandled errors → crash and restart (use PM2/Kubernetes to restart)
process.on('uncaughtException', (error) => {
  logger.fatal({ err: error }, 'Uncaught exception — crashing')
  process.exit(1)  // Let process manager restart
})

process.on('unhandledRejection', (reason) => {
  throw reason  // Convert to uncaught exception → above handler takes over
})
```

---

## 3. Configuration Management

```typescript
// config/index.ts — centralized, validated configuration
import { z } from 'zod'

const ConfigSchema = z.object({
  nodeEnv: z.enum(['development', 'staging', 'production']).default('development'),
  port: z.coerce.number().default(3000),
  
  // Database
  databaseUrl: z.string().url(),
  databaseMaxConnections: z.coerce.number().default(10),
  
  // Redis
  redisUrl: z.string().url().default('redis://localhost:6379'),
  
  // Auth
  jwtSecret: z.string().min(32, 'JWT secret must be at least 32 chars'),
  jwtAccessTokenTtlSec: z.coerce.number().default(900),    // 15 min
  jwtRefreshTokenTtlDays: z.coerce.number().default(30),
  
  // Email
  smtpHost: z.string(),
  smtpPort: z.coerce.number(),
  smtpUser: z.string(),
  smtpPassword: z.string(),
  
  // Feature flags
  featureEmailNotifications: z.coerce.boolean().default(true),
  featureWaitlist: z.coerce.boolean().default(false),
})

function loadConfig() {
  const result = ConfigSchema.safeParse({
    nodeEnv: process.env.NODE_ENV,
    port: process.env.PORT,
    databaseUrl: process.env.DATABASE_URL,
    databaseMaxConnections: process.env.DATABASE_MAX_CONNECTIONS,
    redisUrl: process.env.REDIS_URL,
    jwtSecret: process.env.JWT_SECRET,
    jwtAccessTokenTtlSec: process.env.JWT_ACCESS_TOKEN_TTL_SEC,
    jwtRefreshTokenTtlDays: process.env.JWT_REFRESH_TOKEN_TTL_DAYS,
    smtpHost: process.env.SMTP_HOST,
    smtpPort: process.env.SMTP_PORT,
    smtpUser: process.env.SMTP_USER,
    smtpPassword: process.env.SMTP_PASSWORD,
    featureEmailNotifications: process.env.FEATURE_EMAIL_NOTIFICATIONS,
    featureWaitlist: process.env.FEATURE_WAITLIST,
  })
  
  if (!result.success) {
    const errors = result.error.errors.map(e => `${e.path.join('.')}: ${e.message}`).join('\n')
    throw new Error(`Invalid configuration:\n${errors}`)
  }
  
  return result.data
}

// Singleton — validate once at startup, crash if invalid
export const config = loadConfig()

// Usage
import { config } from './config'
const pool = new Pool({ connectionString: config.databaseUrl, max: config.databaseMaxConnections })
```

---

## 4. Graceful Shutdown

```typescript
// Critical: handle SIGTERM/SIGINT for zero-downtime deployments and Kubernetes

async function shutdown(signal: string) {
  logger.info({ signal }, 'Received shutdown signal, starting graceful shutdown')
  
  // 1. Stop accepting new connections
  server.close(() => {
    logger.info('HTTP server closed')
  })
  
  // 2. Wait for in-flight requests to complete (with timeout)
  const shutdownTimeout = setTimeout(() => {
    logger.warn('Graceful shutdown timeout exceeded — forcing exit')
    process.exit(1)
  }, 30_000)  // 30s max wait
  
  try {
    // 3. Drain job queues (don't pick up new jobs)
    await jobQueue.pause()
    await jobQueue.close()
    
    // 4. Close DB connections (after in-flight queries complete)
    await db.end()
    
    // 5. Close Redis connection
    await redis.quit()
    
    clearTimeout(shutdownTimeout)
    logger.info('Graceful shutdown complete')
    process.exit(0)
    
  } catch (err) {
    logger.error({ err }, 'Error during shutdown')
    process.exit(1)
  }
}

process.on('SIGTERM', () => shutdown('SIGTERM'))  // Kubernetes sends this
process.on('SIGINT', () => shutdown('SIGINT'))    // Ctrl+C in terminal
```

---

## 5. Connection Management

### Database Connection Pool

```typescript
import { Pool } from 'pg'
import { drizzle } from 'drizzle-orm/node-postgres'

// Pool sizing: (core_count * 2) + effective_spindle_count
// For typical cloud DB: max 10-20 connections per app instance
// Remember: each Kubernetes pod = 1 app instance

const pool = new Pool({
  connectionString: config.databaseUrl,
  max: config.databaseMaxConnections,  // Default: 10
  idleTimeoutMillis: 30_000,           // Release idle connections after 30s
  connectionTimeoutMillis: 5_000,      // Fail fast if no connections available
  
  ssl: config.nodeEnv === 'production' ? { rejectUnauthorized: true } : false,
})

// Monitor pool health
pool.on('connect', () => {
  logger.debug({ total: pool.totalCount, idle: pool.idleCount }, 'DB: new connection')
})

pool.on('error', (err) => {
  logger.error({ err }, 'DB: unexpected connection error')
})

export const db = drizzle(pool, { schema })
```

### Redis Connection

```typescript
import Redis from 'ioredis'

const redis = new Redis(config.redisUrl, {
  maxRetriesPerRequest: 3,
  retryStrategy: (times) => Math.min(times * 100, 3000),  // Exponential backoff, max 3s
  lazyConnect: true,  // Don't connect immediately
  enableReadyCheck: true,
  
  // Reconnect on connection loss
  reconnectOnError: (err) => {
    const targetErrors = ['READONLY', 'ECONNRESET', 'ECONNREFUSED']
    return targetErrors.some(e => err.message.includes(e))
  },
})

redis.on('error', (err) => logger.error({ err }, 'Redis: connection error'))
redis.on('connect', () => logger.info('Redis: connected'))
redis.on('ready', () => logger.info('Redis: ready'))

// Separate connection for subscriptions (ioredis limitation)
export const redisSub = redis.duplicate()
```

---

## 6. Async Patterns

### Promise.allSettled for Independent Operations

```typescript
// Use allSettled when operations are independent — don't let one failure kill all
async function sendBookingNotifications(booking: Booking) {
  const [emailResult, smsResult, pushResult] = await Promise.allSettled([
    emailService.sendConfirmation(booking),
    smsService.sendConfirmation(booking),
    pushNotificationService.send(booking.clientId, 'Booking confirmed!'),
  ])
  
  // Log failures but don't throw — notifications are best-effort
  if (emailResult.status === 'rejected') {
    logger.error({ err: emailResult.reason, bookingId: booking.id }, 'Failed to send email')
  }
  if (smsResult.status === 'rejected') {
    logger.warn({ err: smsResult.reason, bookingId: booking.id }, 'Failed to send SMS')
  }
  
  return {
    email: emailResult.status === 'fulfilled',
    sms: smsResult.status === 'fulfilled',
    push: pushResult.status === 'fulfilled',
  }
}
```

### Timeout Wrapper

```typescript
function withTimeout<T>(promise: Promise<T>, ms: number, context: string): Promise<T> {
  let timer: NodeJS.Timeout
  
  const timeout = new Promise<never>((_, reject) => {
    timer = setTimeout(() => reject(new Error(`${context} timed out after ${ms}ms`)), ms)
  })
  
  return Promise.race([promise, timeout]).finally(() => clearTimeout(timer))
}

// Usage
const booking = await withTimeout(
  externalPaymentService.charge(amount),
  5_000,
  'Payment charge'
)
```

### Retry with Exponential Backoff

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  options: { retries: number; baseDelayMs: number; shouldRetry?: (err: Error) => boolean }
): Promise<T> {
  const { retries, baseDelayMs, shouldRetry = () => true } = options
  
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      return await fn()
    } catch (err) {
      if (attempt === retries || !shouldRetry(err as Error)) throw err
      
      const delay = baseDelayMs * 2 ** attempt + Math.random() * 100  // Jitter
      logger.warn({ err, attempt, nextRetryIn: delay }, 'Retrying after error')
      await new Promise(resolve => setTimeout(resolve, delay))
    }
  }
  
  throw new Error('unreachable')
}

// Usage: retry payment up to 3 times, but not for user errors
const result = await withRetry(
  () => stripeClient.charges.create(chargeData),
  {
    retries: 3,
    baseDelayMs: 500,
    shouldRetry: (err) => {
      // Don't retry client errors (card declined, invalid CVC)
      if (err instanceof Stripe.errors.StripeCardError) return false
      return true
    }
  }
)
```

---

## 7. Memory Management

```typescript
// Common Node.js memory issues and fixes

// 1. EventEmitter memory leaks
// Node warns if > 10 listeners attached to same event
emitter.setMaxListeners(0)  // Remove limit (dangerous — masks leaks)
emitter.setMaxListeners(20)  // Raise limit explicitly

// Better: explicitly remove listeners
const handler = () => { /* ... */ }
eventBus.on('booking.created', handler)
// Remember to remove:
eventBus.off('booking.created', handler)

// 2. Avoid large array accumulation
// BAD: accumulate all records in memory
const allUsers = await db.query('SELECT * FROM users')  // 1M users = OOM

// GOOD: stream/cursor-based iteration
const cursor = db.query('SELECT * FROM users').cursor(1000)
for await (const batch of cursor) {
  await processBatch(batch)
}

// 3. Memory usage monitoring
setInterval(() => {
  const { heapUsed, heapTotal, rss } = process.memoryUsage()
  memoryGauge.set({ type: 'heap_used' }, heapUsed)
  memoryGauge.set({ type: 'heap_total' }, heapTotal)
  memoryGauge.set({ type: 'rss' }, rss)
  
  // Alert if heap usage > 80%
  if (heapUsed / heapTotal > 0.8) {
    logger.warn({ heapUsed, heapTotal }, 'High memory usage')
  }
}, 30_000)

// 4. Node.js heap size limit (default: ~1.5GB on 64-bit)
// Set explicitly: NODE_OPTIONS=--max-old-space-size=512
```

---

## 8. Worker Threads

```typescript
// CPU-intensive work (image processing, report generation, cryptography)
// → offload to worker threads to not block the event loop

// worker.ts
import { workerData, parentPort } from 'worker_threads'

async function processHeavyTask(data: ProcessingInput): Promise<ProcessingResult> {
  // CPU-intensive work here
  const result = await generateLargeReport(data)
  return result
}

processHeavyTask(workerData).then(
  result => parentPort!.postMessage({ success: true, result }),
  err => parentPort!.postMessage({ success: false, error: err.message })
)

// main.ts — run in worker thread
import { Worker } from 'worker_threads'
import { cpus } from 'os'

// Thread pool for CPU-bound tasks
class WorkerPool {
  private queue: Array<{ data: unknown; resolve: Function; reject: Function }> = []
  private workers: Worker[] = []
  private activeCount = 0
  
  constructor(private workerPath: string, private maxWorkers = cpus().length) {}
  
  run<T>(data: unknown): Promise<T> {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject })
      this.processQueue()
    })
  }
  
  private processQueue() {
    while (this.queue.length > 0 && this.activeCount < this.maxWorkers) {
      const task = this.queue.shift()!
      this.activeCount++
      
      const worker = new Worker(this.workerPath, { workerData: task.data })
      worker.on('message', ({ success, result, error }) => {
        this.activeCount--
        success ? task.resolve(result) : task.reject(new Error(error))
        this.processQueue()
      })
      worker.on('error', (err) => {
        this.activeCount--
        task.reject(err)
        this.processQueue()
      })
    }
  }
}

const reportPool = new WorkerPool('./worker.js', 4)
const report = await reportPool.run<ReportData>({ userId, period: '2025-Q1' })
```

---

## 9. Job Queues (BullMQ)

```typescript
import { Queue, Worker, QueueEvents } from 'bullmq'

// Define job types
interface EmailJob {
  type: 'booking_confirmation' | 'reminder' | 'cancellation'
  bookingId: string
  recipientEmail: string
}

// Create queue
const emailQueue = new Queue<EmailJob>('emails', {
  connection: redis,
  defaultJobOptions: {
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },  // 2s, 4s, 8s
    removeOnComplete: { count: 1000 },  // Keep last 1000 completed jobs
    removeOnFail: { count: 5000 },      // Keep last 5000 failed jobs for debugging
  },
})

// Add jobs to queue
await emailQueue.add('send', {
  type: 'booking_confirmation',
  bookingId: 'booking_123',
  recipientEmail: 'user@example.com',
}, {
  delay: 0,           // Immediate
  priority: 1,        // Higher priority for immediate sends
})

// Schedule reminders: delay until 24h before appointment
const reminderDelay = new Date(scheduledAt).getTime() - Date.now() - 24 * 60 * 60 * 1000
await emailQueue.add('send', { type: 'reminder', bookingId, recipientEmail }, { delay: reminderDelay })

// Worker to process jobs
const emailWorker = new Worker<EmailJob>(
  'emails',
  async (job) => {
    logger.info({ jobId: job.id, type: job.data.type }, 'Processing email job')
    
    switch (job.data.type) {
      case 'booking_confirmation':
        await emailService.sendBookingConfirmation(job.data.bookingId)
        break
      case 'reminder':
        await emailService.sendReminder(job.data.bookingId)
        break
      case 'cancellation':
        await emailService.sendCancellation(job.data.bookingId)
        break
    }
  },
  {
    connection: redis,
    concurrency: 5,    // Process 5 jobs simultaneously
    limiter: {
      max: 10,         // Max 10 jobs per 1 second (rate limiting)
      duration: 1000,
    },
  }
)

emailWorker.on('completed', (job) => {
  logger.info({ jobId: job.id }, 'Email job completed')
})

emailWorker.on('failed', (job, err) => {
  logger.error({ jobId: job?.id, err }, 'Email job failed')
})
```

---

## 10. TypeScript Best Practices

```typescript
// 1. Strict mode (tsconfig.json)
{
  "compilerOptions": {
    "strict": true,                    // All strict checks enabled
    "noUncheckedIndexedAccess": true,  // arr[0] is T | undefined, not T
    "exactOptionalPropertyTypes": true, // {a?: string} ≠ {a?: string | undefined}
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}

// 2. Discriminated unions for state modeling
type BookingState =
  | { status: 'pending'; paymentMethod: null }
  | { status: 'confirmed'; paymentMethod: string; confirmedAt: Date }
  | { status: 'cancelled'; cancelledAt: Date; reason: string }
  | { status: 'completed'; completedAt: Date }

function processBooking(booking: Booking & BookingState) {
  switch (booking.status) {
    case 'confirmed':
      // TypeScript knows booking.confirmedAt exists here
      console.log(booking.confirmedAt)
      break
    case 'cancelled':
      console.log(booking.reason)
      break
    // TypeScript: exhaustive check — missing case is compile error
    default:
      const _exhaustive: never = booking
  }
}

// 3. Branded types for ID safety
type UserId = string & { readonly __brand: 'UserId' }
type BookingId = string & { readonly __brand: 'BookingId' }

const createUserId = (id: string): UserId => id as UserId
const createBookingId = (id: string): BookingId => id as BookingId

// Now this is a compile error:
function getBooking(id: BookingId) { /* ... */ }
const userId = createUserId('user_123')
getBooking(userId)  // Error: Argument of type 'UserId' is not assignable to 'BookingId'

// 4. Result type (avoid exceptions for expected failures)
type Result<T, E = Error> = 
  | { success: true; data: T }
  | { success: false; error: E }

async function parseBookingId(input: string): Promise<Result<BookingId>> {
  if (!UUID_REGEX.test(input)) {
    return { success: false, error: new ValidationError('Invalid booking ID format') }
  }
  return { success: true, data: createBookingId(input) }
}
```

---

## 11. Security Checklist

```
[ ] Validate all inputs with Zod at API boundaries
[ ] Never trust Content-Type header — always parse JSON explicitly
[ ] Use parameterized queries — never string concatenation in SQL
[ ] Set reasonable request body size limit (express: limit: '10mb')
[ ] Rate limit all endpoints — especially auth endpoints
[ ] Set security headers (Helmet.js)
[ ] JWT: verify algorithm explicitly (algorithms: ['HS256']) — never trust 'alg' in token
[ ] JWT: store secret in env var — minimum 32 chars entropy
[ ] Never log JWT tokens, passwords, or sensitive data
[ ] Mask PII in logs (use pino redact)
[ ] Set Cookie secure, httpOnly, sameSite=Strict
[ ] CORS: explicit allowlist, never origin: '*' for authenticated APIs
[ ] Content Security Policy on all HTML responses
[ ] npm audit in CI — block on high severity
[ ] Scan Docker images in CI (Trivy)
[ ] Use non-root user in Docker
[ ] Disable x-powered-by header (app.disable('x-powered-by'))
```

---

## 12. Performance Tuning

```typescript
// 1. Enable Node.js cluster mode (utilize all CPU cores)
import cluster from 'cluster'
import { cpus } from 'os'

if (cluster.isPrimary) {
  const numCPUs = cpus().length
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork()
  }
  cluster.on('exit', (worker, code, signal) => {
    logger.error({ pid: worker.process.pid, code, signal }, 'Worker died, restarting')
    cluster.fork()  // Restart dead worker
  })
} else {
  // Worker process — start the actual server
  startServer()
}

// 2. HTTP Keep-Alive (reuse TCP connections)
const server = app.listen(config.port)
server.keepAliveTimeout = 65_000   // Slightly higher than AWS ALB's 60s
server.headersTimeout = 66_000     // Must be > keepAliveTimeout

// 3. Response compression
import compression from 'compression'
app.use(compression({
  level: 6,          // Balance between CPU and compression ratio
  threshold: 1024,   // Only compress responses > 1KB
  filter: (req, res) => {
    // Don't compress SSE streams or already-compressed content
    if (req.headers['accept'] === 'text/event-stream') return false
    return compression.filter(req, res)
  }
}))

// 4. Cache static responses
app.get('/api/v1/services', async (req, res) => {
  const cached = await redis.get('services:all')
  if (cached) {
    return res.setHeader('X-Cache', 'HIT').json(JSON.parse(cached))
  }
  
  const services = await serviceRepository.findAll()
  await redis.setex('services:all', 300, JSON.stringify(services))  // 5 min cache
  
  res.setHeader('X-Cache', 'MISS').json({ data: services })
})

// 5. Avoid synchronous operations (especially in hot path)
// BAD: sync file read blocks event loop
const cert = fs.readFileSync('cert.pem')

// GOOD: read once at startup, use from memory
let cert: Buffer
async function init() {
  cert = await fs.promises.readFile('cert.pem')
}
```
