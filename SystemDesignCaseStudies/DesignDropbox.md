# 📁 Design Dropbox
## File Synchronization, Conflict Resolution, and Chunk-Based Storage at Scale

> *"Dropbox syncs 1.2 billion files per day across 700 million registered users. The core challenge isn't storage — it's detecting what changed in a 10GB file, transmitting only the delta, and resolving conflicts when two people edit the same file offline simultaneously."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md), [Blob Storage](../BuildingBlocks/Blob_Storage.md)

---

## 📋 Requirements (RADIO: R)

### Functional Requirements

| Feature | Description |
|---|---|
| **Upload files** | Upload any file type, up to 50GB |
| **Download files** | Download from any device |
| **Sync across devices** | Changes on one device appear on all others automatically |
| **File versioning** | Access previous versions, restore deleted files (30 days) |
| **Sharing** | Share files/folders with permissions (view, edit) |
| **Conflict resolution** | Handle concurrent edits gracefully |
| **Offline support** | Edit files offline, sync when back online |

### Non-Functional Requirements

```
Scale:
  - 700M registered users, 15M paying users
  - 1.2 billion files synced per day
  - Average file: 1MB, largest: 50GB
  - Storage: hundreds of PB total user data
  - Sync latency: < 5 seconds for small file changes on same network
  - Bandwidth: minimize transfer (send only changes, not full files)
  - Reliability: zero data loss (durability > 99.9999999% — 9 nines)
```

---

## 📡 API Design (RADIO: A)

```
POST /v1/files/upload_session/start
  Request: { "file_size": 5242880000 }
  Response: { "session_id": "sess_abc123" }

POST /v1/files/upload_session/append
  Request: { "session_id": "sess_abc123", "offset": 0, chunk_data (binary) }

POST /v1/files/upload_session/finish
  Request: { "session_id": "sess_abc123", "path": "/documents/report.pdf" }
  Response: { "file_id": "f456", "version": 3, "content_hash": "sha256:..." }

GET /v1/files/download?path=/documents/report.pdf&version=3

GET /v1/files/delta?cursor=<last_sync_cursor>
  Response: {
    "entries": [
      { "path": "/docs/file.txt", "action": "modified", "version": 4, "hash": "..." },
      { "path": "/docs/old.txt", "action": "deleted" }
    ],
    "cursor": "new_cursor_xyz",
    "has_more": false
  }

POST /v1/files/sharing
  Request: { "path": "/shared/project", "members": [{"email": "...", "access": "editor"}] }
```

---

## 🗄️ Data Model (RADIO: D)

```
Storage choices:
  - PostgreSQL: file metadata, users, sharing permissions, version history
  - S3: actual file chunks (content-addressed, deduplicated)
  - Redis: sync state, active sessions, notification pubsub
  - Kafka: sync events, change notifications

File metadata schema:
  files: {
    id, user_id, path, filename,
    latest_version, content_hash (SHA-256 of full file),
    size_bytes, is_deleted,
    created_at, modified_at
  }

  file_versions: {
    file_id, version_number,
    chunk_hashes[] (ordered list of chunk hashes composing this version),
    modified_by, modified_at,
    change_description
  }

  chunks: {
    chunk_hash (SHA-256, primary key — content-addressed),
    size_bytes,
    s3_location,
    reference_count (for garbage collection)
  }

Deduplication via content-addressed storage:
  Same chunk appearing in 1000 users' files → stored ONCE on S3
  Example: default "README.md" in every Git repo → 1 copy, 1000 references
```

---

