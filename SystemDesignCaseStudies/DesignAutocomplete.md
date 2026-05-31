# 🔍 Design Autocomplete / Typeahead Suggestions

> *"Every time you type in Google's search box, you see suggestions appear within 100ms — before you even finish the word. Behind those 'simple' suggestions: a trie serving 5.5 BILLION searches per day with sub-100ms latency across every country on Earth. The system must balance freshness (trending topics), personalization (your search history), and speed (faster than your typing speed!)."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟢 Easy-Medium | **🔗 Prerequisites**: [Caching](../../BuildingBlocks/Caching.md), [CDN](../../BuildingBlocks/CDN.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Core Data Structure: Trie](#-core-data-structure-trie)
3. [High-Level Architecture](#-high-level-architecture)
4. [Data Collection Pipeline](#-data-collection-pipeline)
5. [Serving Layer](#-serving-layer)
6. [Ranking & Personalization](#-ranking--personalization)
7. [Scaling Strategies](#-scaling-strategies)
8. [Java Implementation](#-java-implementation)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Return top 5-10 suggestions as user types each character
✅ Support prefix matching ("sys" → "system design", "system admin"...)
✅ Rank suggestions by relevance (popularity, recency, personalization)
✅ Support multiple languages
✅ Handle trending/breaking topics (suggest "earthquake" during one)
✅ Filter inappropriate suggestions
✅ Personalized suggestions based on user history
```

### Non-Functional Requirements
```
✅ Latency < 100ms (must feel instant!)
✅ Availability: 99.99% (users type every second!)
✅ Scale: billions of queries/day
✅ Storage: billions of possible suggestions
✅ Freshness: trending topics within 15 minutes
```

### Scale Estimation
```
Users: 1 billion DAU
Searches/user/day: 6
Characters/search: ~20 (average)
Queries for autocomplete: 20 chars × 6 searches × 1B = 120B prefix lookups/day!

Wait... 120 BILLION? Not all characters trigger API calls!
With debouncing (200ms): ~5 requests per search
→ 30 BILLION autocomplete requests/day
→ 350,000 QPS (average), 700K QPS (peak)
```

---

## 🌳 Core Data Structure: Trie

```
WHY TRIE (Prefix Tree)?
  • O(L) lookup where L = length of prefix (not affected by dataset size!)
  • Natural prefix matching (all words starting with "sys...")
  • Space-efficient with shared prefixes
  
BASIC TRIE STRUCTURE:
                    (root)
                   /  |   \
                  s   d    m
                 /    |     \
                y     e      a
               /      |       \
              s       s        p
             /        |         \
            t         i          s
           /          |
          e           g
         /            |
        m             n

  "system" → follow s→y→s→t→e→m → found! Weight: 95,000 searches/day
  "design" → follow d→e→s→i→g→n → found! Weight: 82,000 searches/day

OPTIMIZED: COMPRESSED TRIE (Patricia Trie)
  Instead of one node per character, compress single-child chains:
  
                    (root)
                   /  |   \
               "sys" "des" "map"
               /      |      \
            "tem"  "ign"    "s"
            /  \
         " d" " a"
         /       \
     "esign"  "dmin"
     
  "system design" → 3 hops instead of 13!
  MUCH less memory, same O(L) lookup.
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      USER TYPING                            │
│          "sys" → "syst" → "syste" → "system"              │
└──────────────────────────┬──────────────────────────────────┘
                           │ (debounced, every 200ms)
                           ▼
            ┌──────────────────────────────┐
            │         CDN / Edge           │ ← Cache popular prefixes!
            │   ("sys" → cached results!)  │    80%+ hit rate!
            └──────────────────┬───────────┘
                               │ (cache miss)
                               ▼
            ┌──────────────────────────────┐
            │    Autocomplete Service       │
            │  (Trie in memory + ranking)  │
            └──────────────────┬───────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
     ┌────────────┐   ┌────────────┐   ┌──────────────┐
     │   Trie     │   │  User      │   │  Trending    │
     │  Storage   │   │  History   │   │  Service     │
     │ (Prebuilt) │   │  Cache     │   │  (Real-time) │
     └────────────┘   └────────────┘   └──────────────┘
              ▲
              │ (rebuilt every 15 min)
     ┌────────────────────────────────┐
     │   Analytics / Data Pipeline    │
     │   (count search frequencies)   │
     └────────────────────────────────┘
```

---

## 📊 Data Collection Pipeline

```
HOW DO WE KNOW WHAT TO SUGGEST?
  → By analyzing what people actually search for!

PIPELINE:
  1. User searches "system design interview"
  2. Search log: { query: "system design interview", timestamp, user_id }
  3. Aggregator (every 15 min): Count queries by prefix
  4. Build frequency map:
     "s" → [("system design", 95K), ("spotify", 80K), ("sql", 75K)...]
     "sy" → [("system design", 95K), ("synonym", 12K)...]
     "sys" → [("system design", 95K), ("system admin", 15K)...]
     
  5. Rebuild Trie with updated weights
  6. Deploy new Trie to serving nodes

FRESHNESS vs. STABILITY:
  Problem: If we only count recent queries, suggestions are volatile!
  Solution: EXPONENTIAL DECAY
  
  weight(query) = Σ count_i × decay^(age_in_hours_i)
  
  A query searched 1000 times yesterday has more weight
  than one searched 1000 times last month.
  
  For trending topics: separate "trending boost" signal
  Earthquake happening NOW → huge spike in last hour → boost!

FILTERING:
  Before adding to Trie, run through:
  1. Profanity filter (blocklist)
  2. Violence/hate speech ML classifier
  3. Legal removal requests (DMCA, right to be forgotten)
  4. Minimum threshold (need 100+ searches to be suggested)
```

---

## 🚀 Serving Layer

```
CLIENT-SIDE OPTIMIZATIONS:
  
  1. DEBOUNCING: Don't send request on EVERY keystroke!
     User types "system" → 6 keystrokes
     Without debounce: 6 API calls
     With 200ms debounce: 2-3 API calls (wait for pause in typing)
     
  2. BROWSER CACHE: Cache prefix→results locally
     "sys" → results cached in browser for 5 minutes
     If user types "sys" again → no network request!
     
  3. PREFETCH: When user types "s", preload "sy" suggestions
     By the time they type second char, results are ready!

SERVER-SIDE OPTIMIZATIONS:

  1. IN-MEMORY TRIE: Entire trie lives in RAM
     10 million suggestions × ~100 bytes = 1 GB (fits in memory!)
     
  2. CDN CACHING: Popular prefixes served from edge
     "a", "b", "c"... single-char results = VERY popular, always cached
     Top 1000 prefixes = 80% of all requests → CDN hit!
     
  3. TRIE SHARDING: For massive tries
     Shard by first character:
     Server A: prefixes starting with a-f
     Server B: prefixes starting with g-m
     Server C: prefixes starting with n-s
     Server D: prefixes starting with t-z

REQUEST FLOW:
  1. Client types "sys" (after 200ms debounce)
  2. Check browser cache → miss
  3. Request → CDN → cache hit? Return!
  4. CDN miss → Route to autocomplete server for "s" shard
  5. Server: trie.search("sys") → top 10 by weight
  6. Merge with: user's personal history + trending boost
  7. Return ranked results (< 50ms server-side)
```

---

## 🎯 Ranking & Personalization

```
RANKING FORMULA:
  score(suggestion, user, context) = 
      w1 × popularity_score          (global frequency)
    + w2 × recency_score             (exponential decay)
    + w3 × personalization_score     (user history match)
    + w4 × trending_boost            (currently trending?)
    + w5 × context_score             (location, time, device)
    
  Where:
    w1 = 0.4 (most important: what everyone searches)
    w2 = 0.2 (recent > old)
    w3 = 0.25 (personalization is powerful!)
    w4 = 0.1 (trending boost)
    w5 = 0.05 (context relevance)

PERSONALIZATION EXAMPLE:
  User who frequently searches Java topics:
  
  Types "str" →
    Regular: ["stranger things", "streaming", "street view"]
    Personalized: ["string java", "streams api", "strategy pattern"]
    
  Stored per user: recent 1000 searches + category weights
  Lookup: Redis hash { userId → recent_queries[] }
  
  Privacy: Only store on user's device + encrypted server-side
  Allow users to clear history!
```

---

## 📈 Scaling Strategies

```
MULTI-REGION DEPLOYMENT:
  
  US users → US edge servers (US-focused trie)
  India users → India edge servers (India-focused trie)
  Japan users → Japan edge servers (Japan-focused trie)
  
  Why? Different countries = different popular queries!
  Reduces latency + more relevant suggestions.

TRIE UPDATE STRATEGY:
  Can't rebuild 1 GB trie on every search!
  
  Strategy: PERIODIC REBUILD + REAL-TIME DELTA
  
  Base Trie: rebuilt every 15 minutes from analytics pipeline
  Delta Trie: small trie of "hot" queries from last 15 min
  
  Search: merge results from Base + Delta tries
  
  On rebuild: Delta merged into new Base → Delta cleared
  
  This gives: stable suggestions + trending freshness!

STORAGE ESTIMATION:
  Unique suggestions: 100 million
  Average length: 20 characters
  Trie nodes (with compression): ~500 million
  Node size: ~50 bytes (char + children pointers + weight)
  Total: ~25 GB for full trie
  
  But! Top 10M suggestions cover 99% of queries → 2.5 GB
  This FITS IN MEMORY on a single server!
  Shard across 4 servers for redundancy + throughput.
```

---

## 💻 Java Implementation

### Trie Node & Search

```java
public class AutocompleteTrie {
    
    private final TrieNode root = new TrieNode();
    
    static class TrieNode {
        Map<Character, TrieNode> children = new HashMap<>();
        // Top-K suggestions at this prefix (pre-computed!)
        List<Suggestion> topSuggestions = new ArrayList<>();
        boolean isEndOfWord;
    }
    
    record Suggestion(String text, long weight) 
        implements Comparable<Suggestion> {
        
        @Override
        public int compareTo(Suggestion other) {
            return Long.compare(other.weight, this.weight); // Descending
        }
    }
    
    /**
     * Insert a suggestion with its weight.
     * Also updates topSuggestions at every prefix node!
     */
    public void insert(String word, long weight) {
        TrieNode current = root;
        Suggestion suggestion = new Suggestion(word, weight);
        
        for (char c : word.toCharArray()) {
            current = current.children
                .computeIfAbsent(c, k -> new TrieNode());
            
            // Maintain top-K at each node (pre-computation!)
            updateTopK(current.topSuggestions, suggestion, 10);
        }
        current.isEndOfWord = true;
    }
    
    /**
     * Search: O(L) where L = prefix length!
     * Results are PRE-COMPUTED at each node!
     */
    public List<Suggestion> search(String prefix, int limit) {
        TrieNode current = root;
        
        for (char c : prefix.toCharArray()) {
            current = current.children.get(c);
            if (current == null) return Collections.emptyList();
        }
        
        return current.topSuggestions.stream()
            .limit(limit)
            .collect(Collectors.toList());
    }
    
    private void updateTopK(List<Suggestion> topK, 
                           Suggestion candidate, int k) {
        // Remove old entry for same word (weight may have changed)
        topK.removeIf(s -> s.text().equals(candidate.text()));
        topK.add(candidate);
        Collections.sort(topK);
        if (topK.size() > k) {
            topK.remove(topK.size() - 1); // Remove lowest weight
        }
    }
}
```

### Autocomplete Service with Personalization

```java
@RestController
@RequestMapping("/api/v1/autocomplete")
public class AutocompleteController {
    
    @Autowired private AutocompleteTrie globalTrie;
    @Autowired private TrendingService trendingService;
    @Autowired private UserHistoryService userHistory;
    
    @GetMapping("/suggest")
    @Cacheable(value = "suggestions", key = "#prefix + #userId")
    public List<SuggestionDTO> suggest(
            @RequestParam String prefix,
            @RequestParam(required = false) String userId,
            @RequestParam(defaultValue = "10") int limit) {
        
        if (prefix.length() < 1) return Collections.emptyList();
        
        // 1. Get global suggestions from trie
        List<Suggestion> global = globalTrie.search(
            prefix.toLowerCase(), limit * 2);
        
        // 2. Get trending suggestions
        List<Suggestion> trending = trendingService
            .getTrendingForPrefix(prefix);
        
        // 3. Get personalized suggestions
        List<Suggestion> personal = userId != null
            ? userHistory.getPersonalSuggestions(userId, prefix)
            : Collections.emptyList();
        
        // 4. Merge and rank
        return rankAndMerge(global, trending, personal, limit);
    }
    
    private List<SuggestionDTO> rankAndMerge(
            List<Suggestion> global, 
            List<Suggestion> trending,
            List<Suggestion> personal, int limit) {
        
        Map<String, Double> scores = new HashMap<>();
        
        // Weight: global popularity (0.4)
        for (Suggestion s : global) {
            scores.merge(s.text(), s.weight() * 0.4, Double::sum);
        }
        
        // Weight: trending boost (0.35)
        for (Suggestion s : trending) {
            scores.merge(s.text(), s.weight() * 0.35, Double::sum);
        }
        
        // Weight: personalization (0.25)
        for (Suggestion s : personal) {
            scores.merge(s.text(), s.weight() * 0.25, Double::sum);
        }
        
        return scores.entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(limit)
            .map(e -> new SuggestionDTO(e.getKey(), e.getValue()))
            .collect(Collectors.toList());
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Problem: Trending Topics Detection

Design a system that detects when a query suddenly spikes (10x normal volume in last hour) and boosts it in autocomplete within 5 minutes.

<details>
<summary>🔑 Answer</summary>

**Approach: Sliding Window Counter**

```java
public class TrendingDetector {
    
    // Redis: query → hourly counts for last 7 days
    // Key: "counts:{query}:{hourOfDay}" → count
    
    public boolean isTrending(String query) {
        long currentHourCount = getCountLastHour(query);
        long avgHourlyCount = getAverageHourlyCount(query, 7); // 7-day avg
        
        // Trending = current hour is 10x+ the weekly average!
        return currentHourCount > avgHourlyCount * 10 
            && currentHourCount > 1000; // Minimum threshold
    }
    
    // Run every 5 minutes: scan top 10K queries for spikes
    @Scheduled(fixedRate = 300_000) 
    public void detectTrending() {
        getTopQueries(10000).stream()
            .filter(this::isTrending)
            .forEach(query -> {
                trendingCache.put(query, calculateBoost(query));
                // Boost will be merged into autocomplete results
            });
    }
}
```

**Key insight:** Don't scan ALL queries (millions!) — only scan top-N by recent volume. If something is trending, it MUST appear in recent top-N.
</details>

---

## ❓ Interview Q&A

**Q1: Why use a Trie instead of a database with LIKE queries?**
> `SELECT * FROM suggestions WHERE text LIKE 'sys%' ORDER BY weight LIMIT 10` works but is SLOW at scale. A B-tree index helps with prefix matching but still requires disk I/O and sorting. A Trie is O(L) for prefix lookup (L = prefix length, independent of dataset size!), lives entirely in memory, and can pre-compute top-K at each node for O(1) result retrieval. At 300K+ QPS, database would crumble; Trie serves from RAM in microseconds.

**Q2: How do you handle real-time trending topics?**
> Two-layer approach: (1) Base trie rebuilt every 15 minutes from analytics, (2) Small "delta trie" updated every 5 minutes with trending queries detected by spike detection (current_count > 10× weekly_average). Merge results at query time. This gives stability (base) + freshness (delta) without expensive full rebuilds.

**Q3: How would you shard the trie for a global system?**
> Multiple dimensions: (1) By first character for load distribution (26 shards), (2) By geography for latency + relevance (US/EU/Asia tries with different popular queries), (3) By language for multilingual support. Each region gets a replica set for redundancy. CDN caches popular short prefixes (1-2 chars) at the edge.

**Q4: What about privacy concerns with autocomplete?**
> (1) Don't suggest queries from fewer than N users (anonymity threshold), (2) Aggressive profanity/hate filtering before suggestions enter the system, (3) "Right to be forgotten" — API to remove specific suggestions, (4) Personalization data stored encrypted, user can delete, (5) Don't suggest sensitive categories (medical, legal) unless user explicitly searching that domain.

**Q5: How do you handle multi-word queries?**
> Treat entire phrase as one entry: "system design" is one trie path, not two separate words. For "system d", search the trie for that exact prefix. Additionally: maintain word-level index for "did you mean" corrections and phrase suggestions based on word N-grams.

---

## 🔗 Related Topics
- [Caching](../../BuildingBlocks/Caching.md) — CDN and browser caching for autocomplete
- [CDN](../../BuildingBlocks/CDN.md) — Edge caching of popular prefix results
- [Search Index](../../BuildingBlocks/SearchIndex.md) — Full search after autocomplete selection
- [Rate Limiting](../../BuildingBlocks/RateLimiting.md) — Protecting autocomplete from abuse

---

*"The best autocomplete is the one that knows what you want before you do. The worst autocomplete is the one that shows you embarrassing suggestions from 2012." — Every User Ever* 🔍
