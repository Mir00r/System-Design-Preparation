# 🔍 Design Google Search
## Web Crawling, Indexing, Ranking, and Sub-Second Query Processing at Planet Scale

> *"Google processes 8.5 billion searches per day — 100,000 queries every second. Behind every search is a pipeline that crawled billions of web pages, extracted meaning, built an inverted index, and ranked results using hundreds of signals — all to return 10 blue links in under 200ms."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md), [Search Index](../BuildingBlocks/SearchIndex.md)

---

## 📋 Requirements (RADIO: R)

### Functional Requirements

| Feature | Description |
|---|---|
| **Web crawling** | Discover and download billions of web pages continuously |
| **Indexing** | Parse pages, extract text, build searchable index |
| **Query processing** | Parse user query, identify intent, expand synonyms |
| **Ranking** | Score and rank results by relevance (PageRank, freshness, quality) |
| **Serving** | Return top-10 results with snippets in < 200ms |
| **Autocomplete** | Suggest queries as user types |
| **Spell correction** | "Did you mean..." for typos |

### Non-Functional Requirements

```
Scale:
  - 100,000 queries per second (8.5B/day)
  - Index: 100+ billion web pages (hundreds of PB of raw content)
  - Crawl: refresh billions of pages daily (freshness)
  - Latency: P99 < 500ms, P50 < 200ms
  - Availability: 99.99% (4 minutes downtime/year)
  - Global: serve from closest data center
```

---

## 📡 API Design (RADIO: A)

```
GET /search?q=system+design+interview&page=1&lang=en&region=us
  Response: {
    "results": [
      {
        "title": "System Design Interview Guide",
        "url": "https://example.com/system-design",
        "snippet": "Learn how to ace your system design interview...",
        "favicon": "https://example.com/favicon.ico"
      },
      ...
    ],
    "total_results": 2340000,
    "did_you_mean": null,
    "related_searches": ["system design primer", "system design examples"],
    "time_taken_ms": 180
  }

GET /autocomplete?q=system+des&lang=en
  Response: { "suggestions": ["system design interview", "system design primer", ...] }
```

---

## 🗄️ Data Model (RADIO: D)

```
Core data structures:

1. DOCUMENT STORE (the raw crawled web):
   document: { doc_id, url, title, body_text, crawl_timestamp, content_hash, links_out[] }
   Storage: distributed file system (GFS/Colossus), hundreds of PB

2. INVERTED INDEX (the searchable structure):
   term → [(doc_id, tf, positions[]), (doc_id, tf, positions[]), ...]
   
   Example:
     "system"  → [doc_42(tf=5), doc_88(tf=3), doc_102(tf=8), ...]
     "design"  → [doc_42(tf=3), doc_88(tf=7), doc_201(tf=2), ...]
     "interview" → [doc_42(tf=2), doc_500(tf=10), ...]
   
   Storage: sharded across thousands of servers (each shard holds a portion of the index)

3. PAGE RANK SCORES (link graph analysis):
   page_rank: { doc_id → score }  (pre-computed offline, updated weekly)

4. URL FRONTIER (crawl scheduler):
   { url, priority, last_crawl_time, crawl_interval, domain_crawl_rate }
```

---

