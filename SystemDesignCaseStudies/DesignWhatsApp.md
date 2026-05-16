# 💬 Design WhatsApp
## Real-Time Messaging, Guaranteed Delivery, and End-to-End Encryption

> *"WhatsApp sends 100 billion messages per day with a team of 50 engineers. It's the most efficient large-scale system per-engineer ever built — and understanding why reveals everything about real-time architecture."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Design Twitter](./DesignTwitter.md), [BuildingBlocks/](../BuildingBlocks/)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Requirements (R)](#-requirements-r)
3. [API Design (A)](#-api-design-a)
4. [Data Model (D)](#-data-model-d)
5. [Infrastructure (I)](#-infrastructure-i)
6. [The Hard Part: Message Delivery & Ordering](#-the-hard-part-message-delivery--ordering)
7. [Group Messaging](#-group-messaging)
8. [Optimization (O)](#-optimization-o)
9. [Code Examples](#-code-examples)
10. [Industry Examples](#-industry-examples)
11. [Common Pitfalls](#-common-pitfalls)
12. [Mini Challenge](#-mini-challenge)
13. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

You send a message to your friend. Then immediately board a flight with no internet. They send a reply 8 hours later.

When you land and reconnect, your app must:
1. Show your message as ✓ (sent), ✓✓ (delivered), 🔵✓✓ (read) — in the right states
2. Receive your friend's reply instantly upon reconnection
3. Show messages in the EXACT correct order (even if you sent while they sent simultaneously)
4. Do all of this for **100 billion messages per day** without ever losing a single one

This is harder than it sounds. A "simple" HTTP request-response doesn't work for real-time. A "simple" database doesn't handle 100B messages/day. And "simple" delivery becomes a distributed systems nightmare.

---

## 📐 Requirements (R)

### Functional Requirements
```
Core features:
  ✅ 1:1 direct messaging (text, images, video, documents, voice notes)
  ✅ Message delivery receipts (sent ✓, delivered ✓✓, read 🔵✓✓)
  ✅ Online presence indicators (last seen, online status)
  ✅ Group chats (up to 1,024 members)
  ✅ Push notifications when app is closed
  ✅ Message history on new device

Out of scope:
  ❌ Voice/video calls (separate WebRTC-based system)
  ❌ Status/Stories
  ❌ Payments
```

### Non-Functional Requirements
```
Scale:
  - 2 billion users, 100M DAU sending messages
  - 100 billion messages/day (~1.15M messages/sec avg, ~5M/sec peak)
  - 500M group chats, avg 30 members
  - Messages are text (~1KB), images (~500KB), videos (~25MB)

Performance:
  - Message delivery: < 100ms P50 (online recipient)
  - Maximum delivery time for offline user: when they reconnect

Reliability:
  - Exactly-once delivery (no duplicates, no losses)
  - Message ordering must be consistent across both sides
  - Messages must persist for user's lifetime

Security:
  - End-to-end encryption (E2EE) — WhatsApp uses Signal Protocol
  - Server must NOT be able to read message content
```

---

## 🌐 API Design (A)

```
# WebSocket connection (persistent, long-lived)
WS  /ws/connect?token={authToken}
  → Establishes bidirectional channel between client and server
  → All real-time events (messages, receipts, presence) flow through this connection

# REST: for operations that don't need real-time
POST   /v1/messages
  Request:  { "chatId": "...", "encryptedContent": "...", "clientMsgId": "uuid", "mediaUrl": "..." }
  Response: { "msgId": "12345", "serverTimestamp": "..." }

GET    /v1/chats/{chatId}/messages?before={msgId}&limit=50
  Response: { "messages": [...], "hasMore": true }

POST   /v1/media/upload-url
  Response: { "uploadUrl": "s3-presigned-url", "mediaId": "uuid" }

# Receipts (sent via WebSocket event, not HTTP)
WS event: message_delivered  { msgId, recipientId, timestamp }
WS event: message_read        { msgId, recipientId, timestamp }
WS event: user_typing         { chatId, userId }
```

---

## 🗄️ Data Model (D)

### Storage Choices

```
messages         → Cassandra (write-heavy, time-series, massive scale)
chats/groups     → MySQL (relational: group members, metadata)
users            → MySQL (user profiles, auth)
message queue    → Redis (offline message buffer, per-user queue)
media files      → S3 + CDN
user presence    → Redis (online/offline, last_seen with TTL)
```

### Schemas

```sql
-- MySQL: Users
CREATE TABLE users (
    user_id      BIGINT PRIMARY KEY,
    phone_number VARCHAR(20) UNIQUE NOT NULL,
    display_name VARCHAR(100),
    public_key   TEXT,              -- for E2EE key exchange
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- MySQL: Chats (1:1 and group)
CREATE TABLE chats (
    chat_id      BIGINT PRIMARY KEY,
    chat_type    ENUM('direct', 'group') NOT NULL,
    created_at   TIMESTAMP
);

CREATE TABLE chat_members (
    chat_id    BIGINT NOT NULL,
    user_id    BIGINT NOT NULL,
    joined_at  TIMESTAMP,
    PRIMARY KEY (chat_id, user_id)
);
CREATE INDEX idx_user_chats ON chat_members(user_id);  -- "get all chats for user"
```

```
-- Cassandra: Messages (partitioned by chat_id, clustered by msg_id time-ordered)
CREATE TABLE messages (
    chat_id      BIGINT,
    msg_id       BIGINT,          -- Snowflake: encodes timestamp + server_id
    sender_id    BIGINT,
    content      BLOB,            -- encrypted payload (E2EE: server can't read this)
    media_id     UUID,
    msg_type     TEXT,            -- 'text', 'image', 'video', 'voice'
    created_at   TIMESTAMP,
    PRIMARY KEY (chat_id, msg_id)
) WITH CLUSTERING ORDER BY (msg_id DESC);

-- Query: "get last 50 messages in chat 42" = single partition scan → O(1)
SELECT * FROM messages WHERE chat_id = 42 LIMIT 50;
```

```
-- Redis: Offline message queue (per user)
Key:   "msgqueue:{user_id}"
Type:  List (RPUSH to enqueue, LRANGE/LPOP to dequeue)
Value: JSON of message events to deliver when user reconnects

-- Redis: User presence
Key:   "presence:{user_id}"
Type:  String → "online" or "offline"
TTL:   10 seconds (renewed by heartbeat every 5s; expires = offline)
GETSET presence:42 "online"
EXPIRE presence:42 10
```

---

## 🏗️ Infrastructure (I)

### High-Level Architecture

```
    ┌──────────────┐    ┌──────────────┐
    │  Alice's App │    │  Bob's App   │
    └──────┬───────┘    └──────┬───────┘
           │ WebSocket          │ WebSocket
    ┌──────▼───────┐    ┌──────▼───────┐
    │ Chat Server  │    │ Chat Server  │
    │ (Instance A) │    │ (Instance B) │
    └──────┬───────┘    └──────┬───────┘
           │                   │
    ┌──────▼───────────────────▼───────────────────────┐
    │                Message Broker                     │
    │               (Kafka / RabbitMQ)                  │
    │    topic: messages (partitioned by chat_id)       │
    └──────────────────────┬────────────────────────────┘
                           │
         ┌─────────────────▼──────────────────────┐
         │           Message Processor             │
         │   - Persist to Cassandra                │
         │   - Route to recipient's Chat Server    │
         │   - Queue for offline users             │
         │   - Trigger push notifications          │
         └────────────┬───────────────┬────────────┘
                      │               │
              ┌───────▼────┐  ┌───────▼────────┐
              │ Cassandra  │  │  Redis         │
              │(messages)  │  │ (offline queue,│
              │            │  │  presence)     │
              └────────────┘  └────────────────┘
```

### Why WebSocket, Not HTTP?

```
HTTP polling every 1 second:
  - 2B users × 1 req/sec = 2B HTTP requests/sec → not feasible
  - High latency: up to 1 second delay before seeing new message
  - Stateless: server doesn't know if user is online

Long polling:
  - Better: server holds request open until message arrives
  - But: expensive connection per user, complex reconnection logic

WebSocket (✅ WhatsApp's approach):
  - One persistent TCP connection per user
  - Server PUSHES messages to client instantly
  - Client sends messages over same connection
  - Sub-100ms delivery for online users
  - 2B users × 1 TCP connection = manageable (Erlang/BEAM VM handles millions per node)
```

### Chat Server Selection — How Does Alice's Message Reach Bob?

```
Problem: Alice connects to Chat Server A. Bob connects to Chat Server B.
How does Alice's message, which arrives at Server A, get to Bob on Server B?

Solution: Routing via internal pub/sub

  Step 1: Alice sends message to Server A via WebSocket
  Step 2: Server A publishes to Kafka topic "messages" (key = chat_id)
  Step 3: Message Processor consumes from Kafka
  Step 4: Look up: "which Chat Server is Bob currently connected to?"
          → Redis: GET connection:{bob_id} → "server_b:instance_42"
  Step 5: Message Processor pushes to Server B via internal gRPC call
  Step 6: Server B delivers to Bob via his open WebSocket connection

  If Bob is offline:
  Step 4: Redis shows Bob is offline
  Step 5: Enqueue in Redis: RPUSH msgqueue:{bob_id} {message_payload}
  Step 6: Trigger push notification via FCM/APNs
  Step 7: When Bob reconnects → Server drains his msgqueue and delivers all pending messages
```

---

## 📬 The Hard Part: Message Delivery & Ordering

### Delivery Guarantees — The Receipt System

```
A message goes through 4 states. Each state transition generates an event:

  SENT       ─── client sent to server (client-side, shown as ✓)
      ↓
  DELIVERED  ─── server confirmed receipt from recipient's device (✓✓)
      ↓
  READ       ─── recipient opened the chat (🔵✓✓)

Event flow for delivery receipts:
  1. Bob receives message → his client sends ack to Server B
  2. Server B sends "delivered" event to Message Processor
  3. Message Processor routes "delivered" event back to Alice's Server A
  4. Server A pushes "delivered" event to Alice's app → ✓ becomes ✓✓
```

### Message Ordering — The Hard Problem

```
Problem: Alice and Bob send messages at the exact same millisecond.

Without ordering guarantees:
  Alice's screen: "Hi" (Alice) → "Hello" (Bob)
  Bob's screen:   "Hello" (Bob) → "Hi" (Alice)  ← Different order!

Solution: Server-assigned Snowflake IDs
  - Client uses a local "client_msg_id" (UUID) for deduplication
  - Server assigns a "server_msg_id" (Snowflake) which encodes the server timestamp
  - Both clients display messages sorted by server_msg_id
  - Result: same ordering on both sides, guaranteed

  Alice's message: server_msg_id = 1683000000001 (first by 1ms)
  Bob's message:   server_msg_id = 1683000000002
  Both screens show: Alice's message first, then Bob's
```

### Exactly-Once Delivery — No Duplicates

```
Problem: Alice sends message. Network drops. Did server receive it?
  - If Alice retries → duplicate if server DID receive it
  - If Alice doesn't retry → lost message if server didn't receive it

Solution: Idempotency keys
  1. Client assigns a UUID to each message: client_msg_id
  2. On retry, client sends same client_msg_id
  3. Server checks: have I seen this client_msg_id before? (Redis SET NX check)
     YES → ignore duplicate, return existing server_msg_id
     NO  → persist, return new server_msg_id
  4. Client deduplicates display using server_msg_id
```

---

## 👥 Group Messaging

```
WhatsApp groups support up to 1,024 members (limit enforced for performance reasons).

Naive approach (fan-out on write):
  - 1 message → 1,024 delivery events
  - At 1M group messages/sec × 1,024 members = 1 BILLION delivery events/sec 💥

WhatsApp's approach:
  - The SERVER is responsible for delivery fan-out
  - When a message is sent to group chat G:
    1. Message stored ONCE in Cassandra under chat_id = G
    2. Server gets member list for group G (cached in Redis)
    3. Server creates delivery task for each online member
    4. For offline members: adds to their offline queue
    5. Each member's client fetches group message from Cassandra when delivered

  Key insight: Only message ID is fanned out (8 bytes × 1024 members)
               Message content (potentially MBs) is fetched once when read
               This reduces fan-out bandwidth by 1000x for media messages
```

---

## ⚡ Optimization (O)

### Chat Server Connection State

```
Problem: 2B users × 1 WebSocket connection = 2B concurrent TCP connections
         Standard Linux server: ~65K file descriptors (connections) per process

Solution: Erlang/BEAM VM (WhatsApp's actual choice)
  - Erlang was designed for telecom (millions of concurrent connections)
  - Lightweight Erlang processes (2KB stack vs 8MB OS thread)
  - 1 server: 1-2 million concurrent WebSocket connections
  - 1,000 servers to handle 2B users (rough estimate)

Alternative: Go coroutines or Java NIO (Netty)
  - Go: 1 goroutine per connection (8KB stack) → ~125K connections/GB RAM
  - Netty: Non-blocking I/O, thread pool shared across all connections
```

### Media Upload Flow (Large Files)

```
Wrong way: Client sends 50MB video to Chat Server → Chat Server uploads to S3
Problem:   Chat Server becomes memory/bandwidth bottleneck

Right way: Pre-signed URL pattern
  1. Client: POST /v1/media/upload-url → server returns S3 pre-signed URL
  2. Client: PUT <presigned-url> (uploads directly to S3, bypassing Chat Server)
  3. S3 triggers Lambda: resize image, generate thumbnail, extract metadata
  4. Lambda: notifies Chat Server that media_id is ready
  5. Client: sends message with media_id reference (tiny, ~36 bytes)
  6. Recipient: fetches media from S3/CDN when they open message
```

### Message Storage — Retention vs Cost

```
Problem: 100B messages/day × 1KB avg = 100TB/day forever

WhatsApp's approach:
  - Messages stored on Cassandra temporarily (7-30 days) for sync/delivery
  - Long-term storage: messages stored E2EE on DEVICE, not WhatsApp servers
  - WhatsApp's servers only buffer messages until delivered to all recipient devices
  - Once delivered to all devices, server copy can be deleted
  - (This is also how E2EE privacy works — server never stores readable content)
```

---

## 💻 Code Examples

### WebSocket Handler — Receive & Route Message

```java
@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {

    @Autowired private MessageService messageService;
    @Autowired private RedisTemplate<String, String> redis;

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = getUserId(session);
        // Register this connection: map userId → this server + session
        redis.opsForValue().set("connection:" + userId,
                serverInstanceId + ":" + session.getId(), Duration.ofHours(1));
        // Mark user online with TTL (heartbeat will renew)
        redis.opsForValue().set("presence:" + userId, "online", Duration.ofSeconds(15));
        // Drain any pending offline messages
        messageService.deliverOfflineMessages(userId, session);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        MessagePayload payload = objectMapper.readValue(message.getPayload(), MessagePayload.class);
        // Route based on event type
        switch (payload.getType()) {
            case "message"    -> messageService.handleIncomingMessage(payload, session);
            case "receipt"    -> messageService.handleReceipt(payload);
            case "typing"     -> messageService.handleTyping(payload);
            case "heartbeat"  -> renewPresence(getUserId(session));
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String userId = getUserId(session);
        redis.delete("connection:" + userId);
        redis.opsForValue().set("presence:" + userId, "offline");
    }

    private void renewPresence(String userId) {
        redis.expire("presence:" + userId, Duration.ofSeconds(15));
    }
}
```

### Message Service — Send & Fan-Out

```java
@Service
public class MessageService {

    @Autowired private KafkaTemplate<String, MessageEvent> kafka;
    @Autowired private MessageRepository messageRepo;  // Cassandra
    @Autowired private RedisTemplate<String, String> redis;

    public void handleIncomingMessage(MessagePayload payload, WebSocketSession senderSession) {
        String clientMsgId = payload.getClientMsgId();

        // Idempotency check — prevent duplicates on retry
        String idempotencyKey = "msg:dedup:" + payload.getSenderId() + ":" + clientMsgId;
        Boolean isNew = redis.opsForValue().setIfAbsent(idempotencyKey, "1", Duration.ofHours(24));
        if (!Boolean.TRUE.equals(isNew)) {
            // Duplicate — return existing server_msg_id
            sendAck(senderSession, clientMsgId, redis.opsForValue().get(idempotencyKey + ":server_id"));
            return;
        }

        // Generate server-side message ID (Snowflake = time-ordered)
        long serverMsgId = SnowflakeIdGenerator.nextId();
        redis.opsForValue().set(idempotencyKey + ":server_id", String.valueOf(serverMsgId));

        // Persist to Cassandra
        Message msg = Message.builder()
                .chatId(payload.getChatId())
                .msgId(serverMsgId)
                .senderId(payload.getSenderId())
                .content(payload.getEncryptedContent())  // E2EE: we store but can't read
                .msgType(payload.getMsgType())
                .createdAt(Instant.now())
                .build();
        messageRepo.save(msg);

        // Send ack to sender (message is now persisted)
        sendAck(senderSession, clientMsgId, String.valueOf(serverMsgId));

        // Publish for async delivery to recipients
        kafka.send("messages", String.valueOf(payload.getChatId()),
                new MessageEvent(msg));
    }
}
```

---

## 🏢 Industry Examples

| Company | Real-Time Tech | Scale |
|---|---|---|
| **WhatsApp** | Erlang/BEAM, Mnesia/Cassandra, custom XMPP protocol | 100B messages/day, 2B users |
| **Telegram** | Go, MTProto protocol, Cassandra + custom storage | 75B messages/day, 900M users |
| **Signal** | Java/Kotlin, Signal Protocol (E2EE pioneer), PostgreSQL | 40M users, focus on privacy |
| **Facebook Messenger** | Iris (custom pub/sub), HBase, React Native | 100B messages/day |
| **Slack** | Node.js, WebSocket, MySQL + Solr | 1B+ messages/day (enterprise) |

**WhatsApp's engineering philosophy** (Wired interview, 2015):
- "All the complexity lies in delivery guarantees, not in features"
- Erlang was chosen because "we needed millions of simultaneous connections and Erlang has been doing this since the 1980s"
- Their "50 engineers for 900M users" ratio is achieved by using battle-tested technology (Erlang, FreeBSD) rather than custom-built solutions

---

## ⚠️ Common Pitfalls

1. **Using HTTP polling instead of WebSockets** — HTTP is request-response. For real-time chat, you need a persistent bidirectional connection. WebSocket is the industry standard; explain the trade-off versus SSE for read-only streams.

2. **Storing message content on the server long-term** — WhatsApp's key architectural decision is that servers only buffer messages until delivered. After delivery, content is deleted server-side. This enables E2EE and reduces storage costs dramatically.

3. **Naive group fan-out** — Fanning out message content to 1,024 members is expensive. Fan out message references (IDs) and let clients fetch content from storage directly.

4. **Not handling offline delivery** — When Bob reconnects, he must receive all messages sent while offline, in order, without duplicates. This requires a durable per-user message queue with acknowledgment.

5. **Missing idempotency** — Users retry on network failure. Without idempotency keys, duplicate messages appear. Always use client-generated UUIDs that the server can check before persisting.

---

## 🧩 Mini Challenge

**Design question**: WhatsApp's "last seen" feature shows when a user was last online. Design it so it:
1. Updates within 1 minute of the user going offline
2. Scales to 2 billion users
3. Allows users to hide their last seen from non-contacts

<details>
<summary>💡 Click to reveal answer</summary>

**Architecture**:

1. **Tracking presence** — On WebSocket heartbeat (every 5 seconds), renew Redis key: `SETEX presence:{userId} 10 "online"`. When WebSocket closes or TTL expires, user is offline. Last-seen timestamp: `SET last_seen:{userId} {timestamp}` on disconnect.

2. **Scaling to 2B users** — Redis cluster with consistent hashing. At 2B users × ~30 bytes per presence record = 60 GB — fits in a medium Redis cluster. Not all 2B users are active; only DAU (~100M) have active presence keys.

3. **Privacy control** — Store privacy setting in MySQL: `last_seen_visibility = ENUM('everyone', 'contacts', 'nobody')`. When Client A queries "last seen of User B":
   - If B's setting = 'nobody': return null
   - If B's setting = 'contacts': check if A is in B's contact list → MySQL lookup → return timestamp or null
   - If B's setting = 'everyone': return Redis last_seen timestamp directly

4. **Trade-off**: The contact list check adds latency (~5ms MySQL lookup). For 'contacts' privacy, cache the contact list membership in Redis as a Bloom filter to reduce false positives without hitting MySQL for every query.

</details>

---

## 📝 Interview Q&A

**Q: How does WhatsApp implement End-to-End Encryption?**
> A: WhatsApp uses the Signal Protocol. Each user device generates a public/private key pair. The public key is uploaded to WhatsApp's servers. When Alice messages Bob, Alice fetches Bob's public key, encrypts the message locally with it — WhatsApp servers only see ciphertext. Only Bob's device has the private key to decrypt. WhatsApp's servers cannot read messages even if subpoenaed.

**Q: How do you handle message ordering in a group chat when multiple members send simultaneously?**
> A: Each message gets a Snowflake ID assigned by the server upon receipt. All clients display messages sorted by server-assigned ID. This ensures consistent ordering across all group members regardless of network timing. For simultaneous messages (same millisecond), the server's sequence counter within the Snowflake ID determines order.

**Q: What happens to messages if the Kafka broker goes down?**
> A: Kafka is configured with `acks=all` (all in-sync replicas must confirm) and `replication.factor=3`. A single broker failure doesn't interrupt message flow. If the entire Kafka cluster fails, the Chat Server temporarily holds messages in memory and retries with exponential backoff. Messages are considered "in-flight" until they reach Cassandra — the sender sees a "clock" icon until the server confirms persistence.

**Q: How would you support message search ("find all messages containing 'meeting'")?**
> A: This is complex with E2EE — server can't index encrypted content. Two approaches: (1) Client-side search: index all decrypted messages locally using SQLite FTS5 on device. (2) Searchable encryption (advanced): homomorphic encryption that allows searching without decrypting — computationally expensive, not yet practical at WhatsApp's scale. WhatsApp actually does client-side search only.

---

## 🔗 What to Read Next

1. **[Design Netflix](./DesignNetflix.md)** — Apply CDN and streaming concepts at massive media scale
2. **[BuildingBlocks/CircuitBreaker.md](../BuildingBlocks/CircuitBreaker.md)** — Resilience patterns when downstream services fail
3. **[MessagingQ/Kafka.md](../MessagingQ/Kafka.md)** — Deep dive into the message broker powering the async delivery pipeline

---

*[← Design Twitter](./DesignTwitter.md) | [Back to Case Studies](./README.md) | [Next: Design Netflix →](./DesignNetflix.md)*
