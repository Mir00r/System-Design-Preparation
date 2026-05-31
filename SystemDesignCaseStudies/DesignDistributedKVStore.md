# 🗄️ Design a Distributed Key-Value Store

> *"Amazon's DynamoDB handles 10 TRILLION requests per day. Redis serves millions of ops/second with sub-millisecond latency. Behind every high-performance system sits a key-value store — the simplest yet most powerful data abstraction. But making it DISTRIBUTED (spanning multiple machines, surviving failures, staying consistent) transforms a HashMap into one of the hardest problems in computer science."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Sharding](../../Database/Sharding.md), [CAP Theorem](../../KeyConcepts/CAPTheorem.md), [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Single-Node Design](#-single-node-design)
3. [Going Distributed: Core Challenges](#-going-distributed-core-challenges)
4. [Consistent Hashing & Partitioning](#-consistent-hashing--partitioning)
5. [Replication](#-replication)
6. [Consistency Models](#-consistency-models)
7. [Failure Detection & Recovery](#-failure-detection--recovery)
8. [Read/Write Path](#-readwrite-path)
9. [Storage Engine (LSM-Tree)](#-storage-engine-lsm-tree)
10. [Java Implementation](#-java-implementation)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ put(key, value) — Store a key-value pair
✅ get(key) — Retrieve value by key
✅ delete(key) — Remove a key-value pair
✅ Support large dataset (can't fit on single machine!)
✅ Automatic data partitioning across nodes
✅ Configurable consistency level
✅ TTL (time-to-live) for automatic expiration
```

### Non-Functional Requirements
```
✅ High availability (always accept writes, even during partitions)
✅ Scalable (add nodes without downtime)
✅ Tunable consistency (strong → eventual, per request!)
✅ Low latency (< 10ms for reads, < 50ms for writes)
✅ Fault tolerance (survive node failures without data loss)
✅ Partition tolerance (network splits don't cause system failure)
```

### CAP Theorem Decision
```
We can only have 2 of 3: Consistency, Availability, Partition Tolerance.

Network partitions WILL happen → must choose P.
So we choose: AP (Available + Partition Tolerant)
→ Like DynamoDB, Cassandra, Riak

WITH tunable consistency:
  - Default: Eventually consistent (fast!)
  - On request: Strong consistency (slower, but correct)
```

---

## 📦 Single-Node Design

```
SIMPLE KEY-VALUE STORE (single machine):

  ┌────────────────────────────────────────────────┐
  │                   API LAYER                     │
  │   put("user:123", "{name: John}")              │
  │   get("user:123") → "{name: John}"             │
  └────────────────────────┬───────────────────────┘
                           │
  ┌────────────────────────▼───────────────────────┐
  │              MEMORY (HashMap/SkipList)          │
  │   Fast reads/writes, but limited by RAM!       │
  │   user:123 → {name: John}                      │
  │   order:456 → {total: 99.99}                   │
  └────────────────────────┬───────────────────────┘
                           │ (flush when memory full)
  ┌────────────────────────▼───────────────────────┐
  │              DISK (SSTable / Log)               │
  │   Persistent storage, survives restarts         │
  │   Sorted String Table for efficient lookups    │
  └────────────────────────────────────────────────┘
  
PROBLEMS WITH SINGLE NODE:
  • RAM limit: 64 GB RAM = max ~64 GB data (need 1 TB!)
  • Single point of failure (server dies = data gone!)
  • Throughput ceiling (one machine = limited QPS)
  • No geographic distribution (high latency for far users)
  
  → WE NEED DISTRIBUTION!
```

---

## 🌐 Going Distributed: Core Challenges

```
DISTRIBUTING A KV STORE INTRODUCES 5 HARD PROBLEMS:

  ┌──────────────────────────────────────────────────────────────┐
  │  Problem           │  Question                               │
  ├──────────────────────────────────────────────────────────────┤
  │  1. Partitioning   │  Which node stores which key?            │
  │  2. Replication    │  How many copies? Where?                 │
  │  3. Consistency    │  After write, when can all read new val? │
  │  4. Failure        │  Node dies — what happens to its data?   │
  │  5. Conflicts      │  Two writes to same key on diff nodes?   │
  └──────────────────────────────────────────────────────────────┘

  Simple hash: node = hash(key) % N  ← TERRIBLE!
  Why? If N changes (add/remove node), EVERY key moves! 😱
  
  Solution: CONSISTENT HASHING!
```

---

## 🔄 Consistent Hashing & Partitioning

```
CONSISTENT HASHING:
  Imagine a circular ring (0 to 2^32 - 1)
  
  Place NODES on the ring (by hashing node IP/ID):
  
          Node A (hash=1000)
              ●
           /     \
     ●          ●
  Node D          Node B
  (hash=8000)    (hash=3000)
           \     /
              ●
          Node C (hash=5000)
  
  For a key: hash(key) → walk CLOCKWISE to find first node!
  
  hash("user:123") = 2500 → clockwise → Node B (3000)!
  hash("order:456") = 7500 → clockwise → Node D (8000)!
  
  ADD a new node E at position 6000:
  Only keys between 5000-6000 move from D to E!
  Everything else stays! (Only K/N keys move on average!)

VIRTUAL NODES (for balance):
  Problem: 4 physical nodes might not divide evenly
  Solution: Each physical node gets 100-200 virtual positions!
  
  Node A → positions [1000, 1500, 2200, 4800, ...] (150 vnodes)
  Node B → positions [3000, 3500, 6700, 9100, ...] (150 vnodes)
  
  Now keys are EVENLY distributed regardless of physical node count!
  
  Also helps when nodes have different capacity:
  Powerful server → more virtual nodes → more data
  Weaker server → fewer virtual nodes → less data
```

---

## 📋 Replication

```
REPLICATION STRATEGY:
  For each key, store on N nodes (typically N=3)
  
  "Preference list": the N nodes clockwise from the key's position
  
  hash("user:123") = 2500
  Replicas: Node B (3000), Node C (5000), Node D (8000)
  → 3 copies on 3 different physical nodes!
  
  WHY 3 REPLICAS?
  • Survive 2 simultaneous node failures!
  • If 1 dies: 2 copies remain → no data loss
  • Read from closest/fastest replica → low latency
  
REPLICATION MODES:
  ┌────────────────────────────────────────────────────────────┐
  │  Mode                │  How It Works                       │
  ├────────────────────────────────────────────────────────────┤
  │  Synchronous         │  Write returns AFTER all N ACK      │
  │  (strong consistency)│  Slow but consistent                │
  ├────────────────────────────────────────────────────────────┤
  │  Asynchronous        │  Write returns AFTER 1 ACK          │
  │  (eventual)          │  Fast but may lose recent writes    │
  ├────────────────────────────────────────────────────────────┤
  │  Quorum              │  Write returns after W ACK          │
  │  (tunable!)          │  Read returns after R replies       │
  │                      │  W + R > N guarantees consistency!  │
  └────────────────────────────────────────────────────────────┘
```

---

## 🎯 Consistency Models

```
QUORUM CONSENSUS (W + R > N):
  N = 3 replicas
  W = 2 (write succeeds after 2 ACKs)
  R = 2 (read queries 2 nodes, takes latest)
  
  W + R = 4 > N(3) → At least 1 node has both read & write!
  → GUARANTEED to read latest write!
  
  CONFIGURATIONS:
  ┌─────────────────────────────────────────────────────────────┐
  │  W  │  R  │  W+R  │  Guarantee            │  Best For       │
  ├─────────────────────────────────────────────────────────────┤
  │  1  │  3  │  4>3  │  Fast write, slow read │  Write-heavy    │
  │  3  │  1  │  4>3  │  Slow write, fast read │  Read-heavy     │
  │  2  │  2  │  4>3  │  Balanced              │  General use    │
  │  1  │  1  │  2<3  │  ⚠️ NOT consistent!    │  Best-effort    │
  └─────────────────────────────────────────────────────────────┘

CONFLICT RESOLUTION (when replicas disagree):
  Two clients write to same key simultaneously on different nodes!
  
  Client A → Node 1: put("x", "A") at timestamp T1
  Client B → Node 2: put("x", "B") at timestamp T2
  
  Now Node 1 has "A" and Node 2 has "B"! CONFLICT!
  
  Resolution strategies:
  1. LAST-WRITE-WINS (LWW): higher timestamp wins
     If T2 > T1 → "B" wins. Simple but can lose data!
     
  2. VECTOR CLOCKS: detect true conflicts
     Each node tracks [Node1:2, Node2:3] version vector
     If vectors are incomparable → true conflict → application resolves!
     
  3. CRDT (Conflict-free Replicated Data Types):
     Data structures that auto-merge (counters, sets, maps)
     No conflicts by construction! (Used by Riak, Redis CRDT)
```

---

## 🔍 Failure Detection & Recovery

```
FAILURE DETECTION (Gossip Protocol):
  Every node periodically pings random peers:
  
  Node A → "Hey B, you alive?" → B responds → A marks B as UP
  Node A → "Hey C, you alive?" → timeout → A suspects C is DOWN
  Node A tells others: "I think C is dead"
  If majority agree C is unresponsive → C declared DOWN
  
  (See Gossip Protocol tutorial for deep dive!)

HINTED HANDOFF (temporary failure):
  Node C is temporarily down. A write for C arrives:
  
  1. Node D (next in ring) accepts the write as a "hint"
  2. D stores: { intended_for: C, key: "x", value: "v" }
  3. When C comes back, D forwards all hints to C!
  4. C is now caught up → hints deleted
  
  This is why AP systems can "always accept writes"!
  Even if the right node is down, SOMEONE accepts it!

ANTI-ENTROPY (permanent failure recovery):
  Node C's disk died → data lost permanently!
  
  Recovery:
  1. New node C' joins, assigned C's position on ring
  2. Copy data from C's replicas (Nodes B and D have copies!)
  3. Use MERKLE TREES to efficiently find missing data:
  
     Merkle Tree: hash tree over key ranges
     Compare root hashes: if different → drill into subtrees
     Only transfer keys that actually differ (not entire dataset!)
     
  Merkle trees reduce data transfer from O(N) to O(log N)!
```

---

## 📝 Read/Write Path

```
WRITE PATH (Coordinator pattern):

  Client: put("user:123", "{name: John}")
     │
     ▼
  ┌─────────────────────────────────────────────────┐
  │  COORDINATOR (any node can be coordinator!)     │
  │  1. hash("user:123") → determine preference list│
  │  2. Forward write to N=3 replica nodes          │
  │  3. Wait for W=2 ACKs                           │
  │  4. Return success to client                     │
  └─────────────────────────────────────────────────┘
     │         │         │
     ▼         ▼         ▼
  Node B    Node C    Node D  (N=3 replicas)
  (ACK ✅)  (ACK ✅)  (slow but will eventually get it)
  
  W=2 achieved! Return success! (Don't wait for Node D!)

READ PATH:
  Client: get("user:123")
     │
     ▼
  Coordinator:
  1. hash("user:123") → same preference list
  2. Send read request to R=2 nodes
  3. Receive responses:
     Node B: { value: "{name: John}", version: 5 }
     Node C: { value: "{name: John}", version: 5 }
  4. Both agree → return value!
  
  What if they disagree?
     Node B: { value: "{name: John}", version: 5 }
     Node C: { value: "{name: Jane}", version: 6 }  ← newer!
  5. Return version 6 (latest!)
  6. REPAIR: send version 6 to Node B (read repair!)
```

---

## 💾 Storage Engine (LSM-Tree)

```
LSM-TREE (Log-Structured Merge Tree):
  Used by: Cassandra, LevelDB, RocksDB, HBase
  
  WHY? Optimized for WRITE-HEAVY workloads!
  Writes go to memory first (fast!), flush to disk in batches.
  
  ARCHITECTURE:
  
  ┌─────────────────────────────────────┐
  │  WRITE-AHEAD LOG (WAL)             │ ← Durability! 
  │  Sequential append (very fast!)     │    Crash recovery!
  └─────────────────────────────────────┘
               │
  ┌────────────▼────────────────────────┐
  │  MEMTABLE (in-memory sorted tree)   │ ← Fast writes!
  │  Red-Black Tree or Skip List        │    O(log N) insert
  │  Size: ~64 MB                       │
  └─────────────────────────────────────┘
               │ (when full → flush to disk!)
  ┌────────────▼────────────────────────┐
  │  LEVEL 0 SSTables (Sorted String)   │ ← May overlap!
  └─────────────────────────────────────┘
               │ (compaction merges overlapping files)
  ┌────────────▼────────────────────────┐
  │  LEVEL 1 SSTables                   │ ← No overlap!
  └─────────────────────────────────────┘
               │
  ┌────────────▼────────────────────────┐
  │  LEVEL 2 SSTables (10x bigger)      │ ← Bigger files!
  └─────────────────────────────────────┘

  WRITE: WAL append → Memtable insert → Done! (microseconds!)
  READ: Memtable → L0 → L1 → L2 → ... (use Bloom filters to skip!)
  
  BLOOM FILTER OPTIMIZATION:
  Before checking an SSTable file: query its Bloom filter
  "Is key X possibly in this file?" 
    NO → skip this file entirely! (saves disk I/O!)
    MAYBE → check the file (rare false positive, ~1%)
```

---

## 💻 Java Implementation

### Core KV Store with Consistent Hashing

```java
public class DistributedKVStore {
    
    private final ConsistentHashRing ring;
    private final int replicationFactor;  // N
    private final int writeQuorum;        // W
    private final int readQuorum;         // R
    
    public DistributedKVStore(List<Node> nodes, int N, int W, int R) {
        this.ring = new ConsistentHashRing(nodes, 150); // 150 vnodes
        this.replicationFactor = N;
        this.writeQuorum = W;
        this.readQuorum = R;
    }
    
    public CompletableFuture<Void> put(String key, String value) {
        List<Node> replicas = ring.getPreferenceList(key, replicationFactor);
        
        VersionedValue vv = new VersionedValue(
            value, System.nanoTime(), UUID.randomUUID());
        
        // Send write to all N replicas, wait for W ACKs
        List<CompletableFuture<Boolean>> futures = replicas.stream()
            .map(node -> node.writeAsync(key, vv))
            .collect(Collectors.toList());
        
        return waitForQuorum(futures, writeQuorum)
            .thenAccept(acks -> {
                if (acks < writeQuorum) {
                    throw new InsufficientReplicasException(
                        "Only " + acks + "/" + writeQuorum + " ACKs");
                }
            });
    }
    
    public CompletableFuture<String> get(String key) {
        List<Node> replicas = ring.getPreferenceList(key, replicationFactor);
        
        // Read from R replicas, return latest version
        List<CompletableFuture<VersionedValue>> futures = replicas.stream()
            .limit(readQuorum)
            .map(node -> node.readAsync(key))
            .collect(Collectors.toList());
        
        return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .thenApply(v -> {
                VersionedValue latest = futures.stream()
                    .map(CompletableFuture::join)
                    .filter(Objects::nonNull)
                    .max(Comparator.comparing(VersionedValue::timestamp))
                    .orElse(null);
                
                // Trigger read repair for stale replicas
                if (latest != null) {
                    triggerReadRepair(key, latest, replicas);
                }
                
                return latest != null ? latest.value() : null;
            });
    }
}
```

### Consistent Hash Ring

```java
public class ConsistentHashRing {
    
    private final TreeMap<Long, Node> ring = new TreeMap<>();
    private final int virtualNodes;
    
    public ConsistentHashRing(List<Node> nodes, int virtualNodes) {
        this.virtualNodes = virtualNodes;
        for (Node node : nodes) {
            addNode(node);
        }
    }
    
    public void addNode(Node node) {
        for (int i = 0; i < virtualNodes; i++) {
            long hash = hash(node.getId() + "#" + i);
            ring.put(hash, node);
        }
    }
    
    public void removeNode(Node node) {
        for (int i = 0; i < virtualNodes; i++) {
            long hash = hash(node.getId() + "#" + i);
            ring.remove(hash);
        }
    }
    
    /**
     * Get N distinct physical nodes clockwise from key's position.
     */
    public List<Node> getPreferenceList(String key, int n) {
        long hash = hash(key);
        Set<String> seen = new HashSet<>();
        List<Node> result = new ArrayList<>();
        
        // Walk clockwise from key's position
        Map.Entry<Long, Node> entry = ring.ceilingEntry(hash);
        if (entry == null) entry = ring.firstEntry(); // Wrap around!
        
        Iterator<Map.Entry<Long, Node>> iter = 
            ring.tailMap(hash).entrySet().iterator();
        
        while (result.size() < n) {
            if (!iter.hasNext()) {
                iter = ring.entrySet().iterator(); // Wrap around ring
            }
            Node node = iter.next().getValue();
            if (seen.add(node.getId())) { // Skip same physical node!
                result.add(node);
            }
        }
        return result;
    }
    
    private long hash(String key) {
        // MurmurHash3 for good distribution
        return Hashing.murmur3_128().hashString(key, UTF_8).asLong();
    }
}
```

### LSM-Tree Storage Engine (Simplified)

```java
public class LSMStorageEngine {
    
    private final ConcurrentSkipListMap<String, VersionedValue> memTable;
    private final int memTableSizeLimit;
    private final List<SSTable> sstables;
    private final WriteAheadLog wal;
    
    public void put(String key, VersionedValue value) {
        // 1. Write to WAL (durability!)
        wal.append(key, value);
        
        // 2. Insert into MemTable (in-memory sorted structure)
        memTable.put(key, value);
        
        // 3. If MemTable full → flush to disk as SSTable
        if (memTable.size() >= memTableSizeLimit) {
            flushMemTable();
        }
    }
    
    public VersionedValue get(String key) {
        // 1. Check MemTable first (most recent data!)
        VersionedValue result = memTable.get(key);
        if (result != null) return result;
        
        // 2. Check SSTables from newest to oldest
        for (int i = sstables.size() - 1; i >= 0; i--) {
            SSTable sst = sstables.get(i);
            
            // Bloom filter: might this key be in this file?
            if (!sst.bloomFilter.mightContain(key)) continue; // Skip!
            
            result = sst.get(key);
            if (result != null) return result;
        }
        
        return null; // Key doesn't exist
    }
    
    private void flushMemTable() {
        // Convert sorted MemTable to immutable SSTable on disk
        SSTable newSST = SSTable.createFrom(memTable);
        sstables.add(newSST);
        memTable.clear();
        wal.truncate(); // MemTable is now persisted, WAL not needed
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Multi-Region Replication

Your KV store needs to work across US-East, EU-West, and Asia regions. A user in Tokyo writes a key, and a user in New York reads it 100ms later. Design the replication strategy.

<details>
<summary>🔑 Answer</summary>

**Cross-Region Replication:**
```
Region Strategy:
  Each key has a "home region" (determined by key prefix or user location)
  
  Write: Replicated SYNCHRONOUSLY within home region (3 nodes, W=2)
         Replicated ASYNCHRONOUSLY to other regions (background)
  
  Read: STRONG read = route to home region (higher latency!)
        EVENTUAL read = read from local region (may be stale!)
        
  Tokyo user writes "user:tokyo:123":
    1. Synchronous write to 3 nodes in Asia region (< 5ms)
    2. Async replication to US-East and EU-West (~100-200ms)
    
  New York user reads "user:tokyo:123" 100ms later:
    • If eventual consistency OK: read local US-East replica 
      → might get stale data (async hasn't arrived yet!)
    • If strong read required: route to Asia region
      → adds 150ms latency but guaranteed fresh!
      
  CONFLICT HANDLING across regions:
    Use vector clocks per region: [Asia:5, US:3, EU:4]
    If two regions write simultaneously → detect conflict
    Resolution: application-specific (LWW, merge, ask user)
```
</details>

---

## ❓ Interview Q&A

**Q1: How does consistent hashing handle node additions/removals?**
> When a node joins: only keys in the new node's range (between it and its predecessor on the ring) need to move. On average, K/N keys move (K=total keys, N=nodes). Virtual nodes (100-200 per physical node) ensure even distribution. When a node leaves: its keys automatically fall to the next node clockwise. No global reshuffling!

**Q2: Explain the trade-off between W, R, and N.**
> N=replication factor (copies). W=write quorum. R=read quorum. If W+R > N: guaranteed to read your writes (strong consistency). Lower W = faster writes but risk of data loss. Lower R = faster reads but might read stale data. W=N, R=1 = write-heavy optimization. W=1, R=N = read-heavy optimization. W=R=(N+1)/2 = balanced quorum.

**Q3: How do you handle network partitions in an AP system?**
> During partition: both sides continue accepting writes (availability!). After partition heals: conflict detection via vector clocks. Resolution: Last-Write-Wins (simple, may lose data), or application-level merge (complex, preserves all writes). Hinted handoff ensures writes intended for unreachable nodes are stored temporarily and forwarded when connectivity restores.

**Q4: Why use LSM-Trees instead of B-Trees for a write-heavy KV store?**
> LSM-Trees: sequential writes (append WAL + MemTable insert), O(log N) write, optimized for write amplification. B-Trees: random I/O for each write (find page, update in-place), better read performance. KV stores are often write-heavy (caching, session stores, time-series) → LSM-Trees win. Read performance recovered through Bloom filters and block cache.

**Q5: How does this compare to Redis vs. DynamoDB?**
> Redis: in-memory, single-threaded, incredible speed but data limited by RAM, simpler replication (leader-follower). DynamoDB: disk-based LSM-Tree, multi-region, strongly consistent reads optional, managed by AWS. Our design is closest to DynamoDB/Cassandra: disk-based, AP by default, tunable consistency, consistent hashing for partitioning. Redis trades durability for speed; DynamoDB trades simplicity for reliability.

---

## 🔗 Related Topics
- [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md) — Ring-based partitioning
- [CAP Theorem](../../KeyConcepts/CAPTheorem.md) — Consistency vs. availability trade-off
- [Sharding](../../Database/Sharding.md) — Data distribution strategies
- [Bloom Filters](../../Database/BloomFilters.md) — Probabilistic key lookup optimization
- [Gossip Protocol](../../Microservices/GossipProtocol.md) — Failure detection mechanism

---

*"A distributed key-value store is just a HashMap that went to grad school. It learned about network failures, clock skew, split brains, and came back slightly traumatized but much more resilient." — Every Distributed Systems Engineer* 🗄️
