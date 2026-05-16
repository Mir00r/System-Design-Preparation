# 🌿 Strangler Fig Pattern: Migrating from Monolith to Microservices Safely

> *"Named after the strangler fig tree that grows around a host tree until it completely replaces it — without ever killing the host. You build new functionality as microservices that gradually replace the monolith, until the monolith can be safely removed."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Microservices](./DesignEmployeeService.md), [API Gateway](../BuildingBlocks/APIGateway.md)

---

## 🤔 What is the Strangler Fig Pattern

```
THE PROBLEM:
  You have a large monolith that's hard to maintain, scale, and deploy.
  "Big bang rewrite" fails 70%+ of the time (Joel Spolsky's worst mistake).
  You need to migrate INCREMENTALLY while keeping the system running.

THE PATTERN:

  Phase 1: Identify a boundary module in the monolith
  Phase 2: Build it as a new microservice
  Phase 3: Route traffic to new service (old monolith still handles the rest)
  Phase 4: Repeat for next module
  Phase 5: Eventually, monolith has nothing left → decommission

VISUAL:
  ┌──────────────────────────────────────────────────────────────────┐
  │ BEFORE: Everything goes through the monolith                     │
  │                                                                  │
  │  [Users] → [Monolith: Auth + Orders + Products + Payments]       │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │ DURING: Strangler facade routes some traffic to new services     │
  │                                                                  │
  │  [Users] → [API Gateway / Proxy] ─┬─▶ [New: Payment Service]   │
  │                                    ├─▶ [New: Product Service]   │
  │                                    └─▶ [Monolith: Auth+Orders]  │
  └──────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────┐
  │ AFTER: All functionality migrated, monolith decommissioned       │
  │                                                                  │
  │  [Users] → [API Gateway] ─┬─▶ [Payment Service]                │
  │                            ├─▶ [Product Service]                │
  │                            ├─▶ [Order Service]                  │
  │                            └─▶ [Auth Service]                   │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 📋 Step-by-Step Migration

```
STEP 1: ADD A PROXY/FACADE (the "strangler")
  Place an API gateway / reverse proxy in front of the monolith
  Initially: ALL traffic still goes to monolith (transparent proxy)
  
  [Client] → [Nginx/API Gateway] → [Monolith]

STEP 2: IDENTIFY EXTRACTION CANDIDATE
  Choose a module to extract first. Good candidates:
  ✅ Well-bounded domain (clear inputs/outputs)
  ✅ Few dependencies on other monolith modules
  ✅ High value (frequent changes, team bottleneck, scaling issue)
  ✅ Self-contained data (has its own tables/entities)
  
  Bad first candidates:
  ❌ Core entities everything depends on (User, Account)
  ❌ Heavily interleaved with other modules (shared transactions)

STEP 3: BUILD MICROSERVICE
  Implement the same functionality as a new service
  Own database (data migration strategy needed)
  Same API contract (backward compatible)

