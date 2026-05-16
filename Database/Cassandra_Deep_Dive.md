# 🏛️ Cassandra Deep Dive: The Wide-Column Store Built for Planet-Scale Writes

> *"When Apple needs to store 10 petabytes of iCloud data across 75,000 nodes with zero downtime, they use Cassandra. It's designed for the scenario where availability and write throughput matter more than anything — even at the cost of complexity."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [SQL vs NoSQL](./SQL_Vs_NoSQL.md), [CAP Theorem](../KeyConcepts/CAPTheorem.md), [Consistent Hashing](../KeyConcepts/Consistent_Hashing.md)

---

## 📋 Table of Contents
1. [What is Cassandra](#-what-is-cassandra)
2. [Architecture](#-architecture-masterless-ring)
3. [Data Model](#-data-model)
4. [Write Path & Read Path](#-write-path--read-path)
5. [Consistency Levels](#-consistency-levels)
6. [Partitioning & Compaction](#-partitioning--compaction)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [When to Use / Not Use](#-when-to-use--not-use)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Cassandra

```
┌─────────────────────────────────────────────────────────┐
│                  CASSANDRA AT A GLANCE                   │
│                                                         │
│  Type:         Wide-column store (partitioned row store)│
│  Architecture: Masterless ring (no single point of failure)│
│  CAP:          AP by default (tunable to CP per query)  │
│  Writes:       Always writable (append-only, no locks)  │
│  Scaling:      Linear horizontal (add nodes = add capacity)│
│  Replication:  Multi-DC replication built-in            │
│  Query:        CQL (Cassandra Query Language — SQL-like) │
│  Users:        Apple (75K nodes), Netflix, Instagram,    │
│                Discord, Uber                             │
└─────────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture (Masterless Ring)

```
CASSANDRA RING (no master — every node is equal):

        Node A
       /      \
    Node F    Node B      ← All nodes are peers
      |          |         ← Any node can serve any request
    Node E    Node C      ← Data partitioned by token range
       \      /
        Node D

  Key differences from master-slave (MongoDB/MySQL):
    ❌ No single point of failure (no master to elect)
    ❌ No failover needed (writes never stop)
    ✅ Any node can accept writes for any partition
    ✅ Scales linearly: 2× nodes = 2× throughput

DATA DISTRIBUTION (consistent hashing):
  Token range: 0 to 2^63
  Each node owns a range:
    Node A: tokens 0 - 1000
    Node B: tokens 1001 - 2000
    Node C: tokens 2001 - 3000
    ...

  When you write: partition_key → hash(key) → token → node
  Example: hash("user_123") = 1500 → Node B is responsible

REPLICATION:
  Replication Factor (RF) = 3
  Data written to RF nodes clockwise on the ring:
    Primary: Node B (token owner)
    Replica: Node C (next clockwise)
    Replica: Node D (next next clockwise)
  
  NetworkTopologyStrategy (multi-DC):
    RF per DC: { "us-east": 3, "eu-west": 3 }
    Writes replicated to 3 nodes in each data center
```

---

## 📊 Data Model

```
CASSANDRA DATA MODEL (query-driven, NOT entity-driven):

  ❌ WRONG approach: design tables like relational DB, then write queries
  ✅ RIGHT approach: define your queries FIRST, then design tables to serve them

STRUCTURE:
  Keyspace (≈ database)
    └── Table (≈ collection)
         └── Partition (group of rows with same partition key)
              └── Row (sorted by clustering columns within partition)

EXAMPLE: Chat messages

  Query 1: "Get all messages in a conversation, sorted by time"
  Query 2: "Get messages from a specific user in a conversation"

  CREATE TABLE messages (
    conversation_id UUID,          -- PARTITION KEY (groups data)
    message_time TIMESTAMP,        -- CLUSTERING KEY (sorts within partition)
    sender_id UUID,
    body TEXT,
    PRIMARY KEY (conversation_id, message_time)
  ) WITH CLUSTERING ORDER BY (message_time DESC);

  Physical storage:
  ┌─────────────────────────────────────────────────────────────┐
  │ Partition: conversation_id = "conv-abc"                     │
  ├─────────────────────────────────────────────────────────────┤
  │ 2024-01-15 10:30 | user_1 | "Hello!"                       │
  │ 2024-01-15 10:31 | user_2 | "Hi there!"                    │
  │ 2024-01-15 10:32 | user_1 | "How are you?"                 │
  └─────────────────────────────────────────────────────────────┘

  SELECT * FROM messages WHERE conversation_id = 'conv-abc' 
    ORDER BY message_time DESC LIMIT 50;
  → Single partition read — extremely fast!

IMPORTANT CONSTRAINTS:
  - Partition key in WHERE clause is MANDATORY (no full table scans!)
  - Filtering on non-primary-key columns requires ALLOW FILTERING (slow!)
  - No JOINs — denormalize data into multiple tables for different queries
  - Design for writes — duplicating data across tables is normal
```

---

## ⚡ Write Path & Read Path

```
WRITE PATH (why writes are so fast):

  [Client] → [Coordinator Node] → [Replica Nodes]
  
  At each replica:
    1. Write to COMMIT LOG (append-only, sequential disk write)
    2. Write to MEMTABLE (in-memory sorted structure)
    3. Acknowledge to coordinator
    Done! (no disk seek, no read-before-write, no locking)
    
    Later (background):
    4. Memtable full → flush to SSTABLE on disk (immutable file)
    5. Compaction: merge multiple SSTables periodically

  Write latency: ~1-2ms (just append to log + memory write)
  Writes NEVER fail (as long as node is alive)


READ PATH (more complex):

  [Client] → [Coordinator] → [Replica Nodes]
  
  At each replica:
    1. Check MEMTABLE (in-memory, fastest)
    2. Check BLOOM FILTER for each SSTABLE (probabilistic: "might be here")
    3. Check PARTITION INDEX → COMPRESSION OFFSET MAP
    4. Read data from SSTABLE on disk
    5. Merge results from multiple SSTables (most recent wins)
    6. Return to coordinator

  Read latency: 2-10ms (depending on how many SSTables to check)
  
  Coordinator reconciles replicas:
    - Read from RF nodes
    - Compare timestamps → latest version wins (Last-Write-Wins)
    - Read repair: if replicas disagree, send correct version to stale replicas


SSTABLE COMPACTION (background maintenance):
  Over time, many SSTables accumulate (one per memtable flush)
  Compaction merges SSTables → reduces read amplification
  
  Strategies:
    SizeTieredCompaction: good for write-heavy (default)
    LeveledCompaction: good for read-heavy (bounded SSTables per level)
    TimeWindowCompaction: good for time-series (compact by time window)
```

---

## 🎚️ Consistency Levels

```
TUNABLE CONSISTENCY (per-query):

  Write Consistency:
    ANY:       write succeeds if ANY node (even hinted handoff) acknowledges
    ONE:       write succeeds if 1 replica acknowledges
    QUORUM:    write succeeds if majority (RF/2 + 1) replicas acknowledge
    ALL:       write succeeds only if ALL replicas acknowledge

  Read Consistency:
    ONE:       read from 1 replica (fastest, may be stale)
    QUORUM:    read from majority, return most recent
    ALL:       read from all replicas (slowest, always consistent)

STRONG CONSISTENCY FORMULA:
  R + W > RF  →  strong consistency guaranteed
  
  Example (RF=3):
    Write QUORUM (W=2) + Read QUORUM (R=2) → 2+2 > 3 ✅ strongly consistent
    Write ONE (W=1) + Read ONE (R=1) → 1+1 < 3 ❌ eventually consistent
    Write ALL (W=3) + Read ONE (R=1) → 3+1 > 3 ✅ strongly consistent

COMMON CONFIGURATIONS:
  High availability (AP): W=ONE, R=ONE (fastest, eventual consistency)
  Balanced:              W=QUORUM, R=QUORUM (strong consistency, good availability)
  Maximum durability:    W=ALL, R=ONE (never lose writes, slower writes)
```

---

## 💻 Spring Boot Integration

```java
// application.yml
spring:
  cassandra:
    keyspace-name: myapp
    contact-points: cassandra-node1,cassandra-node2,cassandra-node3
    local-datacenter: us-east-1
    schema-action: create_if_not_exists

// Entity
@Table("messages")
public class Message {
    @PrimaryKeyColumn(name = "conversation_id", ordinal = 0, type = PrimaryKeyType.PARTITIONED)
    private UUID conversationId;
    
    @PrimaryKeyColumn(name = "message_time", ordinal = 1, type = PrimaryKeyType.CLUSTERED,
                      ordering = Ordering.DESCENDING)
    private Instant messageTime;
    
    private UUID senderId;
    private String body;
}

// Repository
public interface MessageRepository extends CassandraRepository<Message, UUID> {
    
    // Partition key query — fast single-partition read
    List<Message> findByConversationId(UUID conversationId);
    
    // Partition + clustering range — efficient time-range query
    @Query("SELECT * FROM messages WHERE conversation_id = ?0 AND message_time > ?1 LIMIT ?2")
    List<Message> findRecentMessages(UUID conversationId, Instant since, int limit);
}

// Custom consistency per query
@Service
public class MessageService {
    private final CqlSession session;
    
    public void sendMessage(Message msg) {
        PreparedStatement stmt = session.prepare(
            "INSERT INTO messages (conversation_id, message_time, sender_id, body) VALUES (?, ?, ?, ?)");
        
        BoundStatement bound = stmt.bind(
            msg.getConversationId(), msg.getMessageTime(), msg.getSenderId(), msg.getBody())
            .setConsistencyLevel(ConsistencyLevel.LOCAL_QUORUM);  // strong consistency for writes
        
        session.execute(bound);
    }
}
```

---

## ⚖️ When to Use / Not Use

| Use Cassandra When | Avoid Cassandra When |
|---|---|
| Write-heavy workloads (100K+ writes/sec) | Read-heavy with complex queries (aggregations, JOINs) |
| Time-series data (IoT, logs, metrics) | Small dataset (< 10GB — overkill) |
| Always-on availability is critical | Need ad-hoc queries on arbitrary columns |
| Multi-datacenter replication needed | Strong consistency is always required |
| Linear scalability needed (predictable) | Schema changes frequently (rigid data model) |
| Known access patterns (query-driven design) | Need transactions across multiple partitions |

---

## ⚠️ Common Pitfalls

1. **Designing tables like relational DB** — Cassandra requires query-driven modeling. You can't `SELECT * FROM users WHERE email = ?` unless `email` is the partition key. Model one table per query pattern. Duplicating data across tables is expected.

2. **Large partitions** — A partition that grows to millions of rows (all messages for a chat group forever) causes garbage collection pauses and slow reads. Limit partition size to < 100MB / < 100K rows. Use time bucketing: `PRIMARY KEY ((conversation_id, month), message_time)`.

3. **Using ALLOW FILTERING** — This scans potentially all partitions (full cluster scan). It's like a table scan in SQL. Never use in production queries — redesign your table or create a materialized view.

4. **Tombstone accumulation** — Deletes in Cassandra create tombstones (markers), not actual deletes. Millions of tombstones degrade read performance dramatically. Configure `gc_grace_seconds`, use TTL instead of explicit deletes, and run repairs regularly.

5. **Uneven partition sizes** — If some partition keys have millions of rows and others have 10, your cluster is unbalanced. One node handles most traffic (hot partition). Choose partition keys with high cardinality and even distribution.

---

## 🧩 Mini Challenge

**Design a Cassandra table for a social media "timeline" feature. Requirements: show a user's timeline (posts from people they follow), sorted by post time, paginated (20 posts per page).**

<details>
<summary>💡 Click to reveal answer</summary>

**Approach: Fan-out on write (pre-build timelines)**

```sql
-- Timeline table (one partition per user, posts sorted by time)
CREATE TABLE user_timeline (
    user_id UUID,
    post_time TIMESTAMP,
    post_id UUID,
    author_id UUID,
    author_name TEXT,
    content TEXT,
    media_urls LIST<TEXT>,
    PRIMARY KEY (user_id, post_time)
) WITH CLUSTERING ORDER BY (post_time DESC);

-- When user_B posts something:
-- For each follower of user_B, write to their timeline:
INSERT INTO user_timeline (user_id, post_time, post_id, author_id, author_name, content)
VALUES (follower_1, now(), post_id, user_B, 'Bob', 'Hello world!');
-- (repeated for each follower — fan-out on write)

-- Read timeline (single partition, fast!):
SELECT * FROM user_timeline WHERE user_id = ? LIMIT 20;

-- Pagination (using clustering key):
SELECT * FROM user_timeline WHERE user_id = ? AND post_time < ? LIMIT 20;
```

**Time bucketing** (prevent unbounded partition growth):
```sql
CREATE TABLE user_timeline (
    user_id UUID,
    time_bucket TEXT,  -- '2024-01' (monthly bucket)
    post_time TIMESTAMP,
    post_id UUID,
    author_id UUID,
    content TEXT,
    PRIMARY KEY ((user_id, time_bucket), post_time)
) WITH CLUSTERING ORDER BY (post_time DESC);

-- Query current month first, then previous months if needed
SELECT * FROM user_timeline WHERE user_id = ? AND time_bucket = '2024-01' LIMIT 20;
```

**Trade-off**: Write amplification (1 post → N writes for N followers) but read is O(1). This is exactly what Twitter/Instagram use for non-celebrity accounts.

</details>

---

## 📝 Interview Q&A

**Q: How does Cassandra achieve high availability with no single point of failure?**
> A: Cassandra uses a masterless (peer-to-peer) architecture where every node is equal. There's no leader election, no failover delay. Data is replicated to RF nodes using consistent hashing. Any node can serve any request — if the target replica is down, the coordinator routes to another replica. Writes always succeed (append-only, no locking) as long as enough nodes are available to satisfy the write consistency level. Even during network partitions, each partition (in the network sense) can continue serving reads and writes at reduced consistency levels.

**Q: Why is Cassandra write-optimized? What makes reads slower?**
> A: Writes are fast because they're append-only: write to an in-memory memtable + sequential append to commit log. No disk seeks, no read-before-write, no locking. Reads are slower because data for a partition may be spread across multiple SSTables (immutable files from past memtable flushes). A read must check the memtable + potentially many SSTables, merge results, and resolve conflicts by timestamp. Bloom filters and partition indexes optimize this, but reads are inherently more expensive than writes. Compaction (background merge of SSTables) reduces read amplification over time.

---

## 🔗 What to Read Next

1. **[KeyConcepts/Consistent_Hashing.md](../KeyConcepts/Consistent_Hashing.md)** — The algorithm behind Cassandra's data distribution
2. **[KeyConcepts/CAPTheorem.md](../KeyConcepts/CAPTheorem.md)** — Why Cassandra is AP and how tunable consistency works
3. **[Database/Database_Selection_Guide.md](./Database_Selection_Guide.md)** — When Cassandra vs other databases

---

*[← MongoDB Deep Dive](./MongoDB_Deep_Dive.md) | [Back to Database](../INDEX.md) | [Next: Database Selection Guide →](./Database_Selection_Guide.md)*
