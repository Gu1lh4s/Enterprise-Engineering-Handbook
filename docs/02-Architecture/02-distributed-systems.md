# Distributed Systems

> **Category:** Architecture
> **Version:** 1.0.0
> **Level:** Staff Engineer / Principal Engineer

---

## Table of Contents

1. [Fundamental Problems](#1-fundamental-problems)
2. [Consensus Algorithms](#2-consensus-algorithms)
3. [Replication Patterns](#3-replication-patterns)
4. [Distributed Transactions](#4-distributed-transactions)
5. [Clock Synchronization](#5-clock-synchronization)
6. [Failure Detection](#6-failure-detection)
7. [Circuit Breaker Pattern](#7-circuit-breaker-pattern)
8. [Rate Limiting Algorithms](#8-rate-limiting-algorithms)
9. [Distributed Locking](#9-distributed-locking)
10. [Observability in Distributed Systems](#10-observability-in-distributed-systems)

---

## 1. Fundamental Problems

### Why Distributed Systems Are Hard

```
1. Partial Failure
   - A node can crash while others continue
   - A network partition can make a node unreachable but still running
   - A node can be slow (high latency) — indistinguishable from failure
   - Detection is unreliable: "Is the server dead, or is my network dead?"

2. Unreliable Networks
   - Packets can be lost, reordered, duplicated, delayed
   - There is no reliable way to know if a request was received and processed
   - Retrying a request may cause duplicate processing (idempotency problem)

3. No Global Time
   - Clocks on different machines drift
   - NTP synchronization is imperfect (±100ms typical, ±1ms with PTP)
   - Cannot rely on wall clock time to determine event order
   - "Clock skew" can cause event B to appear before event A even if A happened first

4. Concurrency
   - Multiple nodes process requests simultaneously
   - Race conditions without coordinated locking
   - Stale reads even with "up to date" data due to propagation delay

5. The Fallacies of Distributed Computing (Deutsch, 1994)
   - The network is reliable
   - Latency is zero
   - Bandwidth is infinite
   - The network is secure
   - Topology doesn't change
   - There is one administrator
   - Transport cost is zero
   - The network is homogeneous
```

### The Two Generals Problem

Proof that reliable communication over an unreliable channel is theoretically impossible:

```
Two armies (Blue) need to coordinate attack on city (Red)
They can only communicate by messenger through Red's territory
Messengers can be captured (message lost)

Blue General 1: "Attack at dawn?" → [messenger may be captured] → Blue General 2
Blue General 2: "Confirmed!" → [messenger may be captured] → Blue General 1

Blue General 1 cannot know if the confirmation was received.
Blue General 2 cannot know if their confirmation was received.

Lesson: You cannot guarantee consensus over an unreliable channel.
TCP solves practical cases but cannot eliminate the theoretical problem.
```

---

## 2. Consensus Algorithms

Consensus = getting distributed nodes to agree on a single value.

### Raft (Modern, Understandable)

```
Raft guarantees: linearizability, fault tolerance with f < n/2 failures

States: Follower, Candidate, Leader
- All nodes start as Followers
- If no heartbeat from leader: Follower → Candidate (election timeout)
- Candidate requests votes from peers
- If majority vote: Candidate → Leader
- Leader handles all writes, replicates to followers

Write flow:
1. Client sends write to Leader
2. Leader appends to local log
3. Leader sends AppendEntries to all followers
4. Majority acknowledge → entry committed
5. Leader responds to client

Leader election:
- Split vote: two candidates simultaneously, no majority
- Random election timeout (150-300ms) reduces split votes
- New election starts if no winner

Practical implementations:
- etcd (Kubernetes state store)
- CockroachDB (distributed SQL)
- TiKV (distributed KV)
```

### Paxos (Original, Complex)

```
Phases:
1. Prepare: Proposer sends "prepare(n)" to majority of acceptors
2. Promise: Acceptors respond with promise to not accept proposals < n
3. Accept: Proposer sends "accept(n, v)" with proposed value v
4. Accepted: Majority of acceptors accept → consensus reached

Multi-Paxos: optimization for continuous stream of decisions
- Skip Prepare phase after leader established
- Used in: Google Chubby, Apache Zookeeper (ZAB variant)

Why Raft instead of Paxos?
- Raft designed for understandability
- Paxos underspecified — many incomplete implementations
- Raft has clearer specification for leader election and log replication
```

---

## 3. Replication Patterns

### Single-Leader Replication

```
Client → Leader (write) → Followers (read)
         ↕ (replication)
      Follower 1, Follower 2, Follower 3

Sync vs Async replication:
- Synchronous: Leader waits for follower ACK before responding to client
  → Durability guaranteed: if leader dies, follower has data
  → Higher write latency: must wait for network round-trip to follower
  
- Asynchronous: Leader responds to client immediately, replicates in background
  → Lower latency
  → Potential data loss if leader dies before replication completes
  
Semi-synchronous (PostgreSQL, MySQL):
  → One follower is synchronous; rest are async
  → Balance: durability without waiting for all followers
```

### Multi-Leader Replication

```
Client → Leader A  ←→ (async replication) ←→  Leader B ← Client
         (DC-1)                                  (DC-2)

Use cases:
- Multi-datacenter deployment
- Mobile apps with offline editing

Write conflicts:
- User A edits title to "A", User B edits same field to "B"
- Both commit locally
- Sync → conflict
- Conflict resolution strategies:
  1. Last Write Wins (LWW): compare timestamps — but clocks are unreliable!
  2. Higher ID wins: arbitrary but consistent
  3. Merge: concatenate or ask user to resolve
  4. CRDT: Conflict-free Replicated Data Types — automatically merge

CRDTs (Conflict-free Replicated Data Types):
- G-Counter: grow-only counter (never decrements)
  → Each node maintains its own counter; merge = max()
- 2P-Set: grow-only set (add and remove-set)
- LWW-Register: last-write-wins register with vector clock
- OR-Set: observed-remove set (handles concurrent add+remove)
- Used in: Redis CRDT (Redis Enterprise), Riak, Apple Notes sync
```

### Leaderless Replication (Dynamo-style)

```
Client → writes to N/W nodes, reads from N/R nodes
         (W = write quorum, R = read quorum, N = replication factor)

For consistency: W + R > N
Example: N=3, W=2, R=2 (read and write to majority)
- Write: must succeed on 2 of 3 nodes
- Read: reads from 2 of 3 nodes, returns latest (by timestamp or vector clock)
- If W+R > N: at least one node in every read overlaps with last write

Sloppy quorum (for availability):
- If some of the N nodes are unavailable, use other available nodes
- Write "hint" — deliver to correct nodes when they recover
- Increases availability but may violate strict consistency

Hinted handoff:
- Node A is responsible for key K
- A is temporarily down
- Writes go to node D instead (sloppy quorum)
- D stores hint: "this belongs to A when A comes back"
- When A recovers, D hands off to A

Used in: Apache Cassandra, Amazon DynamoDB
```

---

## 4. Distributed Transactions

### Two-Phase Commit (2PC)

```
Problem: How to atomically update two databases in different services?

Phase 1 — Prepare:
  Coordinator: "Can you commit?"
  Participant 1 (DB A): "Yes, I've locked resources and am ready"
  Participant 2 (DB B): "Yes, ready"

Phase 2 — Commit:
  Coordinator: "Commit!"
  Participant 1: "Done" 
  Participant 2: "Done"

Problems:
- Coordinator is SPOF: if coordinator crashes after prepare but before commit →
  participants are stuck holding locks waiting for coordinator to recover
- Blocking: all participants blocked until coordinator recovers
- Not partition-tolerant: if network partition between coordinator and participant →
  participant doesn't know if coordinator committed or aborted

Used in: XA transactions (JDBC, JTA), distributed PostgreSQL clusters
```

### Saga Pattern (Preferred for Microservices)

```
Instead of distributed transactions: use a sequence of local transactions
Each local transaction publishes an event/message triggering the next step
If step N fails: compensating transactions rollback steps 1..N-1

Order Placement Saga:
1. Create Order (status=PENDING) [OrderService]
   → if fails: nothing to rollback
2. Reserve Inventory [InventoryService]
   → if fails: → (C1) Cancel Order [OrderService]
3. Process Payment [PaymentService]
   → if fails: → (C2) Release Inventory [InventoryService]
              → (C1) Cancel Order [OrderService]
4. Update Order to CONFIRMED
   → Final step

Choreography (event-driven):
  Each service publishes events and subscribes to others' events
  No central orchestrator
  Pro: loose coupling, simple
  Con: hard to track overall flow, hard to add new steps

Orchestration (central coordinator):
  Saga Orchestrator sends commands to each service in sequence
  Services respond to orchestrator
  Pro: flow is explicit and visible
  Con: orchestrator knows about all services (higher coupling)
  Use: AWS Step Functions, Netflix Conductor, Temporal
```

### Temporal (Workflow Orchestration)

```typescript
// Temporal makes distributed workflows durable
// If a worker crashes mid-workflow, Temporal replays it from where it left off

import { proxyActivities, defineWorkflow } from '@temporalio/workflow'

const { createOrder, reserveInventory, processPayment, confirmOrder } = proxyActivities({
  startToCloseTimeout: '30s',
  retry: {
    maximumAttempts: 3,
    initialInterval: '1s',
    backoffCoefficient: 2,
  },
})

// This function can run for days, survive restarts, and is automatically retried
export async function orderWorkflow(orderId: string): Promise<void> {
  // Step 1: Create order
  const order = await createOrder(orderId)
  
  try {
    // Step 2: Reserve inventory
    await reserveInventory(order.items)
    
    try {
      // Step 3: Process payment
      await processPayment(order.totalAmount, order.paymentMethod)
      
      // Step 4: Confirm
      await confirmOrder(orderId)
      
    } catch (e) {
      // Payment failed — compensate by releasing inventory
      await releaseInventory(order.items)
      throw e
    }
    
  } catch (e) {
    // Inventory failed — compensate by cancelling order
    await cancelOrder(orderId)
    throw e
  }
}
```

---

## 5. Clock Synchronization

### Lamport Timestamps

```
Problem: Two events A and B on different nodes — which happened first?
Solution: Logical timestamps that capture causality

Rules:
1. Each node maintains counter C
2. Before sending a message: C = C + 1; attach C to message
3. On receiving a message with timestamp T: C = max(C, T) + 1

Example:
  Node 1: C=1 (event A)  →  sends msg (C=2) → Node 2
  Node 2: receives (C=max(0,2)+1=3) → event B (C=3)

A happened before B (Lamport ordering)

Limitation: Lamport timestamps can only say "A happened-before B"
but NOT "A did NOT happen-before B" (concurrent events same timestamp)
```

### Vector Clocks

```
Each node maintains a vector of counters [V1, V2, ..., Vn]
V[i] = number of events node i has observed

Compare: A < B if every component of A is ≤ B (and at least one is <)
Concurrent: neither A < B nor B < A

Example (3 nodes):
  Node A: [1,0,0] → sends to B
  Node B: receives → [1,1,0] → sends to C
  Node C: receives → [1,1,1]
  
  [1,0,0] < [1,1,0] < [1,1,1] — clear causal ordering

  Node A: [2,0,0]  ← concurrent with B's [1,1,0]
  → [2,0,0] and [1,1,0] are concurrent (neither is < the other)
```

---

## 6. Failure Detection

### Heartbeats and Phi Accrual Detector

```typescript
// Naive heartbeat detection
// Problem: binary — either alive or dead, no confidence gradation

class NaiveFailureDetector {
  private lastHeartbeat = new Map<string, number>()
  private readonly timeout = 5000  // 5 seconds

  heartbeat(nodeId: string): void {
    this.lastHeartbeat.set(nodeId, Date.now())
  }

  isAlive(nodeId: string): boolean {
    const last = this.lastHeartbeat.get(nodeId) ?? 0
    return Date.now() - last < this.timeout
  }
  // Problem: either definitely dead (> 5s) or definitely alive (< 5s)
  // No confidence level — can't distinguish slow node from dead node
}

// Phi Accrual Failure Detector (Akka, Cassandra)
// Returns a "suspicion level" phi:
// phi = 1  → 10% probability of failure
// phi = 2  → 1% probability
// phi = 3  → 0.1% probability (usually treated as failure threshold)

class PhiAccrualDetector {
  private heartbeatTimes: number[] = []
  
  heartbeat(): void {
    this.heartbeatTimes.push(Date.now())
    if (this.heartbeatTimes.length > 200) {
      this.heartbeatTimes.shift()  // Keep only last 200
    }
  }
  
  phi(): number {
    if (this.heartbeatTimes.length < 2) return 0
    
    const intervals = []
    for (let i = 1; i < this.heartbeatTimes.length; i++) {
      intervals.push(this.heartbeatTimes[i] - this.heartbeatTimes[i-1])
    }
    
    const mean = intervals.reduce((a, b) => a + b) / intervals.length
    const variance = intervals.reduce((a, b) => a + (b - mean) ** 2, 0) / intervals.length
    const stddev = Math.sqrt(variance)
    
    const timeSinceLastHeartbeat = Date.now() - this.heartbeatTimes[this.heartbeatTimes.length - 1]
    
    // Phi: how many standard deviations is the silence from the mean?
    return timeSinceLastHeartbeat / mean  // Simplified version
  }
  
  isSuspect(threshold = 8): boolean {
    return this.phi() >= threshold
  }
}
```

---

## 7. Circuit Breaker Pattern

```typescript
// Problem: Service A calls Service B
// Service B is slow/failing → Service A's threads pile up waiting
// Service A itself becomes slow → cascades to Service C → system-wide failure

enum CircuitState {
  CLOSED = 'CLOSED',    // Normal: requests pass through
  OPEN = 'OPEN',        // Failed: all requests fail fast
  HALF_OPEN = 'HALF_OPEN',  // Recovery: let one request through to test
}

class CircuitBreaker {
  private state = CircuitState.CLOSED
  private failureCount = 0
  private lastFailureTime: number | null = null
  
  constructor(
    private readonly failureThreshold = 5,
    private readonly openDurationMs = 60_000,   // 1 minute open
    private readonly halfOpenSuccessThreshold = 1
  ) {}
  
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      // Check if it's time to try again
      if (this.lastFailureTime && Date.now() - this.lastFailureTime > this.openDurationMs) {
        this.state = CircuitState.HALF_OPEN
      } else {
        throw new Error('Circuit breaker OPEN: service unavailable')
      }
    }
    
    try {
      const result = await fn()
      
      // Success
      if (this.state === CircuitState.HALF_OPEN) {
        this.state = CircuitState.CLOSED  // Recovery successful
        this.failureCount = 0
      }
      
      return result
      
    } catch (error) {
      this.failureCount++
      this.lastFailureTime = Date.now()
      
      if (this.failureCount >= this.failureThreshold || this.state === CircuitState.HALF_OPEN) {
        this.state = CircuitState.OPEN  // Trip the breaker
      }
      
      throw error
    }
  }
}

// Usage
const paymentCircuitBreaker = new CircuitBreaker(5, 60_000)

async function processPayment(amount: number) {
  return paymentCircuitBreaker.execute(() => 
    stripeClient.charges.create({ amount })
  )
}
```

---

## 8. Rate Limiting Algorithms

### Token Bucket

```typescript
// Most common algorithm: allows bursting up to bucket capacity
// Tokens added at fixed rate; each request consumes one token

class TokenBucket {
  private tokens: number
  private lastRefillTime: number
  
  constructor(
    private readonly capacity: number,      // Max burst size
    private readonly refillRate: number,    // Tokens per second
  ) {
    this.tokens = capacity
    this.lastRefillTime = Date.now()
  }
  
  consume(requested = 1): boolean {
    this.refill()
    
    if (this.tokens >= requested) {
      this.tokens -= requested
      return true   // Request allowed
    }
    
    return false  // Rate limited
  }
  
  private refill(): void {
    const now = Date.now()
    const elapsed = (now - this.lastRefillTime) / 1000  // seconds
    const tokensToAdd = elapsed * this.refillRate
    
    this.tokens = Math.min(this.capacity, this.tokens + tokensToAdd)
    this.lastRefillTime = now
  }
}
```

### Sliding Window Counter (Distributed — Redis)

```typescript
// Sliding window: more accurate than fixed window, handles boundary edge cases
// Implementation: Redis sorted set with timestamps as scores

async function isRateLimited(
  userId: string,
  limit: number,
  windowMs: number
): Promise<boolean> {
  const key = `rate_limit:${userId}`
  const now = Date.now()
  const windowStart = now - windowMs
  
  const multi = redis.multi()
  
  // Remove old entries outside the window
  multi.zremrangebyscore(key, '-inf', windowStart)
  
  // Count requests in window
  multi.zcard(key)
  
  // Add current request
  multi.zadd(key, { score: now, member: `${now}-${Math.random()}` })
  
  // Set expiry (cleanup)
  multi.pexpire(key, windowMs)
  
  const results = await multi.exec()
  const count = results[1] as number
  
  return count >= limit
}

// Usage
app.use(async (req, res, next) => {
  const userId = req.user?.id ?? req.ip
  const limited = await isRateLimited(userId, 100, 60_000)  // 100 req/min
  
  if (limited) {
    return res.status(429).json({
      error: 'Rate limit exceeded',
      retryAfter: 60,
    })
  }
  
  next()
})
```

---

## 9. Distributed Locking

### Redis REDLOCK Algorithm

```typescript
// Problem: Distributed lock needs:
// 1. Safety: only one client holds lock at a time
// 2. Liveness: eventually released even if holder crashes
// 3. Fault-tolerance: works even if some Redis nodes fail

// Single Redis node lock (simpler, single SPOF)
async function acquireLock(
  resource: string,
  ttlMs: number
): Promise<string | null> {
  const lockValue = crypto.randomUUID()
  
  // SET NX PX: only set if Not eXists, with millisecond expiry
  const result = await redis.set(`lock:${resource}`, lockValue, {
    NX: true,     // Only set if key doesn't exist
    PX: ttlMs,    // Expire in ttlMs milliseconds
  })
  
  return result === 'OK' ? lockValue : null
}

async function releaseLock(resource: string, lockValue: string): Promise<void> {
  // Must use Lua script for atomic check-and-delete
  // If we just DEL, we might delete someone else's lock if ours expired
  const script = `
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `
  await redis.eval(script, [1, `lock:${resource}`, lockValue])
}

// Usage
async function processJobExclusively(jobId: string): Promise<void> {
  const lock = await acquireLock(`job:${jobId}`, 30_000)
  
  if (!lock) {
    // Another process holds the lock
    return
  }
  
  try {
    await processJob(jobId)
  } finally {
    await releaseLock(`job:${jobId}`, lock)
  }
}
```

---

## 10. Observability in Distributed Systems

### The Three Pillars

```
Metrics (quantitative):
  - CPU, memory, request rate, error rate, latency (p50/p95/p99)
  - Good for: alerting on thresholds, capacity planning, dashboards
  - Low cardinality — aggregate data
  - Tools: Prometheus, Datadog, CloudWatch

Logs (event records):
  - Structured JSON logs with correlation IDs
  - Good for: debugging specific incidents, audit trails
  - High cardinality — every event
  - Tools: ELK stack, Loki, CloudWatch Logs

Traces (distributed request flow):
  - Single request traced across multiple services
  - Shows: where time was spent, where errors occurred, which service is slow
  - Good for: debugging latency issues, service dependency mapping
  - Tools: Jaeger, Zipkin, AWS X-Ray, Datadog APM
```

### Structured Logging with Correlation IDs

```typescript
import { randomUUID } from 'crypto'

// Middleware: assign trace ID to every request
app.use((req, res, next) => {
  const traceId = req.headers['x-trace-id'] as string ?? randomUUID()
  const spanId = randomUUID()
  
  req.traceContext = { traceId, spanId }
  
  // Forward to downstream services
  res.setHeader('x-trace-id', traceId)
  
  // Make available to all log calls in this request
  asyncLocalStorage.run({ traceId, spanId }, next)
})

// Logger: always include trace context
function log(level: string, message: string, data?: object) {
  const context = asyncLocalStorage.getStore() ?? {}
  
  console.log(JSON.stringify({
    timestamp: new Date().toISOString(),
    level,
    message,
    ...context,  // traceId, spanId
    ...data,
    service: 'booking-service',
    version: process.env.APP_VERSION,
    environment: process.env.NODE_ENV,
  }))
}

// When calling downstream services, propagate trace ID
async function callInventoryService(bookingId: string) {
  const context = asyncLocalStorage.getStore()
  
  return fetch('http://inventory-service/api/check', {
    headers: {
      'x-trace-id': context?.traceId,        // Propagate!
      'x-span-id': context?.spanId,
    },
    body: JSON.stringify({ bookingId }),
    method: 'POST',
  })
}
```

### The RED Method (Distributed Services)

```
For every service, instrument three things:

Rate: requests per second
  → metric: http_requests_total (counter)
  → alert: rate drops to 0 (service down)

Errors: error rate
  → metric: http_errors_total (counter)
  → alert: error rate > 1% for 5 minutes

Duration: request latency distribution
  → metric: http_request_duration_seconds (histogram)
  → alert: p99 > 2 seconds

Prometheus metrics:
```
```typescript
import { Counter, Histogram, register } from 'prom-client'

const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
})

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
})

app.use((req, res, next) => {
  const start = Date.now()
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000
    const labels = {
      method: req.method,
      route: req.route?.path ?? 'unknown',
      status_code: String(res.statusCode),
    }
    
    httpRequestsTotal.inc(labels)
    httpRequestDuration.observe(labels, duration)
  })
  
  next()
})

// Expose /metrics endpoint for Prometheus to scrape
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType)
  res.end(await register.metrics())
})
```
