# 🟣 Intercom | Problem 06 — Chat System Low Level Design (DB Schema)

> **Source:** Intercom System Design Round (recurring)
> **Difficulty:** 🔴 Hard
> **Topics:** Low Level Design · RDBMS · DB Schema · Normalization · Indexing · Messaging Systems

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Design the database schema for a chat-based system that supports:
> - **Private (1:1) conversations** between two users
> - **Group conversations** with multiple participants
> - **Message sending and relaying**
> - **Message history retrieval** (with pagination)
> - **Read receipts / unread counts**
> - **User presence (online/offline)**

**Clarifying Questions to Ask:**
- "Should the schema support both private (1:1) and group chats in a unified model or separate tables?"
- "Do we need to support message editing and deletion (soft delete)?"
- "Do we need read receipts (delivered/seen per user)?"
- "Is there a concept of message threading (replies to a specific message)?"
- "Do we need file/image attachments in messages?"
- "Should we track which users are online/offline (presence)?"
- "What is the expected scale — how many users, messages per day?"
- "Do we need to support message search?"
- "Should users be able to leave groups? Should we soft-delete their membership?"
- "Do we need message reactions (emoji reactions per message)?"

**Confirmed Scope for This Design:**
- ✅ Private (1:1) and group conversations (unified model)
- ✅ Message send, edit, soft-delete
- ✅ Read receipts (delivered_at, read_at per user per message)
- ✅ Message reply/threading (reply_to_message_id)
- ✅ User presence (last_seen_at)
- ✅ Unread count per user per conversation
- ✅ Pagination-friendly message retrieval
- ⬜ Full-text search (mention Elasticsearch as extension)
- ⬜ Reactions (mention as optional extension)

---

### **2️⃣ Breaking Down the Solution**

**Key Design Decisions:**

**Decision 1 — Unified Conversation Model**
> Use a single `conversations` table with a `type` column (`private` | `group`) rather than separate `private_chats` and `group_chats` tables. This simplifies queries and allows a single participant list.

**Decision 2 — Participants Table (not inline)**
> Store participants in a separate `participants` table, not as an array column. This allows:
> - Efficient lookup: "which conversations is user X in?"
> - Per-participant metadata: `last_read_message_id`, `joined_at`, `left_at`

**Decision 3 — Message Receipts vs. Inline Read Status**
> A separate `message_receipts` table allows per-user delivery/read tracking without denormalizing the `messages` table. For group chats, this is essential.

**Decision 4 — Soft Delete for Messages**
> Use `deleted_at IS NOT NULL` pattern instead of hard delete. This preserves message history for audit trails and allows "message deleted" placeholders in UI.

**Decision 5 — Pagination via cursor (message ID)**
> Use `WHERE id < :last_seen_id ORDER BY id DESC LIMIT n` for efficient cursor-based pagination, not `OFFSET` (which is slow on large tables).

---

### **3️⃣ Complete Database Schema**

#### Entity Relationship Overview

```
┌──────────┐     ┌────────────────┐     ┌───────────────┐
│  users   │────<│  participants  │>────│ conversations │
└──────────┘     └────────────────┘     └───────────────┘
                                                │
                                         ┌──────┴──────┐
                                         │             │
                                    ┌────────┐    ┌────────┐
                                    │ groups │    │messages│
                                    └────────┘    └────────┘
                                                       │
                                              ┌────────────────┐
                                              │message_receipts│
                                              └────────────────┘
```

---

#### Table 1 — `users`

```sql
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    username        VARCHAR(100)  UNIQUE NOT NULL,
    email           VARCHAR(255)  UNIQUE NOT NULL,
    display_name    VARCHAR(255)  NOT NULL,
    avatar_url      TEXT,
    password_hash   VARCHAR(255)  NOT NULL,
    status          VARCHAR(20)   NOT NULL DEFAULT 'offline'
                        CHECK (status IN ('online', 'offline', 'away', 'busy')),
    last_seen_at    TIMESTAMP WITH TIME ZONE,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at      TIMESTAMP WITH TIME ZONE  -- soft delete
);

-- Indexes
CREATE INDEX idx_users_email       ON users(email);
CREATE INDEX idx_users_username    ON users(username);
CREATE INDEX idx_users_last_seen   ON users(last_seen_at);
```

**Why these indexes?**
- `email` / `username` → login, @mention lookup, unique constraint enforcement
- `last_seen_at` → presence queries: "show me users active in last 5 minutes"

