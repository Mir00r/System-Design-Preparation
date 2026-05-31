# 🔥 Design Tinder (Dating App)

> *"50 million users swiping through potential matches — that's billions of 'recommendations' per day, each needing to be fresh, relevant, and fast (< 200ms). Tinder's system design combines location-based search, recommendation engines, real-time messaging, and the psychology of 'the swipe.' It's a masterclass in building systems that handle massive scale while delivering personalized experiences."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Location Service](DesignLocationService.md), [Caching](../BuildingBlocks/Caching.md), [Message Queues](../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Core Challenges](#-core-challenges)
3. [High-Level Architecture](#-high-level-architecture)
4. [Recommendation Engine](#-recommendation-engine)
5. [Swipe & Match System](#-swipe--match-system)
6. [Real-Time Messaging](#-real-time-messaging)
7. [Location & Proximity](#-location--proximity)
8. [Java Implementation](#-java-implementation)
9. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Create profile (photos, bio, preferences)
  • Discover nearby users (within distance preference)
  • Swipe right (like) or left (pass)
  • Mutual like → MATCH! (notification to both!)
  • Chat with matches (real-time messaging!)
  • Preferences: age range, distance, gender
  • Super Like, Boost (premium features)
  
NON-FUNCTIONAL:
  • 50M DAU, each sees ~100 profiles/day → 5B recommendations/day!
  • Swipe latency: < 200ms (instant feel!)
  • Match detection: < 1 second (excitement!)
  • Location updates: near real-time (moves to new area!)
  • Storage: 500M profiles × 6 photos each = PETABYTES!

SCALE NUMBERS:
  • 50M daily active users
  • 2B swipes per day
  • 50M matches per day
  • 100 profiles served per user session
  • Each profile: ~5 KB data + 6 photos (~3 MB total)
```

---

## 🎯 Core Challenges

```
CHALLENGE 1: RECOMMENDATION AT SCALE
  User A needs 100 profiles to swipe on.
  Constraints:
  • Must be within distance preference (geo query!)
  • Must match age/gender preferences (BOTH ways!)
  • Must NOT show: already swiped, already matched, blocked users!
  • Should be "attractive" recommendations (engagement!)
  
  Naive approach: query ALL users within 50km, filter → TOO SLOW!
  Need: pre-computed candidate pools + real-time ranking!

CHALLENGE 2: AVOIDING DOUBLE-SHOWS
  User A sees User B → swipes left.
  User A opens app tomorrow → MUST NOT see User B again!
  
  With 50M users, each swiping 100/day:
  Per user: 100 swipes/day × 365 days = 36,500 "seen" profiles/year!
  Total: 50M × 36,500 = 1.8 TRILLION seen-pairs! 💀
  
  Can't store all pairs! Solutions: Bloom filters, time-windowed sets.

CHALLENGE 3: MUTUAL MATCH DETECTION
  User A swipes right on User B.
  Later, User B swipes right on User A.
  → Instantly notify both: "IT'S A MATCH!" 🎉
  
  Must check: "Did the OTHER person already like ME?"
  Billions of likes stored — need O(1) lookup!

CHALLENGE 4: REAL-TIME LOCATION
  User moves from downtown to suburbs → recommendations should change!
  But: can't re-compute recommendations on every GPS update!
  Solution: batch location updates, coarse geo-cells!
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        TINDER ARCHITECTURE                            │
│                                                                       │
│  ┌───────────┐    ┌──────────────────────────────────────────────┐  │
│  │  Mobile   │    │  API Gateway (authentication, rate limiting)  │  │
│  │  App      │───►│  /discover, /swipe, /match, /chat            │  │
│  └───────────┘    └────────┬───────────────┬───────────────┬─────┘  │
│                            │               │               │         │
│            ┌───────────────▼──┐  ┌─────────▼────┐  ┌──────▼──────┐ │
│            │  Recommendation   │  │  Swipe &     │  │  Chat       │ │
│            │  Service          │  │  Match Svc   │  │  Service    │ │
│            │  • Geo filter     │  │  • Record    │  │  • WebSocket│ │
│            │  • Score & rank   │  │  • Check match│  │  • History  │ │
│            │  • Batch generate │  │  • Notify!   │  │  • Media    │ │
│            └───────┬──────────┘  └──────┬───────┘  └──────┬──────┘ │
│                    │                     │                  │         │
│  ┌────────────────┐│  ┌────────────────┐│  ┌─────────────┐│         │
│  │Profile Service ││  │  Likes Store   ││  │  Message DB  ││         │
│  │(user data,     ││  │  (who liked    ││  │  (Cassandra) ││         │
│  │ photos, prefs) ││  │   whom?)       ││  │              ││         │
│  └────────┬───────┘│  └────────┬───────┘│  └─────────────┘│         │
│           │         │           │         │                  │         │
│  ┌────────▼───────┐ │  ┌───────▼────────┐│                  │        │
│  │   PostgreSQL   │ │  │    Redis        ││                  │        │
│  │   (profiles)   │ │  │    (likes,      ││                  │        │
│  │                │ │  │     bloom       ││                  │        │
│  └────────────────┘ │  │     filters)    ││                  │        │
│                      │  └────────────────┘│                  │        │
│  ┌───────────────────▼───────────┐                           │        │
│  │  Geo Index (location-based!)   │                           │        │
│  │  Redis GEO / Elasticsearch     │                           │        │
│  │  "All users within 50km"       │                           │        │
│  └────────────────────────────────┘                           │        │
│                                                                       │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Notification Service (Push: match alerts, messages, boosts!)  │  │
│  └────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🧠 Recommendation Engine

```
GENERATING THE "DECK" (stack of profiles to swipe on):

STEP 1: GEO FILTERING (coarse → fine)
  User A: location (40.7128, -74.0060), distance_pref = 30km
  
  Coarse: geohash prefix → all users in same geo-cell!
  Fine: calculate actual distance (Haversine formula)
  
  Redis GEO: GEORADIUS users 40.7128 -74.0060 30 km
  → Returns candidate set (~10,000 users in 30km radius)

STEP 2: PREFERENCE FILTERING
  User A wants: Women, 25-35, within 30km
  Filter candidates:
  • Gender matches A's preference AND A matches their preference!
  • Age within both users' ranges (mutual compatibility!)
  • Not already swiped (Bloom filter check!)
  • Not blocked/reported
  
  → Reduced to ~2,000 candidates

STEP 3: SCORING & RANKING
  Score each candidate on:
  • "Desirability" score (how many right-swipes they get)
  • Profile completeness (more photos = higher rank)
  • Activity recency (active today > last seen 3 weeks ago)
  • Mutual friends/interests (if available)
  • Elo-like rating (match similar "attractiveness" tiers)
  
  → Top 100 candidates → User's "deck" for today!

STEP 4: PRE-COMPUTATION (batch!)
  Don't compute deck in real-time on every app open!
  
  Background job (every few hours):
  For each active user:
    1. Run geo + preference filter
    2. Score and rank candidates
    3. Store top 200 in Redis: "deck:userA" → [user47, user92, ...]
  
  On app open: LPOP from pre-computed deck! (O(1)!)
  Deck empty? Trigger async re-computation!

THE ELO SYSTEM (simplified):
  Every user has a hidden "score" (like chess Elo!):
  • Get right-swiped → your score increases!
  • Get left-swiped → your score decreases!
  • Match with high-score user → your score increases MORE!
  
  Purpose: Show users profiles at similar "tier"!
  → Higher match probability → better user experience!
  
  (Tinder denies using "Elo" but uses similar concepts)
```

---

## ❤️ Swipe & Match System

```
SWIPE RECORDING & MATCH DETECTION:

  User A swipes RIGHT on User B:
  
  ┌──────┐     ┌────────────────┐     ┌────────────────┐
  │  App │────►│  Swipe Service │────►│  Match Check   │
  └──────┘     └────────┬───────┘     └───────┬────────┘
                        │                      │
                Store A→B like!         Did B already like A?
                        │                      │
                        ▼                      ▼
                ┌────────────────┐     ┌────────────────┐
                │  Redis SET     │     │  Redis SET     │
                │  "likes:A"    │     │  "likes:B"     │
                │  → add B      │     │  → SISMEMBER A │
                └────────────────┘     └────────────────┘
                                              │
                                    A in likes:B?
                                    YES → MATCH! 🎉
                                    NO → just record, no notification.

MATCH DETECTION (O(1)):
  Redis: SISMEMBER likes:userB userA
  → "Did User B already swipe right on User A?"
  → YES! → Create match, notify both!

DATA STRUCTURES:
  Likes:    SET per user → "likes:userA" = {B, C, F, G, ...}
  Passes:   Bloom filter per user → "seen:userA" (space-efficient!)
  Matches:  Sorted SET → "matches:userA" = {B: timestamp, G: timestamp}

WHY BLOOM FILTER FOR "SEEN"?
  User swipes 100/day × 365 = 36,500 profiles/year
  SET: 36,500 × 36 bytes (UUID) = 1.3 MB per user
  × 50M users = 65 TB! 💀
  
  Bloom filter: ~1 KB per user (with 1% false positive!)
  × 50M users = 50 GB ← manageable! ✅
  
  False positive = "already seen" when not seen → user skipped.
  Acceptable! (slightly fewer recommendations, but saves 65 TB!)

MATCH NOTIFICATION:
  Match detected! →
  1. Write to matches DB (both users)
  2. Create chat room
  3. Push notification to User B (if online: WebSocket!)
  4. Push notification to User A (confirmation: "It's a match!")
  5. Show match animation in app! 🎊
```

---

## 💬 Real-Time Messaging

```
AFTER MATCH: Users can chat!

  WebSocket-based real-time messaging:
  
  User A ──WebSocket──► Chat Service ──WebSocket──► User B
                              │
                        Cassandra (persistence!)
                              │
                        "conversation:matchId" →
                        [{sender:A, text:"Hey!", ts:1700000}]

ARCHITECTURE:
  • WebSocket servers (horizontal scaling with sticky sessions!)
  • Each user connects to ONE WebSocket server
  • Server knows: "User A is on WS-Server-3"
  • When A sends message to B:
    1. Persist to Cassandra
    2. Lookup: "Which WS server is B connected to?" (Redis!)
    3. Forward message to that WS server → push to B!
    4. If B is offline → store, send push notification!

MESSAGE STORAGE:
  Cassandra table:
  CREATE TABLE messages (
    conversation_id UUID,
    message_id TIMEUUID,
    sender_id UUID,
    content TEXT,
    sent_at TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
  ) WITH CLUSTERING ORDER BY (message_id DESC);
  
  Partition by conversation_id → all messages for a chat together!
  Ordered by time → efficient pagination!
```

---

## 📍 Location & Proximity

```
LOCATION UPDATES:
  App sends location every 5 minutes (when active).
  NOT every second! (battery drain + unnecessary precision!)
  
  Storage: Redis GEO
  GEOADD users:active longitude latitude userA
  
  When user changes city → new geo-cell → re-compute deck!
  Within same cell → keep current deck (no change needed!)

GEO-CELL APPROACH (for pre-computation):
  Divide world into cells (geohash level 4 = ~40km cells)
  
  Pre-compute per cell:
  • cell:9q8y:users = [all active users in this cell]
  • On deck generation: only look at users in SAME + ADJACENT cells!
  
  User moves to new cell → trigger deck refresh!
  User stays in same cell → no re-computation!

DISTANCE CALCULATION:
  Haversine formula (great-circle distance on Earth surface):
  d = 2r × arcsin(√(sin²((φ₂-φ₁)/2) + cos(φ₁)cos(φ₂)sin²((λ₂-λ₁)/2)))
  
  In practice: Redis GEODIST handles this efficiently!
  GEODIST users userA userB km → "12.5" (km between them!)
```

---

## 💻 Java Implementation

### Recommendation Service

```java
@Service
public class RecommendationService {
    
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private UserProfileRepository profileRepo;
    @Autowired private BloomFilterService bloomFilter;
    
    /**
     * Get next batch of profiles to show user.
     * Uses pre-computed deck (fast!) or generates on-the-fly if empty.
     */
    public List<UserProfile> getRecommendations(String userId, int count) {
        String deckKey = "deck:" + userId;
        
        // Pop from pre-computed deck
        List<String> candidateIds = redis.opsForList()
            .leftPop(deckKey, count);
        
        if (candidateIds == null || candidateIds.size() < count) {
            // Deck empty! Generate fresh recommendations.
            candidateIds = generateFreshDeck(userId, count);
        }
        
        return profileRepo.findAllById(candidateIds);
    }
    
    /**
     * Generate fresh recommendation deck.
     */
    private List<String> generateFreshDeck(String userId, int count) {
        UserProfile user = profileRepo.findById(userId).orElseThrow();
        
        // 1. Geo filter: find users within distance preference
        List<String> nearby = geoFilter(user);
        
        // 2. Preference filter: age, gender (mutual!)
        List<String> compatible = nearby.stream()
            .filter(id -> isCompatible(user, profileRepo.findById(id).orElse(null)))
            .collect(Collectors.toList());
        
        // 3. Remove already-seen (Bloom filter!)
        List<String> unseen = compatible.stream()
            .filter(id -> !bloomFilter.mightContain(userId, id))
            .collect(Collectors.toList());
        
        // 4. Score and rank
        List<ScoredCandidate> scored = unseen.stream()
            .map(id -> new ScoredCandidate(id, calculateScore(user, id)))
            .sorted(Comparator.comparingDouble(ScoredCandidate::score).reversed())
            .limit(200)
            .collect(Collectors.toList());
        
        // 5. Store in Redis deck for future pops
        List<String> deck = scored.stream()
            .map(ScoredCandidate::userId)
            .collect(Collectors.toList());
        
        if (!deck.isEmpty()) {
            redis.opsForList().rightPushAll("deck:" + userId, deck);
            redis.expire("deck:" + userId, Duration.ofHours(4));
        }
        
        return deck.subList(0, Math.min(count, deck.size()));
    }
    
    private List<String> geoFilter(UserProfile user) {
        // Redis GEO: find all users within distance preference
        Point center = new Point(user.getLongitude(), user.getLatitude());
        Distance radius = new Distance(user.getDistancePrefKm(), Metrics.KILOMETERS);
        
        GeoResults<RedisGeoCommands.GeoLocation<String>> results =
            redis.opsForGeo().radius("users:active", 
                new Circle(center, radius));
        
        return results.getContent().stream()
            .map(r -> r.getContent().getName())
            .filter(id -> !id.equals(user.getId()))
            .collect(Collectors.toList());
    }
}
```

### Swipe & Match Service

```java
@Service
public class SwipeService {
    
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private MatchRepository matchRepo;
    @Autowired private NotificationService notificationService;
    @Autowired private BloomFilterService bloomFilter;
    
    /**
     * Record a swipe and check for match!
     */
    public SwipeResult swipe(String swiperId, String targetId, boolean liked) {
        // Record in Bloom filter (seen!)
        bloomFilter.add(swiperId, targetId);
        
        if (!liked) {
            return SwipeResult.passed();
        }
        
        // Record the like
        redis.opsForSet().add("likes:" + swiperId, targetId);
        
        // Check for mutual like (MATCH?)
        Boolean mutualLike = redis.opsForSet()
            .isMember("likes:" + targetId, swiperId);
        
        if (Boolean.TRUE.equals(mutualLike)) {
            // IT'S A MATCH! 🎉
            return createMatch(swiperId, targetId);
        }
        
        return SwipeResult.liked();
    }
    
    private SwipeResult createMatch(String userA, String userB) {
        // Create match record
        Match match = new Match();
        match.setId(UUID.randomUUID().toString());
        match.setUserA(userA);
        match.setUserB(userB);
        match.setMatchedAt(Instant.now());
        matchRepo.save(match);
        
        // Add to both users' match lists
        redis.opsForZSet().add("matches:" + userA, userB, 
            System.currentTimeMillis());
        redis.opsForZSet().add("matches:" + userB, userA, 
            System.currentTimeMillis());
        
        // Remove from likes (cleanup!)
        redis.opsForSet().remove("likes:" + userA, userB);
        redis.opsForSet().remove("likes:" + userB, userA);
        
        // Notify both users!
        notificationService.sendMatchNotification(userA, userB);
        notificationService.sendMatchNotification(userB, userA);
        
        return SwipeResult.matched(match.getId());
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you handle the "already seen" problem at scale?**
> Bloom filter per user! At 1 KB per user with 1% false positive rate, 50M users = 50 GB total (vs 65 TB for exact sets!). False positive means a user occasionally misses seeing a potential match — acceptable trade-off! For premium users who exhaust their bloom filter: reset monthly, or use counting Bloom filter that allows removal. For the likes set (needs exact membership): use Redis SET (smaller — only right-swipes, not all swipes).

**Q2: How do you pre-compute recommendations efficiently?**
> Background job (Spark/Flink) runs every 2-4 hours: (1) Partition users by geo-cell (geohash level 4), (2) For each user: query same + adjacent cells for candidates, (3) Apply preference filters (mutual compatibility!), (4) Score using ML model (engagement prediction), (5) Store top 200 as Redis list per user. On app open: LPOP from list (O(1)!). Trigger async re-computation when deck runs low (<20 remaining). This handles 50M users because geo-partitioning makes it embarrassingly parallel!

**Q3: How do you detect a match in real-time?**
> On swipe-right: (1) Store "A likes B" → `SADD likes:A B`, (2) Check "B likes A?" → `SISMEMBER likes:B A`. Both are O(1) Redis operations! If mutual: create match, notify via WebSocket (if online) or push notification (if offline). Total latency: < 50ms from swipe to match notification! The key insight: you only need to check ONE direction — when A swipes right on B, check if B already liked A. The other direction was already checked when B swiped.

**Q4: How would you implement the "Boost" feature (show profile to more users)?**
> Boost puts your profile at the TOP of nearby users' decks for 30 minutes. Implementation: (1) "Boosted" flag in user profile + expiry time, (2) During deck generation: boosted users get artificially high score (guaranteed top position!), (3) For users with pre-computed decks: inject boosted profile at front of their Redis list (LPUSH), (4) Rate limit: max 1 boost per 12 hours, limit to ~100 insertions (nearby active users). Revenue model: boost is a paid feature ($5/use)!

---

## 🎮 Mini Challenge

Design the "Super Like" feature:
- User can only super-like 1 person per day (free tier)
- Super-liked profile gets special notification (even before mutual like!)
- Super likes are shown with a blue star border
- If super-liked user swipes right → match priority notification

*How does this change the data model and notification flow?*

---

## 🔗 Related Topics
- [Location Service](DesignLocationService.md) — Proximity search
- [WebSockets](../APIs/WebSockets.md) — Real-time messaging
- [Bloom Filters](../Database/BloomFilters.md) — Space-efficient seen tracking
- [Distributed Cache](DesignDistributedCache.md) — Deck storage

---

*"Tinder's genius isn't the swipe gesture — it's the recommendation system behind it. Showing the RIGHT profiles (ones likely to match!) drives engagement. Showing random profiles makes users frustrated and quit. The algorithm IS the product." — Dating App Engineer* 🔥
