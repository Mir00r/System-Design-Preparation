# 🌍 How the Internet Works: End-to-End Request Lifecycle

> *"When a user clicks a button, a remarkable chain of events unfolds: DNS resolution, TCP connections, TLS encryption, load balancing, application logic, database queries, and response rendering — all in under 200ms. Understanding this chain is understanding system design."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Networking](../Networking/README.md), [Operating Systems](../OperatingSystems/README.md)

---

## 🏗️ The Complete Request Journey

```
USER CLICKS "Buy Now" on amazon.com/product/123

═══════════════════════════════════════════════════════════════════
STEP 1: DNS RESOLUTION (~20-100ms, cached: ~0ms)
═══════════════════════════════════════════════════════════════════

  Browser: "What's the IP for amazon.com?"
  
  Check order:
  1. Browser DNS cache          → miss
  2. OS DNS cache (/etc/hosts)  → miss
  3. Router cache               → miss
  4. ISP recursive resolver     → miss
  5. Root nameserver            → "Try .com TLD"
  6. .com TLD nameserver        → "Try ns1.amazon.com"
  7. amazon.com authoritative   → "IP: 54.239.28.85" (TTL: 60s)
  
  Result: 54.239.28.85 (cached for next 60 seconds)

═══════════════════════════════════════════════════════════════════
STEP 2: TCP + TLS CONNECTION (~30-50ms same region)
═══════════════════════════════════════════════════════════════════

  TCP 3-way handshake:
    Client → SYN → Server
    Client ← SYN+ACK ← Server
    Client → ACK → Server
  
  TLS 1.3 handshake (1 RTT):
    Client → ClientHello + KeyShare → Server
    Client ← ServerHello + Certificate + Finished ← Server
    Client → Finished → Server
  
  Connection established! Encrypted channel ready.
  (Subsequent requests reuse this connection: HTTP keep-alive)

═══════════════════════════════════════════════════════════════════
STEP 3: HTTP REQUEST TRAVELS
═══════════════════════════════════════════════════════════════════

  POST /api/orders HTTP/2
  Host: amazon.com
  Authorization: Bearer eyJhbGciOiJSUz...
  Content-Type: application/json
  
  {"productId": "123", "quantity": 1}
  
  Packet routing:
  Client → ISP → Internet backbone → AWS edge → datacenter
  (Multiple hops via BGP routing, ~10-150ms depending on distance)

═══════════════════════════════════════════════════════════════════
STEP 4: CDN / EDGE (~1-5ms)
═══════════════════════════════════════════════════════════════════

  CloudFront edge location:
  - Static content? → Serve from cache (HTML, JS, images)
  - API request? → Forward to origin (our case)
  - DDoS protection, WAF rules applied here

═══════════════════════════════════════════════════════════════════
STEP 5: LOAD BALANCER (~1-2ms)
═══════════════════════════════════════════════════════════════════

  ALB (Application Load Balancer):
  - Health check: which servers are healthy?
  - Routing: /api/orders → order-service target group
  - Algorithm: least connections (send to least busy server)
  - SSL termination: decrypt here, internal traffic in plaintext (or mTLS)
  
  Selected: order-service-pod-3 (10.0.2.45:8080)

═══════════════════════════════════════════════════════════════════
STEP 6: APPLICATION SERVER (~20-50ms)
═══════════════════════════════════════════════════════════════════

  Spring Boot application processes:
  
  1. Authentication: Validate JWT token (check signature, expiry)
  2. Rate limiting: Check Redis counter (< 100 req/min? proceed)
  3. Request validation: productId exists? quantity > 0?
  4. Business logic:
     - Check inventory (Redis cache → DB fallback)
     - Calculate price (product price + tax + shipping)
     - Verify payment method
  5. Database write: INSERT order + UPDATE inventory
  6. Publish event: "OrderCreated" → Kafka topic
  7. Response: 201 Created with order details

═══════════════════════════════════════════════════════════════════
STEP 7: DATABASE (~5-20ms)
═══════════════════════════════════════════════════════════════════

  PostgreSQL (primary):
  - Connection from pool (no handshake needed)
  - BEGIN TRANSACTION
  - INSERT INTO orders (...)
  - UPDATE inventory SET quantity = quantity - 1 WHERE ...
  - COMMIT
  - Write-ahead log (WAL) → replicated to read replicas

═══════════════════════════════════════════════════════════════════
STEP 8: ASYNC PROCESSING (after response)
═══════════════════════════════════════════════════════════════════

  Kafka consumers (parallel, non-blocking):
  - notification-service → sends order confirmation email
  - analytics-service → updates dashboards
  - inventory-service → triggers restock if low
  - fraud-service → async fraud check

═══════════════════════════════════════════════════════════════════
STEP 9: RESPONSE BACK TO CLIENT (~same path, reverse)
═══════════════════════════════════════════════════════════════════

  HTTP/2 200 OK
  Content-Type: application/json
  
  {"orderId": "ORD-789", "status": "confirmed", "total": "$29.99"}
  
  Server → LB → CDN edge → Internet → ISP → Client
  
  Browser: Update UI → show "Order Confirmed!" 🎉

═══════════════════════════════════════════════════════════════════
TOTAL LATENCY BUDGET:
═══════════════════════════════════════════════════════════════════

  DNS:          0ms (cached)
  TCP+TLS:      0ms (connection reused)
  Network:      30ms (client → server)
  LB:           2ms
  App logic:    40ms
  Database:     15ms
  Network back: 30ms
  ─────────────────
  Total:        ~117ms (well under 200ms SLA)
```