---

#### Table 2 — `conversations`

```sql
CREATE TABLE conversations (
    id                  BIGSERIAL PRIMARY KEY,
    type                VARCHAR(10) NOT NULL CHECK (type IN ('private', 'group')),
    last_message_id     BIGINT,                         -- FK added below (circular ref handled)
    last_message_at     TIMESTAMP WITH TIME ZONE,       -- denormalized for sort performance
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Index for inbox: fetch conversations sorted by most recent activity
CREATE INDEX idx_conversations_last_message_at ON conversations(last_message_at DESC);
```

**Why `last_message_at` denormalized?**
> Fetching "inbox" (conversations sorted by latest activity) is a hot path. Computing `MAX(messages.created_at)` per conversation on every inbox load is expensive. Denormalizing this value makes inbox queries O(1) per conversation.

---

#### Table 3 — `groups`

```sql
CREATE TABLE groups (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL UNIQUE REFERENCES conversations(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    avatar_url      TEXT,
    created_by      BIGINT NOT NULL REFERENCES users(id),
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_groups_conversation ON groups(conversation_id);
```

**Note:** `groups.conversation_id` is 1:1 with `conversations`. This avoids NULLs in `conversations` for private chats — the `groups` row simply doesn't exist for private conversations.

---

#### Table 4 — `participants`

```sql
CREATE TABLE participants (
    conversation_id     BIGINT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    user_id             BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role                VARCHAR(20) NOT NULL DEFAULT 'member'
                            CHECK (role IN ('member', 'admin', 'owner')),
    last_read_message_id BIGINT,                         -- for unread count computation
    joined_at           TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    left_at             TIMESTAMP WITH TIME ZONE,        -- NULL = still in group
    muted_until         TIMESTAMP WITH TIME ZONE,

    PRIMARY KEY (conversation_id, user_id)
);

-- Lookup: "what conversations is user X in?"
CREATE INDEX idx_participants_user   ON participants(user_id, left_at);
-- Lookup: "who is in conversation Y?"
CREATE INDEX idx_participants_conv   ON participants(conversation_id, left_at);
```

**`last_read_message_id` for unread count:**
```sql
-- Unread count for a user in a conversation:
SELECT COUNT(*)
FROM   messages m
WHERE  m.conversation_id = :conv_id
AND    m.id > :last_read_message_id     -- messages after last read
AND    m.sender_id != :user_id          -- don't count own messages
AND    m.deleted_at IS NULL;
```

---

#### Table 5 — `messages`

```sql
CREATE TABLE messages (
    id                  BIGSERIAL PRIMARY KEY,
    conversation_id     BIGINT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id           BIGINT NOT NULL REFERENCES users(id),
    content             TEXT,
    message_type        VARCHAR(20) NOT NULL DEFAULT 'text'
                            CHECK (message_type IN ('text', 'image', 'video', 'file', 'system', 'audio')),
    reply_to_message_id BIGINT REFERENCES messages(id) ON DELETE SET NULL,
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP WITH TIME ZONE,       -- NULL if never edited
    deleted_at          TIMESTAMP WITH TIME ZONE        -- NULL if not deleted (soft delete)
);

-- PRIMARY hot-path: paginate messages in a conversation (newest first)
CREATE INDEX idx_messages_conv_time     ON messages(conversation_id, id DESC);
-- Sender lookup: "all messages by user X"
CREATE INDEX idx_messages_sender        ON messages(sender_id, created_at DESC);
-- Thread lookup: "all replies to message Y"
CREATE INDEX idx_messages_reply         ON messages(reply_to_message_id)
    WHERE reply_to_message_id IS NOT NULL;
```

---

#### Table 6 — `message_receipts`

```sql
CREATE TABLE message_receipts (
    message_id      BIGINT NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    delivered_at    TIMESTAMP WITH TIME ZONE,   -- NULL = not yet delivered
    read_at         TIMESTAMP WITH TIME ZONE,   -- NULL = not yet read

    PRIMARY KEY (message_id, user_id)
);

-- Look up all unread messages for a user
CREATE INDEX idx_receipts_user_unread ON message_receipts(user_id, read_at)
    WHERE read_at IS NULL;
```

---

#### Table 7 — `attachments` (Extension)

