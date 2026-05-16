# 🚦 Rate Limiting: Protecting Your API from Abuse and Overload

> **"Without rate limiting, your API is an open buffet. Everyone takes as much as they want until there's nothing left for anyone."**

---

## 🎯 What You'll Learn
- Why every public API needs rate limiting
- 4 rate limiting algorithms — Token Bucket, Leaky Bucket, Fixed Window, Sliding Window
- How to implement rate limiting in a distributed system
- Redis-based distributed rate limiter from scratch
- How Stripe, GitHub, and Twitter implement rate limiting

**⏱️ Estimated Time**: 22 minutes | **🎯 Difficulty**: 🟡 Medium  
**🔗 Prerequisites**: [Caching](./Caching.md) | [API Gateway](./APIGateway.md)  
**🔗 Related Topics**: [Load Balancing](./LoadBalancing.md) | [Circuit Breaker](./CircuitBreaker.md) | [System Design: Rate Limiter](../SystemDesignCaseStudies/DesignRateLimiter.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [What Is Rate Limiting?](#-what-is-rate-limiting)
3. [Algorithm 1: Fixed Window Counter](#-algorithm-1-fixed-window-counter)
4. [Algorithm 2: Sliding Window Log](#-algorithm-2-sliding-window-log)
5. [Algorithm 3: Sliding Window Counter](#-algorithm-3-sliding-window-counter)
6. [Algorithm 4: Token Bucket](#-algorithm-4-token-bucket)
7. [Algorithm 5: Leaky Bucket](#-algorithm-5-leaky-bucket)
8. [Algorithm Comparison](#-algorithm-comparison)
9. [Distributed Rate Limiting](#-distributed-rate-limiting)
10. [Code Examples](#-code-examples)
11. [Rate Limit Headers](#-rate-limit-headers)
12. [Industry Examples](#-industry-examples)
13. [Common Pitfalls](#-common-pitfalls)
14. [Mini Challenge](#-mini-challenge)
15. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

It's 2 AM. You're asleep. A badly-coded bot starts hitting your payment API.

```
WITHOUT RATE LIMITING:

Bot: 10,000 requests/second to POST /api/charge
Your API: accepts all of them
Database: falls over under the write load
Payment service: times out
Real users: "Payment failed" errors
Fraudulent charges: possibly thousands processed
```

Or more legitimately: you offer a free API tier. One power user writes a script that makes 1 million calls per day, consuming all your compute budget.

```
WITH RATE LIMITING:

Free tier:    100 requests/minute  (hard cap)
Pro tier:     1,000 requests/minute
Enterprise:   10,000 requests/minute + burst

Bot makes call 101: HTTP 429 Too Many Requests
                    Retry-After: 42 seconds
                    
Real users: unaffected ✅
Your servers: protected ✅
Revenue model: enforced ✅
```

> 💡 **Key Insight**: Rate limiting serves three purposes — **security** (block abuse), **reliability** (protect servers from overload), and **fairness** (ensure equal access for all users).

---

## 💡 What Is Rate Limiting?

Rate limiting controls **how many requests a client can make in a given time window**.

```
RATE LIMITING DIMENSIONS:

By user:           User A: max 100 req/min
By IP address:     IP 1.2.3.4: max 60 req/min
By API key:        Key xyz123: max 1000 req/min
By endpoint:       POST /auth: max 5 req/min (brute force protection)
By user + endpoint: User A on /api/search: max 30 req/min
```

---

## 🌍 Real-World Analogy

Think of rate limiting like a **nightclub bouncer**:

```
NIGHTCLUB BOUNCER ANALOGY:

Club capacity: 200 people maximum
Rule: Once full, no one enters until someone leaves

Without rate limit: 500 people pile in → fire hazard → chaos
With rate limit: First 200 in, rest wait in orderly queue

Rate limit = the bouncer
Requests = people wanting to enter
Time window = the club's open hours
```

---

## 🔢 Algorithm 1: Fixed Window Counter

The simplest algorithm. Count requests in fixed time windows (e.g., every minute).

```
TIME:   |—00:00—|—00:01—|—00:02—|
         [100]    [100]    [100]
         
Limit: 100 req/min

:00:00 - :00:59 → Request 1-100: ✅ Allowed
:00:45          → Request 101:   ❌ Rejected (window full)
:01:00          → Counter RESETS to 0
:01:00          → Request 1:     ✅ Allowed again
```

**✅ Pros**: Simple to implement, O(1) memory per user  
**❌ Cons**: Boundary vulnerability!

```
THE SPIKE PROBLEM:

Limit: 100 req/min

:00:55 → 100 requests flood in → all allowed (end of window)
:01:00 → counter resets
:01:05 → 100 more requests → all allowed (new window)

Result: 200 requests in 10 seconds — 2x the intended limit!
```

---

## 📜 Algorithm 2: Sliding Window Log

Store the timestamp of every request. Count requests in the last N seconds.

```
Limit: 100 req/min

Current time: 01:30

Log: [00:31, 00:45, 01:00, 01:15, 01:28, 01:29, 01:30]

Window: last 60 seconds = [00:31 to 01:30]
Count requests in window: 7 → 7 < 100 → ✅ Allow

New request at 01:31:
  Remove timestamps < 00:31 (outside window)
  Add 01:31 to log
  Count: still within limit
```

**✅ Pros**: No boundary spike problem, very accurate  
**❌ Cons**: High memory usage — must store timestamp for every request. 1M users × 100 req/min = 100M timestamps stored

---

## 🪟 Algorithm 3: Sliding Window Counter (Hybrid)

Combines Fixed Window's memory efficiency with Sliding Window's accuracy.

```
MATH:
weighted_count = curr_window_count × (curr_window_elapsed/window_size)
               + prev_window_count × (1 - curr_window_elapsed/window_size)

EXAMPLE:
Limit: 100 req/min
Previous window (00:00-01:00): 80 requests
Current window (01:00-02:00): 30 requests so far
Current time: 01:20 → 20/60 = 33% through current window

weighted_count = 30 × 0.33 + 80 × 0.67
              = 10 + 54 = 64

64 < 100 → ✅ Allow
```

**✅ Pros**: Memory efficient (O(1) per user), smooths out boundary spikes  
**❌ Cons**: Approximation, not perfectly accurate (usually within 0.1%)  
**🏆 Best For**: Most production systems — Cloudflare uses this

---

## 🪣 Algorithm 4: Token Bucket

Tokens are added to a bucket at a fixed rate. Each request consumes a token.

```
TOKEN BUCKET:

Configuration:
  bucket_capacity = 10 tokens     (max burst)
  refill_rate = 1 token/second    (steady-state rate)

VISUALIZATION:

  [🪙🪙🪙🪙🪙🪙🪙🪙🪙🪙] 10 tokens full

  Request arrives:
  [🪙🪙🪙🪙🪙🪙🪙🪙🪙░] Take 1 token → allowed ✅
  
  5 burst requests:
  [🪙🪙🪙🪙🪙░░░░░] 5 tokens left
  
  1 token added per second (refill)
  After 5 seconds: [🪙🪙🪙🪙🪙🪙🪙🪙🪙🪙] Full again
  
  If bucket empty:
  [░░░░░░░░░░] Request arrives → no token → rejected ❌ HTTP 429
```

**✅ Pros**: Allows controlled bursts (excellent for APIs), smooth handling  
**❌ Cons**: Tricky to implement in distributed systems (need shared token state)  
**🏆 Best For**: APIs that want to allow short bursts (AWS API Gateway uses this)

---

## 💧 Algorithm 5: Leaky Bucket

Requests enter a queue (bucket). They're processed at a fixed constant rate — "leaking" out.

```
LEAKY BUCKET:

Incoming:  [req, req, req, req, req, req] (bursty)
Queue:     [req → req → req → req]        (buffered)
Outgoing:  1 request per 100ms            (constant rate)

If queue full → new requests dropped (HTTP 429)
```

**✅ Pros**: Smooth output rate, protects downstream services from traffic spikes  
**❌ Cons**: Doesn't handle burst traffic gracefully (unlike token bucket), adds latency  
**🏆 Best For**: When downstream service can only handle a fixed rate (e.g., a rate-limited 3rd party API you're calling)

---

## 📊 Algorithm Comparison

| Algorithm | Memory | Burst Handling | Accuracy | Complexity | Best For |
|---|---|---|---|---|---|
| Fixed Window | O(1) | ❌ Boundary spike | Low | Simple | Internal services |
| Sliding Window Log | O(n) | ✅ Accurate | High | Medium | Low traffic, precise |
| Sliding Window Counter | O(1) | ✅ Good | Medium-High | Medium | **Most production APIs** |
| Token Bucket | O(1) | ✅ Excellent (burst allowed) | High | Medium-Hard | Public APIs, allow bursts |
| Leaky Bucket | O(n) | ❌ Smooths/drops burst | High | Medium | Fixed-rate downstream calls |

---

## 🌐 Distributed Rate Limiting

Single-server rate limiting is simple. But with 50+ API servers, each server tracks its own count — and a user can hit each server separately, bypassing the limit.

```
❌ WITHOUT DISTRIBUTED RATE LIMITING:

User limit: 100 req/min per user
Your fleet: 10 API servers

User sends 100 requests to each server:
  Server 1: 100/100 ✅ (all allowed)
  Server 2: 100/100 ✅ (all allowed)
  ...
  Server 10: 100/100 ✅ (all allowed)
  
Total: 1000 requests — 10x the intended limit!
```

**Solution**: Centralized counter with Redis

```
✅ WITH REDIS DISTRIBUTED RATE LIMITING:

All 10 servers share ONE Redis counter per user

Request hits Server 3:
  INCR rate:user123:2026-01-17-14:30
  → counter is now 47 (across ALL servers)
  47 < 100 → ✅ Allow

Request hits Server 7 (same user, same minute):
  INCR rate:user123:2026-01-17-14:30
  → counter is now 101
  101 > 100 → ❌ Reject (HTTP 429)
```

---

## 💻 Code Examples

### Redis Sliding Window Counter (Java + Spring Boot)

```java
@Component
public class RateLimiter {
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private static final int WINDOW_SIZE_SECONDS = 60;
    private static final int MAX_REQUESTS = 100;
    
    public boolean isAllowed(String userId) {
        long now = System.currentTimeMillis() / 1000; // current second
        String key = "rate:" + userId + ":" + (now / WINDOW_SIZE_SECONDS);
        
        // Atomic increment + expire
        Long count = redisTemplate.opsForValue().increment(key);
        
        if (count == 1) {
            // First request in this window — set expiry
            redisTemplate.expire(key, WINDOW_SIZE_SECONDS * 2, TimeUnit.SECONDS);
        }
        
        return count <= MAX_REQUESTS;
    }
}
```

### Token Bucket with Redis Lua Script (Atomic)

```java
@Component  
public class TokenBucketRateLimiter {
    
    // Lua script for atomic token bucket operation
    private static final String TOKEN_BUCKET_SCRIPT = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])  -- tokens per second
        local now = tonumber(ARGV[3])
        local requested = tonumber(ARGV[4])
        
        local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
        local tokens = tonumber(bucket[1]) or capacity
        local last_refill = tonumber(bucket[2]) or now
        
        -- Refill tokens based on elapsed time
        local elapsed = now - last_refill
        tokens = math.min(capacity, tokens + elapsed * refill_rate)
        
        local allowed = tokens >= requested
        if allowed then
            tokens = tokens - requested
        end
        
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, 3600)
        
        return {allowed and 1 or 0, tokens}
        """;
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private final RedisScript<List<Long>> script = 
        new DefaultRedisScript<>(TOKEN_BUCKET_SCRIPT, List.class);
    
    public boolean isAllowed(String userId, int capacity, double refillRate) {
        String key = "token_bucket:" + userId;
        long now = System.currentTimeMillis() / 1000;
        
        List<Long> result = redisTemplate.execute(
            script,
            List.of(key),
            String.valueOf(capacity),
            String.valueOf(refillRate),
            String.valueOf(now),
            "1"  // requesting 1 token
        );
        
        return result.get(0) == 1L;
    }
}
```

### Spring Boot Rate Limit Filter

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    
    @Autowired
    private RateLimiter rateLimiter;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) 
        throws ServletException, IOException {
        
        String userId = extractUserId(request); // from JWT/API key
        
        if (!rateLimiter.isAllowed(userId)) {
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value()); // 429
            response.setHeader("Retry-After", "60");
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
            return;
        }
        
        filterChain.doFilter(request, response);
    }
    
    private String extractUserId(HttpServletRequest request) {
        // Extract from API key, JWT, or IP address
        String apiKey = request.getHeader("X-API-Key");
        return apiKey != null ? apiKey : request.getRemoteAddr();
    }
}
```

---

## 📬 Rate Limit Headers

Always return these headers so clients know their limit status:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100          ← Max requests allowed per window
X-RateLimit-Remaining: 47       ← Requests remaining in current window
X-RateLimit-Reset: 1684237620   ← Unix timestamp when window resets
X-RateLimit-Window: 60          ← Window size in seconds

HTTP/1.1 429 Too Many Requests
Retry-After: 42                 ← Seconds until they can retry
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1684237620
```

---

## 🏢 Industry Examples

```
GITHUB API:
- Unauthenticated: 60 req/hour
- Authenticated: 5,000 req/hour  
- Uses Token Bucket algorithm
- Returns X-RateLimit-* headers on every response

STRIPE API:
- 100 read requests/second per secret key
- 100 write requests/second (but payments are special-cased)
- Uses sliding window
- Idempotency keys to prevent duplicate charges under retry

TWITTER/X API:
- Varies by endpoint and tier
- Search: 180 req/15min (basic)
- Uses rate limit per user AND per app
- Returns x-rate-limit-remaining in headers

CLOUDFLARE:
- Uses Sliding Window Counter for their WAF rate limiting rules
- Can rate limit by IP, ASN, user agent, cookie, path, header
- Handles 50+ million HTTP requests/second globally
```

---

## ⚠️ Common Pitfalls

### 1. Rate Limiting at the Wrong Layer
```
❌ BAD: Rate limit inside the application code on each server
        (distributed problem — doesn't work as shown above)
✅ GOOD: Rate limit at API Gateway or with shared Redis counter
```

### 2. Not Handling Retry-After on the Client Side
```
❌ BAD client: Receives 429 → immediately retries → more 429s → storm
✅ GOOD client: Reads Retry-After header → waits → retries
                Uses exponential backoff with jitter as fallback
```

### 3. Same Limit for All Endpoints
```
❌ BAD: POST /auth: 1000 req/min   ← Allows brute force!
✅ GOOD: 
  POST /auth:     5 req/min    (strict — prevent brute force)
  GET /products:  1000 req/min  (relaxed — read-heavy)
  POST /payment:  10 req/min    (tight — high value operation)
```

### 4. Race Conditions Without Atomic Operations
```
❌ BAD: 
  Thread 1: GET counter = 99
  Thread 2: GET counter = 99
  Thread 1: SET counter = 100 → allowed
  Thread 2: SET counter = 100 → also allowed!
  Result: 2 requests where only 1 should be allowed

✅ GOOD: Use Redis INCR (atomic) or Lua script
```

---

## 🧩 Mini Challenge

```
🎲 SCENARIO (4 minutes):

You're designing a rate limiter for a financial API.

Requirements:
  - Max 100 requests/minute per user (steady state)
  - Allow burst of 20 extra requests in peak periods
  - POST /transfer: Max 10 per minute (extra strict)
  - Distributed across 20 API servers

QUESTIONS:
1. Which algorithm should you use for the main limit?
2. How does the burst requirement change your design?
3. How do you handle the distributed problem?
4. What happens if Redis goes down?
```

<details>
<summary>💡 Click to reveal answer</summary>

**1. Algorithm**: Token Bucket — specifically designed for "steady rate + burst allowance". Capacity=120 (100+20 burst), refill_rate=100/minute (1.67/second).

**2. Burst design**: Set bucket capacity to 120 (not 100). Normal state: 100 tokens (100 req/min). Burst: can spend up to 20 extra tokens, which then take ~12 seconds to refill.

**3. Distributed**: Use Redis with Lua script for atomic token bucket operations. All 20 servers share one Redis key per user. Use Redis Cluster for high availability.

**4. Redis down (Circuit Breaker)**: 
- Option A: **Fail open** — allow all requests when Redis unavailable (may be abused, but service stays up)
- Option B: **Fail closed** — reject all requests when Redis down (safer for financial API)
- Option C: **Local fallback** — fall back to in-process rate limiter (less accurate but functional)

For a financial API, **fail closed is safer** — accept brief unavailability over allowing unlimited financial transactions.

**POST /transfer**: Separate rate limit key: `rate:user123:transfer`. Strict limit: 10/min. Use fixed window since simplicity is fine here — the risk of boundary spike (20 transfers in 10 sec) is the same risk as 10 transfers/min. Token bucket if you want to be strict.
</details>

---

## 📝 Interview Q&A

**Q1: What is rate limiting and why is it important?**
> Rate limiting controls the number of requests a client can make in a time window. It's important for: security (prevent brute force, DDoS), reliability (protect servers from overload), and fairness (ensure equal resource distribution among users).

**Q2: Explain the Token Bucket algorithm.**
> A bucket holds tokens up to a max capacity. Tokens are added at a fixed rate (e.g., 10/second). Each request consumes 1 token. If bucket is full, extra tokens are discarded. If bucket is empty, requests are rejected. This allows burst traffic (spending multiple tokens at once) while enforcing a steady-state rate.

**Q3: How do you implement rate limiting in a distributed system?**
> Use a centralized counter store like Redis. All servers share one Redis key per user. Use atomic operations (INCR + EXPIRE, or Lua scripts) to prevent race conditions. Redis Cluster provides high availability. For edge cases (Redis down), decide between fail-open or fail-closed based on use case.

**Q4: What is the difference between Fixed Window and Sliding Window rate limiting?**
> Fixed Window resets counters at fixed intervals (e.g., every minute). Has boundary vulnerability — burst at window boundary allows 2x requests. Sliding Window tracks request timestamps or uses weighted counting. No boundary vulnerability, more accurate, but uses more memory (log approach) or is an approximation (counter approach).

**Q5: How do you rate limit without hurting legitimate users?**
> Use tiered limits (free vs paid tiers). Return clear headers (X-RateLimit-Remaining, Retry-After). Apply stricter limits to sensitive endpoints (auth, payment) and lenient limits to read-heavy endpoints. Implement graceful degradation — allow burst for short spikes. Never rate limit internal health checks.

---

## 🔗 What to Read Next

| Topic | Why You Need It |
|---|---|
| [API Gateway](./APIGateway.md) | API Gateways implement rate limiting as a built-in feature — understand how they integrate |
| [Circuit Breaker](./CircuitBreaker.md) | Rate limiting protects YOUR API; Circuit Breaker protects YOU from overloaded downstream services |
| [System Design: Rate Limiter](../SystemDesignCaseStudies/DesignRateLimiter.md) | Full end-to-end design of a distributed rate limiter — the classic interview question |

---

*[← Back to Building Blocks](../BuildingBlocks/) | [← Index](../INDEX.md)*