## 🏗️ Infrastructure (RADIO: I)

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    DROPBOX ARCHITECTURE                                  │
│                                                                          │
│  [Desktop Client] ←─────── sync ──────→ [Mobile Client]                │
│       │                                        │                         │
│       │ file watcher detects change            │                         │
│       ▼                                        ▼                         │
│  ┌──────────────────────────────────────────────────┐                   │
│  │              SYNC SERVICE (stateful)              │                   │
│  │  - Tracks client sync cursors                    │                   │
│  │  - Determines delta (what changed since cursor)  │                   │
│  │  - Resolves conflicts                            │                   │
│  │  - Notifies other clients via long-poll/websocket│                   │
│  └──────────────────────────────────────────────────┘                   │
│       │                            │                                     │
│       ▼                            ▼                                     │
│  ┌──────────────┐         ┌──────────────────┐                          │
│  │ METADATA DB  │         │ BLOCK SERVICE     │                          │
│  │ (PostgreSQL) │         │ (chunk storage)   │                          │
│  │              │         │                    │                          │
│  │ file tree    │         │ 1. Receive chunks  │                          │
│  │ versions     │         │ 2. Deduplicate     │                          │
│  │ permissions  │         │ 3. Compress        │                          │
│  │ sync cursors │         │ 4. Encrypt (AES-256)│                         │
│  └──────────────┘         │ 5. Store to S3     │                          │
│                            └──────────────────┘                          │
│                                     │                                    │
│                                     ▼                                    │
│                            [S3: encrypted chunks]                        │
│                            (content-addressed by SHA-256 hash)           │
│                                                                          │
│  ┌──────────────────────────────────────────────────┐                   │
│  │         NOTIFICATION SERVICE                      │                   │
│  │  - Long-polling / WebSocket connections           │                   │
│  │  - "File X changed" → push to all synced clients │                   │
│  │  - Clients pull delta after notification          │                   │
│  └──────────────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────────────┘
```

### Chunking and Delta Sync (The Core Algorithm)

```
WHY CHUNK? (instead of uploading full files)

  User has a 1GB video file synced to Dropbox.
  They edit 10 bytes of metadata in the file header.

  Without chunking: re-upload entire 1GB (wasteful!)
  With chunking: re-upload only the 4MB chunk that changed

HOW CHUNKING WORKS:

  File: report.pdf (20MB)
  Split into 4MB chunks:
    Chunk 1: [bytes 0 - 4MB]      → SHA-256: "abc123"
    Chunk 2: [bytes 4MB - 8MB]    → SHA-256: "def456"
    Chunk 3: [bytes 8MB - 12MB]   → SHA-256: "ghi789"
    Chunk 4: [bytes 12MB - 16MB]  → SHA-256: "jkl012"
    Chunk 5: [bytes 16MB - 20MB]  → SHA-256: "mno345"

  File metadata: { chunks: ["abc123", "def456", "ghi789", "jkl012", "mno345"] }

  User edits page 3 (in chunk 2):
    Chunk 2 content changes → new hash: "xyz999"
    Only chunk 2 is re-uploaded (4MB instead of 20MB!)
    
  Updated metadata: { chunks: ["abc123", "xyz999", "ghi789", "jkl012", "mno345"] }


CONTENT-DEFINED CHUNKING (advanced):
  Fixed-size chunks have a problem: inserting 1 byte at the start shifts ALL chunk boundaries
  Solution: Rabin fingerprinting — chunk boundaries defined by content patterns
    Roll a window over the file; when hash(window) mod N == 0, create boundary
    Result: insertions/deletions only affect 1-2 chunks, not all subsequent chunks
```

### Conflict Resolution

```
SCENARIO: User edits "report.docx" on laptop (offline) AND phone simultaneously

  Laptop (offline): version 5 → edits → creates version 6-local
  Phone (online): version 5 → edits → creates version 6-server

  Laptop comes back online, tries to sync version 6-local:
    Server says: "conflict! server already has version 6"

