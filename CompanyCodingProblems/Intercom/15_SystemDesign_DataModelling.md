# 🟣 Intercom | System Design & Data Modelling — Deep Dive

> **Round:** 3B — Technical Design / Data Modelling (45 min)
> **Format:** Whiteboard/diagram session — NO coding, just schema + discussion
> **Key Focus:** Messaging system DB schema, indexing, scaling, feature extensions

---

## 📋 Table of Contents

1. [Round Format & What They Evaluate](#round-format--what-they-evaluate)
2. [Core Messaging Schema (Master Version)](#core-messaging-schema)
3. [Extension 1: Emoji Reactions](#extension-1-emoji-reactions)
4. [Extension 2: Message Threading & Replies](#extension-2-message-threading--replies)
5. [Extension 3: User Presence & Typing Indicators](#extension-3-user-presence--typing-indicators)
6. [Extension 4: Notification System](#extension-4-notification-system)
7. [Extension 5: Admin Analytics Dashboard](#extension-5-admin-analytics-dashboard)
8. [Extension 6: AI Agent Integration](#extension-6-ai-agent-integration)
9. [Indexing Strategy Deep Dive](#indexing-strategy-deep-dive)
10. [Scaling Discussion Points](#scaling-discussion-points)
11. [System Design: Intercom Architecture](#system-design-intercom-architecture)
12. [Practice Questions & Answers](#practice-questions--answers)

---

## Round Format & What They Evaluate

### How It Works

```
Interviewer: "Design the database schema for a messaging system 
              that supports private chats, group conversations, and admin inboxes."

You: [Draw entities → Define relationships → Add indexes → Discuss trade-offs]

Interviewer: "Now extend it to support emoji reactions on messages."

You: [Add new table, explain constraints, discuss queries]

Interviewer: "How would this scale to 10M conversations per day?"

You: [Discuss partitioning, caching, read replicas, denormalization]
```

### Evaluation Criteria

| What They Assess | How to Demonstrate |
|-----------------|-------------------|
| Entity identification | Name all core entities immediately |
| Relationship modeling | Correct FK relationships, junction tables |
| Indexing awareness | Propose indexes for specific query patterns |
| Trade-off articulation | "We could denormalize X for read speed, but..." |
| Extensibility thinking | Schema supports future features without breaking changes |
| Real-world experience | Reference actual messaging systems you've built/used |

---

## Core Messaging Schema

### Entity Relationship Diagram

```
┌─────────┐     ┌──────────────┐     ┌──────────────┐
│  users  │────<│ participants │>────│conversations │
└─────────┘     └──────────────┘     └──────────────┘
     │                                       │
     │           ┌──────────┐                │
     └──────────>│ messages │<───────────────┘
                 └──────────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
  ┌──────────┐ ┌──────────┐ ┌────────────┐
  │reactions │ │attachments│ │read_receipts│
  └──────────┘ └──────────┘ └────────────┘
```

### SQL Schema (Draw This from Memory)

```sql
-- 1. USERS (customers + admins)
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    display_name VARCHAR(100) NOT NULL,
    role        VARCHAR(20) NOT NULL DEFAULT 'customer', -- 'customer', 'admin', 'bot'
    avatar_url  TEXT,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- 2. CONVERSATIONS (private chat, group, or admin inbox thread)
CREATE TABLE conversations (
    id          BIGSERIAL PRIMARY KEY,
    type        VARCHAR(20) NOT NULL DEFAULT 'private', -- 'private', 'group', 'support'
    title       VARCHAR(255),         -- NULL for private, set for groups
    status      VARCHAR(20) NOT NULL DEFAULT 'open', -- 'open', 'closed', 'snoozed'
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- 3. PARTICIPANTS (many-to-many: users ↔ conversations)
CREATE TABLE participants (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    user_id         BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(20) NOT NULL DEFAULT 'member', -- 'member', 'admin', 'assignee'
    joined_at       TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    last_read_at    TIMESTAMP WITH TIME ZONE, -- For unread count calculation
    
    UNIQUE (conversation_id, user_id) -- One entry per user per conversation
);

-- 4. MESSAGES
CREATE TABLE messages (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id) ON DELETE CASCADE,
    sender_id       BIGINT NOT NULL REFERENCES users(id),
    content         TEXT NOT NULL,
    message_type    VARCHAR(20) NOT NULL DEFAULT 'text', -- 'text', 'system', 'note'
    reply_to_id     BIGINT REFERENCES messages(id), -- For threading
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    edited_at       TIMESTAMP WITH TIME ZONE,
    deleted_at      TIMESTAMP WITH TIME ZONE -- Soft delete
);

-- 5. ATTACHMENTS (files, images on messages)
CREATE TABLE attachments (
    id          BIGSERIAL PRIMARY KEY,
    message_id  BIGINT NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    file_url    TEXT NOT NULL,
    file_name   VARCHAR(255) NOT NULL,
    file_size   BIGINT NOT NULL,
    mime_type   VARCHAR(100) NOT NULL,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);
```

### Key Indexes

```sql
-- Messages: fetch by conversation (most common query)
CREATE INDEX idx_messages_conversation_time 
    ON messages(conversation_id, created_at DESC);

-- Participants: find user's conversations
CREATE INDEX idx_participants_user 
    ON participants(user_id, conversation_id);

-- Conversations: admin inbox (open conversations sorted by recent)
CREATE INDEX idx_conversations_status_updated 
    ON conversations(status, updated_at DESC);

-- Messages: search within conversation
CREATE INDEX idx_messages_sender 
    ON messages(sender_id, created_at DESC);
```

---

## Extension 1: Emoji Reactions

> **Interviewer prompt:** "How would you extend this to support emoji reactions on messages?"

### Schema Addition

```sql
CREATE TABLE reactions (
    id          BIGSERIAL PRIMARY KEY,
    message_id  BIGINT NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
    user_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    emoji       VARCHAR(50) NOT NULL,  -- Unicode emoji or shortcode
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),

    UNIQUE (message_id, user_id, emoji) -- One per emoji per user per message
);

CREATE INDEX idx_reactions_message ON reactions(message_id);
```

### Key Queries

```sql
-- Get reactions for a message (grouped)
SELECT emoji, COUNT(*) as count,
       ARRAY_AGG(u.display_name) as users,
       BOOL_OR(r.user_id = :current_user_id) as i_reacted
FROM reactions r
JOIN users u ON u.id = r.user_id
WHERE r.message_id = :msg_id
GROUP BY emoji
ORDER BY MIN(r.created_at);

-- Toggle reaction (application logic)
-- 1. Try INSERT; if conflict (UNIQUE violation) → DELETE instead
INSERT INTO reactions (message_id, user_id, emoji)
VALUES (:msg_id, :user_id, :emoji)
ON CONFLICT (message_id, user_id, emoji) DO NOTHING;
-- If 0 rows affected → it existed → DELETE it
```

### Discussion Points

| Question | Answer |
|----------|--------|
| "Custom emoji?" | Add `custom_emojis` table: id, shortcode, image_url, workspace_id |
| "Performance at scale?" | Cache reaction counts per message in Redis. Update async. |
| "Real-time?" | WebSocket push when reaction added/removed |
| "Why not denormalize count into messages?" | Adds complexity for writes; separate table is simpler and UNIQUE constraint prevents duplicates |

---

## Extension 2: Message Threading & Replies

> **Interviewer prompt:** "How would you add threading/replies to messages?"

### Approach 1: Simple reply_to (Already in schema)

```sql
-- Already have: reply_to_id BIGINT REFERENCES messages(id)

-- Get thread (all replies to a message)
SELECT * FROM messages 
WHERE reply_to_id = :parent_message_id 
ORDER BY created_at ASC;

-- Get message with reply count
SELECT m.*, 
       (SELECT COUNT(*) FROM messages r WHERE r.reply_to_id = m.id) as reply_count,
       (SELECT MAX(created_at) FROM messages r WHERE r.reply_to_id = m.id) as last_reply_at
FROM messages m
WHERE m.conversation_id = :conv_id AND m.reply_to_id IS NULL
ORDER BY m.created_at DESC;
```

### Approach 2: Full Thread Model (Slack-style)

```sql
CREATE TABLE threads (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    root_message_id BIGINT NOT NULL REFERENCES messages(id) UNIQUE,
    reply_count     INT NOT NULL DEFAULT 0,         -- Denormalized for list view
    last_reply_at   TIMESTAMP WITH TIME ZONE,       -- Denormalized for sorting
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Add thread_id to messages
ALTER TABLE messages ADD COLUMN thread_id BIGINT REFERENCES threads(id);

CREATE INDEX idx_messages_thread ON messages(thread_id, created_at ASC);
```

### Trade-off Discussion

| Approach | Pros | Cons |
|----------|------|------|
| Simple `reply_to_id` | No new tables, easy to implement | Need subquery for reply count, nested replies complex |
| Separate `threads` table | Clean separation, denormalized counts for speed | More tables, more writes on each reply |

**Recommendation:** Start with `reply_to_id` (simpler), add `threads` table later if needed for Slack-style threading.

---

## Extension 3: User Presence & Typing Indicators

> **Interviewer prompt:** "How would you show online status and typing indicators?"

### Design Decision: This is NOT a DB problem

```
❌ Don't store in PostgreSQL (too much write churn)
✅ Use Redis for ephemeral state
```

### Architecture

```
┌──────────┐    WebSocket    ┌──────────┐    Redis Pub/Sub    ┌──────────┐
│  Client  │ ──────────────> │  Server  │ ──────────────────> │  Redis   │
└──────────┘                 └──────────┘                     └──────────┘
                                                                   │
                             ┌──────────┐                          │
                             │  Server  │ <────────────────────────┘
                             └──────────┘    Subscribe
                                  │
                             ┌──────────┐
                             │  Client  │
                             └──────────┘
```

### Redis Data Model

```
# Online presence (TTL = 60 seconds, refresh every 30s)
SET   user:presence:{user_id}  "online"  EX 60

# Typing indicator (TTL = 5 seconds)
SET   conversation:typing:{conv_id}:{user_id}  "1"  EX 5

# Get who's typing in a conversation
KEYS  conversation:typing:{conv_id}:*
```

### What to Say in Interview

> "Presence and typing are ephemeral state — they don't need durability guarantees. I'd use Redis with TTL keys rather than PostgreSQL. The client sends a heartbeat every 30 seconds for presence, and emits a 'typing' event on keypress (debounced to 3 seconds). We use Redis pub/sub to fan out updates to other participants via WebSocket."

---

## Extension 4: Notification System

> **Interviewer prompt:** "How would you notify users of new messages?"

### Schema

```sql
CREATE TABLE notification_preferences (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    conversation_id BIGINT REFERENCES conversations(id), -- NULL = global default
    channel         VARCHAR(20) NOT NULL, -- 'in_app', 'email', 'push'
    enabled         BOOLEAN NOT NULL DEFAULT true,
    
    UNIQUE (user_id, conversation_id, channel)
);

CREATE TABLE notifications (
    id              BIGSERIAL PRIMARY KEY,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    type            VARCHAR(50) NOT NULL, -- 'new_message', 'mention', 'assignment'
    title           VARCHAR(255) NOT NULL,
    body            TEXT,
    conversation_id BIGINT REFERENCES conversations(id),
    message_id      BIGINT REFERENCES messages(id),
    read            BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_user_unread 
    ON notifications(user_id, read, created_at DESC)
    WHERE read = false; -- Partial index: only unread
```

### Notification Logic

```
When new message arrives in conversation:
1. Find all participants (except sender)
2. For each participant:
   a. Check notification_preferences
   b. Check if they're currently viewing this conversation (presence)
   c. If not viewing + preferences allow → create notification
3. Push via WebSocket (in-app) + async job for email/push
```

---

## Extension 5: Admin Analytics Dashboard

> **Interviewer prompt:** "How would you build metrics for first-response-time and resolution time?"

### Schema

```sql
-- Track conversation lifecycle events
CREATE TABLE conversation_events (
    id              BIGSERIAL PRIMARY KEY,
    conversation_id BIGINT NOT NULL REFERENCES conversations(id),
    event_type      VARCHAR(50) NOT NULL, 
        -- 'created', 'first_response', 'assigned', 'resolved', 'reopened'
    actor_id        BIGINT REFERENCES users(id),
    metadata        JSONB, -- Flexible data per event type
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_conv_events_type_time 
    ON conversation_events(event_type, created_at DESC);

-- Materialized metrics (computed hourly by background job)
CREATE TABLE conversation_metrics (
    id                  BIGSERIAL PRIMARY KEY,
    conversation_id     BIGINT NOT NULL REFERENCES conversations(id) UNIQUE,
    first_response_ms   BIGINT,  -- Time from creation to first admin reply
    resolution_ms       BIGINT,  -- Time from creation to resolved
    message_count       INT NOT NULL DEFAULT 0,
    assigned_agent_id   BIGINT REFERENCES users(id),
    created_at          TIMESTAMP WITH TIME ZONE NOT NULL,
    resolved_at         TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_metrics_agent ON conversation_metrics(assigned_agent_id, created_at);
```

### Key Queries

```sql
-- Average first response time this week, by agent
SELECT u.display_name,
       AVG(cm.first_response_ms) / 1000.0 as avg_first_response_seconds,
       COUNT(*) as conversations_handled
FROM conversation_metrics cm
JOIN users u ON u.id = cm.assigned_agent_id
WHERE cm.created_at >= NOW() - INTERVAL '7 days'
GROUP BY u.id, u.display_name
ORDER BY avg_first_response_seconds;

-- SLA compliance: % of conversations with first response < 5 min
SELECT 
    COUNT(*) FILTER (WHERE first_response_ms < 300000) * 100.0 / COUNT(*) 
    as sla_compliance_pct
FROM conversation_metrics
WHERE created_at >= NOW() - INTERVAL '24 hours';
```

---

## Extension 6: AI Agent Integration

> **This is relevant to Intercom's current product direction (Fin AI Agent)**

### Schema

```sql
CREATE TABLE ai_agents (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    model       VARCHAR(50) NOT NULL, -- 'gpt-4', 'claude-3', 'fin-v2'
    config      JSONB NOT NULL DEFAULT '{}', -- temperature, system prompt, etc.
    enabled     BOOLEAN NOT NULL DEFAULT true,
    created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- Track AI-generated messages for review/feedback
CREATE TABLE ai_message_metadata (
    id              BIGSERIAL PRIMARY KEY,
    message_id      BIGINT NOT NULL REFERENCES messages(id) UNIQUE,
    ai_agent_id     BIGINT NOT NULL REFERENCES ai_agents(id),
    confidence      DECIMAL(3,2), -- 0.00 to 1.00
    sources         JSONB, -- Knowledge base articles referenced
    feedback        VARCHAR(20), -- 'helpful', 'not_helpful', NULL
    escalated       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- When should AI hand off to human?
CREATE TABLE escalation_rules (
    id              BIGSERIAL PRIMARY KEY,
    ai_agent_id     BIGINT NOT NULL REFERENCES ai_agents(id),
    condition_type  VARCHAR(50) NOT NULL, -- 'low_confidence', 'keyword', 'user_request', 'loop_detected'
    condition_value JSONB NOT NULL,
    action          VARCHAR(20) NOT NULL DEFAULT 'escalate', -- 'escalate', 'ask_clarification'
    priority        INT NOT NULL DEFAULT 0
);
```

---

## Indexing Strategy Deep Dive

### Explain This in Interview

| Query Pattern | Index | Justification |
|---------------|-------|---------------|
| "Show messages in conversation" | `(conversation_id, created_at DESC)` | Most common query; composite for range scan |
| "Show user's conversations" | `(user_id)` on participants | Dashboard/inbox load |
| "Unread count per conversation" | `(conversation_id, created_at)` on messages + `last_read_at` on participants | WHERE created_at > last_read_at |
| "Search messages by text" | Full-text index OR Elasticsearch | Don't use LIKE '%text%' at scale |
| "Admin inbox: open conversations" | Partial index: `(status, updated_at) WHERE status = 'open'` | Only index open convos |

### What NOT to Index

- `messages.content` with B-tree (use full-text search instead)
- Every column individually (composite indexes are better)
- Columns only used in rare admin queries (accept slow for rare)

---

## Scaling Discussion Points

### When Interviewer Asks "How Would This Scale?"

| Challenge | Solution | When to Use |
|-----------|----------|-------------|
| Read-heavy on messages | Read replicas | >10k reads/sec on messages table |
| Single large conversations table | Partition by `created_at` (monthly) | >100M conversations |
| Hot conversations (viral support thread) | Cache recent messages in Redis | >1k reads/sec on single conversation |
| Unread count computation | Denormalize: store `unread_count` on participants, update on new message | Always (avoid COUNT(*) per request) |
| Search | Elasticsearch for full-text search | >10M messages |
| Real-time at scale | Redis pub/sub + WebSocket fan-out | >100k concurrent connections |
| Global distribution | Multi-region PostgreSQL (Citus/CockroachDB) | If SLA requires <100ms globally |

### Denormalization Trade-offs

```
✅ Denormalize: unread_count (computed on every page load otherwise)
✅ Denormalize: last_message_preview on conversations (avoid JOIN for inbox)
✅ Denormalize: reply_count on threads (avoid COUNT(*) for thread list)

❌ Don't denormalize: participant list (changes rarely, JOINs are fine)
❌ Don't denormalize: reaction details (only needed on single message view)
```

---

## System Design: Intercom Architecture

### If They Ask "How Would You Design Intercom's Messaging System?"

```
┌──────────────────────────────────────────────────────────────────────┐
│                          CLIENTS                                      │
│  Customer Widget (JS)   │   Admin Dashboard (React)   │  Mobile App  │
└─────────────┬───────────┴──────────────┬──────────────┴──────────────┘
              │ HTTPS + WebSocket         │
              ▼                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       API GATEWAY / LOAD BALANCER                     │
│              (Route: REST → API servers, WS → WS servers)            │
└─────────────┬───────────────────────────────────────┬────────────────┘
              │                                       │
              ▼                                       ▼
┌─────────────────────────┐         ┌─────────────────────────────────┐
│   REST API Servers      │         │    WebSocket Servers            │
│   (Stateless)           │         │    (Stateful connections)       │
│   • CRUD endpoints      │         │    • Real-time delivery         │
│   • Auth / Rate Limit   │         │    • Presence heartbeat         │
└─────────────┬───────────┘         └──────────────┬──────────────────┘
              │                                    │
              ▼                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     MESSAGE QUEUE (Kafka / RabbitMQ)                  │
│    Events: new_message, reaction_added, conversation_assigned, etc.  │
└─────────────┬──────────────┬──────────────┬─────────────────────────┘
              │              │              │
              ▼              ▼              ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
│ Notification │  │   Analytics  │  │   AI/Bot Engine      │
│   Service    │  │   Service    │  │   (Fin Agent)        │
│ • Email      │  │ • Metrics    │  │ • Auto-reply         │
│ • Push       │  │ • Dashboards │  │ • Classification     │
│ • In-app     │  │ • SLA track  │  │ • Escalation         │
└──────────────┘  └──────────────┘  └──────────────────────┘
              │              │              │
              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        DATA LAYER                                     │
│   PostgreSQL (primary)  │  Redis (cache/presence)  │  Elasticsearch  │
│   • Conversations       │  • Session data          │  • Message search│
│   • Messages            │  • Presence TTL keys     │  • Article index │
│   • Users               │  • Rate limiting         │                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions to Discuss

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Message delivery guarantee | At-least-once via queue | Losing messages is unacceptable; idempotent receivers handle duplicates |
| Message ordering | Per-conversation sequence number | Global ordering unnecessary; timestamp + sequence per conversation |
| Real-time delivery | WebSocket + server-push | Polling is wasteful for messaging; SSE is one-directional |
| Storage | PostgreSQL primary + Redis cache | Boring tech, team knows it, strong consistency for messages |
| Search | Elasticsearch async index | PostgreSQL full-text works for small scale; ES for millions of messages |
| File storage | S3/GCS + CDN | Never store files in the database |

---

## Practice Questions & Answers

### Q1: "Design schema for groups, private chat, relay messages"

**Answer framework:**
1. Single `conversations` table with `type` field (private/group/support)
2. `participants` junction table (handles both 1:1 and many:many)
3. `messages` belong to conversation (all message types in one table)
4. Private chat = conversation with exactly 2 participants
5. Group = conversation with 3+ participants + title

### Q2: "How do you handle unread counts efficiently?"

**Answer:**
> "Store `last_read_at` timestamp per participant. Unread count = messages in conversation WHERE created_at > participant.last_read_at. For performance, I'd denormalize by storing a counter that increments on new messages and resets when user reads. Update via async event after message insert."

### Q3: "How would you handle message delivery to offline users?"

**Answer:**
> "Messages are always stored in PostgreSQL first (source of truth). For online users, push via WebSocket immediately. For offline users, create a notification record. When they come online (WebSocket reconnects), query all messages since their `last_seen_at` timestamp. This guarantees no messages are lost regardless of connection state."

### Q4: "What happens if two people send a message at the exact same time?"

**Answer:**
> "PostgreSQL uses MVCC, so both INSERTs succeed. For ordering, I'd use a per-conversation sequence number (generated by `nextval()` or auto-increment within conversation partition). Display order is: sequence number within conversation, not global timestamp."

### Q5: "How would you migrate from storing messages in one table to sharding?"

**Answer:**
> "Phase 1: Add `shard_key` column (conversation_id % shard_count). Phase 2: Route new reads/writes by shard key. Phase 3: Background migration of old data. Phase 4: Remove routing to old single table. Use conversation_id as shard key because all queries for messages are within a single conversation."
