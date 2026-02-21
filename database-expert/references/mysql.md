# MySQL Reference

## Table of Contents
- [Schema Design](#schema-design)
- [InnoDB Internals](#innodb-internals)
- [Indexing Strategies](#indexing-strategies)
- [Replication](#replication)
- [Performance Tuning](#performance-tuning)
- [Common Anti-Patterns](#common-anti-patterns)

---

## Schema Design

### Table Creation Best Practices
```sql
CREATE TABLE users (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL,
    display_name VARCHAR(100) NOT NULL,
    password_hash CHAR(60) NOT NULL,  -- bcrypt fixed length
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    UNIQUE KEY uq_users_email (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### Rules
- Always use `InnoDB` engine (ACID, row-level locking, crash recovery)
- Use `utf8mb4` charset (full Unicode support including emoji)
- `BIGINT UNSIGNED AUTO_INCREMENT` for PKs (sequential = optimal for InnoDB clustered index)
- Use `TIMESTAMP` for automatic timezone handling, `DATETIME` for fixed times
- Define `ON UPDATE CURRENT_TIMESTAMP` for `updated_at` columns
- Avoid `ENUM` — use lookup tables or application-level validation

---

## InnoDB Internals

### Clustered Index
- InnoDB stores data in PK order (clustered index)
- Sequential PKs (AUTO_INCREMENT) → sequential writes → minimal page splits
- UUID PKs → random inserts → page splits → fragmentation
- If using UUIDs, consider `uuid_to_bin(uuid, 1)` for time-ordered storage

### Buffer Pool
```sql
-- Check buffer pool hit ratio (should be > 99%)
SELECT 
    (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100 AS hit_ratio
FROM (
    SELECT 
        VARIABLE_VALUE AS Innodb_buffer_pool_reads 
    FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
) a, (
    SELECT 
        VARIABLE_VALUE AS Innodb_buffer_pool_read_requests 
    FROM performance_schema.global_status WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
) b;
```

---

## Indexing Strategies

### Composite Index Order
```sql
-- Follow the "equality-range-sort" rule
-- WHERE user_id = ? AND status IN (...) ORDER BY created_at DESC
CREATE INDEX idx_orders_user_status_created 
    ON orders (user_id, status, created_at DESC);
```

### Covering Index
```sql
-- Covers SELECT without hitting the clustered index
CREATE INDEX idx_orders_cover 
    ON orders (user_id, status, created_at, total_cents);
-- Query reads only from index (Using index in EXPLAIN)
```

### Prefix Index (long strings)
```sql
-- Index only first N characters for long text columns
CREATE INDEX idx_url_prefix ON links (url(50));
-- Check selectivity: SELECT COUNT(DISTINCT LEFT(url, 50)) / COUNT(*) FROM links;
```

### Full-Text Index
```sql
ALTER TABLE articles ADD FULLTEXT INDEX ft_articles (title, body);
SELECT * FROM articles WHERE MATCH(title, body) AGAINST('database optimization' IN BOOLEAN MODE);
```

### Index Analysis
```sql
-- Find unused indexes
SELECT object_schema, object_name, index_name, count_star
FROM performance_schema.table_io_waits_summary_by_index_usage
WHERE index_name IS NOT NULL AND count_star = 0 AND object_schema = 'mydb'
ORDER BY object_name;
```

---

## Replication

### Source-Replica Architecture
- **Source** → handles writes
- **Replica(s)** → handle reads
- Use `GTID` replication for easier failover

### Read/Write Split Pattern
```
Application → Write queries → Source (primary)
Application → Read queries  → Replica (read pool)
```

### Semi-Synchronous Replication
```sql
-- On source: ensures at least 1 replica acknowledged
SET GLOBAL rpl_semi_sync_source_enabled = 1;
SET GLOBAL rpl_semi_sync_source_wait_for_replica_count = 1;
```

### Rules
- Use GTID-based replication for automated failover
- Monitor `Seconds_Behind_Source` for replication lag
- Never write to replicas
- Use connection pooling (ProxySQL, MySQL Router) for read distribution

---

## Performance Tuning

### EXPLAIN Analysis
```sql
EXPLAIN ANALYZE
SELECT o.id, o.total_cents, u.email
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending' AND o.created_at > '2025-01-01'
ORDER BY o.created_at DESC
LIMIT 20;
```

Key things to spot:
- `type: ALL` → full table scan (needs index)
- `rows` much larger than `filtered` → poor selectivity
- `Using filesort` → consider index for ORDER BY
- `Using temporary` → optimize GROUP BY / DISTINCT

### Key Configuration
```ini
# InnoDB (adjust to hardware)
innodb_buffer_pool_size = 4G      # 60-70% of RAM for dedicated DB server
innodb_log_file_size = 1G         # larger = better write performance
innodb_flush_log_at_trx_commit = 1  # ACID compliance (2 for performance)
innodb_flush_method = O_DIRECT    # avoid double buffering

# Query cache (disabled in MySQL 8.0 — use ProxySQL or app-level cache)
# Connections
max_connections = 200
thread_cache_size = 16
```

---

## Common Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| UUID as PK | Random inserts, page splits, fragmentation | Use ordered UUID or BIGINT AUTO_INCREMENT |
| `SELECT *` | Wastes buffer pool, breaks covering indexes | List needed columns |
| `ORDER BY RAND()` | Full table scan + filesort | Pre-select random IDs |
| `WHERE YEAR(created_at) = 2025` | Function on column kills index | Use range: `created_at >= '2025-01-01'` |
| No connection pooling | Connection overhead per request | Use ProxySQL/PgBouncer |
| Large transactions | Long lock hold times | Break into smaller batches |
