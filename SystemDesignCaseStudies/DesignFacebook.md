# 📱 Design Facebook News Feed

> *"2 billion people opening Facebook expect a personalized, relevant feed — instantly. Behind that simple scroll is a system that: (1) ingests millions of posts per minute, (2) determines which of your 5000 friends' posts YOU should see, (3) ranks them by relevance (not chronology!), and (4) serves the result in < 200ms. It's THE canonical example of fan-out architecture and recommendation systems at extreme scale."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Caching](../BuildingBlocks/Caching.md), [Message Queues](../BuildingBlocks/MessageQueues.md), [Database Sharding](../Database/Sharding.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [The Fan-Out Problem](#-the-fan-out-problem)
3. [High-Level Architecture](#-high-level-architecture)
4. [Feed Generation Strategies](#-feed-generation-strategies)
5. [Ranking & Personalization](#-ranking--personalization)
6. [Storage Design](#-storage-design)
7. [Real-Time Updates](#-real-time-updates)
8. [Java Implementation](#-java-implementation)
9. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Users create posts (text, images, video, links)
  • Users follow/friend other users
  • News Feed: see posts from friends, ranked by relevance
  • Like, comment, share (engagement actions!)
  • Real-time: new posts appear without refresh
  • Pagination: infinite scroll (load more as you scroll!)
  
NON-FUNCTIONAL:
  • 2B monthly active users, 500M DAU
  • Feed generation: < 200ms latency!
  • Average user: 500 friends
  • Post volume: 500K new posts per second!
  • Feed freshness: new posts visible within 5 seconds
  • Read-heavy: 100× more feed reads than post writes

SCALE NUMBERS:
  • 500M feed requests per day
  • 500K posts per second (including shares, comments)
  • Average user has 500 friends
  • Feed shows top ~50 posts (from potentially thousands!)
  • Each post: ~1 KB metadata + media URLs
```

---

## 🌊 The Fan-Out Problem

```
THE CORE CHALLENGE:
  User posts "Having a great day!" 
  → 500 friends should see this in THEIR feeds!
  → WHEN do we put it in their feeds?

TWO APPROACHES:

  FAN-OUT ON WRITE (Push Model):
  ┌────────────────────────────────────────────────────────────────┐
  │  User A posts → immediately push to ALL followers' feed caches!│
  │                                                                 │
  │  A posts ──► Write to A's 500 friends' pre-computed feeds!     │
  │              friend1_feed: [A's post, ...]                      │
  │              friend2_feed: [A's post, ...]                      │
  │              ...                                                │
  │              friend500_feed: [A's post, ...]                    │
  │                                                                 │
  │  On feed request: just READ the pre-computed feed! (fast!)      │
  │                                                                 │
  │  ✅ Read is SUPER FAST (just fetch pre-computed list!)          │
  │  ❌ Write is expensive (500 writes per post!)                   │
  │  ❌ Celebrity problem: Lady Gaga has 50M followers!             │
  │     One post = 50M writes?! IMPOSSIBLE!                         │
  └────────────────────────────────────────────────────────────────┘

  FAN-OUT ON READ (Pull Model):
  ┌────────────────────────────────────────────────────────────────┐
  │  User requests feed → query ALL friends' recent posts NOW!     │
  │                                                                 │
  │  Feed request ──► Get friend list (500 friends!)               │
  │                ──► For each friend: get their recent posts!    │
  │                ──► Merge + rank + return top 50!               │
  │                                                                 │
  │  ✅ Write is cheap (just write once!)                           │
  │  ✅ Celebrity posts handled same as anyone!                     │
  │  ❌ Read is SLOW (500 queries per feed request!)               │
  │  ❌ Latency unacceptable at scale!                             │
  └────────────────────────────────────────────────────────────────┘

  HYBRID APPROACH (Facebook/Twitter reality!):
  ┌────────────────────────────────────────────────────────────────┐
  │  Normal users (< 10K followers): Fan-out on WRITE!             │
  │  → Pre-compute and push to followers' caches!                  │
  │  → Fast reads for 99% of users!                                │
  │                                                                 │
  │  Celebrities (> 10K followers): Fan-out on READ!               │
  │  → DON'T push to millions of caches!                           │
  │  → When user opens feed: merge celebrity posts on-the-fly!     │
  │                                                                 │
  │  User's feed = pre-computed (from normal friends)              │
  │              + merged real-time (from followed celebrities!)    │
  └────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        NEWS FEED SYSTEM                                    │
│                                                                            │
│  ┌─────────┐     ┌───────────────────────────────────────────────────┐   │
│  │  Client  │────►│  API Gateway                                      │   │
│  └─────────┘     └──────┬────────────────────┬────────────────────────┘   │
│                          │                    │                             │
│              ┌───────────▼──────┐   ┌────────▼──────────┐                 │
│              │  Post Service     │   │  Feed Service      │                 │
│              │  • Create post    │   │  • Get feed        │                 │
│              │  • Delete/edit    │   │  • Rank & merge    │                 │
│              └───────┬──────────┘   └────────┬──────────┘                 │
│                      │                        │                             │
│                      │ New post event!        │ Read pre-computed feed     │
│                      ▼                        ▼                             │
│  ┌─────────────────────────────┐   ┌──────────────────────────┐          │
│  │  Fan-Out Service            │   │  Feed Cache (Redis!)     │          │
│  │  • Get poster's followers   │   │  user123_feed: [sorted   │          │
│  │  • Push to each follower's  │   │    list of post IDs      │          │
│  │    feed cache!              │   │    with scores]           │          │
│  │  • Skip celebrities!        │   │                           │          │
│  └──────────┬──────────────────┘   └──────────────────────────┘          │
│             │                                                              │
│             │ (async via Kafka!)                                           │
│             ▼                                                              │
│  ┌──────────────────────────────────────────────────────────────────┐    │
│  │  Kafka: "new-posts" topic                                         │    │
│  │  Fan-out workers consume and push to caches!                      │    │
│  └──────────────────────────────────────────────────────────────────┘    │
│                                                                            │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐       │
│  │  Post Storage     │  │  Social Graph    │  │  User Service     │       │
│  │  (Posts + media)  │  │  (friends list)  │  │  (profiles)       │       │
│  │  Cassandra/HBase  │  │  Redis/Graph DB  │  │  PostgreSQL       │       │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘       │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Feed Generation Strategies

```
PRE-COMPUTED FEED (Fan-out on write):

  When User A posts:
  1. Save post to Posts table (Cassandra)
  2. Send event to Kafka: {postId, userId, timestamp}
  3. Fan-out workers consume:
     a. Get A's follower list (Social Graph service)
     b. For each follower (up to 10K):
        - ZADD feed:{followerId} {timestamp} {postId}
        - ZREMRANGEBYRANK feed:{followerId} 0 -501 (keep top 500!)
     c. For followers > 10K: skip! (celebrity path!)
  
  Reading feed:
  1. ZREVRANGE feed:{userId} 0 49 → top 50 post IDs!
  2. Hydrate: fetch full post data for each ID
  3. Merge with celebrity posts (fan-out on read for celebrities!)
  4. Rank! (apply ML ranking model)
  5. Return!

CELEBRITY HANDLING:
  User follows: 200 normal friends + 50 celebrities
  
  Feed = Redis pre-computed (200 friends' posts, already pushed!)
       + real-time merge (50 celebrities' latest posts, fetched now!)
  
  Celebrity posts stored in: celebrity_posts:{celebrityId} (sorted set!)
  On feed read: ZUNIONSTORE with celebrity posts → merged result!

FEED INVALIDATION:
  User unfriends someone → remove their posts from feed cache!
  User deletes a post → remove from ALL followers' caches!
  (Async: eventual consistency is fine — 1-2s delay acceptable!)
```

---

## 🧠 Ranking & Personalization

```
RAW FEED ≠ FINAL FEED!

After collecting candidate posts, RANK them by predicted engagement!

RANKING SIGNALS:
  ┌────────────────────────────────────────────────────────────────┐
  │  Signal Category    │  Examples                                 │
  ├────────────────────────────────────────────────────────────────┤
  │  Affinity           │  How often you interact with poster?      │
  │  (relationship)     │  Do you message them? Like their posts?   │
  ├────────────────────────────────────────────────────────────────┤
  │  Content type       │  You engage more with videos vs text?     │
  │  (preference)       │  Photos from close friends vs news links? │
  ├────────────────────────────────────────────────────────────────┤
  │  Recency            │  How fresh is the post? (decay function!) │
  │  (time decay)       │  Recent > old (but not ONLY chronological)│
  ├────────────────────────────────────────────────────────────────┤
  │  Engagement         │  How many likes/comments already?         │
  │  (social proof)     │  High engagement = probably interesting!  │
  ├────────────────────────────────────────────────────────────────┤
  │  Creator quality    │  Does poster's content usually get likes? │
  │  (track record)     │  High-quality creators ranked higher!     │
  └────────────────────────────────────────────────────────────────┘

RANKING FORMULA (simplified):
  score = affinity_weight × affinity
        + content_weight × content_type_preference
        + recency_weight × time_decay(post_age)
        + engagement_weight × engagement_score
        + creator_weight × creator_quality

  In reality: ML model (neural network!) trained on billions of
  engagement signals. Predicts P(user clicks/likes/comments on post).

ANTI-PATTERNS TO HANDLE:
  • Engagement bait: "Like if you breathe!" → penalize!
  • Misinformation: fact-checked posts → reduce distribution!
  • Repetitive content: don't show 10 posts from same person!
  • Old content suddenly getting comments: re-rank appropriately!
```

---

## 💾 Storage Design

```
POLYGLOT PERSISTENCE (different DBs for different needs!):

  ┌────────────────────────────────────────────────────────────────┐
  │  Data               │  Storage         │  Why                   │
  ├────────────────────────────────────────────────────────────────┤
  │  Posts (content)    │  Cassandra       │  Write-heavy, sharded! │
  │  Feed cache         │  Redis           │  Fast reads, sorted!   │
  │  Social graph       │  TAO/Graph DB    │  Relationship queries! │
  │  User profiles      │  PostgreSQL      │  Structured, ACID      │
  │  Media (images)     │  Blob store/CDN  │  Large binary files!   │
  │  Counters (likes)   │  Redis           │  Atomic increment!     │
  │  Search             │  Elasticsearch   │  Full-text search!     │
  │  Analytics          │  Hadoop/Spark    │  Batch processing!     │
  └────────────────────────────────────────────────────────────────┘

POST STORAGE (Cassandra):
  Partition key: user_id (all posts by same user together!)
  Clustering key: post_id (TimeUUID — sorted by time!)
  
  CREATE TABLE posts (
    user_id UUID,
    post_id TIMEUUID,
    content TEXT,
    media_urls LIST<TEXT>,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, post_id)
  ) WITH CLUSTERING ORDER BY (post_id DESC);
  
  Get user's recent posts: easy partition scan!
  Get specific post by ID: need secondary index or separate table!

FEED CACHE (Redis):
  Sorted Set per user:
  ZADD feed:user123 1700000000 "post:abc"  (score = timestamp!)
  ZADD feed:user123 1699999000 "post:def"
  
  Get feed: ZREVRANGE feed:user123 0 49 → latest 50 post IDs!
  
  Memory per user: ~500 post IDs × 50 bytes = 25 KB
  Total: 500M users × 25 KB = 12.5 TB Redis cluster!
  (Only active users cached! LRU eviction for inactive!)

SOCIAL GRAPH:
  user123 follows: [user456, user789, user101, ...]
  
  Redis SET or purpose-built graph (Facebook's TAO):
  SMEMBERS following:user123 → all followed users
  SMEMBERS followers:user456 → all followers (for fan-out!)
  SCARD followers:user456 → follower count!
```

---

## ⚡ Real-Time Updates

```
NEW POST → APPEARS IN FEED WITHOUT REFRESH!

APPROACH 1: POLLING (simple but wasteful!)
  Client polls every 30 seconds: GET /feed?since=lastTimestamp
  Server: "Here are 3 new posts since then!"
  
  500M users × 2 polls/min = 1B requests/min! 💀
  Most return empty! (wasteful!)

APPROACH 2: LONG POLLING (better!)
  Client: GET /feed/updates (server holds connection open!)
  Server: waits until new post available → responds!
  Client: immediately reconnects!
  
  Less wasteful but: holding millions of connections is expensive!

APPROACH 3: SERVER-SENT EVENTS / WEBSOCKET (best!)
  Client opens WebSocket/SSE connection.
  Server pushes new posts as they arrive!
  
  Active user opens app → WebSocket to notification service!
  New post for this user → push immediately!
  
  SCALE CHALLENGE: 100M concurrent WebSocket connections!
  Solution: WebSocket servers are stateless!
  Redis Pub/Sub routes events to correct server!
  
  User123 connected to WS-Server-7:
  New post for User123 → Redis PUBLISH "ws:user123" postData
  WS-Server-7 subscribed to "ws:user123" → forwards to client!

HYBRID (practical!):
  • Active users (app open): WebSocket for real-time!
  • Background: no connection (save resources!)
  • App reopened: fetch full feed (pre-computed from cache!)
  • Real-time during session: incremental WebSocket updates!
```

---

## 💻 Java Implementation

### Fan-Out Service

```java
@Service
public class FanOutService {
    
    @Autowired private SocialGraphService socialGraph;
    @Autowired private RedisTemplate<String, String> redis;
    
    private static final int CELEBRITY_THRESHOLD = 10_000;
    private static final int MAX_FEED_SIZE = 500;
    
    /**
     * Fan-out a new post to followers' feed caches.
     * Called asynchronously via Kafka consumer!
     */
    @KafkaListener(topics = "new-posts", groupId = "fanout-workers")
    public void fanOutPost(@Payload PostEvent event) {
        String posterId = event.getUserId();
        String postId = event.getPostId();
        double score = event.getTimestamp(); // Score = timestamp!
        
        // Get poster's followers
        Set<String> followers = socialGraph.getFollowers(posterId);
        
        if (followers.size() > CELEBRITY_THRESHOLD) {
            // Celebrity! Don't fan out — will be merged on read!
            redis.opsForZSet().add("celebrity_posts:" + posterId, 
                postId, score);
            redis.opsForZSet().removeRange("celebrity_posts:" + posterId, 
                0, -101); // Keep latest 100 posts!
            return;
        }
        
        // Normal user: fan out to all followers!
        for (String followerId : followers) {
            String feedKey = "feed:" + followerId;
            
            // Add post to follower's feed
            redis.opsForZSet().add(feedKey, postId, score);
            
            // Trim to MAX_FEED_SIZE (remove oldest!)
            redis.opsForZSet().removeRange(feedKey, 0, -(MAX_FEED_SIZE + 1));
        }
    }
}
```

### Feed Service

```java
@Service
public class FeedService {
    
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private PostService postService;
    @Autowired private SocialGraphService socialGraph;
    @Autowired private RankingService rankingService;
    
    /**
     * Get user's personalized news feed.
     */
    public FeedResponse getFeed(String userId, int page, int pageSize) {
        // 1. Get pre-computed feed (from fan-out on write!)
        String feedKey = "feed:" + userId;
        long start = (long) page * pageSize;
        long end = start + pageSize - 1;
        
        Set<String> preComputedPostIds = redis.opsForZSet()
            .reverseRange(feedKey, start, end + 50); // Fetch extra for ranking!
        
        // 2. Merge celebrity posts (fan-out on read!)
        Set<String> followedCelebrities = socialGraph
            .getFollowedCelebrities(userId);
        
        Set<String> celebrityPostIds = new HashSet<>();
        for (String celeb : followedCelebrities) {
            Set<String> posts = redis.opsForZSet()
                .reverseRange("celebrity_posts:" + celeb, 0, 9);
            if (posts != null) celebrityPostIds.addAll(posts);
        }
        
        // 3. Combine all candidate post IDs
        Set<String> allCandidates = new HashSet<>();
        if (preComputedPostIds != null) allCandidates.addAll(preComputedPostIds);
        allCandidates.addAll(celebrityPostIds);
        
        // 4. Hydrate: fetch full post data
        List<Post> posts = postService.getPostsByIds(allCandidates);
        
        // 5. Rank by ML model!
        List<Post> ranked = rankingService.rank(userId, posts);
        
        // 6. Paginate and return
        List<Post> page = ranked.subList(0, 
            Math.min(pageSize, ranked.size()));
        
        return new FeedResponse(page, page < ranked.size());
    }
}
```

### Ranking Service

```java
@Service
public class RankingService {
    
    @Autowired private UserAffinityService affinityService;
    
    /**
     * Rank posts by predicted engagement (simplified scoring).
     */
    public List<Post> rank(String userId, List<Post> candidates) {
        Map<String, Double> affinityScores = 
            affinityService.getAffinities(userId);
        
        return candidates.stream()
            .map(post -> new ScoredPost(post, calculateScore(userId, post, affinityScores)))
            .sorted(Comparator.comparingDouble(ScoredPost::score).reversed())
            .map(ScoredPost::post)
            .collect(Collectors.toList());
    }
    
    private double calculateScore(String userId, Post post, 
                                   Map<String, Double> affinities) {
        double affinity = affinities.getOrDefault(post.getAuthorId(), 0.5);
        double recency = timeDecay(post.getCreatedAt());
        double engagement = normalizeEngagement(post.getLikeCount() + 
            post.getCommentCount() * 2); // Comments weighted more!
        double contentBoost = getContentTypeBoost(userId, post.getType());
        
        return 0.4 * affinity 
             + 0.3 * recency 
             + 0.2 * engagement 
             + 0.1 * contentBoost;
    }
    
    private double timeDecay(Instant createdAt) {
        long hoursAgo = Duration.between(createdAt, Instant.now()).toHours();
        return 1.0 / (1.0 + 0.1 * hoursAgo); // Exponential decay!
    }
}
```

---

## ❓ Interview Q&A

**Q1: Fan-out on write vs fan-out on read — how do you choose?**
> Hybrid! Fan-out on write for users with < 10K followers (99% of users): pre-compute feeds for fast reads. Fan-out on read for celebrities (> 10K followers): too expensive to push to millions of caches. On feed request: merge pre-computed cache + celebrity posts fetched on-the-fly. This gives O(1)-ish reads for most feed requests while avoiding the celebrity explosion problem. Twitter actually switched from pure fan-out-on-write to hybrid after the "celebrity problem" caused write storms.

**Q2: How do you handle the storage for 500M users' feeds?**
> Redis sorted sets: each user's feed = sorted set of post IDs (not full posts!). Keep only top 500 post IDs per user (trim oldest). Memory: 500 post IDs × 50 bytes = 25 KB/user. 500M × 25 KB = 12.5 TB Redis cluster. But! Only cache ACTIVE users' feeds (DAU = 500M, not all 2B registered!). Inactive users: regenerate on next login (pull from friends' recent posts). LRU eviction: users inactive for 30 days get evicted from cache. Full post content stored separately (Cassandra) — feed cache only has IDs!

**Q3: How do you keep the feed fresh (new posts appear quickly)?**
> Three layers: (1) Fan-out workers process Kafka events in < 5 seconds (push to feed caches), (2) For online users: WebSocket pushes new post notification immediately (client prepends to feed without refresh!), (3) Feed cache has short effective TTL — re-ranked on every fetch. For celebrities (fan-out on read): their latest posts are ALWAYS fresh because we fetch them in real-time during feed generation. Total end-to-end: post created → visible in friends' feeds within 5-10 seconds.

**Q4: How would you implement "unfollow" and "delete post"?**
> Unfollow User B: (1) Remove from social graph immediately, (2) Async: scan feed cache → remove all posts by B (ZREM by post IDs where author=B). But DON'T scan all 500 entries immediately — just let them naturally age out (lazy cleanup!). Delete post: (1) Mark post as deleted in Posts DB, (2) Async: for each follower → ZREM postId from their feed cache, (3) If post appears in feed before async cleanup: check "is_deleted" flag during hydration → skip! Eventually consistent but acceptable (seconds of delay).

---

## 🔗 Related Topics
- [Caching Strategies](../BuildingBlocks/CachingStrategies.md) — Feed cache patterns
- [Pub/Sub](../MessagingQ/PubSub.md) — Event fan-out
- [WebSockets](../APIs/WebSockets.md) — Real-time feed updates
- [Database Sharding](../Database/Sharding.md) — Scaling post storage
- [Design Twitter](DesignTwitter.md) — Similar timeline problem

---

*"The news feed is not a database query — it's a PREDICTION. We're predicting which posts will make you smile, think, or engage. Getting that prediction wrong means users leave. Getting it right means they stay for hours. That's why Facebook employs thousands of ML engineers just for the feed." — Former Facebook Engineer* 📱
