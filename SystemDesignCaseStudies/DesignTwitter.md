# 🐦 Design Twitter / X
## Feed Generation, Fan-Out, and Scaling to 300M Users

> *"Twitter's feed algorithm is deceptively simple on the surface — 'show me tweets from people I follow, newest first' — but behind the scenes it's one of the most complex distributed systems ever built."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [URL Shortener](./DesignURLShortener.md), [BuildingBlocks/](../BuildingBlocks/)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Requirements (R)](#-requirements-r)
3. [API Design (A)](#-api-design-a)
4. [Data Model (D)](#-data-model-d)
5. [Infrastructure (I)](#-infrastructure-i)
6. [The Hard Part: Feed Generation](#-the-hard-part-feed-generation)
7. [Optimization (O)](#-optimization-o)
8. [Code Examples](#-code-examples)
9. [Industry Examples](#-industry-examples)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

It's 2:00 AM. A celebrity with **10 million followers** posts a tweet.

Within seconds, all 10 million followers should see it in their timeline.

The math is brutal:
```
10M followers × 1 tweet = 10M feed inserts needed
At Twitter's scale: 500M tweets/day × avg 200 followers = 100 BILLION feed insertions/day
= 1.16 MILLION feed inserts per second... sustained
```

Now multiply by likes, retweets, replies. Twitter processes **~650K events/sec** at peak.

**The core tension**:
- ✅ Option A: Pre-compute feeds (fast reads, slow/expensive writes)
- ✅ Option B: Compute feeds at read time (slow reads, cheap writes)
- ✅ Option C: Twitter's actual answer — a hybrid of both

This case study explains how Twitter solved it.

---

## 📐 Requirements (R)

### Functional Requirements
```
Core features:
  ✅ Post a tweet (text ≤280 chars, optional media)
  ✅ Follow / unfollow users
  ✅ View home timeline (tweets from people you follow, newest first)
  ✅ View user timeline (all tweets by a specific user)
  ✅ Like and retweet
  ✅ Mentions / @replies

Out of scope:
  ❌ DMs (separate system)
  ❌ Trending / search (separate service)
  ❌ Ads
  ❌ Notifications (covered in DesignNotificationSystem.md)
```

### Non-Functional Requirements
```
Scale:
  - 300M DAU, 1.5B total accounts
  - 500M tweets/day written (~6K/sec avg, ~50K/sec peak)
  - 50B timeline reads/day (~580K/sec avg, ~5M/sec peak)
  - Read:Write = ~100:1 (extremely read-heavy)

Performance:
  - Home timeline load: < 200ms (P99)
  - Tweet posting latency: < 500ms

Reliability:
  - 99.99% availability for timeline reads
  - Eventual consistency acceptable: tweets can appear up to 5 seconds after posting

Storage:
  - 1 tweet = ~300 bytes (text) + avg 50KB (media, separate)
  - 500M tweets/day × 300 bytes = 150 GB/day text
  - Media: ~25 PB/year (stored separately in object storage)
```

---

## 🌐 API Design (A)

```
# Tweet operations
POST   /v1/tweets
  Request:  { "text": "Hello World", "mediaIds": ["id1"] }
  Response: { "tweetId": "...", "createdAt": "...", "authorId": "..." }

DELETE /v1/tweets/{tweetId}
  Response: 204 No Content

GET    /v1/tweets/{tweetId}
  Response: { Tweet object with like_count, retweet_count, author }

# Feed
GET    /v1/home_timeline?cursor={cursor}&count={count}
  Response: { "tweets": [...], "nextCursor": "..." }

GET    /v1/users/{userId}/tweets?cursor={cursor}
  Response: { "tweets": [...], "nextCursor": "..." }

# Social graph
POST   /v1/users/{userId}/follow
POST   /v1/users/{userId}/unfollow
GET    /v1/users/{userId}/followers?cursor={cursor}
GET    /v1/users/{userId}/following?cursor={cursor}

# Engagement
POST   /v1/tweets/{tweetId}/like
DELETE /v1/tweets/{tweetId}/like
POST   /v1/tweets/{tweetId}/retweet
```

**Pagination**: Use cursor-based (not page-based) pagination. A cursor encodes the last seen `tweetId` + `createdAt`. This handles new tweets being inserted while user is scrolling without duplicates.

---

## 🗄️ Data Model (D)

### Storage Choices

```
tweets table      → Cassandra (write-heavy, time-series, wide rows)
users table       → MySQL/PostgreSQL (relational, ACID for user data)
social graph      → MySQL or dedicated graph store (follows = simple edges)
media files       → S3 (object storage, CDN-served)
home timeline     → Redis (pre-computed, sorted set per user)
tweet metadata    → Redis (like count, retweet count — frequently updated)
```

### Schemas

```sql
-- MySQL: Users
CREATE TABLE users (
    user_id      BIGINT       PRIMARY KEY,
    username     VARCHAR(50)  UNIQUE NOT NULL,
    display_name VARCHAR(100),
    bio          TEXT,
    follower_count  INT DEFAULT 0,
    following_count INT DEFAULT 0,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- MySQL: Social graph (follows)
CREATE TABLE follows (
    follower_id  BIGINT NOT NULL,
    followee_id  BIGINT NOT NULL,
    created_at   TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (follower_id, followee_id)
);
CREATE INDEX idx_followee ON follows(followee_id);  -- "who follows me?"
```

```
-- Cassandra: Tweets (optimized for write throughput + time-range scans)
CREATE TABLE tweets (
    tweet_id    BIGINT,        -- Snowflake ID (encodes timestamp)
    user_id     BIGINT,
    text        TEXT,
    media_urls  LIST<TEXT>,
    created_at  TIMESTAMP,
    PRIMARY KEY (user_id, tweet_id)  -- partition by user, cluster by tweet
) WITH CLUSTERING ORDER BY (tweet_id DESC);

-- Query "get latest tweets by user" = single partition scan → fast
```

```
-- Redis: Home timeline (per user, sorted set)
Key:   "timeline:{user_id}"
Value: Sorted Set where score = tweet_id (encodes timestamp), member = tweet_id

ZADD timeline:42 1683000000001 tweet_id_1
ZADD timeline:42 1683000000002 tweet_id_2
ZREVRANGE timeline:42 0 19  → most recent 20 tweet IDs

-- Redis: Tweet engagement counters
HSET tweet:stats:12345  likes 1024  retweets 200  replies 89
HINCRBY tweet:stats:12345 likes 1
```

---

## 🏗️ Infrastructure (I)

### High-Level Architecture

```
                        ┌──────────────────────┐
                        │      Client App       │
                        └──────────┬───────────┘
                                   │ HTTPS
                        ┌──────────▼───────────┐
                        │  CDN (CloudFront)    │
                        │  + DDoS Protection   │
                        └──────────┬───────────┘
                                   │
                        ┌──────────▼───────────┐
                        │    API Gateway        │
                        │  (Auth + Rate Limit)  │
                        └──┬────────────────┬──┘
                           │                │
              ┌────────────▼───┐        ┌───▼────────────┐
              │  Tweet Service │        │  Feed Service  │
              │  (write path)  │        │  (read path)   │
              └────────┬───────┘        └───────┬────────┘
                       │                        │
           ┌───────────▼──────┐    ┌────────────▼────────┐
           │  Kafka (events)  │    │  Redis Timeline     │
           │  tweet_created   │    │  Cache (per user)   │
           └───────────┬──────┘    └────────────┬────────┘
                       │                        │ cache miss
           ┌───────────▼──────────────────────────────────┐
           │          Fan-out Service                      │
           │  (reads followers, writes to timeline caches) │
           └───────────────────┬──────────────────────────┘
                               │
               ┌───────────────▼───────────────┐
               │     Cassandra (Tweets)         │
               │     MySQL (Users + Follows)    │
               │     S3 (Media)                 │
               └───────────────────────────────┘
```

---

## 🔥 The Hard Part: Feed Generation

This is the most commonly asked deep-dive in Twitter interviews. Three approaches:

### Approach 1: Fan-Out on Write (Push Model)
```
When Alice posts a tweet:
  1. Save tweet to Cassandra
  2. Look up all of Alice's followers (could be 10M)
  3. For each follower: insert tweet_id into their Redis timeline sorted set
  4. Timeline read = just read from Redis sorted set → FAST

Pros:  Reads are extremely fast (just a Redis lookup)
Cons:  Celebrity with 10M followers → 10M Redis writes on one tweet
       If a user follows 5,000 people all posting simultaneously → huge write storm
       Storage: each user has a timeline cache (~800 tweet IDs × 8 bytes = 6.4 KB each)
```

### Approach 2: Fan-Out on Read (Pull Model)
```
When Bob opens his timeline:
  1. Fetch Bob's list of 500 followees
  2. For each followee: query Cassandra for their last N tweets
  3. Merge all results, sort by timestamp, return top 20

Pros:  Write is always cheap (just save the tweet, nothing else)
       Storage efficient: no pre-computed timelines
Cons:  Reading timeline requires N Cassandra queries (N = number of followees)
       Bob follows 500 people → 500 parallel DB queries on every timeline refresh
       At 580K reads/sec × 500 queries = 290 BILLION queries/sec → IMPOSSIBLE
```

### Approach 3: Twitter's Hybrid (Best of Both)
```
Rule: fan-out on write for regular users, fan-out on read for celebrities

Threshold: > 1M followers = "celebrity" (Katy Perry, Obama, etc.)

When Alice (500 followers) posts:
  → Fan-out on write: push to all 500 follower timelines in Redis

When Elon (100M followers) posts:
  → Do NOT fan-out: just save tweet to Cassandra + mark as "celebrity tweet"

When Bob reads his timeline:
  → Read pre-computed Redis timeline (gets tweets from regular users he follows)
  → ALSO check: do I follow any celebrities? Fetch their last 20 tweets from Cassandra
  → Merge both result sets, deduplicate, sort by timestamp
  → Return merged timeline

Result:
  Read time: 1 Redis read + K Cassandra reads (K = num celebrities followed, usually < 5)
  Write time: push to followers list (excluding celebrities' followers)
```

### Timeline Assembly

```
┌─────────────────────────────────────────────────────────────┐
│                    Feed Service                              │
│                                                             │
│  1. ZREVRANGE timeline:{userId} 0 99   → 100 tweet IDs     │
│  2. MGET tweet:{id1}, tweet:{id2}, ... → bulk fetch objects │
│     (from Redis tweet cache, or Cassandra on miss)          │
│  3. For each celebrity followed:                            │
│     SELECT * FROM tweets WHERE user_id=? LIMIT 20           │
│  4. Merge + sort all results by tweet_id DESC               │
│  5. Apply business logic (mute, block, sensitive content)   │
│  6. Return paginated result                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚡ Optimization (O)

### Problem 1: Unfollow/Follow Timeline Consistency

```
Scenario: Bob unfollows Alice. Alice's old tweets are still in Bob's Redis timeline.

Solution: Lazy timeline reconstruction
  - Don't remove tweets from Redis on unfollow (expensive reverse lookup)
  - At read time: filter out tweet authors Bob no longer follows
  - Periodically (daily) compact timeline cache to remove unfollowed accounts
```

### Problem 2: Timeline Cache Eviction for Inactive Users

```
Problem: Pre-computing timelines for 1.5B accounts = 1.5B Redis keys

Solution: Only maintain cache for active users
  - Cache timeline only if user logged in within 30 days
  - On login after 30+ days: re-build timeline from Cassandra (cold-start)
  - ~300M DAU out of 1.5B total → only 20% need active timeline caches
  - Storage: 300M × 6.4 KB ≈ 1.9 TB Redis — manageable across Redis cluster
```

### Problem 3: Like Count Consistency

```
Problem: 50K people like a viral tweet simultaneously → 50K HINCRBY calls on one Redis key → hotspot

Solution: Counter batching
  - Buffer like events in Kafka for 1 second
  - Batch: HINCRBY tweet:stats:12345 likes 847  (one Redis call per second, not 847)
  - Persist to Cassandra async: UPDATE tweet_stats SET likes=likes+847 WHERE tweet_id=12345
```

### Problem 4: Search and Trending

```
Out of scope for basic Twitter design, but if asked:
  - Index tweets in Elasticsearch (async via Kafka consumer)
  - Trending: count tweet mentions of hashtags in 15-minute sliding window
  - Use Redis sorted set: ZINCRBY trending:hashtags 1 "#WorldCup" per occurrence
  - Top 10 trending: ZREVRANGE trending:hashtags 0 9
```

---

## 💻 Code Examples

### Tweet Service — Post Tweet

```java
@Service
public class TweetService {

    @Autowired private TweetRepository tweetRepository;      // Cassandra
    @Autowired private KafkaTemplate<String, TweetEvent> kafka;
    @Autowired private RedisTemplate<String, Object> redis;
    @Autowired private MediaService mediaService;

    public Tweet postTweet(Long userId, String text, List<String> mediaIds) {
        // Validate
        if (text == null || text.length() > 280) {
            throw new BadRequestException("Tweet must be 1-280 characters");
        }

        // Generate Snowflake tweet ID
        long tweetId = SnowflakeIdGenerator.nextId();

        // Process media (upload validation, thumbnail generation)
        List<String> mediaUrls = mediaService.processMedia(mediaIds);

        // Persist to Cassandra
        Tweet tweet = Tweet.builder()
                .tweetId(tweetId)
                .userId(userId)
                .text(text)
                .mediaUrls(mediaUrls)
                .createdAt(Instant.now())
                .build();
        tweetRepository.save(tweet);

        // Publish event for async fan-out
        kafka.send("tweet.created", String.valueOf(userId),
                new TweetEvent(tweetId, userId, Instant.now()));

        return tweet;
    }
}
```

### Fan-Out Service — Consume & Distribute

```java
@KafkaListener(topics = "tweet.created", groupId = "fanout-service")
public void handleTweetCreated(TweetEvent event) {
    long userId = event.getUserId();

    // Check if user is a celebrity (>1M followers)
    long followerCount = userService.getFollowerCount(userId);
    if (followerCount > 1_000_000) {
        // Don't fan-out — feed service will fetch at read time
        return;
    }

    // Get all followers (paginated to avoid memory explosion)
    userService.streamFollowers(userId, followerCount).forEach(followerId -> {
        // Push tweet_id to follower's timeline sorted set
        // Score = tweetId (Snowflake IDs are time-ordered, so this sorts correctly)
        String timelineKey = "timeline:" + followerId;
        redis.opsForZSet().add(timelineKey, event.getTweetId(), event.getTweetId());

        // Keep timeline bounded to last 800 tweets
        redis.opsForZSet().removeRange(timelineKey, 0, -801);
    });
}
```

### Feed Service — Assemble Timeline

```java
@Service
public class FeedService {

    public List<Tweet> getHomeTimeline(Long userId, Long cursor, int count) {
        String timelineKey = "timeline:" + userId;

        // 1. Fetch tweet IDs from Redis sorted set (newest first)
        Set<Object> tweetIds;
        if (cursor == null) {
            tweetIds = redis.opsForZSet().reverseRange(timelineKey, 0, count - 1);
        } else {
            tweetIds = redis.opsForZSet().reverseRangeByScore(
                    timelineKey, Double.NEGATIVE_INFINITY, cursor - 1, 0, count);
        }

        // 2. Bulk-fetch tweet objects (from Redis cache or Cassandra)
        List<Tweet> tweets = tweetCacheService.bulkFetch(tweetIds);

        // 3. Fetch tweets from celebrities the user follows (fan-out on read)
        List<Long> celebrities = followService.getCelebrityFollowees(userId);
        for (Long celebId : celebrities) {
            List<Tweet> celebTweets = tweetRepository.findByUserId(celebId, count);
            tweets.addAll(celebTweets);
        }

        // 4. Merge, sort by tweetId (time-ordered), deduplicate, truncate
        return tweets.stream()
                .distinct()
                .sorted(Comparator.comparingLong(Tweet::getTweetId).reversed())
                .limit(count)
                .collect(Collectors.toList());
    }
}
```

---

## 🏢 Industry Examples

| Company | Fan-Out Strategy | Notes |
|---|---|---|
| **Twitter** | Hybrid (push for regular, pull for celebrities) | Uses Flock for social graph, Manhattan for tweets, Redis for timeline cache |
| **Instagram** | Fan-out on write for feed | Uses Cassandra, Redis, TAO graph DB |
| **Facebook** | Fan-out on read (News Feed) | TAO graph, memcached tier, custom ranking algorithm |
| **LinkedIn** | Fan-out on write | Feed is personalized with ML ranking layer on top |

**Twitter's actual stack** (publicly documented):
- **Flock**: Social graph store (follower/following data), custom built
- **Manhattan**: Distributed key-value store for tweets
- **Firehose**: Internal pub/sub for real-time tweet streaming
- **Heron**: Stream processing (successor to Storm) for analytics
- **Redis**: Timeline cache (called "Early Bird" internally)

---

## ⚠️ Common Pitfalls

1. **Only designing fan-out on write** — This works for regular users but explodes for celebrities. Always mention the celebrity problem and the hybrid approach.

2. **Using SQL for tweets** — A high-write table with hundreds of billions of rows in MySQL will become a bottleneck. Cassandra's write-optimized LSM-tree storage is purpose-built for this.

3. **Storing full tweet objects in the timeline** — Only store `tweet_id` in the timeline cache. Fetch tweet content separately. This allows tweet edits/deletions without cache invalidation of every timeline.

4. **Not using cursor-based pagination** — Page-based pagination (`LIMIT 20 OFFSET 100`) breaks when new tweets are inserted. A cursor encodes the last-seen tweet ID, so it's stable across inserts.

5. **Ignoring the unfollow problem** — After unfollowing, pre-computed timeline caches still contain the unfollowed user's tweets. Handle this at read time, not write time, to avoid an expensive reverse-lookup cascade.

---

## 🧩 Mini Challenge

**Design question**: Twitter is launching a "Twitter Spaces" feature (live audio rooms). What new components do you add to the existing architecture? Focus on the real-time broadcasting aspect.

<details>
<summary>💡 Click to reveal answer</summary>

**Key new components needed**:

1. **WebRTC / WebSocket server** — Real-time audio streaming requires persistent connections. Use WebRTC for peer-to-peer audio between speakers, and WebSocket for signaling (room join/leave events). AWS Kinesis Video Streams or Agora.io can handle the audio infrastructure.

2. **Presence Service** — Track who is currently in a Space (connected, speaking, listening). Use Redis pub/sub with heartbeat: clients send `PING` every 5 seconds, server marks disconnected after 15 seconds. Store listener list in Redis sorted set by join time.

3. **Fan-out for live notifications** — When a Space starts (host has > 10K followers), notify followers. Use existing tweet_created Kafka pipeline — "Space started" is just a special tweet type. Mobile push notifications via APNs/FCM for immediate delivery.

4. **Recording + replay** — Store audio stream to S3 in real-time using chunked HLS segments. After Space ends, merge chunks and publish as playback URL.

**Why this is a good answer**: You reused existing components (Kafka, notification pipeline) rather than reinventing everything, identified the genuinely new requirements (real-time bidirectional audio = WebRTC), and handled the scale concern (fan-out for popular hosts).

</details>

---

## 📝 Interview Q&A

**Q: How does Twitter handle retweets in the feed?**
> A: A retweet is stored as a new tweet entity with `retweet_of = original_tweet_id`. It goes through the same fan-out pipeline as a regular tweet. At display time, the feed service fetches the original tweet object for the retweeted content. The retweet entry in the timeline just records "User X retweeted tweet Y at time T."

**Q: How would you implement the "chronological vs algorithmic" feed toggle?**
> A: Store two separate sorted sets per user in Redis: `timeline:chronological:{userId}` (sorted by tweet_id = time) and `timeline:ranked:{userId}` (sorted by ML score). The fan-out service writes to both. The feed API accepts a `mode` parameter to select which sorted set to read from. Ranking scores are updated asynchronously by an ML service reading from Kafka.

**Q: How do you handle tweet deletion at scale?**
> A: Soft-delete in Cassandra (set `deleted_at` timestamp, don't actually delete). The fan-out service cannot efficiently remove tweet IDs from millions of follower timelines. Instead, filter at read time: when assembling the feed, skip tweet IDs that are soft-deleted. Run a periodic cleanup job to compact old deleted tweet IDs from timeline caches.

**Q: What happens to Twitter's architecture during major live events (elections, World Cup)?**
> A: At peak events, Twitter sees 5-10x normal traffic. Mitigations: pre-scaled infrastructure (AWS auto-scaling groups), circuit breakers on non-critical services, graceful degradation (e.g., don't show like counts if the counter service is overloaded), and geo-routing to distribute load across regions. Twitter has published that their peak event was 143,199 tweets per second.

**Q: How would you design the "quote tweet" feature?**
> A: A quote tweet is a new tweet with an embedded reference to another tweet: `{ text: "My comment", quoted_tweet_id: 12345 }`. It goes through normal fan-out. At display time, the feed service fetches the quoted tweet object and embeds it in the response. The quoted tweet's `quote_count` counter is incremented asynchronously via Kafka.

---

## 🔗 What to Read Next

1. **[Design WhatsApp](./DesignWhatsApp.md)** — Apply real-time messaging patterns (WebSocket, guaranteed delivery)
2. **[BuildingBlocks/ServiceDiscovery.md](../BuildingBlocks/ServiceDiscovery.md)** — How Twitter's microservices find each other
3. **[KeyConcepts/Consistent_Hashing.md](../KeyConcepts/Consistent_Hashing.md)** — Understand how Cassandra and Redis cluster sharding works

---

*[← URL Shortener](./DesignURLShortener.md) | [Back to Case Studies](./README.md) | [Next: Design WhatsApp →](./DesignWhatsApp.md)*