---

## 📊 Latency Breakdown at Scale

| Component | Typical | Optimized | How to Optimize |
|---|---|---|---|
| DNS | 20-100ms | 0ms (cached) | Long TTL, pre-fetch |
| TCP+TLS | 30-50ms | 0ms (reuse) | Connection pooling, HTTP/2 |
| CDN | 5-20ms | 1-5ms | Cache aggressively, edge compute |
| Load Balancer | 1-5ms | 1ms | Same-AZ routing |
| Application | 20-200ms | 10-50ms | Cache, async, optimize queries |
| Database | 5-50ms | 1-5ms | Indexes, read replicas, cache |
| Total | 80-400ms | 15-60ms | Measure and optimize each hop |

---

## ⚠️ Common Pitfalls

1. **Not measuring per-component latency** — If your P99 is 2 seconds, you need to know WHERE the time is spent. Add tracing (OpenTelemetry) to measure each hop. Often it's one slow DB query or an external API call.

2. **Synchronous chains** — If Service A calls B calls C calls D synchronously, latencies ADD (50+50+50+50=200ms). Use async processing for non-critical paths and parallel calls where possible.

3. **Ignoring geography** — Light travels at ~100ms coast-to-coast (US). If your users are in Europe and your servers are in US-East, that's 150ms minimum. Use CDN edges and multi-region deployment for global users.

---

## 📝 Interview Q&A

**Q: How would you reduce the latency of this request from 200ms to 50ms?**
> A: (1) **Eliminate network hops**: deploy CDN edge, pre-resolve DNS. (2) **Connection reuse**: HTTP/2, connection pooling (skip TCP/TLS handshakes). (3) **Cache hot data**: Redis for product prices, inventory counts (skip DB). (4) **Async processing**: don't wait for email/analytics — publish event and respond immediately. (5) **Read replicas**: route reads to replicas close to the app server. (6) **Data locality**: same-AZ deployment for app→DB (1ms vs 5ms cross-AZ). (7) **Pre-compute**: materialize frequently-accessed data instead of joining tables at query time.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/CDN.md](../../BuildingBlocks/CDN.md)** — Content delivery in depth
2. **[BuildingBlocks/LoadBalancing.md](../../BuildingBlocks/LoadBalancing.md)** — Load balancing strategies
3. **[Observability/Distributed_Tracing.md](../../Observability/Distributed_Tracing.md)** — Tracing requests across services

---

*[← Operating Systems](../OperatingSystems/README.md) | [Back to Index](../../INDEX.md)*
