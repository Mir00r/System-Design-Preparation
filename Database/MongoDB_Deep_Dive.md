# 🍃 MongoDB Deep Dive: The Document Database for Flexible Schema Applications

> *"When your data model changes every sprint and your schema migrations take longer than your features, you need a database that embraces change. MongoDB stores documents as they are — no ALTER TABLE, no downtime migrations, no ORM impedance mismatch."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [SQL vs NoSQL](./SQL_Vs_NoSQL.md), [Indexing](./Indexing.md)

---

## 📋 Table of Contents
1. [What is MongoDB](#-what-is-mongodb)
2. [Document Model](#-document-model)
3. [Indexing Strategies](#-indexing-strategies)
4. [Replication (Replica Sets)](#-replication-replica-sets)
5. [Sharding](#-sharding)
6. [Aggregation Pipeline](#-aggregation-pipeline)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [When to Use / Not Use](#-when-to-use--not-use)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is MongoDB

```
┌─────────────────────────────────────────────────────────┐
│                  MONGODB AT A GLANCE                     │
│                                                         │
│  Type:        Document database (JSON/BSON)             │
│  Schema:      Schema-less (flexible, per-document)      │
│  Query:       Rich query language (filters, projections)│
│  Scaling:     Horizontal (auto-sharding)                │
│  Consistency: Tunable (strong to eventual)              │
│  Replication: Replica sets (automatic failover)         │
│  Storage:     WiredTiger engine (compression, MVCC)     │
│  Transactions:Multi-document ACID (since v4.0)          │
└─────────────────────────────────────────────────────────┘

SQL vs MongoDB terminology:
  Database    → Database
  Table       → Collection
  Row         → Document
  Column      → Field
  JOIN        → $lookup (or embed)
  PRIMARY KEY → _id (auto-generated ObjectId)
```

---

## 📄 Document Model

```
RELATIONAL (normalized, multiple tables, JOINs):
  users:     { id: 1, name: "John" }
  addresses: { id: 1, user_id: 1, city: "NYC", zip: "10001" }
  orders:    { id: 1, user_id: 1, total: 99.99, date: "2024-01-15" }
  
  → 3 queries or JOIN to get user with addresses and orders

MONGODB (denormalized, single document):
  {
    "_id": ObjectId("507f1f77bcf86cd799439011"),
    "name": "John",
    "email": "john@example.com",
    "addresses": [
      { "type": "home", "city": "NYC", "zip": "10001" },
      { "type": "work", "city": "SF", "zip": "94105" }
    ],
    "orders": [
      { "id": "ord-1", "total": 99.99, "date": ISODate("2024-01-15"), "items": [...] }
    ],
    "metadata": {
      "created_at": ISODate("2024-01-01"),
      "last_login": ISODate("2024-01-15")
    }
  }
  
  → 1 query gets everything (no JOINs!)

WHEN TO EMBED vs REFERENCE:

  EMBED (sub-document) when:
    ✅ Data is always accessed together (user + addresses)
    ✅ One-to-few relationship (user has 2-3 addresses)
    ✅ Data doesn't change independently
    ✅ Sub-document is small (< 16MB document limit)

  REFERENCE (separate collection) when:
    ✅ One-to-many with many growing unbounded (user has 10K orders)
    ✅ Data accessed independently (order viewed without user context)
    ✅ Many-to-many relationship (students ↔ courses)
    ✅ Sub-document is large or frequently updated
```

---

## 📇 Indexing Strategies

```
INDEX TYPES:

  Single field:     db.users.createIndex({ email: 1 })
  Compound:         db.orders.createIndex({ user_id: 1, date: -1 })
  Multi-key:        db.posts.createIndex({ tags: 1 })  (array field)
  Text:             db.articles.createIndex({ title: "text", body: "text" })
  Geospatial:       db.places.createIndex({ location: "2dsphere" })
  TTL:              db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })
  Unique:           db.users.createIndex({ email: 1 }, { unique: true })
  Partial:          db.orders.createIndex({ status: 1 }, { partialFilterExpression: { status: "active" } })

COMPOUND INDEX ORDER MATTERS (ESR Rule):
  E = Equality fields first (exact match)
  S = Sort fields next
  R = Range fields last

  Query: db.orders.find({ user_id: "u123", status: "active" }).sort({ date: -1 })
  Best index: { user_id: 1, status: 1, date: -1 }
              (Equality)  (Equality)  (Sort)

EXPLAIN PLAN:
  db.orders.find({...}).explain("executionStats")
  Look for: 
    "stage": "IXSCAN" (good — using index)
    "stage": "COLLSCAN" (bad — full collection scan)
    "totalDocsExamined" vs "nReturned" ratio (should be close to 1:1)
```

---

## 🔄 Replication (Replica Sets)

```
REPLICA SET ARCHITECTURE:
  ┌─────────────────────────────────────────────────┐
  │                                                 │
  │  [PRIMARY]  ──replicate──▶  [SECONDARY 1]      │
  │      │                      [SECONDARY 2]      │
  │      │                      [ARBITER] (votes,  │
  │      │                       no data)           │
  │      │                                         │
  │  All writes → Primary                          │
  │  Reads → Primary (default) or Secondaries      │
  │                                                 │
  │  Failover:                                      │
  │    Primary fails → election → Secondary promoted│
  │    Typical failover time: 10-12 seconds         │
  └─────────────────────────────────────────────────┘

READ PREFERENCES:
  primary:            always read from primary (consistent)
  primaryPreferred:   primary, fall back to secondary
  secondary:          always read from secondary (may be stale)
  secondaryPreferred: secondary, fall back to primary
  nearest:            lowest-latency node (geo-distributed)

WRITE CONCERN (durability guarantee):
  w: 1               acknowledged by primary only (fast, risk of data loss)
  w: "majority"      acknowledged by majority of nodes (safe, slower)
  w: 3               acknowledged by 3 nodes specifically
  j: true            written to journal (survives crash)
```

---

## 🔀 Sharding

```
SHARDING ARCHITECTURE:
  [Client/Driver]
        │
        ▼
  [mongos Router] ──▶ [Config Servers (replica set)]
        │                   (metadata: which shard has which data)
        │
  ┌─────┼─────┐
  ▼     ▼     ▼
[Shard 1] [Shard 2] [Shard 3]
(replica   (replica   (replica
 set)       set)       set)

SHARD KEY SELECTION (most critical decision):

  ❌ BAD shard key: { created_at: 1 }
     All recent writes go to ONE shard (hot spot!)
     
  ❌ BAD shard key: { country: 1 }
     Low cardinality — only ~200 values, uneven distribution
     
  ✅ GOOD shard key: { user_id: "hashed" }
     High cardinality, even distribution, supports user-scoped queries
     
  ✅ GOOD shard key: { tenant_id: 1, _id: 1 }
     Compound: locality for tenant queries + uniqueness for distribution

SHARDING STRATEGIES:
  Hashed:  even distribution, but range queries scatter to all shards
  Ranged:  good for range queries, but risk of hot spots
  Zoned:   route specific data to specific shards (GDPR: EU data on EU shards)
```

---

## 🔧 Aggregation Pipeline

```
// SQL: SELECT status, COUNT(*), AVG(total) FROM orders 
//      WHERE date > '2024-01-01' GROUP BY status HAVING COUNT(*) > 10

db.orders.aggregate([
  { $match: { date: { $gte: ISODate("2024-01-01") } } },    // WHERE
  { $group: {                                                 // GROUP BY
      _id: "$status",
      count: { $sum: 1 },
      avgTotal: { $avg: "$total" }
  }},
  { $match: { count: { $gt: 10 } } },                        // HAVING
  { $sort: { count: -1 } },                                   // ORDER BY
  { $project: { status: "$_id", count: 1, avgTotal: 1 } }    // SELECT
])

COMMON STAGES:
  $match    → filter documents (use indexes, put EARLY in pipeline)
  $group    → aggregate (sum, avg, count, min, max)
  $project  → reshape (include/exclude/compute fields)
  $sort     → order results
  $limit    → top N
  $unwind   → flatten arrays
  $lookup   → LEFT JOIN with another collection
  $facet    → multiple aggregations in parallel
```

---

## 💻 Spring Boot Integration

```java
// application.yml
spring:
  data:
    mongodb:
      uri: mongodb+srv://user:pass@cluster0.mongodb.net/mydb
      database: mydb

// Entity (Document)
@Document(collection = "orders")
public class Order {
    @Id
    private String id;
    
    @Indexed
    private String userId;
    
    private BigDecimal total;
    private OrderStatus status;
    private List<OrderItem> items;  // embedded sub-documents
    
    @CreatedDate
    private Instant createdAt;
}

// Repository
public interface OrderRepository extends MongoRepository<Order, String> {
    
    List<Order> findByUserIdAndStatus(String userId, OrderStatus status);
    
    @Query("{ 'total': { $gte: ?0 }, 'status': 'COMPLETED' }")
    List<Order> findCompletedOrdersAbove(BigDecimal minTotal);
    
    // Text search
    @Query("{ $text: { $search: ?0 } }")
    List<Order> searchOrders(String text);
}

// MongoTemplate for complex queries
@Service
public class OrderAnalyticsService {
    private final MongoTemplate mongoTemplate;
    
    public List<StatusSummary> getOrderSummaryByStatus() {
        Aggregation agg = Aggregation.newAggregation(
            match(Criteria.where("createdAt").gte(startOfMonth())),
            group("status")
                .count().as("count")
                .avg("total").as("avgTotal"),
            sort(Sort.Direction.DESC, "count")
        );
        
        return mongoTemplate.aggregate(agg, "orders", StatusSummary.class)
            .getMappedResults();
    }
}
```

---

## ⚖️ When to Use / Not Use

| Use MongoDB When | Avoid MongoDB When |
|---|---|
| Rapidly evolving schema (startups, MVPs) | Complex multi-table JOINs are core requirement |
| Document-oriented data (products, content, events) | Strong ACID across multiple documents is critical |
| Hierarchical/nested data (catalogs, comments) | Data is highly relational (many-to-many everywhere) |
| Geospatial queries (location-based services) | Strict schema enforcement needed (financial ledgers) |
| High write throughput with horizontal scaling | Small dataset that fits in PostgreSQL easily |
| Real-time analytics with aggregation pipeline | Need mature tooling for reporting/BI |

---

## ⚠️ Common Pitfalls

1. **Unbounded array growth** — Embedding arrays that grow indefinitely (all user orders in user document) hits the 16MB document limit and degrades performance. If an array can grow beyond ~100 items, use a separate collection with a reference.

2. **Missing indexes on query fields** — MongoDB doesn't auto-index anything except `_id`. Every field in your `find()` filters and `sort()` operations needs an index, or you get collection scans on millions of documents.

3. **Using MongoDB for highly relational data** — If your queries require 5+ `$lookup` stages (JOINs), you're fighting the document model. PostgreSQL with proper indexes would be simpler and faster for relational workloads.

4. **Wrong shard key** — Choosing a shard key with low cardinality (country, status) or monotonically increasing values (timestamp) creates hot spots. This decision is nearly impossible to change later — choose based on your most frequent query pattern.

5. **Ignoring write concern** — Default `w:1` only waits for primary acknowledgment. If primary crashes before replicating, data is lost. For important data, use `w:"majority"` + `j:true` to ensure durability across the replica set.

---

## 🧩 Mini Challenge

**You're building an e-commerce product catalog. Products have varying attributes: electronics have `screen_size`, `ram`, `battery`; clothing has `size`, `color`, `material`. How would you model this in MongoDB vs PostgreSQL?**

<details>
<summary>💡 Click to reveal answer</summary>

**MongoDB (natural fit — flexible schema)**:
```json
// Electronics product
{
  "_id": "prod-1",
  "name": "iPhone 15",
  "category": "electronics",
  "price": 999,
  "attributes": {
    "screen_size": "6.1 inch",
    "ram": "6GB",
    "battery": "3349mAh",
    "chip": "A16 Bionic"
  }
}

// Clothing product (different fields, same collection!)
{
  "_id": "prod-2",
  "name": "Nike Air Max",
  "category": "footwear",
  "price": 150,
  "attributes": {
    "sizes": ["8", "9", "10", "11"],
    "colors": ["black", "white", "red"],
    "material": "mesh + rubber"
  }
}
```
Index on `attributes.*` using wildcard: `db.products.createIndex({ "attributes.$**": 1 })`

**PostgreSQL (requires compromises)**:
- Option A: JSONB column → `attributes JSONB` (flexible but loses type safety and query planning)
- Option B: EAV pattern → separate `product_attributes(product_id, key, value)` table (normalized but 100+ rows per product, slow queries)
- Option C: One table per category → `electronics`, `clothing` tables (type-safe but dozens of tables, complex queries across categories)

**MongoDB wins here**: the document model naturally handles polymorphic schemas. No schema migrations when you add a new product category or attribute.

</details>

---

## 📝 Interview Q&A

**Q: When would you choose MongoDB over PostgreSQL?**
> A: Choose MongoDB when: (1) schema evolves rapidly (startup iteration, A/B testing new fields); (2) data is naturally document-shaped (nested, hierarchical — product catalogs, user profiles with varying attributes); (3) you need horizontal write scaling beyond a single server (sharding is built-in); (4) geospatial queries are central (2dsphere indexes). Choose PostgreSQL when: data is highly relational (many JOINs), strong ACID multi-table transactions are critical (financial systems), or you need mature SQL tooling and ecosystem.

**Q: How does MongoDB ensure consistency in a replica set?**
> A: By default, reads go to the primary (strong consistency for reads). Writes are always to the primary, then asynchronously replicated to secondaries. With `readPreference: secondary`, reads can be stale (eventual consistency). For strong consistency: use `readConcern: "majority"` (read only data acknowledged by majority) + `writeConcern: "majority"` (write confirmed by majority). Since MongoDB 4.0, multi-document transactions provide ACID guarantees with snapshot isolation across multiple documents and collections within a replica set.

---

## 🔗 What to Read Next

1. **[Database/SQL_Vs_NoSQL.md](./SQL_Vs_NoSQL.md)** — When to choose document DB vs relational
2. **[Database/Sharding.md](./Sharding.md)** — Generic sharding concepts MongoDB implements
3. **[Database/Database_Selection_Guide.md](./Database_Selection_Guide.md)** — Complete decision framework for all database types

---

*[← Redis Deep Dive](./Redis_Deep_Dive.md) | [Back to Database](../INDEX.md) | [Next: Cassandra Deep Dive →](./Cassandra_Deep_Dive.md)*
