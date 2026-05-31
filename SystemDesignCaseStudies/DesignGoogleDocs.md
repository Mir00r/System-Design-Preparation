# 📝 Design Google Docs: Real-time Collaborative Editing

> *"Google Docs supports 100 users simultaneously editing the same document with zero conflicts and sub-100ms latency. When you type a character, it appears on everyone else's screen within 200ms — even if they're typing at the same position at the same time. This is achieved through Operational Transformation (OT), the same algorithm that powered Google Wave before it was refined for Docs. It's one of the hardest real-time systems to build correctly."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [WebSockets](../../APIs/WebSockets.md), [Conflict Resolution](../../KeyConcepts/CAPTheorem.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Scale Estimation](#-scale-estimation)
3. [High-Level Architecture](#-high-level-architecture)
4. [The Core Problem: Concurrent Editing](#-the-core-problem-concurrent-editing)
5. [Operational Transformation (OT)](#-operational-transformation-ot)
6. [CRDTs: The Alternative](#-crdts-the-alternative)
7. [Real-time Sync Architecture](#-real-time-sync-architecture)
8. [Document Storage](#-document-storage)
9. [Presence & Cursors](#-presence--cursors)
10. [Offline Support](#-offline-support)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Create, edit, and delete documents
✅ Real-time collaborative editing (multiple users, same document)
✅ Conflict resolution (concurrent edits at same position)
✅ Rich text formatting (bold, italic, lists, tables, images)
✅ Comments and suggestions
✅ Version history (view/restore any previous version)
✅ Share with permissions (viewer, commenter, editor)
✅ Offline editing with sync on reconnect
```

### Non-Functional Requirements
```
✅ Real-time: character appears on other screens in < 200ms
✅ Consistency: all users see SAME document eventually
✅ Support 100 concurrent editors per document
✅ Support billions of documents total
✅ No data loss (every keystroke persisted)
✅ Handle network partitions gracefully (offline mode)
```

---

## 📊 Scale Estimation

```
USERS:
  • 1B total users, 300M monthly active
  • 50M documents edited daily
  • Average 2 collaborators per document
  • Peak: 10M concurrent editing sessions
  • Large docs: up to 100 concurrent editors

OPERATIONS:
  • Average typing speed: 5 characters/second per user
  • 10M sessions × 5 ops/sec = 50M operations/second at peak!
  • Each operation: ~100 bytes (type, position, character, user_id)
  • Daily operations: 50M × 86400 × 100 bytes = ~430 TB/day (!!!)
  • Actually: not all users type constantly, realistic = ~5M ops/sec peak

STORAGE:
  • 5B total documents × average 50KB = 250 TB current content
  • Version history: ~10x current = 2.5 PB
  • Operations log (for undo/history): retained 30 days

BANDWIDTH:
  • Document with 50 editors: 50 × 5 ops/sec × 100 bytes = 25 KB/sec
  • Broadcast to 49 others: 25 KB × 49 = ~1.2 MB/sec per document
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                   │
│    💻 Web Editor    📱 Mobile    📱 Offline Mode                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │ WebSocket
                    ┌──────────▼──────────┐
                    │  WebSocket Gateway  │  (sticky sessions by doc_id)
                    └──────────┬──────────┘
                               │
         ┌─────────────────────┼──────────────────────────┐
         │                     │                          │
         ▼                     ▼                          ▼
┌──────────────┐    ┌──────────────────┐      ┌──────────────────┐
│  Document    │    │  Collaboration   │      │  Presence        │
│  Service     │    │  Service (OT)    │      │  Service         │
│  (CRUD)      │    │  (transforms!)   │      │  (cursors)       │
└──────────────┘    └────────┬─────────┘      └──────────────────┘
                             │
         ┌───────────────────┼───────────────────────┐
         │                   │                       │
         ▼                   ▼                       ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Permission  │    │  Version History │    │  Notification    │
│  Service     │    │  Service         │    │  Service         │
└──────────────┘    └──────────────────┘    └──────────────────┘

DATA LAYER:
┌──────────────────────────────────────────────────────────────────┐
│  Cloud Spanner/PostgreSQL — Document metadata + permissions     │
│  Object Storage (GCS/S3) — Document content snapshots          │
│  Operation Log (Kafka) — All editing operations (event source) │
│  Redis — Presence, active sessions, operation buffers          │
│  Elasticsearch — Document search                               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 💥 The Core Problem: Concurrent Editing

```
THE PROBLEM:

  Document: "Hello World"
  
  User A (position 5): Insert "," → "Hello, World"
  User B (position 6): Delete "W" → "Hello orld"
  
  Both happen "simultaneously" (before seeing each other's change).
  
  If applied naively:
    Start:        "Hello World"
    Apply A:      "Hello, World"
    Apply B at 6: "Hello,World"  ← WRONG! B wanted to delete "W" but 
                                    comma shifted it!
    
  User B intended: "Hello orld" → but position shifted!
  
  WITHOUT CONFLICT RESOLUTION:
    User A sees: "Hello, orld"
    User B sees: "Hello,World"
    DIVERGED! 💀 Different documents on different screens!

THE SOLUTION: Transform operations based on concurrent changes!
```

---

## 🔄 Operational Transformation (OT)

```
OT = Transforming operations so they STILL produce the intended 
     effect even when other operations happened concurrently.

CORE IDEA:
  When you receive a remote operation, TRANSFORM it against all
  operations that have been applied locally since the remote op
  was generated.

EXAMPLE:
  Document: "Hello World" (both users see this)
  
  User A: Insert(",", 5)  → at position 5, insert comma
  User B: Delete(6, 1)    → at position 6, delete 1 char ("W")
  
  Server receives A first → applies → "Hello, World"
  Server receives B (generated against old state) → TRANSFORM!
  
  Transform(Delete(6,1), Insert(",",5)):
    "Insert before position 6 shifted everything right by 1"
    → Delete(7, 1)  ← adjusted position!
    
  Apply transformed B → "Hello, orld" ✅
  Both users converge to same result!

TRANSFORMATION RULES:
  ┌──────────────────────────────────────────────────────────────┐
  │  T(Insert(p1, c1), Insert(p2, c2)):                          │
  │    if p1 < p2: Insert remains at p1                          │
  │    if p1 >= p2: Insert shifts to p1 + len(c2)                │
  │                                                              │
  │  T(Delete(p1, n1), Insert(p2, c2)):                          │
  │    if p1 < p2: Delete remains at p1                          │
  │    if p1 >= p2: Delete shifts to p1 + len(c2)                │
  │                                                              │
  │  T(Insert(p1, c1), Delete(p2, n2)):                          │
  │    if p1 <= p2: Insert remains at p1                         │
  │    if p1 > p2: Insert shifts to p1 - n2                      │
  └──────────────────────────────────────────────────────────────┘
```

### OT Architecture

```
  ┌────────────┐       ┌───────────────────────┐       ┌────────────┐
  │  Client A  │       │   Server              │       │  Client B  │
  │  (editor)  │       │   (single source of   │       │  (editor)  │
  └─────┬──────┘       │    truth for doc)      │       └─────┬──────┘
        │              └──────────┬────────────┘              │
        │  op: Insert(",",5)     │                            │
        │───────────────────────►│                            │
        │                        │  Accept A's op             │
        │                        │  (apply directly)          │
        │                        │                            │
        │                        │◄───────────────────────────│
        │                        │  op: Delete(6,1)           │
        │                        │                            │
        │                        │  Transform B against A:    │
        │                        │  Delete(6,1) → Delete(7,1) │
        │                        │  Apply transformed B       │
        │                        │                            │
        │◄───────────────────────│  Broadcast: Delete(7,1)    │
        │                        │───────────────────────────►│
        │  Transform against     │  Send: ACK + transformed   │
        │  my pending ops        │                            │
        │  (if any)              │                            │
  
  SERVER is the single sequencer — assigns total order to operations!
  Clients must acknowledge server's canonical ordering.
```

---

## 🧬 CRDTs: The Alternative

```
CRDT = Conflict-free Replicated Data Type
  Mathematical data structures that AUTOMATICALLY merge without conflicts!
  No central server needed for conflict resolution!

USED BY: Figma, Apple Notes, Redis (CRDTs for counters)

HOW IT WORKS (simplified for text):
  Each character has a unique ID: (user_id, sequence_number)
  Characters are ordered by their IDs (not positions!)
  
  Insert between 'H' and 'e': new char gets ID between them
  No matter what order operations arrive, result is the same!
  
  User A inserts 'X' between positions (1,0) and (2,0):
    X gets ID (1.5, A)  ← fractional position!
    
  User B inserts 'Y' between same positions:
    Y gets ID (1.5, B)  ← same fraction, different user!
    
  Tiebreaker: sort by user ID → deterministic merge!

OT vs CRDT:
  ┌──────────────┬─────────────────────┬──────────────────────┐
  │              │  OT                 │  CRDT                │
  ├──────────────┼─────────────────────┼──────────────────────┤
  │  Server      │  Required (central) │  Not required (P2P!) │
  │  Complexity  │  Transform functions│  Data structure      │
  │  Latency     │  Server roundtrip   │  Instant local apply │
  │  Memory      │  Low                │  Higher (tombstones) │
  │  Correctness │  Hard to prove      │  Mathematically proven│
  │  Used by     │  Google Docs        │  Figma, Yjs          │
  └──────────────┴─────────────────────┴──────────────────────┘
```

---

## 🔌 Real-time Sync Architecture

```
WEBSOCKET CONNECTION LIFECYCLE:

1. User opens document:
   → WebSocket connection to Collaboration Service
   → Server loads latest document snapshot
   → Server sends full document to client
   → Client renders document

2. User types:
   → Operation created locally (applied immediately to local state!)
   → Operation sent to server via WebSocket
   → Server transforms against any concurrent ops
   → Server applies and broadcasts to all other clients
   → Other clients receive, transform against pending local ops, apply

3. Acknowledgment:
   → Server ACKs client's operation (confirms it was applied)
   → Client can discard from "pending" buffer

SESSION MANAGEMENT:
  ┌─────────────────────────────────────────────────────────┐
  │  Document "doc_123" Session:                            │
  │                                                         │
  │  Server state:                                          │
  │    current_version: 1547                                │
  │    connected_users: [Alice, Bob, Charlie]               │
  │    operation_log: [op_1543, op_1544, ..., op_1547]     │
  │                                                         │
  │  Per-client state:                                      │
  │    Alice: last_acked_version = 1547 (fully synced)     │
  │    Bob:   last_acked_version = 1545 (2 ops behind)     │
  │    Charlie: last_acked_version = 1547 (fully synced)   │
  └─────────────────────────────────────────────────────────┘
```

---

## 💾 Document Storage

```
STORAGE STRATEGY:

  Don't store the document as one big blob!
  Store as: Base snapshot + Operation log
  
  Document state = snapshot(v1000) + ops[1001..1547]
  
  Periodically (every 100 ops): create new snapshot
  snapshot(v1100) = apply(snapshot(v1000), ops[1001..1100])
  
  ┌────────────────────────────────────────────────────────┐
  │  Object Storage (S3/GCS):                              │
  │    /docs/doc_123/snapshot_v1000.json  (50KB)           │
  │    /docs/doc_123/snapshot_v1100.json  (52KB)           │
  │    /docs/doc_123/snapshot_v1200.json  (55KB)           │
  │                                                        │
  │  Operation Log (Kafka/Spanner):                        │
  │    doc_123: [op_1, op_2, ..., op_1547]                │
  │    Each op: {user, type, position, content, version}   │
  │                                                        │
  │  Version History:                                      │
  │    Snapshots at meaningful points (user-triggered save)│
  │    Named versions: "v1 - Initial draft"                │
  └────────────────────────────────────────────────────────┘

LOADING A DOCUMENT:
  1. Fetch latest snapshot (fast! one read)
  2. Fetch ops since snapshot (small, sequential)
  3. Apply ops to snapshot → current state
  4. Send to client, start accepting real-time ops
```

---

## 👥 Presence & Cursors

```
PRESENCE = Showing who else is viewing/editing the document.
  
  ┌─────────────────────────────────────────────────┐
  │  "Hello Wor|ld"                                 │
  │            ↑                                    │
  │         Alice's cursor (blue)                   │
  │                                                 │
  │  "Hello World|"                                 │
  │              ↑                                  │
  │           Bob's cursor (green)                  │
  └─────────────────────────────────────────────────┘

IMPLEMENTATION:
  • Cursor position = operation (sent via same WebSocket)
  • Broadcast cursor updates to all clients (throttled: max 10/sec)
  • Selection highlights: (start, end, user_id, color)
  
  Redis Pub/Sub for presence:
    Channel: "doc:doc_123:presence"
    Message: {user: "Alice", cursor: 42, selection: [42, 50]}
    
  When user disconnects: remove from presence list (TTL 10s)
```

---

## 📴 Offline Support

```
OFFLINE EDITING:

  1. User goes offline:
     → Client continues accepting local edits
     → Operations queued in local IndexedDB
     → UI shows "Offline" indicator
     
  2. User comes back online:
     → Client sends ALL queued operations to server
     → Server transforms each against operations that happened
       while user was offline
     → Server sends back ALL operations user missed
     → Client transforms missed ops against local pending ops
     → Documents converge! ✅

CONFLICT EXAMPLE:
  User A (offline): Changes "Hello" to "Hi" at position 0
  User B (online): Changes "Hello" to "Hello World" (appends)
  
  When A reconnects:
    Server has: "Hello World" (B's change applied)
    A's ops: Delete(0,5) + Insert(0,"Hi")
    Transform A's ops against B's: positions still valid!
    Result: "Hi World" ← Both changes preserved!
```

---

## 🎮 Mini Challenge

### 🧩 Design: Conflict Resolution for Concurrent Formatting

Two users simultaneously:
- User A: Bold characters 5-10 ("Hello **World** today")
- User B: Delete characters 7-12 ("Hello Wday")

How does OT handle this?

<details>
<summary>🔑 Answer</summary>

**Operations generated:**
- A: Format(5, 10, bold=true)
- B: Delete(7, 5) — deletes "orld "

**Server receives A first:**
- Apply A: Characters 5-10 are now bold. Document: "Hello **World** today"

**Transform B against A:**
- B's Delete(7, 5): Does the format change affect positions? NO (format doesn't shift positions)
- B still deletes at position 7-12: "Hello **Wo**day"

**Result:** Characters 5-6 ("Wo") are bold, "rld t" is deleted.

**Key insight:** Format operations don't shift positions (they're metadata on existing chars). But if B's delete overlaps with A's format range, the format applies only to surviving characters!

**Final document:** "Hello **Wo**day" — formatting preserved on characters that still exist.
</details>

---

## ❓ Interview Q&A

**Q1: How does Google Docs handle concurrent editing?**
> Uses Operational Transformation (OT). A central server maintains the canonical document state and assigns a total order to all operations. When concurrent operations arrive, the server transforms later operations against earlier ones so they produce the intended effect at the correct position. All clients converge to the same state.

**Q2: What's the difference between OT and CRDTs for collaborative editing?**
> OT requires a central server to order and transform operations — simpler data model but needs server roundtrip. CRDTs are mathematical data structures that merge automatically without conflicts — can work P2P (no server needed) but use more memory (need unique IDs per character, tombstones for deletes). Google Docs uses OT; Figma uses CRDTs.

**Q3: How do you handle a user going offline and coming back?**
> While offline: queue operations locally (IndexedDB). On reconnect: send all queued ops to server. Server transforms them against all ops that happened during offline period. Server sends back missed operations. Client transforms missed ops against any local pending ops. Both documents converge. The math guarantees convergence regardless of network timing.

**Q4: How do you store documents efficiently?**
> Don't store full document on every keystroke! Use: periodic snapshots (every 100 ops) stored in object storage + operation log stored in append-only log (Kafka). To load: fetch latest snapshot + apply recent ops. For version history: named snapshots at meaningful points. For undo: reverse operations from the log.

**Q5: How do you scale to millions of concurrent documents?**
> Each document session lives on one server (the "document home"). Route by document_id (consistent hashing). Most documents have 1-5 editors (small). Hot documents (100 editors): single server handles transformation. Stateless services for auth, permissions, search. Document homes are stateful but handle one doc each. Scale by adding more document home servers.

---

## 🔗 Related Topics
- [WebSockets](../../APIs/WebSockets.md) — Real-time communication
- [CAP Theorem](../../KeyConcepts/CAPTheorem.md) — Consistency in collaborative systems
- [Event Sourcing](../../Architectures/Event_Driven.md) — Operation log as event source
- [Distributed Locking](../../Microservices/DistributedLocking.md) — Not used here! (OT avoids locks)

---

*"The most impressive thing about Google Docs isn't that 100 people can edit simultaneously — it's that they never even notice the system resolving conflicts behind the scenes." — Collaborative Systems Engineering* 📝