STEP 4: ROUTE TRAFFIC
  Update proxy rules:
    /api/payments/* → new Payment Service
    /api/* (everything else) → Monolith

STEP 5: VERIFY AND REMOVE OLD CODE
  Run both in parallel (shadow mode / canary)
  Compare responses
  Once confident → remove old code from monolith
  → Monolith is now smaller!

STEP 6: REPEAT
  Extract next module → shrink monolith further
  Continue until monolith is empty → decommission
```

---

## 🛠️ Implementation Strategies

```
STRATEGY 1: ROUTE BY URL PATH
  /api/v2/payments → new service
  /api/v2/* → monolith
  Simplest, works for API-level routing

STRATEGY 2: ROUTE BY FEATURE FLAG
  if (featureFlag("new-payment-service")) → route to new service
  else → route to monolith
  Gradual rollout: 5% → 25% → 50% → 100%

STRATEGY 3: ROUTE BY USER SEGMENT
  Internal users → new service (test in production)
  External users → monolith (safe)
  Then gradually: 10% external → 50% → 100%

DATA MIGRATION STRATEGIES:
  1. Shared database (temporary): both services read/write same DB
     ✅ Simple, no data sync needed
     ❌ Coupling, can't change schema independently
     
  2. Database per service + sync:
     New service has own DB
     CDC (Change Data Capture) syncs from monolith → new DB
     Eventually: monolith stops writing, new service is sole owner
     
  3. Event-driven migration:
     Monolith publishes events on writes
     New service consumes events and builds its own state
     Cut over: new service becomes the authority
```

---

## 💻 Code Example (API Gateway Routing)

```java
// Spring Cloud Gateway — gradual migration routing
@Configuration
public class GatewayConfig {
    
    @Bean
    public RouteLocator routes(RouteLocatorBuilder builder) {
        return builder.routes()
            // Extracted services (new microservices)
            .route("payment-service", r -> r
                .path("/api/payments/**")
                .uri("lb://payment-service"))
            
            .route("product-service", r -> r
                .path("/api/products/**")
                .uri("lb://product-service"))
            
            // Everything else → monolith (default fallback)
            .route("monolith-fallback", r -> r
                .path("/api/**")
                .uri("http://monolith:8080"))
            .build();
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Extracting the wrong service first** — Don't start with the most complex, deeply coupled module. Start with something simple and well-bounded to practice the migration process. Build migration muscle before tackling the hard parts.

2. **Shared database for too long** — A "temporary" shared database becomes permanent technical debt. Set a deadline for data ownership transfer. The shared DB phase should be weeks, not months.

3. **Not maintaining backward compatibility** — The monolith's API clients don't change overnight. New services must support the same API contracts initially. Use API versioning to evolve gradually.

4. **No rollback plan** — If the new service has issues, you need to route traffic back to the monolith instantly. Keep monolith code running (even if dormant) until the new service is proven stable for weeks.

---

## 🧩 Mini Challenge

**You have a monolith with 5 modules: Auth, Products, Orders, Payments, Notifications. Orders depends on Products and Payments. What extraction order would you recommend?**

<details>
<summary>💡 Click to reveal answer</summary>

```
RECOMMENDED EXTRACTION ORDER:

1. NOTIFICATIONS (first — lowest risk)
   - Least coupled to other modules
   - Async by nature (can use message queue)
   - If it fails, core business continues
   - Good practice for the team

2. PRODUCTS (second — well-bounded)
   - Read-heavy, clear domain
   - Other modules reference products by ID (loose coupling)
   - Good candidate for separate database
   - Can scale independently (catalog browsing is high-traffic)

3. PAYMENTS (third — clear boundary)
   - External gateway integration (naturally isolated)
   - Strong consistency requirements (own DB crucial)
   - Security benefits of isolation
   - Depends on nothing internal (only external payment gateway)

4. ORDERS (fourth — medium complexity)
   - Depends on Products and Payments (extract those first!)
   - Now Orders can call Product and Payment services via API
   - Core domain, complex logic, benefits from team ownership

5. AUTH (last — most coupled)
   - EVERYTHING depends on Auth (every request checks auth)
   - Highest risk if broken (users can't access anything)
   - Most complex to extract (session management, token validation)
   - By this point, team has experience with 4 extractions

GENERAL RULES:
  - Start with leaf nodes (things that don't depend on much)
  - Extract dependencies BEFORE dependents
  - Save the most critical/coupled modules for last
  - Each extraction gives the team more confidence
```

</details>

---

## 📝 Interview Q&A

**Q: Why is the Strangler Fig pattern preferred over a "big bang" rewrite?**
> A: Big bang rewrites fail because: (1) You're maintaining TWO systems simultaneously for months/years (old + new). (2) Business requirements keep changing — by the time you finish rewriting, the requirements moved. (3) No value delivered until rewrite is complete (could be 1-2 years). (4) High risk — one big switch with no gradual validation. Strangler Fig solves all of these: you deliver value incrementally (each extracted service is a win), validate in production gradually (canary routing), keep the monolith running until proven unnecessary, and can stop/pause the migration at any point without losing progress. Real-world: Amazon, Netflix, Spotify all used incremental migration, not big bang rewrites.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/APIGateway.md](../BuildingBlocks/APIGateway.md)** — The facade that enables strangler routing
2. **[Microservices/ServiceMesh.md](./ServiceMesh.md)** — Infrastructure for service communication
3. **[Architectures/Modular.md](../Architectures/Modular.md)** — Modular monolith as stepping stone

---

*[← Sidecar Pattern](./Sidecar_Pattern.md) | [Back to Index](../INDEX.md)*
