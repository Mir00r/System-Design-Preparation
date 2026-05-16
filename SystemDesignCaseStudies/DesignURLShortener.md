# 🔗 Design a URL Shortener
## Like bit.ly — From Zero to Production in 45 Minutes

> *"The URL shortener is the 'Hello World' of system design. Master it and you've learned hashing, caching, databases, and scaling in one shot."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Requirements (R)](#-requirements-r)
3. [API Design (A)](#-api-design-a)
4. [Data Model (D)](#-data-model-d)
5. [Infrastructure (I)](#-infrastructure-i)
6. [Optimization (O)](#-optimization-o)
7. [Code Examples](#-code-examples)
8. [Industry Examples](#-industry-examples)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

It's 2010. Someone shares a URL in a tweet:

```
https://www.verylongwebsite.com/articles/this-is-an-extremely-long-article-title-that-goes-on/
and-on-and-this-is-the-full-path?utm_source=twitter&utm_medium=social&utm_campaign=2010spring
```

That's 200+ characters — Twitter's limit was 140. They need:
```
https://bit.ly/3xK9pQ
```

22 characters. Same destination.

**The challenge**: When millions of people click `bit.ly/3xK9pQ` per second, the system must redirect them in under **10 milliseconds** — while also storing **billions** of short-to-long URL mappings durably.

---

## 📐 Requirements (R)

### Functional Requirements
```
Core features:
  ✅ Create a short URL from a long URL
  ✅ Redirect short URL to original long URL
  ✅ Optional: custom alias (e.g., bit.ly/my-company)
  ✅ Optional: URL expiration (TTL)

Out of scope (clarify with interviewer):
  ❌ User accounts / authentication (unless asked)
  ❌ Analytics dashboard (click tracking)
  ❌ QR code generation
```

### Non-Functional Requirements
```
Scale:
  - 100M URLs created per day (write: ~1,200/sec, peak ~5K/sec)
  - 10B redirects per day (read: ~115K/sec, peak ~500K/sec)
  - Read:Write ratio = 100:1 (extremely read-heavy)

Performance:
  - Redirect latency: < 10ms (P99)
  - URL creation: < 100ms

Reliability:
  - 99.99% availability for redirects
  - URLs must NEVER be lost or corrupted

Storage:
  - 1 URL record ≈ 500 bytes (long URL + metadata)
  - 100M new URLs/day × 365 days × 5 years = 182 billion records
  - 182B × 500 bytes ≈ 91 TB total storage (over 5 years)
```

---

## 🌐 API Design (A)

```
POST /api/v1/urls
  Request:  { "longUrl": "https://...", "alias": "optional", "expiresAt": "optional ISO8601" }
  Response: { "shortUrl": "https://short.ly/abc123", "shortCode": "abc123", "expiresAt": "..." }
  Errors:   400 (invalid URL), 409 (alias taken), 429 (rate limited)

GET /{shortCode}
  Response: HTTP 301 Redirect → Location: https://original-url.com/...
  Errors:   404 (not found), 410 (expired)

DELETE /api/v1/urls/{shortCode}
  Auth:     required (only creator can delete)
  Response: HTTP 204 No Content
```

**301 vs 302 Redirect — Critical decision**:
```
301 Permanent Redirect:
  → Browser caches the redirect
  → Future clicks go DIRECTLY to destination (bypass our servers)
  → Pros: Less load on our servers
  → Cons: Can't track clicks, can't update destination

302 Temporary Redirect:
  → Browser always asks our server
  → Every click goes through us
  → Pros: Analytics, A/B testing, can change destination
  → Cons: More server load

✅ Use 302 if analytics matter. Use 301 if raw performance matters.
```

---

## 🗄️ Data Model (D)

### Storage Choice

```
Why NOT relational for the redirect table?
  - We need key-value lookup: shortCode → longUrl
  - No complex joins or relational queries
  - Need horizontal scaling (billions of records)

Why NOT pure in-memory (Redis alone)?
  - 91 TB doesn't fit in RAM alone
  - Need durability — URLs can't vanish

✅ Solution: PostgreSQL (primary store) + Redis (cache hot URLs)
   OR: DynamoDB (fully managed, auto-scaling key-value)
```

### Schema

```sql
-- URLs table (primary store)
CREATE TABLE urls (
    short_code   VARCHAR(8)   PRIMARY KEY,  -- e.g., "abc123xy"
    long_url     TEXT         NOT NULL,
    user_id      UUID,                       -- nullable if anonymous
    created_at   TIMESTAMP    NOT NULL DEFAULT NOW(),
    expires_at   TIMESTAMP,                  -- NULL = never expires
    click_count  BIGINT       DEFAULT 0
);

-- Index for user's URLs (list my short URLs feature)
CREATE INDEX idx_urls_user_id ON urls(user_id, created_at DESC);

-- Index to quickly check expired URLs
CREATE INDEX idx_urls_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;
```

### Short Code Format

```
Base62 encoding: [0-9][a-z][A-Z] = 62 characters

6 characters → 62^6 = 56 billion unique codes   ✅ (enough for years)
7 characters → 62^7 = 3.5 trillion unique codes  ✅✅

✅ Use 7-character base62 codes = 3.5 trillion URLs
```

---

## 🏗️ Infrastructure (I)

### High-Level Architecture

```
                    ┌─────────────────────────┐
                    │         Client           │
                    │   (Browser / App / API) │
                    └────────────┬────────────┘
                                 │ HTTPS
                    ┌────────────▼────────────┐
                    │        Cloudflare       │
                    │    (DNS + CDN + DDoS)   │
                    └────────────┬────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      Load Balancer      │
                    │   (AWS ALB / Nginx)     │
                    └──────┬───────┬──────────┘
                           │       │
               ┌───────────▼─┐   ┌▼────────────┐
               │ URL Service │   │ URL Service  │
               │  (Instance 1)│   │  (Instance 2)│
               └──────┬──────┘   └──────┬───────┘
                      │                 │
         ┌────────────▼─────────────────▼────────────┐
         │              Redis Cluster                 │
         │         (Hot URL Cache — 100ms TTL)        │
         └────────────────────┬───────────────────────┘
                              │ cache miss
         ┌────────────────────▼───────────────────────┐
         │            PostgreSQL / DynamoDB            │
         │          (Primary URL Store)                │
         │   [Primary] ←→ [Read Replica] [Read Replica]│
         └────────────────────────────────────────────┘
```

### Redirect Request Flow (Critical Path)

```
GET https://short.ly/abc123

Step 1: DNS resolves short.ly → Cloudflare edge
Step 2: Cloudflare checks its own cache (for popular URLs)
Step 3: If miss → forwards to Load Balancer
Step 4: Load Balancer routes to URL Service instance
Step 5: URL Service checks Redis cache
          HIT:  fetch longUrl → return 302 Redirect   ← ~1-2ms
          MISS: query PostgreSQL → cache in Redis → 302 Redirect  ← ~5-10ms
Step 6: Browser follows redirect → loads destination
```

### URL Creation Flow

```
POST /api/v1/urls { longUrl: "https://..." }

Step 1: Validate URL (is it a valid URL? Is it blocklisted?)
Step 2: Generate short code (see strategies below)
Step 3: Check for collision in DB
Step 4: Insert into PostgreSQL
Step 5: Return shortUrl to client
```

### Short Code Generation — Two Approaches

```
Strategy 1: Hash-Based
  shortCode = base62(md5(longUrl + salt)[0:7])
  Pros: Same long URL → same short code (deduplication)
  Cons: Collision possible, birthday paradox

Strategy 2: Counter-Based (Snowflake-style)
  shortCode = base62(global_counter.increment())
  Pros: No collisions guaranteed, sequential
  Cons: Single counter = bottleneck (use distributed ID like Twitter Snowflake)

✅ Production choice: Snowflake-style distributed ID
  - Each service instance has a unique worker_id (0-1023)
  - ID = timestamp_ms(41 bits) + worker_id(10 bits) + sequence(12 bits)
  - Encode final ID in base62 for human-readable short code
```

---

## ⚡ Optimization (O)

### Bottleneck 1: Read Load (115K redirects/sec)

```
Problem: 115K reads/sec overwhelms any single database

Solution: Multi-layer caching
  Layer 1: Cloudflare Edge Cache
    - Cache popular short codes at CDN PoPs worldwide
    - TTL = 5 minutes for regular URLs
    - Reduces ~70% of traffic before it hits our servers

  Layer 2: Redis Cache (in-process)
    - LRU eviction policy
    - Cache size: 20% of daily URLs cover 80% of traffic (Pareto principle)
    - TTL = 24 hours for fresh URLs

  Layer 3: Read Replicas
    - PostgreSQL with 2-3 read replicas
    - Cache misses fan out to replicas, not primary

Result: Primary DB sees <1% of total redirect traffic
```

### Bottleneck 2: Database Size (91 TB over 5 years)

```
Problem: 91 TB doesn't fit on one machine efficiently

Solution: Horizontal Sharding by short_code
  - Shard 0: short codes starting with [0-9, a-f]
  - Shard 1: short codes starting with [g-m]
  - Shard 2: short codes starting with [n-t]
  - Shard 3: short codes starting with [u-Z]

  Each shard handles ~22 TB — manageable on a single NVMe SSD server
  OR use DynamoDB — auto-sharding, managed service, ~$0.25/GB/month
```

### Bottleneck 3: URL Expiration Cleanup

```
Problem: Billions of expired URLs accumulate, wasting storage and slowing queries

Solution: Lazy expiration + background job
  - At redirect time: check expires_at < NOW(), return 410 Gone, mark for deletion
  - Background job (runs nightly): DELETE FROM urls WHERE expires_at < NOW() - INTERVAL '1 day'
  - Partition the table by created_at month → DROP PARTITION is O(1) vs DELETE is O(n)
```

### Bottleneck 4: Custom Alias Collision

```
Problem: Two users try to register "bit.ly/sale" simultaneously

Solution: Optimistic locking with unique constraint
  - unique constraint on short_code column
  - Try INSERT, catch unique violation, return 409 Conflict
  - No distributed lock needed — DB handles it atomically
```

---

## 💻 Code Examples

### URL Service — Core Logic (Java/Spring Boot)

```java
@RestController
@RequestMapping("/api/v1")
public class UrlController {

    @Autowired private UrlService urlService;

    @PostMapping("/urls")
    public ResponseEntity<UrlResponse> createShortUrl(@RequestBody @Valid UrlRequest request) {
        String shortCode = urlService.createShortUrl(request.getLongUrl(), request.getAlias());
        return ResponseEntity.ok(new UrlResponse("https://short.ly/" + shortCode, shortCode));
    }

    @GetMapping("/{shortCode}")
    public ResponseEntity<Void> redirect(@PathVariable String shortCode, HttpServletResponse response) {
        String longUrl = urlService.getLongUrl(shortCode);  // throws NotFoundException if not found
        return ResponseEntity.status(HttpStatus.FOUND)      // 302
                             .header("Location", longUrl)
                             .build();
    }
}

@Service
public class UrlService {

    @Autowired private UrlRepository urlRepository;
    @Autowired private RedisTemplate<String, String> redisTemplate;
    @Autowired private SnowflakeIdGenerator idGenerator;

    private static final String CACHE_PREFIX = "url:";
    private static final Duration CACHE_TTL = Duration.ofHours(24);

    public String getLongUrl(String shortCode) {
        // 1. Check Redis cache first
        String cached = redisTemplate.opsForValue().get(CACHE_PREFIX + shortCode);
        if (cached != null) {
            return cached;
        }

        // 2. Miss → query DB
        Url url = urlRepository.findByShortCode(shortCode)
                .orElseThrow(() -> new NotFoundException("Short URL not found: " + shortCode));

        // 3. Check expiration
        if (url.getExpiresAt() != null && url.getExpiresAt().isBefore(Instant.now())) {
            throw new GoneException("This URL has expired");
        }

        // 4. Cache for next time
        redisTemplate.opsForValue().set(CACHE_PREFIX + shortCode, url.getLongUrl(), CACHE_TTL);

        return url.getLongUrl();
    }

    public String createShortUrl(String longUrl, String alias) {
        // Validate URL
        if (!isValidUrl(longUrl)) {
            throw new BadRequestException("Invalid URL format");
        }

        String shortCode = (alias != null) ? alias : generateShortCode();

        Url url = Url.builder()
                .shortCode(shortCode)
                .longUrl(longUrl)
                .createdAt(Instant.now())
                .build();

        try {
            urlRepository.save(url);
        } catch (DataIntegrityViolationException e) {
            throw new ConflictException("Short code already taken: " + shortCode);
        }

        return shortCode;
    }

    private String generateShortCode() {
        long id = idGenerator.nextId();  // Snowflake-based distributed ID
        return Base62.encode(id);        // convert to 7-char base62 string
    }

    private boolean isValidUrl(String url) {
        try {
            new URL(url).toURI();
            return url.startsWith("http://") || url.startsWith("https://");
        } catch (Exception e) {
            return false;
        }
    }
}
```

### Base62 Encoder

```java
public class Base62 {
    private static final String CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public static String encode(long id) {
        if (id == 0) return "0";
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(CHARS.charAt((int)(id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
    }

    public static long decode(String code) {
        long result = 0;
        for (char c : code.toCharArray()) {
            result = result * 62 + CHARS.indexOf(c);
        }
        return result;
    }
}
```

---

## 🏢 Industry Examples

| Company | Tech Stack Highlights |
|---|---|
| **bit.ly** | Custom short codes, click analytics per URL, geographic breakdown. Uses MySQL + Cassandra + Redis. |
| **t.co (Twitter)** | All URLs in tweets are automatically wrapped. Scans for malware before redirecting. Serves ~5B redirects/day. |
| **TinyURL** | One of the originals (2002). Simple hash-based approach. Unlimited free tier. |
| **Cloudflare Workers** | Serverless URL shortener with globally distributed KV store — sub-millisecond redirects at any edge PoP. |

---

## ⚠️ Common Pitfalls

1. **Using MD5/SHA1 for short codes** — Hash collisions are real. If `hash(urlA)[0:7] == hash(urlB)[0:7]`, you have a bug. Use a counter-based ID generator or handle collisions explicitly.

2. **301 redirect without thinking** — If you use 301, browsers cache the redirect forever. You can never change the destination or track clicks. Always confirm requirements.

3. **No URL validation** — Without checking the long URL format, users can inject `javascript:` URLs and create phishing vectors. Always validate scheme is `http://` or `https://`.

4. **Forgetting URL blocklist** — Services like bit.ly maintain a blocklist of malware/phishing domains. Without it, your service becomes a phishing delivery mechanism.

5. **Sequential IDs leaking information** — If short codes are sequential (0000001, 0000002...), competitors can enumerate your entire database. Shuffle the ID space or use a random salt.

---

## 🧩 Mini Challenge

**Design question**: Your URL shortener is getting 500K redirects/sec during a Super Bowl ad campaign. Your Redis cluster is showing 95% cache hit rate, but the 5% misses are overwhelming PostgreSQL.

**What are your top 2 immediate actions and 1 longer-term fix?**

<details>
<summary>💡 Click to reveal answer</summary>

**Immediate actions (next 15 minutes)**:

1. **Increase Redis cache TTL** — Change from 24 hours to 7 days for URLs older than 1 week. Super Bowl URLs are created in advance; they're cold-cache only at first redirect.

2. **Read replica scale-out** — Spin up 2-3 additional PostgreSQL read replicas immediately. AWS RDS can provision a read replica in ~5 minutes. Route cache-miss traffic across all replicas with round-robin.

**Longer-term fix** (for next campaign):

**Pre-warm the cache** — Before the ad airs, identify URLs in the campaign (or the top 1,000 most recently created URLs) and load them directly into Redis. If you know "bit.ly/superbowl" will receive 500K clicks at 8:45 PM, cache it at 8:00 PM.

**Why this is senior-level thinking**: You addressed both the immediate fire (TTL + replicas) and the root cause (cold cache at campaign launch). You also quantified the problem (5% miss × 500K req/s = 25K DB queries/sec).

</details>

---

## 📝 Interview Q&A

**Q: How do you prevent malicious long URLs (phishing, malware)?**
> A: Integrate with Google Safe Browsing API or VirusTotal at creation time. Maintain a blocklist of known-bad domains updated daily. If a URL is later reported as malicious, immediately invalidate it in cache and return 404/gone.

**Q: How would you add click analytics (clicks by country, device, time)?**
> A: Fire an async event to a Kafka topic on every redirect — `{ shortCode, timestamp, userAgent, ip }`. A separate analytics service consumes these events and aggregates into ClickHouse or BigQuery. Never block the redirect path for analytics — latency > reliability.

**Q: What if two service instances generate the same short code simultaneously?**
> A: With a Snowflake ID generator, each instance has a unique `worker_id`, so IDs are guaranteed unique across instances without coordination. If using hash-based generation, rely on the database's unique constraint — the second writer gets a unique violation exception and should retry with a different salt.

**Q: How do you handle a URL that 10 million people try to access simultaneously (thundering herd)?**
> A: Use a "mutex" pattern in Redis: only one request should go to the DB on a cache miss. All others wait on a Redis pub/sub channel. When the first request populates the cache, it publishes the result — all waiting requests receive it without hitting the DB. This is sometimes called the "cache stampede" prevention pattern.

**Q: How would you scale writes to 1M URL creations/sec?**
> A: At that scale, shard the write load: use consistent hashing to route creations to specific DB shards based on the first character of the short code. Use async batch inserts with a write-ahead buffer (Kafka queue → bulk insert every 100ms). Consider switching to DynamoDB which handles write throughput natively.

---

## 🔗 What to Read Next

1. **[Design Twitter](./DesignTwitter.md)** — Level up to a more complex system with fan-out and feed generation
2. **[BuildingBlocks/Caching.md](../BuildingBlocks/Caching.md)** — Deep dive into the caching strategies used in this design
3. **[KeyConcepts/Consistent_Hashing.md](../KeyConcepts/Consistent_Hashing.md)** — Understand the sharding strategy for URL storage at scale

---

*[← Back to Case Studies](./README.md) | [Next: Design Twitter →](./DesignTwitter.md)*
