# 💻 Design an Online Code Editor (Real-Time Collaboration)

> *"Google Docs proved that real-time collaboration could work for text. But code is harder — it has syntax, indentation rules, compilation, execution, and debugging. Systems like VS Code Live Share, Replit, and CodeSandbox let multiple developers edit the same file simultaneously, see each other's cursors, and even share a running terminal. This design combines Operational Transformation (or CRDTs), WebSocket-based real-time sync, sandboxed code execution, and the fascinating challenge of making distributed text editing feel local."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [WebSockets](../APIs/WebSockets.md), [Design Google Docs](./DesignGoogleDocs.md), [Caching](../BuildingBlocks/Caching.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Real-Time Collaboration (CRDT/OT)](#-real-time-collaboration-crdtot)
4. [Code Execution Engine](#-code-execution-engine)
5. [File System & Project Management](#-file-system--project-management)
6. [Security & Sandboxing](#-security--sandboxing)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Multi-file code editor with syntax highlighting
  • Real-time collaboration (multiple cursors, live edits!)
  • Code execution (Python, Java, JavaScript, Go, C++!)
  • Terminal access (sandboxed!)
  • File tree / project structure
  • Auto-save + version history
  • Language Server Protocol (autocomplete, errors!)
  • Share via link (no account needed to view!)
  • Chat + video call within editor

NON-FUNCTIONAL:
  • Collaboration latency: < 100ms (feels real-time!)
  • Concurrent editors per file: up to 50!
  • Code execution startup: < 2 seconds
  • File save: < 500ms
  • 99.9% availability
  • Zero data loss (every keystroke persisted!)

SCALE:
  • 10M total projects
  • 500K concurrent editing sessions
  • 100K simultaneous code executions
  • 1M WebSocket connections
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ONLINE CODE EDITOR                                        │
│                                                                             │
│  ┌──────────┐     ┌──────────────────────────────────────────┐            │
│  │  Browser │────►│    Load Balancer (L7, WebSocket-aware!)    │            │
│  │  Editor  │◄───│    (sticky sessions for WS connections!)   │            │
│  │  (Monaco)│     └──────────────────┬───────────────────────┘            │
│  └──────────┘                        │                                     │
│                     ┌────────────────┼────────────────────┐               │
│                     ▼                ▼                    ▼                │
│          ┌──────────────┐  ┌──────────────┐    ┌──────────────┐          │
│          │  HTTP API    │  │ WebSocket    │    │  Execution   │          │
│          │  (REST)      │  │ Server       │    │  API         │          │
│          │  • Project   │  │ • Real-time  │    │  • Run code  │          │
│          │    CRUD      │  │   collab     │    │  • Terminal  │          │
│          │  • File ops  │  │ • Cursors    │    │  • Debug     │          │
│          │  • Auth      │  │ • Chat       │    │              │          │
│          └──────────────┘  └──────┬───────┘    └──────┬───────┘          │
│                                    │                    │                  │
│                                    ▼                    ▼                  │
│                         ┌──────────────────┐  ┌──────────────────┐       │
│                         │  Collaboration   │  │  Sandbox Pool    │       │
│                         │  Engine          │  │  (containers!)   │       │
│                         │  • CRDT/OT       │  │  • Isolated exec │       │
│                         │  • Conflict res. │  │  • Resource limit│       │
│                         │  • Op broadcast  │  │  • Auto-cleanup  │       │
│                         └──────────────────┘  └──────────────────┘       │
│                                                                           │
│  DATA LAYER:                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐     │
│  │PostgreSQL│  │  Redis  │  │  S3/Blob│  │  Kafka  │  │  Docker │     │
│  │(metadata │  │(sessions│  │(project │  │(op log) │  │ Registry│     │
│  │ users)   │  │ presence│  │ files)  │  │         │  │         │     │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘     │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Real-Time Collaboration (CRDT/OT)

```
THE COLLABORATION PROBLEM:
  User A types "Hello" at position 0
  User B types "World" at position 0 (SIMULTANEOUSLY!)
  
  Without conflict resolution: "HWeolrlold" (garbage! 💀)
  With OT/CRDT: "HelloWorld" or "WorldHello" (consistent everywhere!)

OPERATIONAL TRANSFORMATION (OT):
  ┌────────────────────────────────────────────────────────────────┐
  │  How Google Docs does it! Central server transforms ops.        │
  │                                                                 │
  │  User A: Insert("H", pos=0) → Insert("e", pos=1) → ...       │
  │  User B: Insert("W", pos=0) → Insert("o", pos=1) → ...       │
  │                                                                 │
  │  Server receives A's op first:                                  │
  │  → Apply Insert("H", 0) to document                            │
  │  → Now document = "H"                                           │
  │                                                                 │
  │  Server receives B's op:                                        │
  │  → B's Insert("W", 0) was based on empty document!             │
  │  → But document now has "H" → TRANSFORM B's op!                │
  │  → New op: Insert("W", 1) (shift position by 1!)              │
  │  → Document = "HW"                                              │
  │                                                                 │
  │  Transform(opA, opB) → (opA', opB') such that:                │
  │  apply(apply(doc, opA), opB') == apply(apply(doc, opB), opA') │
  │                                                                 │
  │  ✅ Well-proven (Google Docs, 15+ years!)                       │
  │  ❌ Requires central server (single point of ordering!)         │
  │  ❌ Complex edge cases in transformation functions!             │
  └────────────────────────────────────────────────────────────────┘

CRDT (Conflict-free Replicated Data Types):
  ┌────────────────────────────────────────────────────────────────┐
  │  Modern alternative! No central server needed!                   │
  │                                                                 │
  │  Each character has a UNIQUE ID that determines its position!   │
  │  Even if operations arrive out of order → same final result!    │
  │                                                                 │
  │  Popular: Yjs, Automerge                                        │
  │                                                                 │
  │  Example (RGA - Replicated Growable Array):                     │
  │  User A inserts 'H' with ID (A, 1) after root                  │
  │  User B inserts 'W' with ID (B, 1) after root                  │
  │  Both arrive → sort by ID → consistent ordering!               │
  │                                                                 │
  │  ✅ Works peer-to-peer (no central server!)                     │
  │  ✅ Mathematically guaranteed convergence!                       │
  │  ✅ Simpler to reason about correctness!                        │
  │  ❌ Higher memory (each char has metadata!)                     │
  │  ❌ Tombstones for deletes (grow-only structure!)               │
  └────────────────────────────────────────────────────────────────┘

COLLABORATION FLOW:
  ┌──────────────────────────────────────────────────────────────────┐
  │  1. User types character → local operation created               │
  │  2. Apply IMMEDIATELY to local editor (no lag!)                  │
  │  3. Send operation via WebSocket to server                       │
  │  4. Server transforms against concurrent ops (OT!)              │
  │  5. Broadcast transformed op to ALL other users                  │
  │  6. Each client applies transformed op to their doc              │
  │  7. All documents converge to same state! ✅                    │
  │                                                                  │
  │  Latency: user sees own edit instantly (optimistic!)             │
  │  Other users see edit in: network RTT + transform time          │
  │  (typically 50-150ms — feels real-time!)                        │
  └──────────────────────────────────────────────────────────────────┘
```

---

## 🏃 Code Execution Engine

```
SANDBOXED EXECUTION:
  Users run arbitrary code → MUST be isolated! (security!)
  
  ┌────────────────────────────────────────────────────────────────┐
  │  EXECUTION ARCHITECTURE:                                        │
  │                                                                 │
  │  User clicks "Run" → Execution API → Sandbox Pool              │
  │                                                                 │
  │  Sandbox Pool: pre-warmed containers for each language!         │
  │  • Python 3.11 container (pre-installed libraries!)             │
  │  • Java 17 container (JDK ready!)                               │
  │  • Node.js 18 container                                         │
  │  • Go 1.21 container                                            │
  │  • C++ container (GCC/Clang!)                                   │
  │                                                                 │
  │  PRE-WARMING: keep 100 idle containers per language!            │
  │  User requests execution → grab pre-warmed container!           │
  │  No cold start! (container already running!)                    │
  └────────────────────────────────────────────────────────────────┘

RESOURCE LIMITS (prevent abuse!):
  ┌────────────────────────────────────────────────────────────────┐
  │  Per execution:                                                 │
  │  • CPU: 1 core max                                              │
  │  • Memory: 256 MB max                                           │
  │  • Disk: 100 MB (ephemeral!)                                   │
  │  • Time: 30 seconds max (kill after!)                           │
  │  • Network: DISABLED! (no outbound requests!)                   │
  │  • Processes: max 50 (no fork bombs!)                           │
  │                                                                 │
  │  Implementation: cgroups + seccomp + namespace isolation!       │
  │  Or: gVisor/Firecracker microVM for stronger isolation!         │
  │                                                                 │
  │  After execution:                                               │
  │  • Destroy container (prevent state leaks!)                     │
  │  • Or reset to clean state (faster, reuse container!)           │
  └────────────────────────────────────────────────────────────────┘

EXECUTION FLOW:
  1. Client sends: {language: "java", code: "...", stdin: "..."}
  2. Server picks pre-warmed Java container from pool
  3. Write code to container filesystem: /tmp/Main.java
  4. Execute: javac Main.java && java Main < stdin
  5. Capture: stdout, stderr, exit code, execution time
  6. Stream output back to client via WebSocket (real-time!)
  7. Kill container after timeout or completion
  8. Return: {stdout, stderr, exitCode, executionTime, memoryUsed}
```

---

## 📁 File System & Project Management

```
PROJECT STORAGE:
  ┌────────────────────────────────────────────────────────────────┐
  │  Each project = virtual filesystem stored in object storage!    │
  │                                                                 │
  │  Storage: S3 (or similar blob storage)                          │
  │  Path: s3://projects/{user_id}/{project_id}/{file_path}        │
  │                                                                 │
  │  Metadata (PostgreSQL):                                         │
  │  projects: {id, owner_id, name, language, visibility}          │
  │  files: {id, project_id, path, size, content_hash, version}   │
  │  collaborators: {project_id, user_id, permission}              │
  │                                                                 │
  │  HOT FILES: cached in Redis during active editing!              │
  │  • User opens file → load from S3 → cache in Redis            │
  │  • All edits applied to Redis copy (fast!)                      │
  │  • Periodic flush to S3 (every 30 seconds + on close!)         │
  │  • CRDT state also in Redis (collaboration metadata!)          │
  └────────────────────────────────────────────────────────────────┘

VERSION HISTORY:
  Every save creates a snapshot!
  • Store diffs (not full files!) — storage efficient!
  • Last 100 versions kept (per file!)
  • Can restore any previous version!
  • Implementation: similar to Git — content-addressable storage!
  
AUTO-SAVE:
  • Client debounces: save 2 seconds after last keystroke
  • Server persists CRDT state → derive document → write to S3
  • Version created every 5 minutes (or on manual save!)
  • Never lose more than 2 seconds of work!
```

---

## 💻 Java Implementation

### Collaboration WebSocket Handler

```java
@Component
@ServerEndpoint("/ws/collab/{projectId}/{fileId}")
public class CollaborationWebSocket {
    
    @Autowired private CollaborationEngine collaborationEngine;
    @Autowired private PresenceService presenceService;
    @Autowired private RedisTemplate<String, String> redis;
    
    private static final Map<String, Set<Session>> rooms = 
        new ConcurrentHashMap<>();
    
    @OnOpen
    public void onOpen(Session session, 
                       @PathParam("projectId") String projectId,
                       @PathParam("fileId") String fileId) {
        String roomId = projectId + ":" + fileId;
        rooms.computeIfAbsent(roomId, k -> ConcurrentHashMap.newKeySet())
            .add(session);
        
        String userId = getUserId(session);
        
        // Announce presence (show cursor to others!)
        presenceService.userJoined(roomId, userId);
        broadcastToRoom(roomId, session, PresenceMessage.joined(userId));
        
        // Send current document state to new user!
        String documentState = collaborationEngine.getDocumentState(roomId);
        session.getAsyncRemote().sendText(
            new InitMessage(documentState, presenceService.getActiveUsers(roomId))
                .toJson());
    }
    
    @OnMessage
    public void onMessage(Session session, String message,
                          @PathParam("projectId") String projectId,
                          @PathParam("fileId") String fileId) {
        String roomId = projectId + ":" + fileId;
        String userId = getUserId(session);
        
        CollabMessage msg = CollabMessage.fromJson(message);
        
        switch (msg.getType()) {
            case "operation":
                // User made an edit! Transform and broadcast!
                Operation op = msg.getOperation();
                Operation transformed = collaborationEngine
                    .transformAndApply(roomId, op, msg.getVersion());
                
                // Acknowledge to sender (with server version!)
                session.getAsyncRemote().sendText(
                    AckMessage.of(transformed.getVersion()).toJson());
                
                // Broadcast to all OTHER users in the room!
                broadcastToRoom(roomId, session, 
                    OperationMessage.of(userId, transformed));
                break;
                
            case "cursor":
                // Cursor position update (show to others!)
                broadcastToRoom(roomId, session, 
                    CursorMessage.of(userId, msg.getCursorPosition()));
                break;
                
            case "selection":
                // Selection highlight (show what user selected!)
                broadcastToRoom(roomId, session,
                    SelectionMessage.of(userId, msg.getSelection()));
                break;
        }
    }
    
    @OnClose
    public void onClose(Session session, 
                        @PathParam("projectId") String projectId,
                        @PathParam("fileId") String fileId) {
        String roomId = projectId + ":" + fileId;
        rooms.getOrDefault(roomId, Set.of()).remove(session);
        
        String userId = getUserId(session);
        presenceService.userLeft(roomId, userId);
        broadcastToRoom(roomId, null, PresenceMessage.left(userId));
    }
    
    private void broadcastToRoom(String roomId, Session exclude, 
                                  Object message) {
        String json = toJson(message);
        rooms.getOrDefault(roomId, Set.of()).stream()
            .filter(s -> s != exclude && s.isOpen())
            .forEach(s -> s.getAsyncRemote().sendText(json));
    }
}
```

### Code Execution Service

```java
@Service
public class CodeExecutionService {
    
    @Autowired private ContainerPoolManager containerPool;
    
    private static final Map<String, ExecutionConfig> LANGUAGE_CONFIGS = Map.of(
        "java", new ExecutionConfig("javac Main.java && java Main", "Main.java"),
        "python", new ExecutionConfig("python3 main.py", "main.py"),
        "javascript", new ExecutionConfig("node main.js", "main.js"),
        "go", new ExecutionConfig("go run main.go", "main.go"),
        "cpp", new ExecutionConfig("g++ -o main main.cpp && ./main", "main.cpp")
    );
    
    /**
     * Execute code in sandboxed container.
     * Streams output back via callback (WebSocket to client!).
     */
    public CompletableFuture<ExecutionResult> executeCode(
            ExecutionRequest request, Consumer<String> outputCallback) {
        
        ExecutionConfig config = LANGUAGE_CONFIGS.get(request.getLanguage());
        if (config == null) {
            throw new UnsupportedLanguageException(request.getLanguage());
        }
        
        return CompletableFuture.supplyAsync(() -> {
            // Grab pre-warmed container from pool!
            Container container = containerPool.acquire(request.getLanguage());
            
            try {
                // Write code to container
                container.writeFile(config.getFileName(), request.getCode());
                
                // Write stdin if provided
                if (request.getStdin() != null) {
                    container.writeFile("input.txt", request.getStdin());
                }
                
                // Execute with resource limits!
                ProcessResult result = container.execute(
                    config.getCommand(),
                    ExecutionLimits.builder()
                        .timeoutMs(30_000)      // 30 seconds!
                        .memoryMb(256)          // 256 MB!
                        .maxProcesses(50)       // No fork bombs!
                        .build(),
                    outputCallback);            // Stream output real-time!
                
                return ExecutionResult.builder()
                    .stdout(result.getStdout())
                    .stderr(result.getStderr())
                    .exitCode(result.getExitCode())
                    .executionTimeMs(result.getDurationMs())
                    .memoryUsedMb(result.getMemoryUsedMb())
                    .timedOut(result.isTimedOut())
                    .build();
                    
            } finally {
                // Return container to pool (reset state!)
                containerPool.release(container);
            }
        });
    }
}
```

---

## ❓ Interview Q&A

**Q1: OT vs CRDT — which would you choose for a code editor?**
> For a HOSTED editor (like Replit): OT with a central server. Why: (1) Server is already there (hosting the files!), (2) Simpler client implementation, (3) Lower memory overhead (CRDT stores metadata per character!), (4) Google Docs proves it works at massive scale. For a P2P editor (like VS Code Live Share): CRDT. Why: (1) No central server between peers, (2) Works offline (merge when reconnected!), (3) Mathematically guaranteed convergence without coordination. In practice: most online editors use OT (simpler for server-hosted model). Yjs (CRDT library) is gaining popularity for its simplicity and correctness guarantees.

**Q2: How do you execute user code safely?**
> Defense in depth: (1) **Container isolation**: each execution in its own container (namespace isolation: can't see host filesystem/processes!), (2) **cgroups**: CPU + memory limits (kill if exceeded!), (3) **seccomp**: restrict system calls (no raw sockets, no mounting filesystems!), (4) **No network**: DROP all outbound traffic (can't mine crypto, can't DDoS!), (5) **Read-only rootfs**: can't modify the execution environment!, (6) **Time limit**: hard kill after 30 seconds (no infinite loops!), (7) **Ephemeral**: container destroyed after execution (no persistent state!). For stronger isolation: use Firecracker microVMs (separate kernel per execution! — what AWS Lambda uses).

**Q3: How do you handle 500K concurrent WebSocket connections?**
> (1) Horizontal scaling: each WS server handles ~50K connections. Need 10+ servers! (2) Sticky sessions: user's WS connection stays on same server (consistent routing by session ID via LB!). (3) Room-level pub/sub: if collaborators are on different WS servers: use Redis Pub/Sub to relay operations between servers! User A on Server 1 → Redis → Server 2 → User B. (4) Connection multiplexing: one WS connection per user (not per file!) — multiplex project+file channels on single socket. (5) Heartbeats: detect dead connections quickly (ping every 30s), clean up stale sessions.

**Q4: How do you ensure zero data loss?**
> Multiple persistence layers: (1) Client-side: operations stored in IndexedDB (survives tab crash!), (2) Server-side: every operation appended to Kafka (immutable operation log!), (3) Periodic snapshots: full document state saved to S3 every 30 seconds (point-in-time recovery!), (4) On disconnect: client replays any unacknowledged operations when reconnecting. Recovery: if server crashes: replay operations from Kafka to rebuild document state. If S3 snapshot exists: load snapshot + replay operations AFTER snapshot timestamp. Document state = snapshot + ops = guaranteed correct!

---

## 🔗 Related Topics
- [Design Google Docs](./DesignGoogleDocs.md) — Similar collaboration model
- [WebSockets](../APIs/WebSockets.md) — Real-time communication
- [Long Polling vs WebSockets](../Tradeoffs/Long_Polling_vs_WebSockets.md)
- [Design Zoom](./DesignZoom.md) — Real-time WebRTC for video in editor

---

*"The online code editor is a masterclass in real-time systems. Every keystroke must be captured, transformed, broadcast, and persisted — all in under 100ms. It combines the hardest problems in distributed systems (consensus on document state), security (executing arbitrary code!), and UX (making collaboration feel seamless). Build this, and you've proven mastery of real-time distributed architecture."* 💻
