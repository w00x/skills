# Database Architecture Reference

## Table of Contents
- [OLTP vs OLAP](#oltp-vs-olap)
- [Replication Patterns](#replication-patterns)
- [Sharding Strategies](#sharding-strategies)
- [CAP Theorem & Consistency](#cap-theorem--consistency)
- [Zero-Downtime Migrations](#zero-downtime-migrations)
- [Database Selection Guide](#database-selection-guide)

---

## OLTP vs OLAP

| Dimension | OLTP | OLAP |
|---|---|---|
| Workload | Short transactions, high concurrency | Complex analytics, batch reads |
| Schema | Normalized (3NF) | Star/snowflake (denormalized) |
| Indexes | Many B-Tree for point lookups | Few, columnar storage |
| Queries | Simple WHERE, JOINs on PKs | GROUP BY, window functions, aggregations |
| Examples | PostgreSQL, MySQL, CockroachDB | ClickHouse, BigQuery, Redshift, DuckDB |
| Latency | ms | seconds to minutes |

### Rules
- Never run analytics on OLTP databases — replicate to a read replica or data warehouse
- Use CDC (Change Data Capture) to stream OLTP changes to OLAP (Debezium, pg_logical)
- Consider DuckDB for embedded analytics over PostgreSQL/Parquet

---

## Replication Patterns

### Single-Leader (Primary-Replica)
```
Client writes → Primary → replicates to → Replica(s)
Client reads  → Replica(s) (eventual consistency)
```
- **Use when:** Read-heavy workloads, need read scaling
- **Risk:** Replication lag → stale reads
- **Mitigation:** Read-your-writes consistency (read from primary after write)

### Multi-Leader
```
Client → Leader A ←→ Leader B ← Client
         (bidirectional replication)
```
- **Use when:** Multi-region writes, offline-capable apps
- **Risk:** Write conflicts
- **Mitigation:** Last-write-wins, CRDTs, application-level resolution

### Leaderless (Dynamo-style)
```
Client → write to N nodes, read from N nodes
         Quorum: W + R > N for consistency
```
- **Use when:** High availability, partition tolerance (Cassandra, DynamoDB)
- **Risk:** Stale reads if quorum not met
- **Mitigation:** Read repair, anti-entropy (Merkle trees)

---

## Sharding Strategies

### Hash-Based
```
shard = hash(shard_key) % num_shards
```
- Even distribution, no range queries on shard key
- Resharding requires data migration (use consistent hashing)

### Range-Based
```
shard_1: A-M
shard_2: N-Z
```
- Supports range queries, risk of hot spots
- Monitor shard sizes and rebalance

### Directory-Based
```
Lookup service maps key → shard
```
- Most flexible, single point of failure in directory
- Use with a distributed config store (etcd, ZooKeeper)

### Shard Key Selection Criteria
1. **High cardinality** — many distinct values
2. **Even distribution** — no hot shards
3. **Query isolation** — most queries target single shard
4. **Immutability** — key doesn't change (or rarely)

---

## CAP Theorem & Consistency

### Trade-offs
| System | C | A | P | Notes |
|---|---|---|---|---|
| PostgreSQL (single) | ✅ | ✅ | ❌ | CP when replicated |
| MongoDB (replica set) | ✅ | ⚠️ | ✅ | Tunable via write concern |
| Cassandra | ⚠️ | ✅ | ✅ | Tunable via consistency level |
| Redis Cluster | ⚠️ | ✅ | ✅ | Async replication, possible data loss |

### Consistency Levels
- **Strong** (linearizability): Every read returns latest write. Use for financial transactions.
- **Eventual**: Reads may return stale data temporarily. Use for social feeds, analytics.
- **Causal**: Respects happens-before ordering. Use for chat, collaborative editing.

### Rules
- Default to strong consistency unless latency/availability requirements force weaker models
- Document the chosen consistency level and its implications for the team
- Test failure scenarios (network partitions, node failures) before production

---

## Zero-Downtime Migrations

### Expand-Contract Pattern

1. **Expand** — Add new column/table (backward compatible)
```sql
ALTER TABLE users ADD COLUMN phone TEXT;  -- nullable, no default needed
```

2. **Migrate** — Dual-write: application writes to both old and new structures
```sql
-- Backfill existing data
UPDATE users SET phone = profiles.phone FROM profiles WHERE users.id = profiles.user_id;
```

3. **Switch** — Application reads from new structure

4. **Contract** — Remove old column/table
```sql
ALTER TABLE users DROP COLUMN IF EXISTS old_phone;
```

### Large Table Migrations
```sql
-- PostgreSQL: use pg_repack or online DDL tools
-- Never ALTER TABLE on large tables during peak hours

-- For adding NOT NULL with default (PG 11+, instant):
ALTER TABLE orders ADD COLUMN priority INT NOT NULL DEFAULT 0;

-- For adding an index concurrently:
CREATE INDEX CONCURRENTLY idx_orders_priority ON orders (priority);
```

### Rules
- Never combine schema change + data migration in one deployment
- Always have a rollback plan (the contract step should be a separate deployment)
- Use `CREATE INDEX CONCURRENTLY` (PG) or `ALGORITHM=INPLACE` (MySQL) to avoid locks
- Test migrations on a production-size dataset before deploying
- Monitor replication lag during backfills — throttle if needed

---

## Database Selection Guide

| Use Case | Recommended | Reason |
|---|---|---|
| General OLTP, complex queries | PostgreSQL | ACID, rich SQL, JSONB, extensions |
| Simple OLTP, web apps | MySQL | Wide ecosystem, easy replication |
| Schemaless, rapid iteration | MongoDB | Flexible schema, horizontal scaling |
| Caching, sessions, queues | Redis | In-memory speed, rich data structures |
| Time-series, IoT | TimescaleDB (PG) | Auto-partitioning, continuous aggregates |
| Wide-column, high write throughput | Cassandra | Linear scalability, multi-DC |
| Full-text search | Elasticsearch | Inverted indexes, relevance scoring |
| Analytics, data warehouse | ClickHouse / BigQuery | Columnar, compression, fast aggregation |
| Graph relationships | Neo4j | Cypher queries, relationship traversal |
| Embedded analytics | DuckDB | In-process OLAP, Parquet native |

### Decision Framework
1. What is the primary access pattern? (Point lookup, range scan, full-text, graph traversal)
2. What are the consistency requirements? (Strong, eventual, causal)
3. What is the write/read ratio?
4. What is the expected data volume? (GB, TB, PB)
5. Is horizontal scaling needed?
6. What is the operational complexity budget?
