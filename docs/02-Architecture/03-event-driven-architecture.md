# Event-Driven Architecture

> **Category:** Architecture
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Event Types](#2-event-types)
3. [Apache Kafka Deep Dive](#3-apache-kafka-deep-dive)
4. [Event Sourcing](#4-event-sourcing)
5. [CQRS (Command Query Responsibility Segregation)](#5-cqrs-command-query-responsibility-segregation)
6. [Outbox Pattern](#6-outbox-pattern)
7. [Dead Letter Queues](#7-dead-letter-queues)
8. [Schema Registry and Evolution](#8-schema-registry-and-evolution)
9. [Event-Driven Microservices Patterns](#9-event-driven-microservices-patterns)
10. [Testing Event-Driven Systems](#10-testing-event-driven-systems)
11. [When Not to Use EDA](#11-when-not-to-use-eda)

---

## 1. Core Concepts

### What Is an Event?

```
An event is an immutable record that something happened in the past.

Domain event: "OrderPlaced", "PaymentFailed", "UserDeactivated"
- Past tense (it already happened)
- Immutable (cannot be changed)
- Contains all information needed to understand what happened
- Has a timestamp
- Has an identity (event ID)

vs. Command: "PlaceOrder", "ProcessPayment" (intent — may fail or be rejected)
vs. Message: generic communication; may carry either
```

### Event-Driven vs Request-Driven

```
Request-Driven (synchronous):
  Client → Service A → Service B → Service C → Response
  
  + Simple, easy to understand
  + Immediate response
  - Temporal coupling (B must be available when A calls)
  - Spatial coupling (A must know where B is)
  - Cascade failures (B slow → A slow → Client slow)

Event-Driven (asynchronous):
  Producer → Event Bus → Consumer 1
                       → Consumer 2
                       → Consumer 3

  + Loose coupling (producer doesn't know about consumers)
  + Resilient (consumers can be offline, events queue up)
  + Scalable (add consumers without changing producers)
  - Eventually consistent (no immediate response)
  - Harder to debug (no linear request trace)
  - Ordering guarantees are complex
```

### Broker vs Broker-Less

```
Broker-based (most common):
  Producer → Kafka/RabbitMQ/SQS → Consumer
  
  + Durable (broker persists messages)
  + Decoupled (neither knows about each other)
  + Load leveling (broker buffers spikes)
  - Single component (broker must be available)
  - Additional infrastructure

Broker-less (gRPC streaming, ZeroMQ):
  Producer → Consumer (direct, P2P)
  
  + Lower latency (no broker hop)
  + Simpler (no broker to manage)
  - Tight coupling (direct connection required)
  - No durability without consumer storing
```

---

## 2. Event Types

### Domain Events

```typescript
// Domain events: significant business things that happened
interface DomainEvent {
  id: string           // Unique event ID (UUID)
  type: string         // "order.placed", "payment.failed"
  aggregateId: string  // The business entity this is about
  aggregateType: string  // "Order", "User"
  version: number      // Event schema version (for evolution)
  timestamp: string    // ISO 8601
  payload: unknown     // Event-specific data
  metadata: {
    correlationId: string  // Trace the original request
    causationId: string    // ID of the event/command that caused this
    userId?: string        // Who triggered the action
    source: string         // Service that produced this
  }
}

// Concrete event examples
interface OrderPlacedEvent extends DomainEvent {
  type: 'order.placed'
  payload: {
    orderId: string
    customerId: string
    items: Array<{ productId: string; quantity: number; price: number }>
    totalAmount: number
    currency: string
    shippingAddress: Address
  }
}

interface PaymentFailedEvent extends DomainEvent {
  type: 'payment.failed'
  payload: {
    orderId: string
    customerId: string
    amount: number
    currency: string
    reason: 'insufficient_funds' | 'card_declined' | 'expired' | 'fraud_detected'
    stripeErrorCode?: string
  }
}
```

### Event Naming Conventions

```
<aggregate>.<past-tense-action>

user.registered
user.email_verified
user.deactivated
user.password_changed

order.placed
order.confirmed
order.shipped
order.delivered
order.cancelled
order.refund_requested
order.refunded

payment.initiated
payment.succeeded
payment.failed
payment.refunded

inventory.reserved
inventory.released
inventory.low_stock
inventory.depleted
```

---

## 3. Apache Kafka Deep Dive

### Architecture

```
Kafka Cluster:
  Broker 1 (Leader for partitions 0, 3)
  Broker 2 (Leader for partitions 1, 4)
  Broker 3 (Leader for partitions 2, 5)

Topic: "order-events"
  Partition 0: [msg1, msg4, msg7, ...]
  Partition 1: [msg2, msg5, msg8, ...]
  Partition 2: [msg3, msg6, msg9, ...]

Each partition is replicated across N brokers (replication factor)
One broker is leader for each partition — handles all reads and writes
Other brokers are followers — replicate from leader

Consumer Group "payment-service":
  Consumer A → reads Partition 0
  Consumer B → reads Partition 1
  Consumer C → reads Partition 2

Adding Consumer D → rebalance: each consumer reads 0.75 partitions (idle partition)
Rule: can't have more consumers than partitions in a group (excess consumers are idle)
```

### Kafka Producer

```typescript
import { Kafka, Producer, CompressionTypes } from 'kafkajs'

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: ['kafka-1:9092', 'kafka-2:9092', 'kafka-3:9092'],
  ssl: true,
  sasl: {
    mechanism: 'scram-sha-512',
    username: process.env.KAFKA_USERNAME!,
    password: process.env.KAFKA_PASSWORD!,
  },
  retry: {
    initialRetryTime: 100,
    retries: 8,
  },
})

const producer: Producer = kafka.producer({
  maxInFlightRequests: 1,         // Required for idempotent producer
  idempotent: true,               // Exactly-once delivery semantics
  transactionalId: 'order-service-txn',  // Enable transactions
  compression: CompressionTypes.SNAPPY,
})

await producer.connect()

// Publish a domain event
async function publishOrderPlaced(order: Order): Promise<void> {
  const event: OrderPlacedEvent = {
    id: randomUUID(),
    type: 'order.placed',
    aggregateId: order.id,
    aggregateType: 'Order',
    version: 1,
    timestamp: new Date().toISOString(),
    payload: {
      orderId: order.id,
      customerId: order.customerId,
      items: order.items,
      totalAmount: order.totalAmount,
      currency: order.currency,
      shippingAddress: order.shippingAddress,
    },
    metadata: {
      correlationId: asyncLocalStorage.getStore()?.traceId ?? randomUUID(),
      causationId: order.id,
      userId: order.customerId,
      source: 'order-service',
    },
  }

  await producer.send({
    topic: 'order-events',
    messages: [{
      key: order.id,    // Partition key: same order ID → same partition → ordered
      value: JSON.stringify(event),
      headers: {
        'event-type': event.type,
        'source': 'order-service',
      },
    }],
  })
}
```

### Kafka Consumer

```typescript
import { Consumer, EachMessagePayload } from 'kafkajs'

const consumer: Consumer = kafka.consumer({
  groupId: 'payment-service',
  maxBytesPerPartition: 1_048_576,  // 1MB per partition per fetch
  sessionTimeout: 30_000,
  heartbeatInterval: 3_000,
})

await consumer.connect()
await consumer.subscribe({ 
  topics: ['order-events'],
  fromBeginning: false  // Start from latest (change to true for replay)
})

await consumer.run({
  autoCommit: false,  // Manual commit for at-least-once guarantees
  eachMessage: async ({ topic, partition, message, heartbeat }: EachMessagePayload) => {
    const event = JSON.parse(message.value!.toString())
    
    // Route to handler by event type
    try {
      switch (event.type) {
        case 'order.placed':
          await handleOrderPlaced(event)
          break
        case 'order.cancelled':
          await handleOrderCancelled(event)
          break
        default:
          // Unknown event type — log and skip (idempotency: ignore unknowns)
          console.warn('Unknown event type:', event.type)
      }
      
      // Commit offset only after successful processing
      await consumer.commitOffsets([{
        topic,
        partition,
        offset: (Number(message.offset) + 1).toString(),
      }])
      
    } catch (error) {
      // Don't commit — message will be redelivered
      // After max retries: move to dead letter topic
      await sendToDeadLetterTopic(topic, message, error)
      
      // Commit so we don't get stuck retrying forever in consumer
      await consumer.commitOffsets([{
        topic,
        partition,
        offset: (Number(message.offset) + 1).toString(),
      }])
    }
  },
})
```

### Kafka Configuration Best Practices

```properties
# Producer settings
acks=all                    # Wait for all ISR replicas to acknowledge
min.insync.replicas=2       # At least 2 replicas must be in sync
enable.idempotence=true     # Exactly-once writes from producer side
retries=2147483647          # Retry forever (with backoff)
max.in.flight.requests.per.connection=1  # Required with idempotence=true

# Consumer settings
enable.auto.commit=false    # Manual commit for at-least-once
auto.offset.reset=earliest  # For new consumer groups (replay all history)
                            # OR latest (skip old messages)

# Topic settings (via kafka-topics.sh or Terraform provider)
replication.factor=3        # 3 replicas for fault tolerance (survive 2 broker failures)
min.insync.replicas=2       # Must have 2 in-sync before accepting writes
retention.ms=604800000      # 7 days retention
compression.type=snappy     # Compress topic (good ratio, fast compression)
```

---

## 4. Event Sourcing

Event sourcing stores the history of events instead of the current state. Current state is derived by replaying events.

### Traditional vs Event-Sourced

```
Traditional (state-based):
  Database: { orderId: "1", status: "shipped", updatedAt: "..." }
  → Only current state; history lost

Event-Sourced:
  Events: [
    { type: "OrderPlaced",    at: "10:00", payload: {...} },
    { type: "PaymentCharged", at: "10:01", payload: {...} },
    { type: "OrderShipped",   at: "10:30", payload: {...} },
  ]
  → Full history; current state = replay all events
```

### Event Store Implementation

```typescript
interface StoredEvent {
  id: string
  streamId: string    // e.g., "Order-123"
  version: number     // Sequential within stream
  type: string
  payload: string     // JSON
  timestamp: Date
}

class EventStore {
  async append(streamId: string, events: DomainEvent[], expectedVersion: number): Promise<void> {
    await db.transaction(async (trx) => {
      // Optimistic locking: verify no concurrent writes
      const current = await trx.query(
        'SELECT MAX(version) as max_version FROM events WHERE stream_id = $1',
        [streamId]
      )
      
      const currentVersion = current.rows[0].max_version ?? -1
      
      if (currentVersion !== expectedVersion) {
        throw new OptimisticConcurrencyError(
          `Stream ${streamId}: expected version ${expectedVersion}, got ${currentVersion}`
        )
      }
      
      for (let i = 0; i < events.length; i++) {
        await trx.query(
          'INSERT INTO events (id, stream_id, version, type, payload, timestamp) VALUES ($1, $2, $3, $4, $5, $6)',
          [events[i].id, streamId, expectedVersion + i + 1, events[i].type, JSON.stringify(events[i].payload), new Date()]
        )
      }
    })
    
    // Publish events to Kafka for other services
    await publishEvents(events)
  }
  
  async load(streamId: string, fromVersion = 0): Promise<StoredEvent[]> {
    const result = await db.query(
      'SELECT * FROM events WHERE stream_id = $1 AND version >= $2 ORDER BY version ASC',
      [streamId, fromVersion]
    )
    return result.rows
  }
}

// Reconstruct order from events
class Order {
  id: string
  status: 'pending' | 'confirmed' | 'shipped' | 'cancelled' = 'pending'
  items: OrderItem[] = []
  version = -1
  
  static fromEvents(events: StoredEvent[]): Order {
    const order = new Order()
    for (const event of events) {
      order.apply(JSON.parse(event.payload), event.type)
      order.version = event.version
    }
    return order
  }
  
  private apply(payload: any, type: string): void {
    switch (type) {
      case 'OrderPlaced':
        this.id = payload.orderId
        this.items = payload.items
        this.status = 'pending'
        break
      case 'PaymentSucceeded':
        this.status = 'confirmed'
        break
      case 'OrderShipped':
        this.status = 'shipped'
        break
      case 'OrderCancelled':
        this.status = 'cancelled'
        break
    }
  }
}
```

### Snapshots (Performance)

```typescript
// Problem: order with 1000 events takes 1000 DB reads to reconstruct
// Solution: snapshot = materialized state at version N; replay only events after N

interface Snapshot {
  streamId: string
  version: number
  state: string  // JSON of materialized state
  createdAt: Date
}

class OrderRepository {
  async load(orderId: string): Promise<Order> {
    // 1. Load latest snapshot
    const snapshot = await db.query(
      'SELECT * FROM snapshots WHERE stream_id = $1 ORDER BY version DESC LIMIT 1',
      [`Order-${orderId}`]
    )
    
    let order: Order
    let fromVersion: number
    
    if (snapshot.rows.length) {
      order = Order.fromSnapshot(JSON.parse(snapshot.rows[0].state))
      fromVersion = snapshot.rows[0].version + 1
    } else {
      order = new Order()
      fromVersion = 0
    }
    
    // 2. Load and apply events since snapshot
    const events = await eventStore.load(`Order-${orderId}`, fromVersion)
    for (const event of events) {
      order.apply(JSON.parse(event.payload), event.type)
    }
    
    // 3. Create new snapshot every 50 events (amortize replay cost)
    if (events.length >= 50) {
      await this.createSnapshot(order)
    }
    
    return order
  }
}
```

---

## 5. CQRS (Command Query Responsibility Segregation)

Separate the write model (commands that change state) from the read model (queries that return data).

```
                  ┌─────────────────┐
Commands ────────→│  Write Model    │──→ Event Store / DB
                  │  (Aggregates)   │
                  └─────────────────┘
                          │
                      Domain Events
                          │
                          ▼
                  ┌─────────────────┐
                  │  Projections    │──→ Read Model (optimized for queries)
                  │  (Event        │    - Materialized views
                  │   Handlers)    │    - Search indexes
                  └─────────────────┘
Queries ─────────────────────────────→ Read Model (fast reads)
```

### Read Model (Projection)

```typescript
// Write side: normalized, event-sourced
// Read side: denormalized, optimized for specific queries

// Projection: builds order summary from events
// Stored in a separate, read-optimized table
interface OrderSummary {
  orderId: string
  customerName: string
  customerEmail: string
  status: string
  totalAmount: number
  currency: string
  itemCount: number
  createdAt: Date
  lastUpdatedAt: Date
}

// Event handler that updates the read model
async function handleOrderEvent(event: StoredEvent): Promise<void> {
  const payload = JSON.parse(event.payload)
  
  switch (event.type) {
    case 'OrderPlaced':
      const customer = await customerService.get(payload.customerId)
      await readDb.query(`
        INSERT INTO order_summaries (
          order_id, customer_name, customer_email, status, 
          total_amount, currency, item_count, created_at, last_updated_at
        ) VALUES ($1, $2, $3, 'pending', $4, $5, $6, $7, $7)
      `, [payload.orderId, customer.name, customer.email, 
          payload.totalAmount, payload.currency, payload.items.length, new Date()])
      break
      
    case 'OrderShipped':
      await readDb.query(
        'UPDATE order_summaries SET status = $1, last_updated_at = $2 WHERE order_id = $3',
        ['shipped', new Date(), payload.orderId]
      )
      break
  }
}

// Query: extremely fast (no joins, pre-computed)
app.get('/api/orders', async (req, res) => {
  const { status, page = 1 } = req.query
  
  const orders = await readDb.query(`
    SELECT * FROM order_summaries 
    WHERE customer_email = $1 ${status ? 'AND status = $2' : ''}
    ORDER BY created_at DESC
    LIMIT 20 OFFSET $${status ? 3 : 2}
  `, status ? [req.user.email, status, (page - 1) * 20] : [req.user.email, (page - 1) * 20])
  
  res.json(orders.rows)
})
```

---

## 6. Outbox Pattern

The Outbox Pattern solves the "dual write" problem: atomically write to DB and publish an event.

### The Problem

```typescript
// WRONG — dual write, no atomicity
async function placeOrder(order: Order): Promise<void> {
  await db.query('INSERT INTO orders ...')  // Succeeds
  await kafka.produce('order-events', ...)  // Fails! ← order saved but event lost
  
  // Or: Kafka succeeds, DB fails → event without order
}
```

### Outbox Solution

```typescript
// CORRECT — outbox pattern
async function placeOrder(order: Order): Promise<void> {
  await db.transaction(async (trx) => {
    // 1. Save the order
    await trx.query('INSERT INTO orders ...', [order])
    
    // 2. Write event to outbox (same transaction!)
    await trx.query(`
      INSERT INTO outbox_events (id, type, aggregate_id, payload, created_at, processed)
      VALUES ($1, $2, $3, $4, NOW(), false)
    `, [randomUUID(), 'order.placed', order.id, JSON.stringify(order)])
    
    // Transaction commits — both order and event in outbox are saved atomically
    // If either fails, transaction rolls back — both fail, consistent state
  })
}

// Background worker: reads outbox and publishes to Kafka
// Runs every 100ms (or uses PostgreSQL LISTEN/NOTIFY for real-time)
async function outboxWorker(): Promise<void> {
  while (true) {
    const events = await db.query(`
      SELECT * FROM outbox_events 
      WHERE processed = false 
      ORDER BY created_at ASC 
      LIMIT 100
      FOR UPDATE SKIP LOCKED  // Distributed lock for multiple workers
    `)
    
    for (const event of events.rows) {
      await kafka.produce('order-events', {
        key: event.aggregate_id,
        value: event.payload,
      })
      
      await db.query(
        'UPDATE outbox_events SET processed = true, processed_at = NOW() WHERE id = $1',
        [event.id]
      )
    }
    
    await sleep(100)
  }
}

// Debezium (preferred for production): CDC — reads PostgreSQL WAL
// No polling, no background worker, events published milliseconds after commit
// Debezium reads the Postgres write-ahead log (WAL) and publishes changes to Kafka
```

---

## 7. Dead Letter Queues

```typescript
// When a consumer fails to process a message after N retries: send to DLQ
// DLQ = staging area for failed messages
// Operations team can inspect, fix, and replay from DLQ

interface DeadLetterMessage {
  originalTopic: string
  originalPartition: number
  originalOffset: string
  originalMessage: string
  error: string
  stackTrace: string
  failedAt: Date
  retryCount: number
  firstFailedAt: Date
}

async function sendToDeadLetterTopic(
  originalTopic: string,
  message: KafkaMessage,
  error: Error,
  retryCount: number
): Promise<void> {
  const dlqMessage: DeadLetterMessage = {
    originalTopic,
    originalPartition: message.partition,
    originalOffset: message.offset,
    originalMessage: message.value?.toString() ?? '',
    error: error.message,
    stackTrace: error.stack ?? '',
    failedAt: new Date(),
    retryCount,
    firstFailedAt: message.timestamp ? new Date(Number(message.timestamp)) : new Date(),
  }
  
  await producer.send({
    topic: `${originalTopic}.dlq`,
    messages: [{
      key: message.key,
      value: JSON.stringify(dlqMessage),
      headers: {
        'x-original-topic': originalTopic,
        'x-error': error.message,
        'x-retry-count': String(retryCount),
      },
    }],
  })
}

// DLQ monitoring alert
// If DLQ is non-empty: alert on-call (processing failure in production)
// Operator: inspect message, fix bug, replay message to original topic
```

---

## 8. Schema Registry and Evolution

```typescript
// Schema Registry (Confluent):
// - Stores Avro/Protobuf/JSON Schema schemas
// - Validates message compatibility before publish
// - Enables schema evolution without breaking consumers

// Backward compatible changes (safe):
//   Adding optional fields with defaults
//   Renaming fields (with aliases)
// Backward incompatible (breaking):
//   Removing required fields
//   Changing field types
//   Renaming without aliases

// Avro schema for OrderPlaced event
const OrderPlacedV1 = {
  type: 'record',
  name: 'OrderPlaced',
  namespace: 'com.example.events',
  fields: [
    { name: 'orderId', type: 'string' },
    { name: 'customerId', type: 'string' },
    { name: 'totalAmount', type: 'double' },
  ]
}

// Evolved schema (adding optional field — backward compatible)
const OrderPlacedV2 = {
  type: 'record',
  name: 'OrderPlaced',
  namespace: 'com.example.events',
  fields: [
    { name: 'orderId', type: 'string' },
    { name: 'customerId', type: 'string' },
    { name: 'totalAmount', type: 'double' },
    { name: 'currency', type: { type: 'string', default: 'USD' } }  // Optional, backward compat
  ]
}

// Old consumers reading V2 messages: ignore unknown field 'currency'
// New consumers reading V1 messages: use default value 'USD' for missing field
```

---

## 9. Event-Driven Microservices Patterns

### Choreography vs Orchestration (Saga)

```
Choreography (event-driven, decentralized):
  OrderService → publishes OrderPlaced
  InventoryService → subscribes → publishes InventoryReserved
  PaymentService → subscribes → publishes PaymentCharged
  NotificationService → subscribes → sends confirmation email
  
  Pro: fully decoupled services
  Con: business flow is implicit (hidden in events); hard to audit

Orchestration (workflow, centralized):
  OrderOrchestrator:
    1. Command InventoryService.ReserveInventory
    2. Command PaymentService.ChargePayment
    3. Command NotificationService.SendConfirmation
    4. Update order status
  
  Pro: explicit workflow; easy to add steps; visible monitoring
  Con: Orchestrator knows about all services; potential bottleneck

Use Temporal, AWS Step Functions, or Netflix Conductor for orchestration.
```

### Event-Driven vs Direct Communication Decision Tree

```
Should service A call service B directly?

1. Is this a query (read-only)? → YES → REST/gRPC direct call (synchronous is fine)

2. Does A need an immediate response from B? → YES → gRPC (synchronous, low latency)

3. Is the action time-sensitive (< 100ms)? → YES → gRPC direct

4. Is B allowed to be temporarily unavailable without A failing? → YES → async event

5. Do multiple services need to react to the same action? → YES → async event (fan-out)

6. Is this a "fire and forget" notification? → YES → async event

7. Default: lean toward async events (better resilience and scalability)
```

---

## 10. Testing Event-Driven Systems

```typescript
// Unit test for event handler
describe('handleOrderPlaced', () => {
  it('should reserve inventory when order is placed', async () => {
    const mockInventoryRepo = { reserve: jest.fn().mockResolvedValue(true) }
    const handler = new OrderPlacedHandler(mockInventoryRepo)
    
    const event: OrderPlacedEvent = {
      id: 'evt-1',
      type: 'order.placed',
      aggregateId: 'order-1',
      aggregateType: 'Order',
      version: 1,
      timestamp: new Date().toISOString(),
      payload: { orderId: 'order-1', items: [{ productId: 'prod-1', quantity: 2 }] },
      metadata: { correlationId: 'corr-1', causationId: 'order-1', source: 'order-service' },
    }
    
    await handler.handle(event)
    
    expect(mockInventoryRepo.reserve).toHaveBeenCalledWith([
      { productId: 'prod-1', quantity: 2 }
    ])
  })
})

// Integration test with TestContainers (real Kafka)
import { GenericContainer } from 'testcontainers'
import { Kafka } from 'kafkajs'

describe('Order events integration', () => {
  let kafkaContainer: any
  let kafka: Kafka
  
  beforeAll(async () => {
    kafkaContainer = await new GenericContainer('confluentinc/cp-kafka:latest')
      .withEnvironment({ KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true' })
      .withExposedPorts(9092)
      .start()
    
    kafka = new Kafka({ brokers: [`localhost:${kafkaContainer.getMappedPort(9092)}`] })
  })
  
  afterAll(() => kafkaContainer.stop())
  
  it('should process order placed and reserve inventory', async () => {
    const producer = kafka.producer()
    const consumer = kafka.consumer({ groupId: 'test-group' })
    
    await producer.connect()
    await consumer.connect()
    await consumer.subscribe({ topic: 'order-events' })
    
    // Publish event
    await producer.send({
      topic: 'order-events',
      messages: [{ value: JSON.stringify({ type: 'order.placed', payload: { orderId: 'test-1' } }) }]
    })
    
    // Verify processing
    const received = await new Promise<any>(resolve => {
      consumer.run({
        eachMessage: async ({ message }) => {
          resolve(JSON.parse(message.value!.toString()))
        }
      })
    })
    
    expect(received.type).toBe('order.placed')
  })
})
```

---

## 11. When Not to Use EDA

```
Avoid event-driven when:

1. Simple CRUD operations with no side effects
   - Creating a profile photo → no need for events
   - Reading a blog post → synchronous query is fine

2. Strict consistency required
   - Bank transfer: must complete atomically (use distributed transactions, not events)
   - Inventory management: overselling risk with eventual consistency

3. Small-scale applications
   - Monolith with < 5 services: events add complexity without benefit
   - < 1000 RPM: no need for queue-based load leveling

4. When timing/ordering is critical and cannot be relaxed
   - Trading systems: microsecond ordering matters; Kafka partitions help but add complexity
   - Real-time multiplayer games: latency requirements exclude async messaging

5. Debugging difficulty is unacceptable
   - Simple internal tools where a linear request trace is valuable
   - Prototypes and MVPs (defer EDA to when you need it)

Signs you need EDA (growth triggers):
- Multiple services need to react to the same business event
- Service B failures cause cascading failures in Service A
- Need audit trail of all state changes
- Load spikes overwhelm synchronous processing
- Services need to be independently deployable
```