## 🏗️ Infrastructure (RADIO: I)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    GOOGLE SEARCH ARCHITECTURE                            │
│                                                                          │
│  ═══════════════ OFFLINE PIPELINE (batch, continuous) ═══════════════   │
│                                                                          │
│  [URL Frontier] ──▶ [Web Crawler] ──▶ [Document Store (GFS)]           │
│       │                 (10K crawlers, politeness: 1 req/sec/domain)     │
│       │                                     │                            │
│       └──── new URLs discovered ◀───────────┘                           │
│                                             ▼                            │
│                                    [Content Processor]                    │
│                                    - Parse HTML/remove boilerplate       │
│                                    - Extract text, title, links          │
│                                    - Detect language, spam, duplicates   │
│                                             │                            │
│                                             ▼                            │
│                              ┌──────────────────────────┐                │
│                              │   INDEXING PIPELINE        │               │
│                              │   (MapReduce / streaming)  │               │
│                              │                            │               │
│                              │  1. Tokenize text          │               │
│                              │  2. Build inverted index   │               │
│                              │  3. Compute TF-IDF scores  │               │
│                              │  4. Store posting lists     │               │
│                              └──────────────┬─────────────┘               │
│                                             │                            │
│  [PageRank Computation] ◀──── link graph ───┘                           │
│    (iterative algorithm on the entire web graph)                         │
│                                                                          │
│  ═══════════════ ONLINE SERVING (real-time queries) ═══════════════     │
│                                                                          │
│  [User: "system design"] ──▶ [Query Service]                            │
│                                      │                                   │
│                              1. Parse query                              │
│                              2. Spell check / expand synonyms            │
│                              3. Fan-out to index shards                   │
│                                      │                                   │
│                    ┌─────────────────┼─────────────────┐                 │
│                    ▼                 ▼                  ▼                 │
│             [Shard 1]          [Shard 2]         [Shard N]               │
│             (terms A-D)        (terms E-L)       (terms ...)             │
│                    │                 │                  │                 │
│                    └─────────────────┼─────────────────┘                 │
│                                      ▼                                   │
│                              [Merger / Ranker]                            │
│                              - Intersect posting lists                    │
│                              - Score: BM25 + PageRank + freshness        │
│                              - Apply personalization                      │
│                              - Return top 10                             │
│                                      │                                   │
│                                      ▼                                   │
│                              [Snippet Generator]                          │
│                              - Extract relevant passage                   │
│                              - Highlight query terms                      │
│                              - Return in < 200ms total                    │
└──────────────────────────────────────────────────────────────────────────┘
```

### Query Processing Deep Dive

```
User types: "systm desgn interview tips"

Step 1: SPELL CORRECTION (< 5ms)
  "systm" → "system" (edit distance 1, high frequency correction)
  "desgn" → "design" (edit distance 1)
  Corrected query: "system design interview tips"

Step 2: QUERY EXPANSION (< 5ms)
  Synonyms: "tips" → also match "advice", "guide"
  Compound: "system design" recognized as a phrase (boost exact phrase matches)

Step 3: INDEX LOOKUP (< 50ms, parallelized across shards)
  Find posting lists for: "system", "design", "interview", "tips"
  Intersect: documents containing ALL terms (or high-scoring subset)
  Each shard returns its top-K candidates (e.g., top 100)

Step 4: RANKING (< 50ms)
  Score each candidate document:
    BM25 score (text relevance)          × 0.3
    PageRank (authority/link quality)     × 0.2
    Freshness (recently updated pages)   × 0.1
    Click-through rate (historical)      × 0.2
    Content quality (spam score, E-A-T)  × 0.2
  
  Sort by combined score, take top 10

Step 5: SNIPPET GENERATION (< 20ms)
  For each top-10 result:
    Find the passage most relevant to the query
    Highlight matching terms in bold
    Truncate to 150 characters

Step 6: RESPONSE (< 200ms total)
  Return results with title, URL, snippet, favicon
```

---

## 📈 Optimization (RADIO: O)

```
1. INDEX SHARDING STRATEGY
   Problem: 100 billion pages can't fit on one machine
   Solution: Two-level sharding:
     - Document sharding: each shard holds a subset of documents (by doc_id hash)
     - Term partitioning: for extremely common terms, partition the posting list itself
   Each query fans out to all document shards in parallel
   Each shard returns its local top-K results
   A coordinator merges and re-ranks the global top-K

2. CACHING (query results + index)
   - Query cache: 30%+ of queries are repeated ("weather", "news", "facebook")
     Store serialized result pages in Memcached (TTL: 5 minutes)
   - Posting list cache: hot terms ("the", "is", common nouns) cached in RAM
   - Result page cache: popular queries serve from cache without touching index

