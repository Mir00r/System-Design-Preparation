# 🔍 Search Index: Full-Text Search at Scale

> *"Users expect search to 'just work' — type a few characters, get relevant results in 200ms, handle typos, and rank by relevance. Behind that magic is an inverted index, BM25 scoring, and a distributed query engine processing millions of documents."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Database Indexing](../Database/Indexing.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [How Search Indexes Work](#-how-search-indexes-work)
3. [Inverted Index — The Core Data Structure](#-inverted-index--the-core-data-structure)
4. [Elasticsearch Architecture](#-elasticsearch-architecture)
5. [When to Use What](#-when-to-use-what)
6. [Spring Boot + Elasticsearch](#-spring-boot--elasticsearch)
7. [Common Pitfalls](#-common-pitfalls)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
SQL LIKE queries don't scale for search:

SELECT * FROM products WHERE name LIKE '%wireless headphones%'

Problems:
  1. Full table scan (no index can help with leading %)     → O(n) for every query
  2. No relevance ranking (all matches are "equal")         → user gets random order
  3. No typo tolerance ("wirless" → zero results)           → user gives up
  4. No synonym handling ("headphones" vs "earbuds")        → missed relevant results
  5. No tokenization ("Noise-Cancelling" won't match "noise cancelling")
  6. Latency: 2+ seconds on millions of rows               → unacceptable UX

What you need: a purpose-built search engine with:
  ✅ Inverted index (millisecond lookups on any word)
  ✅ Relevance scoring (TF-IDF / BM25)
  ✅ Fuzzy matching (typo tolerance)
  ✅ Tokenization & stemming ("running" → "run")
  ✅ Distributed sharding (handle billions of documents)
```

---

## 🏗️ How Search Indexes Work

```
Document ingestion pipeline:

  Raw document: "Wireless Noise-Cancelling Headphones - Sony WH-1000XM5"
                              │
                              ▼
  [Analyzer Pipeline]
    1. Character filter:  "Wireless Noise-Cancelling Headphones - Sony WH-1000XM5"
       → Remove special chars: "Wireless Noise Cancelling Headphones Sony WH 1000XM5"
    
    2. Tokenizer (split into tokens):
       → ["Wireless", "Noise", "Cancelling", "Headphones", "Sony", "WH", "1000XM5"]
    
    3. Token filters:
       Lowercase:  ["wireless", "noise", "cancelling", "headphones", "sony", "wh", "1000xm5"]
       Stemming:   ["wireless", "nois", "cancel", "headphon", "sony", "wh", "1000xm5"]
       Synonyms:   ["wireless", "nois", "cancel", "headphon", "earphon", "sony", "wh", "1000xm5"]
                              │
                              ▼
  [Inverted Index] ← stores: term → [list of document IDs containing that term]
```

---

## 📖 Inverted Index — The Core Data Structure

```
Traditional index (forward):
  Document 1 → ["wireless", "headphones", "sony"]
  Document 2 → ["bluetooth", "headphones", "bose"]
  Document 3 → ["wireless", "earbuds", "apple"]

  To find "wireless": scan ALL documents → O(n)

Inverted index:
  "wireless"   → [Doc 1, Doc 3]          ← instant lookup
  "headphones" → [Doc 1, Doc 2]          ← instant lookup
  "bluetooth"  → [Doc 2]
  "sony"       → [Doc 1]
  "bose"       → [Doc 2]
  "earbuds"    → [Doc 3]
  "apple"      → [Doc 3]

  Search "wireless headphones":
    "wireless"   → {Doc 1, Doc 3}
    "headphones" → {Doc 1, Doc 2}
    Intersection  → {Doc 1} ← matches both terms
    Union (OR)    → {Doc 1, Doc 2, Doc 3} ← matches either term

  Lookup time: O(1) per term (hash map) + O(posting list size) for intersections
```

### BM25 Relevance Scoring

```
How results are ranked (simplified BM25):

  Score(doc, query) = Σ for each term in query:
    IDF(term) × [ TF(term, doc) × (k+1) / (TF(term, doc) + k × (1 - b + b × |doc| / avgDocLen)) ]

  Where:
    TF  = Term Frequency (how often the term appears in THIS document)
    IDF = Inverse Document Frequency (rarer terms get higher weight)
          "the" appears in 99% of docs → low IDF (not useful for ranking)
          "kubernetes" appears in 0.1% of docs → high IDF (very discriminating)
    |doc| / avgDocLen = document length normalization (don't unfairly boost long docs)

  Intuition: Documents that contain rare query terms, mentioned frequently,
             in relatively short documents → score highest.
```

---

## 🏛️ Elasticsearch Architecture

```
Elasticsearch cluster:

  [Client] → [Coordinating Node] → routes query to correct shards
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
  [Data Node 1]  [Data Node 2]  [Data Node 3]
  Shard 0 (P)    Shard 1 (P)    Shard 2 (P)
  Shard 1 (R)    Shard 2 (R)    Shard 0 (R)

  P = Primary shard (handles writes)
  R = Replica shard (handles reads + failover)

  Index "products" → split into 3 primary shards
  Each shard is a complete Lucene index (self-contained search engine)
  Replicas provide: read scaling + fault tolerance
```

### Write Path

```
1. Client sends: PUT /products/_doc/123 { "name": "Sony headphones", ... }
2. Coordinating node routes to correct primary shard (hash(doc_id) % num_shards)
3. Primary shard indexes the document (analyze → add to inverted index)
4. Primary replicates to replica shards (synchronous for acknowledged writes)
5. Returns 201 Created to client

Near-real-time (NRT): document is searchable ~1 second after indexing
  (Lucene "refresh" opens a new index reader; configurable per-index)
```

### Search Path

```
1. Client sends: GET /products/_search { "query": { "match": { "name": "wireless headphones" } } }
2. Coordinating node fans out query to ALL shards (primary or replica)
3. Each shard runs the query locally against its inverted index
4. Each shard returns top-N results (sorted by BM25 score)
5. Coordinating node merges results from all shards, re-ranks, returns top-N globally
```

---

## 📊 When to Use What

| Solution | Best For | Not For | Latency |
|---|---|---|---|
| **PostgreSQL Full-Text** | Small datasets (< 1M docs), simple search | Complex queries, fuzzy matching | 50-500ms |
| **Elasticsearch** | Full-text search, analytics, logging | Primary data store, transactions | 10-100ms |
| **Apache Solr** | Enterprise search (similar to ES) | Real-time analytics | 10-100ms |
| **MeiliSearch** | Simple API, instant search, typo tolerance | Large scale (> 10M docs) | < 50ms |
| **Algolia** | Instant search SaaS, zero infra overhead | Cost-sensitive, large datasets | < 20ms |
| **Typesense** | Open-source Algolia alternative | Very large datasets | < 50ms |

---

## ⚙️ Spring Boot + Elasticsearch

```java
// Spring Data Elasticsearch — repository pattern
@Document(indexName = "products")
public class Product {
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "standard")
    private String name;
    
    @Field(type = FieldType.Text, analyzer = "english")  // stemming enabled
    private String description;
    
    @Field(type = FieldType.Keyword)  // exact match only (for filtering)
    private String category;
    
    @Field(type = FieldType.Double)
    private double price;
    
    @Field(type = FieldType.Date)
    private Instant createdAt;
}

public interface ProductSearchRepository extends ElasticsearchRepository<Product, String> {
    // Auto-generated query from method name
    List<Product> findByNameContaining(String query);
}

// Custom search with relevance tuning
@Service
public class ProductSearchService {

    private final ElasticsearchOperations operations;

    public SearchHits<Product> search(String query, String category, int page) {
        NativeQuery searchQuery = NativeQuery.builder()
            .withQuery(q -> q
                .bool(b -> b
                    .must(m -> m
                        .multiMatch(mm -> mm
                            .query(query)
                            .fields("name^3", "description^1")  // name 3x more important
                            .fuzziness("AUTO")                   // typo tolerance
                            .type(TextQueryType.BEST_FIELDS)
                        )
                    )
                    .filter(f -> category != null ? 
                        f.term(t -> t.field("category").value(category)) : f.matchAll(ma -> ma)
                    )
                )
            )
            .withPageable(PageRequest.of(page, 20))
            .withHighlightQuery(new HighlightQuery(
                new Highlight(List.of(new HighlightField("name"), new HighlightField("description"))),
                Product.class
            ))
            .build();

        return operations.search(searchQuery, Product.class);
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Using Elasticsearch as a primary database** — ES is not ACID-compliant. Data can be lost during node failures between refreshes. Always store the source of truth in PostgreSQL/MySQL and sync to ES for search. If ES crashes, rebuild the index from the primary database.

2. **Too many shards** — Each shard has overhead (~50MB memory for metadata). An index with 100 shards on 3 nodes wastes resources. Rule of thumb: 10-50GB per shard, number of shards ≤ number of data nodes × 3.

3. **Not defining mappings upfront** — ES auto-detects field types, often incorrectly (numbers as text, dates as keywords). Always define explicit mappings before indexing data.

4. **Indexing and searching on the same cluster without separation** — Heavy indexing (e.g., log ingestion) steals resources from search queries. Use separate clusters or dedicated node roles (ingest nodes vs search nodes).

5. **Not using `keyword` type for filters** — Filtering on a `text` field (analyzed) is slow and may produce incorrect results. Use `keyword` type for fields you filter/aggregate on (category, status, country) and `text` type for fields you search full-text on.

---

## 🧩 Mini Challenge

**You're building an e-commerce search engine.** A user types "red nke shoes" (with a typo — meant "nike"). How do you ensure they get relevant results?

<details>
<summary>💡 Click to reveal answer</summary>

**Multiple layers of tolerance**:

1. **Fuzzy matching** (handles typos): `"fuzziness": "AUTO"` in Elasticsearch allows 1-2 character edits. "nke" → "nike" (1 insertion). The query becomes: match documents where `name` is within edit distance 2 of each query token.

2. **Synonym expansion** (handles brand variations): Configure a synonym filter: `"nike, nke, nikey => nike"`. This catches common misspellings at index time.

3. **N-gram indexing** (handles partial matches): Index "nike" as `["ni", "ik", "ke", "nik", "ike", "nike"]`. The typo "nke" shares n-grams "nk" and "ke" with "nike".

4. **Did-you-mean suggestion**: Use the ES Suggesters API (`term` or `phrase` suggester) to propose "Did you mean: red **nike** shoes?" based on terms that actually exist in the index.

5. **Query pipeline**:
```json
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name": { "query": "red nke shoes", "fuzziness": "AUTO", "boost": 2 } } },
        { "match": { "name": { "query": "red nke shoes" } } },
        { "match": { "color": "red" } },
        { "match": { "brand": { "query": "nke", "fuzziness": 1 } } }
      ],
      "minimum_should_match": 1
    }
  },
  "suggest": {
    "text": "red nke shoes",
    "simple_phrase": { "phrase": { "field": "name.suggest" } }
  }
}
```

</details>

---

## 📝 Interview Q&A

**Q: What is an inverted index and why is it faster than SQL LIKE?**
> A: An inverted index maps each unique term to the list of documents containing that term (term → [doc_ids]). Searching "headphones" is a single hash lookup returning all matching document IDs — O(1) regardless of total documents. SQL LIKE '%headphones%' scans every row sequentially — O(n). Additionally, inverted indexes support relevance scoring (BM25), tokenization, stemming, and fuzzy matching, which SQL LIKE cannot do.

**Q: How does Elasticsearch achieve horizontal scalability?**
> A: An ES index is split into shards (typically 1-5 per index). Each shard is a self-contained Lucene index. Shards are distributed across data nodes in the cluster. Search queries are fanned out to all shards in parallel, each shard returns its local top-N results, and the coordinating node merges them. Adding more nodes automatically rebalances shards, increasing both storage capacity and search throughput linearly.

**Q: When would you NOT use Elasticsearch?**
> A: (1) As a primary database — no ACID transactions, potential data loss; always keep a source of truth elsewhere. (2) For exact-match lookups by ID — a regular database with a primary key index is faster and cheaper. (3) For datasets < 100K documents — PostgreSQL full-text search (with `tsvector`/`tsquery`) is simpler and adequate. (4) For real-time writes that must be immediately searchable — ES has a 1-second refresh interval (configurable but costly). (5) When budget is extremely tight — ES clusters are resource-intensive (RAM-heavy).

---

## 🔗 What to Read Next

1. **[Database/Indexing.md](../Database/Indexing.md)** — Traditional database indexes (B-tree, hash) vs inverted indexes
2. **[BuildingBlocks/Caching.md](./Caching.md)** — Cache search results for popular queries to reduce ES load
3. **[SystemDesignCaseStudies/DesignTwitter.md](../SystemDesignCaseStudies/DesignTwitter.md)** — Twitter's search feature uses inverted indexes at massive scale

---

*[← Blob Storage](./Blob_Storage.md) | [Back to BuildingBlocks](../INDEX.md) | [Next: Message Queues Overview →](./MessageQueues.md)*
