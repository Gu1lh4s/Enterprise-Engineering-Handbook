# System Design Fundamentals

> **Category:** Architecture
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Requirements Engineering](#1-requirements-engineering)
2. [Scalability Concepts](#2-scalability-concepts)
3. [Data Storage](#3-data-storage)
4. [Caching](#4-caching)
5. [Load Balancing](#5-load-balancing)
6. [Message Queues](#6-message-queues)
7. [CDN and Global Distribution](#7-cdn-and-global-distribution)
8. [API Design](#8-api-design)
9. [Consistency Models](#9-consistency-models)
10. [CAP Theorem](#10-cap-theorem)
11. [System Design Interview Framework](#11-system-design-interview-framework)
12. [Reference Architectures](#12-reference-architectures)

---

## 1. Requirements Engineering

Before any design decision, establish requirements. Underspecified requirements lead to over-engineered or under-designed systems.

### Functional Requirements

What the system must **do**:

```
1. Users can post tweets (text ≤280 chars, images, videos)
2. Users can follow other users
3. Timeline shows posts from followed users in reverse-chronological order
4. Search by keyword and hashtag
5. Push notifications for mentions, likes, follows
```

### Non-Functional Requirements

How the system must **behave**:

```
Scale:
- DAU (Daily Active Users): 100M
- Read:Write ratio: 100:1 (read-heavy)
- Tweets per second: 5,000 (write), 500,000 (read)
- Timeline fetch: 100 tweets × 100M DAU = billions of reads/day

Latency:
- Timeline load: p99 < 300ms
- Search results: p99 < 500ms
- Notification delivery: < 5 seconds

Availability:
- 99.99% (52 minutes downtime/year)
- RPO (Recovery Point Objective): 1 hour
- RTO (Recovery Time Objective): 30 minutes

Data:
- Tweet size: ~140 bytes + media references
- Media: ~1MB average photo, 10MB video
- Storage growth: 500GB/day text, 50TB/day media

Consistency:
- Timeline can tolerate eventual consistency (5 second delay acceptable)
- Financial operations require strong consistency
```

### Capacity Estimation (Back-of-Envelope)

```
Twitter-scale example:

Writes:
- 5,000 tweets/sec
- 5,000 × 140 bytes = 700 KB/sec → ~2.5 GB/hour → ~60 GB/day text

Reads:
- 500,000 timeline reads/sec
- Each read fetches 100 tweets = 500,000 × 100 = 50M tweet lookups/sec

Bandwidth:
- Read: 500,000 reads/sec × 100 tweets × 140 bytes = ~7 GB/sec
- This is why caching is critical — cannot serve this from disk

Storage:
- 5 years × 365 × 60 GB/day = ~110 TB for text
- Media: 50 TB/day × 5 years = ~90 PB (needs distributed object storage)

Cache:
- Cache hit ratio target: 95%
- Cache 5% of data handles 95% of reads (80/20 principle applies)
- Memory needed: 110 TB × 5% = 5.5 TB (distributed across cache cluster)
```

---

## 2. Scalability Concepts

### Vertical vs Horizontal Scaling

```
Vertical Scaling (Scale Up):
- Add more CPU, RAM, faster disk to existing server
- Simpler — no distributed system complexity
- Limited by hardware ceiling
- Single point of failure
- Cost: superlinear (doubling RAM doesn't double price)
- Good for: databases that can't be easily sharded, legacy apps

Horizontal Scaling (Scale Out):
- Add more servers; distribute load
- No hard ceiling (add N more servers)
- Requires stateless application layer
- Fault tolerant — N-1 servers still work if 1 fails
- Cost: roughly linear
- Good for: stateless API services, read replicas, caches

Hybrid:
- Vertical scale the database tier (big box with fast SSDs)
- Horizontal scale the application tier (many small, stateless servers)
```

### Stateless Application Design

```typescript
// STATEFUL — bad for horizontal scaling
class SessionHandler {
  private sessions = new Map<string, User>()  // In-memory state
  
  login(user: User): string {
    const id = generateId()
    this.sessions.set(id, user)  // Only this server knows about this session
    return id
  }
  
  // If load balancer routes next request to different server: session not found
}

// STATELESS — scales horizontally
// State lives in Redis (shared across all servers)
async function login(user: User): Promise<string> {
  const sessionId = crypto.randomBytes(32).toString('hex')
  await redis.setex(`session:${sessionId}`, 1800, JSON.stringify(user))
  return sessionId
}

async function getSession(sessionId: string): Promise<User | null> {
  const data = await redis.get(`session:${sessionId}`)
  return data ? JSON.parse(data) : null
}
```

### Database Read Replicas

```
Primary (Read + Write)
      │
      ├── Replica 1 (Read only)
      ├── Replica 2 (Read only)
      └── Replica 3 (Read only)

Replication lag: typically 1-100ms (replication is async)
Read replicas handle: SELECT queries (profile fetch, search, listing)
Primary handles: INSERT/UPDATE/DELETE + reads that require strong consistency
```

### Database Sharding (Horizontal Partitioning)

```sql
-- Without sharding: one large table
-- users table: 1 billion rows — single server bottleneck

-- With sharding: data split across multiple databases
-- Shard key: user_id % 4
-- user_id = 1000 → 1000 % 4 = 0 → shard_0
-- user_id = 1001 → 1001 % 4 = 1 → shard_1
-- user_id = 1002 → 1002 % 4 = 2 → shard_2
-- user_id = 1003 → 1003 % 4 = 3 → shard_3

-- Problems with sharding:
-- 1. Cross-shard JOINs are impossible or extremely slow
-- 2. Resharding (rebalancing) is painful
-- 3. Hot shard: if user 0 is Taylor Swift, shard_0 gets all the load
-- Solution: consistent hashing for better distribution
```

**Consistent Hashing:**
```
Without consistent hashing:
- Add 5th shard → hash changes from user_id%4 to user_id%5
- ~80% of all keys need to move to different shards
- Catastrophic cache/shard invalidation

With consistent hashing:
- Nodes on a "ring"
- Add new node: only 1/N keys move (where N = number of nodes)
- Remove node: only 1/N keys migrate to neighbors
- "Virtual nodes" prevent hot spots by distributing each server's range
```

---

## 3. Data Storage

### Storage Type Selection

| Requirement | Storage Type | Examples |
|---|---|---|
| Relational data, transactions, ACID | RDBMS | PostgreSQL, MySQL, CockroachDB |
| High-speed key-value | In-memory KV | Redis, Memcached |
| Document storage, flexible schema | Document DB | MongoDB, DynamoDB, Firestore |
| Time-series data | TSDB | TimescaleDB, InfluxDB, Prometheus |
| Full-text search | Search Engine | Elasticsearch, OpenSearch |
| Graph relationships | Graph DB | Neo4j, Amazon Neptune |
| Large files, blobs | Object Storage | S3, GCS, Azure Blob |
| Analytics, OLAP | Column Store | BigQuery, Redshift, Snowflake |
| Event streams | Log Store | Kafka, Kinesis |

### PostgreSQL Indexing

```sql
-- Without index: sequential scan — O(n)
-- With index: B-tree lookup — O(log n)

-- Single column index
CREATE INDEX idx_bookings_client_id ON bookings(client_id);

-- Composite index (order matters — leftmost prefix rule)
CREATE INDEX idx_bookings_client_status ON bookings(client_id, status);
-- Efficient for: WHERE client_id = $1, WHERE client_id = $1 AND status = $2
-- NOT efficient for: WHERE status = $1 alone (full scan of the index)

-- Partial index (smaller, faster for common queries)
CREATE INDEX idx_pending_bookings ON bookings(scheduled_at)
WHERE status = 'pending';  -- Only indexes pending bookings

-- GIN index for full-text search
CREATE INDEX idx_services_search ON services USING GIN(to_tsvector('english', title || ' ' || description));
SELECT * FROM services WHERE to_tsvector('english', title || ' ' || description) @@ to_tsquery('english', 'massage');

-- Index for UUID primary keys (UUIDv7 > UUIDv4 for insert performance)
-- UUIDv4: random → random index insertions → B-tree fragmentation
-- UUIDv7: time-ordered → append to end of index → excellent performance

-- EXPLAIN ANALYZE to check index usage
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM bookings WHERE client_id = 'user_123';
-- Look for: "Index Scan" (good) vs "Seq Scan" (check if index missing or small table)
```

### Database Connection Pooling

```typescript
// Without pooling: each request opens a new DB connection
// PostgreSQL default max_connections = 100
// 1,000 requests/sec × connection overhead = connection exhaustion

// With PgBouncer (connection pooler in front of PostgreSQL):
// 10,000 app connections → 50 PostgreSQL connections
// PgBouncer handles connection queue, reuse

// Node.js with pg connection pool
import { Pool } from 'pg'

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,                  // Maximum connections in pool
  idleTimeoutMillis: 30000, // Close idle connections after 30s
  connectionTimeoutMillis: 2000,  // Fail fast if pool exhausted
})

// Pool automatically manages connections — never call client.connect() directly
const result = await pool.query('SELECT * FROM bookings WHERE id = $1', [id])
```

---

## 4. Caching

### Cache Strategies

**Cache-Aside (Lazy Loading):**
```typescript
// Application manages cache explicitly
// Read: check cache → if miss, read DB → populate cache
// Write: write to DB, invalidate cache

async function getUser(userId: string): Promise<User> {
  // 1. Check cache
  const cached = await redis.get(`user:${userId}`)
  if (cached) return JSON.parse(cached)
  
  // 2. Cache miss — read from DB
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId])
  
  // 3. Populate cache with TTL
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user))
  
  return user
}

async function updateUser(userId: string, updates: Partial<User>): Promise<User> {
  const user = await db.update(userId, updates)
  
  // Invalidate cache — next read will fetch from DB
  await redis.del(`user:${userId}`)
  
  return user
}
```

**Write-Through:**
```typescript
// Write to cache AND DB simultaneously
// Reads always hit cache — no cold start
// More complex; cache must be writable

async function updateUser(userId: string, updates: Partial<User>): Promise<User> {
  const user = await db.update(userId, updates)
  
  // Update cache immediately
  await redis.setex(`user:${userId}`, 3600, JSON.stringify(user))
  
  return user
}
```

**Read-Through:**
```
Cache sits in front of DB
Application only talks to cache
Cache fetches from DB on miss automatically
Good for: read-heavy workloads, managed cache solutions (Elasticache)
```

### Cache Eviction Policies

| Policy | Algorithm | Best For |
|---|---|---|
| LRU (Least Recently Used) | Evict oldest accessed | General purpose |
| LFU (Least Frequently Used) | Evict least accessed | Hot spot protection |
| TTL (Time To Live) | Evict after expiry | Time-sensitive data |
| FIFO | Evict oldest inserted | Simple queues |

### Cache Problems and Solutions

**Cache Stampede (Thundering Herd):**
```typescript
// Problem: TTL expires → 1000 requests all miss cache simultaneously → 1000 DB queries

// Solution 1: Probabilistic early expiration (PER)
async function getWithJitter(key: string, ttl: number, fetchFn: () => Promise<any>) {
  const data = await redis.get(key)
  if (data) {
    const { value, expiresAt } = JSON.parse(data)
    
    // Probabilistically recompute before TTL expires
    const timeLeft = expiresAt - Date.now()
    const shouldRefresh = timeLeft < ttl * 0.1 * Math.random()  // Last 10% of TTL
    
    if (shouldRefresh) {
      fetchFn().then(fresh => redis.setex(key, ttl, JSON.stringify({ value: fresh, expiresAt: Date.now() + ttl * 1000 })))
    }
    
    return value
  }
  
  // Cache miss
  const value = await fetchFn()
  await redis.setex(key, ttl, JSON.stringify({ value, expiresAt: Date.now() + ttl * 1000 }))
  return value
}

// Solution 2: Mutex / distributed lock
async function getWithLock(key: string, fetchFn: () => Promise<any>) {
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)
  
  const lockKey = `lock:${key}`
  const lockAcquired = await redis.set(lockKey, '1', 'EX', 10, 'NX')
  
  if (!lockAcquired) {
    // Another process is fetching — wait and retry
    await sleep(50)
    return getWithLock(key, fetchFn)
  }
  
  try {
    const value = await fetchFn()
    await redis.setex(key, 3600, JSON.stringify(value))
    return value
  } finally {
    await redis.del(lockKey)
  }
}
```

**Cache Penetration:**
```typescript
// Problem: attacker requests non-existent IDs → all miss cache → hit DB for every request

// Solution: Cache null results
async function getProduct(id: string) {
  const cached = await redis.get(`product:${id}`)
  if (cached === 'null') return null     // Cached null
  if (cached) return JSON.parse(cached)  // Cached value
  
  const product = await db.findProduct(id)
  
  if (!product) {
    await redis.setex(`product:${id}`, 300, 'null')  // Cache null for 5 minutes
    return null
  }
  
  await redis.setex(`product:${id}`, 3600, JSON.stringify(product))
  return product
}

// Solution 2: Bloom filter (probabilistic membership check)
// Check bloom filter before querying cache/DB
// If bloom filter says "definitely not in DB" → return 404 immediately
```

---

## 5. Load Balancing

### Algorithms

| Algorithm | How | Best For |
|---|---|---|
| Round Robin | Rotate through servers evenly | Equal server capacity |
| Weighted Round Robin | More traffic to higher-capacity servers | Mixed server sizes |
| Least Connections | Route to server with fewest active connections | Long-lived connections |
| IP Hash | Hash client IP → consistent server | Stateful apps (session affinity) |
| Random | Random server selection | Simplest, works well at scale |
| Least Response Time | Route to fastest server | Latency-sensitive |

### Layer 4 vs Layer 7 Load Balancers

```
L4 (Transport Layer):
- Routes based on IP/TCP/UDP
- Cannot inspect HTTP headers, paths, or content
- Very fast — minimal processing
- Good for: raw throughput, non-HTTP protocols
- Examples: AWS NLB, HAProxy (TCP mode)

L7 (Application Layer):
- Routes based on HTTP host, path, headers, cookies, body
- Can do path-based routing, host-based routing, A/B testing
- Slightly more overhead (but negligible at modern scale)
- Can terminate TLS, inspect and modify requests
- Good for: most web apps
- Examples: AWS ALB, Nginx, Traefik, Envoy
```

### Health Checks

```yaml
# Kubernetes liveness and readiness probes
spec:
  containers:
  - name: api
    livenessProbe:
      httpGet:
        path: /health/live        # Returns 200 if process is alive
        port: 3000
      initialDelaySeconds: 10     # Wait before first check
      periodSeconds: 10           # Check every 10s
      failureThreshold: 3         # Remove after 3 failures
    
    readinessProbe:
      httpGet:
        path: /health/ready       # Returns 200 if ready to serve traffic
        port: 3000
      # ready = alive + DB connected + cache connected
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 2         # Remove from pool after 2 failures

# Health endpoint implementation
app.get('/health/live', (req, res) => res.status(200).json({ status: 'alive' }))

app.get('/health/ready', async (req, res) => {
  const checks = await Promise.allSettled([
    db.query('SELECT 1'),
    redis.ping(),
  ])
  
  const healthy = checks.every(c => c.status === 'fulfilled')
  res.status(healthy ? 200 : 503).json({
    status: healthy ? 'ready' : 'not_ready',
    checks: checks.map((c, i) => ({
      name: ['db', 'redis'][i],
      healthy: c.status === 'fulfilled',
    }))
  })
})
```

---

## 6. Message Queues

### When to Use Async Processing

```
Synchronous (immediate):
- User submits payment → await payment processor → return result
- Reading user profile → return immediately

Asynchronous (queue):
- User places order → put order in queue → return "accepted" immediately
  → Worker processes order in background
  → Notify user via webhook/email/push when done

Benefits of queues:
1. Decoupling: producer doesn't depend on consumer availability
2. Load leveling: spike of 10,000 requests/second → queue buffers → consume at 1,000/second
3. Retry logic: worker fails → message returns to queue → retry
4. Backpressure: consumer signals capacity; queue holds until ready
5. Fan-out: one message → multiple consumers (audit log, email, analytics)
```

### Kafka Architecture

```
Producer → Topic (partitioned, replicated) → Consumer Groups

Partitions:
- Topic split into N partitions (parallelism = N consumer instances)
- Messages within a partition maintain order
- Partition key determines which partition (order-id → same partition → sequential processing)

Consumer Groups:
- Group of consumers sharing a topic
- Each partition consumed by exactly one consumer in the group
- 3 partitions + 3 consumers = maximum parallelism
- 3 partitions + 5 consumers = 2 consumers idle (cannot exceed partition count)

Retention:
- Kafka retains messages for configured time (default: 7 days)
- Consumers can replay from any offset
- This enables event sourcing and audit trails

Replication:
- Each partition has 1 leader + N replicas
- Producer writes to leader; replicas follow
- If leader fails: new leader elected from in-sync replicas (ISR)
```

### Message Queue Patterns

**At-Most-Once Delivery:**
```
Message delivered ≤1 time. Can be lost. Never duplicated.
Good for: metrics, logging (duplicate irrelevant; loss acceptable)
```

**At-Least-Once Delivery:**
```
Message delivered ≥1 time. Cannot be lost. May be duplicated.
Good for: most cases; application must be idempotent
Example: payment retry must not double-charge
```

**Exactly-Once Delivery:**
```
Message delivered exactly once. No loss, no duplicate.
Very difficult; requires distributed transactions.
Good for: financial transactions, inventory updates
Kafka supports this with transactions + idempotent producers
```

**Idempotency Pattern:**
```typescript
// Idempotent consumer — safe to process message multiple times
async function processOrder(message: OrderMessage) {
  const { orderId, items, userId } = message
  
  // Check if already processed (by idempotency key)
  const existing = await db.query(
    'SELECT status FROM orders WHERE id = $1',
    [orderId]
  )
  
  if (existing.rows.length && existing.rows[0].status !== 'pending') {
    // Already processed — skip (Kafka will commit offset)
    return
  }
  
  // Process...
  await db.transaction(async (trx) => {
    await trx.query('UPDATE orders SET status = $1 WHERE id = $2', ['processing', orderId])
    await trx.query('INSERT INTO order_items ...', [/* items */])
  })
}
```

---

## 7. CDN and Global Distribution

### CDN Architecture

```
Client (Brazil)  →  CDN Edge (São Paulo)  →  Origin (US East)
                         ↓
                   Cached content (< 5ms)
                   vs Origin fetch (200ms+)

Pull CDN: Edge fetches from origin on first request, caches
Push CDN: You push content to edge nodes proactively
Hybrid: Popular content pushed; long-tail pulled

Cache TTL strategy:
- Static assets (CSS, JS with content hash): immutable (1 year)
  Cache-Control: public, max-age=31536000, immutable
- Images: 30 days
  Cache-Control: public, max-age=2592000
- API responses: short TTL or no-cache
  Cache-Control: no-cache (must revalidate with origin)
- HTML: 0 or short TTL (content changes)
  Cache-Control: no-store
```

### Multi-Region Architecture

```
Global Traffic Manager (Route 53, Cloudflare)
          │
    ┌─────┼──────┐
    │     │      │
  US-E  EU-W   AP-SE
    │     │      │
  App   App    App
    │     │      │
  DB    DB     DB  ← Read replicas (write to primary in US-E)
  
Challenges:
1. Write latency: user in AP-SE writes → travels to US-E primary (200ms round trip)
   Solution: regional write primaries with conflict resolution (CRDTs, last-write-wins)
   
2. Data residency: GDPR requires EU user data stays in EU
   Solution: data partitioning by user region
   
3. Session synchronization: user's session must be accessible from any region
   Solution: JWT (stateless) or global Redis cluster (DynamoDB Global Tables)
```

---

## 8. API Design

### REST vs GraphQL vs gRPC

| Aspect | REST | GraphQL | gRPC |
|---|---|---|---|
| Protocol | HTTP | HTTP | HTTP/2 |
| Format | JSON/XML | JSON | Protobuf (binary) |
| Type safety | By convention | Schema (SDL) | Strong (proto) |
| Overfetching | Common | Eliminated | Eliminated |
| Underfetching | N+1 problem | Eliminated | Eliminated |
| Caching | HTTP cache | Per-query | Requires custom |
| Browser support | Native | Requires client | Requires gRPC-web |
| Best for | Public APIs | Frontend-driven | Internal microservices |

### REST API Design Principles

```
Resource-Oriented:
GET    /users              → list users
POST   /users              → create user
GET    /users/:id          → get user
PUT    /users/:id          → replace user (full update)
PATCH  /users/:id          → update user (partial)
DELETE /users/:id          → delete user

Nested resources (use sparingly):
GET    /users/:id/orders   → user's orders
POST   /users/:id/orders   → create order for user

Query parameters for filtering, sorting, pagination:
GET    /orders?status=pending&sort=created_at:desc&limit=20&cursor=opaque_cursor

HTTP Status Codes:
200 OK              → success (GET, PUT, PATCH)
201 Created         → success (POST with resource created)
204 No Content      → success (DELETE, action with no body)
400 Bad Request     → client error (validation, malformed request)
401 Unauthorized    → authentication required (not logged in)
403 Forbidden       → authenticated but not authorized
404 Not Found       → resource not found
409 Conflict        → duplicate, version conflict
422 Unprocessable   → semantic validation failure
429 Too Many        → rate limited
500 Internal Error  → server error (never expose internals)
503 Unavailable     → service temporarily down
```

### Pagination Patterns

```typescript
// Cursor-based pagination (production-grade)
// Advantages over offset: consistent results, no duplicates, efficient for large datasets
// Offset: SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1000
//   → Full scan of 1020 rows to return 20 — gets slower as pages increase

interface CursorPage<T> {
  data: T[]
  nextCursor: string | null  // Opaque cursor for next page
  hasMore: boolean
}

async function listBookings(userId: string, cursor?: string, limit = 20): Promise<CursorPage<Booking>> {
  // Decode cursor (opaque to client)
  let afterDate: Date | null = null
  if (cursor) {
    const decoded = Buffer.from(cursor, 'base64url').toString()
    const [timestamp] = JSON.parse(decoded)
    afterDate = new Date(timestamp)
  }
  
  const query = afterDate
    ? `SELECT * FROM bookings WHERE client_id = $1 AND scheduled_at < $2 ORDER BY scheduled_at DESC LIMIT $3`
    : `SELECT * FROM bookings WHERE client_id = $1 ORDER BY scheduled_at DESC LIMIT $2`
  
  const params = afterDate ? [userId, afterDate, limit + 1] : [userId, limit + 1]
  const result = await pool.query(query, params)
  
  const hasMore = result.rows.length > limit
  const data = result.rows.slice(0, limit)
  
  const nextCursor = hasMore
    ? Buffer.from(JSON.stringify([data[data.length - 1].scheduled_at])).toString('base64url')
    : null
  
  return { data, nextCursor, hasMore }
}
```

---

## 9. Consistency Models

### The Spectrum of Consistency

```
Strong                              Eventual
     │                                  │
     ▼                                  ▼
Linearizability  Serializability  Causal  Read-Your-Writes  Eventual

Linearizability (strongest):
- All operations appear instantaneous and total ordered
- Every read sees the most recent write
- Expensive: requires coordination → high latency, low availability
- Used for: locks, leader election, single-record counters

Serializability (transactions):
- Transactions appear to execute serially (even if concurrent)
- Does not require real-time ordering
- ACID databases (PostgreSQL SERIALIZABLE isolation level)

Read-Your-Writes:
- After you write X, you can always read X back
- Others may not see it yet (eventual)
- Used for: profile updates, user settings
- Implementation: route user's reads to same replica as their writes

Causal Consistency:
- Operations with causal relationship maintain order
- "Comment on a post" must see the post first
- Distributed systems with vector clocks

Eventual Consistency:
- All replicas will converge to same state eventually
- No guarantees on when
- Used for: DNS, social media timelines, shopping cart
```

### Isolation Levels (SQL)

```sql
-- READ UNCOMMITTED: dirty reads (see uncommitted data from other transactions)
-- READ COMMITTED: sees committed data only (PostgreSQL default)
-- REPEATABLE READ: same SELECT returns same rows during transaction
-- SERIALIZABLE: full transaction isolation — as if serial execution

-- Set isolation level
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Serializable prevents write skew:
-- T1: reads (doctor_count = 1), T2: reads (doctor_count = 1)
-- T1: writes (doctor_count = 0), T2: writes (doctor_count = 0)
-- Both commit → negative doctors! Serializable prevents this.
```

---

## 10. CAP Theorem

**CAP Theorem:** In a distributed system with network partitions, you cannot guarantee both Consistency and Availability simultaneously.

```
C — Consistency: Every read receives the most recent write (or an error)
A — Availability: Every request receives a response (not necessarily up-to-date)
P — Partition Tolerance: System continues operating despite network partitions

(Network partitions are not optional in distributed systems — they happen)

Therefore: CP or AP
CP: Accept downtime when partitioned (bank systems, inventory)
AP: Accept stale data when partitioned (DNS, shopping cart, social feed)
```

### PACELC Extension

PACELC: "If Partition, then Consistency or Availability; Else Latency or Consistency"

```
Even without partitions (normal operation), there's a tradeoff:
Synchronous replication → low latency writes? No — must wait for replicas to confirm
Asynchronous replication → higher availability + lower latency → stale reads

Examples:
- DynamoDB: PA/EL (available under partition, lower latency over consistency)
- CockroachDB: PC/EC (consistent under partition, consistent in normal operation)
- Cassandra: PA/EL (tunable consistency)
- PostgreSQL single node: not distributed (CAP doesn't apply)
- PostgreSQL with replicas: consistency vs read latency tradeoff
```

---

## 11. System Design Interview Framework

### RESHADED Framework

```
R — Requirements (10 min)
  - Clarify functional requirements
  - Estimate scale (DAU, QPS, data size)
  - Define non-functional requirements (latency, availability)

E — Estimation (5 min)
  - Storage estimates
  - Bandwidth estimates  
  - Cache estimates

S — Service Design (10 min)
  - API design: endpoints, request/response formats
  - Core flows: how does data move through the system?

H — High-Level Design (10 min)
  - Block diagram: clients, load balancers, services, databases, caches, queues
  - Choose storage types and justify

A — APIs Deep Dive (5 min)
  - Detail 1-2 critical API calls end-to-end

D — Data Model (10 min)
  - Schema design
  - Indexes
  - Sharding strategy

E — Expand / Evolve (10 min)
  - Identify bottlenecks
  - Discuss trade-offs
  - Handle edge cases (spikes, failures, hot shards)

D — Defend (5 min)
  - Why not X? (defend your choices)
  - What would you do differently at 10× scale?
```

---

## 12. Reference Architectures

### Web Application (SaaS)

```
DNS (Route 53 / Cloudflare)
  │
CDN (CloudFront / Cloudflare) ← Static assets, media files
  │
Load Balancer (ALB)
  │
  ├── App Servers (ECS/K8s)  ← Stateless, horizontal scaling
  │       │
  │   ├── PostgreSQL (RDS)    ← Primary + Read Replicas
  │   ├── Redis (ElastiCache) ← Sessions, cache, queues
  │   └── S3                  ← File storage
  │
  └── Workers (Background Jobs)
          │
      ├── SQS / Kafka
      └── Job processing (emails, notifications, exports)

Monitoring: Datadog / Grafana + Prometheus
Logging: CloudWatch / ELK
Tracing: Jaeger / AWS X-Ray
```

### Microservices

```
API Gateway (Kong / AWS API GW)
  │
  ├── User Service (owns users DB)
  ├── Order Service (owns orders DB)
  ├── Payment Service (owns payment DB)
  ├── Notification Service
  └── Search Service (Elasticsearch)

Communication:
  - Synchronous: REST or gRPC (service-to-service)
  - Asynchronous: Kafka (order placed → payment, notification, inventory)
  
Service Mesh: Istio / Linkerd
  - mTLS between services (East-West traffic)
  - Circuit breaking
  - Observability
```

### Event-Driven

```
User Action → API → Event Bus (Kafka)
                         │
                ┌────────┼────────┐
                ▼        ▼        ▼
          Analytics   Email    Audit Log
           Service   Service   Service
                         │
                   PostgreSQL (events stored)
```