```sql
CREATE TABLE attachments (
    id              BIGSERIAL PRIMARY KEY,
    message_id      BIGINT NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    file_url        TEXT NOT NULL,              -- CDN URL
    file_name       VARCHAR(255) NOT NULL,
    file_size_bytes BIGINT,
    mime_type       VARCHAR(100),               -- e.g., "image/jpeg", "application/pdf"
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_attachments_message ON attachments(message_id);
```

---

### **4️⃣ Key Queries (Explaining to Interviewer)**

#### Q1 — Load Inbox (conversations for a user, sorted by latest message)

```sql
SELECT
    c.id             AS conversation_id,
    c.type,
    c.last_message_at,
    g.name           AS group_name,       -- NULL for private chats
    m.content        AS last_message_preview,
    m.sender_id      AS last_message_sender,
    -- Unread count
    (
        SELECT COUNT(*)
        FROM   messages m2
        WHERE  m2.conversation_id = c.id
        AND    m2.id > COALESCE(p.last_read_message_id, 0)
        AND    m2.sender_id != :current_user_id
        AND    m2.deleted_at IS NULL
    ) AS unread_count
FROM  participants p
JOIN  conversations c  ON c.id = p.conversation_id
LEFT  JOIN groups g    ON g.conversation_id = c.id
LEFT  JOIN messages m  ON m.id = c.last_message_id
WHERE p.user_id = :current_user_id
AND   p.left_at IS NULL
ORDER BY c.last_message_at DESC
LIMIT 20;
```

---

#### Q2 — Load Message History (Cursor-based Pagination)

```sql
-- First page (newest messages)
SELECT m.*, u.display_name AS sender_name
FROM   messages m
JOIN   users u ON u.id = m.sender_id
WHERE  m.conversation_id = :conv_id
AND    m.deleted_at IS NULL
ORDER  BY m.id DESC
LIMIT  50;

-- Next page (cursor = last seen message id from previous page)
SELECT m.*, u.display_name AS sender_name
FROM   messages m
JOIN   users u ON u.id = m.sender_id
WHERE  m.conversation_id = :conv_id
AND    m.id < :cursor_message_id        -- cursor-based, not OFFSET
AND    m.deleted_at IS NULL
ORDER  BY m.id DESC
LIMIT  50;
```

**Why cursor-based over OFFSET?**
> `OFFSET 1000` requires the DB to scan and skip 1000 rows. `WHERE id < :cursor` uses the index directly — O(log n) instead of O(n).

---

#### Q3 — Mark Messages as Read (Batch)

```sql
-- Update last_read_message_id for the participant
UPDATE participants
SET    last_read_message_id = :latest_message_id
WHERE  conversation_id = :conv_id
AND    user_id = :user_id;

-- Insert/update receipts for messages user has now read
INSERT INTO message_receipts (message_id, user_id, read_at)
SELECT m.id, :user_id, NOW()
FROM   messages m
WHERE  m.conversation_id = :conv_id
AND    m.id > COALESCE((
           SELECT last_read_message_id FROM participants
           WHERE  conversation_id = :conv_id AND user_id = :user_id
       ), 0)
AND    m.sender_id != :user_id
ON CONFLICT (message_id, user_id)
DO UPDATE SET read_at = NOW() WHERE message_receipts.read_at IS NULL;
```

---

#### Q4 — Find or Create a Private Conversation Between Two Users

```sql
-- Check if a private conversation already exists between user A and user B
SELECT p1.conversation_id
FROM   participants p1
JOIN   participants p2 ON p2.conversation_id = p1.conversation_id
JOIN   conversations c  ON c.id = p1.conversation_id
WHERE  p1.user_id = :user_a
AND    p2.user_id = :user_b
AND    c.type = 'private'
AND    p1.left_at IS NULL
AND    p2.left_at IS NULL
LIMIT  1;

-- If none found, create one (application-level transaction):
-- 1. INSERT INTO conversations (type) VALUES ('private')
-- 2. INSERT INTO participants (conversation_id, user_id) VALUES (:new_id, :user_a), (:new_id, :user_b)
```

---

### **5️⃣ Discussing Edge Cases & Design Trade-offs**

#### Trade-off Table

