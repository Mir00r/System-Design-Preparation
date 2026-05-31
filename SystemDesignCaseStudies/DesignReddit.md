# 📱 Design Reddit (Social News + Comments)

> *"Reddit is 'the front page of the internet' — a system where content rises and falls based on community votes, discussions nest infinitely deep like Russian dolls, and millions of communities (subreddits) each have their own culture and rules. Behind the simplicity of upvote/downvote lies a sophisticated ranking algorithm, a comment tree stored as a recursive data structure, and a feed generation system that balances freshness, popularity, and personalization. Let's build it!"*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Caching](../BuildingBlocks/Caching.md), [Database](../Database/SQL_Vs_NoSQL.md), [CDN](../BuildingBlocks/CDN.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Post & Vote System](#-post--vote-system)
4. [Comment Tree (Nested Comments)](#-comment-tree-nested-comments)
5. [Feed Ranking Algorithm](#-feed-ranking-algorithm)
6. [Subreddit & Moderation](#-subreddit--moderation)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Create/join subreddits (communities!)
  • Post content (text, links, images, videos!)
  • Upvote/downvote posts and comments
  • Nested comments (unlimited depth!)
  • Feed: Hot, New, Top (today/week/all-time), Rising, Controversial
  • Search posts and subreddits
  • User karma (reputation score!)
  • Awards/badges on posts and comments
  • Moderation tools (remove, ban, pin!)

NON-FUNCTIONAL:
  • 500M monthly active users
  • 100K posts per day, 1M comments per day
  • Feed generation: < 200ms
  • Vote recording: < 50ms (feels instant!)
  • Comment tree load: < 500ms (even 10K+ comments!)
  • Eventually consistent votes (OK if count lags 1-2 seconds!)

SCALE:
  • 100K+ subreddits active daily
  • Popular posts: 50K+ upvotes, 10K+ comments
  • Reads >>> writes (100:1 ratio!)
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                          REDDIT ARCHITECTURE                            │
│                                                                         │
│  ┌──────────┐     ┌──────────┐      ┌─────────────────────────────┐  │
│  │  Mobile  │────►│   CDN    │◄────│  Static Assets (JS/CSS/img)   │  │
│  │  Web App │     │(CloudFront)│    └─────────────────────────────┘  │
│  └──────────┘     └─────┬────┘                                       │
│                          │                                             │
│                    ┌─────▼─────┐                                      │
│                    │ API Gateway│                                      │
│                    │ + Auth     │                                      │
│                    └─────┬─────┘                                      │
│           ┌──────────────┼──────────────────┐                         │
│           ▼              ▼                  ▼                          │
│  ┌──────────────┐ ┌──────────────┐  ┌──────────────┐                │
│  │ Post Service │ │Comment Service│  │ Feed Service │                │
│  │ • CRUD posts │ │ • Tree CRUD  │  │ • Ranking    │                │
│  │ • Votes      │ │ • Threading  │  │ • Pagination │                │
│  └───────┬──────┘ └───────┬──────┘  └───────┬──────┘                │
│          │                 │                  │                        │
│          ▼                 ▼                  ▼                        │
│  ┌──────────────────────────────────────────────────────┐            │
│  │                    DATA LAYER                          │            │
│  │  ┌─────────┐  ┌─────────┐  ┌────────┐  ┌─────────┐ │            │
│  │  │PostgreSQL│  │ Redis   │  │  S3    │  │Elastic- │ │            │
│  │  │(posts,  │  │(votes,  │  │(media) │  │search   │ │            │
│  │  │comments)│  │ feeds,  │  │        │  │(search!)│ │            │
│  │  │         │  │ cache)  │  │        │  │         │ │            │
│  │  └─────────┘  └─────────┘  └────────┘  └─────────┘ │            │
│  └──────────────────────────────────────────────────────┘            │
│                                                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ Vote Service │  │  Karma Svc   │  │ Notification │               │
│  │ (dedicated!) │  │ (async calc) │  │   Service    │               │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 🗳️ Post & Vote System

```
POST SCHEMA:
  ┌────────────────────────────────────────────────────────┐
  │  posts                                                  │
  │  ─────────────────────────────────────────              │
  │  id: UUID (primary key)                                │
  │  subreddit_id: UUID (FK → subreddits)                  │
  │  author_id: UUID (FK → users)                          │
  │  title: VARCHAR(300)                                    │
  │  content_type: ENUM(text, link, image, video)          │
  │  body: TEXT (for text posts)                            │
  │  url: VARCHAR(2000) (for link posts)                   │
  │  media_urls: JSONB (for images/videos)                 │
  │  score: INT (upvotes - downvotes, denormalized!)       │
  │  upvotes: INT                                          │
  │  downvotes: INT                                        │
  │  comment_count: INT (denormalized!)                    │
  │  created_at: TIMESTAMP                                 │
  │  is_pinned: BOOLEAN                                    │
  │  is_locked: BOOLEAN                                    │
  │  is_removed: BOOLEAN                                   │
  └────────────────────────────────────────────────────────┘

VOTE SYSTEM (the core mechanic!):
  ┌────────────────────────────────────────────────────────┐
  │  votes                                                  │
  │  ─────────────────────────────────────────              │
  │  user_id: UUID                                         │
  │  target_id: UUID (post OR comment!)                    │
  │  target_type: ENUM(post, comment)                      │
  │  value: SMALLINT (-1, 0, +1)                           │
  │  created_at: TIMESTAMP                                 │
  │  ─────────────────────────────────────────              │
  │  PRIMARY KEY: (user_id, target_id, target_type)        │
  │  ← One vote per user per item! (UPSERT!)              │
  └────────────────────────────────────────────────────────┘

VOTE FLOW (fast + eventually consistent!):
  1. User clicks upvote
  2. API: UPSERT into votes table (idempotent!)
  3. Return 200 immediately to user (feels instant!)
  4. Async: increment score counter in Redis (INCRBY!)
  5. Async: periodic flush Redis counters → PostgreSQL
  6. Dashboard shows Redis value (near real-time!)
  
  WHY async? A viral post gets 1000 votes/second!
  Synchronous DB update = deadlocks! 💀
  Redis INCRBY = atomic, single-threaded, FAST!
  
  VOTE DEDUPLICATION:
  Redis SET per post: "post:{id}:voters" → stores user_ids
  Before counting vote: SISMEMBER check → already voted?
  • Yes + same direction → ignore (already counted!)
  • Yes + different → remove old, add new (change vote!)
  • No → add to set, increment counter!
```

---

## 🌳 Comment Tree (Nested Comments)

```
STORING NESTED COMMENTS (the hardest part!):

APPROACH 1: Adjacency List (simple but SLOW for deep threads!)
  comments table:
  | id | parent_id | post_id | body | score | depth |
  |  1 | NULL      | P1      | "Great post!" | 42 | 0 |
  |  2 | 1         | P1      | "I agree!"    | 10 | 1 |
  |  3 | 2         | P1      | "Me too!"     |  5 | 2 |
  |  4 | 1         | P1      | "I disagree!" | -3 | 1 |
  
  Tree structure:
  [1] Great post! (42 pts)
    ├── [2] I agree! (10 pts)
    │     └── [3] Me too! (5 pts)
    └── [4] I disagree! (-3 pts)
  
  Load: SELECT * FROM comments WHERE post_id = ? ORDER BY depth, score DESC
  Problem: Loading subtree requires recursive queries! (slow for 10K comments!)

APPROACH 2: Materialized Path (Reddit's actual approach!)
  | id | path          | post_id | body | score |
  |  1 | "1"           | P1      | ...  | 42    |
  |  2 | "1.2"         | P1      | ...  | 10    |
  |  3 | "1.2.3"       | P1      | ...  |  5    |
  |  4 | "1.4"         | P1      | ...  | -3    |
  
  Load entire subtree: WHERE path LIKE '1.%' (fast! one query!)
  Sort children: WHERE path LIKE '1._' ORDER BY score DESC
  No recursive queries needed! ✅
  
  Trade-off: path gets long for deep nesting (but Reddit limits to ~10 levels!)

APPROACH 3: Closure Table (best for complex queries!)
  comment_closure:
  | ancestor_id | descendant_id | depth |
  |     1       |      1        |   0   |
  |     1       |      2        |   1   |
  |     1       |      3        |   2   |
  |     1       |      4        |   1   |
  |     2       |      2        |   0   |
  |     2       |      3        |   1   |
  
  All descendants of comment 1:
  SELECT descendant_id FROM closure WHERE ancestor_id = 1
  
  ✅ Any subtree query = O(1)!
  ❌ More storage (N² in worst case!)
  ❌ Insert requires adding closure rows for ALL ancestors!

REDDIT'S STRATEGY: Load top 200 comments sorted by "best"!
  • Collapse deeply nested (> 5 levels) → "Continue this thread →"
  • "Load more comments" for long threads (lazy load!)
  • Cache hot comment trees in Redis (entire tree as JSON!)
```

---

## 📊 Feed Ranking Algorithm

```
REDDIT'S RANKING ALGORITHMS:

HOT (default feed!):
  ┌─────────────────────────────────────────────────────────────────┐
  │  hot_score = log10(max(|score|, 1)) + (sign × age_factor)       │
  │                                                                  │
  │  where:                                                          │
  │    score = upvotes - downvotes                                   │
  │    sign = +1 if score > 0, -1 if score < 0, 0 if score == 0    │
  │    age_factor = (post_time - epoch) / 45000                     │
  │                                                                  │
  │  KEY INSIGHT: time component dominates!                          │
  │  A new post with 10 votes can beat an old post with 1000 votes! │
  │  This ensures FRESHNESS — Reddit always has new content!         │
  │                                                                  │
  │  Why log10? Diminishing returns!                                 │
  │  First 10 votes matter as much as next 90!                      │
  │  Prevents runaway posts from dominating forever!                 │
  └─────────────────────────────────────────────────────────────────┘

TOP (sort by score in time period):
  Simple: ORDER BY (upvotes - downvotes) DESC
  Filter: WHERE created_at > (now - period)
  Period: today, this week, this month, this year, all time!

BEST (Wilson Score — for comments!):
  ┌─────────────────────────────────────────────────────────────────┐
  │  Instead of raw score, use statistical CONFIDENCE!               │
  │                                                                  │
  │  Post A: 1 upvote, 0 downvotes (100% positive but LOW sample!) │
  │  Post B: 95 upvotes, 5 downvotes (95% positive, HIGH sample!)  │
  │                                                                  │
  │  Wilson score ranks B higher! (more statistical confidence!)     │
  │  Formula: lower bound of 95% confidence interval                 │
  │                                                                  │
  │  This prevents: a comment with 1 vote beating one with 1000!    │
  └─────────────────────────────────────────────────────────────────┘

CONTROVERSIAL:
  controversial_score = (upvotes + downvotes) / max(|upvotes - downvotes|, 1)
  High score = many votes but close to 50/50 split!
  Example: 500 up + 490 down = very controversial! 🔥

FEED GENERATION:
  Pre-compute: hourly Spark job calculates hot scores for all posts!
  Store: sorted set in Redis per subreddit
    ZADD "feed:hot:r/programming" {hot_score} {post_id}
  
  User's home feed: merge feeds from subscribed subreddits!
  ZUNIONSTORE "user:42:home_feed" 50 "feed:hot:r/programming" 
    "feed:hot:r/java" "feed:hot:r/systemdesign" ...
  
  Pagination: ZREVRANGEBYSCORE with cursor!
```

---

## 💻 Java Implementation

### Vote Service

```java
@Service
public class VoteService {
    
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private VoteRepository voteRepository;
    @Autowired private KafkaTemplate<String, VoteEvent> kafka;
    
    /**
     * Record a vote (upvote/downvote/unvote).
     * Fast path: Redis for real-time count.
     * Slow path: Kafka → DB for persistence.
     */
    public VoteResponse vote(String userId, String targetId, 
                             VoteType type, int value) {
        String voteKey = "votes:" + targetId;
        String voterSetKey = "voters:" + targetId;
        String userVoteKey = "uservote:" + userId + ":" + targetId;
        
        // Check existing vote
        String existingVote = redis.opsForValue().get(userVoteKey);
        int scoreDelta = value;
        
        if (existingVote != null) {
            int oldValue = Integer.parseInt(existingVote);
            if (oldValue == value) {
                return VoteResponse.noChange(); // Already voted same way!
            }
            scoreDelta = value - oldValue; // E.g., switching +1 to -1 = delta of -2
        }
        
        // Update Redis (atomic pipeline!)
        redis.execute(new SessionCallback<>() {
            @Override
            public Object execute(RedisOperations ops) {
                ops.multi();
                ops.opsForValue().set(userVoteKey, String.valueOf(value));
                ops.opsForHash().increment(voteKey, "score", scoreDelta);
                if (value > 0) ops.opsForHash().increment(voteKey, "ups", 1);
                if (value < 0) ops.opsForHash().increment(voteKey, "downs", 1);
                return ops.exec();
            }
        });
        
        // Async: persist to DB via Kafka
        kafka.send("votes", targetId, VoteEvent.builder()
            .userId(userId)
            .targetId(targetId)
            .targetType(type)
            .value(value)
            .timestamp(Instant.now())
            .build());
        
        return VoteResponse.success(scoreDelta);
    }
}
```

### Comment Tree Service

```java
@Service
public class CommentService {
    
    @Autowired private CommentRepository commentRepo;
    @Autowired private RedisTemplate<String, String> redis;
    
    /**
     * Load comment tree for a post using materialized path!
     * Returns top-level comments sorted by "best", with nested children.
     */
    public CommentTree getCommentTree(String postId, String sortBy, 
                                       int limit, int maxDepth) {
        // Try cache first (hot posts have cached trees!)
        String cacheKey = "comments:" + postId + ":" + sortBy;
        String cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            return deserialize(cached);
        }
        
        // Load from DB — materialized path approach!
        List<Comment> comments = commentRepo
            .findByPostIdAndDepthLessThanEqual(postId, maxDepth,
                Sort.by(getSortField(sortBy)).descending(),
                PageRequest.of(0, limit));
        
        // Build tree structure in memory
        CommentTree tree = buildTree(comments);
        
        // Cache for 60 seconds (hot posts get frequent requests!)
        redis.opsForValue().set(cacheKey, serialize(tree), 
            Duration.ofSeconds(60));
        
        return tree;
    }
    
    /**
     * Build nested tree from flat list using path-based approach.
     */
    private CommentTree buildTree(List<Comment> flatComments) {
        Map<String, CommentNode> nodeMap = new LinkedHashMap<>();
        List<CommentNode> roots = new ArrayList<>();
        
        for (Comment c : flatComments) {
            CommentNode node = new CommentNode(c);
            nodeMap.put(c.getId(), node);
            
            if (c.getParentId() == null) {
                roots.add(node); // Top-level comment!
            } else {
                CommentNode parent = nodeMap.get(c.getParentId());
                if (parent != null) {
                    parent.addChild(node);
                }
            }
        }
        
        return new CommentTree(roots, flatComments.size());
    }
    
    /**
     * Add a new comment (with path for nested structure!).
     */
    @Transactional
    public Comment addComment(String postId, String parentId, 
                              String userId, String body) {
        String path;
        int depth;
        
        if (parentId == null) {
            // Top-level comment
            path = null;
            depth = 0;
        } else {
            Comment parent = commentRepo.findById(parentId)
                .orElseThrow(() -> new NotFoundException("Parent not found"));
            path = parent.getPath() == null 
                ? parent.getId() 
                : parent.getPath() + "." + parent.getId();
            depth = parent.getDepth() + 1;
        }
        
        Comment comment = Comment.builder()
            .id(UUID.randomUUID().toString())
            .postId(postId)
            .parentId(parentId)
            .path(path)
            .depth(depth)
            .authorId(userId)
            .body(body)
            .score(1) // Auto-upvote by author!
            .createdAt(Instant.now())
            .build();
        
        commentRepo.save(comment);
        
        // Increment post's comment count (async!)
        redis.opsForHash().increment("post:" + postId, "comment_count", 1);
        
        // Invalidate comment tree cache!
        redis.delete("comments:" + postId + ":*");
        
        return comment;
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you handle a viral post getting 10,000 votes per second?**
> Never update the DB score directly (row-level lock contention!). Instead: (1) Redis INCRBY for real-time score (single-threaded, no locks!), (2) Each vote published to Kafka topic, (3) Kafka consumer batches votes and does bulk DB update every 5 seconds (single UPDATE with accumulated delta), (4) Client reads score from Redis (always fresh!). This turns 10K writes/sec into 1 DB write every 5 seconds. For extreme cases: shard the counter — 10 Redis keys that all get summed for display (write to random shard, read all shards and sum!).

**Q2: How would you design the home feed for a user subscribed to 500 subreddits?**
> Pre-compute approach: (1) Each subreddit has a sorted set of hot posts (refreshed every 5 minutes by background job), (2) When user requests feed: ZUNIONSTORE across subscribed subreddits into a temporary user feed key (TTL 5 min), (3) Paginate using ZREVRANGEBYSCORE with cursor. Optimization: for users with many subscriptions, use a pre-computed "popular across subscriptions" feed generated by a Spark job. Personalization: weight subreddits by user engagement (subreddits they comment in rank higher!). Cache the first 2 pages aggressively (90% of users never scroll past page 2!).

**Q3: How do you store and query nested comments efficiently?**
> Materialized path (Reddit's approach): each comment stores the full path of ancestor IDs (e.g., "1.5.23.42"). To load a subtree: `WHERE path LIKE '1.5.%'` (fast with B-tree index on path!). To load top-level + first 2 levels: `WHERE post_id = ? AND depth <= 2 ORDER BY score DESC LIMIT 200`. Deeper comments loaded on demand ("load more replies"). For hot posts: cache entire comment tree as JSON in Redis (denormalized, fast to serve!). Trade-off vs closure table: less flexible queries but simpler writes and no space explosion.

**Q4: Why does Reddit use time-decay in the hot ranking instead of just sorting by votes?**
> Without time decay: a 5-year-old post with 100K votes would permanently occupy the top spot. No new content could ever surface! Reddit's formula `log10(score) + time_factor` ensures: (1) Time dominates — a new post always starts competitive, (2) Logarithmic votes — diminishing returns prevent runaway dominance, (3) Fresh content surfaces naturally — after 24 hours, a post needs exponentially MORE votes to stay on top. This creates the "front page" effect: always fresh, always relevant. Hackernews uses similar decay (gravity formula). The constant (45000) tunes how fast posts age off.

---

## 🔗 Related Topics
- [Caching Strategies](../BuildingBlocks/CachingStrategies.md) — Feed caching
- [Push vs Pull](../Tradeoffs/Push_vs_Pull_Architecture.md) — Feed generation
- [Database Indexing](../Database/Indexing.md) — Comment tree queries
- [Design Facebook](./DesignFacebook.md) — Similar feed system

---

*"Reddit's brilliance isn't in what it shows you — it's in what it hides. The ranking algorithm, the vote fuzzing, the spam detection — 90% of the system design is about preventing bad content from reaching users while making good content surface effortlessly. A well-designed social platform should feel like serendipity, but it's actually math."* 📱
