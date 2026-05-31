# 🌐 Design a Content Delivery Network (CDN)

> *"When Netflix streams a movie to 230 million subscribers worldwide, the video doesn't come from one server in California — it comes from a server in YOUR city! CDNs cache content at 'edge' locations close to users, reducing latency from 200ms to 20ms. Understanding CDN architecture is essential because every large-scale system uses one, and sometimes you need to BUILD one."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [CDN Basics](../BuildingBlocks/CDN.md), [Caching](../BuildingBlocks/Caching.md), [DNS](../Foundations/Networking/DNS.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Content Routing](#-content-routing)
4. [Caching Strategy](#-caching-strategy)
5. [Cache Invalidation](#-cache-invalidation)
6. [Performance Optimizations](#-performance-optimizations)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Serve static content (images, JS, CSS, video) from edge
  • Dynamic content acceleration (API responses close to users)
  • SSL/TLS termination at edge
  • Content invalidation (purge stale content!)
  • Origin shielding (protect origin from thundering herd)
  
NON-FUNCTIONAL:
  • Latency: < 50ms for cached content (20ms target!)
  • Availability: 99.99% (4 nines!)
  • Hit ratio: > 95% for static, > 80% for dynamic
  • Global: serve from 200+ PoPs (Points of Presence)
  • Bandwidth: terabits per second aggregate!

SCALE:
  • Serve 100K+ requests/second per PoP
  • Store petabytes of cached content across all PoPs
  • Handle traffic spikes (viral content, flash sales!)
```

---

## 🏗️ High-Level Architecture

```
USER REQUEST FLOW:

  User (Tokyo) → "images.example.com/photo.jpg"
  
  1. DNS Resolution:
     images.example.com → CNAME → cdn.provider.com
     → GeoDNS returns nearest PoP IP (Tokyo PoP!)
  
  2. Edge PoP (Tokyo):
     ┌────────────────────────────────────┐
     │  Tokyo PoP                          │
     │  ┌──────────┐                       │
     │  │  L7 LB   │ (HAProxy/Nginx)       │
     │  └────┬─────┘                       │
     │       │                             │
     │  ┌────▼─────┐                       │
     │  │  Cache    │ SSD + RAM (hot!)      │
     │  │  Server   │ HIT? → Return! ✅     │
     │  └────┬─────┘ MISS? ↓              │
     │       │                             │
     └───────┼─────────────────────────────┘
             │ Cache miss!
             ▼
  3. Mid-Tier Shield (Regional):
     ┌────────────────────────────────────┐
     │  Asia Shield (Singapore)            │
     │  Aggregates misses from all         │
     │  Asian PoPs → reduces origin load!  │
     │  HIT? → Return! ✅                  │
     │  MISS? ↓                            │
     └───────┬────────────────────────────┘
             │
             ▼
  4. Origin Server (US-West):
     ┌────────────────────────────────────┐
     │  Origin (your servers!)             │
     │  Generate content → return to       │
     │  shield → cache → all PoPs get it!  │
     └────────────────────────────────────┘

FULL ARCHITECTURE:
  ┌─────────────────────────────────────────────────────────────────┐
  │                        GLOBAL CDN                                │
  │                                                                  │
  │  Users      PoPs (Edge)       Shield (Mid)      Origin          │
  │                                                                  │
  │  🇯🇵 ──────► [Tokyo]  ─────┐                                    │
  │  🇰🇷 ──────► [Seoul]  ─────┼──► [Asia Shield] ─┐               │
  │  🇮🇳 ──────► [Mumbai] ─────┘                    │               │
  │                                                  ├──► [Origin]  │
  │  🇬🇧 ──────► [London] ─────┐                    │   (US-West)  │
  │  🇩🇪 ──────► [Frankfurt]───┼──► [EU Shield] ───┘               │
  │  🇫🇷 ──────► [Paris] ──────┘                                    │
  │                                                                  │
  └─────────────────────────────────────────────────────────────────┘

WHY ORIGIN SHIELD?
  Without shield: 200 PoPs all miss → 200 requests to origin! 💀
  With shield: 200 PoPs miss → 1 regional shield request → origin!
  Origin sees 3 requests (one per region) instead of 200!
```

---

## 🗺️ Content Routing

```
HOW DOES A USER REACH THE "NEAREST" PoP?

METHOD 1: DNS-BASED (most common!)
  User → DNS query → Authoritative DNS sees user's resolver IP →
  Returns IP of nearest PoP based on resolver location!
  
  Problem: resolver IP ≠ user IP (Google DNS 8.8.8.8 is in... everywhere!)
  Solution: EDNS Client Subnet — DNS includes user's /24 subnet!

METHOD 2: ANYCAST (advanced, used by Cloudflare!)
  All PoPs advertise the SAME IP address via BGP!
  Internet routing naturally sends packets to nearest PoP!
  
  Advantages: 
  • Instant failover (BGP withdrawal → next nearest PoP!)
  • Works for UDP (DNS, QUIC!) and TCP
  • No DNS TTL issues!
  
  Challenge: TCP connection breaks if routing changes (rare!)

METHOD 3: HTTP REDIRECT
  User → any PoP → 302 Redirect → nearest PoP
  (Adds one RTT! Only used as fallback.)

ROUTING DECISION FACTORS:
  1. Geographic proximity (nearest PoP)
  2. PoP health and capacity (avoid overloaded PoPs!)
  3. Network path quality (latency, not just distance!)
  4. Content availability (does this PoP have the content cached?)
  5. Cost (some networks are cheaper to route through!)
```

---

## 📦 Caching Strategy

```
WHAT TO CACHE AND FOR HOW LONG:

  ┌────────────────────────────────────────────────────────────────┐
  │  Content Type     │  TTL         │  Cache-Control Header       │
  ├────────────────────────────────────────────────────────────────┤
  │  Static assets    │  1 year!     │  max-age=31536000, immutable│
  │  (versioned JS)   │  (v2.3.js)   │  (filename changes = fresh) │
  ├────────────────────────────────────────────────────────────────┤
  │  Images           │  30 days     │  max-age=2592000            │
  │  (user avatars)   │              │  stale-while-revalidate=86400│
  ├────────────────────────────────────────────────────────────────┤
  │  API responses    │  60 seconds  │  max-age=60, s-maxage=300   │
  │  (product list)   │              │  (CDN caches longer!)       │
  ├────────────────────────────────────────────────────────────────┤
  │  HTML pages       │  0 (always   │  no-cache, must-revalidate  │
  │  (index.html)     │  revalidate!)│  (ETag for conditional GET) │
  ├────────────────────────────────────────────────────────────────┤
  │  User-specific    │  NEVER cache!│  private, no-store          │
  │  (auth tokens)    │              │  (SECURITY!)                │
  └────────────────────────────────────────────────────────────────┘

CACHE KEY DESIGN:
  Simple: URL path
  With variants: URL + Accept-Encoding + Accept-Language
  
  Same URL, different content:
  • Compressed: Accept-Encoding: gzip → cached gzip version
  • Mobile: User-Agent based → careful! Low hit ratio!
  • Region: country-specific content → Vary: X-Country

STORAGE TIERS AT EDGE:
  ┌───────────────────────────────────────────────────────────────┐
  │  Tier    │  Capacity  │  Speed     │  Content                 │
  ├───────────────────────────────────────────────────────────────┤
  │  RAM     │  64-256 GB │  μs        │  Top 0.1% hottest items  │
  │  NVMe SSD│  2-8 TB    │  0.1ms     │  Active working set      │
  │  HDD     │  50+ TB    │  5-10ms    │  Long-tail cold content  │
  └───────────────────────────────────────────────────────────────┘
  
  Hot content promoted to RAM automatically (LFU eviction!)
  Cold content evicted from SSD → refetch from origin on next request.
```

---

## 🔄 Cache Invalidation

```
THE HARDEST PROBLEM IN CDN DESIGN!

  You update a product price. CDN has old price cached at 200 PoPs.
  How quickly does EVERY PoP serve the new price?

STRATEGY 1: TTL-BASED (eventual consistency)
  Set short TTL → content naturally refreshes!
  Simple but: stale content served until TTL expires!
  Good for: non-critical content (blog posts, images)

STRATEGY 2: PURGE API (immediate!)
  POST /purge {"urls": ["images.example.com/hero.jpg"]}
  → CDN broadcasts purge to ALL PoPs → content deleted!
  → Next request → cache miss → fresh from origin!
  
  Challenge: purging 200 PoPs takes 2-5 seconds globally!
  Some users may see stale content during propagation.

STRATEGY 3: VERSIONED URLs (best for static assets!)
  Old: /static/app.js → /static/app.v2.3.js
  New: /static/app.v2.4.js (completely different URL!)
  
  Old version stays cached (harmless — no one requests it!)
  New version has no cache → fetches fresh from origin!
  
  Implementation: hash content into filename!
  app.a3f8b2c1.js ← content hash in filename!

STRATEGY 4: STALE-WHILE-REVALIDATE (best user experience!)
  Cache-Control: max-age=60, stale-while-revalidate=3600
  
  After 60s: content is "stale" BUT:
  • Serve stale to current request (FAST! no waiting!)
  • Background: revalidate with origin!
  • Next request gets fresh content!
  
  User always gets fast response! (stale or fresh)

PURGE ARCHITECTURE:
  ┌──────────┐     ┌───────────┐     ┌─────────────┐
  │   API    │────►│   Purge   │────►│  Fanout to  │
  │  Client  │     │  Service  │     │  all PoPs!  │
  └──────────┘     └───────────┘     └─────────────┘
                                      │  │  │  │  │
                                      ▼  ▼  ▼  ▼  ▼
                                    [Each PoP deletes from local cache]
```

---

## ⚡ Performance Optimizations

```
1. CONNECTION OPTIMIZATION:
   • TCP connection reuse (Keep-Alive between edge ↔ origin!)
   • HTTP/2 multiplexing (one connection, many requests!)
   • Pre-warmed connections to origin (avoid cold TCP handshakes!)
   • TLS session resumption (skip full handshake on reconnect!)

2. PROTOCOL OPTIMIZATION:
   • Brotli compression (20-30% smaller than gzip for text!)
   • HTTP/3 (QUIC) at edge (0-RTT connection, no head-of-line blocking!)
   • Early Hints (103) — push CSS/JS before HTML finishes!

3. CONTENT OPTIMIZATION:
   • Image optimization (WebP/AVIF conversion on-the-fly!)
   • Video adaptive bitrate (HLS/DASH chunk caching!)
   • Minification (HTML/CSS/JS whitespace removal at edge!)

4. PREFETCHING:
   • Predict next request based on current page
   • Push related assets before browser asks!
   • Warm popular content proactively at all PoPs!

5. NEGATIVE CACHING:
   • Cache 404 responses! (don't hit origin repeatedly for missing content!)
   • Cache 301 redirects (permanent redirect → cache forever!)
   • Short TTL for 5xx (origin error → retry soon, not forever!)
```

---

## 💻 Java Implementation

### CDN-Friendly Spring Boot Configuration

```java
@Configuration
public class CdnConfiguration {
    
    /**
     * Set proper Cache-Control headers for CDN caching.
     */
    @Bean
    public WebMvcConfigurer cacheConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addResourceHandlers(ResourceHandlerRegistry registry) {
                // Static assets: cache for 1 year (use content hash in filename!)
                registry.addResourceHandler("/static/**")
                    .addResourceLocations("classpath:/static/")
                    .setCacheControl(CacheControl
                        .maxAge(365, TimeUnit.DAYS)
                        .cachePublic());
            }
        };
    }
}

@RestController
@RequestMapping("/api")
public class ProductController {
    
    @GetMapping("/products/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable String id) {
        Product product = productService.findById(id);
        
        String etag = "\"" + product.getVersion() + "\"";
        
        return ResponseEntity.ok()
            .cacheControl(CacheControl
                .maxAge(60, TimeUnit.SECONDS)    // Browser cache: 60s
                .sMaxAge(300, TimeUnit.SECONDS)  // CDN cache: 5 min!
                .staleWhileRevalidate(3600, TimeUnit.SECONDS))
            .eTag(etag)
            .body(product);
    }
    
    /**
     * Purge CDN cache when product is updated.
     */
    @PutMapping("/products/{id}")
    @Transactional
    public Product updateProduct(@PathVariable String id, 
                                  @RequestBody ProductUpdate update) {
        Product saved = productService.update(id, update);
        
        // Purge CDN cache for this product!
        cdnPurgeService.purge(
            List.of("/api/products/" + id,
                    "/pages/products/" + id));
        
        return saved;
    }
}

/**
 * CDN Purge Service — invalidate cached content across all PoPs.
 */
@Service
public class CdnPurgeService {
    
    private final WebClient webClient;
    
    public CdnPurgeService(@Value("${cdn.purge.api}") String purgeApi,
                           @Value("${cdn.purge.token}") String token) {
        this.webClient = WebClient.builder()
            .baseUrl(purgeApi)
            .defaultHeader("Authorization", "Bearer " + token)
            .build();
    }
    
    public void purge(List<String> paths) {
        webClient.post()
            .uri("/purge")
            .bodyValue(Map.of("paths", paths))
            .retrieve()
            .bodyToMono(Void.class)
            .subscribe(
                v -> log.info("Purged: {}", paths),
                e -> log.error("Purge failed: {}", e.getMessage()));
    }
}
```

### Simple Edge Cache Server (Conceptual)

```java
/**
 * Simplified edge cache server logic (what runs at each PoP).
 */
public class EdgeCacheServer {
    
    // Two-tier local storage: hot (RAM) + warm (SSD)
    private final Cache<String, CachedResponse> hotCache;  // Caffeine (RAM)
    private final DiskCache warmCache;                      // SSD-backed
    private final HttpClient originClient;
    
    public EdgeCacheServer(String originUrl) {
        this.hotCache = Caffeine.newBuilder()
            .maximumSize(100_000)
            .build();
        this.warmCache = new DiskCache("/cache/ssd", "50GB");
        this.originClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(2))
            .build();
    }
    
    public CachedResponse handleRequest(HttpRequest request) {
        String cacheKey = buildCacheKey(request);
        
        // 1. Check hot cache (RAM — microseconds!)
        CachedResponse hot = hotCache.getIfPresent(cacheKey);
        if (hot != null && !hot.isExpired()) {
            return hot.withHeader("X-Cache", "HIT-RAM");
        }
        
        // 2. Check warm cache (SSD — sub-millisecond!)
        CachedResponse warm = warmCache.get(cacheKey);
        if (warm != null && !warm.isExpired()) {
            hotCache.put(cacheKey, warm); // Promote to hot!
            return warm.withHeader("X-Cache", "HIT-SSD");
        }
        
        // 3. Fetch from origin (with coalescing!)
        CachedResponse fresh = fetchFromOriginCoalesced(cacheKey, request);
        
        // 4. Store in both tiers
        hotCache.put(cacheKey, fresh);
        warmCache.put(cacheKey, fresh);
        
        return fresh.withHeader("X-Cache", "MISS");
    }
    
    /**
     * Request coalescing: if 100 requests miss for same key,
     * only ONE request goes to origin! Others wait for result.
     */
    private final Map<String, CompletableFuture<CachedResponse>> inflight 
        = new ConcurrentHashMap<>();
    
    private CachedResponse fetchFromOriginCoalesced(
            String key, HttpRequest request) {
        
        return inflight.computeIfAbsent(key, k -> 
            CompletableFuture.supplyAsync(() -> {
                try {
                    return fetchFromOrigin(request);
                } finally {
                    inflight.remove(k);
                }
            })
        ).join(); // All waiters get the same result!
    }
}
```

---

## ❓ Interview Q&A

**Q1: How does a CDN handle cache misses for viral content?**
> Challenge: new content goes viral → millions of requests but NOT in cache yet! Solutions: (1) Request coalescing — first miss fetches from origin, concurrent misses WAIT for that one fetch (not 1M origin requests!), (2) Origin shield — regional cache absorbs misses from all PoPs in that region, only one request reaches origin, (3) Stale-while-revalidate — serve slightly stale content while refreshing, (4) Pre-warming — for known events (product launches), proactively push content to all PoPs BEFORE launch!

**Q2: CDN vs Reverse Proxy — what's the difference?**
> A reverse proxy sits in ONE location in front of your servers. A CDN is a reverse proxy replicated at 200+ global locations. Both cache and serve content, but CDN's key advantage is geographic distribution — users hit the nearest PoP (20ms) instead of your data center on another continent (200ms). CDN also provides DDoS protection (absorb attacks across all PoPs), SSL termination at edge, and bandwidth offloading (your origin handles 1% of traffic, CDN serves 99%!).

**Q3: How would you handle personalized content with a CDN?**
> Personalized content (user-specific) is traditionally NOT cached. But optimizations exist: (1) Edge Side Includes (ESI) — cache page template at CDN, inject dynamic fragments at edge, (2) Split static/dynamic — cache page structure, AJAX-load personalized widgets, (3) Edge compute (Cloudflare Workers) — run JavaScript at edge to personalize cached base content, (4) Cache with Vary header — if finite variants (country, language), cache each variant separately. Key: maximize the cacheable portions!

**Q4: How do CDNs handle video streaming at scale?**
> Video split into small chunks (2-10 seconds each, HLS/DASH format). Each chunk is a separate cacheable file! Flow: (1) Client requests manifest file (lists all chunk URLs), (2) Client requests chunks sequentially, (3) CDN caches popular chunks at edge (first few chunks of popular videos = HOT!). Adaptive bitrate: multiple quality versions of each chunk — client switches quality based on bandwidth. Netflix/YouTube: pre-position popular content at PoPs during off-peak hours!

---

## 🔗 Related Topics
- [CDN Basics](../BuildingBlocks/CDN.md) — Fundamental concepts
- [Caching](../BuildingBlocks/Caching.md) — Cache strategies
- [DNS](../Foundations/Networking/DNS.md) — GeoDNS routing
- [Load Balancer](DesignLoadBalancer.md) — Traffic distribution

---

*"A CDN is not just a cache — it's a globally distributed system that makes physics less painful. You can't make light travel faster, but you can put your content closer to the user. That's the entire philosophy of a CDN in one sentence." — CDN Architect* 🌐
