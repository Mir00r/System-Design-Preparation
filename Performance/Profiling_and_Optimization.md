# 🔍 Profiling & Optimization: Finding and Fixing Bottlenecks

> *"Don't guess where the bottleneck is — MEASURE. Profiling tells you exactly which 5% of code consumes 95% of execution time. Optimizing without profiling is premature optimization — the root of all evil."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Performance README](./README.md), Java basics

---

## 📋 Table of Contents
1. [Profiling Methodology](#-methodology)
2. [CPU Profiling](#-cpu-profiling)
3. [Memory Profiling](#-memory-profiling)
4. [Flame Graphs](#-flame-graphs)
5. [Common Bottlenecks](#-common-bottlenecks)
6. [Optimization Techniques](#-optimization-techniques)
7. [Interview Q&A](#-interview-qa)

---

## 🎯 Methodology

```
THE PERFORMANCE OPTIMIZATION LOOP:

  1. MEASURE (baseline)
     → What is current p50, p95, p99 latency?
     → What is current throughput (RPS)?
     
  2. IDENTIFY (bottleneck)
     → Profile: WHERE is time being spent?
     → CPU? I/O? Memory? Network? Database?
     
  3. HYPOTHESIZE
     → Why is this slow? What would make it faster?
     
  4. FIX (one change at a time)
     → Apply the optimization
     
  5. MEASURE AGAIN
     → Did it actually improve? By how much?
     → No improvement? Revert and try different hypothesis
     
  6. REPEAT until SLO is met

GOLDEN RULES:
  ❌ Never optimize without measuring first
  ❌ Never optimize more than one thing at a time
  ❌ Never optimize something that doesn't matter
  ✅ Focus on the BIGGEST bottleneck first (Amdahl's Law)
  ✅ Measure before AND after every change
```

---

## 🧠 CPU Profiling

```
TOOLS:
  async-profiler:  Low-overhead sampling profiler for JVM (production-safe)
  VisualVM:        GUI profiler (development use)
  JFR (Flight Recorder): Built into JDK, low overhead, production-safe
  YourKit:         Commercial, powerful (free for open source)

ASYNC-PROFILER (recommended for production):
  # Attach to running JVM and profile for 30 seconds
  ./profiler.sh -d 30 -f flamegraph.html <PID>
  
  # Profile specific events
  ./profiler.sh -e cpu -d 30 <PID>           # CPU time
  ./profiler.sh -e wall -d 30 <PID>          # Wall clock (includes I/O waits)
  ./profiler.sh -e alloc -d 30 <PID>         # Memory allocations
  ./profiler.sh -e lock -d 30 <PID>          # Lock contention

JFR (Java Flight Recorder):
  # Start recording
  jcmd <PID> JFR.start duration=60s filename=recording.jfr
  
  # Analyze with JDK Mission Control (JMC)
  # Shows: hot methods, allocations, I/O, threads, GC

WHAT TO LOOK FOR:
  High CPU:
    - Tight loops (inefficient algorithms)
    - Excessive object creation (GC pressure)
    - Regex compilation in hot paths
    - String concatenation in loops
    
  Low CPU but slow:
    - Waiting for I/O (database, network, disk)
    - Lock contention (threads waiting for each other)
    - GC pauses (stop-the-world events)
```

---

## 🔥 Flame Graphs

```
FLAME GRAPH = Visual representation of stack traces over time

  Width = time spent (wider = more time)
  Height = call stack depth
  Color = usually random (or indicates package)

READING A FLAME GRAPH:
  
  ┌─────────────────────────────────────────────────────────────┐
  │                    Thread.run()                              │  ← entry point
  ├───────────────────────────────┬─────────────────────────────┤
  │     RequestHandler.handle()   │    GC.collect()             │
  ├─────────────────┬─────────────┤                             │
  │ Service.process │ DB.query()  │                             │
  ├─────────┬───────┤             │                             │
  │ calc()  │parse()│             │                             │
  └─────────┴───────┴─────────────┴─────────────────────────────┘
  
  INTERPRETATION:
  - RequestHandler.handle() is the widest → most time here
  - DB.query() takes significant time → database is slow
  - GC.collect() is wide → too much garbage collection
  - calc() and parse() are narrow → not bottlenecks
  
  ACTION: Focus on DB.query() and GC pressure (top-wide frames)
```

---

## 💡 Common Bottlenecks

```
1. DATABASE (most common — 60% of performance issues)
   Symptoms: Slow API responses, high DB CPU, many connections
   Causes:   N+1 queries, missing indexes, full table scans, large result sets
   Fix:      Add indexes, use JOINs/batch fetch, pagination, caching

2. MEMORY / GC
   Symptoms: Periodic latency spikes, high GC pause times
   Causes:   Excessive object creation, large heaps, wrong GC algorithm
   Fix:      Reduce allocations, tune GC, use object pooling

3. NETWORK I/O
   Symptoms: Threads blocked on I/O, slow external service calls
   Causes:   Sequential external calls, no timeouts, chatty protocols
   Fix:      Parallel calls, circuit breaker, connection pooling, caching

4. SERIALIZATION
   Symptoms: High CPU in JSON/XML parsing
   Causes:   Large payloads, reflection-based serializers
   Fix:      Use efficient formats (protobuf), reduce payload size, cache

5. LOCK CONTENTION
   Symptoms: Threads WAITING, low CPU despite high load
   Causes:   Shared mutable state, synchronized blocks
   Fix:      Reduce lock scope, use concurrent data structures, lock-free
```

---

## 🛠️ Optimization Techniques

```java
// TECHNIQUE 1: Batch database operations (N+1 → 1 query)
// BAD: N+1 queries
for (Order order : orders) {
    Customer c = customerRepo.findById(order.getCustomerId()); // N queries!
}
// GOOD: Single query with JOIN or IN clause
List<Customer> customers = customerRepo.findAllByIdIn(customerIds); // 1 query

// TECHNIQUE 2: Connection pooling
// BAD: New connection per request
Connection conn = DriverManager.getConnection(url); // 50-200ms per connect!
// GOOD: Pool (HikariCP)
// spring.datasource.hikari.maximum-pool-size=20

// TECHNIQUE 3: Caching
@Cacheable(value = "products", key = "#id")
public Product getProduct(Long id) {
    return productRepository.findById(id).orElseThrow();
}

// TECHNIQUE 4: Async/parallel external calls
// BAD: Sequential (3 calls × 200ms = 600ms)
UserProfile profile = userService.get(userId);      // 200ms
List<Order> orders = orderService.getRecent(userId); // 200ms
int points = loyaltyService.getPoints(userId);       // 200ms

// GOOD: Parallel (max 200ms total)
CompletableFuture<UserProfile> profileF = CompletableFuture.supplyAsync(() -> userService.get(userId));
CompletableFuture<List<Order>> ordersF = CompletableFuture.supplyAsync(() -> orderService.getRecent(userId));
CompletableFuture<Integer> pointsF = CompletableFuture.supplyAsync(() -> loyaltyService.getPoints(userId));
CompletableFuture.allOf(profileF, ordersF, pointsF).join(); // 200ms total
```

---

## ⚠️ Common Pitfalls

1. **Premature optimization** — Don't optimize code that runs once per day. Focus on hot paths (code that runs thousands/millions of times). Profile FIRST, then optimize only what the profiler shows as slow.

2. **Micro-optimizing while ignoring architecture** — Saving 1ms in a loop doesn't matter if the database query takes 500ms. Fix the biggest bottleneck first. Usually: DB > Network > Algorithm > Micro-optimization.

3. **Optimizing for throughput when latency matters (or vice versa)** — Batching improves throughput but increases latency per request. Caching reduces latency but can serve stale data. Understand which metric matters for YOUR use case.

---

## 📝 Interview Q&A

**Q: Your API's p99 latency spiked from 200ms to 5 seconds. How do you diagnose?**
> A: (1) **Check if it's one endpoint or all** — if one endpoint, likely a code/query change. If all, likely infrastructure (GC, resource exhaustion). (2) **Check recent deployments** — correlate with deploy timestamps (most common cause). (3) **Check infrastructure** — CPU, memory, disk I/O, network. High GC? Memory leak? (4) **Check dependencies** — are downstream services/databases slow? Add distributed tracing (Jaeger/Zipkin) to see where time is spent. (5) **Profile** — attach async-profiler, generate flame graph, find the wide bars. (6) **Check database** — slow query log, connection pool exhaustion, lock waits. (7) **p99 specifically** — means 1% of requests are slow. Could be: cold cache misses, GC pauses (affects a few requests), one slow DB query, or one slow external service call.

---

## 🔗 What to Read Next

1. **[Performance/Database_Performance.md](./Database_Performance.md)** — Query optimization
2. **[Performance/JVM_Tuning.md](./JVM_Tuning.md)** — GC and JVM flags
3. **[Performance/Memory_Management.md](./Memory_Management.md)** — Memory profiling and leaks

---

*[← Performance README](./README.md) | [Back to Index](../INDEX.md) | [Next: Database Performance →](./Database_Performance.md)*
