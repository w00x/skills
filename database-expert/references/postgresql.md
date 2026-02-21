# PostgreSQL Reference

## Table of Contents
- [Schema Design](#schema-design)
- [Indexing Strategies](#indexing-strategies)
- [Partitioning](#partitioning)
- [JSONB Operations](#jsonb-operations)
- [Performance Tuning](#performance-tuning)
- [Common Anti-Patterns](#common-anti-patterns)

---

## Schema Design

### Normalization (3NF Default)
```sql
-- Users table (3NF)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Orders normalized with FK
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    total_cents BIGINT NOT NULL CHECK (total_cents >= 0),
    status TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending','paid','shipped','cancelled')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Strategic Denormalization
```sql
-- When reads >> writes and JOIN cost is measured via EXPLAIN
ALTER TABLE orders ADD COLUMN user_email TEXT;
-- Maintain via trigger or application layer
-- Trade-off: write amplification vs read latency reduction
```

### Rules
- Use `UUID` or `BIGSERIAL` for PKs — UUID for distributed, serial for single-node
- Always `NOT NULL` unless truly optional
- Use `CHECK` constraints for domain validation
- Use `TIMESTAMPTZ` (never `TIMESTAMP`) for timezone safety
- Use `BIGINT` for monetary values (cents) — never floats

---

## Indexing Strategies

### B-Tree (default, most queries)
```sql
-- Composite index covering WHERE + ORDER BY
CREATE INDEX idx_orders_user_status ON orders (user_id, status, created_at DESC);

-- Partial index (smaller, faster)
CREATE INDEX idx_orders_pending ON orders (user_id, created_at DESC)
    WHERE status = 'pending';
```

### GIN (full-text, JSONB, arrays)
```sql
-- JSONB containment queries
CREATE INDEX idx_profile_data ON users USING GIN (profile_data jsonb_path_ops);

-- Full-text search
CREATE INDEX idx_search ON articles USING GIN (to_tsvector('english', title || ' ' || body));
```

### GiST (range, geometric, nearest-neighbor)
```sql
CREATE INDEX idx_events_range ON events USING GIST (tstzrange(start_at, end_at));
```

### BRIN (large tables, naturally ordered data)
```sql
-- For append-only time-series with physical correlation
CREATE INDEX idx_logs_time ON logs USING BRIN (created_at) WITH (pages_per_range = 32);
```

### Index Analysis
```sql
-- Find unused indexes
SELECT schemaname, relname, indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (sequential scans on large tables)
SELECT relname, seq_scan, seq_tup_read, idx_scan,
       pg_size_pretty(pg_relation_size(relid)) AS size
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND pg_relation_size(relid) > 10485760
ORDER BY seq_tup_read DESC;
```

---

## Partitioning

### Range Partitioning (time-series)
```sql
CREATE TABLE events (
    id UUID NOT NULL DEFAULT gen_random_uuid(),
    event_type TEXT NOT NULL,
    payload JSONB,
    created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_q1 PARTITION OF events
    FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
CREATE TABLE events_2025_q2 PARTITION OF events
    FOR VALUES FROM ('2025-04-01') TO ('2025-07-01');

-- Automate with pg_partman for production
```

### Rules
- Partition when tables exceed ~100M rows or need data lifecycle management
- Partition key must be in the PK or have a unique constraint
- Use `pg_partman` for automated partition creation/retention
- Always include partition key in WHERE clauses for pruning

---

## JSONB Operations

```sql
-- Query nested JSONB
SELECT id, profile_data->>'name' AS name
FROM users
WHERE profile_data @> '{"role": "admin"}';

-- Partial JSONB update
UPDATE users
SET profile_data = jsonb_set(profile_data, '{preferences,theme}', '"dark"')
WHERE id = $1;

-- JSONB aggregation
SELECT event_type, COUNT(*), jsonb_agg(payload->'details')
FROM events
GROUP BY event_type;
```

### Rules
- Use `jsonb_path_ops` GIN for containment (`@>`) queries
- Use `->` for JSONB result, `->>` for text result
- Don't shove relational data into JSONB — use it for truly schemaless data
- Index specific JSONB paths with expression indexes when query patterns are known

---

## Performance Tuning

### EXPLAIN Reading
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) 
SELECT * FROM orders WHERE user_id = $1 AND status = 'pending' 
ORDER BY created_at DESC LIMIT 20;
```

Key metrics to examine:
- **Seq Scan** vs **Index Scan** — seq scan on large tables = missing index
- **Rows estimated vs actual** — large mismatch = stale statistics (`ANALYZE`)
- **Buffers shared hit vs read** — low hit ratio = insufficient `shared_buffers`
- **Sort Method: external merge** = `work_mem` too low

### Key Configuration
```sql
-- For OLTP workloads (adjust to hardware)
shared_buffers = '4GB'           -- 25% of RAM
effective_cache_size = '12GB'    -- 75% of RAM
work_mem = '64MB'                -- per-operation sort/hash
maintenance_work_mem = '1GB'     -- for VACUUM, CREATE INDEX
random_page_cost = 1.1           -- for SSDs (default 4.0 is for HDDs)
```

### Cursor-Based Pagination
```sql
-- Instead of OFFSET (scans all skipped rows)
SELECT id, created_at, title
FROM articles
WHERE created_at < $1  -- cursor: last seen created_at
ORDER BY created_at DESC
LIMIT 20;
```

---

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| `SELECT *` | Reads unneeded columns, breaks covering indexes | List needed columns |
| `OFFSET` pagination | O(n) — scans all skipped rows | Keyset pagination |
| Missing `LIMIT` | Unbounded result sets | Always `LIMIT` or stream |
| `NOT IN (subquery)` | Poor performance with NULLs | Use `NOT EXISTS` |
| Implicit casts in WHERE | Prevents index usage | Cast explicitly or fix types |
| No `VACUUM`/`ANALYZE` | Stale stats, table bloat | Tune `autovacuum` |
