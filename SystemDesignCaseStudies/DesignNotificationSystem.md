# 🔔 Design a Notification System
## Push, SMS, Email — Reliable Delivery at 100M/day

> *"Every app you use sends you notifications. But designing a notification system that's reliable, fast, prioritized, rate-limited, and multi-channel — without spamming users or losing critical alerts — is a surprisingly deep distributed systems problem."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Requirements (R)](#-requirements-r)
3. [API Design (A)](#-api-design-a)
4. [Data Model (D)](#-data-model-d)
5. [Infrastructure (I)](#-infrastructure-i)
6. [The Hard Part: Delivery Guarantees](#-the-hard-part-delivery-guarantees)
7. [Optimization (O)](#-optimization-o)
8. [Code Examples](#-code-examples)
9. [Industry Examples](#-industry-examples)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

It's Black Friday. You're running an e-commerce platform. In the next hour you need to:

- Send **50M promotional push notifications** ("Flash Sale starts NOW!")
- Deliver **order confirmation emails** to every purchaser within 30 seconds
- Send **2FA SMS codes** to users logging in — these must arrive in < 10 seconds
- Trigger **real-time price drop alerts** to users watching specific items

Each of these has different urgency, different channels, and different failure behavior:
- If a 2FA SMS fails → user can't log in → **critical**, retry immediately
- If a promotional email fails → user misses a discount → **low priority**, retry later or skip
- If an order confirmation email fails → **medium priority**, retry 3 times with backoff

One system must handle all of these with different SLAs, without one slow channel blocking another.

---

## 📐 Requirements (R)

### Functional Requirements
```
Core features:
  ✅ Send push notifications (iOS APNs, Android FCM)
  ✅ Send SMS via third-party provider (Twilio, SNS)
  ✅ Send transactional emails (SendGrid, SES)
  ✅ Support notification templates with variable substitution
  ✅ User notification preferences (opt-out per channel/category)
  ✅ Delivery status tracking (sent, delivered, failed)
  ✅ Scheduled notifications (send at specific time)
  ✅ Bulk/broadcast notifications (send to all users in a segment)

Out of scope:
  ❌ In-app notification center (separate UI feature)
  ❌ A/B testing for notification content
  ❌ ML-based send time optimization
```

### Non-Functional Requirements
```
Scale:
  - 100M notifications/day across all channels
  - Peak: 10M notifications in 1 hour (marketing blast)
  - 10M registered devices (push tokens)

Performance SLAs by priority:
  CRITICAL  (2FA, password reset)  → delivered in < 10 seconds
  HIGH      (order confirmation)   → delivered in < 30 seconds
  MEDIUM    (social: like, follow) → delivered in < 5 minutes
  LOW       (marketing, promo)     → delivered in < 1 hour

Reliability:
  - CRITICAL notifications: at-least-once delivery
  - LOW notifications: best-effort (can drop if system overloaded)
  - Deduplication: no user should receive same notification twice
```

---

## 🌐 API Design (A)

```
# Send a notification (used by other services internally)
POST   /v1/notifications/send
  Request:  {
    "recipients": ["user_id_1", "user_id_2"],  // OR "segment": "all_users"
    "channels":   ["push", "email"],            // try these channels
    "priority":   "HIGH",                       // CRITICAL|HIGH|MEDIUM|LOW
    "templateId": "order_confirmation",
    "templateVars": { "orderId": "12345", "amount": "$29.99" },
    "scheduledAt": "2026-05-17T20:00:00Z"       // optional, null = send now
  }
  Response: { "notificationId": "uuid", "status": "queued" }

# Check delivery status
GET    /v1/notifications/{notificationId}
  Response: {
    "notificationId": "...",
    "status": "delivered|failed|pending",
    "channel": "push",
    "sentAt": "...",
    "deliveredAt": "..."
  }

# User preferences
GET    /v1/users/{userId}/notification-preferences
PUT    /v1/users/{userId}/notification-preferences
  Request: { "push": true, "email": true, "sms": false, "categories": { "marketing": false } }

# Device token registration (for push)
POST   /v1/users/{userId}/devices
  Request:  { "token": "fcm_token_here", "platform": "android", "deviceId": "uuid" }
```

---

## 🗄️ Data Model (D)

```sql
-- MySQL: Notification templates
CREATE TABLE templates (
    template_id   VARCHAR(50) PRIMARY KEY,  -- 'order_confirmation', '2fa_code'
    channel       VARCHAR(20),              -- 'push', 'email', 'sms'
    subject       VARCHAR(255),             -- for email
    body_template TEXT NOT NULL,            -- "Your order {{orderId}} is confirmed!"
    created_at    TIMESTAMP
);

-- MySQL: User device tokens (for push notifications)
CREATE TABLE device_tokens (
    device_id   UUID PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    platform    VARCHAR(10) NOT NULL,   -- 'ios', 'android', 'web'
    token       TEXT NOT NULL,          -- FCM/APNs token
    is_active   BOOLEAN DEFAULT TRUE,
    updated_at  TIMESTAMP,
    INDEX idx_user_id (user_id)
);

-- MySQL: User notification preferences
CREATE TABLE notification_preferences (
    user_id    BIGINT PRIMARY KEY,
    push_enabled  BOOLEAN DEFAULT TRUE,
    email_enabled BOOLEAN DEFAULT TRUE,
    sms_enabled   BOOLEAN DEFAULT FALSE,
    opt_out_categories JSON  -- {"marketing": false, "transactional": true}
);
```

```
-- Cassandra: Notification log (write-heavy, status tracking)
CREATE TABLE notification_log (
    notification_id  UUID,
    user_id          BIGINT,
    channel          TEXT,
    template_id      TEXT,
    status           TEXT,   -- 'queued'|'sent'|'delivered'|'failed'
    sent_at          TIMESTAMP,
    delivered_at     TIMESTAMP,
    failure_reason   TEXT,
    PRIMARY KEY (notification_id, user_id)
);
-- Query by user: CREATE INDEX IF NOT EXISTS ON notification_log(user_id);
```

---

## 🏗️ Infrastructure (I)

### High-Level Architecture

```
 Other Services (Order Service, Auth Service, etc.)
             │
             │ POST /v1/notifications/send
             ▼
 ┌───────────────────────────────────────────────┐
 │           NOTIFICATION SERVICE                │
 │  - Validate request                           │
 │  - Check user preferences (opt-out check)     │
 │  - Render template (substitute variables)     │
 │  - Route to priority queue                    │
 └──────┬──────────────┬────────────────┬────────┘
        │              │                │
        ▼              ▼                ▼
  [CRITICAL Q]   [HIGH Q]          [MEDIUM/LOW Q]
  (Kafka topic)  (Kafka topic)     (Kafka topic)
        │              │                │
        ▼              ▼                ▼
 ┌──────────┐  ┌──────────┐     ┌──────────────┐
 │  Push    │  │  Email   │     │  SMS         │
 │ Worker   │  │ Worker   │     │  Worker      │
 │(FCM/APNs)│  │(SendGrid)│     │  (Twilio)    │
 └────┬─────┘  └────┬─────┘     └──────┬───────┘
      │              │                  │
      └──────────────▼──────────────────┘
                     │ delivery callbacks
 ┌───────────────────▼───────────────────────────┐
 │          NOTIFICATION LOG (Cassandra)         │
 │  status: queued → sent → delivered/failed     │
 └───────────────────────────────────────────────┘
```

### Priority Queue Architecture

```
Why separate queues per priority?

Problem: Without prioritization, a 10M marketing blast will occupy all workers,
         causing 2FA SMS (critical) to queue behind 10M promotional emails.
         User can't log in for 30 minutes. This is unacceptable.

Solution: Separate Kafka topics per priority level

  Topic: notifications.critical   → 20 partitions, 50 workers
  Topic: notifications.high       → 10 partitions, 20 workers
  Topic: notifications.medium     → 5 partitions, 10 workers
  Topic: notifications.low        → 5 partitions, 10 workers

Workers consume from critical queue FIRST.
If critical queue is empty, also consume from high queue, etc.
Critical gets dedicated worker pool — never blocked by low-priority work.
```

### Channel Workers — External Provider Integration

```
Push Worker:
  - Android: Firebase Cloud Messaging (FCM)
    POST https://fcm.googleapis.com/v1/projects/{project}/messages:send
    Payload: { token: "device_token", notification: { title, body } }

  - iOS: Apple Push Notification Service (APNs)
    HTTP/2 POST https://api.push.apple.com/3/device/{device_token}
    Payload: { aps: { alert: { title, body }, badge: 1, sound: "default" } }

SMS Worker:
  - Twilio API: POST https://api.twilio.com/2010-04-01/Accounts/{SID}/Messages.json
  - Vonage (Nexmo) as fallback if Twilio fails (circuit breaker)

Email Worker:
  - SendGrid API or AWS SES
  - HTML + text versions of templates
  - Unsubscribe link required (CAN-SPAM compliance)
```

---

## 📬 The Hard Part: Delivery Guarantees

### Challenge 1: Invalid/Stale Device Tokens

```
Problem: User uninstalls app → device token becomes invalid
         Next push to that token → FCM returns 404

Solution: Token hygiene
  1. On FCM/APNs 404 error → mark token as inactive in DB
     UPDATE device_tokens SET is_active = FALSE WHERE token = '...'
  2. Batch cleanup job (weekly): delete inactive tokens older than 30 days
  3. Token refresh: when app opens, re-register current token
     (handles case where token changes after app reinstall)
```

### Challenge 2: Deduplication

```
Problem: Notification Service crashes after writing to Kafka but before updating DB.
         On restart, same notification is sent twice.

Solution: Idempotency key
  1. Assign notification_id (UUID) when notification first received
  2. Before writing to Kafka: SET notification:{notification_id} "queued" NX EX 86400
     (SET if Not eXists — atomic Redis operation)
  3. If key already exists → duplicate request, skip
  4. Workers also check before sending: has this notification_id been processed?

  Redis key TTL of 24 hours handles most duplicates without unbounded storage.
```

### Challenge 3: Retry Strategy

```
Different channels need different retry behavior:

CRITICAL (2FA SMS):
  - Max 5 retries
  - Exponential backoff: 1s, 2s, 4s, 8s, 16s
  - If all fail → alert on-call engineer + try fallback channel (email)
  - Dead-letter queue (DLQ) for manual inspection

HIGH (Order confirmation):
  - Max 3 retries: 30s, 5min, 30min
  - If all fail → write to DLQ, log failure

LOW (Marketing email):
  - Max 1 retry after 1 hour
  - If fails → drop (not worth resending days-old promo email)
  - Log for analytics (delivery rate tracking)
```

### Challenge 4: Rate Limiting Per User

```
Problem: System sends 20 push notifications in 5 minutes to same user → spammy

Solution: Per-user rate limiting with Redis
  key = "notif_rate:{user_id}:{channel}:{minute_bucket}"
  val = count
  TTL = 2 minutes (sliding window)

  Before sending to user:
    count = INCR "notif_rate:42:push:2026051720"
    EXPIRE "notif_rate:42:push:2026051720" 120
    if count > 5: drop or defer notification

  Different limits per channel:
    Push:  5/minute, 20/hour, 50/day
    SMS:   2/minute, 10/hour (more expensive, more intrusive)
    Email: 3/minute, 10/hour, 50/day
```

---

## ⚡ Optimization (O)

### Bulk Fan-Out for Marketing Blasts

```
Scenario: Marketing team sends a push to all 50M users

Wrong approach: Loop through 50M users, send one notification each → takes hours

Right approach: Segment-based fan-out
  1. Marketing creates a "segment" (e.g., "users active in last 30 days")
  2. Notification Service receives: { segment: "active_30d", template: "flash_sale" }
  3. Fan-out Service queries user segment in batches (1,000 users/batch)
  4. For each batch → write 1,000 messages to Kafka (low priority queue)
  5. Workers consume at sustainable rate — respects throttling without overwhelming providers

Time estimate: 50M users ÷ 10,000 push/sec worker throughput = 83 minutes
(Acceptable for "Flash Sale starts NOW" — you pre-schedule 2 hours ahead)
```

### Multi-Provider Failover

```
Problem: FCM goes down for 30 minutes (it has happened: March 2019, April 2020)

Solution: Multi-provider strategy with circuit breaker

  try:
      fcm.send(token, message)  // primary
  except ProviderException:
      if circuit_breaker.is_open("fcm"):
          apns_bridge.send(...)  // fallback for iOS (different path to same device)
      raise  // trigger retry with exponential backoff

  // Circuit breaker: after 50 failures in 60s → OPEN for 5 min
  // After 5 min → HALF_OPEN, try one request, if success → CLOSE
```

---

## 💻 Code Examples

### Notification Service — Route to Priority Queue

```java
@Service
public class NotificationService {

    @Autowired private UserPreferencesRepository prefsRepo;
    @Autowired private TemplateEngine templateEngine;
    @Autowired private KafkaTemplate<String, NotificationMessage> kafka;
    @Autowired private RedisTemplate<String, String> redis;

    public String sendNotification(NotificationRequest request) {
        String notificationId = UUID.randomUUID().toString();

        // Idempotency: prevent duplicate sends
        Boolean isNew = redis.opsForValue()
                .setIfAbsent("notif:" + notificationId, "queued", Duration.ofDays(1));
        if (!Boolean.TRUE.equals(isNew)) {
            return notificationId; // already queued
        }

        // Process each recipient
        for (String userId : request.getRecipients()) {
            // Check user's opt-out preferences
            UserPreferences prefs = prefsRepo.findByUserId(userId);
            List<String> allowedChannels = filterChannels(request.getChannels(), prefs);

            if (allowedChannels.isEmpty()) continue;  // user opted out of everything

            // Render template for this user
            String renderedBody = templateEngine.render(
                    request.getTemplateId(), request.getTemplateVars());

            // Build message
            NotificationMessage message = NotificationMessage.builder()
                    .notificationId(notificationId)
                    .userId(userId)
                    .channels(allowedChannels)
                    .body(renderedBody)
                    .priority(request.getPriority())
                    .scheduledAt(request.getScheduledAt())
                    .build();

            // Route to appropriate Kafka topic based on priority
            String topic = "notifications." + request.getPriority().toLowerCase();
            kafka.send(topic, userId, message);
        }

        return notificationId;
    }

    private List<String> filterChannels(List<String> requested, UserPreferences prefs) {
        return requested.stream()
                .filter(ch -> switch (ch) {
                    case "push"  -> prefs.isPushEnabled();
                    case "email" -> prefs.isEmailEnabled();
                    case "sms"   -> prefs.isSmsEnabled();
                    default      -> false;
                })
                .collect(Collectors.toList());
    }
}
```

### Push Worker — FCM with Retry

```java
@KafkaListener(topics = "notifications.critical", groupId = "push-worker")
public void handleCriticalPush(NotificationMessage message) {
    if ("push".equals(message.getChannel())) {
        sendPushWithRetry(message, 5, Duration.ofSeconds(1));
    }
}

@Retryable(value = PushDeliveryException.class, maxAttempts = 5,
           backoff = @Backoff(delay = 1000, multiplier = 2))
public void sendPushWithRetry(NotificationMessage message, int maxAttempts, Duration initialDelay) {
    String userId = message.getUserId();
    List<DeviceToken> tokens = deviceTokenRepo.findActiveByUserId(userId);

    for (DeviceToken device : tokens) {
        try {
            FcmResponse response = fcmClient.send(FcmMessage.builder()
                    .token(device.getToken())
                    .notification(Notification.builder()
                            .title(message.getTitle())
                            .body(message.getBody())
                            .build())
                    .build());

            // Log success
            notificationLog.updateStatus(message.getNotificationId(),
                    userId, "delivered", Instant.now());

        } catch (InvalidTokenException e) {
            // Token is stale — deactivate it
            deviceTokenRepo.deactivate(device.getToken());
        } catch (PushDeliveryException e) {
            // Transient failure — will be retried by @Retryable
            notificationLog.updateStatus(message.getNotificationId(), userId, "failed",
                    Instant.now(), e.getMessage());
            throw e;
        }
    }
}
```

---

## 🏢 Industry Examples

| Company | Stack | Scale |
|---|---|---|
| **Facebook** | Iris (internal), MCQ (push framework), async fan-out | 10B+ notifications/day |
| **Twitter** | Pushd (internal service), Kafka fan-out | 1B+ notifications/day |
| **Uber** | Cherami (internal queue), multi-channel with priority routing | Real-time driver/rider alerts |
| **Airbnb** | AWS SNS + SQS, SendGrid, Twilio | 10M+ emails/day, multi-region |
| **LinkedIn** | Nemo notification platform | 5B+ notifications/day across channels |

**Common commercial choices**:
- Push: **Firebase FCM** (Android/Web), **Apple APNs** (iOS)
- Email: **AWS SES** (cheap, reliable), **SendGrid** (better analytics)
- SMS: **Twilio** (developer-friendly), **AWS SNS** (integrated), **Vonage**
- Orchestration: **AWS SNS + SQS** (managed fan-out to multiple channels)

---

## ⚠️ Common Pitfalls

1. **Single queue for all notification types** — A marketing blast of 10M emails will block 2FA SMS codes for minutes. Always use priority queues with dedicated workers for critical notifications.

2. **Not handling stale tokens** — FCM/APNs return error codes for invalid tokens. Without token cleanup, you'll accumulate millions of stale tokens, wasting processing and money.

3. **No rate limiting per user** — Without per-user rate limiting, automated events can spam a single user (e.g., 100 social notifications in 1 minute). This leads to user opt-outs and app uninstalls.

4. **Synchronous notification sending** — Never send notifications synchronously in the request path. Always fire-and-forget to a queue. The calling service (Order Service) should not wait for notification delivery.

5. **No opt-out mechanism** — CAN-SPAM (email) and GDPR require easy opt-out. Always check user preferences before sending. Missing this is both a UX issue and a legal compliance risk.

---

## 🧩 Mini Challenge

**Design question**: A user signs up for price drop alerts on 50 products. When any product's price drops, they get a push notification within 5 minutes. 1 million users have set up such alerts. 100K products can have price changes per day. Design the alert triggering system.

<details>
<summary>💡 Click to reveal answer</summary>

**The challenge**: When a product's price drops, you need to efficiently find which users are watching it and trigger notifications — without querying the database on every price change.

**Data model**:
```
Redis: alert_subscribers:{product_id} → SET of user_ids
  SADD alert_subscribers:42  user_1, user_2, user_99
  SMEMBERS alert_subscribers:42  → returns all watchers

(1M users × 50 products = 50M entries — ~400MB in Redis, very manageable)
```

**Flow**:
1. Price Service detects price drop → publishes `price_dropped` event to Kafka
2. Alert Processor (Kafka consumer):
   a. `SMEMBERS alert_subscribers:{productId}` → get list of watchers (could be 50K users)
   b. For each watcher: check threshold (user set "alert if price drops below $X") → filter
   c. Batch remaining users into groups of 1,000 → enqueue to `notifications.medium` topic
3. Notification Service delivers push notifications over 5 minutes (respecting rate limits)

**Scale math**: 
- 100K price changes/day = ~1.2/second
- Average 10K watchers per product → 12K notifications/second → easily within push worker capacity (FCM handles millions/second)
- 5-minute delivery window = comfortable buffer, no real-time pressure

</details>

---

## 📝 Interview Q&A

**Q: How do you handle notification delivery when the user's device is offline?**
> A: FCM and APNs both support message queuing at the platform level. FCM stores undelivered messages for up to 4 weeks (with TTL). When the device comes online, messages are delivered. For time-sensitive notifications (2FA), set a short TTL (60 seconds) — if device isn't online within 60 seconds, the push is expired and fallback to SMS/email is triggered.

**Q: How would you implement a "Do Not Disturb" mode for notifications?**
> A: Store per-user DND schedules in MySQL: `{ user_id, dnd_start: "22:00", dnd_end: "08:00", timezone: "America/New_York" }`. Before enqueuing a non-critical notification, check if current time (in user's timezone) is within DND window. If yes: for LOW/MEDIUM → delay to end of DND window (update `scheduledAt`). For CRITICAL (2FA) → bypass DND and send immediately.

**Q: How do you track notification delivery rates and debug failed notifications?**
> A: FCM/APNs provide delivery receipts via callback webhooks or polling their APIs. Store delivery status in Cassandra (notification_id, user_id, channel, status, timestamp). Dashboard: query Cassandra for delivery rates by channel/template/time. Alerts: if push delivery rate drops below 95% in a 5-minute window → PagerDuty alert. For debugging: dead-letter queue (DLQ) in Kafka for all failed notifications — SREs can inspect and manually replay.

**Q: How would you design notification batching? ("You have 5 new likes" vs 5 separate "X liked your post")**
> A: Batching service aggregates notifications of the same type within a time window (e.g., 30 seconds): instead of sending 5 individual "X liked" pushes, wait 30 seconds, count them, send "5 people liked your post." Implementation: Redis counter + delayed Kafka message. `INCR notif_batch:{user_id}:{type}:{window}` — when window expires (via Redis TTL trigger or scheduled job), flush the batch count into one notification.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/MessageQueues.md](../BuildingBlocks/MessageQueues.md)** — Deep dive into Kafka for the async notification pipeline
2. **[BuildingBlocks/RateLimiting.md](../BuildingBlocks/RateLimiting.md)** — Per-user rate limiting implementation details
3. **[Security/JWT_Deep_Dive.md](../Security/JWT_Deep_Dive.md)** — How 2FA tokens and secure notification auth works

---

*[← Design Uber](./DesignUber.md) | [Back to Case Studies](./README.md)*
