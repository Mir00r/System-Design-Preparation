# 🌐 Network & I/O Performance: Taming the Latency Monster

> *"Light takes 67ms to travel from New York to London. You can't break physics, but you can be SMART about how many round trips you make."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Caching Performance](./Caching_Performance.md)

---

## 📋 Table of Contents
1. [Network Latency Breakdown](#-network-latency-breakdown)
2. [Connection Pooling](#-connection-pooling)
3. [Batching & Reducing Round Trips](#-batching--reducing-round-trips)
4. [Compression & Serialization](#-compression--serialization)
5. [I/O Models (Blocking vs Non-Blocking)](#-io-models)
6. [Real-World Optimizations](#-real-world-optimizations)
7. [Interview Q&A](#-interview-qa)
8. [Boss Battle](#-boss-battle)

---

## 🏗️ Network Latency Breakdown

```
ANATOMY OF AN HTTP REQUEST:

  ┌─────────────────────────────────────────────────────────┐
  │ 1. DNS Lookup          ~1-50ms (cached: 0ms)            │
  │ 2. TCP Handshake       ~RTT/2 (3-way: SYN/SYN-ACK/ACK) │
  │ 3. TLS Handshake       ~1-2 RTT (key exchange)          │
  │ 4. Request Transfer    ~size/bandwidth                   │
  │ 5. Server Processing   ~varies (your code!)             │
  │ 6. Response Transfer   ~size/bandwidth                   │
  └─────────────────────────────────────────────────────────┘

  SAME DATACENTER:     RTT ≈ 0.5ms
  SAME REGION:         RTT ≈ 1-5ms
  CROSS-REGION:        RTT ≈ 30-100ms
  CROSS-CONTINENT:     RTT ≈ 100-200ms

KEY INSIGHT: A new HTTPS connection costs 3-4 RTTs before ANY data flows!
  DNS + TCP + TLS + Request = 3.5 RTTs minimum
  Cross-continent: 3.5 × 150ms = 525ms just for setup! 😱
  
SOLUTION: Keep connections alive! (Connection pooling, HTTP/2)
```

---

## 🔗 Connection Pooling

```
WITHOUT CONNECTION POOL:
  Request 1: Open → Query → Close   (100ms overhead)
  Request 2: Open → Query → Close   (100ms overhead)
  Request 3: Open → Query → Close   (100ms overhead)
  Total overhead: 300ms for 3 requests

WITH CONNECTION POOL:
  Startup: Open 10 connections (one-time cost)
  Request 1: Borrow → Query → Return   (0ms overhead)
  Request 2: Borrow → Query → Return   (0ms overhead)
  Request 3: Borrow → Query → Return   (0ms overhead)
  Total overhead: ~0ms! Connections are pre-established.
```

### HikariCP Configuration (The Fastest Java Pool)

```java
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # Max connections
      minimum-idle: 5                # Keep 5 ready even when idle
      connection-timeout: 30000      # Wait 30s for connection (then fail)
      idle-timeout: 600000           # Close idle connections after 10 min
      max-lifetime: 1800000          # Recycle connections every 30 min
      leak-detection-threshold: 5000 # Alert if connection held > 5s
```

### HTTP Client Connection Pool

```java
// RestTemplate with connection pooling
@Bean
public RestTemplate restTemplate() {
    PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
    cm.setMaxTotal(200);                // Total connections across all hosts
    cm.setDefaultMaxPerRoute(50);       // Max connections per host
    
    CloseableHttpClient client = HttpClients.custom()
        .setConnectionManager(cm)
        .setKeepAliveStrategy((response, context) -> 30_000) // 30s keep-alive
        .build();
        
    return new RestTemplate(new HttpComponentsClientHttpRequestFactory(client));
}
```

---

## 📦 Batching & Reducing Round Trips

```
EVERY NETWORK ROUND TRIP = latency tax!

❌ N+1 PROBLEM (The Performance Killer):
  // Get user's 50 orders
  List<Order> orders = getOrders(userId);  // 1 query
  for (Order order : orders) {
      Product p = getProduct(order.productId);  // 50 queries! 😱
  }
  // Total: 51 round trips × 5ms = 255ms

✅ BATCH APPROACH:
  List<Order> orders = getOrders(userId);  // 1 query
  Set<Long> productIds = orders.stream().map(Order::getProductId).collect(toSet());
  Map<Long, Product> products = getProductsByIds(productIds);  // 1 query!
  // Total: 2 round trips × 5ms = 10ms  (25x faster!)
```

### Strategies for Reducing Round Trips

| Technique | Before | After | Savings |
|-----------|--------|-------|---------|
| **Batch API calls** | N calls | 1 call | (N-1) × RTT |
| **HTTP/2 multiplexing** | Sequential requests | Parallel on 1 connection | TLS handshakes |
| **GraphQL** | Multiple REST calls | 1 shaped query | (N-1) × RTT |
| **JOINs instead of N+1** | N+1 DB queries | 1 JOIN query | N × DB RTT |
| **Prefetching** | Load on demand | Predict & preload | Perceived latency |
| **Compression** | Large payloads | Smaller payloads | Transfer time |

---

## 🗜️ Compression & Serialization

```
SERIALIZATION FORMAT COMPARISON:

Format      │ Size (bytes) │ Serialize │ Deserialize │ Human Readable
────────────┼──────────────┼───────────┼─────────────┼───────────────
JSON        │ 100 (1.0x)   │ Medium    │ Medium      │ ✅ Yes
Protobuf    │ 40 (0.4x)   │ Fast      │ Fast        │ ❌ No
MessagePack │ 55 (0.55x)  │ Fast      │ Fast        │ ❌ No
XML         │ 150 (1.5x)  │ Slow      │ Slow        │ ✅ Yes (verbose)
Avro        │ 38 (0.38x)  │ Fast      │ Fast        │ ❌ No (schema)

RULE OF THUMB:
  - External APIs: JSON (universal, debuggable)
  - Internal service-to-service: Protobuf/gRPC (2-3x smaller, faster)
  - Event streaming (Kafka): Avro (schema evolution!)
  - Mobile clients: Protobuf (bandwidth matters!)

COMPRESSION:
  JSON: 1000 bytes
  JSON + gzip: ~200 bytes (80% reduction!)
  Protobuf: 400 bytes
  Protobuf + gzip: ~150 bytes (85% reduction!)
  
  Enable compression: spring.server.compression.enabled=true
  Threshold: Only compress if payload > 2KB (CPU cost vs size savings)
```

---

## ⚙️ I/O Models

```
1. BLOCKING I/O (Traditional — Thread per request)
   Thread-1: ████████░░░░░░░░████  (██=working, ░░=waiting for I/O)
   Thread-2: ███░░░░░░░░░░████████
   Thread-3: ██████░░░░░░░░░░░████
   
   Problem: Thread BLOCKED during I/O → wasted resource
   1 thread per connection → 10K connections = 10K threads = 10GB RAM
   
2. NON-BLOCKING I/O (Reactor/Netty — Event loop)
   Thread-1: ████████████████████████████████  (always working!)
   
   One thread handles MANY connections
   When I/O needed: register callback, move to next request
   When I/O complete: callback fires, continue processing
   
   10K connections = 1-2 threads = minimal RAM!

3. VIRTUAL THREADS (Java 21+ — Best of both worlds!)
   Write code like blocking (simple!)
   Runtime handles like non-blocking (efficient!)
   1M connections = lightweight virtual threads
   No callback hell, no reactive complexity!
```

### When to Use Which?

| Model | Framework | Best For |
|-------|-----------|----------|
| Blocking + thread pool | Spring MVC | Typical CRUD APIs, moderate load |
| Non-blocking reactive | Spring WebFlux | High-concurrency streaming, gateways |
| Virtual threads | Spring MVC (Java 21+) | High-concurrency with simple code |

---

## 🏢 Real-World Optimizations

| Company | Problem | Solution | Result |
|---------|---------|----------|--------|
| **Netflix** | 800+ microservice calls per page | Parallel fan-out + fallbacks | <200ms page load |
| **Discord** | Millions of WebSocket connections | Elixir + BEAM (lightweight processes) | 5M concurrent on single cluster |
| **Uber** | Geolocation queries across regions | Regional sharding + edge caching | p99 < 50ms |
| **LinkedIn** | N+1 API calls from mobile | GraphQL-like batching (Rest.li) | 3x fewer round trips |

---

## 🎓 Interview Q&A

### Q1: "How do you optimize a service that makes many downstream API calls?"
**A**: Parallelize independent calls (CompletableFuture.allOf), batch where possible, set aggressive timeouts with fallbacks, use connection pooling with HTTP/2 multiplexing, cache responses for repeated calls, consider GraphQL to reduce over-fetching.

### Q2: "What's the N+1 problem and how do you solve it?"
**A**: Making 1 query to get a list, then N queries for each item's details. Solutions: batch loading (IN clause), JOINs, DataLoader pattern (GraphQL), eager fetching with entity graphs (JPA). Always profile queries to detect N+1 patterns.

### Q3: "Blocking vs non-blocking I/O — when do you choose which?"
**A**: For most CRUD APIs with moderate concurrency (<10K concurrent connections), blocking with a thread pool (Spring MVC) is simpler and sufficient. For extreme concurrency (100K+ connections) or streaming/real-time apps, non-blocking (WebFlux/Netty) or virtual threads (Java 21+) are better. Virtual threads give the simplicity of blocking with the efficiency of non-blocking.

---

## 🎲 Boss Battle: API Performance Audit 🔍

> **Scenario**: A mobile app's "home feed" API takes 2 seconds. Profile shows:
> - 3 sequential downstream HTTP calls (each ~400ms)
> - Response payload: 800KB JSON
> - New TCP connection per request (no pooling)
> - Serving data that changes once per hour
>
> **Challenge**: Get it under 200ms. List all optimizations.
>
> <details>
> <summary>🔓 Click to reveal answer</summary>
>
> **Optimization Stack:**
>
> 1. **Parallelize** 3 calls: 3×400ms → max(400ms) = **400ms** ✅
> 2. **Connection pool**: Eliminate TCP/TLS overhead → **350ms** ✅
> 3. **Cache** (data changes hourly): Cache-Aside with 30min TTL, hit ratio ~95% → **2ms** for cached ✅
> 4. **Compress** response: 800KB → 150KB with gzip → faster transfer ✅
> 5. **Paginate/trim**: Mobile doesn't need all 800KB. Return first 10 items = ~50KB ✅
> 6. **HTTP/2**: Multiplex parallel calls on single connection ✅
>
> **Combined Result:**
> - Cache hit (95%): 2ms response + 150KB transfer
> - Cache miss (5%): 400ms parallel + compress = ~450ms
> - Weighted average: 0.95×2 + 0.05×450 = **24ms** 🚀
>
> **From 2000ms → 24ms = 83x improvement!**
> </details>

---

👉 **[Next: Performance Testing & SRE →](./Performance_Testing_SRE.md)**  
👉 **[Back to Performance Overview →](./README.md)**
