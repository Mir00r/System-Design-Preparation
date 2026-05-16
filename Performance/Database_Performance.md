# 🗄️ Database Performance: Query Optimization, N+1, and Connection Pooling

> *"The database is the #1 performance bottleneck in most applications. A single missing index can turn a 5ms query into a 5-second full table scan. Understanding how to diagnose and fix slow queries is the highest-ROI performance skill."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Database/Indexing](../Database/Indexing.md), [Database/QueryOptimization](../Database/QueryOptimization.md)

---

## 📋 Table of Contents
1. [The N+1 Query Problem](#-n1-problem)
2. [Query Optimization](#-query-optimization)
3. [Connection Pooling](#-connection-pooling)
4. [Indexing Strategy](#-indexing-strategy)
5. [Common Patterns](#-common-patterns)
6. [Interview Q&A](#-interview-qa)

---

## 🐌 N+1 Problem

```
THE MOST COMMON PERFORMANCE BUG IN ORM-BASED APPS:

  // Load 100 orders
  List<Order> orders = orderRepository.findAll(); // 1 query: SELECT * FROM orders
  
  // For each order, load customer (LAZY loading triggers N queries!)
  for (Order order : orders) {
      String name = order.getCustomer().getName(); // SELECT * FROM customers WHERE id = ?
  }
  
  RESULT: 1 + 100 = 101 queries! (N+1 where N=100)
  
  With 1000 orders: 1001 queries → 10+ seconds!

FIXES:

  // FIX 1: JOIN FETCH (single query with JOIN)
  @Query("SELECT o FROM Order o JOIN FETCH o.customer")
  List<Order> findAllWithCustomers();
  // Result: 1 query with JOIN
  
  // FIX 2: @EntityGraph
  @EntityGraph(attributePaths = {"customer"})
  List<Order> findAll();
  // Result: 1 query with LEFT JOIN
  
  // FIX 3: Batch fetching
  @BatchSize(size = 50)  // on Customer entity
  // Result: 1 + ceil(100/50) = 3 queries (batched IN clauses)
  
  // FIX 4: DTO projection (no entity, no lazy loading)
  @Query("SELECT new com.app.dto.OrderSummary(o.id, c.name, o.total) " +
         "FROM Order o JOIN o.customer c")
  List<OrderSummary> findOrderSummaries();
  // Result: 1 query, no entities loaded at all (fastest!)
```

---

## 🔍 Query Optimization

```
EXPLAIN ANALYZE (PostgreSQL):
  EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 'cust-123' AND status = 'PENDING';
  
  BAD result:
    Seq Scan on orders  (cost=0.00..15234.00 rows=50 width=128)
    → Full table scan! Reads ALL rows. Time: 500ms
    
  GOOD result (with index):
    Index Scan using idx_orders_customer_status on orders  (cost=0.43..12.50 rows=50 width=128)
    → Uses index. Time: 2ms

COMMON SLOW QUERY PATTERNS:

  1. Missing index on WHERE/JOIN columns
     Fix: CREATE INDEX idx_orders_customer ON orders(customer_id);
     
  2. SELECT * when you need 3 columns
     Fix: SELECT id, status, total FROM orders (less I/O, fits in index)
     
  3. LIKE '%search%' (can't use index)
     Fix: Full-text search (PostgreSQL ts_vector) or Elasticsearch
     
  4. ORDER BY without index
     Fix: Composite index matching WHERE + ORDER BY
     CREATE INDEX idx_orders_date ON orders(customer_id, created_at DESC);
     
  5. Subquery that runs for each row (correlated subquery)
     Fix: Rewrite as JOIN or window function
     
  6. No LIMIT on large result sets
     Fix: Always paginate. LIMIT 20 OFFSET 40 (or keyset pagination)
```

---

## 🏊 Connection Pooling

```
WHY POOL:
  Opening a DB connection: TCP handshake + SSL + auth = 50-200ms
  With pool: borrow existing connection = <1ms
  
  Without pool: 100 concurrent requests = 100 simultaneous connections
    → DB overwhelmed (PostgreSQL: each connection = ~10MB RAM on server)
  With pool: 100 requests share 20 connections (queue when all busy)

HIKARICP CONFIGURATION (Spring Boot):
  spring:
    datasource:
      hikari:
        maximum-pool-size: 20        # max connections in pool
        minimum-idle: 5               # keep 5 connections warm
        connection-timeout: 30000     # wait 30s for connection (else error)
        idle-timeout: 600000          # close idle connections after 10min
        max-lifetime: 1800000         # refresh connections every 30min
        leak-detection-threshold: 60000  # warn if connection held > 60s

SIZING THE POOL:
  Formula: pool_size = (core_count * 2) + disk_spindles
  Example: 4 cores, SSD → pool_size = (4 * 2) + 1 = 9 connections
  
  Common mistake: setting pool to 100 (PostgreSQL can only handle ~300 total)
  Better: small pool (10-20) with proper queueing
  
  If pool is too small: requests queue → latency increases
  If pool is too large: DB overwhelmed → everything slows down
```

---

## 📊 Indexing Strategy

```
INDEX DECISION MATRIX:
  ┌─────────────────────────────┬─────────────────────────────────────┐
  │ Query Pattern               │ Index Strategy                      │
  ├─────────────────────────────┼─────────────────────────────────────┤
  │ WHERE col = value           │ B-tree index on col                 │
  │ WHERE col1 = ? AND col2 = ? │ Composite index (col1, col2)       │
  │ WHERE col1 = ? ORDER BY col2│ Composite index (col1, col2)       │
  │ WHERE col IN (...)          │ B-tree index on col                 │
  │ WHERE col LIKE 'prefix%'    │ B-tree index (prefix matching only) │
  │ WHERE col LIKE '%middle%'   │ Full-text index / trigram           │
  │ Range query (BETWEEN, >, <) │ B-tree (range column LAST in composite)│
  │ JSON field query            │ GIN index on jsonb column           │
  └─────────────────────────────┴─────────────────────────────────────┘

COMPOSITE INDEX ORDER MATTERS:
  Index on (customer_id, status, created_at)
  
  ✅ WHERE customer_id = ?                          (uses index)
  ✅ WHERE customer_id = ? AND status = ?           (uses index)
  ✅ WHERE customer_id = ? AND status = ? AND created_at > ? (uses index)
  ❌ WHERE status = ?                               (can't skip customer_id!)
  ❌ WHERE created_at > ?                           (can't skip first columns!)
  
  Rule: leftmost prefix must be present in WHERE clause
```

---

## ⚠️ Common Pitfalls

1. **Adding indexes without checking write impact** — Each index slows down INSERT/UPDATE (index must be maintained). Table with 10 indexes: inserts are 10x slower. Only index columns you actually query on.

2. **ORM-generated queries you never inspect** — Enable SQL logging in development (`spring.jpa.show-sql=true`). Check what Hibernate actually generates. You'll be surprised by the N+1 queries hiding in your code.

3. **Not using pagination** — `findAll()` loading 1M rows into memory crashes the JVM. Always use `Pageable` or keyset pagination for any query that could return large result sets.

---

## 📝 Interview Q&A

**Q: Your API suddenly became slow. Database CPU is 95%. How do you fix it?**
> A: (1) **Immediate**: identify the slow query — check `pg_stat_activity` for long-running queries, enable slow query log (queries > 1s). (2) **Analyze**: `EXPLAIN ANALYZE` on the slow query — is it a Seq Scan (missing index)? Nested Loop on large tables (missing JOIN index)? (3) **Quick fixes**: add missing index (immediate effect), add LIMIT if fetching too many rows, kill long-running queries blocking others. (4) **Medium-term**: add caching for frequent reads (ElastiCache), optimize ORM queries (N+1 → JOIN FETCH), use read replicas for read-heavy workloads. (5) **Prevent recurrence**: enable slow query alerts, add query cost CI checks, review all new queries in PR reviews.

---

## 🔗 What to Read Next

1. **[Performance/JVM_Tuning.md](./JVM_Tuning.md)** — GC tuning
2. **[Database/Indexing.md](../Database/Indexing.md)** — Indexing deep dive
3. **[Database/ConnectionPooling.md](../Database/ConnectionPooling.md)** — Pool configuration

---

*[← Profiling](./Profiling_and_Optimization.md) | [Back to Index](../INDEX.md) | [Next: JVM Tuning →](./JVM_Tuning.md)*
