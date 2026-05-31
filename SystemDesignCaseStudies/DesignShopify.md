# 🛒 Design Shopify (E-Commerce Platform)

> *"Shopify isn't just an online store — it's a platform that lets ANYONE build an online store. It powers 4.4 million stores worldwide, handling Black Friday traffic spikes of 80× normal load, processing $200B+ in sales, and supporting everything from a single-person candle business to Kylie Cosmetics doing $420M in a weekend. The system design challenge: multi-tenancy at scale, where one tenant's viral product launch shouldn't impact other stores, and the platform must handle wildly different traffic patterns — from 0 orders/day to 10,000 orders/minute during flash sales."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Microservices](../Microservices/), [Caching](../BuildingBlocks/Caching.md), [Rate Limiting](../BuildingBlocks/RateLimiting.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Multi-Tenancy Design](#-multi-tenancy-design)
4. [Product Catalog & Storefront](#-product-catalog--storefront)
5. [Order Processing & Checkout](#-order-processing--checkout)
6. [Handling Flash Sales (Spiky Traffic)](#-handling-flash-sales-spiky-traffic)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Merchants create/manage online stores (themes, products, pages!)
  • Product catalog with variants (size, color, material!)
  • Shopping cart + checkout flow
  • Payment processing (multiple gateways: Stripe, PayPal, etc.)
  • Inventory management (real-time stock tracking!)
  • Order management (create, fulfill, ship, refund!)
  • Custom domains (mystore.com → Shopify-hosted!)
  • App ecosystem (3rd party plugins!)
  • Analytics dashboard for merchants
  • Multi-currency, multi-language!

NON-FUNCTIONAL:
  • 4M+ stores (tenants!)
  • 99.99% availability (merchants DEPEND on this for income!)
  • Black Friday: 80× normal traffic!
  • Checkout latency: < 3 seconds (any slower = abandoned carts!)
  • Tenant isolation: one store's spike can't affect others!
  • Global: serve from nearest region!

SCALE:
  • 500K orders per minute during peak (Black Friday!)
  • 50B+ API calls per day
  • 100M+ products across all stores
  • 1B+ monthly storefront visits
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    SHOPIFY PLATFORM ARCHITECTURE                             │
│                                                                             │
│  Shoppers            ┌───────────────────────────────────────────┐         │
│  ┌────────┐         │              EDGE LAYER                     │         │
│  │Buyer at│────────►│  • CDN (static assets, product images!)     │         │
│  │Store A │         │  • Edge workers (storefront rendering!)     │         │
│  └────────┘         │  • DDoS protection, rate limiting!         │         │
│  ┌────────┐         │  • Custom domain → tenant routing!          │         │
│  │Buyer at│────────►│                                             │         │
│  │Store B │         └───────────────────┬─────────────────────────┘         │
│  └────────┘                             │                                   │
│                                         ▼                                   │
│  Merchants          ┌───────────────────────────────────────────┐          │
│  ┌────────┐         │           APPLICATION LAYER                 │          │
│  │Merchant│────────►│                                             │          │
│  │Admin   │         │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │          │
│  └────────┘         │  │Storefront│ │  Admin   │ │ Checkout │  │          │
│                      │  │ Service  │ │  API     │ │ Service  │  │          │
│                      │  └──────────┘ └──────────┘ └──────────┘  │          │
│                      │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │          │
│                      │  │ Product  │ │  Order   │ │ Inventory│  │          │
│                      │  │ Service  │ │  Service │ │ Service  │  │          │
│                      │  └──────────┘ └──────────┘ └──────────┘  │          │
│                      │  ┌──────────┐ ┌──────────┐ ┌──────────┐  │          │
│                      │  │ Payment  │ │ Shipping │ │  App     │  │          │
│                      │  │ Service  │ │  Service │ │ Platform │  │          │
│                      │  └──────────┘ └──────────┘ └──────────┘  │          │
│                      └───────────────────────────────────────────┘          │
│                                                                             │
│                      ┌───────────────────────────────────────────┐          │
│                      │            DATA LAYER                      │          │
│                      │  ┌─────────┐ ┌─────────┐ ┌─────────────┐│          │
│                      │  │  MySQL  │ │  Redis  │ │ Kafka       ││          │
│                      │  │(sharded │ │(cache,  │ │(events,     ││          │
│                      │  │ by shop)│ │ sessions)│ │ webhooks)   ││          │
│                      │  └─────────┘ └─────────┘ └─────────────┘│          │
│                      └───────────────────────────────────────────┘          │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🏠 Multi-Tenancy Design

```
MULTI-TENANCY APPROACHES:

OPTION 1: Shared Everything (one DB for all!)
  ┌─────────────────────────────────────────┐
  │  products table: shop_id + product_id   │
  │  Every query: WHERE shop_id = ?         │
  │  ✅ Simple, cost-effective              │
  │  ❌ Noisy neighbor! One store can hog DB│
  │  ❌ Hard to scale individual stores     │
  └─────────────────────────────────────────┘

OPTION 2: Database-per-Tenant (full isolation!)
  ┌─────────────────────────────────────────┐
  │  Store A → database_store_a             │
  │  Store B → database_store_b             │
  │  ✅ Full isolation, easy to migrate     │
  │  ❌ 4M databases? Operational nightmare!│
  │  ❌ Connection pool exhaustion!         │
  └─────────────────────────────────────────┘

OPTION 3: SHOPIFY'S APPROACH (Shard-per-Pod!)
  ┌─────────────────────────────────────────────────────────────────┐
  │  "Pods" — groups of ~5000 stores sharing infrastructure!        │
  │                                                                  │
  │  Pod 1: MySQL shard + Redis + App servers                       │
  │    └── Stores 1-5000                                            │
  │  Pod 2: MySQL shard + Redis + App servers                       │
  │    └── Stores 5001-10000                                        │
  │  Pod N: ...                                                      │
  │                                                                  │
  │  ✅ Isolation: one pod's overload doesn't affect other pods!    │
  │  ✅ Scalable: add more pods for more stores!                    │
  │  ✅ Migrateable: move hot store to its own pod!                 │
  │  ✅ Manageable: ~1000 pods (not 4M databases!)                 │
  │                                                                  │
  │  Routing: shop_id → pod_id (lookup in routing table!)           │
  │  Custom domain → shop_id → pod_id (DNS + routing layer!)       │
  └─────────────────────────────────────────────────────────────────┘

NOISY NEIGHBOR PREVENTION:
  • Per-store rate limiting (API calls, bandwidth!)
  • Per-store query quotas (prevent expensive queries!)
  • Auto-detect hot stores → migrate to dedicated pod!
  • Circuit breaker: if one store overwhelms pod → throttle it!
```

---

## 🛍️ Product Catalog & Storefront

```
STOREFRONT RENDERING (must be FAST!):

  ┌────────────────────────────────────────────────────────────────┐
  │  APPROACH: Server-Side Rendering at the Edge!                   │
  │                                                                 │
  │  1. Shopper visits mystore.com/products/blue-widget            │
  │  2. Edge worker (Cloudflare/Fastly!) intercepts                │
  │  3. Check edge cache: full HTML page cached? (80% hit!)        │
  │     YES → return immediately! (< 50ms!)                        │
  │     NO → render at edge or proxy to origin!                    │
  │                                                                 │
  │  4. Render: Template (Liquid!) + Product data + Theme           │
  │  5. Cache rendered page (TTL: 5 min for product pages!)        │
  │  6. Dynamic parts (cart count, user name): loaded via JS!      │
  └────────────────────────────────────────────────────────────────┘

PRODUCT SCHEMA:
  products:
    id, shop_id, title, description, handle(slug),
    product_type, vendor, tags[], status, created_at
    
  variants (each product has 1-100 variants!):
    id, product_id, title, price, compare_at_price,
    sku, barcode, inventory_quantity, weight,
    option1(size), option2(color), option3(material)
  
  images:
    id, product_id, src(CDN URL!), alt_text, position

INVENTORY TRACKING:
  • Real-time: decrement on order, increment on restock/return
  • Multi-location: warehouse A has 50, warehouse B has 30
  • Reserve on add-to-cart? NO! (abandoned carts would lock inventory!)
  • Reserve on checkout start? YES! (10 min hold!)
  • If out of stock during checkout → show error, remove from cart!
```

---

## 💰 Order Processing & Checkout

```
CHECKOUT FLOW (most critical path!):

  ┌────────────────────────────────────────────────────────────────┐
  │  1. CART → CHECKOUT                                             │
  │     • Validate all items still in stock!                        │
  │     • Calculate pricing (discounts, taxes, shipping!)           │
  │     • Create "draft order" (not yet committed!)                │
  │                                                                 │
  │  2. CUSTOMER INFO                                               │
  │     • Email, shipping address, billing address                  │
  │     • Calculate shipping rates (carrier APIs!)                  │
  │     • Apply discount codes (validate!)                          │
  │                                                                 │
  │  3. PAYMENT                                                     │
  │     • Authorize payment (hold funds, don't capture yet!)        │
  │     • If auth fails → show error, retry!                       │
  │                                                                 │
  │  4. PLACE ORDER (the critical transaction!):                    │
  │     BEGIN TRANSACTION:                                           │
  │       • Decrement inventory (CHECK quantity > 0!)               │
  │       • Create order record (status: confirmed!)                │
  │       • Capture payment (actually charge card!)                 │
  │     COMMIT!                                                      │
  │                                                                 │
  │     If inventory check fails → abort → refund auth!            │
  │     If payment capture fails → abort → restore inventory!      │
  │                                                                 │
  │  5. POST-ORDER (async!):                                        │
  │     • Send confirmation email                                   │
  │     • Update analytics                                          │
  │     • Trigger fulfillment workflow                              │
  │     • Fire webhooks to merchant's apps!                        │
  └────────────────────────────────────────────────────────────────┘
```

---

## ⚡ Handling Flash Sales (Spiky Traffic)

```
FLASH SALE: 0 → 100K requests/second in 1 minute! 💥

  Problem: store normally gets 10 req/min. Merchant tweets "50% OFF!"
  → 100K people flood the store simultaneously!

SHOPIFY'S STRATEGIES:
  ┌────────────────────────────────────────────────────────────────┐
  │  1. QUEUE-BASED CHECKOUT (Shopify Queue-It!):                   │
  │     Too many checkouts → overflow into virtual queue!            │
  │     "You're #4,521 in line! Estimated wait: 8 min"             │
  │     Process checkouts at sustainable rate!                       │
  │     ✅ No crashes! ❌ Some users wait (but fair!)              │
  │                                                                 │
  │  2. EDGE CACHING (absorb read traffic!):                       │
  │     Product pages: cached at CDN (most traffic = browsing!)     │
  │     Only checkout = write traffic → MUCH smaller load!          │
  │     Flash sale: 100K browsers but only 10K checkouts!           │
  │                                                                 │
  │  3. AUTO-SCALING:                                               │
  │     Pre-warm: merchant warns "sale at 9 AM" → scale UP before!  │
  │     Reactive: auto-scale on request rate spike!                  │
  │     Pod-level: only the affected pod scales!                    │
  │                                                                 │
  │  4. INVENTORY RESERVATION:                                      │
  │     Don't oversell! Atomic decrement:                           │
  │     UPDATE variants SET inventory = inventory - 1               │
  │     WHERE id = ? AND inventory > 0;                             │
  │     If affected_rows = 0 → SOLD OUT! (atomic, no race!)        │
  │                                                                 │
  │  5. CIRCUIT BREAKERS:                                           │
  │     If payment gateway slow → timeout + fallback!               │
  │     If shipping API down → use cached rates!                    │
  │     If webhooks backed up → queue (don't block checkout!)      │
  └────────────────────────────────────────────────────────────────┘
```

---

## 💻 Java Implementation

### Checkout Service

```java
@Service
public class CheckoutService {
    
    @Autowired private InventoryService inventoryService;
    @Autowired private PaymentGateway paymentGateway;
    @Autowired private OrderRepository orderRepo;
    @Autowired private KafkaTemplate<String, OrderEvent> kafka;
    
    /**
     * Process checkout — the most critical path!
     * Must be: fast, atomic, correct!
     */
    @Transactional(isolation = Isolation.READ_COMMITTED)
    public CheckoutResult processCheckout(CheckoutRequest request) {
        String shopId = request.getShopId();
        
        // 1. Validate cart items still available!
        for (CartItem item : request.getItems()) {
            int available = inventoryService.getAvailable(
                shopId, item.getVariantId());
            if (available < item.getQuantity()) {
                return CheckoutResult.outOfStock(item.getVariantId(), available);
            }
        }
        
        // 2. Calculate totals (with discounts, taxes, shipping!)
        OrderTotals totals = calculateTotals(request);
        
        // 3. Authorize payment (hold funds!)
        PaymentAuth auth = paymentGateway.authorize(
            request.getPaymentMethod(), totals.getTotal());
        if (!auth.isSuccess()) {
            return CheckoutResult.paymentFailed(auth.getError());
        }
        
        try {
            // 4. Reserve inventory (atomic decrement!)
            for (CartItem item : request.getItems()) {
                boolean reserved = inventoryService.decrementAtomic(
                    shopId, item.getVariantId(), item.getQuantity());
                if (!reserved) {
                    // Race condition: someone bought last item!
                    paymentGateway.voidAuth(auth.getTransactionId());
                    return CheckoutResult.outOfStock(item.getVariantId(), 0);
                }
            }
            
            // 5. Create order!
            Order order = Order.builder()
                .id(generateOrderId(shopId))
                .shopId(shopId)
                .customerId(request.getCustomerId())
                .items(request.getItems())
                .totals(totals)
                .paymentId(auth.getTransactionId())
                .status(OrderStatus.CONFIRMED)
                .createdAt(Instant.now())
                .build();
            orderRepo.save(order);
            
            // 6. Capture payment (actually charge!)
            paymentGateway.capture(auth.getTransactionId());
            
            // 7. Async: post-order processing!
            kafka.send("order-events", shopId, OrderEvent.created(order));
            
            return CheckoutResult.success(order);
            
        } catch (Exception e) {
            // COMPENSATE: void payment if anything fails!
            paymentGateway.voidAuth(auth.getTransactionId());
            throw new CheckoutException("Checkout failed", e);
        }
    }
}
```

### Inventory Service (Atomic Decrement)

```java
@Service
public class InventoryService {
    
    @Autowired private JdbcTemplate jdbc;
    @Autowired private RedisTemplate<String, String> redis;
    
    /**
     * Atomic inventory decrement — prevents overselling!
     * Returns true if decrement succeeded, false if insufficient stock.
     */
    public boolean decrementAtomic(String shopId, String variantId, 
                                    int quantity) {
        // Atomic: only decrements if enough inventory!
        int affected = jdbc.update("""
            UPDATE variants 
            SET inventory_quantity = inventory_quantity - ? 
            WHERE id = ? AND shop_id = ? AND inventory_quantity >= ?
            """, quantity, variantId, shopId, quantity);
        
        if (affected > 0) {
            // Update Redis cache (for fast reads!)
            redis.opsForHash().increment(
                "inventory:" + shopId, variantId, -quantity);
            return true;
        }
        return false; // Not enough stock!
    }
    
    /**
     * Fast inventory read (from Redis cache!).
     * Cache refreshed every 5 seconds from DB.
     */
    public int getAvailable(String shopId, String variantId) {
        String cached = (String) redis.opsForHash().get(
            "inventory:" + shopId, variantId);
        if (cached != null) {
            return Integer.parseInt(cached);
        }
        // Cache miss → query DB and cache!
        int qty = jdbc.queryForObject(
            "SELECT inventory_quantity FROM variants WHERE id = ? AND shop_id = ?",
            Integer.class, variantId, shopId);
        redis.opsForHash().put("inventory:" + shopId, variantId, String.valueOf(qty));
        return qty;
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you handle the "noisy neighbor" problem in a multi-tenant platform?**
> Shopify's pod architecture: group ~5000 stores per pod (shared DB shard + app servers). Noisy neighbor can only affect their pod, not all 4M stores! Additional protection: (1) Per-store rate limiting at API gateway (even within a pod!), (2) Query timeout per store (kill expensive queries after 2s!), (3) Auto-detect hot stores (high CPU/traffic) → migrate to dedicated pod (more resources!), (4) Traffic shedding: if pod overwhelmed, queue excess requests (Shopify Queue). The key insight: full isolation (DB per store) is too expensive at 4M stores. Pod isolation gives 99% of the benefit at manageable cost!

**Q2: How do you prevent overselling during a flash sale?**
> Atomic database operation: `UPDATE SET inventory = inventory - 1 WHERE inventory > 0`. This is a single atomic operation (no read-then-write race condition!). If affected_rows = 0 → out of stock. This handles 10K concurrent checkouts correctly! For extreme scale: use Redis as the source of truth for inventory DURING the flash sale. Redis DECRBY is single-threaded → naturally serialized → no races! After sale: reconcile Redis count back to MySQL. Also: DON'T reserve on add-to-cart (abandoned carts would lock inventory). Reserve ONLY at checkout submission!

**Q3: How would you design the checkout to survive payment gateway outages?**
> (1) Multi-gateway failover: if Stripe is down → try PayPal → try Adyen! (2) Circuit breaker: after 3 failures → stop trying that gateway (don't waste time + avoid cascading!). (3) Timeout: if gateway doesn't respond in 5s → abort and try backup! (4) Idempotency key: if request timed out but payment actually went through → use idempotency key to check status without double-charging! (5) Async capture: authorize immediately, capture asynchronously (more reliable!). (6) If ALL gateways down: allow "pay later" option (send payment link via email!).

**Q4: How does Shopify handle Black Friday (80× normal traffic)?**
> Preparation starts months ahead! (1) **Load testing**: simulate Black Friday traffic, identify bottlenecks. (2) **Pre-scaling**: known large sales (e.g., Kylie Cosmetics) → dedicated infrastructure pre-provisioned! (3) **Edge absorption**: product pages served from CDN (90% of traffic is browsing!). Only 10% reaches origin (checkouts). (4) **Queue system**: when checkouts exceed capacity → virtual queue (fair, prevents crashes!). (5) **Feature flags**: disable non-essential features during peak (analytics, recommendations). (6) **Database read replicas**: read traffic → replicas, only writes → primary. Result: Shopify handles 80× spikes without downtime!

---

## 🔗 Related Topics
- [Design Amazon E-Commerce](./DesignAmazonEcommerce.md) — Similar checkout patterns
- [Rate Limiting](../BuildingBlocks/RateLimiting.md) — Tenant isolation
- [Caching Strategies](../BuildingBlocks/CachingStrategies.md) — Storefront performance
- [Circuit Breaker](../BuildingBlocks/CircuitBreaker.md) — Payment resilience

---

*"Shopify's genius isn't in any single technical innovation — it's in making infrastructure DISAPPEAR for merchants. A seller shouldn't have to think about CDNs, database sharding, or flash sale queues. They just click 'create store' and trust that when their product goes viral, the platform handles the rest. That trust is the product — built on millions of engineering hours."* 🛒
