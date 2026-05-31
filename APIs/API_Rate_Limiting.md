# 🚦 API Rate Limiting: Protecting APIs from Abuse 🚀

> **"Without rate limiting, your API is not a service — it's an all-you-can-eat buffet for bots."**

**⏱️ Estimated Time**: 30 minutes  
**🎯 Difficulty**: 🟡 Intermediate  
**🔗 Prerequisites**: [API Security](./API_Security.md) | [Caching](../BuildingBlocks/Caching.md) | [Redis Fundamentals](../BuildingBlocks/Redis.md)

---

## 📋 Table of Contents

1. [Why Rate Limiting Matters](#-why-rate-limiting-matters)
2. [Four Core Algorithms](#-four-core-algorithms)
3. [Token Bucket Algorithm](#-token-bucket-algorithm)
4. [Sliding Window Log & Counter](#-sliding-window-log--counter)
5. [Leaky Bucket Algorithm](#-leaky-bucket-algorithm)
6. [Fixed Window Counter](#-fixed-window-counter)
7. [Choosing the Right Algorithm](#-choosing-the-right-algorithm)
8. [Distributed Rate Limiting with Redis](#-distributed-rate-limiting-with-redis)
9. [Rate Limiting Granularity](#-rate-limiting-granularity)
10. [Spring Boot Implementation](#-spring-boot-implementation)
11. [HTTP Headers for Rate Limits](#-http-headers)
12. [Interview Q&A](#-interview-qa)
13. [Mini Challenge](#-mini-challenge)

---

## 💀 Why Rate Limiting Matters

It's 3 AM. A competitor's scraper bot hits your pricing API at 500 requests/second. By 6 AM it has downloaded your entire product catalog — 5 years of pricing research and market intelligence — and your database server is on its knees. Your morning users get 504 errors.

Rate limiting is not optional. It's infrastructure.

```
WITHOUT RATE LIMITING:               WITH RATE LIMITING:
  ┌────────────────────┐               ┌──────────────────────┐
  │  Scraper Bot       │               │  Scraper Bot         │
  │  500 req/sec ──────┼──────→        │  500 req/sec ────────┼──→ 429 Too Many Requests
  │                    │               │                      │    Retry-After: 60s
  │  Legitimate Users  │               │                      │
  │  10 req/sec ───────┼──────→ 504    │  Legitimate Users    │
  │                    │   (overloaded)│  10 req/sec ─────────┼──→ 200 OK (protected)
  └────────────────────┘               └──────────────────────┘
```

**What rate limiting protects:**
- **DoS/DDoS attacks**: Volumetric attacks from single or distributed sources
- **Credential stuffing**: Bots trying millions of username/password combinations
- **Data scraping**: Systematic harvesting of your data catalog
- **Cost control**: Cloud APIs (AI, SMS, email) with per-call pricing
- **Fairness**: Preventing one tenant from monopolizing shared infrastructure

---

## 🔢 Four Core Algorithms

```
ALGORITHM        SMOOTHNESS   BURST HANDLING   COMPLEXITY   BEST FOR
──────────────────────────────────────────────────────────────────────────
Token Bucket     High         ✅ Allows burst   Medium       General purpose APIs
Sliding Window   Very High    ❌ No burst       High         Fair per-user limits
Leaky Bucket     Perfect      ❌ Discards burst Medium       Outbound rate control
Fixed Window     Low          ❌ Edge bursts    Low          Simple/cheap use cases
```

---

## 🪣 Token Bucket Algorithm

The most popular rate-limiting algorithm. Imagine a bucket that holds tokens:

```
TOKEN BUCKET MECHANICS:
─────────────────────────────────────────────────────────────────────
Bucket capacity: 100 tokens (max burst)
Refill rate: 10 tokens/second (sustained rate)

t=0s:   Bucket full [■■■■■■■■■■] 100 tokens
        Request arrives → consume 1 token → 99 remaining ✅

t=0s+:  10 requests burst → consume 10 tokens → 90 remaining ✅
        (burst is ALLOWED because tokens were accumulated)

t=5s:   50 tokens accumulated (5s × 10/s = 50 new tokens)
        100 requests arrive → consume 100 → 0 remaining ✅ then ❌
        (burst allowed up to bucket size)

t=10s:  0 tokens → request arrives → 429 Too Many Requests ❌
        Response: Retry-After: 1 (wait 1 second for 10 new tokens)
```

```java
// Token Bucket implementation using Bucket4j:
@Service
public class TokenBucketRateLimiter {
    
    private final Cache<String, Bucket> bucketCache;
    
    public TokenBucketRateLimiter() {
        this.bucketCache = Caffeine.newBuilder()
            .expireAfterAccess(1, TimeUnit.HOURS)
            .maximumSize(100_000)
            .build();
    }
    
    public RateLimitResult tryConsume(String key, RateLimitTier tier) {
        Bucket bucket = bucketCache.get(key, k -> createBucket(tier));
        ConsumptionProbe probe = bucket.tryConsumeAndReturnRemaining(1);
        
        return new RateLimitResult(
            probe.isConsumed(),
            probe.getRemainingTokens(),
            probe.getNanosToWaitForRefill() / 1_000_000 // ms until next token
        );
    }
    
    private Bucket createBucket(RateLimitTier tier) {
        return Bucket.builder()
            .addLimit(
                // Sustained rate: 100 per minute
                Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1)))
            )
            .addLimit(
                // Burst: up to 20 per second (spike protection)
                Bandwidth.classic(20, Refill.greedy(20, Duration.ofSeconds(1)))
            )
            .build();
    }
}

record RateLimitResult(boolean allowed, long remaining, long retryAfterMs) {}
```

**Why two limits?** The second limit prevents 100 requests being fired in the first second of each minute (boundary burst problem).

---

## 📊 Sliding Window Log & Counter

### Sliding Window Log (Precise but Memory-Heavy)

```
SLIDING WINDOW LOG — 5 requests per 60 seconds:

Timeline:  ──────────────────────────────────────────────────────────→
           0s    10s   20s   30s   40s   50s   60s   70s   80s

Requests:  R1    R2    R3    R4    R5          R6

At t=70s, when R6 arrives:
  Window = [70s-60s, 70s] = [10s, 70s]
  Log: [R2=10s, R3=20s, R4=30s, R5=40s]  ← R1 fell outside window
  Count in window = 4 → 4 < 5 → ✅ Allow R6

Memory: Must store timestamp of EVERY request → expensive at scale
```

### Sliding Window Counter (Approximate but Efficient)

```
SLIDING WINDOW COUNTER — uses two fixed windows + interpolation:

Window size: 60s, Limit: 100 requests

Current fixed window:   [60s-120s], count = 40 requests
Previous fixed window:  [0s-60s],   count = 70 requests
Current time: t = 90s  → 50% through current window

Estimated count = (70 × (1 - 0.5)) + 40 = 35 + 40 = 75

75 < 100 → ✅ Allow request

Why "approximate"? We assume the previous window's requests were spread evenly.
The error rate is typically <1% — good enough for production.
Memory: Only 2 counters per key vs N timestamps → 10,000x more efficient
```

```java
// Sliding window counter with Redis:
@Service
public class SlidingWindowRateLimiter {
    
    private final StringRedisTemplate redis;
    private final long windowSizeMs = 60_000;
    private final int limit = 100;
    
    public boolean isAllowed(String key) {
        long now = System.currentTimeMillis();
        long currentWindowStart = (now / windowSizeMs) * windowSizeMs;
        long prevWindowStart = currentWindowStart - windowSizeMs;
        double percentIntoCurrentWindow = (double)(now - currentWindowStart) / windowSizeMs;
        
        String currentKey = key + ":" + currentWindowStart;
        String prevKey = key + ":" + prevWindowStart;
        
        Long currentCount = redis.opsForValue().increment(currentKey);
        redis.expire(currentKey, Duration.ofMillis(windowSizeMs * 2));
        
        String prevCountStr = redis.opsForValue().get(prevKey);
        long prevCount = prevCountStr == null ? 0 : Long.parseLong(prevCountStr);
        
        double estimatedCount = (prevCount * (1 - percentIntoCurrentWindow)) + currentCount;
        return estimatedCount <= limit;
    }
}
```

---

## 🪣 Leaky Bucket Algorithm

Best for controlling OUTBOUND rate (sending to a 3rd party API with strict limits):

```
LEAKY BUCKET MECHANICS:
─────────────────────────────────────────────────────────────────────
Bucket capacity: 100 (queue size)
Leak rate: 10 requests/second (constant outbound rate)

Requests arrive at any rate:
  ┌─────────────────────────┐
  │  Incoming: 50 req/burst  │  → 50 items enter bucket
  │  ──────────────────────  │
  │  Bucket: [■■■■■■■■■■...] │  ← 50 items queued
  │  ──────────────────────  │
  │  Outgoing: 10 req/sec   │  → Processed at CONSTANT rate (smooth!)
  └─────────────────────────┘

If bucket overflows (>100): new requests DROPPED (no burst tolerated)

USE CASE: Calling Stripe API (100 req/s limit) — you smooth your traffic
          to exactly 100/s regardless of how spiky your internal load is
```

```java
// Leaky bucket using BlockingQueue (single-node implementation):
@Service
public class LeakyBucketLimiter {
    private final BlockingQueue<Runnable> bucket;
    private final ScheduledExecutorService scheduler;
    
    public LeakyBucketLimiter(int capacity, int ratePerSecond) {
        this.bucket = new ArrayBlockingQueue<>(capacity);
        this.scheduler = Executors.newSingleThreadScheduledExecutor();
        
        // Leak at constant rate
        long intervalMs = 1000L / ratePerSecond;
        scheduler.scheduleAtFixedRate(() -> {
            Runnable task = bucket.poll();
            if (task != null) task.run();
        }, 0, intervalMs, TimeUnit.MILLISECONDS);
    }
    
    public boolean submit(Runnable request) {
        return bucket.offer(request); // Returns false if bucket full → drop
    }
}
```

---

## 📦 Fixed Window Counter

Simplest algorithm — good for low-stakes use cases:

```
FIXED WINDOW COUNTER:

Window: [0s-60s], Limit: 100

t=0s:  Counter = 0
t=1s:  Request → Counter = 1  ✅
...
t=59s: Request → Counter = 100 ✅
t=59s: Request → Counter = 101 ❌ (rejected)
t=60s: WINDOW RESETS → Counter = 0  🔄

THE BOUNDARY BURST PROBLEM:
  User fires 100 requests at t=59s → allowed (fills window)
  Window resets at t=60s
  User fires 100 requests at t=60s → allowed again!
  → 200 requests in 2 seconds despite "100 per minute" limit!
  
SOLUTION: Use token bucket or sliding window if boundary bursts are a concern
```

---

## 📐 Choosing the Right Algorithm

```
DECISION TREE:
─────────────────────────────────────────────────────────────────────
Do you need to SMOOTH OUTBOUND traffic to an external service?
  YES → Leaky Bucket
  NO  ↓

Do you need PRECISE per-user fairness with no burst tolerance?
  YES → Sliding Window Log (if memory allows) or Sliding Window Counter
  NO  ↓

Do you need to allow SHORT BURSTS while enforcing a sustained rate?
  YES → Token Bucket (most common choice)
  NO  ↓

Do you need the SIMPLEST possible implementation?
  YES → Fixed Window Counter (with awareness of boundary burst issue)
```

---

## 🗄️ Distributed Rate Limiting with Redis

Single-node rate limiting breaks in distributed deployments. Use Redis as the shared counter:

```
SINGLE NODE (broken):                  DISTRIBUTED (correct):
  ┌──────────────┐                       ┌──────────────┐
  │  Server 1    │                       │  Server 1    │──┐
  │  Local map   │← each server has      │              │  │  ┌─────────────┐
  │  user:X = 50 │  its own counter      │              │  ├──│    Redis    │
  └──────────────┘                       └──────────────┘  │  │  user:X=100 │
  ┌──────────────┐                       ┌──────────────┐  │  └─────────────┘
  │  Server 2    │                       │  Server 2    │──┘
  │  Local map   │                       │              │
  │  user:X = 50 │                       │              │
  └──────────────┘                       └──────────────┘
  User X used 100 req                    User X used 100 req
  but each server sees 50                One shared counter = accurate
  → User bypasses limit!
```

```java
// Distributed Token Bucket with Redis using Lua (atomic):
@Service
public class RedisRateLimiter {

    private static final String LUA_SCRIPT = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refillRate = tonumber(ARGV[2])  -- tokens per second
        local now = tonumber(ARGV[3])         -- current timestamp ms
        local requested = tonumber(ARGV[4])   -- tokens requested
        
        local lastRefill = tonumber(redis.call('hget', key, 'lastRefill') or now)
        local tokens = tonumber(redis.call('hget', key, 'tokens') or capacity)
        
        -- Add tokens based on elapsed time
        local elapsed = (now - lastRefill) / 1000.0
        tokens = math.min(capacity, tokens + (elapsed * refillRate))
        
        local allowed = 0
        if tokens >= requested then
            tokens = tokens - requested
            allowed = 1
        end
        
        redis.call('hset', key, 'tokens', tokens, 'lastRefill', now)
        redis.call('expire', key, 3600)
        
        return {allowed, math.floor(tokens)}
        """;
    
    private final RedisScript<List<Long>> script = 
        RedisScript.of(LUA_SCRIPT, List.class);
    
    public RateLimitResult tryConsume(String userId, int capacity, int refillRatePerSec) {
        String key = "ratelimit:" + userId;
        long now = System.currentTimeMillis();
        
        List<Long> result = redisTemplate.execute(script,
            List.of(key),
            String.valueOf(capacity),
            String.valueOf(refillRatePerSec),
            String.valueOf(now),
            "1"
        );
        
        boolean allowed = result.get(0) == 1L;
        long remaining = result.get(1);
        return new RateLimitResult(allowed, remaining, 0);
    }
}
```

**Why Lua?** Redis executes Lua scripts atomically — the read-modify-write of the token count happens without race conditions.

---

## 🔬 Rate Limiting Granularity

Not all rate limits are equal. Layer them for defense in depth:

```
GRANULARITY LAYERS (most to least specific):

1. GLOBAL LIMIT     — Total requests across ALL clients: 100,000 req/s
   Protects server capacity regardless of who's asking

2. IP LIMIT         — Per IP: 1,000 req/min (anti-DDoS)
   Stops volumetric attacks from single sources

3. API KEY LIMIT    — Per API key: 100 req/min (standard), 1,000 (premium)
   Business tier enforcement — monetization lever

4. USER LIMIT       — Per authenticated user: 60 req/min
   Fair use per account (prevents scraping under legitimate keys)

5. ENDPOINT LIMIT   — Per specific endpoint:
   - POST /auth/login: 5 req/min (brute force protection)
   - GET /search: 30 req/min (expensive queries)
   - POST /emails/send: 10/hour (cost control)

SPRING BOOT MULTI-TIER CONFIGURATION:
@Component
public class MultiTierRateLimiter {
    
    public boolean isAllowed(HttpServletRequest request, String userId) {
        String ip = getClientIP(request);
        String endpoint = request.getRequestURI();
        
        // Check all tiers — reject if ANY limit exceeded:
        return globalLimiter.tryConsume("global")
            && ipLimiter.tryConsume("ip:" + ip)
            && (userId == null || userLimiter.tryConsume("user:" + userId))
            && endpointLimiter.tryConsume("endpoint:" + endpoint + ":" + ip);
    }
}
```

---

## 💻 Spring Boot Implementation

```java
// Complete rate limiting filter with proper headers:
@Component
@Order(1) // Apply before authentication
public class RateLimitingFilter extends OncePerRequestFilter {
    
    private final RedisRateLimiter rateLimiter;
    
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        String key = extractRateLimitKey(req);
        RateLimitResult result = rateLimiter.tryConsume(key, 100, 2); // 100 cap, 2/s refill
        
        // Always set rate limit headers (even when allowed):
        res.setHeader("X-RateLimit-Limit", "100");
        res.setHeader("X-RateLimit-Remaining", String.valueOf(result.remaining()));
        res.setHeader("X-RateLimit-Reset", 
            String.valueOf(Instant.now().plusSeconds(60).getEpochSecond()));
        
        if (!result.allowed()) {
            res.setStatus(429);
            res.setContentType("application/json");
            res.setHeader("Retry-After", String.valueOf(result.retryAfterMs() / 1000));
            res.getWriter().write("""
                {
                  "error": "RATE_LIMIT_EXCEEDED",
                  "message": "Too many requests. Please slow down.",
                  "retryAfterSeconds": %d
                }
                """.formatted(result.retryAfterMs() / 1000));
            return;
        }
        
        chain.doFilter(req, res);
    }
    
    private String extractRateLimitKey(HttpServletRequest req) {
        // Prefer authenticated user ID over IP (more stable, harder to spoof)
        String authHeader = req.getHeader("Authorization");
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            // Extract user ID from JWT without full validation (fast path)
            return "user:" + extractSubjectFromJwt(authHeader.substring(7));
        }
        // Fall back to IP-based limiting for unauthenticated requests
        return "ip:" + getClientIP(req);
    }
    
    private String getClientIP(HttpServletRequest req) {
        String forwardedFor = req.getHeader("X-Forwarded-For");
        if (forwardedFor != null && !forwardedFor.isBlank()) {
            return forwardedFor.split(",")[0].trim(); // First IP = original client
        }
        return req.getRemoteAddr();
    }
}
```

---

## 📬 HTTP Headers

Standard rate limit headers that API clients expect:

```
RESPONSE HEADERS:
─────────────────────────────────────────────────────────────────────
X-RateLimit-Limit: 100          Max requests per window
X-RateLimit-Remaining: 42       Requests left in current window
X-RateLimit-Reset: 1718000000   Unix timestamp when window resets
Retry-After: 30                 Seconds to wait (only in 429 responses)

DRAFT IETF STANDARD HEADERS (newer, use alongside traditional):
RateLimit-Limit: 100            Same as X-RateLimit-Limit
RateLimit-Remaining: 42         Same as X-RateLimit-Remaining
RateLimit-Reset: 30             Seconds until reset (relative, not absolute)

WHY THESE MATTER:
  Good API clients use these to implement backoff:
  if (response.status == 429) {
      sleep(response.headers['Retry-After'] * 1000);
      retry();
  }
  Without them, clients retry immediately → amplifying the problem
```

---

## 💡 Interview Q&A

**Q1: Which rate limiting algorithm would you use for a production API? Why?**
```
Token bucket is the industry default (used by Stripe, GitHub, Twitter) because:
1. Allows short bursts — real users/apps naturally send bursts (page load = 10 API calls)
2. Enforces sustained rate — bot scrapers maintaining 100 req/s are blocked
3. Simple to understand — easy to explain limits to customers
4. O(1) per request — just check and decrement a counter

Sliding window is better when exact fairness matters (shared B2B APIs where
each customer pays for a precise quota). It's also harder to game by timing bursts.

Leaky bucket is for controlling OUTBOUND calls — when you're calling a third-party
API with strict limits and need to smooth your traffic.
```

**Q2: How do you implement rate limiting in a load-balanced environment?**
```
Centralized counter store (Redis recommended):
- All app servers increment the same Redis key atomically
- Use Redis INCR + EXPIRE, or Lua scripts for token bucket logic
- Redis is fast enough: sub-millisecond per operation, 100k+ ops/sec

Alternatives:
1. API Gateway (Kong, AWS API Gateway) — offload rate limiting entirely
   Pros: Not in application code, scales independently
   Cons: Less flexibility, another system to manage

2. Sticky sessions — route user X always to server X (via load balancer)
   Pros: Local counters work, no Redis needed
   Cons: Session affinity complicates deployment, single server failure = disruption

3. Approximate local + periodic sync — local counter, sync to Redis every 100ms
   Pros: Ultra-low latency for the common case
   Cons: Small window of over-allowance during sync lag

Production answer: Use Redis with Lua atomic scripts. It's the industry standard.
```

**Q3: What HTTP status code does rate limiting return, and what should clients do?**
```
429 Too Many Requests (RFC 6585)

The response MUST include:
  Retry-After: 60  (seconds until the client should retry)

Clients should implement exponential backoff:
  Wait = min(retryAfter, 2^attempt * baseDelay + random(0, baseDelay))

What clients MUST NOT do:
  - Retry immediately (amplifies the problem)
  - Ignore the Retry-After header
  - Treat 429 as a fatal error (it's transient by design)
```

**Q4: How do you rate limit specific expensive endpoints differently from regular ones?**
```java
// Endpoint-specific limits using annotations:
@RateLimit(limit = 5, window = 60, unit = SECONDS)  // Custom annotation
@PostMapping("/auth/login")
public LoginResponse login(@RequestBody LoginRequest req) { ... }

@RateLimit(limit = 30, window = 60, unit = SECONDS)
@GetMapping("/search")
public SearchResponse search(@RequestParam String query) { ... }

@RateLimit(limit = 10, window = 3600, unit = SECONDS)  // Very strict hourly
@PostMapping("/reports/generate")
public Report generateReport(@RequestBody ReportRequest req) { ... }

// Implement with AOP:
@Around("@annotation(rateLimit)")
public Object enforceRateLimit(ProceedingJoinPoint jp, RateLimit rateLimit) throws Throwable {
    String key = "endpoint:" + jp.getSignature().getName() + ":" + currentUser();
    // ... check rate limit, throw 429 if exceeded
}
```

---

## 🎲 Mini Challenge

> 🎲 **CHALLENGE** (8 minutes):
> Design a rate limiting system for a social media API with these requirements:
> - Free tier: 100 requests per minute, no burst
> - Pro tier: 1,000 requests per minute, 50/second burst allowed  
> - Admin: Unlimited
> - POST /posts/create: max 5 per minute per user regardless of tier
> - All tiers: share a global server limit of 500,000 req/min

<details>
<summary>💡 Click to reveal solution architecture</summary>

```java
enum Tier { FREE, PRO, ADMIN }

@Service
public class SocialApiRateLimiter {
    
    private final Map<Tier, BucketConfig> tierConfigs = Map.of(
        Tier.FREE, new BucketConfig(100, 100, Duration.ofMinutes(1), 100),
        Tier.PRO,  new BucketConfig(1000, 50, Duration.ofMinutes(1), 50),
        Tier.ADMIN, null  // No limit
    );
    
    // 3 Redis key spaces:
    // "global"           — shared server limit
    // "user:{id}"        — per-user tier limit
    // "endpoint:{method}:{path}:{userId}" — per-endpoint limit
    
    public RateLimitDecision isAllowed(String userId, Tier tier, String endpoint) {
        if (tier == Tier.ADMIN) return RateLimitDecision.ALLOWED;
        
        // Check global server limit (500k/min)
        if (!globalBucket.tryConsume(1)) 
            return RateLimitDecision.rejected("Global capacity reached");
        
        // Check user tier limit
        Bucket userBucket = getUserBucket(userId, tierConfigs.get(tier));
        if (!userBucket.tryConsume(1)) 
            return RateLimitDecision.rejected("User rate limit exceeded");
        
        // Check endpoint-specific limit (POST /posts/create = 5/min)
        if ("POST /posts/create".equals(endpoint)) {
            Bucket postBucket = getPostCreationBucket(userId);
            if (!postBucket.tryConsume(1)) 
                return RateLimitDecision.rejected("Post creation limit: 5/minute");
        }
        
        return RateLimitDecision.ALLOWED;
    }
    
    private Bucket getUserBucket(String userId, BucketConfig config) {
        // Use Redis for distributed state
        return redisBackedBuckets.computeIfAbsent(userId, id -> 
            Bucket.builder()
                .addLimit(Bandwidth.classic(config.capacity(),
                    Refill.intervally(config.capacity(), config.window())))
                .addLimit(Bandwidth.classic(config.burstCapacity(),
                    Refill.greedy(config.burstCapacity(), Duration.ofSeconds(1))))
                .build()
        );
    }
}
```

**Key design decisions:**
- Tiers are stored in the JWT claims → no DB lookup per request
- Multiple `Bandwidth` limits in Bucket4j enforce BOTH sustained AND burst
- Admin bypass at the first check — no Redis hit needed
- Global limit uses a separate Bucket shared across all instances

</details>

---

## 🏢 Industry Applications

| Company | Rate Limiting Approach | Limits |
|---------|----------------------|--------|
| GitHub API | Token bucket per OAuth app | 5,000 req/hour authenticated, 60 unauthenticated |
| Twitter/X API | Sliding window per endpoint | 300 tweets/3h, 900 lookups/15min |
| Stripe API | Leaky bucket (smooth) | 100 req/s test, 25 req/s live (some endpoints) |
| Twilio | Fixed window with retry headers | 1 req/s SMS by default |
| OpenAI API | Token bucket on tokens (not requests) | Varies by model and tier |

---

## 🔗 What to Read Next

| Article | Why |
|---------|-----|
| [API Security](./API_Security.md) | Rate limiting is one layer — see the full picture |
| [Caching Strategies](../BuildingBlocks/CachingStrategies.md) | Rate limit state often lives in cache |
| [Redis Fundamentals](../BuildingBlocks/Redis.md) | Understand the distributed counter store |
| [System Design: Design a Rate Limiter](../SystemDesignCaseStudies/) | Full system design interview answer |

---

*Previous: [← API Security](./API_Security.md) | Next: [Back to API Overview →](./README.md)*
