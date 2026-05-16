# 🚪 API Gateway: The Front Door to Your Microservices

> **"An API Gateway is like a hotel concierge — one point of contact who handles authentication, routing, translation, and knows exactly where everyone needs to go."**

---

## 🎯 What You'll Learn
- Why every microservices system needs an API Gateway
- Core responsibilities: routing, auth, rate limiting, transformation
- API Gateway vs Load Balancer vs Service Mesh — when to use which
- How to configure Kong, AWS API Gateway, and Traefik
- How Netflix, Uber, and Amazon use API Gateways at massive scale

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium  
**🔗 Prerequisites**: [Load Balancing](./LoadBalancing.md) | [Rate Limiting](./RateLimiting.md)  
**🔗 Related Topics**: [Service Discovery](./ServiceDiscovery.md) | [Circuit Breaker](./CircuitBreaker.md) | [Security/OAuth2](../Security/OAuth2.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [What Is an API Gateway?](#-what-is-an-api-gateway)
3. [Core Responsibilities](#-core-responsibilities)
4. [API Gateway vs Load Balancer vs Service Mesh](#-api-gateway-vs-load-balancer-vs-service-mesh)
5. [Request Lifecycle Through a Gateway](#-request-lifecycle-through-a-gateway)
6. [Authentication & Authorization at the Gateway](#-authentication--authorization-at-the-gateway)
7. [Code Examples](#-code-examples)
8. [Popular API Gateway Tools](#-popular-api-gateway-tools)
9. [Industry Examples](#-industry-examples)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)
13. [What to Read Next](#-what-to-read-next)

---

## 🤔 The Problem

You're building a mobile app. Your backend has grown into 15 microservices.

```
❌ WITHOUT API GATEWAY — Client calls each service directly:

Mobile App
  ├──► UserService:8001/api/users/{id}
  ├──► OrderService:8002/api/orders
  ├──► ProductService:8003/api/products
  ├──► CartService:8004/api/cart
  ├──► NotificationService:8005/api/notifications
  └──► RecommendationService:8006/api/recommend

PROBLEMS:
  - App must know addresses of ALL 15 services
  - Every service independently implements: auth, rate limiting, logging
  - Service addresses change when they scale/redeploy
  - Hard to do A/B testing or canary releases
  - No single place to enforce security policies
  - CORS, SSL termination repeated in every service
  - Mobile app makes 6 separate HTTP calls to render 1 page
```

Now add 100+ mobile developers, 3 external partners needing API access, and a security team that wants a single audit log...

```
✅ WITH API GATEWAY — Single entry point:

Mobile App
  └──► API Gateway (api.yourapp.com)
          │ (handles: auth, rate limiting, logging, SSL)
          ├──► UserService (internal)
          ├──► OrderService (internal)
          ├──► ProductService (internal)
          └──► [aggregated response]

SOLUTIONS:
  - One URL for clients to call
  - Auth, rate limiting, logging in one place
  - Service discovery abstracted away
  - Request aggregation (1 call → multiple services → 1 response)
  - Easy A/B testing and canary deployments
```

> 💡 **Key Insight**: An API Gateway is the **traffic cop, security checkpoint, and switchboard** of your microservices architecture — all in one.

---

## 💡 What Is an API Gateway?

An **API Gateway** is a server that acts as the single entry point for all clients into your microservices backend. It sits between the client and your services, handling cross-cutting concerns so individual services don't have to.

```
API GATEWAY POSITION IN ARCHITECTURE:

External World          Internal Network
─────────────         ────────────────────────────
                       
  Mobile App ──┐       ┌── User Service
  Web App ─────┼──►   ─┼── Order Service
  Partner ─────┘  API  ├── Product Service
  CLI ─────────►  Gate ├── Payment Service
                  way  ├── Notification Service
                       └── Recommendation Service
```

---

## 🌍 Real-World Analogy

```
HOTEL ANALOGY:

You check into a hotel.
You talk to ONE PERSON: the concierge.

You say: "I need restaurant reservation, car service, and wake-up call."

The concierge:
  1. Verifies you're a hotel guest (authentication)
  2. Checks your booking level (authorization) 
  3. Calls the restaurant (routes to service 1)
  4. Calls car service (routes to service 2)
  5. Schedules wake-up (routes to service 3)
  6. Reports back to you with one unified answer

You never talked to the restaurant directly.
You never knew car service was a separate company.
One point of contact. All cross-cutting concerns handled.

That's an API Gateway.
```

---

## 🏗️ Core Responsibilities

### 1. 🔀 Routing
```
/api/users/*    → UserService:8001
/api/orders/*   → OrderService:8002
/api/products/* → ProductService:8003

PATH REWRITING:
  Client: GET /api/v2/users/123
  Gateway: GET /users/123         (strips /api/v2 prefix)
  
HEADER-BASED ROUTING:
  X-Version: beta → Beta service instances
  X-Version: stable → Production instances
```

### 2. 🔐 Authentication & Authorization
```
Client sends: Authorization: Bearer <JWT>

Gateway:
  1. Validates JWT signature ✅
  2. Checks expiry ✅
  3. Extracts user_id and roles
  4. Passes user context to downstream service:
     X-User-ID: 123
     X-User-Role: admin
     
Downstream service: trusts these headers (no auth code needed!)
```

### 3. 🚦 Rate Limiting
```
Per client: 1000 req/min
Per endpoint: POST /auth: 5 req/min
Per tier: Free=100/min, Pro=1000/min
```

### 4. 📦 Request Aggregation (BFF Pattern)
```
Mobile app needs: user profile + orders + recommendations

WITHOUT AGGREGATION: 3 separate API calls from mobile
WITH AGGREGATION:   Gateway calls 3 services in parallel
                    Returns 1 combined response to mobile

Result: 3x fewer round trips, faster mobile experience
```

### 5. 📊 Observability
```
Every request logged: timestamp, path, user, latency, status
Distributed trace started: X-Trace-ID: abc-123
Metrics: request counts, p50/p95/p99 latency per endpoint
```

### 6. 🔄 Protocol Translation
```
Client speaks: HTTP/REST
Backend uses:  gRPC

Gateway: Translates REST → gRPC → REST
         Client never knows gRPC exists internally
```

### 7. ⚡ Caching
```
GET /api/products  → Same response for 5 minutes
Gateway caches it → Origin not hit for repeated calls
```

### 8. 🏥 Health & Circuit Breaking
```
ProductService is slow (500ms avg):
Gateway: starts circuit breaker → returns cached/fallback response
         Stops sending traffic → service recovers
         Gradually restores traffic
```

---

## 📐 API Gateway vs Load Balancer vs Service Mesh

This is a common interview confusion point. Know these distinctions:

```
LOAD BALANCER:
  What: Distributes traffic across identical server instances
  When: Same service, multiple instances (horizontal scaling)
  Knows about: IP, port, HTTP headers (L7)
  Example: 3 instances of UserService — LB spreads traffic evenly

API GATEWAY:
  What: Entry point for clients, routes to DIFFERENT services
  When: Many different services, one client entry point
  Knows about: Business routing rules, auth policies, rate limits
  Example: Routes /api/users to UserService, /api/orders to OrderService

SERVICE MESH:
  What: Handles service-to-service communication INSIDE the cluster
  When: Microservices need mTLS, observability between them
  Knows about: Traffic policies, service identity, retries, circuit breaking
  Example: UserService → OrderService communication secured with mTLS
```

```
ARCHITECTURE LAYERS:

                 [Client]
                    │
          ┌─────────▼─────────┐
          │    API GATEWAY     │ ← Client-facing layer
          │ (auth, routing,    │
          │  rate limit)       │
          └─────────┬─────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
 [UserSvc]     [OrderSvc]    [ProductSvc]
     │              │              │
     └──────┬───────┘              │
            │   Service Mesh ──────┘
            │   (mTLS, traces, retries between services)
            ▼
       [Database]
```

| Dimension | Load Balancer | API Gateway | Service Mesh |
|---|---|---|---|
| Scope | Instance-level | Service-level | Service-to-service |
| Client type | Any | External clients | Internal services |
| Auth/AuthZ | ❌ | ✅ | ✅ (mTLS identity) |
| Rate Limiting | ❌ | ✅ | ✅ |
| Business routing | ❌ | ✅ | ❌ |
| Example | Nginx, AWS ALB | Kong, AWS API GW | Istio, Linkerd |

---

## 🔄 Request Lifecycle Through a Gateway

```
STEP-BY-STEP REQUEST FLOW:

Client ──► [API Gateway] ──► Service
            │
            ├─ 1. SSL Termination: Decrypt HTTPS
            │
            ├─ 2. Authentication: Validate JWT/API key
            │    ├── Invalid: Return 401 Unauthorized
            │    └── Valid: Extract user context
            │
            ├─ 3. Rate Limiting: Check Redis counter
            │    ├── Exceeded: Return 429 Too Many Requests
            │    └── OK: Increment counter
            │
            ├─ 4. Authorization: Check user permissions
            │    ├── No access: Return 403 Forbidden
            │    └── Access OK: Continue
            │
            ├─ 5. Request Routing: Match path → service
            │    └── /api/orders → OrderService:8002
            │
            ├─ 6. Request Transform: Headers, path rewrite
            │    └── Add: X-User-ID: 123, X-Correlation-ID: uuid
            │
            ├─ 7. Forward to Service
            │
            ├─ 8. Response Transform: Format, filter sensitive data
            │
            ├─ 9. Log & Metrics: Record request data
            │
            └─ 10. Return response to client
```

---

## 🔐 Authentication & Authorization at the Gateway

### JWT Validation Pattern

```java
// Gateway-level JWT filter (Spring Cloud Gateway example)
@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {
    
    @Autowired
    private JwtUtils jwtUtils;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        
        // Skip auth for public endpoints
        String path = request.getPath().value();
        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }
        
        // Extract token
        String token = extractToken(request);
        if (token == null) {
            return unauthorized(exchange, "Missing Authorization header");
        }
        
        // Validate token
        try {
            Claims claims = jwtUtils.validateToken(token);
            
            // Add user context headers for downstream services
            ServerHttpRequest mutatedRequest = request.mutate()
                .header("X-User-ID", claims.getSubject())
                .header("X-User-Role", claims.get("role", String.class))
                .header("X-Tenant-ID", claims.get("tenantId", String.class))
                .build();
            
            return chain.filter(exchange.mutate().request(mutatedRequest).build());
            
        } catch (JwtException e) {
            return unauthorized(exchange, "Invalid or expired token");
        }
    }
    
    private String extractToken(ServerHttpRequest request) {
        String auth = request.getHeaders().getFirst("Authorization");
        if (auth != null && auth.startsWith("Bearer ")) {
            return auth.substring(7);
        }
        return null;
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        byte[] bytes = ("{\"error\": \"" + message + "\"}").getBytes();
        DataBuffer buffer = exchange.getResponse().bufferFactory().wrap(bytes);
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }
    
    @Override
    public int getOrder() { return -1; } // Run first
    
    private boolean isPublicPath(String path) {
        return path.startsWith("/api/auth/") || path.startsWith("/api/health");
    }
}
```

### Spring Cloud Gateway Route Config

```yaml
# application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE    # Load balanced (service discovery)
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users/(?<segment>.*), /users/${segment}
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 150
                key-resolver: "#{@userKeyResolver}"
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users

        - id: order-service
          uri: lb://ORDER-SERVICE
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/orders/(?<segment>.*), /orders/${segment}
```

---

## 🛠️ Popular API Gateway Tools

| Tool | Type | Best For | Key Features |
|---|---|---|---|
| **Kong** | Open Source / Enterprise | High performance, plugin ecosystem | 50+ plugins, custom Lua plugins, Admin API |
| **AWS API Gateway** | Managed Cloud | AWS ecosystem, serverless | Lambda integration, WebSocket, REST + HTTP APIs |
| **Spring Cloud Gateway** | OSS (Java) | Spring Boot microservices | Reactive, filter chains, service discovery |
| **Traefik** | OSS | Kubernetes-native | Auto-discovery from k8s labels, Let's Encrypt |
| **Nginx** | OSS / Commercial | General purpose, high throughput | Modules, Lua scripting, high performance |
| **Apigee** (Google) | Enterprise | Enterprise API management | Analytics, monetization, developer portal |
| **Azure API Management** | Managed Cloud | Azure ecosystem | Policy engine, developer portal |
| **Envoy** | OSS | Service mesh, advanced use cases | xDS protocol, telemetry, gRPC |

---

## 🏢 Industry Examples

```
NETFLIX:
- Uses "Zuul" (their open-source API Gateway)
- Handles authentication, routing, logging for all Netflix clients
- Zuul 2: Non-blocking, handles 100B+ API calls per day
- Later evolved to Spring Cloud Gateway internally

UBER:
- Built custom "API Router" for their Gateway
- Routes 1M+ RPM to 2000+ microservices
- Handles protocol translation (REST clients → Thrift internally)

AMAZON:
- AWS API Gateway powers their own internal APIs
- Throttles by default: 10,000 rps per account (adjustable)
- Built "API-First" culture — everything exposed via API Gateway

SPOTIFY:
- Uses their own "API Gateway" (Apollo platform)
- Aggregates responses from 100+ microservices for client apps
- Implemented BFF (Backend For Frontend) pattern per client type
```

---

## ⚠️ Common Pitfalls

### 1. API Gateway as a Business Logic Layer
```
❌ BAD: Gateway contains if-else business logic, database queries
        "If order total > $500, apply discount..."

✅ GOOD: Gateway only handles cross-cutting concerns
         Business logic stays in services
         Gateway = infrastructure, not application code
```

### 2. The Gateway Becomes a Monolith
```
❌ BAD: All routing, all auth, all transformations in one massive config
        Hard to deploy, hard to test, one change breaks everything

✅ GOOD: Per-service route config files (separate YAML per service)
         Or use multiple Gateways (one per domain/team)
```

### 3. Not Handling Gateway Failure
```
❌ BAD: One API Gateway → single point of failure
✅ GOOD: Multiple Gateway instances behind a Load Balancer
         Circuit breakers on downstream services
         Graceful degradation (cached responses)
```

### 4. Ignoring Observability at the Gateway
```
❌ BAD: No logging, no traces, no metrics at the gateway
✅ GOOD: Every request logged with correlation ID
         Gateway starts distributed trace (X-Trace-ID)
         Metrics: request count, latency p99, error rate per route
```

### 5. Putting Secret Logic in Gateway Config
```
❌ BAD: API keys, passwords, JWT secrets in gateway config files
✅ GOOD: Use environment variables or secrets manager (Vault, AWS SM)
         Reference secrets by name, never by value in config
```

---

## 🧩 Mini Challenge

```
🎲 SCENARIO (4 minutes):

You're building an e-commerce platform with these services:
  - ProductService: read-heavy, public (no auth required)
  - CartService: requires user auth
  - PaymentService: requires auth + "payment:write" permission
  - AdminService: requires auth + "admin" role

Your Gateway needs to handle:
  - 10,000 requests/second
  - DDoS protection
  - JWT validation
  - Different rate limits per endpoint

DESIGN QUESTION:
1. What should the Gateway do for each service?
2. How would you handle unauthenticated product browsing?
3. How do you protect PaymentService from being rate-limited
   alongside public product browsing?
```

<details>
<summary>💡 Click to reveal answer</summary>

**Gateway config per route:**

```
ProductService (/api/products/*):
  - Auth: SKIP (public endpoint)
  - Rate Limit: 500 req/min per IP (for unauthenticated bots)
  - Cache: YES — product catalog changes infrequently
  - Circuit Breaker: YES

CartService (/api/cart/*):
  - Auth: REQUIRED — validate JWT, extract user_id
  - Rate Limit: 200 req/min per user
  - Cache: NO — cart is user-specific, dynamic

PaymentService (/api/payment/*):
  - Auth: REQUIRED — validate JWT  
  - AuthZ: Check claims for "payment:write" scope
  - Rate Limit: STRICT — 10 req/min per user
  - Cache: NEVER
  - Extra: Add X-Idempotency-Key validation

AdminService (/api/admin/*):
  - Auth: REQUIRED
  - AuthZ: Check claims for "admin" role
  - Rate Limit: 1000 req/min per admin user
  - IP Allowlist: Only from corporate network
```

**Key insight**: Separate rate limit pools per route type. Product browsing getting hammered shouldn't affect payment processing.

**Separate pools:**
- Public endpoints: 1 rate limit bucket
- Authenticated endpoints: separate bucket per user
- Payment endpoints: strictest, separate bucket
</details>

---

## 📝 Interview Q&A

**Q1: What is an API Gateway and why is it needed in microservices?**
> An API Gateway is a single entry point for all client requests. In microservices, it centralizes cross-cutting concerns: auth, rate limiting, logging, routing, SSL termination. Without it, every service must implement these independently, creating inconsistency and duplication. Clients also need to know about every service's address.

**Q2: What is the difference between an API Gateway and a Load Balancer?**
> A Load Balancer distributes traffic across identical instances of the same service. An API Gateway routes traffic to DIFFERENT services based on business rules (path, header), while handling auth, rate limiting, and request transformation. They often work together: Gateway routes to service, LB distributes within the service pool.

**Q3: What is the BFF (Backend for Frontend) pattern?**
> BFF creates separate API Gateway instances per client type (mobile, web, third-party). Each BFF aggregates and shapes data for its specific client's needs. Mobile BFF returns compact responses; web BFF returns richer data. Avoids one-size-fits-all API compromises.

**Q4: How do you prevent the API Gateway from becoming a single point of failure?**
> Run multiple Gateway instances behind a Load Balancer. Use stateless Gateways (JWT validation is stateless). Implement circuit breakers so if a downstream service fails, Gateway returns fallback responses. Use health checks to detect Gateway failure. Deploy across multiple availability zones.

**Q5: Should business logic ever be in the API Gateway?**
> No. The API Gateway should only handle infrastructure concerns: routing, auth, rate limiting, protocol translation, logging. Business logic belongs in the services. Putting business logic in the Gateway creates tight coupling, makes testing harder, and means deploying Gateway for business changes.

---

## 🔗 What to Read Next

| Topic | Why You Need It |
|---|---|
| [Service Discovery](./ServiceDiscovery.md) | API Gateway needs to know WHERE to route requests — service discovery provides the addresses |
| [Circuit Breaker](./CircuitBreaker.md) | API Gateway should implement circuit breakers for downstream services — understand how they work |
| [Security/OAuth2](../Security/OAuth2.md) | API Gateways enforce OAuth2/JWT auth — understand the full auth flow they're implementing |

---

*[← Back to Building Blocks](../BuildingBlocks/) | [← Index](../INDEX.md)*
