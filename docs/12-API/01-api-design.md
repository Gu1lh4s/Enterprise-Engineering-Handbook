# API Design

> **Category:** API
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [REST API Design Principles](#1-rest-api-design-principles)
2. [API Versioning](#2-api-versioning)
3. [Error Handling](#3-error-handling)
4. [Rate Limiting](#4-rate-limiting)
5. [Authentication and Authorization](#5-authentication-and-authorization)
6. [OpenAPI / Swagger Spec](#6-openapi--swagger-spec)
7. [GraphQL Design](#7-graphql-design)
8. [gRPC Design](#8-grpc-design)
9. [Webhooks](#9-webhooks)
10. [API Security](#10-api-security)
11. [API Gateway Patterns](#11-api-gateway-patterns)
12. [Contract Testing](#12-contract-testing)

---

## 1. REST API Design Principles

### Resource Design

```
Resources = nouns (not verbs)
HTTP methods = verbs (GET, POST, PUT, PATCH, DELETE)

WRONG:
  POST /createUser
  GET  /getUserById?id=123
  POST /deleteUser/123
  POST /updateUserEmail

CORRECT:
  POST   /users                    → Create user
  GET    /users                    → List users
  GET    /users/{id}               → Get user
  PUT    /users/{id}               → Replace user (full update)
  PATCH  /users/{id}               → Update user (partial update)
  DELETE /users/{id}               → Delete user

Nested resources (use sparingly — max 2 levels):
  GET    /users/{userId}/bookings          → User's bookings
  POST   /users/{userId}/bookings          → Create booking for user
  GET    /users/{userId}/bookings/{bookingId} → Specific booking

For complex operations (can't fit as resource action):
  POST /bookings/{id}/cancel               → Cancel booking (action on resource)
  POST /payments/{id}/refund               → Refund payment
  POST /documents/{id}/publish             → Publish document
```

### HTTP Methods Semantics

| Method | Idempotent | Safe | Body | Response |
|---|---|---|---|---|
| GET | Yes | Yes | No | Resource |
| POST | No | No | Yes | Created resource or result |
| PUT | Yes | No | Yes | Updated resource |
| PATCH | No* | No | Yes | Updated resource |
| DELETE | Yes | No | No | Empty or confirmation |

```
Safe: does not change server state (GET is always safe)
Idempotent: same request N times = same result as once

PUT /users/123 with body {name: "Alice"} → always produces user with name "Alice"
DELETE /users/123 → first call deletes; subsequent calls: 404 (but server state same: no user)

*PATCH can be made idempotent with conditional If-Match header (ETags)
```

### Query Parameters

```
Filtering:
  GET /bookings?status=pending
  GET /bookings?status=pending&status=confirmed   → multiple values: status IN (pending, confirmed)
  GET /bookings?scheduled_after=2025-01-01T00:00:00Z
  GET /services?price_min=50&price_max=200

Sorting:
  GET /bookings?sort=scheduled_at:asc
  GET /bookings?sort=-created_at                  → minus prefix for descending
  GET /bookings?sort=status:asc,scheduled_at:desc → multiple sort fields

Pagination (cursor-based):
  GET /bookings?limit=20
  GET /bookings?limit=20&after=cursor_opaque_value

Field selection (GraphQL-style for REST):
  GET /users?fields=id,name,email                → Only return specified fields
  GET /bookings?include=client,service           → Include related resources

Search:
  GET /services?q=massagem+relaxante
```

### Response Structure

```typescript
// Successful single resource
{
  "data": {
    "id": "booking_123",
    "status": "confirmed",
    "scheduledAt": "2025-06-27T10:00:00Z",
    "client": {
      "id": "user_456",
      "name": "Maria Silva"
    }
  }
}

// Successful collection
{
  "data": [
    { "id": "booking_123", "status": "confirmed" },
    { "id": "booking_124", "status": "pending" }
  ],
  "meta": {
    "total": 47,
    "hasMore": true,
    "nextCursor": "eyJpZCI6ImJvb2tpbmdfMTI0In0"
  }
}

// Error response
{
  "error": {
    "code": "BOOKING_NOT_FOUND",
    "message": "Booking with ID booking_999 not found",
    "status": 404,
    "details": [],          // Additional context (field errors, etc.)
    "requestId": "req_abc123",  // Correlate with server logs
    "docsUrl": "https://docs.example.com/errors/BOOKING_NOT_FOUND"
  }
}

// Validation error (422)
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request body",
    "status": 422,
    "details": [
      { "field": "scheduledAt", "message": "Must be a future date" },
      { "field": "duration_min", "message": "Must be between 30 and 480" }
    ],
    "requestId": "req_abc123"
  }
}
```

---

## 2. API Versioning

### Versioning Strategies

```
1. URI Path Versioning (most common, most explicit)
   /v1/users
   /v2/users
   
   Pro: obvious, easy to route, cacheable
   Con: not "pure REST" (resource URI changes)

2. Header Versioning
   GET /users
   Accept: application/vnd.example.v2+json
   API-Version: 2
   
   Pro: clean URIs
   Con: harder to test in browser, harder to cache

3. Query Parameter Versioning
   GET /users?version=2
   
   Pro: easy to test
   Con: easily forgotten, pollutes query string

4. Content Negotiation (most RESTful)
   Accept: application/vnd.myapp+json;version=2
   
   Pro: per-representation versioning
   Con: complex, rarely worth it

Recommendation: URI path versioning (/v1/, /v2/)
```

### Versioning Policy

```
Semantic versioning for APIs:
  /v1.0/ — Major version in URL
  /v1.1/ — Minor version via header (backward compatible)
  
Breaking changes (require new major version):
  - Removing or renaming a field
  - Changing field type
  - Changing authentication method
  - Removing an endpoint
  - Changing error codes
  - Changing behavior of an existing endpoint

Non-breaking changes (safe without new version):
  - Adding new optional fields to response
  - Adding new endpoints
  - Adding new optional query parameters
  - Adding new values to enum (if consumers ignore unknown values)

Deprecation policy:
  1. Announce deprecation with deprecation date
  2. Add Deprecation header to responses: Deprecation: Sun, 01 Jun 2026 00:00:00 GMT
  3. Add Sunset header: Sunset: Mon, 01 Dec 2026 00:00:00 GMT
  4. Notify API consumers via email/changelog
  5. Keep deprecated version alive for minimum 12 months after announcement
  6. Sunset: return 410 Gone after cutoff date
```

---

## 3. Error Handling

### Standard Error Codes

```typescript
// Application error codes (not HTTP status codes — more specific)
export const ERROR_CODES = {
  // Auth
  UNAUTHORIZED: 'UNAUTHORIZED',
  FORBIDDEN: 'FORBIDDEN',
  TOKEN_EXPIRED: 'TOKEN_EXPIRED',
  INVALID_TOKEN: 'INVALID_TOKEN',
  
  // Resource
  NOT_FOUND: 'NOT_FOUND',
  BOOKING_NOT_FOUND: 'BOOKING_NOT_FOUND',
  USER_NOT_FOUND: 'USER_NOT_FOUND',
  
  // Validation
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  INVALID_DATE: 'INVALID_DATE',
  PAST_DATE: 'PAST_DATE',
  SLOT_UNAVAILABLE: 'SLOT_UNAVAILABLE',
  
  // Rate limiting
  RATE_LIMIT_EXCEEDED: 'RATE_LIMIT_EXCEEDED',
  
  // Server
  INTERNAL_ERROR: 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE: 'SERVICE_UNAVAILABLE',
  DEPENDENCY_ERROR: 'DEPENDENCY_ERROR',
} as const

// Error factory
function createApiError(code: keyof typeof ERROR_CODES, message: string, status: number, details?: object[]) {
  return {
    error: {
      code,
      message,
      status,
      details: details ?? [],
      requestId: asyncLocalStorage.getStore()?.traceId,
    }
  }
}

// Express error handler
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  // Log the error (with trace ID)
  logger.error({ err, traceId: req.traceId, url: req.url }, 'Request failed')
  
  // Operational errors (expected, safe to expose)
  if (err instanceof AppError) {
    return res.status(err.statusCode).json(
      createApiError(err.code, err.message, err.statusCode, err.details)
    )
  }
  
  // Programming errors (unexpected — never expose internals)
  // In production: generic message; in development: stack trace
  const isDev = process.env.NODE_ENV === 'development'
  
  return res.status(500).json({
    error: {
      code: 'INTERNAL_ERROR',
      message: isDev ? err.message : 'An unexpected error occurred',
      status: 500,
      requestId: req.traceId,
      ...(isDev && { stack: err.stack }),
    }
  })
})
```

---

## 4. Rate Limiting

```typescript
// Standard rate limit headers (IETF RFC 6585, draft-ietf-httpapi-ratelimit-headers)
interface RateLimitHeaders {
  'X-RateLimit-Limit': string      // Request limit per window
  'X-RateLimit-Remaining': string  // Remaining requests in window
  'X-RateLimit-Reset': string      // Unix timestamp when window resets
  'X-RateLimit-Policy': string     // Policy name (e.g., "100;w=60")
  'Retry-After'?: string           // Seconds to wait (on 429 only)
}

// Rate limit tiers
const RATE_LIMITS = {
  anonymous:      { requests: 20,   window: 60 },    // 20/min for unauthenticated
  authenticated:  { requests: 100,  window: 60 },    // 100/min for users
  premium:        { requests: 1000, window: 60 },    // 1000/min for premium
  api_key:        { requests: 5000, window: 60 },    // 5000/min for API key users
  admin:          { requests: 10000, window: 60 },   // Practically unlimited for admins
}

// Rate limit middleware with tier selection
function rateLimiter() {
  return async (req: Request, res: Response, next: NextFunction) => {
    const tier = req.user?.tier ?? 'anonymous'
    const config = RATE_LIMITS[tier]
    const identifier = req.user?.id ?? req.ip
    
    const { allowed, remaining, resetAt, total } = await checkRateLimit(
      identifier, config.requests, config.window * 1000
    )
    
    // Always include rate limit headers
    res.setHeader('X-RateLimit-Limit', String(total))
    res.setHeader('X-RateLimit-Remaining', String(remaining))
    res.setHeader('X-RateLimit-Reset', String(Math.ceil(resetAt / 1000)))
    
    if (!allowed) {
      res.setHeader('Retry-After', String(Math.ceil((resetAt - Date.now()) / 1000)))
      return res.status(429).json(
        createApiError('RATE_LIMIT_EXCEEDED', 
          `Rate limit exceeded. Try again in ${Math.ceil((resetAt - Date.now()) / 1000)} seconds`, 
          429)
      )
    }
    
    next()
  }
}
```

---

## 5. Authentication and Authorization

```typescript
// Middleware stack for protected routes
const protectedRoute = [
  authenticate,           // Verify JWT, set req.user
  authorize(['user', 'admin']),  // Check role
  rateLimiter(),          // Apply rate limits based on user tier
]

// Authenticate middleware
async function authenticate(req: Request, res: Response, next: NextFunction) {
  const authHeader = req.headers.authorization
  
  if (!authHeader?.startsWith('Bearer ')) {
    return res.status(401).json(createApiError('UNAUTHORIZED', 'Authorization header required', 401))
  }
  
  const token = authHeader.slice(7)
  
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET!, { algorithms: ['HS256'] }) as JwtPayload
    req.user = { id: payload.sub!, email: payload.email, role: payload.role }
    next()
  } catch (err) {
    if (err instanceof jwt.TokenExpiredError) {
      return res.status(401).json(createApiError('TOKEN_EXPIRED', 'Access token expired', 401))
    }
    return res.status(401).json(createApiError('INVALID_TOKEN', 'Invalid access token', 401))
  }
}

// Authorize middleware
function authorize(roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json(createApiError('FORBIDDEN', 'Insufficient permissions', 403))
    }
    next()
  }
}
```

---

## 6. OpenAPI / Swagger Spec

```yaml
# openapi.yaml
openapi: "3.1.0"
info:
  title: Bookings API
  version: "1.0.0"
  description: Studio booking management API
  contact:
    email: api@example.com
  license:
    name: MIT
    url: https://opensource.org/licenses/MIT

servers:
  - url: https://api.example.com/v1
    description: Production
  - url: https://api.staging.example.com/v1
    description: Staging

security:
  - BearerAuth: []  # Default: all endpoints require auth

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  
  schemas:
    Booking:
      type: object
      required: [id, clientId, serviceId, scheduledAt, status, createdAt]
      properties:
        id:
          type: string
          format: uuid
          example: "550e8400-e29b-41d4-a716-446655440000"
        clientId:
          type: string
          format: uuid
        serviceId:
          type: string
          format: uuid
        scheduledAt:
          type: string
          format: date-time
        status:
          type: string
          enum: [pending, confirmed, cancelled, completed]
        notes:
          type: string
          nullable: true
        createdAt:
          type: string
          format: date-time
    
    Error:
      type: object
      required: [error]
      properties:
        error:
          type: object
          required: [code, message, status]
          properties:
            code: { type: string }
            message: { type: string }
            status: { type: integer }
            details:
              type: array
              items:
                type: object
                properties:
                  field: { type: string }
                  message: { type: string }
            requestId: { type: string }
  
  parameters:
    BookingId:
      name: bookingId
      in: path
      required: true
      schema:
        type: string
        format: uuid
    
    Limit:
      name: limit
      in: query
      schema:
        type: integer
        minimum: 1
        maximum: 100
        default: 20
    
    After:
      name: after
      in: query
      description: Opaque cursor for pagination
      schema:
        type: string
  
  responses:
    Unauthorized:
      description: Authentication required
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }
    NotFound:
      description: Resource not found
      content:
        application/json:
          schema: { $ref: '#/components/schemas/Error' }

paths:
  /bookings:
    get:
      summary: List bookings
      operationId: listBookings
      tags: [Bookings]
      parameters:
        - $ref: '#/components/parameters/Limit'
        - $ref: '#/components/parameters/After'
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, confirmed, cancelled, completed]
      responses:
        "200":
          description: List of bookings
          content:
            application/json:
              schema:
                type: object
                properties:
                  data:
                    type: array
                    items: { $ref: '#/components/schemas/Booking' }
                  meta:
                    type: object
                    properties:
                      hasMore: { type: boolean }
                      nextCursor: { type: string, nullable: true }
        "401": { $ref: '#/components/responses/Unauthorized' }
    
    post:
      summary: Create a booking
      operationId: createBooking
      tags: [Bookings]
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [serviceId, scheduledAt]
              properties:
                serviceId:
                  type: string
                  format: uuid
                scheduledAt:
                  type: string
                  format: date-time
                notes:
                  type: string
                  maxLength: 1000
      responses:
        "201":
          description: Booking created
          content:
            application/json:
              schema:
                type: object
                properties:
                  data: { $ref: '#/components/schemas/Booking' }
        "422":
          description: Validation error
          content:
            application/json:
              schema: { $ref: '#/components/schemas/Error' }
```

---

## 7. GraphQL Design

```typescript
// Schema-first GraphQL design
const typeDefs = gql`
  type Query {
    me: User
    booking(id: ID!): Booking
    bookings(filter: BookingFilter, pagination: PaginationInput): BookingConnection!
    services(category: ServiceCategory): [Service!]!
  }
  
  type Mutation {
    createBooking(input: CreateBookingInput!): CreateBookingPayload!
    cancelBooking(id: ID!): CancelBookingPayload!
    updateProfile(input: UpdateProfileInput!): UpdateProfilePayload!
  }
  
  type Subscription {
    bookingUpdated(bookingId: ID!): BookingUpdatedEvent!
  }
  
  type User {
    id: ID!
    email: String!
    name: String!
    phone: String
    bookings(filter: BookingFilter, pagination: PaginationInput): BookingConnection!
    # avatarUrl: String  # Lazy field — only fetched when requested
  }
  
  type Booking {
    id: ID!
    client: User!
    service: Service!
    scheduledAt: DateTime!
    status: BookingStatus!
    notes: String
    createdAt: DateTime!
  }
  
  # Relay-spec connection pattern (pagination)
  type BookingConnection {
    edges: [BookingEdge!]!
    pageInfo: PageInfo!
    totalCount: Int!
  }
  
  type BookingEdge {
    node: Booking!
    cursor: String!
  }
  
  type PageInfo {
    hasNextPage: Boolean!
    hasPreviousPage: Boolean!
    startCursor: String
    endCursor: String
  }
  
  # Mutation payload pattern (includes errors)
  type CreateBookingPayload {
    booking: Booking
    errors: [UserError!]!
  }
  
  type UserError {
    field: [String!]
    message: String!
    code: String!
  }
  
  input CreateBookingInput {
    serviceId: ID!
    scheduledAt: DateTime!
    notes: String
  }
  
  enum BookingStatus {
    PENDING
    CONFIRMED
    CANCELLED
    COMPLETED
  }
`

// Resolvers with DataLoader (N+1 prevention)
import DataLoader from 'dataloader'

const createUserLoader = () => new DataLoader<string, User>(async (ids) => {
  const users = await db.query('SELECT * FROM users WHERE id = ANY($1)', [ids])
  return ids.map(id => users.rows.find(u => u.id === id) ?? null)
})

const resolvers = {
  Query: {
    booking: async (_: any, { id }: { id: string }, ctx: Context) => {
      return ctx.loaders.bookings.load(id)
    },
  },
  
  Booking: {
    client: async (booking: Booking, _: any, ctx: Context) => {
      return ctx.loaders.users.load(booking.clientId)  // Batched!
    },
    service: async (booking: Booking, _: any, ctx: Context) => {
      return ctx.loaders.services.load(booking.serviceId)  // Batched!
    },
  },
  
  Mutation: {
    createBooking: async (_: any, { input }: any, ctx: Context) => {
      if (!ctx.user) throw new GraphQLError('Unauthorized', { extensions: { code: 'UNAUTHORIZED' } })
      
      try {
        const booking = await bookingService.create({ ...input, clientId: ctx.user.id })
        return { booking, errors: [] }
      } catch (err) {
        if (err instanceof ValidationError) {
          return { booking: null, errors: err.details }
        }
        throw err
      }
    }
  }
}
```

---

## 8. gRPC Design

```protobuf
// bookings.proto
syntax = "proto3";

package com.example.bookings.v1;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

service BookingService {
  // Unary RPC (most common)
  rpc GetBooking(GetBookingRequest) returns (Booking);
  rpc CreateBooking(CreateBookingRequest) returns (Booking);
  rpc CancelBooking(CancelBookingRequest) returns (Booking);
  
  // Server streaming (client subscribes to updates)
  rpc StreamBookingUpdates(StreamBookingUpdatesRequest) returns (stream BookingUpdate);
  
  // Client streaming (upload multiple items)
  rpc ImportBookings(stream ImportBookingItem) returns (ImportBookingsResponse);
  
  // Bidirectional streaming
  rpc ChatWithSupport(stream SupportMessage) returns (stream SupportMessage);
}

message GetBookingRequest {
  string booking_id = 1;
}

message CreateBookingRequest {
  string service_id = 1;
  google.protobuf.Timestamp scheduled_at = 2;
  string notes = 3;  // optional — use optional keyword in proto3 to distinguish empty vs unset
}

message Booking {
  string id = 1;
  string client_id = 2;
  string service_id = 3;
  google.protobuf.Timestamp scheduled_at = 4;
  BookingStatus status = 5;
  string notes = 6;
  google.protobuf.Timestamp created_at = 7;
}

enum BookingStatus {
  BOOKING_STATUS_UNSPECIFIED = 0;  // Always define 0 as UNSPECIFIED for forward compat
  BOOKING_STATUS_PENDING = 1;
  BOOKING_STATUS_CONFIRMED = 2;
  BOOKING_STATUS_CANCELLED = 3;
  BOOKING_STATUS_COMPLETED = 4;
}
```

```typescript
// gRPC server implementation (Node.js)
import * as grpc from '@grpc/grpc-js'
import * as protoLoader from '@grpc/proto-loader'

const packageDef = protoLoader.loadSync('bookings.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
})

const bookingProto = grpc.loadPackageDefinition(packageDef) as any

const server = new grpc.Server()
server.addService(bookingProto.com.example.bookings.v1.BookingService.service, {
  GetBooking: async (call: grpc.ServerUnaryCall<any, any>, callback: grpc.sendUnaryData<any>) => {
    try {
      const booking = await bookingService.findById(call.request.booking_id)
      if (!booking) {
        callback({
          code: grpc.status.NOT_FOUND,
          message: `Booking ${call.request.booking_id} not found`,
        })
        return
      }
      callback(null, booking)
    } catch (err) {
      callback({ code: grpc.status.INTERNAL, message: 'Internal error' })
    }
  },
})
```

---

## 9. Webhooks

```typescript
// Webhook delivery with HMAC-SHA256 signature verification

// Sender: sign each webhook
function signWebhookPayload(payload: object, secret: string): string {
  const timestamp = Math.floor(Date.now() / 1000)
  const payloadString = JSON.stringify(payload)
  const signedContent = `${timestamp}.${payloadString}`
  
  const signature = crypto
    .createHmac('sha256', secret)
    .update(signedContent, 'utf8')
    .digest('hex')
  
  return `t=${timestamp},v1=${signature}`
}

// Delivery with retry
async function deliverWebhook(endpoint: WebhookEndpoint, event: WebhookEvent): Promise<void> {
  const payload = {
    id: randomUUID(),
    type: event.type,
    created: Math.floor(Date.now() / 1000),
    data: event.data,
  }
  
  const signature = signWebhookPayload(payload, endpoint.signingSecret)
  
  for (let attempt = 1; attempt <= 5; attempt++) {
    try {
      const response = await fetch(endpoint.url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'X-Webhook-Signature': signature,
          'X-Webhook-ID': payload.id,
          'X-Webhook-Attempt': String(attempt),
        },
        body: JSON.stringify(payload),
        signal: AbortSignal.timeout(10_000),  // 10s timeout per attempt
      })
      
      if (response.ok) return
      
      // Non-2xx: schedule retry (exponential backoff)
      await sleep(1000 * 2 ** attempt)  // 2s, 4s, 8s, 16s, 32s
      
    } catch (err) {
      if (attempt === 5) {
        await markEndpointFailed(endpoint.id)
        await notifyEndpointOwner(endpoint, err)
      }
    }
  }
}

// Receiver: verify webhook signature
app.post('/webhooks/stripe', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-webhook-signature'] as string
  const [tPart, v1Part] = signature.split(',')
  const timestamp = tPart.split('=')[1]
  const receivedSig = v1Part.split('=')[1]
  
  // Prevent replay attacks: reject webhooks older than 5 minutes
  if (Math.abs(Date.now() / 1000 - Number(timestamp)) > 300) {
    return res.status(400).json({ error: 'Timestamp too old' })
  }
  
  // Verify signature
  const expectedSig = crypto
    .createHmac('sha256', process.env.WEBHOOK_SECRET!)
    .update(`${timestamp}.${req.body.toString()}`, 'utf8')
    .digest('hex')
  
  // Constant-time comparison (prevent timing attacks)
  if (!crypto.timingSafeEqual(Buffer.from(receivedSig), Buffer.from(expectedSig))) {
    return res.status(401).json({ error: 'Invalid signature' })
  }
  
  // Process event asynchronously (respond 200 immediately to avoid timeout)
  const event = JSON.parse(req.body.toString())
  setImmediate(() => processWebhookEvent(event))
  
  res.json({ received: true })
})
```

---

## 10. API Security

```typescript
// Security headers for API responses
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff')
  res.setHeader('X-Frame-Options', 'DENY')
  res.setHeader('Cache-Control', 'no-store')  // Don't cache API responses
  res.setHeader('Content-Type', 'application/json')  // Always set content type
  next()
})

// Input validation with Zod
import { z } from 'zod'

const CreateBookingSchema = z.object({
  serviceId: z.string().uuid('Invalid service ID'),
  scheduledAt: z.string().datetime({ message: 'Must be ISO 8601 datetime' })
    .refine(d => new Date(d) > new Date(), { message: 'Must be a future date' }),
  notes: z.string().max(1000, 'Notes too long').optional(),
})

app.post('/api/bookings', authenticate, async (req, res) => {
  const parsed = CreateBookingSchema.safeParse(req.body)
  
  if (!parsed.success) {
    return res.status(422).json(createApiError(
      'VALIDATION_ERROR',
      'Invalid request body',
      422,
      parsed.error.errors.map(e => ({ field: e.path.join('.'), message: e.message }))
    ))
  }
  
  const booking = await bookingService.create({ ...parsed.data, clientId: req.user.id })
  res.status(201).json({ data: booking })
})

// Mass assignment prevention via schema
// Only parsed.data is passed to service — no extra fields from request body accepted
```

---

## 11. API Gateway Patterns

```yaml
# Kong API Gateway configuration
# Handles: auth, rate limiting, logging, routing BEFORE reaching services

services:
  - name: bookings-api
    url: http://booking-service:3000
    
plugins:
  # Auth: JWT verification at gateway (services trust gateway header)
  - name: jwt
    config:
      secret_is_base64: false
      key_claim_name: kid
  
  # Rate limiting per consumer
  - name: rate-limiting
    config:
      minute: 100
      hour: 1000
      policy: redis
      redis_host: redis
  
  # Request logging
  - name: http-log
    config:
      http_endpoint: http://log-aggregator:3000/kong
  
  # CORS
  - name: cors
    config:
      origins: ["https://app.example.com", "https://admin.example.com"]
      methods: [GET, POST, PUT, PATCH, DELETE, OPTIONS]
      headers: [Authorization, Content-Type, X-Request-ID]
      credentials: true
  
  # Request size limit
  - name: request-size-limiting
    config:
      allowed_payload_size: 10  # 10 MB

routes:
  - name: public-routes
    paths: [/api/v1/services, /api/v1/auth]
    strip_path: false
    plugins:
      # No auth required for public routes
      - name: rate-limiting
        config:
          minute: 20  # Stricter rate limit for unauthenticated
```

---

## 12. Contract Testing

```typescript
// Pact — consumer-driven contract testing
// Consumer defines what it expects from provider
// Provider verifies it can satisfy all consumer contracts

// Consumer test (in consumer's codebase)
import { PactV4, MatchersV3 } from '@pact-foundation/pact'

const provider = new PactV4({
  consumer: 'BookingUI',
  provider: 'BookingAPI',
})

describe('Booking API Contract', () => {
  it('should get a booking by ID', async () => {
    await provider
      .addInteraction()
      .given('a booking with ID booking_123 exists')
      .uponReceiving('a request to get booking booking_123')
      .withRequest({
        method: 'GET',
        path: '/api/v1/bookings/booking_123',
        headers: { Authorization: MatchersV3.like('Bearer token') }
      })
      .willRespondWith({
        status: 200,
        body: {
          data: {
            id: 'booking_123',
            status: MatchersV3.oneOf(['pending', 'confirmed', 'cancelled', 'completed']),
            scheduledAt: MatchersV3.timestamp('yyyy-MM-dd\'T\'HH:mm:ssZ', '2025-06-27T10:00:00Z'),
          }
        }
      })
      .executeTest(async (mockServer) => {
        const client = new BookingApiClient(mockServer.url)
        const booking = await client.getBooking('booking_123')
        expect(booking.data.id).toBe('booking_123')
      })
    
    // Pact file published to broker
    // Provider CI verifies against all consumer contracts
  })
})
```
