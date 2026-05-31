# 🔐 Distributed Locking: Coordinating Access Across Machines

> *"At Uber, two drivers can't accept the same ride. At Ticketmaster, two fans can't buy the same seat. At AWS, two Lambda functions can't process the same SQS message. Each of these problems is solved by distributed locking — and getting it wrong means double-bookings, lost money, and angry users at scale."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Consensus](../KeyConcepts/Consensus.md), [CAP Theorem](../KeyConcepts/CAPTheorem.md), [Redis Deep Dive](../Database/Redis_Deep_Dive.md)

---

## 📋 Table of Contents
1. [Why Distributed Locking is Hard](#-why-distributed-locking-is-hard)
2. [The Problem — Race Conditions at Scale](#-the-problem--race-conditions-at-scale)
3. [Lock Properties (Correctness)](#-lock-properties-correctness)
4. [Implementation Approaches](#-implementation-approaches)
5. [Redis-Based Locking (Redlock)](#-redis-based-locking-redlock)
6. [ZooKeeper-Based Locking](#-zookeeper-based-locking)
7. [Database-Based Locking](#-database-based-locking)
8. [The Martin Kleppmann vs Redis Debate](#-the-kleppmann-vs-redis-debate)
9. [Java Implementation](#-java-implementation)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 Why Distributed Locking is Hard

```
╔══════════════════════════════════════════════════════════════════╗
║  In a single process: synchronized/mutex works perfectly.      ║
║  In a distributed system: NOTHING is that simple.              ║
║                                                                ║
║  Challenges:                                                   ║
║  • Network can partition (lock holder appears dead but isn't)  ║
║  • Clocks drift (timeouts are unreliable)                     ║
║  • Processes can pause (GC pauses, swap, scheduling)          ║
║  • Messages can be delayed, duplicated, or lost               ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Key to the Meeting Room

```
SINGLE OFFICE (easy):
  One key on a hook. Take key → room is yours. Return key → others can use it.
  
DISTRIBUTED OFFICE (hard):
  10 buildings, 1 meeting room. Key must be "shared" across buildings.
  
  Problems:
  🔑 Person takes key, then their building loses power → Key is "lost"
  🔑 Two people both think they have the key (network split)
  🔑 Person takes key, falls asleep for 2 hours (GC pause)
  🔑 Key-tracking system disagrees with itself (split-brain)
```

---

## 💥 The Problem — Race Conditions at Scale

```
WITHOUT DISTRIBUTED LOCK:

  Service Instance A                    Service Instance B
  ──────────────────                    ──────────────────
  Read: inventory = 1                   Read: inventory = 1
  Check: 1 > 0? Yes!                   Check: 1 > 0? Yes!
  Deduct: inventory = 0                 Deduct: inventory = 0
  Confirm: "Order placed!" ✅          Confirm: "Order placed!" ✅
  
  RESULT: 2 orders placed, but only 1 item existed! 💀
          Customer A and B BOTH think they bought the last item!

WITH DISTRIBUTED LOCK:

  Service Instance A                    Service Instance B
  ──────────────────                    ──────────────────
  ACQUIRE LOCK("item-123") ✅          ACQUIRE LOCK("item-123") ❌ BLOCKED!
  Read: inventory = 1                   ... waiting ...
  Check: 1 > 0? Yes!                   ... waiting ...
  Deduct: inventory = 0                 ... waiting ...
  Confirm: "Order placed!" ✅          ... waiting ...
  RELEASE LOCK ✅                      ACQUIRE LOCK ✅
                                        Read: inventory = 0
                                        Check: 0 > 0? No!
                                        "Out of stock" ← CORRECT!
```

---

## ✅ Lock Properties (Correctness)

```
A correct distributed lock MUST guarantee:

┌─────────────────────────────────────────────────────────────────┐
│  1. MUTUAL EXCLUSION (Safety)                                   │
│     At most ONE client holds the lock at any time.              │
│     Two clients NEVER think they both hold it.                  │
│                                                                 │
│  2. DEADLOCK-FREE (Liveness)                                    │
│     Lock is ALWAYS eventually released, even if holder crashes. │
│     Uses TTL (time-to-live) as safety net.                     │
│                                                                 │
│  3. FAULT-TOLERANT                                              │
│     Lock service remains available even if some nodes fail.     │
│     Doesn't require ALL nodes to be up.                        │
└─────────────────────────────────────────────────────────────────┘

NICE TO HAVE:
  4. Fairness (FIFO ordering of lock requests)
  5. Reentrancy (same client can acquire lock multiple times)
  6. Fencing (detect stale lock holders)
```

---

## 🛠️ Implementation Approaches

```
┌─────────────────┬──────────────┬──────────────┬─────────────────┐
│  Approach       │  Consistency │  Availability│  Complexity     │
├─────────────────┼──────────────┼──────────────┼─────────────────┤
│  Redis (SET NX) │  Eventual    │  High        │  Low            │
│  Redis Redlock  │  Better      │  High        │  Medium         │
│  ZooKeeper      │  Strong (CP) │  Medium      │  Medium-High    │
│  etcd           │  Strong (CP) │  Medium      │  Medium         │
│  Database (row) │  Strong      │  Medium      │  Low            │
│  Consul         │  Strong (CP) │  Medium      │  Medium         │
└─────────────────┴──────────────┴──────────────┴─────────────────┘
```

---

## 🔴 Redis-Based Locking (Redlock)

### Simple Redis Lock (Single Instance)

```
ACQUIRE:
  SET resource_name my_random_value NX PX 30000
  
  NX = Only set if Not eXists (atomic!)
  PX = Expires in 30000 milliseconds (deadlock prevention)
  my_random_value = Unique ID to prevent wrong client from releasing
  
RELEASE:
  -- Lua script (atomic check + delete)
  if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
  else
      return 0
  end
  
  WHY Lua? Prevents race condition:
    Client A: GET lock → "my_value" ← correct!
    Client A: DEL lock ← But lock already expired and Client B acquired it!
    Lua script: Check AND delete atomically ✅
```

### Redlock Algorithm (Multiple Redis Instances)

```
Redlock uses 5 INDEPENDENT Redis instances (not replicas!):

  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
  │ R1  │  │ R2  │  │ R3  │  │ R4  │  │ R5  │
  └─────┘  └─────┘  └─────┘  └─────┘  └─────┘

ALGORITHM:
  1. Get current time T1
  2. Try to acquire lock on ALL 5 instances (with short timeout)
  3. Calculate elapsed time: T2 - T1
  4. Lock acquired IF:
     - Majority (≥3 of 5) instances granted the lock
     - Total time elapsed < lock TTL
  5. Lock validity time = TTL - elapsed time
  6. If lock NOT acquired → release on ALL instances

WHY 5 instances?
  - Can tolerate 2 failures (majority = 3)
  - No single point of failure
  - No replication needed (each is independent)
```

---

## 🐘 ZooKeeper-Based Locking

```
ZooKeeper: Strongly consistent, CP system (uses ZAB consensus)

ALGORITHM (using ephemeral sequential nodes):

  1. Client creates: /locks/resource-001/lock-0000000001 (ephemeral + sequential)
  2. Client lists all children of /locks/resource-001
  3. If MY node has the LOWEST sequence number → I HAVE THE LOCK! ✅
  4. Otherwise, watch the node with NEXT LOWER number (wait for deletion)
  5. When holder crashes → ephemeral node auto-deleted → next in line gets lock

  /locks/resource-001/
    ├── lock-0000000001  (Client A - holder!)
    ├── lock-0000000002  (Client B - waiting, watches 001)
    └── lock-0000000003  (Client C - waiting, watches 002)

  If Client A crashes:
    → 001 auto-deleted (ephemeral!)
    → Client B sees deletion → checks if it's lowest → YES → GOT LOCK!
    
ADVANTAGES:
  ✅ Strong consistency (linearizable)
  ✅ No TTL guessing (ephemeral = auto-cleanup on disconnect)
  ✅ Fair ordering (FIFO via sequential nodes)
  ✅ Watch mechanism (efficient waiting, no polling)
  
DISADVANTAGES:
  ❌ Higher latency than Redis (~5-50ms vs ~1ms)
  ❌ Operational complexity (ZK cluster management)
  ❌ Session timeout can cause premature lock release
```

---

## 🗄️ Database-Based Locking

```
Simple but effective for many use cases:

-- ACQUIRE (PostgreSQL advisory lock)
SELECT pg_try_advisory_lock(hashtext('resource-123'));
-- Returns true if acquired, false if already held

-- RELEASE
SELECT pg_advisory_unlock(hashtext('resource-123'));

-- OR: Row-level lock (SELECT FOR UPDATE)
BEGIN;
SELECT * FROM resources WHERE id = 'resource-123' FOR UPDATE;
-- ... do work ...
COMMIT; -- releases lock

-- OR: Dedicated locks table
INSERT INTO distributed_locks (resource_id, owner, expires_at)
VALUES ('order-456', 'instance-1', NOW() + INTERVAL '30 seconds')
ON CONFLICT (resource_id) DO NOTHING;
-- If inserted: lock acquired!
-- If conflict: someone else has it.
```

---

## 🔥 The Kleppmann vs Redis Debate

```
Martin Kleppmann (author of "Designing Data-Intensive Applications")
argued that Redlock is fundamentally broken:

THE PROBLEM — GC Pause Attack:

  Client A                    Redis              Client B
  ─────────                   ─────              ─────────
  Acquire lock (TTL=30s)      [locked by A]      
  Start processing...         
  ◄── GC PAUSE (35 sec!) ──►  
                              [lock expired!]     
                                                 Acquire lock ✅
                                                 Start processing...
  GC resume!                                     
  "I still have the lock!"   [locked by B]       "I have the lock!"
  Do write! 💀                                   Do write! 💀
  
  BOTH think they have the lock! MUTUAL EXCLUSION VIOLATED!

SOLUTION: Fencing Tokens
  Lock acquisition returns a monotonically increasing TOKEN.
  Storage system rejects writes with OLD tokens.
  
  Client A: token=33, writes with token=33 → accepted
  Client B: token=34, writes with token=34 → accepted  
  Client A: (after GC) writes with token=33 → REJECTED (34 > 33)
  
BOTTOM LINE:
  - If you need ABSOLUTE correctness → ZooKeeper + fencing tokens
  - If you can tolerate rare duplicates → Redis is fine (simpler)
  - Most systems: Redis is "good enough" with proper TTL + idempotency
```

---

## 💻 Java Implementation

### Redis Distributed Lock (Redisson)

```java
@Service
public class InventoryService {
    
    @Autowired private RedissonClient redisson;
    
    public boolean reserveItem(String itemId, String orderId) {
        // Get distributed lock for this specific item
        RLock lock = redisson.getLock("lock:inventory:" + itemId);
        
        try {
            // Try to acquire lock (wait 5s, auto-release after 30s)
            boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);
            
            if (!acquired) {
                return false; // Someone else is processing this item
            }
            
            try {
                // Critical section: only ONE instance executes this
                int stock = getStock(itemId);
                if (stock > 0) {
                    decrementStock(itemId);
                    createReservation(itemId, orderId);
                    return true;
                }
                return false; // Out of stock
            } finally {
                lock.unlock(); // Always release!
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```

### ZooKeeper Lock (Apache Curator)

```java
@Service
public class PaymentProcessingService {
    
    @Autowired private CuratorFramework curator;
    
    public PaymentResult processPayment(String paymentId) throws Exception {
        // Create distributed lock using ZooKeeper
        InterProcessMutex lock = new InterProcessMutex(
            curator, "/locks/payments/" + paymentId);
        
        try {
            // Acquire with timeout
            if (!lock.acquire(10, TimeUnit.SECONDS)) {
                throw new LockAcquisitionException("Could not acquire lock");
            }
            
            // Critical section: guaranteed mutual exclusion
            // Even across GC pauses (ephemeral node = session-based)
            return doProcessPayment(paymentId);
            
        } finally {
            if (lock.isAcquiredInThisProcess()) {
                lock.release();
            }
        }
    }
}
```

### Database Lock with Spring

```java
@Service
public class DatabaseLockService {
    
    @Autowired private JdbcTemplate jdbc;
    
    public boolean tryAcquireLock(String resourceId, String ownerId, Duration ttl) {
        int rows = jdbc.update("""
            INSERT INTO distributed_locks (resource_id, owner_id, expires_at)
            VALUES (?, ?, ?)
            ON CONFLICT (resource_id) 
            DO UPDATE SET owner_id = EXCLUDED.owner_id, 
                         expires_at = EXCLUDED.expires_at
            WHERE distributed_locks.expires_at < NOW()
            """,
            resourceId, ownerId, Instant.now().plus(ttl));
        
        return rows > 0; // 1 = acquired, 0 = already held by someone else
    }
    
    public void releaseLock(String resourceId, String ownerId) {
        jdbc.update("""
            DELETE FROM distributed_locks 
            WHERE resource_id = ? AND owner_id = ?
            """, resourceId, ownerId);
    }
}
```

---

## ⚠️ Common Pitfalls

| Pitfall | Why It's Dangerous | Fix |
|---------|-------------------|-----|
| 🔴 No TTL/expiration | Client crashes → lock held forever (deadlock) | Always set TTL as safety net |
| 🔴 Releasing wrong lock | Client releases lock it doesn't own | Include owner ID, check before release |
| 🔴 Assuming lock survives GC pauses | Lock expires during pause, other acquires | Use fencing tokens |
| 🔴 Single Redis instance | Redis crash → lock lost → mutual exclusion violated | Use Redlock or ZooKeeper |
| 🟡 Too short TTL | Lock expires before work completes | Auto-renewal (watchdog timer) |
| 🟡 Too long TTL | Client crash → long wait before lock is released | Balance: 30s-60s typical |

---

## 🎮 Mini Challenge

### 🧩 Design: Distributed Lock for Flash Sale

A flash sale starts at 12:00:00. 10,000 users try to buy the last 100 iPhones simultaneously.

Requirements:
- No overselling (strong mutual exclusion)
- Low latency (< 100ms per purchase)
- Handle 10K concurrent lock requests
- Survive server failures

**Design questions:**
1. Redis or ZooKeeper? Why?
2. Lock per item or lock per SKU?
3. What's your TTL?
4. How do you handle the "thundering herd" problem at 12:00:00?
5. What happens if lock holder crashes mid-transaction?

<details>
<summary>🔑 Answers</summary>

1. **Redis** — Lower latency (< 1ms), handles high concurrency better. ZooKeeper would become a bottleneck at 10K qps.
2. **Lock per SKU** with stock counter — Don't lock individual items, lock the decrement operation on the counter.
3. **5 seconds** — Purchase should complete in < 1s, 5s gives buffer for retries.
4. **Rate limiting + queue** — Don't let all 10K hit the lock simultaneously. Queue requests, process in batches.
5. **TTL auto-releases** — After 5s, lock expires. Next client in queue acquires it. Use idempotency key to prevent double-purchase on retry.
</details>

---

## ❓ Interview Q&A

**Q1: Why can't you use a simple mutex for distributed locking?**
> Mutexes work within a single process/JVM. In distributed systems, multiple processes on different machines need coordination. A distributed lock must be stored in a shared system (Redis, ZooKeeper, DB) that all processes can access, and must handle network failures, clock drift, and process crashes.

**Q2: Explain the Redlock algorithm.**
> Use 5 independent Redis instances. To acquire: try to SET NX on all 5 with the same TTL. If majority (≥3) grant the lock within the TTL period, the lock is acquired. If not, release on all instances. This ensures no single Redis failure breaks the lock guarantee.

**Q3: When would you choose ZooKeeper over Redis for locking?**
> When you need STRONG consistency guarantees and can tolerate higher latency (~10-50ms). ZooKeeper uses consensus (ZAB), so locks survive leader failures correctly. Also when you need fair ordering (FIFO) of lock acquisitions. Redis is better when speed matters and rare duplicates are acceptable.

**Q4: What are fencing tokens and why do they matter?**
> A monotonically increasing number returned with each lock acquisition. The storage layer rejects writes with tokens lower than the current maximum. This prevents stale lock holders (e.g., after GC pause) from corrupting data — even if they believe they still hold the lock.

**Q5: How do you prevent deadlocks in distributed locking?**
> (1) Always set a TTL/expiration (safety net if holder crashes), (2) Use lock ordering (always acquire locks in same order), (3) Use try-lock with timeout (give up after N seconds), (4) Automatic lock renewal (extend TTL while still processing).

---

## 🔗 Related Topics
- [Consensus Algorithms](../KeyConcepts/Consensus.md) — The theory behind lock correctness
- [Redis Deep Dive](../Database/Redis_Deep_Dive.md) — Redis lock implementation details
- [CAP Theorem](../KeyConcepts/CAPTheorem.md) — Why distributed locks are hard
- [Circuit Breaker](../BuildingBlocks/CircuitBreaker.md) — Handling lock service failures

---

*"Distributed locking is the dark art of distributed systems. It looks simple until you realize that time, networks, and processes are all conspiring against you." — Every distributed systems engineer ever* 🔐
