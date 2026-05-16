# 🔄 Replication: Keeping Data Copies in Sync Across Nodes

> *"Every time you read from a database replica, you're trusting that the copy is close enough to the truth. Replication is the art of keeping multiple copies of data consistent — and the trade-offs you make define your system's behavior under failure."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [CAP Theorem](./CAPTheorem.md), [Availability](./Availability.md)

---

## 📋 Table of Contents
1. [Why Replication](#-why-replication)
2. [Replication Models](#-replication-models)
3. [Leader-Follower (Master-Slave)](#-leader-follower)
4. [Multi-Leader (Master-Master)](#-multi-leader)
5. [Leaderless (Quorum-Based)](#-leaderless)
6. [Conflict Resolution](#-conflict-resolution)
7. [Replication Lag & Consistency Guarantees](#-replication-lag--consistency-guarantees)
8. [Code Example](#-code-example)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 Why Replication

```
GOALS OF REPLICATION:
  1. HIGH AVAILABILITY:  If one node dies, others serve traffic (no downtime)
  2. READ SCALING:       Distribute reads across N replicas (N× read throughput)
  3. GEOGRAPHIC LOCALITY: Place data close to users (lower latency across regions)
  4. DISASTER RECOVERY:  Data survives datacenter failures

THE FUNDAMENTAL CHALLENGE:
  How do you keep N copies of data consistent when:
    - Networks can partition (messages lost/delayed)
    - Nodes can crash at any moment
    - Writes happen concurrently
    - You want low latency (can't wait for all nodes every time)

  Answer: You CAN'T have perfect consistency + availability + partition tolerance
          (CAP theorem). Replication strategies choose different trade-offs.
```

---

## 🏗️ Replication Models

```
┌────────────────────────────────────────────────────────────────────────────┐
│ MODEL              │ WRITES TO   │ CONSISTENCY  │ USE CASE                 │
├────────────────────┼─────────────┼──────────────┼──────────────────────────┤
│ Leader-Follower    │ Leader only │ Strong*      │ Most databases (default) │
│ (Master-Slave)     │             │              │ PostgreSQL, MySQL, Redis │
├────────────────────┼─────────────┼──────────────┼──────────────────────────┤
│ Multi-Leader       │ Any leader  │ Eventual     │ Multi-DC, collaborative  │
│ (Master-Master)    │             │              │ editing, offline-first   │
├────────────────────┼─────────────┼──────────────┼──────────────────────────┤
│ Leaderless         │ Any node    │ Tunable      │ Cassandra, DynamoDB,     │
│ (Quorum-based)     │             │ (quorum)     │ Riak                     │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 👑 Leader-Follower

```
ARCHITECTURE:
  [Client] → WRITE → [Leader/Master]
                        │
                   replicate (async or sync)
                        │
              ┌─────────┼─────────┐
              ▼         ▼         ▼
          [Follower] [Follower] [Follower]
              │         │         │
              ← ← ←  READ  ← ← ←
                    [Client]

SYNCHRONOUS vs ASYNCHRONOUS REPLICATION:

  SYNCHRONOUS:
    Leader waits for follower(s) to confirm write before acknowledging client
    ✅ Follower guaranteed to have latest data (strong consistency)
    ❌ Write latency = leader + slowest follower (higher latency)
    ❌ If any sync follower is down, writes block
    Used: PostgreSQL (synchronous_commit), MySQL (semi-sync)

  ASYNCHRONOUS:
    Leader acknowledges immediately, replicates in background
    ✅ Low write latency (leader only)
    ✅ Leader doesn't block if followers are slow/down
    ❌ Followers may lag behind (stale reads)
    ❌ If leader crashes before replication, data lost
    Used: Most default configurations, Redis Sentinel

  SEMI-SYNCHRONOUS (practical compromise):
    One follower is synchronous (guaranteed up-to-date)
    Other followers are asynchronous
    If sync follower fails, promote another to sync role
    Used: MySQL semi-synchronous replication

FAILOVER PROCESS:
  1. Detect leader failure (heartbeat timeout, typically 10-30s)
  2. Choose new leader (most up-to-date follower)
  3. Reconfigure followers to follow new leader
  4. Redirect client writes to new leader
  
  Dangers during failover:
    - Split brain: old leader comes back, thinks it's still leader
    - Data loss: if async replication, new leader may miss writes
    - Stale reads: clients reading from stale followers during transition
```

---

## 🔀 Multi-Leader

```
ARCHITECTURE:
  [DC: US-East]          [DC: EU-West]          [DC: AP-South]
  ┌──────────┐           ┌──────────┐           ┌──────────┐
  │ Leader 1 │◄─────────▶│ Leader 2 │◄─────────▶│ Leader 3 │
  │Followers │           │Followers │           │Followers │
  └──────────┘           └──────────┘           └──────────┘
       ▲                      ▲                      ▲
   US users               EU users               Asia users
   write here             write here             write here

WHEN TO USE:
  ✅ Multi-datacenter deployments (each DC has a leader for local writes)
  ✅ Offline-first apps (phone/laptop is a "leader" when offline)
  ✅ Collaborative editing (Google Docs — each user's session is a "leader")

THE CONFLICT PROBLEM:
  User A in US updates profile name to "Alice"    (at time T1)
  User B in EU updates profile name to "Bob"      (at time T1)
  
  Both leaders accept the write locally → conflict when replicating!
  
  Which write wins? How do you resolve?
```

---

## 🎲 Leaderless

```
ARCHITECTURE (Dynamo-style):
  [Client] → WRITE to N nodes simultaneously (or coordinator fans out)
  
  Write to 3 nodes:   Read from 3 nodes:
  ┌──────┐            ┌──────┐
  │Node A│ ← write    │Node A│ → returns v2
  │Node B│ ← write    │Node B│ → returns v2
  │Node C│ ← write    │Node C│ → returns v1 (stale!)
  └──────┘            └──────┘
                            ↓
                      Client takes newest version (v2)

QUORUM FORMULA:
  N = total replicas
  W = write acknowledgments needed
  R = read replicas queried
  
  Strong consistency: W + R > N
  
  Example (N=3):
    W=2, R=2 → 2+2=4 > 3 ✅ (at least 1 node has latest for both read & write)
    W=1, R=1 → 1+1=2 < 3 ❌ (may read from a node that hasn't received the write)

ANTI-ENTROPY & READ REPAIR:
  Problem: stale replicas need to catch up
  
  Read repair: when client reads and detects stale replica, it sends the 
               latest value back to the stale node
  
  Anti-entropy: background process that compares replicas and syncs differences
                (Merkle trees for efficient comparison)
```

---

## ⚔️ Conflict Resolution

```
STRATEGIES WHEN CONCURRENT WRITES CONFLICT:

1. LAST-WRITE-WINS (LWW):
   Attach timestamp to each write. Highest timestamp wins.
   ✅ Simple, deterministic
   ❌ Data loss (earlier write silently discarded)
   ❌ Clock skew can cause "wrong" winner
   Used by: Cassandra, DynamoDB (default)

2. VERSION VECTORS (detect conflicts, let app resolve):
   Each node maintains a version counter
   On conflict: return BOTH versions to the client
   Client/app decides how to merge
   Used by: Riak, CouchDB

3. CRDT (Conflict-free Replicated Data Types):
   Data structures mathematically guaranteed to converge
   Examples: G-Counter, OR-Set, LWW-Register
   ✅ No conflicts by design (automatic merge)
   ❌ Limited data structures available
   Used by: Redis (CRDT-based active-active), Riak

4. OPERATIONAL TRANSFORMATION (OT):
   Transform concurrent operations to be compatible
   Used by: Google Docs real-time collaborative editing
   Complex to implement correctly

5. APPLICATION-LEVEL RESOLUTION:
   On conflict, store both versions
   Show user a "conflict" and let them choose
   Used by: Git (merge conflicts), Dropbox (conflicted copy)
```

---

## ⏱️ Replication Lag & Consistency Guarantees

```
REPLICATION LAG = time between write on leader and visible on follower

Typical lag:
  Same datacenter: 1-100ms
  Cross-datacenter: 100-1000ms
  Under heavy load: seconds to minutes

CONSISTENCY GUARANTEES (what your app sees):

  READ-YOUR-WRITES (Read-after-write):
    After you write, you always see your own write
    Implementation: route your reads to leader (or read from follower 
    only if follower has caught up to your write timestamp)

  MONOTONIC READS:
    You never see time go backward (no seeing newer then older data)
    Implementation: always read from the same replica

  CONSISTENT PREFIX READS:
    If write A happened before B, no one reads B without A
    Implementation: causal ordering of related writes to same partition

  LINEARIZABILITY (strongest):
    System behaves as if there's a single copy of data
    All reads see the most recent write, globally
    Implementation: synchronous replication or consensus protocol
    Cost: higher latency, lower availability during partitions
```

---

## 💻 Code Example

```java
// Spring Boot — Read/Write splitting with @Transactional
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = Map.of(
            "master", masterDataSource(),
            "slave", slaveDataSource()
        );
        
        var routingDS = new ReadWriteRoutingDataSource();
        routingDS.setTargetDataSources(targetDataSources);
        routingDS.setDefaultTargetDataSource(masterDataSource());
        return routingDS;
    }
}

public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly() 
            ? "slave" : "master";
    }
}

@Service
public class UserService {
    
    @Transactional  // writes go to master
    public User createUser(CreateUserRequest req) {
        return userRepository.save(new User(req.name(), req.email()));
    }
    
    @Transactional(readOnly = true)  // reads go to slave
    public User getUser(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    // Read-your-writes: after create, read from master
    @Transactional  // forces master read
    public User createAndReturn(CreateUserRequest req) {
        User user = userRepository.save(new User(req.name(), req.email()));
        return userRepository.findById(user.getId()).orElseThrow();
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Ignoring replication lag in application code** — Reading from a replica immediately after writing to the leader may return stale data. Ensure "read-your-writes" consistency for user-facing flows (read from master after writes, or wait for replication to catch up).

2. **Split-brain during failover** — If the old leader comes back online after failover without knowing a new leader was elected, you get two leaders accepting writes. Use fencing tokens, STONITH, or consensus-based leader election to prevent this.

3. **Assuming synchronous replication is free** — Synchronous replication increases write latency proportional to the slowest replica. One slow/distant replica blocks all writes. Use semi-synchronous (one sync + N async) for a practical balance.

4. **Not monitoring replication lag** — Replication lag can grow silently until users report stale data. Monitor `seconds_behind_master` (MySQL) or `replay_lag` (PostgreSQL) and alert when lag exceeds your SLO threshold.

---

## 🧩 Mini Challenge

**A user updates their profile picture. They immediately refresh the page but see the old picture. 10 seconds later, the new picture appears. Explain what's happening and propose two solutions.**

<details>
<summary>💡 Click to reveal answer</summary>

**What's happening:**
The write goes to the primary/leader database, but the subsequent page refresh reads from a replica that hasn't received the update yet (replication lag). After ~10 seconds, the replica catches up.

**Solution 1: Read-your-writes consistency**
```
After a user writes, for the next N seconds, route THEIR reads to the leader:
  - Set a cookie/session flag: "last_write_timestamp = 1705312800"
  - On subsequent reads, if current_time - last_write_timestamp < 30s:
    → Route to leader (guaranteed to see own write)
  - After 30s, resume reading from replicas (lag has caught up)
```

**Solution 2: Causal consistency with version check**
```
  Write response includes a version/LSN: { "version": 42 }
  On next read request, client sends: "If-None-Match: version-42"
  Server checks if replica has reached version 42:
    - If yes → serve from replica
    - If no → route to leader (or wait for replica to catch up)
```

**Solution 3: Client-side optimistic update**
```
  Update UI immediately (optimistic) before server confirms
  Background: server replicates to all nodes
  If server fails → rollback UI change
  User never sees stale data because client already shows the new value
```

</details>

---

## 📝 Interview Q&A

**Q: Compare synchronous vs asynchronous replication. When would you choose each?**
> A: **Synchronous**: Leader waits for follower acknowledgment before confirming write. Guarantees no data loss on failover (follower is always up-to-date). Choose for: financial transactions, user authentication state, any data where loss is unacceptable. Trade-off: higher write latency, writes blocked if sync follower is down. **Asynchronous**: Leader confirms immediately, replicates in background. Choose for: high-throughput systems where small data loss on failover is acceptable (social media posts, analytics events). Trade-off: if leader crashes before replicating, those writes are lost. **Practical choice**: semi-synchronous — one follower is sync (guaranteed up-to-date backup), others are async (for read scaling without blocking writes).

**Q: How do distributed databases like Cassandra handle replication differently from PostgreSQL?**
> A: PostgreSQL uses leader-follower: all writes go to one leader, replicated to followers. Simple, strong consistency from leader, but single write bottleneck. Cassandra uses leaderless/quorum replication: writes go to multiple nodes simultaneously (coordinator fans out to N replicas). Client specifies consistency level per query (ONE, QUORUM, ALL). No single point of failure for writes — any node can accept any write. Trade-off: conflicts are possible (resolved by LWW timestamps), eventual consistency by default (tunable to strong with QUORUM reads + QUORUM writes), and stale replicas need anti-entropy/read-repair to converge.

---

## 🔗 What to Read Next

1. **[KeyConcepts/Consensus.md](./Consensus.md)** — How nodes agree on a leader (Raft, Paxos)
2. **[KeyConcepts/CAPTheorem.md](./CAPTheorem.md)** — Why you can't have it all
3. **[Database/Sharding.md](../Database/Sharding.md)** — Combining replication with partitioning

---

*[← Consistent Hashing](./Consistent_Hashing.md) | [Back to Index](../INDEX.md) | [Next: Consensus →](./Consensus.md)*
