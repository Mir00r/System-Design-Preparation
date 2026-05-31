# 🔑 Idempotency: Making Operations Safe to Retry

> *"Stripe processes $1 TRILLION in payments annually. Every single API call includes an idempotency key. Why? Because when a $10,000 payment request times out, the client will retry — and without idempotency, you'd charge the customer TWICE. Idempotency isn't a nice-to-have; it's the difference between a trusted platform and a lawsuit."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [RESTful APIs](./RESTful.md), [HTTP](./HTTP.md)

---

## 📋 Table of Contents
1. [What is Idempotency?](#-what-is-idempotency)
2. [Why Idempotency Matters](#-why-idempotency-matters)
3. [HTTP Methods & Idempotency](#-http-methods--idempotency)
4. [Implementation Patterns](#-implementation-patterns)
5. [Idempotency Keys](#-idempotency-keys)
6. [Real-World Implementations](#-real-world-implementations)
7. [Java/Spring Boot Implementation](#-javaspring-boot-implementation)
8. [Database Patterns](#-database-patterns)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Idempotency?

```
╔══════════════════════════════════════════════════════════════════╗
║  IDEMPOTENT = Doing something multiple times has the SAME      ║
║  effect as doing it once.                                      ║
║                                                                ║
║  f(x) = f(f(x)) = f(f(f(x))) = ... same result!              ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 Real-Life Idempotent vs Non-Idempotent

```
IDEMPOTENT (safe to repeat):
  🔘 Pressing elevator button: Press 10 times = same as once (still goes to floor 3)
  💡 Light switch ON: Flip "ON" 5 times = light is ON (same result)
  🏧 Setting balance to $100: Do it 3 times = balance is still $100
  🗑️ Delete email: Delete same email twice = it's still deleted

NON-IDEMPOTENT (repeating causes problems):
  💰 Transfer $100: Do it 3 times = transferred $300! 😱
  📧 Send email: Send 3 times = recipient gets 3 emails!
  🛒 Add to cart: Click 3 times = 3 items in cart!
  ➕ Increment counter: +1 three times = +3 (not +1!)
```

---

## 💥 Why Idempotency Matters

```
THE NETWORK IS UNRELIABLE:

  Client ──── POST /payment ────► Server (processes payment ✅)
  Client ◄─── [TIMEOUT] ────────── Server (response LOST in transit!)
  
  Client doesn't know: Did the payment succeed or fail?
  Client's only option: RETRY!
  
  WITHOUT IDEMPOTENCY:
    Client ──── POST /payment ────► Server (processes AGAIN = DOUBLE CHARGE! 💀)
    
  WITH IDEMPOTENCY:
    Client ──── POST /payment {key: "abc123"} ────► Server 
    Server: "I already processed abc123, here's the cached result"
    Client gets original response ✅ (no duplicate processing!)

SCENARIOS WHERE RETRIES HAPPEN:
  • Network timeout (most common)
  • Load balancer retry (upstream timeout)
  • Client-side retry logic
  • Message queue redelivery (at-least-once)
  • Kubernetes pod restart during processing
```

---

## 🌐 HTTP Methods & Idempotency

```
┌─────────────┬─────────────┬───────────────────────────────────────┐
│  Method     │  Idempotent │  Why?                                 │
├─────────────┼─────────────┼───────────────────────────────────────┤
│  GET        │  ✅ Yes     │  Reading doesn't change state         │
│  HEAD       │  ✅ Yes     │  Same as GET without body             │
│  PUT        │  ✅ Yes     │  "Set resource to THIS state"         │
│  DELETE     │  ✅ Yes     │  Deleting already-deleted = still gone│
│  OPTIONS    │  ✅ Yes     │  Informational only                   │
│  POST       │  ❌ No      │  "Create new" = duplicates possible!  │
│  PATCH      │  ❌ Maybe   │  Depends on operation                 │
└─────────────┴─────────────┴───────────────────────────────────────┘

WHY PUT IS IDEMPOTENT:
  PUT /users/123 {name: "Alice"}  ← Sets name to Alice
  PUT /users/123 {name: "Alice"}  ← Sets name to Alice (same!)
  Result: Alice. Always Alice. No matter how many times.

WHY POST IS NOT IDEMPOTENT:
  POST /orders {item: "book"}  ← Creates order #1
  POST /orders {item: "book"}  ← Creates order #2 (duplicate!)
  Result: Two orders! Not what the user wanted!
  
🎯 FIX: Make POST idempotent using IDEMPOTENCY KEYS
```

---

## 🛠️ Implementation Patterns

### Pattern 1: Idempotency Key (Most Common)

```
Client includes unique key with every request:

  POST /api/payments
  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  
  {
    "amount": 100.00,
    "currency": "USD",
    "recipient": "user_456"
  }

Server behavior:
  1. Receive request with key "550e8400..."
  2. Check: Have I seen this key before?
     YES → Return stored result (no re-processing!)
     NO  → Process request, store (key → result), return result
```

### Pattern 2: Natural Idempotency (Design for It)

```
INSTEAD OF:
  POST /api/counter/increment  ← Non-idempotent! Each call adds 1!
  
USE:
  PUT /api/counter/value/42    ← Idempotent! Sets to 42 regardless.

INSTEAD OF:
  POST /api/transfer {from: A, to: B, amount: 100}  ← Could duplicate!
  
USE:
  PUT /api/transfer/txn-uuid-123 {from: A, to: B, amount: 100}
  ← Same transaction ID → same transfer → idempotent!
```

### Pattern 3: Conditional Updates (ETags/Versions)

```
  GET /api/account/123
  → Response: {balance: 500, version: 7}
  
  PUT /api/account/123
  If-Match: 7                    ← "Only update if version is still 7"
  {balance: 400}
  
  First request: version 7 → 8, balance = 400 ✅
  Retry:         version is NOW 8, not 7 → 409 Conflict! ← Safe!
```

---

## 🔑 Idempotency Keys

```
LIFECYCLE OF AN IDEMPOTENCY KEY:

  ┌──────────┐     ┌──────────────┐     ┌──────────────┐
  │  Client  │     │   Server     │     │  Key Store   │
  │generates │────►│  checks key  │────►│  (Redis/DB)  │
  │  UUID    │     │              │     │              │
  └──────────┘     └──────────────┘     └──────────────┘
  
  Flow:
  1. Client generates UUID v4 (globally unique)
  2. Server receives request + key
  3. Server checks key store:
     a. Key NOT found → Process request, store (key, status, result)
     b. Key found, status=PROCESSING → Return 409 (in progress)
     c. Key found, status=COMPLETED → Return stored result
  4. Key expires after 24-48 hours (cleanup)

KEY STORAGE STATES:
  ┌─────────────┬──────────────────────────────────────────┐
  │  State      │  Meaning                                 │
  ├─────────────┼──────────────────────────────────────────┤
  │  STARTED    │  Request received, processing begun      │
  │  PROCESSING │  Work in progress                        │
  │  COMPLETED  │  Done, result stored                     │
  │  FAILED     │  Failed, safe to retry with same key     │
  └─────────────┴──────────────────────────────────────────┘
```

---

## 🏢 Real-World Implementations

### Stripe's Approach
```
Every Stripe API request supports:
  curl https://api.stripe.com/v1/charges \
    -H "Idempotency-Key: unique-key-123" \
    -d amount=2000 \
    -d currency=usd

Rules:
  - Keys expire after 24 hours
  - Results cached for the full 24h window
  - Different request body + same key = 400 error (mismatch!)
  - POST methods only (GET/PUT already idempotent)
  
  Stripe stores: key → (request_hash, response, status_code)
```

### AWS SQS + Lambda (At-Least-Once Delivery)
```
SQS guarantees AT-LEAST-ONCE delivery:
  Message might be delivered 2+ times!
  
  Solution: Message deduplication
  - SQS MessageDeduplicationId (FIFO queues)
  - Application-level idempotency check in Lambda

  Lambda receives message → 
    Check DynamoDB: "Have I processed message ID X?"
    YES → Skip (deduplicate)
    NO  → Process, then mark as processed
```

---

## 💻 Java/Spring Boot Implementation

### Complete Idempotency Filter

```java
@Component
public class IdempotencyFilter extends OncePerRequestFilter {
    
    @Autowired private RedisTemplate<String, IdempotencyRecord> redis;
    private static final Duration KEY_TTL = Duration.ofHours(24);
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {
        
        // Only apply to non-idempotent methods
        if (!"POST".equals(request.getMethod())) {
            chain.doFilter(request, response);
            return;
        }
        
        String idempotencyKey = request.getHeader("Idempotency-Key");
        if (idempotencyKey == null) {
            chain.doFilter(request, response); // No key = no protection
            return;
        }
        
        String redisKey = "idempotency:" + idempotencyKey;
        
        // Check if we've seen this key before
        IdempotencyRecord existing = redis.opsForValue().get(redisKey);
        
        if (existing != null) {
            if (existing.getStatus() == Status.COMPLETED) {
                // Return cached response (no re-processing!)
                response.setStatus(existing.getStatusCode());
                response.setContentType("application/json");
                response.getWriter().write(existing.getResponseBody());
                return;
            }
            if (existing.getStatus() == Status.PROCESSING) {
                response.setStatus(409); // Conflict - still processing
                response.getWriter().write("{\"error\": \"Request in progress\"}");
                return;
            }
        }
        
        // Mark as processing
        redis.opsForValue().set(redisKey, 
            new IdempotencyRecord(Status.PROCESSING), KEY_TTL);
        
        // Wrap response to capture output
        ContentCachingResponseWrapper wrappedResponse = 
            new ContentCachingResponseWrapper(response);
        
        try {
            chain.doFilter(request, wrappedResponse);
            
            // Store successful result
            redis.opsForValue().set(redisKey, new IdempotencyRecord(
                Status.COMPLETED,
                wrappedResponse.getStatus(),
                new String(wrappedResponse.getContentAsByteArray())
            ), KEY_TTL);
            
            wrappedResponse.copyBodyToResponse();
        } catch (Exception e) {
            // Mark as failed (safe to retry)
            redis.opsForValue().set(redisKey,
                new IdempotencyRecord(Status.FAILED), KEY_TTL);
            throw e;
        }
    }
}
```

### Annotation-Based Approach

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    String keyHeader() default "Idempotency-Key";
    long ttlSeconds() default 86400; // 24 hours
}

@RestController
public class PaymentController {
    
    @Idempotent
    @PostMapping("/api/payments")
    public ResponseEntity<Payment> createPayment(@RequestBody PaymentRequest req) {
        // This method will only execute ONCE per idempotency key!
        Payment payment = paymentService.processPayment(req);
        return ResponseEntity.ok(payment);
    }
}
```

---

## 🗄️ Database Patterns

### UPSERT Pattern (Natural Idempotency)

```sql
-- Instead of INSERT (duplicates possible):
INSERT INTO orders (id, user_id, amount) VALUES (?, ?, ?);

-- Use UPSERT (idempotent!):
INSERT INTO orders (id, user_id, amount) 
VALUES ('order-uuid-123', 'user-456', 100.00)
ON CONFLICT (id) DO NOTHING;  -- If exists, skip silently!

-- Or with UPDATE on conflict:
INSERT INTO orders (id, user_id, amount, status)
VALUES ('order-uuid-123', 'user-456', 100.00, 'CONFIRMED')
ON CONFLICT (id) DO UPDATE SET status = 'CONFIRMED';
```

### Deduplication Table

```sql
CREATE TABLE processed_requests (
    idempotency_key VARCHAR(64) PRIMARY KEY,
    status VARCHAR(20) NOT NULL,
    response_body JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL
);

-- Index for cleanup job
CREATE INDEX idx_expires ON processed_requests(expires_at);
```

---

## ⚠️ Common Pitfalls

| Pitfall | Why It's Dangerous | Fix |
|---------|-------------------|-----|
| 🔴 No idempotency on payment APIs | Network retry = double charge | ALWAYS use idempotency keys for money |
| 🔴 Storing key AFTER processing | Crash between process and store = no record | Store key BEFORE processing (status=PROCESSING) |
| 🔴 Same key, different body | Silently processes wrong request | Validate request body hash matches stored |
| 🟡 Keys never expire | Storage grows forever | Set 24-48h TTL, run cleanup job |
| 🟡 Not distinguishing failed vs completed | Client retries a permanently failed request forever | Store failure status, allow retry after failure |

---

## 🎮 Mini Challenge

### 🧩 Design: Idempotent Payment System

Design the idempotency layer for a payment processing API handling:
- 10,000 payments/second
- Must never double-charge
- Must handle network timeouts gracefully
- Must work across multiple API server instances

**Questions:**
1. Where do you store idempotency keys? (Redis? Database? Both?)
2. What happens if the server crashes MID-processing?
3. Client retries with same key but DIFFERENT amount — what do you do?
4. How long do you keep idempotency records?

<details>
<summary>🔑 Answers</summary>

1. **Redis** for fast lookup + **Database** as source of truth. Redis for hot path (< 1ms), DB for durability.
2. Key is in PROCESSING state. On retry: return 409 "in progress." Background job detects stuck processing (> timeout) and marks as FAILED.
3. **Return 422 error** — "Idempotency key already used with different parameters." Never silently process a different request.
4. **24-48 hours** — Long enough for all retry windows, short enough to not waste storage.
</details>

---

## ❓ Interview Q&A

**Q1: What is idempotency and why is it important in distributed systems?**
> An operation is idempotent if performing it multiple times has the same effect as performing it once. It's critical because networks are unreliable — clients will retry failed requests. Without idempotency, retries cause duplicate operations (double charges, duplicate orders, etc.).

**Q2: How do you make a POST endpoint idempotent?**
> Use an idempotency key: (1) Client generates unique key (UUID), (2) Includes key in request header, (3) Server checks if key was seen before, (4) If seen → return cached result, (5) If new → process and cache result. Store keys in Redis with TTL.

**Q3: What's the difference between idempotency and exactly-once delivery?**
> Exactly-once delivery means a message is processed exactly once (very hard in distributed systems). Idempotency is a practical alternative: accept AT-LEAST-ONCE delivery but make the processing idempotent, so duplicates don't matter. Easier to implement and more resilient.

**Q4: Which HTTP methods are idempotent by specification?**
> GET, PUT, DELETE, HEAD, OPTIONS are idempotent. POST and PATCH are NOT idempotent by default. PUT is "set to this state" (repeatable), POST is "create new" (creates duplicates). Making POST idempotent requires explicit implementation (idempotency keys).

**Q5: How does Stripe handle idempotency?**
> Every POST request accepts an `Idempotency-Key` header. Stripe stores the (key → response) mapping for 24 hours. Retries with same key return the stored response without re-processing. Different body with same key returns a 400 error. This prevents double-charges even during network failures.

---

## 🔗 Related Topics
- [RESTful APIs](./RESTful.md) — HTTP method idempotency semantics
- [Message Queues](../BuildingBlocks/MessageQueues.md) — At-least-once delivery needs idempotency
- [Reliability](../KeyConcepts/Reliability.md) — Idempotency enables reliable retries
- [Payment System Design](../SystemDesignCaseStudies/DesignPaymentSystem.md) — Critical use case

---

*"In a distributed system, 'did my request succeed?' is not a yes/no question — it's 'yes', 'no', or 'I don't know, but it's safe to try again.'" — The Art of Idempotent Design* 🔑
