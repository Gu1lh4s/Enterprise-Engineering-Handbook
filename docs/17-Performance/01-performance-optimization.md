# Performance Optimization

> **Category:** Performance Engineering
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Performance Culture](#1-performance-culture)
2. [Profiling and Measurement](#2-profiling-and-measurement)
3. [Database Performance](#3-database-performance)
4. [Caching Strategies](#4-caching-strategies)
5. [HTTP and API Performance](#5-http-and-api-performance)
6. [Node.js Specific Optimizations](#6-nodejs-specific-optimizations)
7. [Frontend Performance](#7-frontend-performance)
8. [CDN and Edge Caching](#8-cdn-and-edge-caching)
9. [Load Testing and Capacity Planning](#9-load-testing-and-capacity-planning)
10. [Performance Regression Prevention](#10-performance-regression-prevention)

---

## 1. Performance Culture

```
Principles:
  1. Measure before optimizing — assumptions are wrong
  2. Profile, then fix the top bottleneck — Pareto: 80% of time in 20% of code
  3. Optimize for p99, not average — averages hide tail latency
  4. Make it work, then make it fast — premature optimization is waste
  5. Set performance budgets — what you don't measure, you don't improve

Performance Budget (enforce in CI):
  API p99 latency:         < 300ms
  API error rate:          < 0.1%
  Frontend LCP:            < 2.5s
  Frontend bundle size:    < 200KB (initial JS, compressed)
  DB query time (p99):     < 50ms
  Cache hit rate:          > 80%

"The Three Laws of Performance":
  1. Do less work (eliminate unnecessary computation)
  2. Do it less often (cache, batch)
  3. Do it faster (better algorithms, parallelism)
```

---

## 2. Profiling and Measurement

### Node.js CPU Profiling

```bash
# CPU profile: find what's consuming CPU
node --prof server.js  # Run with profiler

# Process the profile
node --prof-process isolate-*.log > profile.txt

# Or use clinic.js (more user-friendly)
npx clinic flame -- node server.js  # Flame graph
npx clinic bubbleprof -- node server.js  # Async timeline
npx clinic doctor -- node server.js  # Auto-diagnosis

# Production: V8 CPU profiler via inspector
# Connect via: chrome://inspect or VS Code debugger
node --inspect server.js
```

```typescript
// Programmatic profiling for specific code paths
import { Session } from 'inspector'
import { writeFileSync } from 'fs'

async function profileFunction<T>(name: string, fn: () => Promise<T>): Promise<T> {
  const session = new Session()
  session.connect()
  
  await new Promise<void>(resolve => session.post('Profiler.enable', resolve))
  await new Promise<void>(resolve => session.post('Profiler.start', resolve))
  
  const result = await fn()
  
  const { profile } = await new Promise<any>(resolve =>
    session.post('Profiler.stop', (_, r) => resolve(r))
  )
  
  writeFileSync(`profile-${name}-${Date.now()}.cpuprofile`, JSON.stringify(profile))
  session.disconnect()
  
  return result
}

// Usage: find which query builder call is slow
const bookings = await profileFunction('findBookings', () => bookingService.findAll())
```

### Memory Heap Snapshot

```typescript
import v8 from 'v8'
import { writeFileSync } from 'fs'

// Take heap snapshot (use Chrome DevTools to analyze)
function takeHeapSnapshot(label: string) {
  const snapshot = v8.writeHeapSnapshot(`heap-${label}-${Date.now()}.heapsnapshot`)
  console.log('Heap snapshot written to:', snapshot)
}

// Compare: take snapshot before and after suspect operation
// Open in Chrome → Memory tab → Load snapshot
// Look for: retained objects, detached DOM nodes, closure leaks
```

### Performance Timing

```typescript
// Measure specific operations
async function measureOperation<T>(name: string, fn: () => Promise<T>): Promise<T> {
  const start = performance.now()
  try {
    const result = await fn()
    const duration = performance.now() - start
    
    // Record to Prometheus histogram
    operationDuration.observe({ operation: name }, duration / 1000)
    
    if (duration > 1000) {
      logger.warn({ operation: name, durationMs: duration }, 'Slow operation detected')
    }
    
    return result
  } catch (err) {
    const duration = performance.now() - start
    operationDuration.observe({ operation: name, status: 'error' }, duration / 1000)
    throw err
  }
}
```

---

## 3. Database Performance

### Query Analysis

```sql
-- Find slow queries
SELECT
  query,
  calls,
  mean_exec_time AS avg_ms,
  max_exec_time AS max_ms,
  total_exec_time AS total_ms,
  rows / calls AS avg_rows,
  (total_exec_time / sum(total_exec_time) OVER ()) * 100 AS pct_of_total
FROM pg_stat_statements
WHERE calls > 100  -- Only queries called frequently
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find missing indexes (sequential scans on large tables)
SELECT
  relname AS table,
  seq_scan,
  idx_scan,
  n_live_tup AS row_count,
  seq_scan - idx_scan AS seq_vs_idx
FROM pg_stat_user_tables
WHERE seq_scan > 0
  AND n_live_tup > 10000  -- Only large tables
ORDER BY seq_scan DESC;

-- Find unused indexes (waste of write overhead)
SELECT
  schemaname || '.' || relname AS table,
  indexrelname AS index,
  idx_scan AS scans,
  pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_relation_size(indexrelid) DESC;

-- EXPLAIN ANALYZE deep dive
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT b.*, u.name, s.name
FROM bookings b
JOIN users u ON u.id = b.client_id
JOIN services s ON s.id = b.service_id
WHERE b.tenant_id = 'org_123'
  AND b.status = 'pending'
  AND b.scheduled_at > NOW()
ORDER BY b.scheduled_at ASC
LIMIT 20;

-- Look for: Seq Scan on large tables → add index
--           Hash Join → may prefer Nested Loop if inner is small
--           high 'Buffers: shared hit' → cache utilization good
--           high 'Buffers: shared read' → disk reads, consider increasing shared_buffers
```

### Index Strategy

```sql
-- Composite index: column order matters
-- Rule: Equality columns first, then range, then sort

-- BAD: range on first column wastes index for equality filter
CREATE INDEX idx_bookings_bad ON bookings(scheduled_at, tenant_id, status);

-- GOOD: equality filters first, range last
CREATE INDEX idx_bookings_good ON bookings(tenant_id, status, scheduled_at);

-- Covering index: include columns needed by SELECT (avoid table heap fetch)
CREATE INDEX idx_bookings_covering ON bookings(tenant_id, status, scheduled_at)
  INCLUDE (id, service_id, client_id);  -- INCLUDE: stored but not searchable

-- Partial index: only index rows that match filter
-- Great for status-filtered queries where most rows are 'completed'
CREATE INDEX idx_bookings_pending ON bookings(tenant_id, scheduled_at)
  WHERE status IN ('pending', 'confirmed');  -- 5% of rows vs 100%

-- GIN index for JSONB search
CREATE INDEX idx_bookings_metadata ON bookings USING GIN (metadata);
-- SELECT * FROM bookings WHERE metadata @> '{"source": "mobile"}';

-- Full-text search index
ALTER TABLE services ADD COLUMN search_vector tsvector;
CREATE INDEX idx_services_fts ON services USING GIN(search_vector);

CREATE OR REPLACE FUNCTION update_service_search_vector() RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector := to_tsvector('portuguese', COALESCE(NEW.name, '') || ' ' || COALESCE(NEW.description, ''));
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER services_search_vector_update
  BEFORE INSERT OR UPDATE ON services
  FOR EACH ROW EXECUTE FUNCTION update_service_search_vector();
```

### N+1 Problem

```typescript
// BAD: N+1 — one query per booking to get service
const bookings = await db.select().from(bookings)  // 1 query: 50 rows
for (const booking of bookings) {
  booking.service = await db.select().from(services)  // 50 more queries!
    .where(eq(services.id, booking.serviceId))
}

// GOOD: single JOIN query
const bookingsWithServices = await db
  .select({
    id: bookings.id,
    scheduledAt: bookings.scheduledAt,
    status: bookings.status,
    service: {
      id: services.id,
      name: services.name,
      price: services.price,
    },
  })
  .from(bookings)
  .leftJoin(services, eq(services.id, bookings.serviceId))
  .where(eq(bookings.tenantId, tenantId))
  .orderBy(desc(bookings.createdAt))
  .limit(20)

// GOOD: DataLoader (for GraphQL resolvers where JOIN is hard)
const serviceLoader = new DataLoader<string, Service>(async (ids) => {
  const result = await db.select().from(services)
    .where(inArray(services.id, [...ids]))
  const map = Object.fromEntries(result.map(s => [s.id, s]))
  return ids.map(id => map[id])
})
```

### Connection Pool Tuning

```typescript
// Each connection = ~10MB RAM on PostgreSQL server
// Total max connections = instance RAM / 10MB (rough guide)
// Leave headroom for superuser connections and migrations

// Formula: (cores * 2) + effective_spindle_count
// Cloud managed DB: test empirically — start at 10, increase under load

const pool = new Pool({
  max: 10,                      // Max connections per app instance
  idleTimeoutMillis: 30_000,    // Release idle connections after 30s
  connectionTimeoutMillis: 3_000, // Fail fast if pool exhausted
})

// Monitor pool metrics
setInterval(async () => {
  poolMetrics.set({ type: 'total' }, pool.totalCount)
  poolMetrics.set({ type: 'idle' }, pool.idleCount)
  poolMetrics.set({ type: 'waiting' }, pool.waitingCount)
}, 10_000)

// PgBouncer: connection multiplexer for many app instances
// transaction mode: connection held only during transaction
// 100 app connections → 10 actual DB connections
```

---

## 4. Caching Strategies

### Cache Hierarchy

```
L1: Application memory (in-process)
  → Fastest: nanoseconds
  → Smallest: MBs (garbage collected)
  → Lost on restart
  → Per-instance (horizontal scaling → stale across instances)
  → Use for: config, feature flags, rarely-changing lookup tables

L2: Redis (shared, in-memory)
  → Fast: microseconds via LAN
  → Larger: GBs
  → Survives app restarts
  → Shared across instances
  → Use for: sessions, rate limiting, frequently-read data, computed aggregates

L3: CDN (edge cache)
  → Fast: milliseconds from nearest PoP
  → Huge capacity
  → Public content only (authenticated responses generally not cached)
  → Use for: static assets, public API responses, image optimization

L4: Database query cache (app-level)
  → Caches query results for a short TTL
  → Use for: expensive aggregations, search results
```

### Cache-Aside Pattern (most common)

```typescript
class BookingCache {
  constructor(private redis: Redis, private db: Database) {}
  
  async getByTenant(tenantId: string, filters: BookingFilters): Promise<Booking[]> {
    const cacheKey = `bookings:${tenantId}:${hashFilters(filters)}`
    
    // 1. Check cache
    const cached = await this.redis.get(cacheKey)
    if (cached) {
      cacheHits.inc({ key: 'tenant_bookings' })
      return JSON.parse(cached)
    }
    
    cacheMisses.inc({ key: 'tenant_bookings' })
    
    // 2. Fetch from DB
    const bookings = await this.db.findByTenant(tenantId, filters)
    
    // 3. Store in cache with TTL
    await this.redis.setex(cacheKey, 60, JSON.stringify(bookings))  // 1 min TTL
    
    return bookings
  }
  
  // Invalidate on write
  async invalidateTenant(tenantId: string) {
    // Pattern: scan and delete all keys for this tenant
    // WARNING: KEYS is O(N) — use SCAN in production
    const pattern = `bookings:${tenantId}:*`
    let cursor = '0'
    
    do {
      const [nextCursor, keys] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100)
      cursor = nextCursor
      if (keys.length > 0) {
        await this.redis.del(...keys)
      }
    } while (cursor !== '0')
  }
}
```

### Cache Stampede Prevention

```typescript
// Problem: many requests hit DB simultaneously when cache expires
// Solution: probabilistic early recompute (PER algorithm)

async function getCachedWithPer<T>(
  key: string,
  ttl: number,
  compute: () => Promise<T>
): Promise<T> {
  const [value, expiryTime] = await redis.mget(key, `${key}:expiry`)
  
  if (value) {
    const remainingTtl = parseInt(expiryTime ?? '0') - Date.now()
    const beta = 1  // Tune: higher → more aggressive early recompute
    
    // Probabilistically recompute before expiry
    // As remainingTtl → 0, probability of recompute → 1
    if (-beta * ttl * Math.log(Math.random()) < remainingTtl) {
      return JSON.parse(value)
    }
  }
  
  // Compute fresh value
  const freshValue = await compute()
  const newExpiry = Date.now() + ttl * 1000
  
  await redis.multi()
    .setex(key, ttl, JSON.stringify(freshValue))
    .setex(`${key}:expiry`, ttl, String(newExpiry))
    .exec()
  
  return freshValue
}

// Mutex lock (simpler, more aggressive):
async function getCachedWithMutex<T>(key: string, ttl: number, compute: () => Promise<T>): Promise<T> {
  const cached = await redis.get(key)
  if (cached) return JSON.parse(cached)
  
  // Acquire lock (SET NX = only set if not exists)
  const lockAcquired = await redis.set(`${key}:lock`, '1', 'EX', 10, 'NX')
  
  if (!lockAcquired) {
    // Wait for lock holder to finish (poll with backoff)
    await sleep(50 + Math.random() * 50)
    return getCachedWithMutex(key, ttl, compute)  // Retry
  }
  
  try {
    const value = await compute()
    await redis.setex(key, ttl, JSON.stringify(value))
    return value
  } finally {
    await redis.del(`${key}:lock`)
  }
}
```

### Cache Key Design

```typescript
// Hierarchical cache keys enable targeted invalidation
const cacheKeys = {
  // Tenant-level (invalidate all on org change)
  tenantBookings: (tenantId: string, status?: string) =>
    `v1:bookings:tenant:${tenantId}${status ? `:${status}` : ''}`,
  
  // Resource-level (invalidate on specific resource change)
  booking: (id: string) => `v1:bookings:id:${id}`,
  
  // Aggregation (invalidate on any booking change in org)
  tenantStats: (tenantId: string, month: string) =>
    `v1:stats:tenant:${tenantId}:month:${month}`,
  
  // Public list (shared across all users, invalidate rarely)
  services: (tenantId: string) => `v1:services:tenant:${tenantId}`,
}

// Version prefix 'v1:' → cache bust all keys by incrementing version
// Cache by version number stored in Redis
const version = await redis.get('cache:version') ?? '1'
const keyWithVersion = `v${version}:bookings:...`
```

---

## 5. HTTP and API Performance

### Response Compression

```typescript
import compression from 'compression'

app.use(compression({
  level: 6,                 // gzip level 1-9 (6 = good balance)
  threshold: 1024,          // Only compress responses > 1KB
  filter: (req, res) => {
    // Don't compress: already compressed files, SSE streams
    if (req.path.match(/\.(jpg|png|gif|webp|avif|mp4|zip|gz)$/)) return false
    if (req.headers.accept?.includes('text/event-stream')) return false
    return compression.filter(req, res)
  },
}))
```

### HTTP Caching Headers

```typescript
// Cache-Control strategy per resource type

// Static assets (JS, CSS, images with content hash in filename)
res.setHeader('Cache-Control', 'public, max-age=31536000, immutable')
// max-age=31536000 = 1 year; immutable = browser skips revalidation
// Works because filename changes when content changes (e.g., main.a3f9b2.js)

// API responses (mutable)
res.setHeader('Cache-Control', 'no-store')  // Sensitive data: never cache
res.setHeader('Cache-Control', 'private, max-age=0, must-revalidate')  // User-specific

// Public, cacheable API responses (e.g., list of services)
res.setHeader('Cache-Control', 'public, max-age=60, stale-while-revalidate=300')
res.setHeader('ETag', generateETag(response))  // Enable conditional requests

// CDN cache (separate from browser cache)
res.setHeader('Cache-Control', 'public, s-maxage=300, max-age=60')
// s-maxage: CDN TTL; max-age: browser TTL
```

### Pagination Performance

```typescript
// OFFSET pagination: O(n) — slow for large offsets
// SELECT * FROM bookings ORDER BY created_at LIMIT 20 OFFSET 10000
// → DB must scan and discard 10000 rows

// Cursor pagination: O(1) — always fast regardless of position
async function getCursorPage(cursor: string | null, limit: number) {
  const cursorData = cursor ? decodeCursor(cursor) : null
  
  const bookings = await db
    .select()
    .from(bookings)
    .where(
      cursorData
        ? or(
            lt(bookings.createdAt, cursorData.createdAt),
            and(
              eq(bookings.createdAt, cursorData.createdAt),
              lt(bookings.id, cursorData.id)
            )
          )
        : undefined
    )
    .orderBy(desc(bookings.createdAt), desc(bookings.id))
    .limit(limit + 1)  // Fetch one extra to check hasMore
  
  const hasMore = bookings.length > limit
  const items = hasMore ? bookings.slice(0, -1) : bookings
  
  return {
    data: items,
    meta: {
      hasMore,
      nextCursor: hasMore ? encodeCursor(items.at(-1)!) : null,
    }
  }
}

function encodeCursor(item: { id: string; createdAt: Date }): string {
  return Buffer.from(JSON.stringify({ id: item.id, createdAt: item.createdAt })).toString('base64url')
}

function decodeCursor(cursor: string): { id: string; createdAt: Date } {
  return JSON.parse(Buffer.from(cursor, 'base64url').toString())
}
```

---

## 6. Node.js Specific Optimizations

### Event Loop Health

```typescript
// Monitor event loop lag (high lag = something blocking the event loop)
import { monitorEventLoopDelay } from 'perf_hooks'

const histogram = monitorEventLoopDelay({ resolution: 10 })
histogram.enable()

setInterval(() => {
  const lagMs = histogram.mean / 1e6  // Convert nanoseconds to ms
  
  eventLoopLag.set(lagMs)
  histogram.reset()
  
  if (lagMs > 100) {
    logger.warn({ lagMs }, 'High event loop lag — possible blocking code')
  }
}, 5000)

// Common blocking operations to avoid in hot path:
// JSON.parse of large payloads (use streaming parser)
// Regex with catastrophic backtracking
// crypto.pbkdf2Sync, bcrypt.hashSync → use async versions
// fs.readFileSync in request handler
// Large synchronous loops
```

### Streaming Responses

```typescript
import { pipeline } from 'stream/promises'
import { createReadStream } from 'fs'
import { createGzip } from 'zlib'

// BAD: load entire file into memory, then send
app.get('/export/bookings', async (req, res) => {
  const allBookings = await db.query('SELECT * FROM bookings')  // Could be GBs
  res.json(allBookings)  // OOM risk
})

// GOOD: stream response (no memory spike)
app.get('/export/bookings', async (req, res) => {
  res.setHeader('Content-Type', 'application/json')
  res.setHeader('Content-Encoding', 'gzip')
  
  const gzip = createGzip()
  
  // PostgreSQL streaming cursor
  const client = await pool.connect()
  
  try {
    const cursor = client.query(new Cursor('SELECT * FROM bookings WHERE tenant_id = $1', [req.user.tenantId]))
    
    res.write('[')
    let first = true
    
    while (true) {
      const rows = await cursor.read(100)  // 100 rows at a time
      if (rows.length === 0) break
      
      for (const row of rows) {
        if (!first) res.write(',')
        res.write(JSON.stringify(row))
        first = false
      }
    }
    
    res.write(']')
    res.end()
  } finally {
    client.release()
  }
})
```

---

## 7. Frontend Performance

### Bundle Analysis and Reduction

```bash
# Analyze bundle
npx next build && npx next analyze
# Or: ANALYZE=true npm run build

# Key metrics to check:
# - First Load JS: should be < 100KB per route
# - Largest chunks: target for code splitting
# - Duplicate packages: e.g., moment + date-fns both included

# Common bundle wins:
# 1. Replace moment.js (67KB) with date-fns (tree-shakeable)
# 2. Replace lodash (71KB) with lodash-es (tree-shakeable) or native
# 3. Replace chart libraries with lighter alternatives
# 4. Use dynamic imports for rarely-used features

# Tree-shaking: import only what you use
import { format, parseISO, addDays } from 'date-fns'  # ✓ tree-shakes
import _ from 'lodash'                                 # ✗ imports all of lodash
import { debounce } from 'lodash'                      # ✓ still pulls whole lodash
import { debounce } from 'lodash-es'                   # ✓ tree-shakes
```

### Image Optimization

```tsx
// Next.js Image component: automatic WebP/AVIF, lazy loading, layout shift prevention
import Image from 'next/image'

function ServiceCard({ service }: { service: Service }) {
  return (
    <div>
      <Image
        src={service.imageUrl}
        alt={service.name}           // Required: accessibility
        width={400}
        height={300}
        loading="lazy"               // Default for below-fold images
        // loading="eager"           // For above-fold hero images
        sizes="(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 33vw"
        priority={false}             // Set true for LCP image
        placeholder="blur"
        blurDataURL="data:image/jpeg;base64,..."
      />
    </div>
  )
}
```

---

## 8. CDN and Edge Caching

### Cache-Control Strategy

```
Public vs Private:
  public  → CDN can cache (static assets, public pages)
  private → CDN must NOT cache (authenticated responses, personal data)

TTL Strategy:
  Static assets (with hash): max-age=31536000, immutable   # Forever (hash changes)
  HTML pages:                max-age=0, must-revalidate    # Revalidate each time
  Public API lists:          s-maxage=60, stale-while-revalidate=300
  Authenticated API:         no-store (never cache)

Stale-while-revalidate: serve stale while fetching fresh
  → User gets immediate response even if cache is stale
  → Background fetch updates cache
  → Great for non-critical data (service list, pricing)
  
Stale-if-error: serve stale if origin errors
  → Resilience: CDN serves cached content during outages
```

```typescript
// API Gateway or Express: set CDN-specific cache headers
function setCacheHeaders(res: Response, options: {
  isPublic: boolean
  maxAge: number          // Browser TTL (seconds)
  sMaxAge?: number        // CDN TTL (seconds)
  staleWhileRevalidate?: number
  staleIfError?: number
}) {
  const directives: string[] = [
    options.isPublic ? 'public' : 'private',
    `max-age=${options.maxAge}`,
  ]
  
  if (options.sMaxAge !== undefined) directives.push(`s-maxage=${options.sMaxAge}`)
  if (options.staleWhileRevalidate) directives.push(`stale-while-revalidate=${options.staleWhileRevalidate}`)
  if (options.staleIfError) directives.push(`stale-if-error=${options.staleIfError}`)
  
  res.setHeader('Cache-Control', directives.join(', '))
}

// Public services endpoint (cacheable)
app.get('/api/v1/services', async (req, res) => {
  setCacheHeaders(res, { isPublic: true, maxAge: 60, sMaxAge: 300, staleWhileRevalidate: 600 })
  res.json(await servicesService.findPublic())
})
```

---

## 9. Load Testing and Capacity Planning

```javascript
// k6 scenario: simulate realistic traffic patterns
import http from 'k6/http'
import { check, sleep } from 'k6'

export const options = {
  scenarios: {
    // Realistic usage: morning ramp-up, busy period, evening wind-down
    realistic_traffic: {
      executor: 'ramping-vus',
      stages: [
        { duration: '5m',  target: 10 },   // Early morning: low traffic
        { duration: '10m', target: 50 },   // Business hours ramp
        { duration: '20m', target: 100 },  // Peak: 100 concurrent users
        { duration: '10m', target: 50 },   // Evening wind-down
        { duration: '5m',  target: 0 },    // Night
      ],
    },
    
    // Spike test: social media post sends sudden traffic
    viral_spike: {
      executor: 'ramping-vus',
      startTime: '50m',  // Start after baseline
      stages: [
        { duration: '30s', target: 500 },  // Sudden spike: 0→500 in 30s
        { duration: '2m',  target: 500 },  // Sustain spike
        { duration: '30s', target: 0 },    // Drop off
      ],
    },
  },
  
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
  },
}
```

### Capacity Planning

```
Capacity formula:
  Max sustainable QPS = (target latency / avg response time) × replicas
  
  Example:
    Target latency: p99 < 500ms
    Current avg response: 50ms per request
    Pod handles: 500ms / 50ms = 10 concurrent requests
    3 pods: 30 concurrent requests
    
    If requests arrive at 100 QPS with avg 50ms duration:
    Concurrency = 100 × 0.05 = 5 in-flight requests
    3 pods (30 capacity) → 83% headroom ✓

Rule of thumb: size for 3× average traffic
  → Handles 3× traffic spikes without degradation
  → Remaining capacity: safety buffer during incidents + rolling deployments

Database capacity:
  Read replicas: add when read traffic > 70% of primary capacity
  Connection pooling (PgBouncer): allows 100s of app connections → 10 DB connections
  Read replicas + connection pooling = baseline for most web apps to 10M users
```

---

## 10. Performance Regression Prevention

```yaml
# CI performance gate: fail build if performance degrades
name: Performance Check

on: [pull_request]

jobs:
  performance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Start services
        run: docker-compose up -d && sleep 10
      
      - name: Run k6 performance test
        uses: grafana/k6-action@v0.3.0
        with:
          filename: performance/smoke-test.js
        env:
          BASE_URL: http://localhost:3000
      
      - name: Check bundle size
        run: |
          npm run build
          BUNDLE_SIZE=$(cat .next/build-manifest.json | python3 -c "...")
          
          # Fail if > 200KB
          if [ "$BUNDLE_SIZE" -gt 204800 ]; then
            echo "::error::Bundle size ${BUNDLE_SIZE}B exceeds limit 204800B (200KB)"
            exit 1
          fi
          echo "Bundle size: ${BUNDLE_SIZE}B ✓"
      
      - name: Lighthouse CI
        uses: treosh/lighthouse-ci-action@v10
        with:
          urls: http://localhost:3000
          budgetPath: .lighthouse-budget.json
          
# .lighthouse-budget.json
[{
  "path": "/*",
  "resourceSizes": [
    { "resourceType": "script", "budget": 200 },
    { "resourceType": "total", "budget": 500 }
  ],
  "timings": [
    { "metric": "first-contentful-paint", "budget": 2000 },
    { "metric": "interactive", "budget": 5000 }
  ]
}]
```
