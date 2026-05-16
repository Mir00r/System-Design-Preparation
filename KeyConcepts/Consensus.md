# 🤝 Consensus: How Distributed Nodes Agree on Truth

> *"In a distributed system, the hardest problem isn't performance — it's getting multiple machines to agree on something when any of them can crash at any moment and messages can be lost. Consensus protocols like Raft and Paxos solve this impossible-sounding problem."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Replication](./Replication.md), [CAP Theorem](./CAPTheorem.md), [Fault Tolerance](./FaultTolerance.md)

---

## 📋 Table of Contents
1. [Why Consensus Matters](#-why-consensus-matters)
2. [The Consensus Problem](#-the-consensus-problem)
3. [Raft Protocol (Visual Guide)](#-raft-protocol)
4. [Paxos (Simplified)](#-paxos-simplified)
5. [Leader Election](#-leader-election)
6. [Real-World Implementations](#-real-world-implementations)
7. [Code Example](#-code-example)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 Why Consensus Matters

```
PROBLEMS THAT REQUIRE CONSENSUS:

1. LEADER ELECTION:
   "Which node is the leader right now?"
   If nodes disagree → split brain → data corruption

2. ATOMIC COMMIT:
   "Did this distributed transaction commit or abort?"
   All nodes must agree on the same decision

3. TOTAL ORDER BROADCAST:
   "In what order did these writes happen?"
   All replicas must process writes in the same order

4. DISTRIBUTED LOCK:
   "Who holds the lock right now?"
   Only one node can hold it — all must agree

WITHOUT CONSENSUS:
  Node A thinks: "I am the leader"
  Node B thinks: "I am the leader"
  → Both accept writes → conflicting data → DISASTER

WITH CONSENSUS:
  All nodes agree: "Node A is the leader" (even if some nodes are down)
  → Single source of truth → consistent system
```

---

## 🎯 The Consensus Problem

```
FORMAL DEFINITION:
  Given N nodes that can crash and messages that can be delayed/lost,
  all non-faulty nodes must:
    1. AGREEMENT: decide on the same value
    2. VALIDITY: the decided value was proposed by some node
    3. TERMINATION: all non-faulty nodes eventually decide
    4. INTEGRITY: each node decides at most once

FLP IMPOSSIBILITY (1985):
  In an ASYNCHRONOUS system (no timing assumptions),
  consensus is IMPOSSIBLE if even ONE node can crash.
  
  Implication: All practical consensus protocols rely on partial synchrony
  (timeouts) to detect failures and make progress.

FAULT TOLERANCE:
  With N nodes, tolerate F failures: N ≥ 2F + 1
  - 3 nodes → tolerate 1 failure (majority = 2)
  - 5 nodes → tolerate 2 failures (majority = 3)
  - 7 nodes → tolerate 3 failures (majority = 4)
  
  Why 2F+1? Need a majority that ALWAYS overlaps between any two quorums,
  ensuring at least one node in every majority has the latest state.
```

---

## 🗳️ Raft Protocol

```
RAFT = "Reliable, Replicated, Redundant, And Fault-Tolerant"
Designed to be UNDERSTANDABLE (unlike Paxos)

THREE ROLES:
  Leader:    handles all client requests, replicates to followers
  Follower:  passively receives entries from leader
  Candidate: trying to become leader (during election)

STATE MACHINE:
  [Follower] ──timeout──▶ [Candidate] ──wins election──▶ [Leader]
       ▲                       │                            │
       │                       │ loses election             │ discovers higher term
       │                       ▼                            │
       └───────────────── [Follower] ◀─────────────────────┘

═══════════════════════════════════════════════════════
PHASE 1: LEADER ELECTION
═══════════════════════════════════════════════════════

  Normal operation:
    Leader sends heartbeats every 150ms
    Followers reset their election timeout on each heartbeat

  Leader fails:
    Follower's election timeout fires (randomized: 150-300ms)
    Follower becomes Candidate:
      1. Increments current term (epoch number)
      2. Votes for itself
      3. Sends RequestVote RPC to all other nodes
    
    Other nodes vote:
      - Grant vote if: candidate's log is at least as up-to-date
      - Each node votes for at most ONE candidate per term
    
    Election outcomes:
      a) Wins majority → becomes Leader, starts sending heartbeats
      b) Another node wins → steps down to Follower
      c) Timeout (split vote) → start new election with higher term

  Example (5 nodes, leader crashes):
    T=0:   Leader (Node A) crashes
    T=200: Node C timeout fires first (random 200ms)
    T=200: Node C becomes Candidate, term=2, requests votes
    T=210: Nodes B, D, E grant votes (C has up-to-date log)
    T=210: Node C has 4 votes (including self) → LEADER
    T=210: Node C starts heartbeats, clients redirected to C

═══════════════════════════════════════════════════════
PHASE 2: LOG REPLICATION
═══════════════════════════════════════════════════════

  Client sends write to Leader:
    1. Leader appends entry to its log: [term=2, index=5, cmd="SET x=7"]
    2. Leader sends AppendEntries RPC to all followers
    3. Followers append to their log, acknowledge
    4. When MAJORITY acknowledges → entry is COMMITTED
    5. Leader applies to state machine, responds to client
    6. Leader notifies followers of commit in next heartbeat
    7. Followers apply committed entries to their state machines

  LOG STRUCTURE:
    Index:  1    2    3    4    5    6    7
    Leader: [T1] [T1] [T1] [T2] [T2] [T2] [T2]
    Node B: [T1] [T1] [T1] [T2] [T2] ← catching up
    Node C: [T1] [T1] [T1] [T2] [T2] [T2] ← caught up

  SAFETY GUARANTEE:
    If entry at index I is committed, no other entry can ever
    be committed at index I in any term. (Log matching property)

═══════════════════════════════════════════════════════
PHASE 3: SAFETY & LOG CONSISTENCY
═══════════════════════════════════════════════════════

  Log Matching Property:
    If two logs contain an entry with same index and term,
    ALL preceding entries are identical.
  
  How? Leader includes (prevLogIndex, prevLogTerm) in AppendEntries.
    Follower rejects if doesn't match → Leader retries with earlier entry.
    Eventually finds common prefix → overwrites divergent entries.
```

---

## 📜 Paxos (Simplified)

```
PAXOS — the original consensus protocol (Lamport, 1989)
  Correct but NOTORIOUSLY hard to understand and implement

THREE ROLES:
  Proposer:  proposes a value
  Acceptor:  votes on proposals  
  Learner:   learns the decided value

TWO PHASES:

  PHASE 1 (Prepare):
    Proposer picks unique proposal number N
    Proposer → all Acceptors: "PREPARE(N)"
    Acceptor: if N > any previously seen proposal:
      Promise: "I won't accept proposals < N"
      Return: any previously accepted value (if any)

  PHASE 2 (Accept):
    If Proposer receives majority promises:
      If any Acceptor returned a previously accepted value:
        Proposer MUST propose THAT value (can't change it)
      Else:
        Proposer proposes its own value
      Proposer → Acceptors: "ACCEPT(N, value)"
      Acceptor: if no higher proposal seen, ACCEPT.

  DECIDED when majority of Acceptors accept the same (N, value)

MULTI-PAXOS (practical optimization):
  Run Phase 1 ONCE to elect a stable leader
  Leader skips Phase 1 for subsequent proposals (only Phase 2)
  Effectively becomes Raft-like leader-based consensus
  
RAFT vs PAXOS:
  Raft:  designed for understandability, leader-based from the start
  Paxos: more general, harder to implement correctly
  In practice: most new systems use Raft (etcd, CockroachDB, TiDB)
```

---

## 🏢 Real-World Implementations

```
┌──────────────────────────────────────────────────────────────────┐
│ SYSTEM          │ PROTOCOL  │ WHAT IT DECIDES                    │
├─────────────────┼───────────┼────────────────────────────────────┤
│ etcd            │ Raft      │ Kubernetes cluster state, configs  │
│ ZooKeeper       │ ZAB*      │ Leader election, distributed locks │
│ CockroachDB     │ Raft      │ Transaction ordering, replication  │
│ TiDB (TiKV)    │ Raft      │ Key-value replication              │
│ Consul          │ Raft      │ Service discovery, KV store        │
│ Google Spanner  │ Paxos     │ Global transaction ordering        │
│ Apache Kafka    │ KRaft**   │ Metadata, partition leadership     │
│ MongoDB         │ Raft-like │ Replica set elections              │
│ Redis Sentinel  │ Raft-like │ Master election                    │
└──────────────────────────────────────────────────────────────────┘

* ZAB = Zookeeper Atomic Broadcast (Paxos variant)
** KRaft = Kafka's Raft implementation (replacing ZooKeeper)
```

---

## 💻 Code Example

```java
// Using etcd (Raft-based) for leader election in Spring Boot
// Dependency: io.etcd:jetcd-core

@Service
public class LeaderElectionService {
    private final Client etcdClient;
    private final String serviceName;
    private final String instanceId;
    
    @PostConstruct
    public void startElection() {
        // Create a lease (TTL = 10s, acts as heartbeat)
        long leaseId = etcdClient.getLeaseClient()
            .grant(10).get().getID();
        
        // Keep lease alive (renew every 3s)
        etcdClient.getLeaseClient().keepAlive(leaseId, new StreamObserver<>() {
            @Override public void onNext(LeaseKeepAliveResponse response) { }
            @Override public void onError(Throwable t) { handleLostLeadership(); }
            @Override public void onCompleted() { }
        });
        
        // Try to become leader (atomic put-if-absent)
        ByteSequence key = ByteSequence.from("/election/" + serviceName, UTF_8);
        ByteSequence value = ByteSequence.from(instanceId, UTF_8);
        
        TxnResponse txn = etcdClient.getKVClient().txn()
            .If(new Cmp(key, Cmp.Op.EQUAL, CmpTarget.createRevision(0)))  // key doesn't exist
            .Then(Op.put(key, value, PutOption.builder().withLeaseId(leaseId).build()))
            .Else(Op.get(key, GetOption.DEFAULT))
            .commit().get();
        
        if (txn.isSucceeded()) {
            log.info("I am the leader: {}", instanceId);
            startLeaderTasks();
        } else {
            log.info("Another instance is leader. Watching for changes...");
            watchForLeaderChange(key);
        }
    }
    
    private void watchForLeaderChange(ByteSequence key) {
        // Watch the key — when leader's lease expires, key is deleted
        etcdClient.getWatchClient().watch(key, event -> {
            if (event.getEventType() == EventType.DELETE) {
                log.info("Leader gone! Attempting to become leader...");
                startElection();  // retry
            }
        });
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Using consensus for every operation** — Consensus is expensive (network round-trips to majority). Don't use it for every read/write. Use it for leader election and metadata, then route data operations through the elected leader without consensus per-operation.

2. **Even number of nodes** — 4 nodes tolerates 1 failure (same as 3 nodes, since majority of 4 = 3). Always use odd numbers: 3, 5, or 7 nodes. Even numbers waste resources without improving fault tolerance.

3. **Not handling split-brain during network partition** — A minority partition might still have the old leader. Use lease-based leadership with TTL: if leader can't reach majority within TTL, it must step down. Fencing tokens prevent stale leaders from making changes.

4. **Confusing consensus with replication** — Consensus decides WHAT to replicate (ordering, leadership). Replication is the mechanism of copying data. You need consensus to agree on the order of writes, then replicate that ordered log to all nodes.

---

## 🧩 Mini Challenge

**You have a 5-node Raft cluster. The network partitions into two groups: {A, B} and {C, D, E}. Node A was the leader before the partition. What happens? Can both groups elect a leader?**

<details>
<summary>💡 Click to reveal answer</summary>

**Analysis:**
- Majority of 5 nodes = 3 nodes needed for any decision
- Group {A, B}: 2 nodes — CANNOT form a majority
- Group {C, D, E}: 3 nodes — CAN form a majority

**What happens:**

1. **Group {A, B}** (old leader A):
   - Node A tries to send heartbeats to C, D, E — fails
   - Node A can still send heartbeats to B (B stays follower)
   - Node A CANNOT commit new entries (needs 3 acks, only has 2)
   - Old leader A is effectively "stalled" — accepts client requests but can't commit them
   - Eventually A's clients timeout and find the new leader

2. **Group {C, D, E}** (no leader initially):
   - C, D, E stop receiving heartbeats from A
   - After election timeout, one becomes Candidate (say C)
   - C requests votes from D and E
   - C gets 3 votes (C, D, E) → majority → NEW LEADER
   - C starts accepting and committing writes (has majority)

3. **When partition heals:**
   - A discovers C has higher term → A steps down to Follower
   - A and B's uncommitted entries are overwritten by C's log
   - Cluster converges to C's state

**Key insight**: Only ONE group can elect a leader (the one with majority). The minority partition is frozen — it can't elect a leader or commit writes. This prevents split-brain by design.

</details>

---

## 📝 Interview Q&A

**Q: Explain Raft's leader election in simple terms. Why is it correct?**
> A: A node becomes a candidate when it doesn't hear from a leader (timeout). It increments its term number and asks other nodes to vote. Each node votes for at most one candidate per term (first-come-first-served). A candidate needs a majority to win. **Why correct**: (1) At most one leader per term — because a majority is needed, and two different candidates can't both get a majority (they'd need overlapping voters, but each voter votes once). (2) The leader has the most up-to-date log — voters reject candidates with less up-to-date logs (election restriction). (3) Randomized timeouts prevent infinite split votes.

**Q: Why do distributed systems use odd numbers of nodes (3, 5, 7)?**
> A: Because fault tolerance depends on majority quorum. With N nodes, you tolerate ⌊(N-1)/2⌋ failures. 3 nodes: majority=2, tolerate 1 failure. 4 nodes: majority=3, tolerate 1 failure. Same fault tolerance! But 4 nodes costs more. 5 nodes: majority=3, tolerate 2 failures. 6 nodes: majority=4, tolerate 2 failures. Again, same fault tolerance for higher cost. Odd numbers are the "sweet spot" where adding a node actually improves fault tolerance.

---

## 🔗 What to Read Next

1. **[KeyConcepts/Replication.md](./Replication.md)** — How data is replicated after consensus establishes order
2. **[KeyConcepts/FaultTolerance.md](./FaultTolerance.md)** — Building systems that survive failures
3. **[KeyConcepts/CAPTheorem.md](./CAPTheorem.md)** — The theoretical limits consensus operates within

---

*[← Replication](./Replication.md) | [Back to Index](../INDEX.md) | [Next: Fault Tolerance →](./FaultTolerance.md)*
