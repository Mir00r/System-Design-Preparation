# 🕷️ Design a Distributed Web Crawler

> *"Google's web crawler (Googlebot) discovers and indexes 100+ BILLION web pages. It crawls 20 billion pages per day, making decisions every millisecond: 'Should I crawl this URL? When should I re-crawl it? How deep should I go?' A distributed web crawler must be fast (billions of pages!), polite (don't DDoS websites!), smart (prioritize important pages), and resilient (handle broken links, infinite loops, and spider traps)."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Message Queues](../../BuildingBlocks/MessageQueues.md), [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Crawler Architecture](#-crawler-architecture)
3. [URL Frontier (Priority Queue)](#-url-frontier-priority-queue)
4. [Politeness & Rate Limiting](#-politeness--rate-limiting)
5. [Deduplication](#-deduplication)
6. [Content Processing](#-content-processing)
7. [Handling Spider Traps](#-handling-spider-traps)
8. [Scaling & Distribution](#-scaling--distribution)
9. [Java Implementation](#-java-implementation)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Crawl the web starting from seed URLs
✅ Discover new URLs from crawled pages (link extraction)
✅ Respect robots.txt (be a good citizen!)
✅ Handle different content types (HTML, PDF, images)
✅ Store crawled content for indexing
✅ Re-crawl pages based on change frequency
✅ Support URL prioritization (important pages first!)
✅ Handle redirects (301, 302)
✅ Support authenticated/JS-rendered content
```

### Non-Functional Requirements
```
✅ Scale: crawl 1 billion pages per day
✅ Politeness: max 1 request/second per domain
✅ Robustness: handle failures, broken links gracefully
✅ Distributed: multiple crawlers across data centers
✅ Avoid duplicates: don't crawl same URL twice
✅ Freshness: popular pages re-crawled more often
✅ Extensibility: easy to add new content processors
```

### Scale Estimation
```
Target: 1 billion pages/day
Time: 24 hours = 86,400 seconds
QPS: 1B / 86,400 ≈ 11,570 pages/second!

Average page size: 500 KB
Daily storage: 1B × 500KB = 500 TB/day! 😱
(Compressed: ~100 TB/day)

URLs in frontier queue: 100 billion+ (most of the web!)
```

---

## 🏗️ Crawler Architecture

```
HIGH-LEVEL DESIGN:

  ┌─────────────────────────────────────────────────────────────────┐
  │  SEED URLS → [google.com, wikipedia.org, github.com, ...]      │
  └──────────────────────────┬──────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  URL FRONTIER   │ ← Priority queue with politeness!
                    │  (billions of   │
                    │   URLs to crawl)│
                    └────────┬────────┘
                             │ (dispatch URLs to workers)
           ┌─────────────────┼─────────────────────┐
           ▼                 ▼                     ▼
  ┌──────────────┐  ┌──────────────┐     ┌──────────────┐
  │  Crawler     │  │  Crawler     │     │  Crawler     │
  │  Worker 1    │  │  Worker 2    │     │  Worker N    │
  │              │  │              │     │              │
  │ 1. Fetch     │  │ 1. Fetch     │     │ 1. Fetch     │
  │ 2. Parse     │  │ 2. Parse     │     │ 2. Parse     │
  │ 3. Extract   │  │ 3. Extract   │     │ 3. Extract   │
  └──────┬───────┘  └──────┬───────┘     └──────┬───────┘
         │                  │                    │
         └──────────────────┼────────────────────┘
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
     ┌──────────────┐ ┌─────────┐  ┌──────────────┐
     │  Content     │ │  URL    │  │  URL         │
     │  Store       │ │  Filter │  │  Frontier    │
     │  (S3/HDFS)   │ │  (dedup)│  │  (new URLs!) │
     └──────────────┘ └─────────┘  └──────────────┘
                            │
                   ┌────────▼─────────┐
                   │  Seen URL Store  │ (Bloom Filter + DB)
                   │  "Already crawled │
                   │   this URL?"      │
                   └──────────────────┘

CRAWLER LOOP (per worker):
  1. Pull URL from frontier (highest priority, respecting politeness)
  2. Check robots.txt for this domain (cached!)
  3. Fetch page (HTTP GET with timeout)
  4. Parse HTML → extract text + links
  5. Store content → Content Store
  6. For each extracted link:
     a. Normalize URL (remove fragments, canonicalize)
     b. Check: already seen? (Bloom filter)
     c. If new → add to URL Frontier with priority
  7. Loop!
```

---

## 📋 URL Frontier (Priority Queue)

```
THE FRONTIER IS THE BRAIN OF THE CRAWLER!

  Not just a simple queue — it's a PRIORITIZED, POLITENESS-AWARE,
  RE-CRAWL-SCHEDULING system!

  TWO DIMENSIONS:
  ┌──────────────────────────────────────────────────────────────┐
  │  1. PRIORITY: Which URLs are most important?                  │
  │     - PageRank / domain authority                            │
  │     - Freshness need (news sites > static docs)             │
  │     - Depth from seed (shallow = more important)            │
  │                                                              │
  │  2. POLITENESS: Don't overwhelm any single host!            │
  │     - Max 1 request/second per domain                       │
  │     - Can't crawl google.com 10,000 times/second!          │
  └──────────────────────────────────────────────────────────────┘

  ARCHITECTURE:
  
  ┌─────────────────────────────────────────────────┐
  │           PRIORITIZER                            │
  │  Assigns priority (1=highest, N=lowest)         │
  └─────────┬───────────────────────────────────────┘
            │
  ┌─────────▼───────────────────────────────────────┐
  │  PRIORITY QUEUES (by importance)                │
  │                                                  │
  │  Priority 1: ████████ [news sites, Wikipedia]    │
  │  Priority 2: ██████ [popular blogs, stackoverflow]│
  │  Priority 3: ████ [regular sites]                │
  │  Priority 4: ██ [deep links, low authority]      │
  └─────────┬───────────────────────────────────────┘
            │
  ┌─────────▼───────────────────────────────────────┐
  │  POLITENESS ROUTER                               │
  │  Routes URL to per-host queue                   │
  │                                                  │
  │  Queue[google.com]: url1, url2, url3            │
  │  Queue[github.com]: url4, url5                  │
  │  Queue[example.com]: url6                       │
  │                                                  │
  │  Each host queue: 1 URL dispatched per second!  │
  └─────────────────────────────────────────────────┘

  SELECTION ALGORITHM:
  1. Worker requests next URL
  2. Select priority bucket (weighted random: P1 gets 50%, P2 30%, etc.)
  3. From selected bucket, find a host queue whose "last crawl" was > 1s ago
  4. Pop URL from that host queue → give to worker!
  5. Update "last crawl time" for this host
```

---

## 🤝 Politeness & Rate Limiting

```
ROBOTS.TXT COMPLIANCE:

  Before crawling ANY page on a domain:
  1. Fetch and parse robots.txt
  2. Cache it (refresh every 24 hours)
  3. Obey Crawl-delay directive
  4. Respect Disallow rules
  
  Example robots.txt:
  User-agent: *
  Crawl-delay: 2            ← Wait 2 seconds between requests!
  Disallow: /admin/         ← Don't crawl admin pages
  Disallow: /private/
  Allow: /public/
  Sitemap: https://example.com/sitemap.xml

RATE LIMITING PER DOMAIN:
  
  ┌───────────────────────────────────────────────────────────┐
  │  Domain          │  Rate Limit  │  Why                    │
  ├───────────────────────────────────────────────────────────┤
  │  Large sites     │  1 req/sec   │  Default politeness     │
  │  (google.com)    │              │                         │
  │  Small sites     │  1 req/5sec  │  Don't overwhelm!       │
  │  (personal blog) │              │                         │
  │  With Crawl-delay│  As specified│  Respect their wishes   │
  │  News sites      │  1 req/sec   │  Need freshness!        │
  └───────────────────────────────────────────────────────────┘
  
  Implementation: Token bucket per domain
    Each domain gets a bucket: refills 1 token per second
    To crawl: take 1 token. Empty bucket? Wait!
    
  ADDITIONAL POLITENESS:
  • Use descriptive User-Agent: "MyBot/1.0 (+https://mybot.com/about)"
  • Provide contact info for webmasters
  • Crawl during off-peak hours for small sites
  • Stop crawling if getting 429 (Too Many Requests) or 503
  • Exponential backoff on errors
```

---

## 🔍 Deduplication

```
THREE LEVELS OF DEDUPLICATION:

LEVEL 1: URL DEDUPLICATION (before crawling)
  "Have I already crawled this URL?"
  
  Storage: Bloom Filter (10 billion URLs, 1% false positive)
  Size: ~1.2 GB for 10 billion URLs! (fits in memory!)
  
  URL normalization first:
  • https://Example.COM/path?a=1&b=2 → https://example.com/path?a=1&b=2
  • Remove fragments: /page#section → /page
  • Remove tracking params: ?utm_source=X → (strip)
  • Resolve relative URLs: ../page → absolute URL
  • Remove default ports: :80, :443
  
  Bloom filter check:
    If "probably seen" → skip (accept rare false positives)
    If "definitely not seen" → crawl + add to filter!

LEVEL 2: CONTENT DEDUPLICATION (after crawling)
  "Have I already stored this CONTENT?" (same page, different URL!)
  
  Many URLs serve identical content:
  http://example.com and https://example.com → same page!
  /index.html and / → same page!
  
  Solution: Simhash / MinHash
  • Compute fingerprint of page content
  • Compare with existing fingerprints
  • If similarity > 95% → duplicate! Don't store again.
  
  Simhash: locality-sensitive hash
  Similar documents → similar hashes (Hamming distance ≤ 3)

LEVEL 3: NEAR-DUPLICATE DETECTION
  Pages that are ALMOST the same (boilerplate + ads differ)
  • Strip boilerplate (nav, footer, ads)
  • Hash core content only
  • Useful for news articles syndicated across sites
```

---

## 📄 Content Processing

```
PROCESSING PIPELINE (after fetching page):

  Raw HTML
     │
     ▼
  ┌────────────────────┐
  │  1. Parse HTML     │  (Jsoup for Java)
  │     Extract DOM    │
  └────────┬───────────┘
           │
     ┌─────┴──────┐
     ▼            ▼
  ┌─────────┐  ┌────────────┐
  │ Extract │  │ Extract    │
  │ Links   │  │ Content    │
  └────┬────┘  └─────┬──────┘
       │              │
       ▼              ▼
  ┌─────────┐  ┌────────────┐
  │ Filter  │  │ Store in   │
  │ & Dedup │  │ Content    │
  │ & Queue │  │ Store      │
  └─────────┘  └────────────┘

  LINK EXTRACTION:
  • <a href="..."> → follow! (HTML links)
  • <link rel="..."> → follow sitemap, canonical
  • JavaScript links → render page first (expensive!)
  • PDF/DOC links → specialized extractors
  
  CONTENT EXTRACTION:
  • Title: <title> tag
  • Headings: h1-h6
  • Main content: strip nav/footer/ads (readability algorithms)
  • Meta: description, keywords, og:tags
  • Structured data: JSON-LD, microdata
  
  LANGUAGE DETECTION:
  • HTML lang attribute
  • Character encoding analysis
  • N-gram language classification

  DNS CACHING:
  Crawling 1B pages = billions of DNS lookups!
  Solution: Local DNS cache (Redis/in-memory)
  domain → IP mapping, refresh every hour
  Saves massive DNS overhead!
```

---

## 🕸️ Handling Spider Traps

```
SPIDER TRAPS: Websites that generate INFINITE URLs!

  Example 1: Calendar trap
  /calendar?date=2024-01-01 → links to 2024-01-02
  /calendar?date=2024-01-02 → links to 2024-01-03
  ... INFINITE PAGES! Same template, different dates forever!
  
  Example 2: Session ID in URL
  /page?session=abc123 → same page, different session!
  /page?session=def456 → same page, new URL every visit!
  
  Example 3: Infinite directory depth
  /a/b/c/d/e/f/g/h/i/j/k/l/m/... → generated dynamically!

DEFENSES:
  ┌──────────────────────────────────────────────────────────────┐
  │  Defense              │  How It Works                         │
  ├──────────────────────────────────────────────────────────────┤
  │  Max depth limit      │  Don't follow links beyond depth 15  │
  │  (from seed)          │  Most useful content is shallow!     │
  ├──────────────────────────────────────────────────────────────┤
  │  URL pattern detection│  If 100+ URLs match same regex       │
  │                       │  pattern → likely trap! Stop.        │
  ├──────────────────────────────────────────────────────────────┤
  │  Max pages per domain │  Cap at 1M pages per domain          │
  │                       │  (even Wikipedia has < 60M pages)    │
  ├──────────────────────────────────────────────────────────────┤
  │  Content similarity   │  If last 10 pages from same host     │
  │                       │  have >95% similar content → stop!   │
  ├──────────────────────────────────────────────────────────────┤
  │  URL length limit     │  URL > 200 chars? Probably trap!     │
  ├──────────────────────────────────────────────────────────────┤
  │  Blacklist patterns   │  /calendar, /tag/*, ?page=9999      │
  └──────────────────────────────────────────────────────────────┘
  
  MANUAL OVERRIDE: Human reviewers can blacklist specific trap patterns
```

---

## 📈 Scaling & Distribution

```
DISTRIBUTING THE CRAWLER:

  1 billion pages/day ÷ 11,570 pages/second
  Each worker: ~10 pages/second (limited by network + politeness)
  Workers needed: 11,570 / 10 ≈ 1,200 crawler workers!

  PARTITIONING STRATEGY: By DOMAIN
  
  Why? Politeness is per-domain → one worker handles one domain!
  
  Worker 1: crawls [aaa.com - dzz.com]
  Worker 2: crawls [eaa.com - hzz.com]
  Worker 3: crawls [iaa.com - lzz.com]
  ...
  
  Benefits:
  • Each worker maintains its own politeness timer per domain
  • robots.txt cached locally per worker (no sharing needed!)
  • DNS results cached per worker
  • No coordination needed between workers for same domain!
  
  Use CONSISTENT HASHING:
  hash(domain) → assigned worker
  Worker failure → consistent hashing redistributes minimally!

MULTI-REGION DEPLOYMENT:
  ┌───────────────────────────────────────────────────┐
  │  Region     │  Crawls               │  Why        │
  ├───────────────────────────────────────────────────┤
  │  US-East    │  .com, .org, .net     │  Most sites │
  │  EU-West    │  .de, .fr, .uk, .eu   │  EU sites   │
  │  Asia       │  .cn, .jp, .kr, .in   │  Asia sites │
  └───────────────────────────────────────────────────┘
  
  Crawl from geographically close location → lower latency!
  Also respects data sovereignty for some regions.

CHECKPOINT & RECOVERY:
  Worker crashes → what about URLs it was processing?
  
  Solution: URL claimed via lease (5 min TTL in Redis)
  If worker crashes → lease expires → URL goes back to frontier
  Periodic checkpointing: save frontier state every 10 minutes
```

---

## 💻 Java Implementation

### Crawler Worker

```java
@Service
public class CrawlerWorker {
    
    @Autowired private URLFrontier frontier;
    @Autowired private RobotsChecker robotsChecker;
    @Autowired private ContentStore contentStore;
    @Autowired private URLDeduplicator deduplicator;
    @Autowired private RestTemplate httpClient;
    
    @Scheduled(fixedDelay = 100) // Process continuously
    public void crawl() {
        // 1. Get next URL from frontier (respects priority + politeness!)
        CrawlTarget target = frontier.getNext();
        if (target == null) return; // Nothing to crawl
        
        String url = target.getUrl();
        
        try {
            // 2. Check robots.txt
            if (!robotsChecker.isAllowed(url)) {
                log.debug("Blocked by robots.txt: {}", url);
                return;
            }
            
            // 3. Fetch the page
            CrawlResult result = fetchPage(url);
            
            if (result.getStatusCode() == 301 || result.getStatusCode() == 302) {
                // Handle redirect: add redirect target to frontier
                String redirectUrl = result.getRedirectLocation();
                if (redirectUrl != null && !deduplicator.isSeen(redirectUrl)) {
                    frontier.add(redirectUrl, target.getPriority());
                }
                return;
            }
            
            if (result.getStatusCode() != 200) return;
            
            // 4. Parse and extract content + links
            ParsedPage parsed = parseHTML(result.getBody(), url);
            
            // 5. Store content
            contentStore.store(ContentDocument.builder()
                .url(url)
                .title(parsed.getTitle())
                .content(parsed.getMainContent())
                .crawledAt(Instant.now())
                .contentHash(computeSimhash(parsed.getMainContent()))
                .build());
            
            // 6. Process extracted links
            for (String link : parsed.getLinks()) {
                String normalized = normalizeURL(link, url);
                if (normalized == null) continue;
                if (deduplicator.isSeen(normalized)) continue;
                if (isSpiderTrap(normalized, target)) continue;
                
                int priority = calculatePriority(normalized, target);
                frontier.add(normalized, priority);
                deduplicator.markSeen(normalized);
            }
            
        } catch (Exception e) {
            handleCrawlError(url, e);
        }
    }
    
    private CrawlResult fetchPage(String url) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("User-Agent", 
            "MyBot/1.0 (+https://mybot.com/about)");
        headers.set("Accept", "text/html,application/xhtml+xml");
        
        RequestEntity<Void> request = RequestEntity
            .get(URI.create(url))
            .headers(headers)
            .build();
        
        ResponseEntity<String> response = httpClient.exchange(
            request, String.class);
        
        return new CrawlResult(
            response.getStatusCodeValue(),
            response.getBody(),
            response.getHeaders().getFirst("Location"));
    }
}
```

### URL Frontier with Politeness

```java
@Service
public class URLFrontier {
    
    @Autowired private StringRedisTemplate redis;
    
    private static final Duration POLITENESS_INTERVAL = Duration.ofSeconds(1);
    
    /**
     * Add URL to frontier with priority.
     * Routed to appropriate host queue for politeness.
     */
    public void add(String url, int priority) {
        String domain = extractDomain(url);
        String queueKey = "frontier:" + priority + ":" + domain;
        
        // Add to per-domain queue within priority bucket
        redis.opsForList().rightPush(queueKey, url);
        
        // Register domain in priority bucket's domain set
        redis.opsForSet().add("domains:" + priority, domain);
    }
    
    /**
     * Get next URL to crawl.
     * Respects priority (higher priority first) AND politeness (1 req/sec/domain)
     */
    public CrawlTarget getNext() {
        // Weighted priority selection
        int priority = selectPriorityBucket();
        
        // Get domains in this priority bucket
        Set<String> domains = redis.opsForSet()
            .members("domains:" + priority);
        
        if (domains == null || domains.isEmpty()) return null;
        
        // Find a domain we can crawl (politeness check!)
        for (String domain : domains) {
            String lastCrawlKey = "last-crawl:" + domain;
            String lastCrawlStr = redis.opsForValue().get(lastCrawlKey);
            
            if (lastCrawlStr != null) {
                Instant lastCrawl = Instant.parse(lastCrawlStr);
                if (lastCrawl.plus(POLITENESS_INTERVAL).isAfter(Instant.now())) {
                    continue; // Too soon! Try another domain.
                }
            }
            
            // This domain is eligible! Pop a URL.
            String queueKey = "frontier:" + priority + ":" + domain;
            String url = redis.opsForList().leftPop(queueKey);
            
            if (url != null) {
                // Record crawl time for politeness
                redis.opsForValue().set(lastCrawlKey, 
                    Instant.now().toString(), 
                    Duration.ofMinutes(5));
                
                return new CrawlTarget(url, priority, domain);
            }
        }
        
        return null; // All domains are in cooldown!
    }
    
    private int selectPriorityBucket() {
        // Weighted random: P1=50%, P2=30%, P3=15%, P4=5%
        double rand = ThreadLocalRandom.current().nextDouble();
        if (rand < 0.50) return 1;
        if (rand < 0.80) return 2;
        if (rand < 0.95) return 3;
        return 4;
    }
}
```

### Spider Trap Detection

```java
@Component
public class SpiderTrapDetector {
    
    // Track URL patterns per domain
    private final Map<String, PatternCounter> domainPatterns = 
        new ConcurrentHashMap<>();
    
    public boolean isSpiderTrap(String url, CrawlTarget parent) {
        // Rule 1: Max depth from seed
        if (parent.getDepth() > 15) return true;
        
        // Rule 2: URL too long
        if (url.length() > 200) return true;
        
        // Rule 3: Too many similar URLs from same domain
        String domain = extractDomain(url);
        String pattern = extractPattern(url); // e.g., "/calendar?date=*"
        
        PatternCounter counter = domainPatterns
            .computeIfAbsent(domain, k -> new PatternCounter());
        
        int count = counter.increment(pattern);
        if (count > 100) return true; // 100+ URLs matching same pattern!
        
        // Rule 4: Repeating path segments
        if (hasRepeatingSegments(url)) return true;
        // e.g., /a/b/a/b/a/b → clearly a loop!
        
        return false;
    }
    
    private String extractPattern(String url) {
        // Replace numbers with * to detect patterns
        // /calendar?date=2024-01-15 → /calendar?date=*-*-*
        return url.replaceAll("\\d+", "*");
    }
    
    private boolean hasRepeatingSegments(String url) {
        String path = URI.create(url).getPath();
        String[] segments = path.split("/");
        
        // Check if any subsequence of 2+ segments repeats 3+ times
        for (int len = 2; len <= segments.length / 3; len++) {
            for (int start = 0; start + len * 3 <= segments.length; start++) {
                boolean repeats = true;
                for (int i = 0; i < len && repeats; i++) {
                    String seg = segments[start + i];
                    for (int rep = 1; rep < 3; rep++) {
                        if (!seg.equals(segments[start + rep * len + i])) {
                            repeats = false;
                            break;
                        }
                    }
                }
                if (repeats) return true;
            }
        }
        return false;
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Crawl Frequency Scheduler

Some pages change every hour (news), some every month (Wikipedia articles). Design a system that decides HOW OFTEN to re-crawl each URL.

<details>
<summary>🔑 Answer</summary>

```java
public class RecrawlScheduler {
    
    /**
     * Adaptive re-crawl frequency based on change history.
     */
    public Duration getRecrawlInterval(String url) {
        ChangeHistory history = getChangeHistory(url);
        
        if (history == null) {
            return Duration.ofDays(7); // Default: weekly for new URLs
        }
        
        // Method: Exponential moving average of change intervals
        double avgChangePeriodHours = history.getAvgChangeInterval();
        double lastChangeHoursAgo = history.getHoursSinceLastChange();
        
        // If page changes every 2 hours → re-crawl every 1 hour
        // If page changes every 30 days → re-crawl every 15 days
        // Re-crawl at HALF the average change interval!
        Duration interval = Duration.ofHours(
            (long) (avgChangePeriodHours * 0.5));
        
        // Bounds: minimum 30 min, maximum 30 days
        if (interval.toMinutes() < 30) interval = Duration.ofMinutes(30);
        if (interval.toDays() > 30) interval = Duration.ofDays(30);
        
        // Boost for important pages (high PageRank)
        double importance = getPageRank(url);
        if (importance > 0.8) {
            interval = interval.dividedBy(2); // Crawl important pages 2x often
        }
        
        return interval;
    }
    
    // Track: did the content actually CHANGE since last crawl?
    public void recordCrawl(String url, String contentHash) {
        String previousHash = getPreviousHash(url);
        boolean changed = !contentHash.equals(previousHash);
        updateChangeHistory(url, changed, Instant.now());
    }
}
```

**Key insight:** Don't waste crawl budget on pages that never change! Focus resources on dynamic content. Google uses "adaptive crawl rate" — if a page hasn't changed in 5 consecutive crawls, exponentially back off.
</details>

---

## ❓ Interview Q&A

**Q1: How do you avoid crawling the same URL twice?**
> Two-layer deduplication: (1) Bloom filter (in-memory, ~1.2 GB for 10B URLs, O(1) lookup, 1% false positive rate — acceptable since false positive just means we skip a URL), (2) URL normalization before checking (lowercase, remove fragments, strip tracking params, resolve relative paths). The Bloom filter is distributed across crawler nodes using consistent hashing (each node owns a URL space partition).

**Q2: How do you handle websites that require JavaScript rendering?**
> Headless browser pool (Chrome via Puppeteer/Playwright): expensive! Only use for high-value pages where JS-rendered content differs from raw HTML. Decision: first crawl with simple HTTP. If page has minimal content but heavy JS → re-crawl with headless browser. Maintain a "needs-rendering" list. Headless crawling is 10-50x slower, so limit to 5-10% of total crawl volume.

**Q3: How would you detect and handle content changes efficiently?**
> Conditional HTTP requests! Send `If-Modified-Since` header (server returns 304 if unchanged — saves bandwidth!). Also: compare content hash (Simhash for near-duplicate detection). Track change frequency per URL → adaptive re-crawl scheduling (crawl changing pages often, stable pages rarely). ETag support: send `If-None-Match` header for exact change detection.

**Q4: How do you prioritize which URLs to crawl first?**
> Multi-signal priority: (1) PageRank/domain authority (backlink count), (2) Freshness need (news = high priority), (3) Depth from seed URL (shallow = higher priority), (4) User signals (pages frequently searched for but stale), (5) Sitemap priority hints, (6) Content type (articles > listing pages). These signals combine into a priority score that determines which frontier bucket the URL enters.

**Q5: How would you estimate the size of the web for storage planning?**
> Google indexes ~100 billion pages. Average page: 500KB raw HTML. Compressed (gzip): ~50KB. At 100B pages: 5 petabytes compressed. But we also store metadata, link graphs, rendering results → multiply by 3-5x. Total: 15-25 PB for full web index. Daily incremental: ~1-2 PB/day for crawl volume of 20B pages. Use tiered storage: hot (SSD for recent), warm (HDD for indexed), cold (tape/glacier for historical).

---

## 🔗 Related Topics
- [Message Queues](../../BuildingBlocks/MessageQueues.md) — URL frontier implementation
- [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md) — Distributing URLs to workers
- [Bloom Filters](../../Database/BloomFilters.md) — URL deduplication
- [Search Index](../../BuildingBlocks/SearchIndex.md) — What happens after crawling

---

*"A web crawler is a robot that reads the entire internet, respects everyone's wishes (robots.txt), never visits too often (politeness), avoids infinite loops (spider traps), and somehow does this for 100 billion pages. It's basically the most considerate reader in history." — Web Infrastructure Engineer* 🕷️
