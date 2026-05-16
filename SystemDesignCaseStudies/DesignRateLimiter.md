# ⏱️ Design a Rate Limiter
## Token Buckets, Sliding Windows, and Protecting Your APIs at Scale

> *"A public API without rate limiting is like a bank with no daily withdrawal limit. A single customer with bad intent — or a buggy client — can drain the whole system dry."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md), [Redis](../BuildingBlocks/Caching.md)

---

## 📋 Table of Contents
1. [Requirements (RADIO: R)](#-requirements)
2. [API Design (RADIO: A)](#-api-design)
3. [Rate Limiting Algorithms (RADIO: D)](#-rate-limiting-algorithms)
4. [Infrastructure & Architecture (RADIO: I)](#-infrastructure--architecture)
5. [Optimization (RADIO: O)](#-optimization)
6. [Mini Challenge](#-mini-challenge)
7. [Interview Q&A](#-interview-qa)

---

## 📋 Requirements

### Functional Requirements

| Feature | Description |
|---|---|
| **Limit requests** | Restrict the number of API calls a client can make per time window |
| **Granular limits** | Different limits per user, API key, IP, endpoint, or tier |
| **Flexible rules** | Configure limits without redeployment (rule-based) |
| **Informative responses** | Return headers showing remaining quota, reset time |
| **Multiple strategies** | Support token bucket, fixed window, sliding window |

### Non-Functional Requirements

```
Scale:
  - 100K+ requests/second total across all clients
  - Rate limiting decision must add < 5ms latency to every request
  - Rules apply across multiple API server instances (distributed enforcement)
  - 99.99% availability — rate limiter must not be the system's single point of failure

Consistency:
  - Approximate enforcement is acceptable (we can allow slight over-limit in race conditions)
  - Being 5% over the limit is better than 50% false-rejections
  
Flexibility:
  - Per-user limits (free: 100/min, pro: 1000/min, enterprise: 10000/min)
  - Per-endpoint limits (/search: 10/min, /upload: 5/min, /login: 5/min)
  - Per-IP limits to block DDoS
  - Burst allowance (allow 2× for 10 seconds, then enforce strictly)
```

---

## 📡 API Design

Rate limiting is a **cross-cutting concern** — implemented in API Gateway or middleware, not in business services. But the response format matters:

```
All responses include rate limit headers:
  X-RateLimit-Limit:     100         ← maximum requests per window
  X-RateLimit-Remaining: 73          ← requests remaining this window
  X-RateLimit-Reset:     1747488000  ← Unix timestamp when window resets
  X-RateLimit-Policy:    100;w=60    ← 100 requests per 60-second window

When limit is exceeded:
  HTTP 429 Too Many Requests
  Retry-After: 37                    ← seconds until the client can retry
  Content-Type: application/json
  {
    "error": "rate_limit_exceeded",
    "message": "You have exceeded 100 requests per minute. Please retry after 37 seconds.",
    "limit": 100,
    "window": "60s",
    "reset_at": "2026-05-17T10:31:00Z"
  }
```

### Rate Limit Rule Configuration API (internal/admin)

```
POST /admin/rate-limit-rules
  {
    "name": "pro-tier-global",
    "match": { "tier": "pro" },
    "limit": 1000,
    "window": 60,           // seconds
    "algorithm": "sliding_window_log",
    "burst_multiplier": 2,  // allow 2000/min burst for 10s
    "priority": 10          // higher = evaluated first
  }
```

---

## 🧮 Rate Limiting Algorithms

### Algorithm 1: Fixed Window Counter

```
Concept:
  Divide time into fixed windows (e.g., each minute: 14:00:00-14:01:00).
  Count requests per client per window.
  Reject if count > limit.

  14:00:00 ─────────── 14:01:00 ─────────── 14:02:00
  │   Window 1 (100)   │   Window 2 (100)   │
  │  ████████ 100 OK   │  ████████ 100 OK   │

Redis implementation:
  key = "ratelimit:{user_id}:{minute_unix_timestamp}"
  INCR key
  EXPIRE key 60
  if count > 100: reject

Problems:
  Boundary attack: 100 requests at 14:00:59, 100 at 14:01:01 = 200 requests
  in a 2-second window — but both windows are "OK" per counter.

  │ Window 1 │     Window 2     │
  │    100 ███│ 100 ██           │
  └──────────┘───────────────────┘
                ↑ 200 requests in 2 seconds!
```

### Algorithm 2: Sliding Window Log

```
Concept:
  For each request, store a timestamp in a sorted set.
  Count timestamps within the sliding window [now - window, now].
  Reject if count > limit.

Redis implementation (per user):
  key = "ratelimit:sliding:{user_id}"
  
  On each request:
    now = current_timestamp_ms
    window_start = now - 60000  (60 seconds ago)
    
    MULTI
      ZREMRANGEBYSCORE key 0 window_start  # remove old entries
      ZADD key now now                      # add current timestamp
      count = ZCARD key                     # count in window
      EXPIRE key 60
    EXEC
    
    if count > 100: reject

Advantages: Accurate, no boundary spike problem
Disadvantages: High memory (stores every request timestamp); 
               at 100 req/min × 1M users = 100M stored timestamps
```

### Algorithm 3: Sliding Window Counter (Best Balance)

```
Concept:
  Combine fixed window efficiency with sliding accuracy.
  Count requests in current window + weighted fraction of previous window.
  
  estimate_count = 
    current_window_count + 
    previous_window_count × (window_remaining / window_size)

Example:
  Window: 60 seconds, limit: 100
  Current minute (50s elapsed): 70 requests
  Previous minute: 90 requests
  
  estimate = 70 + 90 × (60-50)/60
           = 70 + 90 × 0.167
           = 70 + 15 = 85 ✅ under limit

Redis (2 counters per user, very memory efficient):
  current_key  = "ratelimit:{user_id}:{current_minute}"
  previous_key = "ratelimit:{user_id}:{previous_minute}"
  
  current_count  = INCR current_key, EXPIRE current_key 120
  previous_count = GET previous_key OR 0
  elapsed_ratio  = (current_second_within_minute) / 60
  estimate = current_count + previous_count × (1 - elapsed_ratio)
  
  if estimate > 100: reject (and roll back: DECR current_key)

Best for: General purpose rate limiting — accurate, memory-efficient, fast.
```

### Algorithm 4: Token Bucket

```
Concept:
  Each user has a "bucket" holding tokens (capacity = burst limit).
  Tokens refill at a constant rate (e.g., 10 tokens/second).
  Each request consumes 1 token.
  If no tokens available → reject.
  
  Capacity = 100 (max burst)
  Refill rate = 10 tokens/second
  
  Time 0:    bucket = 100 (full)
  10 burst requests arrive: bucket = 90
  ...
  100 burst requests: bucket = 0, all 100 served
  101st request: rejected (no tokens)
  Wait 0.1s: 1 token refills → request allowed
  Wait 10s:  bucket = 100 again (full)
  
  Allow bursts above steady-state rate, but enforce long-term average.

Redis implementation (atomic with Lua script):
  tokens = GET bucket_key OR capacity
  last_refill = GET refill_key OR now
  
  elapsed = now - last_refill
  tokens = min(capacity, tokens + elapsed × refill_rate)
  
  if tokens >= 1:
    tokens -= 1
    SET bucket_key tokens
    SET refill_key now
    allow request
  else:
    reject

Best for: APIs where bursts are acceptable (CDN, uploads, search).
```

### Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst Handling | Complexity |
|---|---|---|---|---|
| **Fixed Window** | O(1) | Low (boundary spike) | No | Very Low |
| **Sliding Window Log** | O(requests in window) | Perfect | No | Medium |
| **Sliding Window Counter** | O(1) | Good (~boundary spike) | No | Low |
| **Token Bucket** | O(1) | Good | ✅ Yes | Medium |
| **Leaky Bucket** | O(queue) | Good | ✅ Smoothed | Medium |

---

## 🏗️ Infrastructure & Architecture

### Where to Implement Rate Limiting

```
Option 1: API Gateway (recommended)
  Nginx, Kong, AWS API Gateway, Envoy
  Pros: centralized, zero app code changes, works for all services
  Cons: single point of failure if gateway goes down (mitigate with HA)

Option 2: Service-Level Middleware
  Spring Boot filter, Express middleware
  Pros: fine-grained per-endpoint control, no gateway required
  Cons: logic duplicated across services, misses traffic that bypasses the service

Option 3: Dedicated Rate Limit Service
  Separate microservice (e.g., Envoy's ratelimit service, Lyft's ratelimit)
  Called by API Gateway via gRPC on every request
  Pros: centralized logic, language-agnostic, highly configurable
  Cons: added latency (gRPC call), another service to operate

Best practice: API Gateway + Redis (co-located, same datacenter)
  Request → API Gateway → Redis (check limit, ~1ms) → Backend service
```

### Full Architecture Diagram

```
┌───────────────────────────────────────────────────────────────────┐
│                   RATE LIMITER ARCHITECTURE                       │
│                                                                   │
│  Client                                                           │
│    │                                                              │
│    ▼                                                              │
│  [Load Balancer]  (distributes across API gateway instances)      │
│    │                                                              │
│    ▼                                                              │
│  [API Gateway Cluster] (Kong / Nginx / Envoy)                    │
│    │  1. Extract identifier (user_id from JWT, IP, API key)       │
│    │  2. Determine applicable rules (from config cache)           │
│    │  3. Call Redis: check/increment counter (Lua script)         │
│    │  4. If allowed: forward to backend                          │
│    │     If rejected: return HTTP 429                             │
│    │                                                              │
│  ┌─┴─────────────────────────────────────────────┐               │
│  │  [Redis Cluster]                               │               │
│  │  - Sharded by user_id (consistent hashing)     │               │
│  │  - Rate limit counters (expire automatically)  │               │
│  │  - Sliding window state                        │               │
│  │  - Replication: 1 primary + 2 replicas/shard   │               │
│  └───────────────────────────────────────────────┘               │
│                                                                   │
│  [Rules Config Service] → pushes rules to gateway config cache    │
│  [Analytics Service]    ← reads from Kafka (sampled limit events) │
└───────────────────────────────────────────────────────────────────┘
```

### Distributed Rate Limiting: The Race Condition Problem

```
Problem: 3 API gateway instances, user makes 3 concurrent requests

Without synchronization:
  Instance 1: reads count=99, increments to 100 → ALLOW
  Instance 2: reads count=99, increments to 100 → ALLOW  (reads stale value!)
  Instance 3: reads count=99, increments to 100 → ALLOW  (3× over limit allowed!)

Solution: Atomic Redis operations
  Use Lua script (executes atomically on Redis — no race conditions):

local current = redis.call('INCR', KEYS[1])
if current == 1 then
    redis.call('EXPIRE', KEYS[1], tonumber(ARGV[1]))  -- set TTL on first increment
end
if current > tonumber(ARGV[2]) then
    return 0  -- over limit
else
    return 1  -- allowed
end

  EVAL lua_script 1 "ratelimit:user123:1747488000" 60 100
  
  Atomic: read + increment + check happens as one operation.
  No race condition possible within a single Redis node.
```

---

## 📈 Optimization

### Redis at 100K RPS — Scaling Strategy

```
Single Redis: handles ~100K operations/second (each request = 1 Redis operation)
→ At 100K requests/second, we're at Redis's limit.

Strategy 1: Client-side caching
  API gateway holds local token bucket state in memory (per gateway instance).
  Sync to Redis every 100ms (not per request).
  Trade-off: can overshoot limit by N_gateways × local_bucket_size.
  Acceptable for approximate enforcement.

Strategy 2: Redis Cluster sharding
  Shard by user_id: user_hash % N_shards → specific Redis shard
  100 shards × 100K ops/shard = 10M ops/second capacity
  Add shards as traffic grows

Strategy 3: Two-tier limiting
  Tier 1 (per API gateway instance, in-memory): coarse limit, no Redis
    Reject obviously excessive requests (>10x their limit) locally.
  Tier 2 (Redis): precise limit for borderline cases.
  90% of requests are far below limits → most never hit Redis.
```

### Handling Redis Failures (Failopen vs Failclosed)

```
Redis goes down: what do you do?

Option A: FAIL OPEN (allow all requests)
  Pros: 100% availability, customers never impacted by rate limiter outage
  Cons: during Redis failure, rate limiting is disabled — potential abuse window

Option B: FAIL CLOSED (reject all requests / return 503)
  Pros: security maintained
  Cons: complete outage of your API for all customers — terrible UX

Recommended: FAIL OPEN with local fallback
  If Redis is unreachable:
    Use in-memory counter (per gateway instance) with conservative limit
    (e.g., 50% of normal limit → prevents total abuse without full outage)
  Set Redis timeout low (100ms) → fail fast, don't hold up requests
  Alert immediately when Redis failover happens
```

---

## 🧩 Mini Challenge

**Design a rate limiter for a login endpoint with these requirements**:
- Max 5 failed login attempts per IP per 15 minutes
- After 5 failures: block that IP for 15 minutes
- Successful login resets the counter
- Must handle 10K login attempts/second

**Which algorithm, which Redis data structure, and what's the key design?**

<details>
<summary>💡 Click to reveal answer</summary>

**Algorithm**: Sliding Window Counter — we need accurate enforcement for security (no boundary spikes that could allow 10 attempts in 2 seconds at window boundaries).

**Redis data structure**: Sorted Set (for sliding window log) — but since login failures are rare compared to successes, and we need per-IP tracking with expiry, HASH is also acceptable with a different approach.

**Key design**:

```lua
-- Lua script: atomic check and increment for failed login
local key = "login_failures:" .. ARGV[1]  -- ARGV[1] = IP address
local window = 900  -- 15 minutes in seconds
local max_failures = 5

-- Check if IP is currently blocked
local blocked_key = "login_block:" .. ARGV[1]
if redis.call('EXISTS', blocked_key) == 1 then
    return -1  -- blocked
end

-- Count recent failures using sorted set
local now = tonumber(ARGV[2])  -- current timestamp
local window_start = now - window

-- Remove old entries
redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

-- Count failures in window
local failure_count = redis.call('ZCARD', key)

if failure_count >= max_failures then
    -- Block the IP
    redis.call('SET', blocked_key, 1, 'EX', window)
    return -1  -- blocked
end

-- Allow (don't increment yet — call only on actual failure)
return failure_count
```

**On failed login attempt**:
```lua
ZADD "login_failures:{ip}" {timestamp} {timestamp}
EXPIRE "login_failures:{ip}" 900
```

**On successful login**:
```lua
DEL "login_failures:{ip}"  -- reset counter on success
```

**Serving 10K login attempts/second**: Redis Cluster sharded by IP address. Each shard handles ~100-1000 IPs; consistent hashing routes requests. Two sorted set operations per request (ZREMRANGEBYSCORE + ZCARD = ~0.1ms). 10K ops/second is well within single Redis node capacity (~100K ops/sec). Only failed attempts are recorded (much less than 10K/sec in normal operation).

**Additional defense**: Combine with CAPTCHA after 3 failures, and notify the account owner via email when failures occur.

</details>

---

## 📝 Interview Q&A

**Q: What are the trade-offs between the token bucket and sliding window algorithms?**
> A: Token bucket allows controlled bursts — a user who hasn't made requests recently can make many requests in quick succession (up to bucket capacity), then falls back to the steady refill rate. It's ideal for bursty traffic like CDN or batch APIs. Sliding window log is more restrictive — no burst allowance; requests are spread evenly over time. It's better for endpoints where bursts are genuinely problematic (login endpoints, payment APIs). Sliding window counter offers a practical middle ground: O(1) memory like token bucket, better accuracy than fixed window, but no burst handling.

**Q: How do you implement rate limiting in a distributed environment where multiple servers share state?**
> A: Use Redis as the shared counter store with atomic Lua scripts. A Lua script executes atomically on Redis — the read-increment-check is one operation, eliminating race conditions between distributed servers. The key is `{identifier}:{time_window}` (e.g., `ratelimit:user123:14:23` for minute 14:23). All API gateway instances query the same Redis cluster, ensuring consistent enforcement. For Redis failures, implement a fail-open strategy with in-memory fallback counters to maintain availability.

**Q: How does rate limiting at the API gateway differ from rate limiting inside each microservice?**
> A: API Gateway rate limiting: (1) Enforces before requests reach any service — saves backend compute on rejected requests; (2) Centralized, no code duplication; (3) Works for all services uniformly; (4) Can rate limit on any attribute available at the gateway (IP, JWT claims, API key). Service-level rate limiting: (1) More granular — can limit per specific endpoint or feature flag; (2) Applies to internal service calls (not just external); (3) Required if services are called directly (bypassing gateway). Best practice: use both — gateway for coarse-grained user/IP protection, service-level for fine-grained endpoint protection and internal call protection.

---

## 🔗 What to Read Next

1. **[SystemDesignCaseStudies/DesignPaymentSystem.md](./DesignPaymentSystem.md)** — Payment APIs require strict rate limiting to prevent fraud
2. **[BuildingBlocks/APIGateway.md](../BuildingBlocks/APIGateway.md)** — Rate limiting is a core API Gateway feature; see the bigger architectural context
3. **[BuildingBlocks/Caching.md](../BuildingBlocks/Caching.md)** — Redis is the backbone of distributed rate limiting; master it

---

*[← Design Payment System](./DesignPaymentSystem.md) | [Back to Case Studies](./README.md)*
