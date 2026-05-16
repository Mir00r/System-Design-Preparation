# 🔍 Elasticsearch Deep Dive: The Distributed Search Engine for Full-Text and Analytics

> *"When you type 'python list comprehension' into Stack Overflow and get results in 50ms across 50 million questions, that's Elasticsearch. It turns unstructured text into searchable, analyzable, rankable knowledge — at scale."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [SQL vs NoSQL](./SQL_Vs_NoSQL.md), [Search Index](../BuildingBlocks/SearchIndex.md)

---

## 📋 Table of Contents
1. [What is Elasticsearch](#-what-is-elasticsearch)
2. [Architecture](#-architecture)
3. [Inverted Index](#-inverted-index)
4. [Mappings & Analyzers](#-mappings--analyzers)
5. [Query DSL](#-query-dsl)
6. [Aggregations](#-aggregations)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [Cluster Management & Scaling](#-cluster-management--scaling)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Elasticsearch

```
┌─────────────────────────────────────────────────────────┐
│               ELASTICSEARCH AT A GLANCE                  │
│                                                         │
│  Type:        Distributed search & analytics engine     │
│  Based on:    Apache Lucene                             │
│  Data:        JSON documents                            │
│  Query:       RESTful API + Query DSL (JSON)            │
│  Scaling:     Horizontal (sharding + replication)       │
│  Near RT:     ~1 second indexing-to-searchable latency  │
│  Use Cases:   Full-text search, log analytics, metrics  │
│  Ecosystem:   ELK Stack (Elasticsearch + Logstash + Kibana)│
└─────────────────────────────────────────────────────────┘

Key Concepts:
  Index    ≈ Database table (collection of documents)
  Document ≈ Row (JSON object)
  Field    ≈ Column
  Shard    ≈ Partition (subset of an index)
  Replica  ≈ Copy of a shard (for HA and read scaling)
```

---

## 🏗️ Architecture

```
CLUSTER ARCHITECTURE:

  [Client] → [Coordinating Node] → routes to correct shard
  
  ┌─────────────────────────────────────────────────────────┐
  │ CLUSTER: "production-search"                            │
  │                                                         │
  │  Node 1 (Master-eligible)    Node 2                     │
  │  ┌─────────────────────┐    ┌─────────────────────────┐│
  │  │ Shard 0 (Primary)   │    │ Shard 0 (Replica)       ││
  │  │ Shard 2 (Replica)   │    │ Shard 1 (Primary)       ││
  │  └─────────────────────┘    └─────────────────────────┘│
  │                                                         │
  │  Node 3                      Node 4                     │
  │  ┌─────────────────────┐    ┌─────────────────────────┐│
  │  │ Shard 2 (Primary)   │    │ Shard 1 (Replica)       ││
  │  │ Shard 1 (Replica)   │    │ Shard 2 (Replica)       ││
  │  └─────────────────────┘    └─────────────────────────┘│
  └─────────────────────────────────────────────────────────┘

NODE ROLES:
  Master:       cluster state, index creation/deletion, shard allocation
  Data:         stores shards, executes queries
  Coordinating: routes requests, merges results (no data storage)
  Ingest:       pre-process documents (pipelines)

WRITE FLOW:
  1. Client sends document to any node (coordinating)
  2. Coordinator routes to correct primary shard (hash(_id) % num_shards)
  3. Primary shard indexes document
  4. Primary replicates to replica shards
  5. Acknowledge to client

SEARCH FLOW (scatter-gather):
  1. Client sends query to coordinator
  2. Coordinator sends query to ALL shards (primary OR replica)
  3. Each shard returns top-N matching doc IDs + scores
  4. Coordinator merges results, fetches full documents
  5. Returns to client
```

---

## 📚 Inverted Index

```
WHY ELASTICSEARCH IS FAST FOR SEARCH:

  Traditional DB (B-tree index on text):
    SELECT * FROM articles WHERE body LIKE '%distributed systems%'
    → Full table scan! No index helps with substring matching.

  Elasticsearch (inverted index):
    Document 1: "Redis is an in-memory distributed cache"
    Document 2: "Cassandra is a distributed database for large-scale systems"
    Document 3: "MongoDB is a distributed document database"

  INVERTED INDEX (term → document list):
    ┌──────────────┬──────────────────┐
    │ Term         │ Documents        │
    ├──────────────┼──────────────────┤
    │ redis        │ [1]              │
    │ memory       │ [1]              │
    │ distributed  │ [1, 2, 3]       │
    │ cache        │ [1]              │
    │ cassandra    │ [2]              │
    │ database     │ [2, 3]          │
    │ large-scale  │ [2]              │
    │ systems      │ [2]              │
    │ mongodb      │ [3]              │
    │ document     │ [3]              │
    └──────────────┴──────────────────┘

  Search "distributed database":
    "distributed" → [1, 2, 3]
    "database"    → [2, 3]
    Intersection  → [2, 3] (both terms present)
    Score: Doc 2 has both terms + "distributed" in relevant context → higher score

  This is O(1) lookup per term, then set intersection!
  Works for: full-text search, fuzzy matching, stemming, synonyms
```

---

## 🔧 Mappings & Analyzers

```
ANALYZER PIPELINE (text processing at index time):

  Input: "The Quick Brown FOX jumped over the lazy dog!"
    │
    ▼ Character Filter (strip HTML, replace chars)
  "The Quick Brown FOX jumped over the lazy dog!"
    │
    ▼ Tokenizer (split into tokens)
  ["The", "Quick", "Brown", "FOX", "jumped", "over", "the", "lazy", "dog"]
    │
    ▼ Token Filters (lowercase, stemming, stop words)
  ["quick", "brown", "fox", "jump", "over", "lazi", "dog"]
    │
  Stored in inverted index

FIELD TYPES:
  text:     analyzed (full-text search) — "The quick brown fox" → ["quick","brown","fox"]
  keyword:  NOT analyzed (exact match, aggregations) — "user@email.com" stays as-is
  integer/long/double: numeric (range queries, sorting)
  date:     date parsing and range queries
  boolean:  true/false filtering
  geo_point: latitude/longitude

MAPPING EXAMPLE:
  PUT /products
  {
    "mappings": {
      "properties": {
        "name": { "type": "text", "analyzer": "english" },
        "sku": { "type": "keyword" },
        "price": { "type": "double" },
        "description": { "type": "text", "analyzer": "english" },
        "category": { "type": "keyword" },
        "created_at": { "type": "date" },
        "tags": { "type": "keyword" }  // array of keywords
      }
    }
  }
```

---

## 🔎 Query DSL

```json
// FULL-TEXT SEARCH with relevance scoring
{
  "query": {
    "bool": {
      "must": [
        { "match": { "description": "distributed database scalable" } }
      ],
      "filter": [
        { "term": { "category": "technology" } },
        { "range": { "price": { "gte": 10, "lte": 100 } } },
        { "range": { "created_at": { "gte": "2024-01-01" } } }
      ],
      "should": [
        { "match_phrase": { "title": "system design" } }
      ],
      "must_not": [
        { "term": { "status": "draft" } }
      ]
    }
  },
  "sort": [
    { "_score": "desc" },
    { "created_at": "desc" }
  ],
  "from": 0,
  "size": 20,
  "highlight": {
    "fields": { "description": {} }
  }
}

// QUERY TYPES:
//   match:          full-text search with analysis (OR by default)
//   match_phrase:   exact phrase in order
//   term:          exact value match (no analysis) — for keywords
//   range:         numeric/date ranges
//   bool:          combine queries (must/should/must_not/filter)
//   fuzzy:         typo-tolerant ("distributd" → "distributed")
//   wildcard:      pattern matching ("dis*" → "distributed")
//   multi_match:   search across multiple fields
```

---

## 📊 Aggregations

```json
// SQL: SELECT category, COUNT(*), AVG(price), MAX(price) 
//      FROM products WHERE status = 'active' GROUP BY category

{
  "query": { "term": { "status": "active" } },
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 20 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } },
        "max_price": { "max": { "field": "price" } },
        "price_histogram": {
          "histogram": { "field": "price", "interval": 50 }
        }
      }
    },
    "overall_stats": {
      "stats": { "field": "price" }
    }
  },
  "size": 0  // don't return documents, only aggregations
}

// AGGREGATION TYPES:
//   Bucket: group documents (terms, histogram, date_histogram, range)
//   Metric: compute stats (avg, sum, min, max, cardinality, percentiles)
//   Pipeline: operate on other aggregation results (derivative, moving_avg)
```

---

## 💻 Spring Boot Integration

```java
// application.yml
spring:
  elasticsearch:
    uris: http://elasticsearch:9200
    username: elastic
    password: ${ES_PASSWORD}

// Entity
@Document(indexName = "products")
public class Product {
    @Id
    private String id;
    
    @Field(type = FieldType.Text, analyzer = "english")
    private String name;
    
    @Field(type = FieldType.Text, analyzer = "english")
    private String description;
    
    @Field(type = FieldType.Keyword)
    private String category;
    
    @Field(type = FieldType.Double)
    private Double price;
    
    @Field(type = FieldType.Keyword)
    private List<String> tags;
}

// Repository (simple queries)
public interface ProductSearchRepository extends ElasticsearchRepository<Product, String> {
    List<Product> findByCategory(String category);
    List<Product> findByNameContaining(String text);
    List<Product> findByPriceBetween(Double min, Double max);
}

// ElasticsearchOperations (complex queries)
@Service
public class ProductSearchService {
    private final ElasticsearchOperations esOps;
    
    public SearchPage<Product> search(String query, String category, 
                                       Double minPrice, Double maxPrice, 
                                       Pageable pageable) {
        var boolQuery = QueryBuilders.bool();
        
        if (query != null) {
            boolQuery.must(QueryBuilders.multiMatch()
                .fields("name^3", "description")  // name 3x boost
                .query(query)
                .fuzziness("AUTO"));
        }
        if (category != null) {
            boolQuery.filter(QueryBuilders.term().field("category").value(category));
        }
        if (minPrice != null || maxPrice != null) {
            var range = QueryBuilders.range().field("price");
            if (minPrice != null) range.gte(JsonData.of(minPrice));
            if (maxPrice != null) range.lte(JsonData.of(maxPrice));
            boolQuery.filter(range);
        }
        
        NativeQuery searchQuery = NativeQuery.builder()
            .withQuery(boolQuery.build()._toQuery())
            .withPageable(pageable)
            .withHighlightQuery(new HighlightQuery(
                new Highlight(List.of(new HighlightField("description")))))
            .build();
        
        return esOps.search(searchQuery, Product.class);
    }
}
```

---

## ⚙️ Cluster Management & Scaling

```
SHARD SIZING BEST PRACTICES:
  - Target: 20-50GB per shard
  - Rule of thumb: number_of_shards = index_size / 50GB
  - Too many shards → overhead (each shard = Lucene instance)
  - Too few shards → can't distribute load

INDEX LIFECYCLE MANAGEMENT (ILM):
  Hot phase:   recent data, fast SSDs, more replicas (actively writing + searching)
  Warm phase:  older data, cheaper storage, fewer replicas (searching only)
  Cold phase:  archived data, minimal resources, searchable snapshots
  Delete phase: auto-delete after retention period

  Example (logs):
    hot:    0-7 days    (SSD, 1 primary + 2 replicas)
    warm:   7-30 days   (HDD, 1 primary + 1 replica, force-merge to 1 segment)
    cold:   30-90 days  (searchable snapshots in S3)
    delete: 90+ days    (auto-purge)

SCALING STRATEGIES:
  Read scaling:   add replicas (more nodes serve read queries)
  Write scaling:  add primary shards (must create new index — can't reshard in-place)
  Data tiering:   ILM moves data to cheaper storage over time
  Cross-cluster:  search across multiple clusters (geo-distributed)
```

---

## ⚠️ Common Pitfalls

1. **Mapping explosion** — Dynamic mappings create a field for every new JSON key. If you index user-generated JSON with thousands of unique keys, your mapping grows unboundedly (memory pressure, slow cluster state). Set `dynamic: strict` or `dynamic: false` for unknown fields.

2. **Too many shards (over-sharding)** — Each shard is a Lucene instance with memory overhead. An index with 100 shards for only 1GB of data wastes resources. Start with 1 shard for small indices; scale based on actual data size (target 20-50GB/shard).

3. **Using Elasticsearch as primary database** — ES is not designed for strong consistency or ACID transactions. Data loss can occur during network partitions. Always keep your source of truth in a proper database (PostgreSQL, MongoDB) and sync to ES for search.

4. **Deep pagination (from + size > 10,000)** — Default limit is 10K results. Deep pagination is expensive (all shards must return and sort from+size documents). Use `search_after` for deep pagination or `scroll` API for bulk export.

5. **Not using filters for exact-match queries** — `filter` context skips scoring (faster) and caches results. Use `filter` for exact matches (status, category, date ranges) and reserve `must` for full-text relevance queries.

---

## 🧩 Mini Challenge

**Design an autocomplete/search-as-you-type feature for a product search box. Requirements: suggestions appear after 2 characters typed, show top 5 product names matching the prefix, sub-50ms latency.**

<details>
<summary>💡 Click to reveal answer</summary>

**Approach: Custom analyzer with edge n-grams + completion suggester**

```json
// Index mapping with edge n-gram analyzer
PUT /products_autocomplete
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete_analyzer": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "autocomplete_filter"]
        },
        "autocomplete_search": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase"]
        }
      },
      "filter": {
        "autocomplete_filter": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 15
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "autocomplete_analyzer",
        "search_analyzer": "autocomplete_search"
      },
      "suggest": {
        "type": "completion"
      }
    }
  }
}

// Indexing: "iPhone 15 Pro" creates terms: ["ip", "iph", "ipho", "iphon", "iphone", ...]
// Search: user types "iph" → matches edge n-gram "iph" → finds "iPhone 15 Pro"

// Query (sub-50ms — single shard lookup):
POST /products_autocomplete/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "iph",
      "completion": {
        "field": "suggest",
        "size": 5,
        "fuzzy": { "fuzziness": 1 }
      }
    }
  }
}
```

**Why this is fast**: Completion suggesters use in-memory FST (Finite State Transducer) data structure — prefix lookups are O(prefix_length), not O(total_documents).

</details>

---

## 📝 Interview Q&A

**Q: How does Elasticsearch achieve near real-time search?**
> A: When a document is indexed, it's written to an in-memory buffer. Every 1 second (default `refresh_interval`), this buffer is flushed to a new Lucene segment (a searchable, immutable file). This "refresh" makes the document searchable. So there's a ~1 second delay between indexing and searchability. For true real-time, you can call `_refresh` API after indexing (but this is expensive at scale). The 1-second delay is called "near real-time" (NRT) and is acceptable for most use cases.

**Q: When would you use Elasticsearch vs a relational database with full-text search (PostgreSQL tsvector)?**
> A: Use Elasticsearch when: (1) search is a primary feature with complex requirements (fuzzy matching, synonyms, relevance tuning, faceted search); (2) data volume exceeds what a single PostgreSQL instance handles efficiently for search; (3) you need real-time analytics/aggregations on large datasets; (4) horizontal scaling for search throughput. Use PostgreSQL tsvector when: (1) simple keyword search on moderate data; (2) you don't want operational complexity of another system; (3) strong consistency between data and search results is critical; (4) search is a secondary feature (not the core product).

---

## 🔗 What to Read Next

1. **[BuildingBlocks/SearchIndex.md](../BuildingBlocks/SearchIndex.md)** — Search fundamentals and patterns
2. **[Observability/ELK_Stack.md](../Observability/ELK_Stack.md)** — Elasticsearch in the logging ecosystem
3. **[Database/Database_Selection_Guide.md](./Database_Selection_Guide.md)** — When to choose ES vs other options

---

*[← Cassandra Deep Dive](./Cassandra_Deep_Dive.md) | [Back to Database](../INDEX.md) | [Next: TimeSeries Databases →](./TimeSeries_Databases.md)*
