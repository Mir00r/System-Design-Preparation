# ☁️ Design Distributed Cloud Storage (Google Drive/Dropbox)

> *"Dropbox started as a simple idea: keep files in sync across all your devices. But behind that simplicity lies one of the hardest distributed systems problems — syncing files across millions of devices with conflict resolution, deduplication that saves petabytes of storage, chunked uploads that handle unreliable networks, and delta sync that only transfers the bytes that changed. This design covers content-addressable storage, Merkle trees for sync detection, and the fascinating block-level deduplication that makes cloud storage economically viable."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Blob Storage](../BuildingBlocks/Blob_Storage.md), [Checksums](../Foundations/Networking/Checksums.md), [Consistency](../Tradeoffs/Strong_vs_Eventual_Consistency.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Chunking & Deduplication](#-chunking--deduplication)
4. [Sync Protocol](#-sync-protocol)
5. [Conflict Resolution](#-conflict-resolution)
6. [Metadata & File Versioning](#-metadata--file-versioning)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Upload/download files (any type, up to 50 GB!)
  • Automatic sync across devices (phone, laptop, tablet!)
  • File sharing (links, permissions: view/edit!)
  • Folder structure (create, move, rename!)
  • Version history (restore any previous version!)
  • Conflict handling (same file edited on 2 devices offline!)
  • Real-time collaboration on documents
  • Selective sync (choose which folders to sync locally!)
  • Trash/recycle bin (30-day recovery!)

NON-FUNCTIONAL:
  • 500M users, 100M daily active
  • Average user: 5 GB storage, 100 files synced/day
  • Upload speed: maximize user's bandwidth!
  • Sync latency: < 5 seconds after save
  • Consistency: eventual (within 10s across devices!)
  • Availability: 99.99% (data NEVER lost!)
  • Durability: 99.999999999% (11 nines!)

SCALE:
  • 1 exabyte total storage (10^18 bytes!)
  • 10B files stored total
  • 1B file operations per day
  • 50M concurrent connected clients
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED CLOUD STORAGE                                 │
│                                                                             │
│  ┌──────────────────┐         ┌───────────────────────────────────┐       │
│  │  Desktop Client  │◄───────►│      Sync Service (WebSocket!)     │       │
│  │  • File watcher  │   sync  │  • Detect changes                  │       │
│  │  • Chunker       │         │  • Resolve conflicts               │       │
│  │  • Local DB      │         │  • Coordinate upload/download      │       │
│  └──────────────────┘         └────────────────┬──────────────────┘       │
│                                                 │                           │
│  ┌──────────────────┐         ┌────────────────┼──────────────────┐       │
│  │  Mobile Client   │◄───────►│                │                   │       │
│  └──────────────────┘         │                ▼                   │       │
│                                │  ┌──────────────────────────────┐ │       │
│  ┌──────────────────┐         │  │     Metadata Service          │ │       │
│  │  Web Client      │◄───────►│  │  • File tree (path → blocks!)│ │       │
│  └──────────────────┘         │  │  • Permissions                │ │       │
│                                │  │  • Version history            │ │       │
│                                │  └──────────────────────────────┘ │       │
│                                │                                    │       │
│                                │  ┌──────────────────────────────┐ │       │
│                                │  │     Block Service             │ │       │
│                                │  │  • Upload/download chunks     │ │       │
│                                │  │  • Deduplication!             │ │       │
│                                │  │  • Content-addressable store  │ │       │
│                                │  └──────────────┬───────────────┘ │       │
│                                └─────────────────┼─────────────────┘       │
│                                                  │                          │
│                                                  ▼                          │
│                               ┌──────────────────────────────────┐         │
│                               │       STORAGE LAYER               │         │
│                               │  ┌──────┐ ┌──────┐ ┌──────────┐ │         │
│                               │  │  S3  │ │  S3  │ │  S3      │ │         │
│                               │  │US-E1 │ │EU-W1 │ │AP-SE1   │ │         │
│                               │  └──────┘ └──────┘ └──────────┘ │         │
│                               │  (3 replicas across regions!)     │         │
│                               └──────────────────────────────────┘         │
│                                                                             │
│  SUPPORTING SERVICES:                                                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│  │Notification│ │  Sharing │  │  Quota   │  │  Search  │                 │
│  │  Service  │  │  Service │  │  Service │  │  Service │                 │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                 │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 🧱 Chunking & Deduplication

```
WHY CHUNK FILES? (The key insight!):

  Without chunking:
  • 1 GB file modified → re-upload ENTIRE 1 GB! 💀
  • Storage: 2 copies of 1 GB file = 2 GB stored!
  
  With chunking (4 MB blocks!):
  • 1 GB file = 256 blocks
  • Modify 1 byte → only 1 block changes → upload 4 MB! (0.4%!)
  • 2 copies of same file = 256 blocks stored ONCE! (dedup!)

CONTENT-DEFINED CHUNKING (variable-size!):
  ┌────────────────────────────────────────────────────────────────┐
  │  Fixed-size chunks (simple but BAD!):                           │
  │  Insert 1 byte at beginning → ALL chunks shift → all re-upload!│
  │                                                                 │
  │  Content-defined chunking (Rabin fingerprinting!):              │
  │  Chunk boundaries determined by FILE CONTENT, not position!     │
  │                                                                 │
  │  How: slide a window across file, compute rolling hash.         │
  │  When hash % TARGET_SIZE == MAGIC_VALUE → chunk boundary!       │
  │                                                                 │
  │  Insert 1 byte → only 1-2 chunks change! (boundaries stable!)  │
  │                                                                 │
  │  Chunk sizes: min 1 MB, target 4 MB, max 8 MB                  │
  │  (prevents tiny chunks from too-frequent boundaries!)           │
  └────────────────────────────────────────────────────────────────┘

BLOCK-LEVEL DEDUPLICATION:
  ┌────────────────────────────────────────────────────────────────┐
  │  Each block identified by its SHA-256 hash!                     │
  │                                                                 │
  │  File "report.pdf" → chunks → SHA-256 each chunk:             │
  │  Block 1: sha256 = "a3f8b2c1..." → stored in S3 at this key! │
  │  Block 2: sha256 = "7d2e4f91..." → stored in S3 at this key! │
  │  Block 3: sha256 = "a3f8b2c1..." → SAME as Block 1! NOT stored!│
  │                                                                 │
  │  File metadata: ["a3f8b2c1", "7d2e4f91", "a3f8b2c1"]          │
  │  (ordered list of block hashes = the file!)                     │
  │                                                                 │
  │  GLOBAL dedup: if ANY user uploads the same block → reuse!     │
  │  • Email attachment sent to 1000 people? Stored ONCE!           │
  │  • Software installer (100K users)? Stored ONCE!                │
  │  • Typical dedup ratio: 50-70% savings! 🚀                     │
  │                                                                 │
  │  Upload flow:                                                    │
  │  1. Client computes SHA-256 of each chunk                       │
  │  2. Sends list of hashes to server: "do you have these?"        │
  │  3. Server checks block store: "I have blocks 1 and 3!"        │
  │  4. Client uploads ONLY block 2! (saves 66% bandwidth!)        │
  └────────────────────────────────────────────────────────────────┘
```

---

## 🔄 Sync Protocol

```
SYNC DETECTION (how to know what changed?):

CLIENT-SIDE:
  • File system watcher (inotify on Linux, FSEvents on macOS!)
  • Detects: create, modify, delete, rename, move
  • On change → compute new block hashes → compare with last sync

SERVER-SIDE:
  • Maintain "journal" of all changes (append-only log!)
  • Each entry: {path, operation, version, timestamp, block_list}
  • Client polls: "give me changes since version 42"
  • Or: WebSocket push (real-time notification!)

SYNC PROTOCOL:
  ┌────────────────────────────────────────────────────────────────┐
  │  CLIENT SYNC LOOP (runs continuously!):                         │
  │                                                                 │
  │  1. DETECT local changes (file watcher!)                        │
  │     Changed files → chunk → compute hashes                     │
  │                                                                 │
  │  2. UPLOAD new/changed blocks                                   │
  │     Send only blocks that server doesn't have!                  │
  │     (ask server: "do you have hash X?" → skip if yes!)        │
  │                                                                 │
  │  3. UPDATE metadata on server                                   │
  │     "File /docs/report.pdf is now [hash1, hash2, hash3]"      │
  │     Server assigns new version number!                          │
  │                                                                 │
  │  4. PULL remote changes                                         │
  │     "What changed since my last sync (version 42)?"            │
  │     Server returns: list of changed files + their block lists   │
  │                                                                 │
  │  5. DOWNLOAD missing blocks                                     │
  │     For each changed file: check which blocks I don't have     │
  │     Download only those blocks!                                  │
  │                                                                 │
  │  6. APPLY changes to local filesystem                           │
  │     Reconstruct files from blocks!                              │
  │     Update local database with new state!                       │
  └────────────────────────────────────────────────────────────────┘

DELTA SYNC (rsync-style optimization!):
  For small changes to large files:
  • Client: "block 3 changed, here's a binary diff!"
  • Server: applies diff to existing block → new block!
  • Saves bandwidth: 1 KB diff instead of 4 MB block!
  
  Only for blocks that are "almost the same" (< 10% changed!)
  For larger changes: upload full new block (simpler!)
```

---

## ⚔️ Conflict Resolution

```
CONFLICT: same file edited on 2 devices while offline!

  Device A (airplane!): edits report.pdf, saves locally
  Device B (office!): edits report.pdf, syncs to server
  Device A lands, connects to wifi → tries to sync!
  
  CONFLICT DETECTED! (A's base version ≠ server's current version!)

RESOLUTION STRATEGIES:
  ┌────────────────────────────────────────────────────────────────┐
  │  1. LAST WRITER WINS (simple, but loses data!)                 │
  │     Whoever syncs last overwrites the other!                    │
  │     ❌ Data loss! Only for non-critical files!                 │
  │                                                                 │
  │  2. KEEP BOTH (Dropbox approach!):                             │
  │     Original: report.pdf (server version wins!)                │
  │     Conflict: report (User's conflicted copy 2024-01-15).pdf  │
  │     User manually resolves! (merge by hand!)                    │
  │     ✅ No data loss! ❌ Confusing for users!                   │
  │                                                                 │
  │  3. AUTOMATIC MERGE (for supported file types!):               │
  │     Text files: 3-way merge (like Git!)                        │
  │     If changes don't overlap → auto-merge! ✅                  │
  │     If changes overlap → create conflict markers!              │
  │     Only works for text-based files (not binary!)              │
  │                                                                 │
  │  4. OPERATIONAL TRANSFORM (for real-time collab!):            │
  │     Like Google Docs: transform concurrent edits!              │
  │     No conflicts possible! (edits are composable!)             │
  │     Only for supported formats (docs, sheets!)                 │
  └────────────────────────────────────────────────────────────────┘

DROPBOX'S ACTUAL APPROACH:
  • Binary files (PDF, images): keep both copies!
  • Text files: attempt 3-way merge, fallback to keep both!
  • Google Docs-like files: OT (real-time, no conflicts!)
  • Folder conflicts (both create same-name folder): merge contents!
```

---

## 💻 Java Implementation

### Block Service (Chunking + Dedup)

```java
@Service
public class BlockService {
    
    @Autowired private S3Client s3;
    @Autowired private BlockMetadataRepository blockMetaRepo;
    
    private static final int TARGET_CHUNK_SIZE = 4 * 1024 * 1024; // 4 MB
    private static final int MIN_CHUNK_SIZE = 1 * 1024 * 1024;     // 1 MB
    private static final int MAX_CHUNK_SIZE = 8 * 1024 * 1024;     // 8 MB
    
    /**
     * Chunk a file and upload only NEW blocks (deduplication!).
     * Returns ordered list of block hashes representing the file.
     */
    public List<String> uploadFile(InputStream fileStream, String userId) {
        List<BlockInfo> blocks = chunkFile(fileStream);
        List<String> blockHashes = new ArrayList<>();
        
        for (BlockInfo block : blocks) {
            String hash = computeSHA256(block.getData());
            blockHashes.add(hash);
            
            // DEDUP CHECK: does this block already exist?
            if (!blockExists(hash)) {
                // New block! Upload to S3!
                uploadBlock(hash, block.getData());
                blockMetaRepo.save(new BlockMetadata(hash, block.getSize(),
                    Instant.now(), 1)); // refCount = 1
            } else {
                // Block already exists! Just increment reference count!
                blockMetaRepo.incrementRefCount(hash);
            }
        }
        
        return blockHashes;
    }
    
    /**
     * Content-defined chunking using Rabin fingerprint!
     * Chunk boundaries are determined by content, not position!
     */
    private List<BlockInfo> chunkFile(InputStream stream) {
        List<BlockInfo> chunks = new ArrayList<>();
        RabinFingerprint rabin = new RabinFingerprint(WINDOW_SIZE);
        ByteArrayOutputStream currentChunk = new ByteArrayOutputStream();
        
        int b;
        while ((b = stream.read()) != -1) {
            currentChunk.write(b);
            rabin.pushByte((byte) b);
            
            int chunkSize = currentChunk.size();
            
            // Check if we hit a boundary!
            boolean isBoundary = (rabin.getFingerprint() % TARGET_CHUNK_SIZE == 1);
            boolean isMinSize = chunkSize >= MIN_CHUNK_SIZE;
            boolean isMaxSize = chunkSize >= MAX_CHUNK_SIZE;
            
            if ((isBoundary && isMinSize) || isMaxSize) {
                // Chunk boundary found!
                chunks.add(new BlockInfo(currentChunk.toByteArray()));
                currentChunk = new ByteArrayOutputStream();
                rabin.reset();
            }
        }
        
        // Last chunk (may be smaller than min!)
        if (currentChunk.size() > 0) {
            chunks.add(new BlockInfo(currentChunk.toByteArray()));
        }
        
        return chunks;
    }
    
    /**
     * Check which blocks from a list the server already has.
     * Client sends hashes → server returns "missing" list!
     */
    public List<String> findMissingBlocks(List<String> hashes) {
        Set<String> existing = blockMetaRepo.findExistingHashes(hashes);
        return hashes.stream()
            .filter(h -> !existing.contains(h))
            .collect(Collectors.toList());
    }
    
    /**
     * Download file by reconstructing from blocks!
     */
    public InputStream downloadFile(List<String> blockHashes) {
        // Stream blocks in order → reconstruct file!
        return new SequenceInputStream(
            Collections.enumeration(
                blockHashes.stream()
                    .map(this::downloadBlock)
                    .collect(Collectors.toList())));
    }
}
```

### Sync Service

```java
@Service
public class SyncService {
    
    @Autowired private FileMetadataRepository metadataRepo;
    @Autowired private BlockService blockService;
    @Autowired private SyncJournalRepository journalRepo;
    @Autowired private NotificationService notificationService;
    
    /**
     * Client reports local changes → server processes sync!
     */
    @Transactional
    public SyncResponse processClientChanges(String userId, String deviceId,
                                              List<FileChange> changes) {
        SyncResponse response = new SyncResponse();
        
        for (FileChange change : changes) {
            switch (change.getOperation()) {
                case CREATE:
                case MODIFY:
                    // Client uploaded blocks already, now update metadata!
                    FileMetadata meta = FileMetadata.builder()
                        .userId(userId)
                        .path(change.getPath())
                        .blockHashes(change.getBlockHashes())
                        .size(change.getFileSize())
                        .modifiedAt(change.getModifiedAt())
                        .version(getNextVersion(userId, change.getPath()))
                        .build();
                    
                    // Check for conflict!
                    FileMetadata current = metadataRepo.findByPath(
                        userId, change.getPath());
                    if (current != null && 
                        change.getBaseVersion() < current.getVersion()) {
                        // CONFLICT! Client's base version is stale!
                        response.addConflict(ConflictInfo.builder()
                            .path(change.getPath())
                            .serverVersion(current)
                            .clientChange(change)
                            .build());
                        continue;
                    }
                    
                    metadataRepo.save(meta);
                    journalRepo.append(userId, change);
                    break;
                    
                case DELETE:
                    metadataRepo.markDeleted(userId, change.getPath());
                    journalRepo.append(userId, change);
                    break;
                    
                case MOVE:
                    metadataRepo.movePath(userId, change.getOldPath(), 
                        change.getPath());
                    journalRepo.append(userId, change);
                    break;
            }
        }
        
        // Notify other devices of changes!
        notificationService.notifyOtherDevices(userId, deviceId, changes);
        
        // Return server changes that client doesn't have yet!
        response.setServerChanges(
            journalRepo.getChangesSince(userId, changes.get(0).getBaseVersion()));
        
        return response;
    }
    
    /**
     * Client polls for remote changes (or receives via WebSocket!).
     */
    public List<FileChange> getChangesSince(String userId, long sinceVersion) {
        return journalRepo.getChangesSince(userId, sinceVersion);
    }
}
```

---

## ❓ Interview Q&A

**Q1: How does block-level deduplication save storage at scale?**
> Content-addressable storage: each block stored by its SHA-256 hash. If two users upload the same file → same blocks → same hashes → stored ONCE! Real-world impact: (1) Email attachments sent to 100 people = stored once! (2) OS installer downloaded by 1M users = stored once! (3) Slight modifications to large files = most blocks unchanged = store only new blocks! Dropbox reported 75% dedup ratio (storing 4× less than naive approach!). At exabyte scale: saves hundreds of petabytes = massive cost savings! The hash acts as a natural dedup key — no coordination needed between users!

**Q2: Why content-defined chunking instead of fixed-size chunks?**
> Fixed-size: if you insert 1 byte at position 0 of a 1 GB file → ALL 256 chunks shift by 1 byte → all 256 hashes change → must re-upload entire file! Content-defined (Rabin fingerprint): chunk boundaries determined by local content patterns. Inserting 1 byte only affects 1-2 chunks (boundaries at other positions stay the same!). This is critical for delta sync: edit a document in the middle → only 1-2 blocks need re-upload, not 256! Trade-off: variable chunk sizes (harder to manage), slightly slower chunking (rolling hash computation). Worth it for the massive bandwidth savings!

**Q3: How do you handle sync conflicts when a user edits the same file on two offline devices?**
> Detection: each file has a version number. Client A syncs with base_version=5. Server has version=7 (device B already synced!). Conflict detected: base < current! Resolution: (1) For binary files: keep both versions — create "conflicted copy" file with device name + timestamp. User resolves manually. (2) For text files: attempt 3-way merge (common ancestor + version A + version B). If merge succeeds without overlap → auto-resolve! If overlapping changes → keep both. (3) Prevent conflicts: real-time sync via WebSocket reduces window for conflicts. Most conflicts happen during offline editing.

**Q4: How do you achieve 11-nines durability (99.999999999%)?**
> Multi-layer redundancy: (1) Each block stored in 3+ geographic regions (US, EU, Asia). (2) Within each region: erasure coding (6+3 scheme: data survives loss of 3 of 9 shards!). (3) Continuous integrity checks: background job verifies checksums of stored blocks (detect bit rot!). (4) If corruption detected: auto-heal from another replica! (5) Immutable storage: blocks are write-once, never modified (content-addressable!). Math: probability of losing all 3 regional copies simultaneously ≈ 10^-11 per year = 11 nines! (6) Metadata stored separately with its own 3× replication + WAL + snapshots.

---

## 🔗 Related Topics
- [Blob Storage](../BuildingBlocks/Blob_Storage.md) — Underlying storage
- [Checksums](../Foundations/Networking/Checksums.md) — Data integrity
- [Design Google Docs](./DesignGoogleDocs.md) — Real-time collaboration layer
- [Consistency Trade-offs](../Tradeoffs/Strong_vs_Eventual_Consistency.md) — Sync consistency

---

*"Cloud storage teaches you that the most impactful optimizations aren't about faster algorithms — they're about NOT doing work. Don't upload blocks that already exist (dedup). Don't re-upload unchanged blocks (delta sync). Don't transfer whole files when 1% changed (chunking). The best byte is the one you never send."* ☁️
