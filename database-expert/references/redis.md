# Redis Reference

## Table of Contents
- [Data Structures](#data-structures)
- [Caching Patterns](#caching-patterns)
- [Pub/Sub & Streams](#pubsub--streams)
- [Persistence](#persistence)
- [Performance & Operations](#performance--operations)

---

## Data Structures

### Strings (key-value, counters, sessions)
```redis
SET user:123:session "token_abc" EX 3600       -- TTL 1 hour
GET user:123:session
INCR rate:api:user:123                          -- atomic counter
SETEX cache:user:123 300 '{"name":"Alice"}'     -- cache with 5min TTL
```

### Hashes (objects, partial updates)
```redis
HSET user:123 name "Alice" email "alice@example.com" plan "pro"
HGET user:123 plan
HINCRBY user:123 login_count 1
HGETALL user:123
```

### Sorted Sets (rankings, rate limiting, time-series)
```redis
ZADD leaderboard 1500 "user:42" 2300 "user:7" 1800 "user:15"
ZREVRANGE leaderboard 0 9 WITHSCORES             -- top 10
ZRANGEBYSCORE leaderboard 1000 2000               -- score range
ZRANK leaderboard "user:42"                       -- rank lookup
```

### Lists (queues, recent items)
```redis
LPUSH queue:emails '{"to":"alice","subject":"Welcome"}'
BRPOP queue:emails 30                              -- blocking pop, 30s timeout
LTRIM recent:user:123 0 99                         -- keep last 100 items
```

### Sets (tags, unique tracking, intersections)
```redis
SADD online:users "user:1" "user:2" "user:3"
SISMEMBER online:users "user:1"                    -- O(1) membership check
SINTER online:users premium:users                  -- intersection
SCARD online:users                                 -- count
```

### HyperLogLog (cardinality estimation)
```redis
PFADD daily:visitors:2025-01-15 "user:1" "user:2" "user:1"
PFCOUNT daily:visitors:2025-01-15                  -- ~2 (probabilistic)
```

---

## Caching Patterns

### Cache-Aside (Lazy Loading)
```
Read:  App → Redis? HIT → return
                  MISS → DB → write to Redis → return
Write: App → DB → delete Redis key
```
- Most common pattern. Simple, but cold-start penalty
- Always set TTL even on write — safety net

### Write-Through
```
Write: App → Redis → DB (synchronous)
Read:  App → Redis (always warm)
```
- Higher write latency, but reads always hit cache
- Use for data that is read heavily after write

### Cache Stampede Prevention
```redis
-- Use SET NX + short TTL as a lock
SET cache:lock:user:123 1 NX EX 5
-- Only winner rebuilds cache; losers wait and retry
```

### TTL Strategy
| Data Type | TTL | Reason |
|---|---|---|
| Session tokens | 1-24h | Security |
| User profiles | 5-15min | Stale data tolerance |
| Rate limit counters | Window duration | Auto-expire |
| Computed aggregations | 1-5min | Freshness vs cost |

---

## Pub/Sub & Streams

### Pub/Sub (fire-and-forget)
```redis
SUBSCRIBE events:orders              -- subscriber
PUBLISH events:orders '{"id":"123","status":"paid"}'  -- publisher
```
- No persistence — messages lost if no subscriber online
- Use for real-time notifications, invalidation signals

### Streams (persistent, consumer groups)
```redis
-- Produce
XADD events:orders * order_id 123 status paid amount 5000

-- Create consumer group
XGROUP CREATE events:orders processing $ MKSTREAM

-- Consume (consumer group)
XREADGROUP GROUP processing worker-1 COUNT 10 BLOCK 5000 STREAMS events:orders >

-- Acknowledge processed
XACK events:orders processing 1234567890-0
```
- Persistent, consumer groups, exactly-once semantics
- Use for task queues, event sourcing, audit logs

---

## Persistence

### RDB (Snapshots)
```redis
save 900 1       -- snapshot every 15min if ≥1 write
save 300 10      -- snapshot every 5min if ≥10 writes
```
- Point-in-time snapshots, compact file
- Data loss window = time since last snapshot

### AOF (Append-Only File)
```redis
appendonly yes
appendfsync everysec    -- fsync every second (recommended balance)
```
- Every write logged, minimal data loss (≤1 second)
- Larger files, slower restart — use `BGREWRITEAOF`

### Recommendation
- Use AOF + RDB together for production
- AOF for durability, RDB for faster restarts and backups

---

## Performance & Operations

### Memory Optimization
```redis
maxmemory 4gb
maxmemory-policy allkeys-lru    -- evict least recently used
```

| Eviction Policy | Use Case |
|---|---|
| `noeviction` | Persistent data store (error on full) |
| `allkeys-lru` | General cache |
| `volatile-lru` | Cache with mix of persistent + expiring keys |
| `allkeys-random` | When access pattern is uniform |

### Key Naming Convention
```
{entity}:{id}:{attribute}
user:123:profile
cache:api:v2:users:list
rate:api:user:123:minute
```

### Anti-Patterns
| Anti-Pattern | Problem | Fix |
|---|---|---|
| `KEYS *` in production | Blocks server (O(n)) | Use `SCAN` cursor |
| Large values (>100KB) | Network latency, memory fragmentation | Compress or split |
| No TTL on cache keys | Memory grows unbounded | Always set TTL |
| Single key hot spot | Uneven load | Shard key across slots |
| `FLUSHALL` without thought | Data loss | Use `FLUSHDB` on specific DB |
