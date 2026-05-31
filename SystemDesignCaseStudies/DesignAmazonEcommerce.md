# 🛒 Design Amazon E-Commerce

> *"Amazon processes 4,000+ orders per SECOND during peak. Behind that 'Buy Now' button lies one of the most complex distributed systems ever built: real-time inventory across millions of warehouses, personalized recommendations for 300M+ users, a payment system handling billions, and a logistics engine that can deliver packages in hours. This is the ULTIMATE system design question — it touches every fundamental concept."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Microservices](../Microservices/), [CQRS](../Architectures/CQRS.md), [Message Queues](../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [Requirements & Scale](#-requirements--scale)
2. [High-Level Architecture](#-high-level-architecture)
3. [Product Catalog & Search](#-product-catalog--search)
4. [Shopping Cart](#-shopping-cart)
5. [Order Processing](#-order-processing)
6. [Inventory Management](#-inventory-management)
7. [Payment System](#-payment-system)
8. [Recommendations](#-recommendations)
9. [Java Implementation](#-java-implementation)
10. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements & Scale

```
FUNCTIONAL:
  • Product catalog (search, browse, filter, sort)
  • User accounts (profile, addresses, payment methods)
  • Shopping cart (add/remove items, persist across sessions!)
  • Checkout & order placement
  • Payment processing (multiple methods!)
  • Order tracking (real-time status updates!)
  • Reviews & ratings
  • Recommendations ("Customers also bought...")
  • Seller management (third-party marketplace!)
  
NON-FUNCTIONAL:
  • 300M+ active users
  • 350M+ products in catalog
  • 4,000 orders/second (peak: 50K+ during Prime Day!)
  • Search latency: < 200ms
  • Checkout: < 2 seconds end-to-end
  • 99.99% availability (downtime = millions lost!)
  • Global deployment (serve users worldwide!)

DATA SCALE:
  • Product catalog: 350M products × 5 KB = 1.75 TB
  • User data: 300M users × 2 KB = 600 GB
  • Order history: 10B orders × 1 KB = 10 TB
  • Product images: 350M × 10 images = 500 PB (S3!)
  • Search index: ~50 TB (Elasticsearch)
```

---

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          E-COMMERCE PLATFORM                              │
│                                                                           │
│  Client (Web/Mobile)                                                      │
│       │                                                                   │
│       ▼                                                                   │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  CDN (Static assets, product images, JS/CSS)                       │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│       │                                                                   │
│       ▼                                                                   │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  API Gateway (auth, rate limiting, routing)                        │   │
│  └────────┬──────────┬──────────┬──────────┬──────────┬──────────────┘   │
│           │          │          │          │          │                    │
│     ┌─────▼──┐ ┌─────▼──┐ ┌────▼───┐ ┌───▼────┐ ┌──▼──────┐           │
│     │Product │ │  Cart  │ │ Order  │ │Payment │ │  User   │           │
│     │Service │ │Service │ │Service │ │Service │ │ Service │           │
│     └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └────────┘           │
│         │          │          │          │                              │
│   ┌─────▼──┐  ┌───▼────┐  ┌──▼─────┐  ┌▼────────┐                    │
│   │Search  │  │ Redis  │  │Inventory│  │Payment  │                    │
│   │(Elastic│  │ (cart  │  │Service  │  │Gateway  │                    │
│   │search) │  │  store)│  │         │  │(Stripe) │                    │
│   └────────┘  └────────┘  └─────────┘  └─────────┘                    │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────────┐   │
│  │  Event Bus (Kafka) — order events, inventory updates, analytics    │   │
│  └───────────────────────────────────────────────────────────────────┘   │
│                                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                   │
│  │Recommendation│  │  Notification │  │  Analytics   │                   │
│  │   Engine     │  │   Service     │  │  (orders,    │                   │
│  │ (ML models)  │  │ (email, SMS,  │  │   revenue)   │                   │
│  │              │  │  push)        │  │              │                   │
│  └──────────────┘  └──────────────┘  └──────────────┘                   │
└──────────────────────────────────────────────────────────────────────────┘

MICROSERVICES DECOMPOSITION:
  Each service:
  • Owns its data (no shared databases!)
  • Communicates via API (sync) or Events (async)
  • Can be scaled independently
  • Can be deployed independently
  • Has its own database (polyglot persistence!)
```

---

## 🔍 Product Catalog & Search

```
CHALLENGE: 350M products, search in < 200ms!

CATALOG SERVICE:
  Database: PostgreSQL (primary) + Elasticsearch (search!)
  
  Write path: Seller adds product →
    1. Save to PostgreSQL (source of truth!)
    2. CDC (Debezium) → Kafka → Elasticsearch indexer
    3. Elasticsearch index updated (near real-time!)
    4. CDN cache invalidated for product page!

SEARCH ARCHITECTURE:
  ┌────────────────────────────────────────────────────────────────┐
  │  User: "wireless noise canceling headphones under $200"        │
  │                                                                 │
  │  1. Query parsing: extract keywords + filters                   │
  │     keywords: "wireless noise canceling headphones"             │
  │     filters: price < 200                                        │
  │                                                                 │
  │  2. Elasticsearch query:                                        │
  │     {                                                           │
  │       "query": { "bool": {                                      │
  │         "must": {"multi_match": {"query": "wireless...",       │
  │                  "fields": ["title^3", "description"]}},       │
  │         "filter": {"range": {"price": {"lte": 200}}}           │
  │       }},                                                       │
  │       "sort": [{"_score": "desc"}, {"sales_rank": "asc"}]     │
  │     }                                                           │
  │                                                                 │
  │  3. Results: ranked by relevance + popularity + personalization │
  │                                                                 │
  │  4. Augment: add real-time price, availability, Prime badge!    │
  └────────────────────────────────────────────────────────────────┘

PRODUCT PAGE DATA:
  Product detail page needs data from MULTIPLE services:
  • Product info: catalog service
  • Price: pricing service (dynamic pricing, deals!)
  • Availability: inventory service
  • Reviews: reviews service (avg rating, count)
  • Recommendations: recommendation service
  
  Pattern: API composition at gateway OR GraphQL!
  
  Cache strategy:
  • Product details: CDN (5 min TTL) — rarely changes!
  • Price: short cache (1 min) — can change during sales!
  • Availability: NO CACHE! (real-time from inventory!)
  • Reviews: cache (5 min) — acceptable staleness!
```

---

## 🛒 Shopping Cart

```
REQUIREMENTS:
  • Persist across sessions (user logs out → comes back → cart intact!)
  • Real-time price/availability updates
  • Handle out-of-stock gracefully
  • Support guest cart → merge on login!
  • Handle concurrency (two tabs, same user, same item!)

STORAGE:
  Logged-in users: Redis (fast!) + periodic PostgreSQL backup
  Guest users: Redis with session ID + cookie!
  
  Redis data model:
  HSET cart:user42 "product:123" '{"qty":2,"addedAt":1700000}'
  HSET cart:user42 "product:456" '{"qty":1,"addedAt":1700001}'
  
  Why Redis?
  • Sub-millisecond reads (cart shown on every page!)
  • Atomic operations (HINCRBY for quantity!)
  • TTL for guest carts (expire after 30 days)
  • Pub/Sub for multi-device sync!

CART MERGE (guest → logged-in):
  1. User browses as guest, adds items to guest cart
  2. User logs in
  3. Merge guest cart INTO user cart:
     • Same item in both? → keep higher quantity!
     • Different items? → union!
  4. Delete guest cart

PRICE/AVAILABILITY VALIDATION:
  Cart shows items added days ago — prices may have changed!
  
  On cart view:
  • Fetch current price for each item (pricing service)
  • Check availability (inventory service)  
  • Show: "Price dropped from $50 to $40!" or "Only 2 left!"
  • If out of stock: show warning, offer "Save for Later"
```

---

## 📦 Order Processing

```
ORDER LIFECYCLE (State Machine):
  
  ┌────────┐    ┌────────┐    ┌─────────┐    ┌────────┐
  │PENDING │───►│CONFIRMED│───►│PROCESSING│───►│SHIPPED │
  └────────┘    └────────┘    └─────────┘    └───┬────┘
       │                                          │
       │                                          ▼
       │                                    ┌─────────┐
       └──── (payment fail) ──► CANCELLED   │DELIVERED│
                                             └─────────┘

CHECKOUT FLOW (critical path!):
  1. Validate cart (prices, availability — REAL-TIME check!)
  2. Reserve inventory (soft hold: 10 minutes!)
  3. Calculate totals (subtotal + tax + shipping)
  4. Process payment (charge card!)
  5. Confirm order (write to orders DB!)
  6. Trigger fulfillment (pick, pack, ship!)
  7. Send confirmation (email + push notification!)
  
  ANY step fails → compensating actions! (Saga pattern!)

SAGA PATTERN FOR ORDER:
  ┌───────────────────────────────────────────────────────────────┐
  │  Step              │  Action            │  Compensation        │
  ├───────────────────────────────────────────────────────────────┤
  │  1. Reserve stock  │  RESERVE 2 units   │  RELEASE 2 units    │
  │  2. Charge payment │  CHARGE $99        │  REFUND $99         │
  │  3. Create order   │  INSERT order      │  CANCEL order       │
  │  4. Update loyalty │  ADD 99 points     │  REMOVE 99 points   │
  └───────────────────────────────────────────────────────────────┘
  
  If step 2 fails (payment declined):
  → Run compensation for step 1 (release stock!)
  → Order NOT created! User sees: "Payment failed."

ORDER DATABASE:
  PostgreSQL (ACID for financial data!)
  Sharded by user_id (user's orders always on same shard!)
  
  Hot path: recent orders (last 90 days) → fast DB!
  Cold path: old orders → archival storage (S3 + Athena!)
```

---

## 📊 Inventory Management

```
THE HARDEST PROBLEM IN E-COMMERCE!

  Multiple warehouses + multiple sellers + flash sales =
  "How many of product X are ACTUALLY available right now?"

INVENTORY LEVELS:
  Physical stock: actual units in warehouse (counted!)
  Available stock: physical - reserved - damaged
  Reserved stock: held for pending orders (10 min timeout!)
  
  available = physical - reserved - damaged

RESERVATION SYSTEM:
  Problem: 100 people click "Buy" for last item simultaneously!
  
  Solution: Atomic reservation with Redis!
  
  INVENTORY_RESERVE (Lua script — atomic!):
  local available = redis.call('GET', 'inventory:product123')
  if tonumber(available) >= tonumber(ARGV[1]) then
    redis.call('DECRBY', 'inventory:product123', ARGV[1])
    return 1  -- SUCCESS!
  end
  return 0  -- OUT OF STOCK!

  If payment fails → release reservation!
  If 10 min timeout → release reservation!

MULTI-WAREHOUSE:
  Product X available at:
  • Warehouse NYC: 50 units
  • Warehouse LA: 30 units
  • Warehouse Chicago: 20 units
  
  User in NYC orders 1 unit → decrement NYC warehouse!
  
  What if NYC is out? → Check nearest warehouse with stock!
  → "Ships from LA" (longer delivery time!)

OVERSELLING PREVENTION:
  Never show "In Stock" if actually out!
  But DON'T block sales for slightly stale counts!
  
  Strategy: pessimistic for reservation (Redis atomic!),
  optimistic for display (cache with 1-min staleness OK!)
  
  Flash sale (100K people, 1000 units):
  → Queue-based: first 1000 in queue get the item!
  → Others: "Sold Out!" (fair, prevents oversell!)
```

---

## 💳 Payment System

```
PAYMENT FLOW:
  ┌──────────────────────────────────────────────────────────────┐
  │  Order Service → Payment Service → Payment Gateway (Stripe)  │
  │                                                               │
  │  1. AUTHORIZE: Can this card be charged $99?                  │
  │     → Gateway validates card, holds funds                     │
  │     → Returns authorization token!                            │
  │                                                               │
  │  2. CAPTURE: Actually charge the $99!                         │
  │     → Done AFTER inventory confirmed + order created!         │
  │     → If fulfillment fails: don't capture → auth expires!    │
  │                                                               │
  │  3. (Optional) REFUND: Customer returns item                  │
  │     → Reverse the charge (full or partial!)                   │
  └──────────────────────────────────────────────────────────────┘

WHY AUTHORIZE THEN CAPTURE (2-step)?
  • Auth: verify funds exist (hold them!)
  • Capture: actually take the money
  • If order can't be fulfilled → release auth (no charge!)
  • Protects customer: not charged for undeliverable orders!

IDEMPOTENCY (critical for payments!):
  Network timeout during payment → did it charge or not?!
  
  Solution: Idempotency key!
  POST /payments (Idempotency-Key: "order-123-attempt-1")
  
  Payment service: 
  • Check if this key was already processed!
  • YES → return same result (don't double-charge!)
  • NO → process payment, store result keyed by idempotency key!

MULTIPLE PAYMENT METHODS:
  Strategy pattern! (credit card, debit, PayPal, gift card, crypto)
  
  Split payment: $50 from gift card + $49 from credit card!
  → Two authorization + two captures → BOTH must succeed!
  → If one fails → cancel the other (compensating transaction!)
```

---

## 🧠 Recommendations

```
"CUSTOMERS WHO BOUGHT THIS ALSO BOUGHT..."

APPROACHES:
  1. Collaborative Filtering:
     User A bought {shoes, belt, watch}
     User B bought {shoes, belt, ???}
     Recommend: watch! (similar users → similar tastes!)
  
  2. Content-Based:
     User browsed "running shoes" → recommend other running shoes!
     Based on product attributes (category, brand, features)
  
  3. Real-time signals:
     User viewed product X → show related products immediately!
     "Frequently bought together" (basket analysis!)

ARCHITECTURE:
  ┌────────────────────────────────────────────────────────────┐
  │  Offline pipeline (batch — daily):                          │
  │  User behavior logs → Spark → Train ML models              │
  │  → Pre-compute "similar items" for all products!           │
  │  → Store in Redis: "similar:product123" → [456, 789, ...]  │
  │                                                             │
  │  Online (real-time):                                        │
  │  User browsing → current session items → real-time model    │
  │  → Personalized ranking of candidates!                      │
  │  → Serve in < 100ms!                                        │
  └────────────────────────────────────────────────────────────┘

  Pre-computed: "Similar items" (shown on product page)
  Real-time: "Recommended for you" (personalized homepage!)
```

---

## 💻 Java Implementation

### Order Service with Saga

```java
@Service
public class OrderService {
    
    @Autowired private InventoryClient inventoryClient;
    @Autowired private PaymentClient paymentClient;
    @Autowired private OrderRepository orderRepo;
    @Autowired private KafkaTemplate<String, OrderEvent> kafka;
    
    /**
     * Place order using Saga pattern (orchestration).
     * Each step has a compensating action on failure!
     */
    @Transactional
    public Order placeOrder(OrderRequest request) {
        Order order = createPendingOrder(request);
        
        try {
            // Step 1: Reserve inventory
            ReservationResult reservation = inventoryClient
                .reserve(request.getItems());
            
            if (!reservation.isSuccess()) {
                order.setStatus(OrderStatus.FAILED);
                order.setFailureReason("Items out of stock");
                orderRepo.save(order);
                return order;
            }
            
            // Step 2: Process payment
            PaymentResult payment;
            try {
                payment = paymentClient.charge(
                    request.getPaymentMethod(),
                    order.getTotalAmount(),
                    order.getId()); // Idempotency key!
            } catch (PaymentException e) {
                // COMPENSATE Step 1: release inventory!
                inventoryClient.release(reservation.getReservationId());
                order.setStatus(OrderStatus.PAYMENT_FAILED);
                orderRepo.save(order);
                return order;
            }
            
            // Step 3: Confirm order!
            order.setStatus(OrderStatus.CONFIRMED);
            order.setPaymentId(payment.getTransactionId());
            orderRepo.save(order);
            
            // Publish event for downstream services!
            kafka.send("order-events", order.getId(),
                new OrderEvent("CONFIRMED", order));
            
            return order;
            
        } catch (Exception e) {
            // Unexpected failure: compensate everything!
            order.setStatus(OrderStatus.FAILED);
            orderRepo.save(order);
            throw new OrderFailedException("Order placement failed", e);
        }
    }
}
```

### Inventory Service with Atomic Reservation

```java
@Service
public class InventoryService {
    
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private InventoryRepository inventoryRepo;
    
    private static final String RESERVE_SCRIPT = """
        local available = tonumber(redis.call('GET', KEYS[1]))
        local requested = tonumber(ARGV[1])
        if available and available >= requested then
            redis.call('DECRBY', KEYS[1], requested)
            return 1
        end
        return 0
        """;
    
    /**
     * Atomically reserve inventory for order items.
     * Uses Redis Lua script for atomic check-and-decrement!
     */
    public ReservationResult reserve(List<OrderItem> items) {
        List<String> reservedItems = new ArrayList<>();
        
        for (OrderItem item : items) {
            String key = "inventory:" + item.getProductId();
            
            Long result = redis.execute(
                RedisScript.of(RESERVE_SCRIPT, Long.class),
                List.of(key),
                String.valueOf(item.getQuantity()));
            
            if (result != null && result == 1) {
                reservedItems.add(item.getProductId());
            } else {
                // Out of stock! Release everything reserved so far!
                rollbackReservations(reservedItems, items);
                return ReservationResult.failed(item.getProductId());
            }
        }
        
        // Set expiry for reservation (10 min timeout!)
        String reservationId = UUID.randomUUID().toString();
        scheduleReservationExpiry(reservationId, items, 
            Duration.ofMinutes(10));
        
        return ReservationResult.success(reservationId);
    }
    
    /**
     * Release reservation (payment failed or timeout!)
     */
    public void release(String reservationId) {
        List<OrderItem> items = getReservationItems(reservationId);
        for (OrderItem item : items) {
            redis.opsForValue().increment(
                "inventory:" + item.getProductId(), 
                item.getQuantity());
        }
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you handle inventory for flash sales (100K users, 100 items)?**
> Queue-based approach: (1) User clicks "Buy" → request enters a FIFO queue, (2) Worker processes queue sequentially — first 100 succeed, rest get "Sold Out!", (3) Use Redis atomic DECR to track remaining count (O(1), no race conditions!), (4) Virtual waiting room: show queue position to users, (5) Pre-warm everything: no cold caches during flash sale! Key insight: don't even TRY to handle 100K concurrent DB writes — funnel through a queue and process serially. It's fast enough (Redis: 100K ops/sec = done in 1 second!).

**Q2: How do you ensure the checkout process is reliable?**
> Saga pattern with compensating transactions! Each step is idempotent and reversible: reserve inventory → charge payment → confirm order. If any step fails: run compensations in reverse order. Payment uses idempotency keys (retry-safe!). Orders use state machine (PENDING → CONFIRMED → SHIPPED). Exactly-once semantics: unique order ID + idempotent operations. For catastrophic failures: reconciliation job runs hourly, matches inventory holds with confirmed orders, releases orphaned reservations.

**Q3: How would you design the product search to handle 350M products?**
> Elasticsearch cluster: sharded by category (or hash-based). Index structure optimized for: (1) Full-text search (product title, description), (2) Faceted filtering (category, brand, price range, rating), (3) Personalized ranking (boost items based on user history). Write path: product updates flow via CDC → Kafka → ES indexer (near real-time!). Read path: query ES → merge with real-time data (price, availability from services) → return. Cache: frequent searches cached at CDN level (popular queries like "iPhone" rarely change results!).

**Q4: How do you handle the "Buy Now" button across multiple warehouses?**
> (1) Product shows "In Stock" if ANY warehouse has units, (2) On checkout: routing algorithm picks optimal warehouse (nearest with stock → lowest shipping cost!), (3) If nearest is out: check next nearest (fallback chain!), (4) Split shipment: order has items from different warehouses → ship separately (notify customer!), (5) Inventory aggregation: Redis stores per-warehouse counts AND total available. The "In Stock" display uses total (slightly stale, cached), but reservation targets specific warehouse (real-time, atomic!).

---

## 🔗 Related Topics
- [CQRS](../Architectures/CQRS.md) — Separating read/write models
- [Event-Driven Architecture](../Architectures/Event_Driven.md) — Async processing
- [Distributed Locking](../Microservices/DistributedLocking.md) — Inventory reservation
- [CDC](../MessagingQ/CDC.md) — Keeping search index in sync
- [Design Ticket Booking](DesignTicketBooking.md) — Similar reservation pattern

---

*"Amazon's entire architecture philosophy boils down to: each team owns a service, each service owns its data, services communicate through well-defined APIs. This organizational structure (two-pizza teams) directly maps to the technical architecture (microservices). Conway's Law in action." — Werner Vogels, CTO Amazon* 🛒