RESOLUTION STRATEGIES:

  Strategy 1: LAST-WRITER-WINS (simple but lossy)
    Later timestamp wins, earlier edit is discarded
    Used for: non-collaborative files, settings files
    Risk: user loses work

  Strategy 2: CONFLICT COPY (Dropbox's default)
    Keep BOTH versions:
      report.docx (server's version 6)
      report (laptop's conflicted copy 2024-01-15).docx
    User manually resolves
    Used for: any file type (universal, safe, no data loss)

  Strategy 3: OPERATIONAL TRANSFORM / CRDT (Google Docs style)
    Merge changes at the operation level (not file level)
    Insert at position 5 + insert at position 10 → both applied
    Used for: collaborative editing (requires structured documents)
    Dropbox Paper uses this; regular file sync does not

Dropbox's actual approach:
  - Detect conflict (version mismatch on upload)
  - Create conflict copy (no data loss, ever)
  - Notify user via desktop notification
  - User resolves manually (or automated for specific file types)
```

---

## 📈 Optimization (RADIO: O)

```
1. DEDUPLICATION (saves 50%+ storage)
   Problem: millions of users store the same files (default OS files, popular downloads)
   Solution: content-addressed storage — chunks identified by SHA-256 hash
     Two users upload the same 4MB chunk → stored once, referenced twice
     reference_count tracks how many files use each chunk
     Garbage collection: delete chunk from S3 only when reference_count → 0

2. BANDWIDTH OPTIMIZATION
   - Compression: chunks compressed with LZ4 (fast) before upload
   - Delta encoding: for text files, compute diff and send only changed bytes
   - Streaming dedup: client computes chunk hashes locally BEFORE uploading
     Client: "I have chunks [abc, def, ghi]"
     Server: "I already have [abc, ghi]. Only upload [def]"
     Result: upload 4MB instead of 12MB (server-side dedup check)

3. SYNC PROTOCOL OPTIMIZATION
   - Long-polling: client maintains connection, server pushes "changes available"
   - Batched sync: aggregate multiple rapid changes (user saves 10 times in 5 seconds)
     Wait 2 seconds after last change, then sync once
   - Priority sync: small text files sync immediately; large binaries queue behind

4. SCALABILITY OF METADATA
   Problem: 700M users × avg 500 files = 350 billion file metadata rows
   Solution: 
     - Shard metadata DB by user_id
     - Cache hot users' file trees in Redis
     - Namespace isolation: each user's file tree is independent (no cross-user joins)
```

---

## 🧩 Mini Challenge

**Two users share a folder. User A renames "report.docx" to "final_report.docx" while offline. User B edits "report.docx" while online. When User A comes back online, what happens?**

<details>
<summary>💡 Click to reveal answer</summary>

**The tricky part**: User A's rename and User B's edit are on the "same" file but are different operations.

**Resolution approach**:

1. When User A comes online, the sync client sends: "rename report.docx → final_report.docx (based on version 5)"
2. Server state: "report.docx" is now at version 6 (User B's edit)
3. **Conflict detection**: The base version for A's operation (version 5) doesn't match the current server version (6)

**Correct behavior (what Dropbox does)**:
- Apply User B's edit first (already on server): `report.docx` is at version 6 with B's changes
- Apply User A's rename: rename `report.docx` → `final_report.docx`
- Result: `final_report.docx` contains User B's edits (version 7, renamed)
- Both operations are preserved — no conflict copy needed!

**Why this works**: Rename and edit are *non-conflicting operations* — they modify different aspects of the file (name vs. content). The sync engine recognizes this and applies both sequentially. A conflict copy would only be created if both users modified the *content* of the same file.

**Edge case**: If User A also edited the file content before renaming → create a conflict copy:
- `final_report.docx` (User A's content, renamed)
- `final_report (B's conflicted copy).docx` (User B's content)

</details>

---

## 📝 Interview Q&A

**Q: How does Dropbox minimize bandwidth usage when syncing large files?**
> A: Three mechanisms: (1) **Chunking** — files are split into 4MB blocks, and only modified chunks are uploaded. A 1GB file with 10 bytes changed → upload only one 4MB chunk instead of 1GB. (2) **Content-defined chunking** (Rabin fingerprinting) — chunk boundaries are determined by content patterns, not fixed offsets, so insertions don't cascade into re-uploading all subsequent chunks. (3) **Server-side dedup check** — before uploading, the client sends chunk hashes to the server. The server responds with "I already have chunks X and Y, only send Z." If another user already uploaded the same chunk, it's not uploaded again.

**Q: How would you handle a user with 100,000 files that all change simultaneously (e.g., a Git checkout)?**
> A: Batch processing with prioritization. (1) The file watcher detects all 100K changes but debounces — waits 5 seconds after the last change before starting sync. (2) Group changes into a single sync batch sent to the server as one transaction. (3) Prioritize: small files and recently-opened files sync first; large binary files queue behind. (4) Rate limit uploads to avoid saturating the user's bandwidth. (5) Server-side: process the batch atomically — either all file metadata updates succeed or none do (prevents partial sync states).

---

## 🔗 What to Read Next

1. **[BuildingBlocks/Blob_Storage.md](../BuildingBlocks/Blob_Storage.md)** — Object storage underpins Dropbox's chunk storage layer
2. **[SystemDesignCaseStudies/DesignWhatsApp.md](./DesignWhatsApp.md)** — Similar offline-first sync challenges in messaging
3. **[KeyConcepts/Consistent_Hashing.md](../KeyConcepts/Consistent_Hashing.md)** — Sharding metadata across servers

---

*[← Design Google Search](./DesignGoogleSearch.md) | [Back to Case Studies](./README.md)*
