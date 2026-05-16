# 📸 Design Instagram
## Photo Sharing, Feed Generation, and the Follower Graph at Scale

> *"Instagram serves 2 billion monthly active users, each with a personalized feed drawn from millions of creators. The challenge isn't storing photos — it's deciding which of the 500 million daily posts each user should see, in what order, in under 200ms."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md), [CDN](../BuildingBlocks/CDN.md), [Design Twitter](./DesignTwitter.md)

---

## 📋 Requirements (RADIO: R)

### Functional Requirements

| Feature | Description |
|---|---|
| **Upload photo/video** | Users can post images and short videos with captions, tags, filters |
| **Follow users** | Follow/unfollow, follower/following counts |
| **News Feed** | Personalized feed of posts from people you follow (ranked, not chronological) |
| **Like & Comment** | Engage with posts |
| **Stories** | Ephemeral 24-hour content |
| **Explore/Discover** | Algorithmically recommended content from accounts you don't follow |
| **Direct Messages** | Private messaging between users |

### Non-Functional Requirements

```
Scale:
  - 2B monthly active users, 500M daily active users
  - 100M photos uploaded daily (avg 5MB each = 500TB/day new media)
  - Feed generation: P99 < 500ms
  - Upload-to-visible: < 30 seconds
  - Read-heavy: 100:1 read/write ratio
  - 99.9% availability
```

---

## 📡 API Design (RADIO: A)

```
POST /v1/posts
  Body: { "media_ids": ["m123"], "caption": "Sunset 🌅", "location": "Bali", "tags": ["travel"] }
  Response: { "post_id": "p456", "created_at": "..." }

GET /v1/feed?cursor=<token>&limit=20
  Response: { "posts": [...], "next_cursor": "..." }

POST /v1/posts/{id}/like
POST /v1/posts/{id}/comments  { "text": "Beautiful!" }

POST /v1/users/{id}/follow
DELETE /v1/users/{id}/follow

POST /v1/media/upload-url
  Response: { "upload_url": "https://s3.../presigned", "media_id": "m123" }
```

---

## 🗄️ Data Model (RADIO: D)

```
Storage choices:
  - PostgreSQL: users, follows, posts metadata
  - Redis: feed cache, counters (likes, followers), session
  - S3 + CDN: media files (photos, videos, stories)
  - Cassandra: activity feed (write-heavy, wide-column, time-sorted)
  - Elasticsearch: explore/search (hashtags, users, locations)

Schema (simplified):
  users:    { id, username, bio, profile_pic_url, follower_count, following_count }
  posts:    { id, user_id, media_urls[], caption, location, created_at, like_count, comment_count }
  follows:  { follower_id, followee_id, created_at }  (partitioned by follower_id)
  likes:    { post_id, user_id, created_at }
  comments: { id, post_id, user_id, text, created_at }
```

---

## 🏗️ Infrastructure (RADIO: I)

```
┌──────────────────────────────────────────────────────────────────────┐
│                    INSTAGRAM ARCHITECTURE                            │
│                                                                      │
│  [Mobile App] ──▶ [API Gateway / LB]                                 │
│                         │                                            │
│         ┌───────────────┼────────────────────┐                       │
│         ▼               ▼                    ▼                       │
│  [Post Service]   [Feed Service]      [User Service]                 │
│    │                    │                    │                        │
│    │ upload media       │ generate feed      │ follow graph           │
│    ▼                    ▼                    ▼                        │
│  [S3 + CloudFront] [Feed Cache: Redis]  [PostgreSQL]                 │
│                         │                                            │
│  [Feed Generation]      │ fan-out on write (for regular users)       │
│  [Ranking ML Model]     │ fan-out on read (for celebrities)          │
│         │               │                                            │
│  [Kafka: post.created]  │                                            │
│    ├──▶ Feed workers (fan-out to follower feeds)                     │
│    ├──▶ Notification Service                                         │
│    └──▶ Search Indexer (Elasticsearch)                               │
│                                                                      │
│  [Cassandra] ← activity feeds (time-series, write-optimized)        │
│  [Redis] ← hot feed cache, counters, trending                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Feed Generation: Hybrid Fan-Out

```
Problem: Cristiano Ronaldo has 600M followers.
  Fan-out on write: one post → write to 600M feeds = impractical
  Fan-out on read: each feed request queries all followees = slow for normal users