3. TIERED INDEX
   - Tier 1 (in-memory): top 1B highest-quality pages, serves 80% of queries
   - Tier 2 (SSD): next 10B pages, consulted if Tier 1 has insufficient results
   - Tier 3 (disk): full 100B page index, for rare/long-tail queries only
   Most queries are answered from Tier 1 alone (fastest)

4. CRAWL PRIORITIZATION
   - Crawl frequency proportional to page change rate:
     News sites: every 5 minutes
     Blogs: daily
     Static pages: weekly
   - Discover new URLs from sitemap.xml, RSS feeds, link analysis
```

---

## 🧩 Mini Challenge

**Design the autocomplete system. When a user types "sys", suggest "system design interview", "system requirements", etc. How do you serve suggestions in < 50ms for 100K QPS?**

<details>
<summary>💡 Click to reveal answer</summary>

**Data structure**: Precomputed Trie with frequency scores

```
Build (offline, updated hourly from query logs):
  - Aggregate query frequencies from search logs
  - Build a trie where each node stores top-K completions for that prefix
  
  Trie structure:
    "s" → [system (10M), samsung (8M), spotify (7M), ...]
    "sy" → [system (10M), sydney (3M), synonym (1M), ...]
    "sys" → [system design (5M), system requirements (2M), system32 (1M), ...]
    "syst" → [system design interview (3M), system design primer (2M), ...]

Storage: Trie serialized into a compact format, replicated to serving nodes

Serving (online):
  - User types "sys" → hash prefix to correct serving shard
  - Lookup "sys" node in in-memory trie → return pre-computed top-10 suggestions
  - Latency: < 5ms (single hash lookup + memory read)
  
Personalization:
  - Blend global popular completions with user's recent search history
  - User who recently searched "system design" → boost "system design" suggestions

Scaling:
  - Shard by prefix range: server A handles a-m, server B handles n-z
  - Each server holds its trie entirely in RAM (~10GB for all prefixes)
  - 100K QPS distributed across 50 servers = 2K QPS/server (trivial)

Freshness:
  - Trending queries (breaking news) injected via streaming pipeline
  - "earthquake" trends → immediately appears in suggestions for "ear..."
```

</details>

---

## 📝 Interview Q&A

**Q: How does PageRank work?**
> A: PageRank models the web as a directed graph where pages are nodes and hyperlinks are edges. A page's rank is proportional to the number and quality of pages linking to it. Intuitively, a page linked by many high-quality pages is itself high-quality. The algorithm iteratively computes scores: each page distributes its rank equally among all pages it links to. After many iterations (~50-100), scores converge. A damping factor (0.85) models the probability that a random surfer follows a link vs. jumping to a random page. This prevents rank sinks (pages with no outgoing links) from accumulating all the rank.

**Q: How do you keep the search index fresh when web pages change constantly?**
> A: Continuous crawling with adaptive scheduling. Pages are prioritized by: (1) historical change frequency — news sites re-crawled every few minutes, static pages weekly; (2) importance — high-PageRank pages crawled more frequently; (3) signals — RSS feeds, sitemap.xml lastmod timestamps, PubSubHubbub notifications. When a changed page is re-crawled, it enters a streaming indexing pipeline that updates the inverted index incrementally (not a full rebuild). The index is served from a combination of a base index (rebuilt weekly) plus incremental updates (applied in real-time).

---

## 🔗 What to Read Next

1. **[BuildingBlocks/SearchIndex.md](../BuildingBlocks/SearchIndex.md)** — Deep dive into inverted indexes and search infrastructure
2. **[SystemDesignCaseStudies/DesignTwitter.md](./DesignTwitter.md)** — Search within a social platform (different scale, same principles)
3. **[BuildingBlocks/Caching.md](../BuildingBlocks/Caching.md)** — Query caching is critical for search performance

---

*[← Design YouTube](./DesignYouTube.md) | [Back to Case Studies](./README.md) | [Next: Design Dropbox →](./DesignDropbox.md)*
