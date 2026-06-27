# PostgreSQL Patterns for Production

> **Category:** Database
> **Version:** 1.0.0
> **Level:** Staff Engineer

---

## Table of Contents

1. [Schema Design Principles](#1-schema-design-principles)
2. [Indexing Strategy](#2-indexing-strategy)
3. [Query Optimization](#3-query-optimization)
4. [Transactions and Locking](#4-transactions-and-locking)
5. [Partitioning](#5-partitioning)
6. [Full-Text Search](#6-full-text-search)
7. [JSON and JSONB](#7-json-and-jsonb)
8. [Row Level Security](#8-row-level-security)
9. [Connection Management](#9-connection-management)
10. [Migrations](#10-migrations)
11. [Backup and Recovery](#11-backup-and-recovery)
12. [Performance Monitoring](#12-performance-monitoring)

---

## 1. Schema Design Principles

### Data Types

```sql
-- UUIDs: use gen_random_uuid() (pg 13+) or uuid_generate_v4() (pgcrypto)
-- For high-insert tables: UUIDv7 (time-ordered) > UUIDv4 (random)
-- Random UUIDs cause B-tree fragmentation on insert; time-ordered UUIDs append

CREATE TABLE users (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email       TEXT NOT NULL,
  name        TEXT NOT NULL,
  role        TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin', 'staff')),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),  -- Always use TIMESTAMPTZ (timezone-aware)
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Monetary values: use NUMERIC not FLOAT
-- FLOAT is approximate (0.1 + 0.2 = 0.30000000000000004)
-- NUMERIC(12, 2) = exact decimal, up to 12 digits, 2 after decimal point
CREATE TABLE payments (
  id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  amount    NUMERIC(12, 2) NOT NULL,  -- Never FLOAT for money
  currency  CHAR(3) NOT NULL DEFAULT 'BRL',
  ...
);

-- Status fields: TEXT with CHECK constraint vs ENUM
-- TEXT + CHECK: easier to evolve (add values without ALTER TYPE)
-- ENUM: type-safe, smaller storage (4 bytes), slightly faster comparisons
-- Recommendation: TEXT + CHECK for evolving states; ENUM for stable small sets

status TEXT NOT NULL DEFAULT 'pending' 
  CHECK (status IN ('pending', 'confirmed', 'cancelled', 'completed')),

-- vs

CREATE TYPE booking_status AS ENUM ('pending', 'confirmed', 'cancelled', 'completed');
status booking_status NOT NULL DEFAULT 'pending',
```

### Naming Conventions

```sql
-- Tables: snake_case, plural
-- Columns: snake_case
-- Indexes: idx_{table}_{columns}
-- Foreign keys: fk_{table}_{referenced_table}
-- Constraints: uq_{table}_{columns} for UNIQUE, chk_{table}_{constraint} for CHECK

CREATE TABLE bookings (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  client_id       UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  service_id      UUID NOT NULL REFERENCES services(id) ON DELETE RESTRICT,
  scheduled_at    TIMESTAMPTZ NOT NULL,
  duration_min    INTEGER NOT NULL,
  status          TEXT NOT NULL DEFAULT 'pending',
  notes           TEXT,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  
  CONSTRAINT chk_bookings_duration CHECK (duration_min > 0 AND duration_min <= 480),
  CONSTRAINT chk_bookings_status CHECK (status IN ('pending', 'confirmed', 'cancelled', 'completed')),
  CONSTRAINT uq_bookings_slot UNIQUE (service_id, scheduled_at)  -- No double-booking same service
);

-- Auto-update updated_at via trigger
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_bookings_updated_at
  BEFORE UPDATE ON bookings
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

---

## 2. Indexing Strategy

### B-tree Index (Default)

```sql
-- Single column (most common)
CREATE INDEX idx_bookings_client_id ON bookings(client_id);
CREATE INDEX idx_bookings_status ON bookings(status);

-- Composite: column order matters
-- Rule: put equality conditions first, range conditions last
-- WHERE client_id = $1 AND scheduled_at > $2 ORDER BY scheduled_at ASC
CREATE INDEX idx_bookings_client_scheduled ON bookings(client_id, scheduled_at);
-- This index serves: WHERE client_id = $1, WHERE client_id = $1 AND scheduled_at > $2
-- It does NOT serve: WHERE scheduled_at > $2 (only range column, leftmost not used)

-- Partial index: smaller, faster when condition matches most queries
CREATE INDEX idx_pending_bookings ON bookings(scheduled_at)
WHERE status = 'pending';  -- Only index pending bookings
-- Use when: 90% of queries filter by status = 'pending'
-- Avoids indexing cancelled/completed records (saves space)

-- Index with covering column (index-only scan)
CREATE INDEX idx_bookings_client_scheduled_status ON bookings(client_id, scheduled_at)
INCLUDE (status, duration_min);  -- Covered columns (not part of key, but stored in index)
-- Query can be answered entirely from index (no table heap access = faster)

-- Unique index (enforces constraint + creates index)
CREATE UNIQUE INDEX uq_users_email ON users(email);
CREATE UNIQUE INDEX uq_users_email_lower ON users(lower(email));  -- Case-insensitive
```

### GIN Index (Full-Text Search, Arrays, JSONB)

```sql
-- Full-text search
CREATE INDEX idx_services_fts ON services 
USING GIN(to_tsvector('portuguese', title || ' ' || COALESCE(description, '')));

-- JSONB fields
CREATE INDEX idx_users_metadata ON users USING GIN(metadata);

-- Array columns
CREATE INDEX idx_events_tags ON events USING GIN(tags);

-- Query
SELECT * FROM services 
WHERE to_tsvector('portuguese', title || ' ' || COALESCE(description, '')) 
  @@ to_tsquery('portuguese', 'massagem & relaxamento');
```

### Index Maintenance

```sql
-- Find unused indexes (waste of write overhead)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan < 50  -- Used less than 50 times since last stats reset
  AND NOT indexname LIKE '%pkey%'  -- Exclude primary keys
ORDER BY idx_scan;

-- Find missing indexes (sequential scans on large tables)
SELECT schemaname, tablename, seq_scan, seq_tup_read, 
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 100  -- Many sequential scans
  AND n_live_tup > 10000  -- On large tables
ORDER BY seq_scan DESC;

-- Bloated indexes (fragmented, needs REINDEX)
-- pg_bloat from pgstattuple extension
SELECT indexname, bloat_ratio
FROM pg_bloat_index
WHERE bloat_ratio > 30  -- 30%+ bloat: consider REINDEX CONCURRENTLY
ORDER BY bloat_ratio DESC;

-- Online reindex (no table lock!)
REINDEX INDEX CONCURRENTLY idx_bookings_client_id;
```

---

## 3. Query Optimization

### EXPLAIN ANALYZE

```sql
-- Always use EXPLAIN (ANALYZE, BUFFERS) in development
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) 
SELECT b.*, u.name as client_name, s.title as service_title
FROM bookings b
JOIN users u ON u.id = b.client_id
JOIN services s ON s.id = b.service_id
WHERE b.client_id = 'user-123'
  AND b.status = 'pending'
  AND b.scheduled_at > NOW()
ORDER BY b.scheduled_at ASC;

-- Key nodes to look for:
-- "Seq Scan" on large table → missing index
-- "Nested Loop" with large outer result → consider Hash Join
-- "Sort" with high cost → index that covers ORDER BY
-- "Buffers: read" → cold cache; "hit" → cache hit (good)
-- "Rows Removed by Filter" >> "Rows" → bad index selectivity

-- pg_stat_statements: find slow queries
SELECT query, calls, mean_exec_time, max_exec_time, total_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- Average > 100ms
ORDER BY mean_exec_time DESC
LIMIT 20;
```

### N+1 Query Prevention

```sql
-- BAD: N+1 — one query per booking to get client name
-- for each booking: SELECT name FROM users WHERE id = booking.client_id
-- If 100 bookings: 101 queries total

-- GOOD: single JOIN
SELECT 
  b.id,
  b.scheduled_at,
  b.status,
  u.name AS client_name,
  u.email AS client_email,
  s.title AS service_title,
  s.duration_min
FROM bookings b
JOIN users u ON u.id = b.client_id
JOIN services s ON s.id = b.service_id
WHERE b.scheduled_at::date = CURRENT_DATE
ORDER BY b.scheduled_at;

-- For optional related data: LEFT JOIN
SELECT 
  b.id,
  b.scheduled_at,
  u.name,
  r.rating,  -- May be NULL if no review yet
  r.comment
FROM bookings b
JOIN users u ON u.id = b.client_id
LEFT JOIN reviews r ON r.booking_id = b.id
WHERE b.status = 'completed';
```

### Pagination

```sql
-- OFFSET pagination: simple but gets slower as pages increase
-- Fetches and discards all previous rows on each page
SELECT * FROM bookings ORDER BY created_at DESC LIMIT 20 OFFSET 1000;
-- Page 50 = scan 1000 rows to discard them

-- Cursor pagination (production-grade)
-- First page
SELECT id, scheduled_at, status FROM bookings
WHERE client_id = $1
ORDER BY scheduled_at DESC, id DESC
LIMIT 21;  -- Fetch 21 to check if there's a next page

-- Next page (use last row's values as cursor)
SELECT id, scheduled_at, status FROM bookings
WHERE client_id = $1
  AND (scheduled_at, id) < ($cursor_scheduled_at, $cursor_id)  -- Composite cursor
ORDER BY scheduled_at DESC, id DESC
LIMIT 21;
```

---

## 4. Transactions and Locking

### Isolation Levels

```sql
-- READ COMMITTED (default): each statement sees committed data
-- REPEATABLE READ: same query same results throughout transaction
-- SERIALIZABLE: full ACID, highest isolation

-- For concurrent updates (avoid lost updates):
-- Option 1: SELECT FOR UPDATE (pessimistic lock)
BEGIN;
  SELECT * FROM accounts WHERE id = $1 FOR UPDATE;  -- Lock the row
  UPDATE accounts SET balance = balance - $2 WHERE id = $1;
COMMIT;

-- Option 2: Optimistic locking (with version column)
-- Check version, fail if changed
UPDATE bookings 
SET status = 'confirmed', version = version + 1
WHERE id = $1 AND version = $expected_version;  -- Fails if version changed
-- Check: if rowcount = 0 → conflict, retry

-- Option 3: UPDATE with RETURNING (most efficient)
UPDATE accounts 
SET balance = balance - $amount
WHERE id = $account_id AND balance >= $amount  -- Atomic check and update
RETURNING balance;
-- If no rows returned: insufficient balance (race condition safe)
```

### Deadlock Prevention

```sql
-- Deadlocks occur when T1 locks A then B; T2 locks B then A simultaneously
-- Prevention: always acquire locks in consistent order

-- WRONG: different lock orders can deadlock
-- Transaction 1: UPDATE accounts WHERE id = A; then UPDATE accounts WHERE id = B
-- Transaction 2: UPDATE accounts WHERE id = B; then UPDATE accounts WHERE id = A

-- CORRECT: always lock in ascending ID order
-- Application enforces: sort IDs before acquiring locks
async function transfer(fromId: string, toId: string, amount: number) {
  const [firstId, secondId] = [fromId, toId].sort()  // Consistent order
  
  await db.transaction(async trx => {
    const [first, second] = await Promise.all([
      trx.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [firstId]),
      trx.query('SELECT * FROM accounts WHERE id = $1 FOR UPDATE', [secondId]),
    ])
    // No deadlock: always locked in same order regardless of from/to
  })
}

-- Set lock timeout to fail fast (don't wait forever for lock)
SET lock_timeout = '5s';
-- Raises: ERROR:  canceling statement due to lock timeout
```

---

## 5. Partitioning

```sql
-- Partition large tables by time or category
-- Postgres 10+: declarative partitioning

-- Range partitioning by month
CREATE TABLE events (
  id          UUID NOT NULL DEFAULT gen_random_uuid(),
  type        TEXT NOT NULL,
  actor_id    UUID,
  payload     JSONB,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE events_2025_01 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE events_2025_02 PARTITION OF events
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Automate partition creation with pg_partman extension
SELECT partman.create_parent(
  p_parent_table => 'public.events',
  p_control => 'created_at',
  p_type => 'native',
  p_interval => 'monthly',
  p_premake => 3  -- Pre-create 3 future partitions
);

-- Benefits:
-- Partition pruning: query for 2025-01 data only scans events_2025_01
-- Efficient retention: DROP TABLE events_2024_01 (instant vs DELETE)
-- Maintenance: VACUUM per partition, not whole table

-- Foreign keys and unique constraints:
-- Unique constraints must include partition key
-- Foreign keys from partition table work; to partition table requires each partition
```

---

## 6. Full-Text Search

```sql
-- tsvector: preprocessed tokens for search
-- tsquery: search query syntax

-- Add FTS column (computed, always up to date)
ALTER TABLE services ADD COLUMN fts_vector TSVECTOR 
  GENERATED ALWAYS AS (
    to_tsvector('portuguese', 
      COALESCE(title, '') || ' ' || 
      COALESCE(description, '') || ' ' || 
      COALESCE(category, '')
    )
  ) STORED;

CREATE INDEX idx_services_fts ON services USING GIN(fts_vector);

-- Simple search
SELECT title, description,
  ts_rank(fts_vector, query) AS rank
FROM services, to_tsquery('portuguese', 'massagem') query
WHERE fts_vector @@ query
ORDER BY rank DESC
LIMIT 10;

-- Boolean operators: massagem & terapeutica (AND), massagem | pilates (OR), !laser (NOT)
SELECT * FROM services
WHERE fts_vector @@ to_tsquery('portuguese', 'massagem & (relaxamento | terapeutico)');

-- Highlight matches in results
SELECT 
  title,
  ts_headline('portuguese', description, to_tsquery('portuguese', 'massagem'), 
    'StartSel=<b>, StopSel=</b>, MaxWords=50, MinWords=25') AS excerpt
FROM services
WHERE fts_vector @@ to_tsquery('portuguese', 'massagem')
LIMIT 10;

-- Phrase search (exact phrase)
SELECT * FROM services
WHERE fts_vector @@ phraseto_tsquery('portuguese', 'massagem relaxante');

-- Prefix search (autocomplete)
SELECT * FROM services
WHERE fts_vector @@ to_tsquery('portuguese', 'mass:*');  -- Matches "massagem", "massoterapia", etc.
```

---

## 7. JSON and JSONB

```sql
-- JSON: stored as text, slower queries, preserves key order and duplicates
-- JSONB: stored as binary, faster queries (indexable), no duplicates, no key order
-- Use JSONB (almost always)

-- Storing flexible metadata
ALTER TABLE services ADD COLUMN metadata JSONB DEFAULT '{}';

-- Operators
-- -> returns JSON element: metadata->'price' returns "{"min": 50, "max": 200}"
-- ->> returns text: metadata->>'category' returns "wellness"
-- @> contains: metadata @> '{"featured": true}'
-- ? has key: metadata ? 'discount'
-- #> path: metadata #> '{pricing,min}' returns 50
-- #>> path to text: metadata #>> '{pricing,min}' returns "50"

-- Query JSONB
SELECT * FROM services
WHERE metadata @> '{"active": true}'
  AND (metadata->>'price_min')::numeric < 100
  AND metadata ? 'online';

-- Update JSONB (immutable, must rebuild entire document)
UPDATE services
SET metadata = metadata || '{"featured": true, "updated": "2025-01-01"}'
WHERE id = $1;

-- Remove a key
UPDATE services
SET metadata = metadata - 'deprecated_field'
WHERE id = $1;

-- Index for JSONB queries
CREATE INDEX idx_services_metadata ON services USING GIN(metadata);
-- Supports: @>, ?, ?|, ?&

-- Index on specific JSONB path (more efficient than GIN for single field)
CREATE INDEX idx_services_category ON services ((metadata->>'category'));
CREATE INDEX idx_services_price_min ON services (((metadata->>'price_min')::numeric));
```

---

## 8. Row Level Security

```sql
-- Enable RLS — no row accessible unless a policy allows it
ALTER TABLE bookings ENABLE ROW LEVEL SECURITY;

-- Force RLS even for table owner (prevents accidental bypass)
ALTER TABLE bookings FORCE ROW LEVEL SECURITY;

-- Policy: clients see only their own bookings
CREATE POLICY client_sees_own_bookings ON bookings
  AS PERMISSIVE
  FOR SELECT
  TO app_user  -- Role
  USING (client_id = auth.uid());  -- Condition using Supabase auth helper

-- Policy: clients can create bookings for themselves only
CREATE POLICY client_inserts_own_bookings ON bookings
  AS PERMISSIVE
  FOR INSERT
  TO app_user
  WITH CHECK (client_id = auth.uid());

-- Policy: clients can cancel their own pending bookings
CREATE POLICY client_cancels_own_pending ON bookings
  AS PERMISSIVE
  FOR UPDATE
  TO app_user
  USING (client_id = auth.uid() AND status = 'pending')
  WITH CHECK (status = 'cancelled');

-- Policy: admins see everything
CREATE POLICY admin_full_access ON bookings
  AS PERMISSIVE
  FOR ALL
  TO app_admin  -- Separate admin role
  USING (true)
  WITH CHECK (true);

-- Verify RLS is working
SET role app_user;
SET request.jwt.claims = '{"sub": "user_123"}';  -- Simulates Supabase auth
SELECT COUNT(*) FROM bookings;  -- Should only return user_123's bookings
RESET role;
```

---

## 9. Connection Management

```sql
-- PostgreSQL default max_connections = 100 (not enough for 10+ services)
-- Solution: PgBouncer (connection pooler)

-- PgBouncer modes:
-- Session pooling: one server connection per client connection (like no pooling)
-- Transaction pooling: server connection returned to pool after each transaction (recommended)
-- Statement pooling: server connection returned after each statement (most efficient, limitations)

-- pgbouncer.ini
[databases]
mydb = host=postgres-primary port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000   -- App connections
default_pool_size = 25   -- Postgres connections per user+db
max_db_connections = 100 -- Total postgres connections
server_idle_timeout = 600
client_idle_timeout = 0

-- Connection string for apps (via PgBouncer):
-- postgresql://user:password@pgbouncer:5432/mydb

-- Limitations of transaction pooling:
-- No SET statements that persist across transactions
-- No LISTEN/NOTIFY (use direct connection for these)
-- No prepared statements (or use session pooling for these connections)
```

---

## 10. Migrations

```sql
-- Migration principles:
-- 1. Forward only (never edit applied migrations)
-- 2. Idempotent when possible
-- 3. Never lock tables in production (use CONCURRENTLY)
-- 4. Separate schema changes from data migrations

-- Adding column (safe)
ALTER TABLE bookings ADD COLUMN source TEXT;  -- nullable by default: no table rewrite

-- Adding NOT NULL column (unsafe: rewrites entire table!)
-- Safe approach: 3 steps
-- Step 1: Add nullable column
ALTER TABLE bookings ADD COLUMN payment_method TEXT;
-- Step 2: Backfill existing rows (in batches)
UPDATE bookings SET payment_method = 'credit_card' WHERE payment_method IS NULL;
-- Step 3: Add NOT NULL constraint (validates all rows in pg 12+: faster than rewrite)
ALTER TABLE bookings ALTER COLUMN payment_method SET NOT NULL;
-- pg 12+: NOT NULL validation is separate from the alter, faster (shares lock shorter)

-- Adding index online (no lock!)
CREATE INDEX CONCURRENTLY idx_bookings_payment_method ON bookings(payment_method);
-- CONCURRENTLY builds index without blocking reads or writes
-- Takes longer but safe for production

-- Migration tool: Flyway, Liquibase, or Drizzle Kit
-- File naming: V001__create_users.sql, V002__add_bookings.sql
-- Each migration = one transaction (auto-rollback on error)
```

---

## 11. Backup and Recovery

```bash
# pg_dump — logical backup
pg_dump -h localhost -U postgres -Fc mydb > backup_$(date +%Y%m%d_%H%M%S).dump

# Restore
pg_restore -h localhost -U postgres -d mydb_restore backup.dump

# pg_basebackup — physical backup (faster for large DBs)
pg_basebackup -h localhost -U replication -D /backup/pg_base \
  -Ft -z -Xs -P --checkpoint=fast

# Point-in-Time Recovery (PITR):
# Continuously archive WAL (Write-Ahead Log) → restore to any point in time
# Archive command in postgresql.conf:
# archive_command = 'aws s3 cp %p s3://my-pg-wal-archive/%f'

# Recovery target: restore to specific time
# recovery.conf:
# restore_command = 'aws s3 cp s3://my-pg-wal-archive/%f %p'
# recovery_target_time = '2025-06-15 14:30:00 UTC'

# RDS/Cloud: managed snapshots
# AWS RDS: automated daily snapshots + transaction log archiving → PITR with 5-min granularity
```

---

## 12. Performance Monitoring

```sql
-- pg_stat_statements: slow query analysis
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Top 10 slowest queries (by average execution time)
SELECT 
  left(query, 100) AS query,
  calls,
  round(mean_exec_time::numeric, 2) AS avg_ms,
  round(max_exec_time::numeric, 2) AS max_ms,
  round(total_exec_time::numeric, 2) AS total_ms,
  rows,
  100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0) AS cache_hit_pct
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Cache hit ratio (should be > 99% in production)
SELECT 
  SUM(blks_hit) / NULLIF(SUM(blks_hit) + SUM(blks_read), 0) * 100 AS cache_hit_ratio
FROM pg_stat_database;

-- Table bloat (dead tuples waiting for VACUUM)
SELECT 
  tablename,
  n_live_tup AS live_rows,
  n_dead_tup AS dead_rows,
  round(n_dead_tup::numeric / NULLIF(n_live_tup + n_dead_tup, 0) * 100, 1) AS bloat_pct,
  last_autovacuum,
  last_autoanalyze
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Lock contention
SELECT 
  pg_stat_activity.pid,
  pg_stat_activity.query,
  pg_blocking_pids(pg_stat_activity.pid) AS blocked_by,
  age(NOW(), pg_stat_activity.query_start) AS duration
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pg_stat_activity.pid)) > 0;

-- Connection count by state
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state 
ORDER BY count DESC;
-- idle: connection open but no query (wasted; PgBouncer helps)
-- active: running query
-- idle in transaction: transaction open but waiting (dangerous: holds locks)
```