Solution: HYBRID approach (what Instagram actually does)

  Regular users (< 10K followers):
    Fan-out on WRITE: when they post, push to all followers' feeds
    Fast read: followers just read from their pre-built feed cache

  Celebrities (> 10K followers):
    Fan-out on READ: their posts are NOT pushed to follower feeds
    At read time: merge celebrity posts into the feed dynamically
    
  Feed read request:
    1. Get pre-computed feed from Redis (contains posts from regular followees)
    2. Fetch recent posts from celebrities the user follows
    3. Merge + rank by ML model (engagement probability, recency, relationship)
    4. Return top 20 posts with pagination cursor
```

---

## 📈 Optimization (RADIO: O)

```
Media upload optimization:
  - Client uploads directly to S3 via pre-signed URL (server handles 0 bytes)
  - Lambda trigger on S3 → generate multiple resolutions (thumbnail, medium, full)
  - CDN serves images globally (cache at edge for popular content)
  - WebP/AVIF format for 30-50% smaller files than JPEG

Feed ranking (ML-based):
  Features: post recency, user relationship strength, content type preference,
            engagement history, time spent on similar posts
  Model: lightweight model (< 10ms inference) at feed generation time
  Fallback: chronological feed if ML service is down (graceful degradation)

Counter scaling (likes, followers):
  Problem: 1M likes/sec on viral posts → hot key on single DB row
  Solution: Redis INCR for counters (atomic, in-memory)
            Async persistence to PostgreSQL via batch writes every 5 seconds
```

---

## 🧩 Mini Challenge

**Instagram Stories expire after 24 hours. Design the storage and cleanup mechanism for Stories.**

<details>
<summary>💡 Click to reveal answer</summary>

**Storage**: S3 with a 24h lifecycle policy (but don't rely solely on S3 lifecycle for UX — it's not instant).

**Active stories tracking**: Redis sorted set per user, score = expiry_timestamp:
```
ZADD stories:user:123 1747574400 "story_media_id_1"
ZADD stories:user:123 1747578000 "story_media_id_2"
```

**Read stories**: `ZRANGEBYSCORE stories:user:123 {now} +inf` → only non-expired stories returned.

**Cleanup**: Background job runs every minute:
```
ZREMRANGEBYSCORE stories:user:* 0 {now}  → remove expired entries from Redis
```
S3 lifecycle rule deletes media files after 25 hours (1h buffer for edge cases).

**Availability trick**: Stories are served from Redis (fast). Even if the cleanup job lags, the ZRANGEBYSCORE filter ensures expired stories are never shown to users — they just take slightly longer to be purged from storage.

</details>

---

## 📝 Interview Q&A

**Q: How would you handle the "celebrity problem" in feed generation?**
> A: Use a hybrid fan-out approach. For regular users (< 10K followers), fan-out on write — push their posts into all followers' feed caches at post time. For celebrities (> 10K followers), fan-out on read — their posts are NOT pre-pushed. When a user requests their feed, the system merges the pre-built cache (regular followee posts) with fresh celebrity posts fetched at read time. This bounds the write amplification of celebrity posts while keeping feed reads fast for the common case.

**Q: How do you store and serve 500TB of new photos per day?**
> A: Object storage (S3) for durability and infinite scale, fronted by a CDN (CloudFront/Fastly) for low-latency global delivery. Upload path: client gets a pre-signed URL from the app server, uploads directly to S3 (zero bytes through application tier). Post-upload: a Lambda function generates multiple resolutions (150px thumbnail, 640px feed, original). The CDN caches popular images at edge locations — most photo reads are CDN cache hits (< 50ms globally), never touching S3 or app servers.

---

## 🔗 What to Read Next

1. **[SystemDesignCaseStudies/DesignTwitter.md](./DesignTwitter.md)** — Similar feed generation architecture with different trade-offs
2. **[BuildingBlocks/CDN.md](../BuildingBlocks/CDN.md)** — How CDNs serve Instagram's media globally
3. **[BuildingBlocks/Blob_Storage.md](../BuildingBlocks/Blob_Storage.md)** — Object storage patterns for media-heavy applications

---

*[← Design WhatsApp](./DesignWhatsApp.md) | [Back to Case Studies](./README.md) | [Next: Design YouTube →](./DesignYouTube.md)*
