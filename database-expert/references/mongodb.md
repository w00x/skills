# MongoDB Reference

## Table of Contents
- [Document Modeling](#document-modeling)
- [Indexing](#indexing)
- [Aggregation Pipeline](#aggregation-pipeline)
- [Sharding](#sharding)
- [Performance Tuning](#performance-tuning)

---

## Document Modeling

### Embedding vs Referencing

**Embed when:**
- Data is read together (1:few relationship)
- Child data doesn't change independently
- Document size stays under 16MB

**Reference when:**
- Many-to-many relationships
- Child data is large or changes frequently
- Child data is queried independently

### Embedded Pattern
```javascript
// Order with embedded items (read together, items belong to order)
{
  _id: ObjectId("..."),
  user_id: ObjectId("..."),
  status: "pending",
  items: [
    { product_id: ObjectId("..."), name: "Widget", qty: 2, price_cents: 1500 },
    { product_id: ObjectId("..."), name: "Gadget", qty: 1, price_cents: 3000 }
  ],
  total_cents: 6000,
  created_at: ISODate("2025-01-15T10:00:00Z")
}
```

### Referenced Pattern
```javascript
// User profile (referenced from orders — queried independently)
{ _id: ObjectId("..."), email: "user@example.com", name: "Alice" }

// Order references user
{ _id: ObjectId("..."), user_id: ObjectId("..."), total_cents: 6000 }
```

### Patterns
- **Bucket pattern** — group time-series data into buckets (1 doc per hour/day)
- **Computed pattern** — pre-compute aggregations to avoid runtime computation
- **Outlier pattern** — handle documents that exceed typical size with overflow docs
- **Subset pattern** — embed most recent N items, reference the rest

---

## Indexing

### Single Field & Compound
```javascript
// Single field
db.orders.createIndex({ user_id: 1 })

// Compound (ESR rule: Equality → Sort → Range)
db.orders.createIndex({ user_id: 1, created_at: -1, status: 1 })

// Unique
db.users.createIndex({ email: 1 }, { unique: true })
```

### Partial Index
```javascript
// Index only active orders (smaller, faster)
db.orders.createIndex(
  { user_id: 1, created_at: -1 },
  { partialFilterExpression: { status: { $in: ["pending", "processing"] } } }
)
```

### Text Index
```javascript
db.articles.createIndex({ title: "text", body: "text" },
  { weights: { title: 10, body: 1 } })

db.articles.find({ $text: { $search: "database optimization" } },
  { score: { $meta: "textScore" } })
  .sort({ score: { $meta: "textScore" } })
```

### Index Analysis
```javascript
// Check index usage
db.orders.aggregate([{ $indexStats: {} }])

// Explain query plan
db.orders.find({ user_id: ObjectId("...") }).explain("executionStats")
// Look for: IXSCAN (good) vs COLLSCAN (bad)
// Check: totalKeysExamined vs totalDocsReturned ratio
```

---

## Aggregation Pipeline

### Common Stages
```javascript
db.orders.aggregate([
  // Filter early (uses indexes)
  { $match: { status: "paid", created_at: { $gte: ISODate("2025-01-01") } } },
  
  // Group and compute
  { $group: {
      _id: { $dateToString: { format: "%Y-%m", date: "$created_at" } },
      total_revenue: { $sum: "$total_cents" },
      order_count: { $sum: 1 },
      avg_order: { $avg: "$total_cents" }
  }},
  
  // Sort results
  { $sort: { _id: -1 } },
  
  // Reshape output
  { $project: {
      month: "$_id",
      total_revenue: 1,
      order_count: 1,
      avg_order: { $round: ["$avg_order", 0] }
  }}
])
```

### Rules
- `$match` and `$sort` first — they can use indexes
- `$project` early to reduce document size in pipeline
- Use `$lookup` sparingly — it's a left outer join, can be expensive
- `allowDiskUse: true` for large aggregations exceeding 100MB memory limit

---

## Sharding

### Shard Key Selection

| Criteria | Good Shard Key | Bad Shard Key |
|---|---|---|
| Cardinality | High (user_id, order_id) | Low (status, country) |
| Distribution | Even across shards | Monotonic (timestamp, auto-increment) |
| Query isolation | Targets single shard | Scatter-gather across all |

### Hashed Sharding
```javascript
// Even distribution, but no range queries on shard key
sh.shardCollection("mydb.events", { _id: "hashed" })
```

### Ranged Sharding
```javascript
// Good for range queries, risk of hot spots
sh.shardCollection("mydb.logs", { tenant_id: 1, created_at: 1 })
```

### Rules
- Shard key is **immutable** after creation — choose carefully
- Compound shard keys (tenant_id + timestamp) balance isolation and distribution
- Monitor chunk distribution: `sh.status()`
- Target 64MB-256MB chunks

---

## Performance Tuning

### Read Preferences
- `primary` — strong consistency (default)
- `primaryPreferred` — failover reads to secondary
- `secondary` — read from replicas (eventual consistency, reduces primary load)
- `nearest` — lowest latency (for geo-distributed)

### Write Concerns
```javascript
// Acknowledge after majority replication (recommended for durability)
db.orders.insertOne(doc, { writeConcern: { w: "majority", j: true } })
```

### Connection Pooling
```javascript
// Node.js driver example
const client = new MongoClient(uri, {
  maxPoolSize: 50,
  minPoolSize: 5,
  maxIdleTimeMS: 30000,
  serverSelectionTimeoutMS: 5000
})
```

### Anti-Patterns
| Anti-Pattern | Problem | Fix |
|---|---|---|
| Unbounded arrays | Document grows past 16MB | Bucket pattern or reference |
| No indexes on $lookup | Full collection scan | Index the foreign field |
| Large $in arrays | Slow query, memory pressure | Break into batches |
| `$where` with JavaScript | No index usage, security risk | Use query operators |
