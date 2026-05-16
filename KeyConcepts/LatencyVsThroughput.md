# ⚡ Latency vs Throughput: The Fundamental Performance Trade-Off

> *"Latency is how long you wait for one thing. Throughput is how many things you complete per unit time. Optimizing for one often hurts the other — and understanding WHY is the key to building performant distributed systems."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟢 Beginner-Medium | **🔗 Prerequisites**: [Scalability](./Scalability.md)

---

## 📋 Table of Contents
1. [Definitions](#-definitions)
2. [The Relationship](#-the-relationship)
3. [Little's Law](#-littles-law)
4. [Measuring Correctly](#-measuring-correctly)
5. [Optimization Strategies](#-optimization-strategies)
6. [System Design Applications](#-system-design-applications)
7. [Code Example](#-code-example)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 📐 Definitions

```
LATENCY (response time):
  Time from request sent → response received
  Measured in: milliseconds (ms) or microseconds (µs)
  
  Components:
    Network latency:    time for packets to travel (speed of light + hops)
    Processing time:    time server spends computing the response
    Queuing time:       time request waits before being processed
    Serialization:      time to encode/decode data
  
  Total latency = network + queue + processing + serialization

THROUGHPUT (bandwidth):
  Number of operations completed per unit time
  Measured in: requests/second (RPS), transactions/second (TPS),
               bytes/second (MB/s), messages/second

BANDWIDTH (capacity):
  Maximum data transfer rate of a channel
  Measured in: bits/second (Mbps, Gbps)
  
  Analogy:
    Bandwidth  = width of a highway (how many lanes)
    Throughput = actual cars per hour (depends on traffic)
    Latency    = time for ONE car to travel from A to B
```

---

## 🔗 The Relationship

```
THE BATHTUB ANALOGY:
  ┌─────────────────────────────┐
  │  Incoming requests (faucet) │  = arrival rate
  │         │                   │
  │         ▼                   │
  │  ┌─────────────────┐       │
  │  │  Queue/System   │       │  = concurrent requests
  │  │  (bathtub)      │       │
  │  └────────┬────────┘       │
  │           │                 │
  │           ▼                 │
  │  Completed responses (drain)│  = throughput
  └─────────────────────────────┘

  If faucet (arrivals) > drain (throughput):
    → Bathtub fills up (queue grows)
    → Latency increases (waiting in queue)
    → Eventually overflow (requests rejected/timed out)

THE TRADE-OFF:
  High throughput often means:
    - Batching (process many at once → each waits for batch to fill)
    - Buffering (queue requests → higher latency under load)
    - Sharing resources (context switching, contention)

  Low latency often means:
    - Process immediately (no batching → lower throughput)
    - Dedicated resources (expensive, less sharing)
    - Over-provision capacity (handle peaks instantly)

RELATIONSHIP UNDER LOAD:
  
  Latency
  (ms)
   │
   │                      ╱ (system saturated)
   │                    ╱
   │                  ╱
   │               ╱
   │           ╱
   │       ╱
   │___╱___________________ → Throughput (RPS)
   │   ↑                ↑
       Sweet spot        Max capacity
       (low latency,     (latency spikes,
        good throughput)   queuing delay)
```

---

## 📏 Little's Law

```
LITTLE'S LAW (fundamental queueing theory):

  L = λ × W

  L = average number of requests in the system (concurrency)
  λ = average arrival rate (throughput — requests/second)
  W = average time a request spends in the system (latency)

EXAMPLES:

  Example 1: Web server
    Throughput: 1000 req/sec
    Latency: 50ms (0.05s)
    Concurrent requests: L = 1000 × 0.05 = 50 concurrent connections

  Example 2: Database connection pool
    Want to handle: 2000 queries/sec
    Average query time: 10ms (0.01s)
    Pool size needed: L = 2000 × 0.01 = 20 connections minimum

  Example 3: Capacity planning
    Target latency: 100ms (SLO)
    Available threads: 200
    Max throughput: λ = L / W = 200 / 0.1 = 2000 req/sec

PRACTICAL APPLICATION:
  "How many instances do I need?"
  
  Target: 10,000 req/sec at < 200ms latency
  Each instance handles: 200 / 0.2 = 1000 req/sec (200 threads, 200ms each)
  Instances needed: 10,000 / 1000 = 10 instances
  Add 50% headroom: 15 instances (for spikes and failures)
```

---

## 📊 Measuring Correctly

```
DON'T USE AVERAGES FOR LATENCY — USE PERCENTILES:

  Average: 50ms ← meaningless! Hides outliers.
  
  Percentiles tell the truth:
    p50 (median): 30ms   (half of requests are faster)
    p90:          80ms   (90% of requests are faster)
    p95:          150ms  (95% of requests are faster)
    p99:          500ms  (1% of requests take > 500ms!)
    p99.9:        2000ms (1 in 1000 takes > 2 seconds)

WHY P99 MATTERS:
  If you have 1000 users/sec, 10 users/sec experience p99 latency.
  Those 10 users are often your MOST VALUABLE (heavy users, many requests).
  Amazon found: every 100ms of p99 latency = 1% revenue loss.

TAIL LATENCY AMPLIFICATION:
  Microservice call chain: A → B → C → D

  If each service has p99 = 100ms:
    Single call probability of hitting p99: 1%
    Chain of 4 calls probability ANY hits p99: ~4%
    → 4% of user requests experience > 100ms on ONE hop

  If service B fans out to 10 shards (scatter-gather):
    Probability all 10 respond within p99: (0.99)^10 = 90%
    → 10% of requests hit tail latency on at least one shard!

MEASUREMENT TOOLS:
  - Histograms (Prometheus histogram_quantile)
  - HdrHistogram (high dynamic range, no data loss)
  - Percentile aggregation across instances (NOT average of percentiles!)
```

---

## 🎯 Optimization Strategies

```
OPTIMIZE FOR LATENCY:
  1. Cache aggressively (Redis, local cache — avoid network hops)
  2. Reduce network hops (colocate services, edge computing)
  3. Connection pooling (avoid TCP handshake per request)
  4. Async I/O (don't block threads waiting for responses)
  5. Pre-compute (compute during write, not during read)
  6. Geographic placement (CDN, regional deployments)

OPTIMIZE FOR THROUGHPUT:
  1. Batching (process many requests in one operation)
  2. Pipelining (send multiple requests without waiting for responses)
  3. Compression (send less data per request)
  4. Horizontal scaling (more instances = more total capacity)
  5. Async processing (queue work, process in background)
  6. Connection multiplexing (HTTP/2, gRPC — many requests per connection)

OPTIMIZE FOR BOTH (the holy grail):
  1. Reduce processing time (faster algorithms, less work per request)
  2. Eliminate waste (unnecessary serialization, logging, allocations)
  3. Right-size resources (avoid contention AND over-provisioning)
  4. Load shedding (reject excess load early, protect system from overload)
```

---

## 🏗️ System Design Applications

```
DATABASE OPERATIONS:
  Low latency needed:     point lookup by primary key (O(1), in-memory index)
  High throughput needed: bulk inserts, ETL (batch, disable indexes during load)
  
MESSAGING SYSTEMS:
  Kafka (throughput-optimized): batch messages, high throughput, higher latency
  Redis Pub/Sub (latency-optimized): immediate delivery, lower throughput
  
API DESIGN:
  REST (per-request): lower latency per call, more network round trips
  GraphQL (batched): one request gets everything, potentially higher per-request latency

CACHING STRATEGY:
  Write-through: higher write latency (write to cache + DB), lower read latency
  Write-behind: lower write latency (write to cache only), risk of data loss
  
CONSISTENCY vs PERFORMANCE:
  Strong consistency: higher latency (wait for all replicas)
  Eventual consistency: lower latency (respond after primary acknowledges)
```

---

## 💻 Code Example

```java
// Demonstrating latency vs throughput trade-off with batching

// APPROACH 1: Process immediately (low latency, lower throughput)
@Service
public class ImmediateProcessor {
    public void processEvent(Event event) {
        // Each event processed individually — low latency per event
        // But: 1 DB call per event = max ~1000 events/sec (DB bottleneck)
        repository.save(event);  // ~5ms per call
    }
    // Latency: 5ms per event
    // Throughput: ~200 events/sec/thread
}

// APPROACH 2: Batch processing (higher latency, much higher throughput)
@Service
public class BatchProcessor {
    private final BlockingQueue<Event> buffer = new LinkedBlockingQueue<>(10000);
    
    public void processEvent(Event event) {
        buffer.offer(event);  // returns immediately — submillisecond
    }
    
    @Scheduled(fixedDelay = 100)  // flush every 100ms
    public void flush() {
        List<Event> batch = new ArrayList<>();
        buffer.drainTo(batch, 1000);  // grab up to 1000 events
        if (!batch.isEmpty()) {
            repository.saveAll(batch);  // 1 DB call for 1000 events — ~20ms total
        }
    }
    // Latency: 50-150ms per event (waiting for batch + processing)
    // Throughput: ~50,000 events/sec (1000 per batch, 10 batches/sec)
}

// APPROACH 3: Adaptive — batch under load, immediate when idle
@Service
public class AdaptiveProcessor {
    private final AtomicInteger pendingCount = new AtomicInteger(0);
    
    public CompletableFuture<Void> processEvent(Event event) {
        int pending = pendingCount.incrementAndGet();
        
        if (pending < 10) {
            // Low load: process immediately (low latency)
            return CompletableFuture.runAsync(() -> {
                repository.save(event);
                pendingCount.decrementAndGet();
            });
        } else {
            // High load: add to batch (high throughput)
            return batchProcessor.add(event)
                .whenComplete((v, ex) -> pendingCount.decrementAndGet());
        }
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Measuring average latency instead of percentiles** — An average of 50ms can hide that 1% of requests take 5 seconds. Always report p50, p95, p99. Set SLOs on p99 (e.g., "p99 latency < 200ms").

2. **Confusing bandwidth with throughput** — Your network has 10Gbps bandwidth, but actual throughput may be 2Gbps due to protocol overhead, packet loss, TCP window sizes, and application-level serialization bottlenecks.

3. **Optimizing for throughput when latency is the problem** — Adding more servers increases throughput but doesn't reduce per-request latency if the bottleneck is a slow database query or external API call. Profile first, then optimize the right dimension.

4. **Ignoring tail latency in distributed systems** — If your service calls 5 microservices in parallel (fan-out), the overall latency = slowest response. One slow service ruins the entire request. Set aggressive timeouts and use fallbacks for slow dependencies.

---

## 🧩 Mini Challenge

**You have a service that handles 5,000 requests/second. Average latency is 40ms but p99 is 2 seconds. Using Little's Law, calculate: (a) average concurrent connections, (b) how many of those are "stuck" on p99 requests, (c) propose a fix.**

<details>
<summary>💡 Click to reveal answer</summary>

**Calculations:**

(a) Average concurrent connections:
```
L = λ × W = 5000 req/s × 0.040s = 200 concurrent connections (average)
```

(b) P99 connections (1% of requests take 2s):
```
P99 requests: 5000 × 0.01 = 50 req/s experiencing 2-second latency
These 50 req/s each occupy a connection for 2s:
L_p99 = 50 × 2.0 = 100 connections stuck on slow requests!

So: 100 out of 200 concurrent connections (50%!) are occupied by 
the slowest 1% of requests. This is the tail latency problem.
```

(c) **Fixes:**
1. **Timeout at 500ms**: Kill requests that take > 500ms, return error or fallback. 
   - New p99 = 500ms (capped), connections freed sooner
   - Freed: 100 connections → only 50 × 0.5 = 25 connections for slow requests

2. **Find root cause of 2s tail**: Profile p99 requests. Common causes:
   - GC pauses (tune GC, reduce allocations)
   - One slow database query (add index, cache)
   - Network timeout to one dependency (circuit breaker)

3. **Hedged requests**: After 100ms, send duplicate request to another server.
   First response wins. Dramatically reduces tail latency.

4. **Separate fast/slow paths**: Route potentially-slow requests to a different 
   pool/queue so they don't block fast requests from getting connections.

</details>

---

## 📝 Interview Q&A

**Q: A system handles 10K RPS with p50=20ms but p99=5 seconds. What would you investigate?**
> A: This pattern (low median, extreme p99) suggests an intermittent bottleneck affecting 1% of requests. I'd investigate: (1) **Garbage collection** — check GC logs for stop-the-world pauses matching the 5s duration. (2) **Connection pool exhaustion** — if pool size is too small, requests queue for connections. (3) **Lock contention** — synchronized blocks or row-level locks causing some threads to wait. (4) **Downstream dependency** — one backend service occasionally times out at 5s. (5) **Resource limits** — file descriptors, thread pool saturation, TCP backlog overflow. I'd correlate p99 events with system metrics (CPU, GC, thread dumps) to identify the bottleneck. Tools: async-profiler for CPU/lock profiling, thread dumps for contention, distributed tracing to identify slow hops.

**Q: Explain Little's Law and how you'd use it in capacity planning.**
> A: Little's Law states L = λ × W (concurrent requests = throughput × latency). For capacity planning: if my SLO is p99 < 200ms at 20,000 RPS, then I need at minimum L = 20,000 × 0.2 = 4,000 concurrent connections available. If each server handles 500 concurrent connections (thread pool + async I/O), I need 8 servers minimum. Add 50-100% headroom for spikes: 12-16 servers. For database pool sizing: if app makes 2 DB queries per request at 5ms each, DB concurrency = 20,000 × 0.005 × 2 = 200 connections. With 4 app servers, each needs a pool of 50 connections to the database.

---

## 🔗 What to Read Next

1. **[KeyConcepts/Scalability.md](./Scalability.md)** — How to scale latency and throughput
2. **[Performance/Profiling_and_Optimization.md](../Performance/Profiling_and_Optimization.md)** — Finding and fixing bottlenecks
3. **[BuildingBlocks/LoadBalancing.md](../BuildingBlocks/LoadBalancing.md)** — Distributing load for throughput

---

*[← Fault Tolerance](./FaultTolerance.md) | [Back to Index](../INDEX.md)*
