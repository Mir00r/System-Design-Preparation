# 🎵 Design Spotify: Music Streaming at Scale

> *"Spotify serves 600 million users with 100 million songs. When you hit play, audio must start streaming within 200ms. The recommendation engine processes 4 PETABYTES of data daily to suggest your next favorite song. This is a system that must handle real-time streaming, massive personalization, and global distribution — all while keeping your music playing without a single buffer."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [CDN](../../BuildingBlocks/CDN.md), [Caching](../../BuildingBlocks/Caching.md), [Message Queues](../../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Scale Estimation](#-scale-estimation)
3. [High-Level Architecture](#-high-level-architecture)
4. [Core Components](#-core-components)
5. [Music Storage & Delivery](#-music-storage--delivery)
6. [Search System](#-search-system)
7. [Recommendation Engine](#-recommendation-engine)
8. [Playlist Service](#-playlist-service)
9. [Data Model](#-data-model)
10. [Deep Dives](#-deep-dives)
11. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ Users can search for songs, artists, albums, playlists
✅ Users can stream songs (play, pause, skip, seek)
✅ Users can create and manage playlists
✅ Personalized recommendations (Discover Weekly, Daily Mix)
✅ Social features (follow artists, share playlists)
✅ Offline download support
✅ Cross-device sync (phone, desktop, web)
```

### Non-Functional Requirements
```
✅ Low latency playback (< 200ms to start streaming)
✅ High availability (99.99% — music should never stop!)
✅ Support 600M users, 200M daily active
✅ 100M songs in catalog
✅ Global distribution (users in 180+ countries)
✅ Smooth playback (no buffering on stable connection)
```

---

## 📊 Scale Estimation

```
USERS:
  • 600M total users, 200M DAU
  • Average session: 30 minutes
  • Average songs per session: 10 songs
  • Peak concurrent users: 50M

STORAGE:
  • 100M songs × 5MB average (compressed) = 500 TB audio files
  • Multiple quality levels (low/normal/high/lossless):
    500TB × 4 qualities = 2 PB total audio storage
  • Metadata: 100M songs × 1KB = 100 GB

BANDWIDTH:
  • 200M DAU × 10 songs × 5MB = 10 PB/day transfer
  • Peak: 50M concurrent × 160kbps = 8 Tbps bandwidth!
  • CDN absorbs most of this (hot songs cached at edge)

API CALLS:
  • 200M DAU × ~100 API calls/session = 20B API calls/day
  • ~230K requests/second average, ~1M peak
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                         CLIENTS                                     │
│    📱 Mobile App    💻 Desktop App    🌐 Web Player                │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                     ┌─────────▼─────────┐
                     │   API Gateway      │ (Authentication, Rate Limiting)
                     │   + Load Balancer  │
                     └─────────┬─────────┘
                               │
         ┌─────────────────────┼───────────────────────────┐
         │                     │                           │
         ▼                     ▼                           ▼
┌──────────────┐    ┌──────────────────┐      ┌──────────────────┐
│   User       │    │   Music          │      │  Search          │
│   Service    │    │   Streaming      │      │  Service         │
│              │    │   Service        │      │  (Elasticsearch) │
└──────────────┘    └────────┬─────────┘      └──────────────────┘
                             │
         ┌───────────────────┼───────────────────────┐
         │                   │                       │
         ▼                   ▼                       ▼
┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  Playlist    │    │  Recommendation  │    │  Social          │
│  Service     │    │  Engine          │    │  Service         │
└──────────────┘    └──────────────────┘    └──────────────────┘
         │                   │
         ▼                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                      DATA LAYER                                   │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐  ┌────────────────┐  │
│  │PostgreSQL│  │Cassandra │  │  Redis    │  │  Object Storage│  │
│  │(metadata)│  │(activity)│  │  (cache)  │  │  (audio files) │  │
│  └─────────┘  └──────────┘  └───────────┘  └────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
                               │
                     ┌─────────▼─────────┐
                     │   CDN (Edge)      │ ← Audio files cached globally!
                     │   (CloudFront/    │
                     │    Fastly)        │
                     └───────────────────┘
```

---

## 🧩 Core Components

### 1. Music Streaming Service

```
PLAYBACK FLOW:
  1. Client: "Play song ABC" → API Gateway
  2. API: Check user subscription tier (free/premium)
  3. API: Determine audio quality (based on tier + connection)
  4. API: Generate signed CDN URL (time-limited, user-specific)
  5. Client: Stream audio chunks from CDN
  6. Client: Report playback events (for royalties + recommendations)

CDN STRATEGY:
  ┌─────────────────────────────────────────────────────────────┐
  │  Hot songs (top 1% = 1M songs):                             │
  │    Cached at ALL CDN edge locations worldwide               │
  │    = 99% of plays served from edge (< 50ms latency!)        │
  │                                                             │
  │  Warm songs (top 20% = 20M songs):                          │
  │    Cached at regional CDN nodes                             │
  │    = served from nearby region (< 200ms)                    │
  │                                                             │
  │  Cold songs (bottom 80% = 80M songs):                       │
  │    Stored in origin (S3/GCS)                                │
  │    = served from origin on first request, then cached       │
  └─────────────────────────────────────────────────────────────┘
```

### 2. Audio File Storage

```
AUDIO ENCODING:
  Each song stored in multiple formats/qualities:
  ┌─────────────────────────────────────────────────┐
  │  Quality       │ Format │ Bitrate  │  File Size │
  ├─────────────────────────────────────────────────┤
  │  Low (mobile)  │ OGG    │ 24 kbps  │  ~1 MB     │
  │  Normal        │ OGG    │ 96 kbps  │  ~3.5 MB   │
  │  High          │ OGG    │ 160 kbps │  ~5 MB     │
  │  Very High     │ OGG    │ 320 kbps │  ~10 MB    │
  │  Lossless      │ FLAC   │ ~1000kbps│  ~30 MB    │
  └─────────────────────────────────────────────────┘
  
  Storage: S3/GCS with CDN in front
  Chunked: Songs split into 5-second chunks for adaptive streaming
```

---

## 🔍 Search System

```
SEARCH REQUIREMENTS:
  • Fuzzy matching (typo-tolerant: "Beattles" → "Beatles")
  • Multi-entity (songs, artists, albums, playlists, podcasts)
  • Autocomplete (suggest as user types)
  • Personalized ranking (show YOUR artists first)

ARCHITECTURE:
  ┌──────────┐   query    ┌────────────────┐   ranked results
  │  Client  │──────────►│  Search API    │──────────────────►
  └──────────┘           └───────┬────────┘
                                 │
                    ┌────────────┼────────────────┐
                    ▼            ▼                ▼
            ┌────────────┐ ┌────────────┐ ┌────────────────┐
            │ Songs Index│ │Artist Index│ │Playlist Index  │
            │(Elastic)   │ │(Elastic)   │ │(Elastic)       │
            └────────────┘ └────────────┘ └────────────────┘
  
RANKING FACTORS:
  • Text relevance (BM25 score)
  • Popularity (global play count)
  • Personal relevance (user's listening history)
  • Recency (newer releases boosted)
  • Social signals (friends listen to this artist)
```

---

## 🤖 Recommendation Engine

```
SPOTIFY'S THREE RECOMMENDATION MODELS:

1. COLLABORATIVE FILTERING:
   "Users who like Song A also like Song B"
   Based on 600M users' listening patterns!
   
   User-song matrix (600M × 100M) → Matrix factorization
   → Similar user clusters → Recommend what cluster listens to

2. CONTENT-BASED (Audio Analysis):
   Analyze actual audio: tempo, key, energy, danceability
   "This song SOUNDS like songs you like"
   
   CNN analyzes raw audio → 128-dim embedding
   Find nearest neighbors in embedding space

3. NLP ON PLAYLISTS/REVIEWS:
   "Songs that appear in playlists with similar titles"
   "Chill Vibes" playlist → other songs in "Chill" playlists
   
   Word2Vec on playlist track sequences → Song embeddings

DISCOVER WEEKLY PIPELINE:
  ┌──────────┐    ┌──────────────┐    ┌───────────┐    ┌──────────┐
  │ Listening │───►│ Spark Batch  │───►│ Candidate │───►│ 30 songs │
  │ History   │    │ (daily)      │    │ Ranking   │    │ playlist │
  │ (30 days) │    │              │    │ Model     │    │ (Monday) │
  └──────────┘    └──────────────┘    └───────────┘    └──────────┘
  
  Runs every Sunday night. Processes 4 PB of listening data!
  Uses collaborative filtering + content analysis + NLP.
```

---

## 📝 Playlist Service

```
DATA MODEL:
  Playlist:
    id: UUID
    owner_id: user_id
    name: "My Running Mix"
    description: "..."
    tracks: [song_id_1, song_id_2, ...] (ordered list!)
    followers: 1,234,567
    collaborative: true/false
    
CHALLENGES:
  1. Large playlists (some have 10,000+ songs!)
  2. Collaborative editing (multiple users add/remove simultaneously)
  3. Ordering (drag to reorder must be consistent)
  
SOLUTION: 
  • Store playlist tracks in Cassandra (wide-column: playlist_id → tracks)
  • Use fractional indexing for ordering (avoids rewriting all positions)
  • Collaborative: operational transform (like Google Docs!)
  • Cache hot playlists in Redis (Sorted Sets for ordered tracks)
```

---

## 🗄️ Data Model

```sql
-- PostgreSQL (strong consistency for core entities)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    display_name VARCHAR(100),
    subscription_tier VARCHAR(20), -- free, premium, family
    country VARCHAR(2),
    created_at TIMESTAMP
);

CREATE TABLE songs (
    id UUID PRIMARY KEY,
    title VARCHAR(500),
    artist_id UUID REFERENCES artists(id),
    album_id UUID REFERENCES albums(id),
    duration_ms INT,
    audio_url VARCHAR(500), -- S3/GCS path
    popularity_score FLOAT, -- 0-100
    release_date DATE
);

CREATE TABLE playlists (
    id UUID PRIMARY KEY,
    owner_id UUID REFERENCES users(id),
    name VARCHAR(200),
    is_public BOOLEAN,
    follower_count INT DEFAULT 0
);
```

```
-- Cassandra (high write throughput for activity data)
CREATE TABLE listening_history (
    user_id UUID,
    listened_at TIMESTAMP,
    song_id UUID,
    duration_played_ms INT,
    context VARCHAR, -- playlist, album, radio, search
    PRIMARY KEY (user_id, listened_at)
) WITH CLUSTERING ORDER BY (listened_at DESC);
-- Partition by user_id, ordered by time. Fast "recently played" queries!

CREATE TABLE playlist_tracks (
    playlist_id UUID,
    position FLOAT, -- Fractional index for ordering
    song_id UUID,
    added_by UUID,
    added_at TIMESTAMP,
    PRIMARY KEY (playlist_id, position)
);
```

---

## 🔍 Deep Dives

### Gapless Playback & Crossfade
```
  Challenge: No silence between songs!
  Solution: Prefetch next song while current is playing.
  
  Playing Song A (3:45 long):
    At 3:30 → Begin downloading Song B chunks
    At 3:43 → Start decoding Song B audio
    At 3:45 → Seamlessly switch (or crossfade 3 seconds)
  
  Client maintains 30-second buffer ahead of playback position.
```

### Offline Mode
```
  Premium users can download songs for offline listening.
  
  Download: Encrypted audio file + license (expires in 30 days)
  DRM: Widevine (Android) / FairPlay (iOS)
  Sync: When online, sync "offline available" metadata
  Limit: 10,000 songs per device, 5 devices per account
```

### Royalty Tracking
```
  Every play event must be accurately recorded for artist payments!
  
  Play event → Kafka → Royalty Calculator (batch, daily)
  
  Rules:
    • Minimum 30 seconds played = counts as a "stream"
    • Pro-rata model: artist share = artist_streams / total_streams × revenue
    • Monthly payout, minimum threshold $10
  
  Scale: 1B+ play events per day → processed in Spark batch jobs
```

---

## 🎮 Mini Challenge

Design the "Recently Played" feature for 600M users.

<details>
<summary>🔑 Answer</summary>

**Requirements:** Show last 50 songs played, across all devices, real-time sync.

**Design:**
- Write path: On play → publish to Kafka topic `play-events`
- Consumer: Writes to Cassandra (user_id, timestamp, song_id) with TTL 90 days
- Read path: `SELECT * FROM listening_history WHERE user_id = ? LIMIT 50`
- Cache: Redis List per user (LPUSH on play, LTRIM to 50 items), TTL 24h
- Cross-device: WebSocket push when play event received from another device

**Why this works:**
- Cassandra: High write throughput, partition by user_id
- Redis: Sub-ms reads for the hot path (opening app)
- Kafka: Decouples play tracking from all downstream consumers
</details>

---

## ❓ Interview Q&A

**Q1: How would you design the audio streaming pipeline?**
> Audio stored in S3/GCS in multiple quality levels (24kbps to lossless). CDN caches popular songs at edge. On play: API generates time-limited signed URL → client streams chunks from nearest CDN node. Adaptive bitrate: client switches quality based on network speed. Prefetch next song for gapless playback.

**Q2: How does Spotify handle 50M concurrent streams?**
> CDN handles 99% of audio delivery (hot songs cached globally). Origin servers only serve cache misses for unpopular songs. Audio chunked into 5-second segments for efficient caching. Multiple CDN providers for redundancy. Client-side buffering (30s ahead) absorbs brief network issues.

**Q3: How would you design the recommendation system?**
> Three-model approach: (1) Collaborative filtering on user-song interaction matrix (600M users × 100M songs), (2) Content-based on audio features (CNN embeddings), (3) NLP on playlist/context data. Batch pipeline (Spark) processes daily for Discover Weekly. Real-time model for "Up Next" recommendations using online features (current session context).

**Q4: How do you track plays for royalty payments?**
> Every play event published to Kafka (at-least-once). Play = 30+ seconds listened. Events consumed by royalty calculation pipeline (daily Spark batch). Pro-rata model distributes monthly revenue proportionally. Idempotency: deduplicate by (user_id, song_id, timestamp) within 30-second window. Audit trail kept for 7 years.

**Q5: How would you handle the "cold start" problem for new users?**
> New user: no listening history for collaborative filtering! Solutions: (1) Onboarding quiz (pick favorite artists/genres), (2) Demographics-based recommendations (popular in your country/age group), (3) Content-based from first few songs played, (4) Trending/viral songs as default. Transition to personalized within 1-2 weeks of usage.

---

## 🔗 Related Topics
- [CDN](../../BuildingBlocks/CDN.md) — Audio file delivery
- [Caching Strategies](../../BuildingBlocks/CachingStrategies.md) — Playlist and metadata caching
- [Search Index](../../BuildingBlocks/SearchIndex.md) — Music search
- [Batch vs Stream](../../Tradeoffs/Batch_vs_Stream_Processing.md) — Recommendation pipeline

---

*"Spotify doesn't just stream music — it streams a personalized radio station that knows you better than you know yourself." — Music Tech* 🎵