| Decision | Option A | Option B | Chosen | Reason |
|----------|----------|----------|--------|--------|
| Unified vs separate tables | `conversations` + `type` | `private_chats` + `group_chats` | Unified | Simpler queries; participants model works for both |
| Read status tracking | `last_read_message_id` in participants | Full `message_receipts` table | Both | `participants.last_read_message_id` for fast unread count; receipts for per-message status |
| Pagination | OFFSET-based | Cursor-based (message ID) | Cursor | O(log n) vs O(n); stable across new inserts |
| Message delete | Hard delete | Soft delete (`deleted_at`) | Soft delete | Audit trail; "Message deleted" placeholder in UI |
| `last_message_at` | Compute from MAX | Denormalized column | Denormalized | Inbox sort is hot path; avoid expensive aggregation |
| Groups metadata | In `conversations` | Separate `groups` table | Separate | Private chats have no group metadata; avoids NULLs |

#### Scalability Considerations

```
Scale challenge: At 10M users × 100 msgs/day = 1B messages/month

Solutions:
1. Partition messages table by conversation_id (horizontal sharding)
2. Archive old messages to cold storage (S3 + Glacier) — keep last 90 days hot
3. Use Redis for:
   - Presence (user online status) — TTL-based keys
   - Unread counts — Redis counters, avoid DB query on every inbox load
   - Recent messages cache — sorted sets by timestamp
4. Use Elasticsearch for full-text message search
5. Fan-out on write vs read for group message delivery
```

#### Privacy & Security Constraints

```sql
-- Ensure a user can only read messages from conversations they're in:
-- (enforced at application layer, but schema supports it)
SELECT m.*
FROM   messages m
JOIN   participants p ON p.conversation_id = m.conversation_id
WHERE  m.id = :message_id
AND    p.user_id = :requesting_user_id  -- access control check
AND    p.left_at IS NULL;
```

---

### **6️⃣ Common Mistakes & How to Avoid Them**

❌ **Mistake 1: Separate tables for private vs group chat**
> Leads to duplicate participant logic, joins become complex when querying "all conversations for a user".

❌ **Mistake 2: Storing participants as an array column in `conversations`**
> Cannot efficiently query "which conversations is user X in?" — no index on array elements.
> Cannot store per-participant metadata (read cursor, role, join time).

❌ **Mistake 3: Using OFFSET for message pagination**
> Slow on large tables, unstable when new messages are inserted between pages.
> Use cursor-based pagination with `WHERE id < :last_seen_id`.

❌ **Mistake 4: No soft delete on messages**
> Hard deleting messages breaks conversation history and prevents "Message deleted" UI state.

❌ **Mistake 5: No `last_message_at` denormalization**
> Computing `MAX(messages.created_at)` per conversation on every inbox load is O(n) per row.
> Denormalize to avoid this — update `conversations.last_message_at` on every INSERT to `messages`.

❌ **Mistake 6: Missing composite indexes**
> `messages` without `(conversation_id, id DESC)` forces a full table scan on every message load.

---

## 🚀 Schema Summary (Quick Reference)

```
users
  id, username, email, display_name, avatar_url, password_hash,
  status, last_seen_at, created_at, deleted_at

conversations
  id, type ('private'|'group'), last_message_id, last_message_at, created_at

groups
  id, conversation_id(FK→conversations), name, description, avatar_url,
  created_by(FK→users), created_at

participants
  PK(conversation_id, user_id), role, last_read_message_id,
  joined_at, left_at, muted_until

messages
  id, conversation_id(FK), sender_id(FK→users), content, message_type,
  reply_to_message_id(FK→messages), created_at, updated_at, deleted_at

message_receipts
  PK(message_id, user_id), delivered_at, read_at

attachments
  id, message_id(FK→messages), file_url, file_name, file_size_bytes,
  mime_type, created_at
```

---

## 📎 Related Topics

| Topic | Link |
|-------|------|
| Database Normalization | [Normalization.md](../../Database/Normalization.md) |
| Indexing strategies | [Indexing.md](../../Database/Indexing.md) |
| SQL Query Optimization | [QueryOptimization.md](../../Database/QueryOptimization.md) |
| ACID properties | [ACID.md](../../Database/ACID.md) |
| Connection Pooling | [ConnectionPooling.md](../../Database/ConnectionPooling.md) |
| System Design — WebSockets (for real-time) | [WebSockets.md](../../APIs/WebSockets.md) |
| Caching (Redis for unread counts) | [Caching.md](../../BuildingBlocks/Caching.md) |

---

*Previous: [← 05 TransactionHistogram](./05_TransactionHistogram.md) | Next: [07 TaskScheduler →](./07_TaskScheduler.md)*
