# Testing Strategy

> **Category:** Quality Engineering
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Testing Pyramid](#1-testing-pyramid)
2. [Unit Testing](#2-unit-testing)
3. [Integration Testing](#3-integration-testing)
4. [End-to-End Testing](#4-end-to-end-testing)
5. [Contract Testing](#5-contract-testing)
6. [Performance Testing](#6-performance-testing)
7. [Security Testing](#7-security-testing)
8. [Test Data Management](#8-test-data-management)
9. [CI/CD Testing Pipeline](#9-cicd-testing-pipeline)
10. [Coverage Strategy](#10-coverage-strategy)

---

## 1. Testing Pyramid

```
                    ┌──────────────┐
                    │     E2E      │  Few, slow, expensive
                    │   (Playwright│  Test critical user journeys
                    └──────┬───────┘  ~5-10% of tests
                           │
              ┌────────────┴────────────┐
              │      Integration        │  Some, moderate speed
              │ (real DB, real services)│  Test service boundaries
              └────────────┬────────────┘  ~20-30% of tests
                           │
         ┌─────────────────┴─────────────────┐
         │              Unit                  │  Many, fast, cheap
         │    (pure functions, mocked deps)   │  Test logic
         └───────────────────────────────────┘  ~60-70% of tests

Speed:   Unit: <10ms    Integration: <1s    E2E: 5-60s
Feedback: Unit: instant  Integration: local  E2E: CI only

Anti-pattern: "Ice Cream Cone" — mostly E2E, few unit tests
  → Slow CI (30+ minutes), flaky tests, poor signal
  
Anti-pattern: "Pure Unit Only" — 100% unit tests, all mocks
  → High coverage, but integration bugs slip through
```

---

## 2. Unit Testing

### Test Naming Convention

```typescript
// Pattern: describe("what"), it("should behavior when condition")
// Or: test("given [precondition] when [action] then [result]")

describe('BookingService', () => {
  describe('createBooking', () => {
    it('should create booking when slot is available', async () => {})
    it('should throw SlotUnavailableError when slot is taken', async () => {})
    it('should throw ValidationError when date is in the past', async () => {})
    it('should send confirmation email after successful booking', async () => {})
  })
})
```

### Unit Test Structure (AAA)

```typescript
import { BookingService } from '../booking.service'
import { createMockRepository, createMockEmailService } from '../testing/mocks'

describe('BookingService', () => {
  let bookingService: BookingService
  let mockBookingRepo: jest.Mocked<BookingRepository>
  let mockEmailService: jest.Mocked<EmailService>
  
  beforeEach(() => {
    // Fresh mocks for each test (prevent state leak)
    mockBookingRepo = createMockRepository()
    mockEmailService = createMockEmailService()
    
    bookingService = new BookingService(mockBookingRepo, mockEmailService)
  })
  
  describe('createBooking', () => {
    it('should create booking when slot is available', async () => {
      // ─── Arrange ─────────────────────────────────────────────────────────
      const input = {
        serviceId: 'service_123',
        scheduledAt: new Date('2025-07-01T10:00:00Z'),
        clientId: 'user_456',
      }
      
      mockBookingRepo.isSlotAvailable.mockResolvedValue(true)
      mockBookingRepo.create.mockResolvedValue({
        id: 'booking_789',
        ...input,
        status: 'pending',
        createdAt: new Date(),
      })
      
      // ─── Act ──────────────────────────────────────────────────────────────
      const booking = await bookingService.createBooking(input)
      
      // ─── Assert ───────────────────────────────────────────────────────────
      expect(booking.id).toBe('booking_789')
      expect(booking.status).toBe('pending')
      
      expect(mockBookingRepo.create).toHaveBeenCalledOnce()
      expect(mockBookingRepo.create).toHaveBeenCalledWith(
        expect.objectContaining({ serviceId: input.serviceId })
      )
      
      expect(mockEmailService.sendBookingConfirmation).toHaveBeenCalledOnce()
      expect(mockEmailService.sendBookingConfirmation).toHaveBeenCalledWith(
        expect.objectContaining({ bookingId: 'booking_789' })
      )
    })
    
    it('should throw SlotUnavailableError when slot is taken', async () => {
      // Arrange
      mockBookingRepo.isSlotAvailable.mockResolvedValue(false)
      
      // Act & Assert
      await expect(
        bookingService.createBooking({
          serviceId: 'service_123',
          scheduledAt: new Date('2025-07-01T10:00:00Z'),
          clientId: 'user_456',
        })
      ).rejects.toThrow(SlotUnavailableError)
      
      // Verify no booking was created
      expect(mockBookingRepo.create).not.toHaveBeenCalled()
      expect(mockEmailService.sendBookingConfirmation).not.toHaveBeenCalled()
    })
    
    it('should throw ValidationError when date is in the past', async () => {
      await expect(
        bookingService.createBooking({
          serviceId: 'service_123',
          scheduledAt: new Date('2020-01-01T10:00:00Z'),  // Past date
          clientId: 'user_456',
        })
      ).rejects.toThrow(new ValidationError('scheduledAt must be a future date'))
    })
  })
})
```

### Testing Pure Functions

```typescript
// Pure functions are the easiest to test — no setup, no teardown, no mocks
describe('calculateBookingPrice', () => {
  const cases: Array<[string, BookingInput, Money]> = [
    ['standard service, 60 min', { serviceId: 'standard', durationMin: 60 }, { amount: 15000, currency: 'BRL' }],
    ['standard service, 90 min', { serviceId: 'standard', durationMin: 90 }, { amount: 22500, currency: 'BRL' }],
    ['premium service, 60 min', { serviceId: 'premium', durationMin: 60 }, { amount: 25000, currency: 'BRL' }],
  ]
  
  it.each(cases)('%s', (_, input, expected) => {
    expect(calculateBookingPrice(input)).toEqual(expected)
  })
})

// Edge cases
describe('parseScheduledAt', () => {
  it('should handle timezone-naive strings by assuming UTC', () => {
    expect(parseScheduledAt('2025-06-27T10:00:00')).toEqual(new Date('2025-06-27T10:00:00Z'))
  })
  
  it('should reject invalid date strings', () => {
    expect(() => parseScheduledAt('not-a-date')).toThrow(ValidationError)
    expect(() => parseScheduledAt('')).toThrow(ValidationError)
    expect(() => parseScheduledAt(null as any)).toThrow(ValidationError)
  })
  
  it('should preserve timezone offset', () => {
    const result = parseScheduledAt('2025-06-27T10:00:00-03:00')
    expect(result.toISOString()).toBe('2025-06-27T13:00:00.000Z')
  })
})
```

### Snapshot Testing (Use Sparingly)

```typescript
// Good for: complex serialized structures, generated reports, email templates
// Bad for: small components (changes become noisy and ignored)

it('should generate correct invoice structure', () => {
  const invoice = generateInvoice({ bookingId: 'booking_123', amount: 15000 })
  
  expect(invoice).toMatchSnapshot()
  // Creates __snapshots__/invoice.test.ts.snap
  // Update with: jest --updateSnapshot
})
```

---

## 3. Integration Testing

### Database Integration Tests

```typescript
// Use TestContainers — real PostgreSQL in Docker, not in-memory SQLite
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql'
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'

let container: StartedPostgreSqlContainer
let db: ReturnType<typeof drizzle>

beforeAll(async () => {
  // Start real PostgreSQL container (takes ~5s)
  container = await new PostgreSqlContainer('postgres:16-alpine')
    .withDatabase('test_db')
    .withUsername('test')
    .withPassword('test')
    .start()
  
  const client = postgres(container.getConnectionUri())
  db = drizzle(client, { schema })
  
  // Run migrations
  await migrate(db, { migrationsFolder: './drizzle' })
}, 30_000)

afterAll(async () => {
  await container.stop()
})

beforeEach(async () => {
  // Truncate all tables before each test (clean state)
  await db.execute(sql`TRUNCATE TABLE bookings, users, services CASCADE`)
})

describe('BookingRepository', () => {
  it('should find bookings by client ID', async () => {
    // Arrange: insert test data
    const [user] = await db.insert(users).values({ email: 'test@example.com', name: 'Test User' }).returning()
    const [service] = await db.insert(services).values({ name: 'Massage', price: 15000 }).returning()
    
    await db.insert(bookings).values([
      { clientId: user.id, serviceId: service.id, scheduledAt: new Date('2025-07-01T10:00:00Z') },
      { clientId: user.id, serviceId: service.id, scheduledAt: new Date('2025-07-02T10:00:00Z') },
    ])
    
    // Act
    const repo = new BookingRepository(db)
    const results = await repo.findByClientId(user.id)
    
    // Assert
    expect(results).toHaveLength(2)
    expect(results[0].clientId).toBe(user.id)
  })
  
  it('should enforce unique constraint on time slot', async () => {
    const [user] = await db.insert(users).values({ email: 'test@example.com', name: 'Test' }).returning()
    const [service] = await db.insert(services).values({ name: 'Massage', price: 15000 }).returning()
    
    // First booking succeeds
    await db.insert(bookings).values({ clientId: user.id, serviceId: service.id, scheduledAt: new Date('2025-07-01T10:00:00Z') })
    
    // Second booking at same time should fail
    await expect(
      db.insert(bookings).values({ clientId: user.id, serviceId: service.id, scheduledAt: new Date('2025-07-01T10:00:00Z') })
    ).rejects.toThrow()
  })
})
```

### HTTP Integration Tests (Supertest)

```typescript
import request from 'supertest'
import { app } from '../../src/app'
import { db } from '../../src/db'
import { generateTestJwt } from '../helpers/auth'

describe('POST /api/v1/bookings', () => {
  let authToken: string
  let testUser: User
  
  beforeEach(async () => {
    await db.execute(sql`TRUNCATE TABLE bookings, users CASCADE`)
    testUser = await createTestUser()
    authToken = generateTestJwt({ userId: testUser.id, role: 'user' })
  })
  
  it('should create booking and return 201', async () => {
    const response = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        serviceId: testService.id,
        scheduledAt: '2025-07-01T10:00:00Z',
      })
      .expect(201)
    
    expect(response.body.data.status).toBe('pending')
    expect(response.body.data.clientId).toBe(testUser.id)
    
    // Verify it's actually in the database
    const booking = await db.query.bookings.findFirst({ where: eq(bookings.id, response.body.data.id) })
    expect(booking).toBeDefined()
  })
  
  it('should return 401 without auth token', async () => {
    await request(app)
      .post('/api/v1/bookings')
      .send({ serviceId: 'service_123', scheduledAt: '2025-07-01T10:00:00Z' })
      .expect(401)
  })
  
  it('should return 422 with invalid body', async () => {
    const response = await request(app)
      .post('/api/v1/bookings')
      .set('Authorization', `Bearer ${authToken}`)
      .send({ serviceId: 'invalid-uuid', scheduledAt: 'not-a-date' })
      .expect(422)
    
    expect(response.body.error.code).toBe('VALIDATION_ERROR')
    expect(response.body.error.details).toHaveLength(2)
  })
})
```

---

## 4. End-to-End Testing

### Playwright Setup

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  timeout: 30_000,
  retries: process.env.CI ? 2 : 0,  // Retry flaky tests in CI
  workers: process.env.CI ? 1 : undefined,  // Sequential in CI, parallel locally
  
  reporter: [
    ['html', { outputFolder: 'playwright-report', open: 'never' }],
    ['junit', { outputFile: 'test-results.xml' }],  // For CI
    process.env.CI ? ['github'] : ['list'],
  ],
  
  use: {
    baseURL: process.env.BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',       // Record trace on retry (helps debugging)
    screenshot: 'only-on-failure', // Screenshot on failure
    video: 'on-first-retry',
  },
  
  projects: [
    { name: 'Desktop Chrome', use: { ...devices['Desktop Chrome'] } },
    { name: 'Mobile Safari', use: { ...devices['iPhone 14'] } },
  ],
  
  // Start dev server before tests
  webServer: {
    command: 'npm run start:test',
    url: 'http://localhost:3000/health',
    reuseExistingServer: !process.env.CI,
    timeout: 60_000,
  },
})
```

### Page Object Model

```typescript
// e2e/pages/booking.page.ts
import { Page, Locator } from '@playwright/test'

export class BookingPage {
  private readonly page: Page
  
  readonly serviceSelect: Locator
  readonly dateInput: Locator
  readonly timeSelect: Locator
  readonly confirmButton: Locator
  readonly successMessage: Locator
  
  constructor(page: Page) {
    this.page = page
    this.serviceSelect = page.getByRole('combobox', { name: /service/i })
    this.dateInput = page.getByLabel(/date/i)
    this.timeSelect = page.getByLabel(/time/i)
    this.confirmButton = page.getByRole('button', { name: /confirm booking/i })
    this.successMessage = page.getByText(/booking confirmed/i)
  }
  
  async goto() {
    await this.page.goto('/book')
  }
  
  async bookService(service: string, date: string, time: string) {
    await this.serviceSelect.selectOption(service)
    await this.dateInput.fill(date)
    await this.timeSelect.selectOption(time)
    await this.confirmButton.click()
    await this.successMessage.waitFor()
  }
}

// e2e/booking.spec.ts
import { test, expect } from '@playwright/test'
import { BookingPage } from './pages/booking.page'
import { LoginPage } from './pages/login.page'

test.describe('Booking flow', () => {
  test.beforeEach(async ({ page }) => {
    // Login before each test
    const loginPage = new LoginPage(page)
    await loginPage.login('test@example.com', 'password123')
  })
  
  test('user can book a service', async ({ page }) => {
    const bookingPage = new BookingPage(page)
    await bookingPage.goto()
    await bookingPage.bookService('Relaxing Massage', '2025-07-15', '10:00')
    
    // Verify confirmation
    await expect(bookingPage.successMessage).toBeVisible()
    await expect(page).toHaveURL(/\/bookings\/booking_/)
  })
  
  test('user cannot book past date', async ({ page }) => {
    const bookingPage = new BookingPage(page)
    await bookingPage.goto()
    await bookingPage.dateInput.fill('2020-01-01')
    await bookingPage.confirmButton.click()
    
    await expect(page.getByText(/date must be in the future/i)).toBeVisible()
  })
})
```

### API Testing with Playwright

```typescript
// Test API directly without UI
test('should return user bookings via API', async ({ request }) => {
  // Login to get token
  const authResponse = await request.post('/api/v1/auth/login', {
    data: { email: 'test@example.com', password: 'password123' }
  })
  const { data: { accessToken } } = await authResponse.json()
  
  // Create a booking
  await request.post('/api/v1/bookings', {
    headers: { Authorization: `Bearer ${accessToken}` },
    data: { serviceId: 'service_123', scheduledAt: '2025-07-01T10:00:00Z' },
  })
  
  // Verify it appears in list
  const listResponse = await request.get('/api/v1/bookings', {
    headers: { Authorization: `Bearer ${accessToken}` }
  })
  const { data: bookings } = await listResponse.json()
  
  expect(listResponse.status()).toBe(200)
  expect(bookings.length).toBeGreaterThan(0)
})
```

---

## 5. Contract Testing

```typescript
// See docs/12-API/01-api-design.md §12 for full Pact example

// Provider verification (in provider's CI)
import { Verifier } from '@pact-foundation/pact'

describe('Provider verification', () => {
  it('should satisfy all consumer contracts', async () => {
    await new Verifier({
      providerBaseUrl: 'http://localhost:3000',
      pactBrokerUrl: process.env.PACT_BROKER_URL,
      providerVersion: process.env.APP_VERSION,
      publishVerificationResult: true,
      
      stateHandlers: {
        'a booking with ID booking_123 exists': async () => {
          await db.insert(bookings).values({ id: 'booking_123', ... })
        },
        'no bookings exist': async () => {
          await db.execute(sql`TRUNCATE TABLE bookings`)
        },
      }
    }).verifyProvider()
  })
})
```

---

## 6. Performance Testing

### k6 Load Testing

```javascript
// k6 script: booking_load_test.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate, Trend } from 'k6/metrics'

const errorRate = new Rate('errors')
const bookingDuration = new Trend('booking_creation_time')

export const options = {
  scenarios: {
    // Ramp up gradually
    load_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '2m', target: 10 },   // Warm up: 0→10 users in 2 min
        { duration: '5m', target: 50 },   // Ramp: 10→50 users in 5 min
        { duration: '10m', target: 50 },  // Sustain: 50 users for 10 min
        { duration: '2m', target: 0 },    // Cool down
      ],
    },
    
    // Spike test (sudden traffic burst)
    spike_test: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '10s', target: 100 },  // Instant spike
        { duration: '1m', target: 100 },   // Hold
        { duration: '10s', target: 0 },    // Instant drop
      ],
    },
  },
  
  thresholds: {
    http_req_duration: ['p(99)<1000'],  // P99 < 1s
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
    errors: ['rate<0.01'],
  },
}

export function setup() {
  // Login once, share token across all VUs
  const response = http.post(`${__ENV.BASE_URL}/api/v1/auth/login`, JSON.stringify({
    email: 'loadtest@example.com',
    password: 'loadtest123',
  }), { headers: { 'Content-Type': 'application/json' } })
  
  return { token: response.json('data.accessToken') }
}

export default function({ token }) {
  const headers = { 
    Authorization: `Bearer ${token}`, 
    'Content-Type': 'application/json' 
  }
  
  const startTime = Date.now()
  
  const response = http.post(
    `${__ENV.BASE_URL}/api/v1/bookings`,
    JSON.stringify({ serviceId: 'service_123', scheduledAt: '2025-07-01T10:00:00Z' }),
    { headers }
  )
  
  bookingDuration.add(Date.now() - startTime)
  
  check(response, {
    'status is 201': (r) => r.status === 201,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has booking ID': (r) => r.json('data.id') !== undefined,
  })
  
  errorRate.add(response.status !== 201)
  
  sleep(1)  // 1 second between requests per user
}

// Run: k6 run --env BASE_URL=http://localhost:3000 booking_load_test.js
// Run in cloud: k6 cloud booking_load_test.js
```

---

## 7. Security Testing

```bash
# OWASP ZAP - automated DAST scanning
# Run as part of CI against staging environment

docker run -t owasp/zap2docker-stable zap-baseline.py \
  -t https://staging.example.com \
  -g gen.conf \
  -r zap-report.html \
  -x zap-report.xml

# Check report for:
# - SQL Injection
# - XSS vulnerabilities
# - Security headers missing
# - Information disclosure

# Nikto - web server vulnerability scanner
nikto -h https://staging.example.com -output nikto-report.html

# SSL/TLS configuration check
testssl.sh https://staging.example.com

# API security testing with OWASP ZAP API scan
zap-api-scan.py -t https://api.staging.example.com/openapi.json -f openapi
```

---

## 8. Test Data Management

```typescript
// Test factories with Faker.js — generate realistic test data
import { faker } from '@faker-js/faker'

// Factory pattern: buildX() for objects, createX() for DB-persisted records
const buildUser = (overrides?: Partial<User>): Omit<User, 'id' | 'createdAt'> => ({
  email: faker.internet.email(),
  name: faker.person.fullName(),
  phone: faker.phone.number(),
  role: 'user',
  ...overrides,
})

const createUser = async (overrides?: Partial<User>): Promise<User> => {
  const [user] = await db.insert(users).values(buildUser(overrides)).returning()
  return user
}

const buildBooking = (clientId: string, serviceId: string, overrides?: Partial<Booking>) => ({
  clientId,
  serviceId,
  scheduledAt: faker.date.soon({ days: 30 }),  // Within next 30 days
  status: 'pending' as const,
  notes: faker.lorem.sentence(),
  ...overrides,
})

// Usage in tests
it('should list user bookings', async () => {
  const user = await createUser({ role: 'admin' })
  const service = await createService()
  
  // Create 3 bookings for this user
  await Promise.all([
    createBooking(user.id, service.id, { status: 'confirmed' }),
    createBooking(user.id, service.id, { status: 'pending' }),
    createBooking(user.id, service.id, { status: 'cancelled' }),
  ])
  
  const bookings = await repo.findByClientId(user.id)
  expect(bookings).toHaveLength(3)
})
```

### Database Seeding

```typescript
// seed.ts — for development and staging environments
export async function seed() {
  console.log('🌱 Seeding database...')
  
  // Idempotent: upsert so re-running seed doesn't fail
  const [admin] = await db
    .insert(users)
    .values({
      id: 'user_admin_001',  // Fixed ID for seed data
      email: 'admin@studio.com',
      name: 'Studio Admin',
      role: 'admin',
    })
    .onConflictDoUpdate({ target: users.id, set: { updatedAt: new Date() } })
    .returning()
  
  const serviceNames = ['Massagem Relaxante', 'Massagem Terapêutica', 'Reflexologia']
  
  await db
    .insert(services)
    .values(serviceNames.map((name, i) => ({
      id: `service_00${i + 1}`,
      name,
      price: (100 + i * 50) * 100,  // Stored in centavos
      durationMin: 60,
    })))
    .onConflictDoNothing()
  
  console.log('✅ Seed complete')
}
```

---

## 9. CI/CD Testing Pipeline

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # ─── Lint and Type Check ───────────────────────────────────────────────────
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck
  
  # ─── Unit Tests ────────────────────────────────────────────────────────────
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run test:unit --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
  
  # ─── Integration Tests ─────────────────────────────────────────────────────
  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ['5432:5432']
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports: ['6379:6379']
    
    env:
      DATABASE_URL: postgresql://test:test@localhost:5432/test_db
      REDIS_URL: redis://localhost:6379
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run db:migrate
      - run: npm run test:integration
  
  # ─── E2E Tests ─────────────────────────────────────────────────────────────
  e2e:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]  # Only run if other tests pass
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm run build
      - run: npm run test:e2e
        env:
          BASE_URL: http://localhost:3000
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
  
  # ─── Security Scan ─────────────────────────────────────────────────────────
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm audit --audit-level=high
      - uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
```

---

## 10. Coverage Strategy

```
Coverage targets:
  Statements: 80%+
  Branches:   75%+
  Functions:  85%+
  Lines:      80%+

What matters more than % coverage:
  1. Business logic paths covered
  2. Error/edge cases tested
  3. Boundary conditions validated
  4. Integration points verified

What 100% coverage does NOT guarantee:
  - Correctness (tests can pass with wrong assertions)
  - Absence of race conditions
  - Performance is acceptable
  - Security vulnerabilities
  - Integration with real external services

Coverage exclusions (don't count these):
  - Generated code (protobuf, GraphQL types)
  - Type definitions
  - Migration files
  - Configuration files
  - Test helpers themselves
  
# jest.config.ts coverage config
export default {
  coverageThreshold: {
    global: {
      statements: 80,
      branches: 75,
      functions: 85,
      lines: 80,
    },
    // Per-file thresholds for critical modules
    './src/payments/': {
      statements: 95,  // Payment code: higher bar
    },
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
    '!src/**/migrations/**',
    '!src/**/*.generated.ts',
    '!src/testing/**',
  ],
}
```
