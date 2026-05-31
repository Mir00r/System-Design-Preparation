# 🔐 Design a Distributed Locking Service (Chubby/ZooKeeper)

> *"In a single-server world, you use `synchronized` or `ReentrantLock` and life is simple. But when 100 servers need to coordinate — only ONE should process a payment, only ONE should run a cron job, only ONE should hold leadership — you need a DISTRIBUTED lock. This is the design behind Google's Chubby, Apache ZooKeeper, and etcd's lock API. It combines consensus algorithms (Raft/Paxos), lease-based locks with automatic expiry, fencing tokens to prevent zombie processes, and the fundamental CAP theorem trade-off that makes distributed locking one of the hardest problems in computer science."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Distributed Locking](../Microservices/DistributedLocking.md), [CAP Theorem](../KeyConcepts/), [Consistency](../Tradeoffs/Strong_vs_Eventual_Consistency.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Consensus & Replication (Raft)](#-consensus--replication-raft)
4. [Lock Semantics & Lease Mechanism](#-lock-semantics--lease-mechanism)
5. [Fencing Tokens (Preventing Zombie Locks)](#-fencing-tokens-preventing-zombie-locks)
6. [Leader Election](#-leader-election)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Acquire lock (with timeout!)
  • Release lock (explicit or on session death!)
  • Lock with lease/TTL (auto-expire if holder crashes!)
  • Fencing token (monotonic token to detect stale locks!)
  • Blocking wait (wait until lock available!)
  • Try-lock (non-blocking: get lock or fail immediately!)
  • Read-write locks (multiple readers OR single writer!)
  • Lock groups/hierarchical locks
  • Leader election (built on top of locking!)
  • Watch/notification (get notified when lock released!)

NON-FUNCTIONAL:
  • Strong consistency (linearizability! CP system!)
  • Lock acquisition: < 10ms (for available locks!)
  • Handles network partitions gracefully!
  • Survives minority node failures (3/5, 2/3!)
  • Fencing: NEVER allow two holders to both succeed!
  • Auto-release on client crash (lease timeout!)

SCALE:
  • 1M active locks simultaneously
  • 100K lock operations per second
  • 5-7 node cluster (odd numbers for majority!)
  • Clients: 10K+ concurrent connections
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                  DISTRIBUTED LOCKING SERVICE                                 │
│                                                                             │
│  Clients                    Locking Cluster (Raft consensus!)               │
│  ┌──────────┐              ┌───────────────────────────────────────┐       │
│  │ Service A│──acquire──►  │                                       │       │
│  └──────────┘              │   ┌─────────┐                         │       │
│  ┌──────────┐              │   │ LEADER  │ ← all writes go here!  │       │
│  │ Service B│──acquire──►  │   │ Node 1  │                         │       │
│  └──────────┘              │   └────┬────┘                         │       │
│  ┌──────────┐              │        │ Raft replication!            │       │
│  │ Service C│──acquire──►  │   ┌────┼────────────────┐            │       │
│  └──────────┘              │   ▼    ▼                ▼            │       │
│                             │ ┌────────┐ ┌────────┐ ┌────────┐   │       │
│                             │ │Follower│ │Follower│ │Follower│   │       │
│                             │ │ Node 2 │ │ Node 3 │ │ Node 4 │   │       │
│                             │ └────────┘ └────────┘ └────────┘   │       │
│                             │                                      │       │
│                             │ ┌────────┐                           │       │
│                             │ │Follower│   (5-node cluster!        │       │
│                             │ │ Node 5 │    tolerates 2 failures!) │       │
│                             │ └────────┘                           │       │
│                             └───────────────────────────────────────┘       │
│                                                                             │
│  LOCK STATE (replicated via Raft!):                                        │
│  ┌──────────────────────────────────────────────────────────────────┐     │
│  │  Lock Key          │ Owner     │ Fence Token │ Lease Until       │     │
│  │ "/payments/proc-1" │ client-A  │     42      │ 2024-01-15T10:05 │     │
│  │ "/cron/daily-job"  │ client-B  │     87      │ 2024-01-15T10:03 │     │
│  │ "/leader/svc-X"    │ client-C  │    156      │ 2024-01-15T10:10 │     │
│  └──────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🗳️ Consensus & Replication (Raft)

```
WHY CONSENSUS? (the fundamental question!):
  
  Single Redis lock: if Redis crashes → lock lost → double execution! 💀
  We need: lock state REPLICATED across multiple nodes!
  But replication needs AGREEMENT: which write happened first?
  
  Answer: RAFT CONSENSUS (or Paxos!)

RAFT PROTOCOL (simplified!):
  ┌────────────────────────────────────────────────────────────────────┐
  │  LEADER ELECTION:                                                   │
  │  • One node is LEADER at any time (handles all writes!)            │
  │  • Leader sends heartbeats every 150ms                              │
  │  • If follower doesn't receive heartbeat for 300ms → election!     │
  │  • Candidate requests votes from all nodes                          │
  │  • Majority (3/5) votes → becomes new leader!                      │
  │  • Guaranteed: only ONE leader per term (split-brain impossible!)  │
  │                                                                     │
  │  LOG REPLICATION:                                                    │
  │  • Client sends "acquire lock X" to leader                         │
  │  • Leader appends to its log: {term: 5, op: "acquire", key: "X"}  │
  │  • Leader replicates to followers (AppendEntries RPC!)             │
  │  • Once MAJORITY acknowledge → COMMITTED!                          │
  │  • Leader applies to state machine → responds to client!           │
  │                                                                     │
  │  KEY PROPERTY:                                                      │
  │  If entry committed → guaranteed on majority of nodes!             │
  │  Even if leader crashes → new leader has committed entries!        │
  │  → Lock state is NEVER lost! (as long as majority alive!)         │
  └────────────────────────────────────────────────────────────────────┘

LINEARIZABILITY (strongest consistency!):
  ┌────────────────────────────────────────────────────────────────────┐
  │  Once a lock is acquired:                                           │
  │  • ALL subsequent reads see it as acquired!                        │
  │  • Even if they go to different nodes!                             │
  │  • No stale reads! (reads must go through leader OR verify!)      │
  │                                                                     │
  │  Implementation:                                                    │
  │  • Reads go through leader (simple, slight overhead!)             │
  │  • OR: read from follower but verify with leader first (ReadIndex)│
  │  • This is what makes it a CP system (not AP!)                    │
  │                                                                     │
  │  Trade-off: during network partition:                              │
  │  • Minority side: CANNOT acquire locks (no majority!)             │
  │  • Majority side: continues operating normally!                   │
  │  • SAFE: never two holders! (but some clients blocked!)           │
  └────────────────────────────────────────────────────────────────────┘
```

---

## ⏱️ Lock Semantics & Lease Mechanism

```
LEASE-BASED LOCKS (the CRITICAL design!):

  WHY LEASES? Without them:
  Client acquires lock → crashes → lock held FOREVER! (deadlock! 💀)
  
  With leases:
  Client acquires lock with TTL (e.g., 30 seconds!)
  Client must RENEW before TTL expires! (heartbeat!)
  If client crashes → no renewal → lock auto-expires! ✅

LOCK LIFECYCLE:
  ┌────────────────────────────────────────────────────────────────┐
  │  1. ACQUIRE:                                                    │
  │     Client → Leader: "Lock /job/daily, TTL=30s"                │
  │     Leader checks: is it free?                                  │
  │     YES: grant lock, set lease_until = now + 30s!              │
  │     NO: reject (or wait in queue!)                             │
  │     Return: {success, fencing_token: 42, lease_until: T+30}   │
  │                                                                 │
  │  2. RENEW (keep-alive!):                                       │
  │     Client sends renewal every 10s (well before 30s TTL!)      │
  │     Leader: extend lease_until by another 30s!                  │
  │     If 3 renewals fail → assume leader/network issue!          │
  │                                                                 │
  │  3. RELEASE (explicit!):                                       │
  │     Client → Leader: "Release /job/daily"                      │
  │     Leader: delete lock, notify waiters!                        │
  │                                                                 │
  │  4. EXPIRE (crash recovery!):                                  │
  │     Client crashed! No renewals!                                │
  │     Background thread: check all locks every 1s                 │
  │     lease_until < now? → EXPIRE lock! Free it!                 │
  │     Waiters notified: "lock available!"                        │
  └────────────────────────────────────────────────────────────────┘

SESSION-BASED APPROACH (ZooKeeper-style!):
  • Client creates ephemeral node (session-tied!)
  • Node exists = lock held!
  • Client dies → session dies → ephemeral node deleted → lock released!
  • No TTL management needed! (session handles it!)
  • Session maintained by heartbeats to ZK cluster!
```

---

## 🛡️ Fencing Tokens (Preventing Zombie Locks)

```
THE ZOMBIE PROBLEM (why TTL alone isn't enough!):

  ┌────────────────────────────────────────────────────────────────┐
  │  Time 0: Client A acquires lock, token=42                      │
  │  Time 5: Client A does GC pause (30 seconds!) 😱              │
  │  Time 35: Lock expires (TTL passed!)                           │
  │  Time 36: Client B acquires lock, token=43                     │
  │  Time 37: Client A wakes up! Thinks it still has lock!        │
  │  Time 38: Client A writes to database! ← WRONG! 💀            │
  │           Client B also writes! BOTH think they have lock!     │
  └────────────────────────────────────────────────────────────────┘
  
  Lock TTL expired but client A DIDN'T KNOW!
  Result: TWO clients both acting as lock holder! 😱

FENCING TOKEN SOLUTION:
  ┌────────────────────────────────────────────────────────────────┐
  │  Each lock acquisition gets a MONOTONICALLY INCREASING token!   │
  │                                                                 │
  │  Client A gets lock: fencing_token = 42                         │
  │  Client B gets lock after expiry: fencing_token = 43           │
  │                                                                 │
  │  ALL operations must include the fencing token!                 │
  │                                                                 │
  │  Client A (zombie!) tries: "write X, token=42"                 │
  │  Storage checks: "I've seen token 43, rejecting token 42!"    │
  │  REJECTED! ✅ No corruption!                                   │
  │                                                                 │
  │  Client B tries: "write X, token=43"                           │
  │  Storage checks: "43 ≥ my highest seen (43), ACCEPTED!"       │
  │                                                                 │
  │  RULE: storage/database MUST reject operations with            │
  │        fencing tokens LOWER than the highest seen!             │
  │                                                                 │
  │  This requires storage layer cooperation!                       │
  │  (not just the lock service — downstream systems too!)         │
  └────────────────────────────────────────────────────────────────┘

PRACTICAL FENCING:
  // Database operation with fencing!
  UPDATE accounts 
  SET balance = balance - 100 
  WHERE id = 'user-42' 
    AND lock_token <= 43;  -- Reject if higher token already used!
  
  // If affected rows = 0 → our token was superseded! ABORT!
```

---

## 👑 Leader Election

```
LEADER ELECTION (built on distributed locks!):

  Multiple instances of a service → only ONE should be "leader"!
  (Leader handles: cron jobs, rebalancing, coordination!)

  ┌────────────────────────────────────────────────────────────────┐
  │  IMPLEMENTATION:                                                │
  │                                                                 │
  │  All instances try to acquire lock "/leader/my-service"         │
  │                                                                 │
  │  Instance A: acquire("/leader/my-service") → SUCCESS! (leader!)│
  │  Instance B: acquire("/leader/my-service") → BLOCKED (waits!)  │
  │  Instance C: acquire("/leader/my-service") → BLOCKED (waits!)  │
  │                                                                 │
  │  Instance A: renews lease every 10 seconds (stay leader!)       │
  │                                                                 │
  │  Instance A crashes!                                            │
  │  → Lease expires after 30 seconds!                             │
  │  → Lock released!                                               │
  │  → Instance B (first in queue) acquires → NEW LEADER!          │
  │                                                                 │
  │  WATCH pattern (ZooKeeper!):                                    │
  │  Instance B: watch("/leader/my-service")                        │
  │  When lock released → B gets notification → B tries to acquire!│
  │  (No polling needed! Event-driven!)                            │
  └────────────────────────────────────────────────────────────────┘

SPLIT-BRAIN PREVENTION:
  Old leader (network isolated!) still thinks it's leader!
  New leader elected by majority!
  
  Solution: FENCING TOKEN! 
  Old leader's token (42) < new leader's token (43)
  All operations by old leader REJECTED by downstream systems!
  
  Also: leader should CHECK its lease before critical operations!
  "Is my lease still valid?" → If not → STOP immediately!
```

---

## 💻 Java Implementation

### Lock Server (Core Logic)

```java
@Service
public class DistributedLockManager {
    
    private final ConcurrentHashMap<String, LockEntry> locks = new ConcurrentHashMap<>();
    private final AtomicLong fencingTokenGenerator = new AtomicLong(0);
    private final Map<String, Queue<CompletableFuture<LockGrant>>> waitQueues = 
        new ConcurrentHashMap<>();
    private final RaftConsensus raft; // Raft for replication!
    
    /**
     * Acquire a distributed lock with lease TTL.
     * Replicated via Raft for fault tolerance!
     */
    public CompletableFuture<LockGrant> acquireLock(String lockKey, 
                                                     String clientId,
                                                     Duration leaseDuration,
                                                     Duration waitTimeout) {
        // Must go through Raft leader!
        if (!raft.isLeader()) {
            throw new NotLeaderException(raft.getLeaderAddress());
        }
        
        // Create lock command
        LockCommand cmd = LockCommand.acquire(lockKey, clientId, 
            leaseDuration, fencingTokenGenerator.incrementAndGet());
        
        // Replicate via Raft (committed once majority acknowledges!)
        return raft.propose(cmd).thenApply(committed -> {
            if (committed) {
                return applyAcquire(cmd);
            }
            throw new LockException("Failed to replicate lock command");
        });
    }
    
    /**
     * Apply acquire command to state machine (after Raft commits!).
     */
    private LockGrant applyAcquire(LockCommand cmd) {
        String lockKey = cmd.getLockKey();
        
        synchronized (getLockObject(lockKey)) {
            LockEntry existing = locks.get(lockKey);
            
            // Check if lock is free (or expired!)
            if (existing == null || existing.isExpired()) {
                // GRANT the lock!
                LockEntry entry = LockEntry.builder()
                    .lockKey(lockKey)
                    .owner(cmd.getClientId())
                    .fencingToken(cmd.getFencingToken())
                    .leaseUntil(Instant.now().plus(cmd.getLeaseDuration()))
                    .build();
                locks.put(lockKey, entry);
                
                return LockGrant.granted(
                    lockKey, entry.getFencingToken(), entry.getLeaseUntil());
            }
            
            // Lock held by someone else → queue the request!
            CompletableFuture<LockGrant> future = new CompletableFuture<>();
            waitQueues.computeIfAbsent(lockKey, k -> new ConcurrentLinkedQueue<>())
                .add(future);
            
            // Schedule timeout
            scheduler.schedule(() -> {
                if (!future.isDone()) {
                    future.complete(LockGrant.timedOut(lockKey));
                    waitQueues.get(lockKey).remove(future);
                }
            }, cmd.getWaitTimeout().toMillis(), TimeUnit.MILLISECONDS);
            
            return future.join(); // Block until granted or timeout!
        }
    }
    
    /**
     * Renew lease (client heartbeat!).
     */
    public boolean renewLease(String lockKey, String clientId, 
                              long fencingToken, Duration extension) {
        LockEntry entry = locks.get(lockKey);
        
        if (entry == null || !entry.getOwner().equals(clientId)) {
            return false; // Not the owner!
        }
        if (entry.getFencingToken() != fencingToken) {
            return false; // Token mismatch (lock was re-acquired!)
        }
        
        // Extend lease!
        entry.setLeaseUntil(Instant.now().plus(extension));
        
        // Replicate renewal via Raft!
        raft.propose(LockCommand.renew(lockKey, clientId, extension));
        
        return true;
    }
    
    /**
     * Release lock explicitly.
     */
    public void releaseLock(String lockKey, String clientId, long fencingToken) {
        LockEntry entry = locks.get(lockKey);
        
        if (entry == null || !entry.getOwner().equals(clientId)) {
            return; // Not the owner, nothing to release!
        }
        if (entry.getFencingToken() != fencingToken) {
            return; // Stale release (token superseded!)
        }
        
        // Remove lock!
        locks.remove(lockKey);
        
        // Notify next waiter!
        Queue<CompletableFuture<LockGrant>> waiters = waitQueues.get(lockKey);
        if (waiters != null && !waiters.isEmpty()) {
            CompletableFuture<LockGrant> next = waiters.poll();
            // Grant to next in queue (will go through Raft!)
            next.complete(LockGrant.granted(lockKey, 
                fencingTokenGenerator.incrementAndGet(), 
                Instant.now().plus(Duration.ofSeconds(30))));
        }
    }
    
    /**
     * Background: expire stale locks (crashed clients!).
     */
    @Scheduled(fixedRate = 1000) // Every 1 second!
    public void expireLeases() {
        if (!raft.isLeader()) return; // Only leader expires locks!
        
        Instant now = Instant.now();
        locks.forEach((key, entry) -> {
            if (entry.getLeaseUntil().isBefore(now)) {
                log.warn("Lock {} expired! Owner {} missed renewal!", 
                    key, entry.getOwner());
                releaseLock(key, entry.getOwner(), entry.getFencingToken());
            }
        });
    }
}
```

### Client-Side Lock Usage

```java
/**
 * Client wrapper for using distributed locks safely!
 */
@Component
public class DistributedLockClient {
    
    private final LockServiceStub lockService;
    private final ScheduledExecutorService renewalExecutor;
    
    /**
     * Execute a task while holding a distributed lock.
     * Handles: acquire, renewal, release, fencing!
     */
    public <T> T withLock(String lockKey, Duration timeout, 
                           Callable<T> task) throws Exception {
        // Acquire lock!
        LockGrant grant = lockService.acquire(lockKey, clientId, 
            Duration.ofSeconds(30), timeout);
        
        if (!grant.isGranted()) {
            throw new LockNotAcquiredException("Could not acquire: " + lockKey);
        }
        
        // Start renewal thread (keep lock alive!)
        ScheduledFuture<?> renewal = renewalExecutor.scheduleAtFixedRate(
            () -> {
                boolean renewed = lockService.renew(lockKey, clientId, 
                    grant.getFencingToken(), Duration.ofSeconds(30));
                if (!renewed) {
                    log.error("Lock renewal failed! Aborting task!");
                    Thread.currentThread().interrupt();
                }
            }, 
            10, 10, TimeUnit.SECONDS); // Renew every 10s (well before 30s TTL!)
        
        try {
            // Execute the protected task!
            // Pass fencing token to downstream operations!
            FencingContext.set(grant.getFencingToken());
            return task.call();
        } finally {
            // Stop renewal + release lock!
            renewal.cancel(false);
            lockService.release(lockKey, clientId, grant.getFencingToken());
            FencingContext.clear();
        }
    }
}

// Usage:
lockClient.withLock("/payments/process-batch", Duration.ofSeconds(5), () -> {
    // Only ONE instance executes this at a time!
    long token = FencingContext.get();
    processPaymentBatch(token); // Pass fencing token to DB operations!
    return null;
});
```

---

## ❓ Interview Q&A

**Q1: Why can't you just use Redis SETNX for distributed locking?**
> Redis SETNX works for BEST-EFFORT locking but fails for CORRECTNESS-CRITICAL locking! Problems: (1) Single point of failure: Redis master crashes → lock lost → two holders! (2) Even with Redis Cluster: during failover, lock may not replicate to new master (async replication!) → two holders! (3) No fencing tokens: zombie clients can still corrupt data! (4) Clock-dependent TTL: clock skew between Redis and client → premature expiry! Redis is fine for: rate limiting, cache stampede prevention, job deduplication (where occasional double-execution is OK). For SAFETY: use Raft-based systems (ZooKeeper, etcd) that guarantee linearizable operations!

**Q2: Explain the Redlock algorithm and its criticism.**
> Redlock (by Redis creator): acquire lock on N/2+1 independent Redis masters. If majority acquired within timeout → lock is held! Criticism (Martin Kleppmann): (1) Still clock-dependent: if a Redis node's clock jumps forward → TTL expires early → lock released prematurely on that node → quorum breaks! (2) No fencing tokens: even with Redlock, a paused client can resume after lock expired and corrupt data! (3) Partially correct: provides probabilistic safety, not guaranteed linearizability. Kleppmann's recommendation: if you need correctness → use consensus-based systems (ZK/etcd). If you need efficiency → single Redis is fine (just accept occasional double-execution).

**Q3: How does ZooKeeper implement distributed locks?**
> Ephemeral sequential znodes! (1) Client creates: /lock/my-lock/seq-000001 (ephemeral + sequential!), (2) Client lists all children of /lock/my-lock/, (3) If MY node has lowest sequence → I hold the lock! (4) If NOT lowest: set WATCH on the node JUST BEFORE mine (not all nodes! — avoids herd effect!), (5) When watched node deleted → re-check if I'm now lowest! (6) If client dies → session dies → ephemeral node deleted → next waiter gets lock! Advantages: no TTL management, no renewal threads, session-based lifecycle, herd effect avoided (each waiter watches only predecessor!).

**Q4: How do you handle the scenario where a lock holder has a long GC pause?**
> This is THE hardest problem in distributed locking! Solutions: (1) **Fencing tokens**: the ONLY complete solution! Lock holder gets token=42. After GC pause, lock expired, new holder gets token=43. Old holder's operations rejected by storage (token 42 < 43). (2) **Short leases + aggressive renewal**: lease=30s, renew every 5s. If renewal fails 3x → assume lock lost, STOP operations! (3) **Cooperative checking**: before each critical operation, client VERIFIES its lock is still valid (`isStillHeld()?`). (4) Accept the trade-off: for some workloads, occasional double-execution is acceptable → use simpler Redis locks + idempotent operations!

---

## 🏆 Mini Challenge

```
🎮 CHALLENGE: Design a "Read-Write Lock" on top of this system!

  Rules:
  • Multiple readers can hold the lock simultaneously
  • Only ONE writer can hold it (exclusive!)
  • Writers have priority over readers (prevent writer starvation!)
  
  Hint: Use TWO lock keys + a counter!
  • "lock:X:writer" — exclusive lock for writers
  • "lock:X:readers" — counter of active readers
  • Writer: acquire "writer" lock THEN wait for readers → 0!
  • Reader: check no "writer" lock → increment reader count!
  
  What about fairness? How to prevent starvation of either side?
```

---

## 🔗 Related Topics
- [Distributed Locking](../Microservices/DistributedLocking.md) — Practical patterns
- [Consistency Trade-offs](../Tradeoffs/Strong_vs_Eventual_Consistency.md) — CP vs AP
- [Design Distributed KV Store](./DesignDistributedKVStore.md) — Similar consensus needs
- [Heartbeats](../Microservices/Heartbeats.md) — Session/lease maintenance

---

*"A distributed lock is a coordination primitive that appears simple but hides profound complexity. The moment you account for network partitions, clock skew, process pauses, and partial failures, you realize why Leslie Lamport won a Turing Award for consensus algorithms. The lock itself is easy — guaranteeing it's correct in the face of everything that can go wrong is one of the hardest problems in computing."* 🔐
