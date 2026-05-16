# 🎬 Design YouTube
## Video Transcoding, Streaming at Scale, and the World's Second-Largest Search Engine

> *"YouTube serves 1 billion hours of video daily across 100+ countries. Every minute, 500 hours of new video are uploaded. The engineering challenge isn't playing video — it's transcoding every upload into 20+ formats, storing petabytes of data, and streaming it to any device at any quality level, instantly."*

**⏱️ Estimated Time**: 55 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [How to Approach System Design](./How_To_Approach_System_Design.md), [CDN](../BuildingBlocks/CDN.md), [Blob Storage](../BuildingBlocks/Blob_Storage.md)

---

## 📋 Requirements (RADIO: R)

### Functional Requirements

| Feature | Description |
|---|---|
| **Upload video** | Creators upload videos (up to 12 hours, 256GB) |
| **Transcode** | Convert to multiple resolutions and codecs (144p → 4K, H.264/VP9/AV1) |
| **Stream video** | Adaptive bitrate streaming (adjusts quality based on bandwidth) |
| **Search** | Full-text search on titles, descriptions, transcripts |
| **Recommendations** | Personalized video suggestions |
| **Comments & Likes** | Social engagement features |
| **Monetization** | Ad insertion at appropriate points |

### Non-Functional Requirements

```
Scale:
  - 2.5B monthly active users
  - 500 hours of video uploaded every minute
  - 1 billion hours watched per day
  - Average video: 7 minutes, 50MB after compression
  - Storage: multiple PB of video data
  - Global streaming: adaptive quality, buffer-free experience
  - Upload-to-playable: < 10 minutes for short videos, hours for 4K long-form
```

---

## 📡 API Design (RADIO: A)

```
POST /v1/videos/upload-url
  Request: { "title": "My Video", "file_size": 524288000, "content_type": "video/mp4" }
  Response: { "video_id": "v789", "upload_url": "https://upload.youtube.com/resumable/..." }

POST /v1/videos/{id}/metadata
  Request: { "title": "...", "description": "...", "tags": [...], "visibility": "public" }

GET /v1/videos/{id}
  Response: { "video_id": "...", "title": "...", "streaming_url": "...", "thumbnails": [...] }

GET /v1/videos/{id}/stream?quality=720p
  Response: HLS/DASH manifest with segment URLs

GET /v1/search?q=system+design&page=1
GET /v1/feed/recommendations?limit=20
```

---

## 🗄️ Data Model (RADIO: D)

```
Storage choices:
  - PostgreSQL: video metadata, users, channels
  - S3/GCS: raw uploads, transcoded segments, thumbnails
  - Bigtable/Cassandra: view counts, watch history (high-write, time-series)
  - Elasticsearch: video search (title, description, transcript, tags)
  - Redis: hot video metadata cache, trending rankings
  - CDN: serve video segments globally (95%+ cache hit rate for popular videos)

Video metadata schema:
  videos: {
    id, channel_id, title, description, tags[],
    duration_sec, upload_status (processing|ready|failed),
    resolutions_available: ["144p","360p","720p","1080p","4K"],
    view_count, like_count,
    created_at, published_at
  }

Video segment storage (on S3):
  /videos/{video_id}/original/raw.mp4
  /videos/{video_id}/720p/segment_001.ts
  /videos/{video_id}/720p/segment_002.ts
  /videos/{video_id}/1080p/segment_001.ts
  /videos/{video_id}/manifest.m3u8  (HLS playlist)
```

---

## 🏗️ Infrastructure (RADIO: I)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     YOUTUBE ARCHITECTURE                                │
│                                                                         │
│  UPLOAD PATH:                                                           │
│  [Creator] ──resumable upload──▶ [Upload Service] ──▶ [S3: raw video]  │
│                                        │                                │
│                                   [Kafka: video.uploaded]               │
│                                        │                                │
│                              ┌─────────▼──────────┐                     │
│                              │  Transcoding Pipeline│                    │
│                              │  (distributed workers)│                   │
│                              │                       │                   │
│                              │  1. Split into chunks │                   │
│                              │  2. Transcode parallel│                   │
│                              │  3. Generate thumbnails│                  │
│                              │  4. Extract audio      │                  │
│                              │  5. Generate subtitles │                  │
│                              │  6. Create manifests   │                  │
│                              └─────────┬─────────────┘                  │
│                                        ▼                                │
│                              [S3: transcoded segments]                   │
│                              [Kafka: video.ready] → notify creator      │
│                                                                         │
│  STREAMING PATH:                                                        │
│  [Viewer] ──▶ [CDN Edge] ──cache miss──▶ [Origin: S3]                  │
│                    │                                                    │
│              cache hit (95%+ for popular videos)                         │
│                    ▼                                                    │
│             [Adaptive Bitrate: HLS/DASH manifest]                       │
│             Player requests segments at appropriate quality              │
│                                                                         │
│  SEARCH + RECOMMENDATIONS:                                              │
│  [Search Service] ← Elasticsearch (title, transcript, tags)            │
│  [Recommendation Engine] ← ML model (collaborative filtering,          │
│                              watch history, content similarity)          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Video Transcoding Pipeline (Critical Path)

```
Raw video upload: 1080p, H.264, 2GB, 10 minutes

Step 1: CHUNKING (split into 10-second segments)
  → 60 chunks of ~33MB each
  → Each chunk can be transcoded independently (parallelism!)

Step 2: PARALLEL TRANSCODING (60 chunks × 6 resolutions = 360 jobs)
  Each transcoding worker processes one chunk at one resolution:
    chunk_001 → 144p (128×72)
    chunk_001 → 360p (640×360)
    chunk_001 → 720p (1280×720)
    chunk_001 → 1080p (1920×1080)
    chunk_001 → 1440p (2K)
    chunk_001 → 2160p (4K)
  Codecs: H.264 (compatibility) + VP9 (smaller, better quality) + AV1 (newest, smallest)

Step 3: MANIFEST GENERATION (HLS + DASH)
  master.m3u8 → lists all quality levels
    #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
    360p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=2500000,RESOLUTION=1280x720
    720p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=5000000,RESOLUTION=1920x1080
    1080p/playlist.m3u8

Step 4: CDN DISTRIBUTION
  Pre-warm CDN for popular channels (upload segment to edge before viewers request)
  Long-tail content: served from origin on first request, cached at edge after

Total time: 5-60 minutes depending on video length and resolution
  (360 parallel jobs on GPU-equipped workers significantly reduce wall time)
```

### Adaptive Bitrate Streaming (ABR)

```
How the player switches quality seamlessly:

  User's bandwidth: 5 Mbps initially
  Player downloads: 1080p segments (requires 5 Mbps) ✅

  Bandwidth drops to 2 Mbps (enters tunnel):
  Player detects buffering risk → switches to 720p segments (requires 2.5 Mbps)
  Then to 360p if bandwidth drops further

  Bandwidth recovers to 8 Mbps:
  Player upgrades back to 1080p

  The manifest (m3u8/mpd) lists segment URLs for ALL quality levels.
  The player decides which quality to request for each segment.
  Each segment is 2-10 seconds of video — switching happens at segment boundaries.
```

---

## 📈 Optimization (RADIO: O)

```
1. CDN CACHE EFFICIENCY
   Problem: 800M videos exist, but only 1% are watched frequently
   Solution: 
     - Hot tier: top 1% videos pre-cached at all CDN edges (covers 80% of views)
     - Warm tier: next 10% cached at regional CDN POPs
     - Cold tier: served directly from origin S3 (rarely accessed)
   Cache hit rate target: 95%+ (reduces origin bandwidth by 20×)

2. TRANSCODING COST OPTIMIZATION
   Problem: transcoding 500 hours/min at 6 resolutions + 3 codecs = enormous compute
   Solution:
     - Lazy transcoding: initially transcode only 360p + 720p
     - If video gets > 1000 views → trigger 1080p + 4K transcoding
     - 90% of videos get < 100 views → never need 4K transcoding
     - Saves 60%+ of transcoding compute cost

3. VIEW COUNT AT SCALE
   Problem: viral video gets 100K views/second → DB writes overwhelm
   Solution:
     - In-memory counter (Redis INCR) for real-time approximate count
     - Batch flush to Cassandra every 5 seconds
     - Exact count (for monetization) reconciled hourly from Kafka event log
```

---

## 🧩 Mini Challenge

**A 4K video (10GB, 2 hours long) is uploaded. Design the transcoding strategy to minimize time-to-first-playable while still producing all quality levels.**

<details>
<summary>💡 Click to reveal answer</summary>

**Strategy: Progressive transcoding with priority ordering**

1. **Immediately** (within 30 seconds): Start transcoding the first 2 minutes of the video in 360p only. This produces a "preview" that the creator can verify and viewers can start watching.

2. **Priority 1** (first 5 minutes): Transcode the full video in 720p (most-watched resolution). Use chunk-parallel processing: split the 2-hour video into 720 chunks (10s each), transcode all 720 chunks in parallel on 720 workers. Wall time: ~2 minutes.

3. **Priority 2** (next 15 minutes): Transcode 1080p using the same parallel strategy. Video is now "HD ready."

4. **Priority 3** (background, 30-60 minutes): Transcode 4K, 1440p, 360p, 144p. These run at lower priority on the transcoding cluster.

5. **Codec optimization** (hours later): Produce VP9 and AV1 encodes for bandwidth-efficient delivery. These use 30-50% less bandwidth than H.264 but take 5-10x longer to encode.

**Result**: Creator sees "video processing..." and within 5 minutes, 720p is available. Viewers start watching in 720p while 1080p and 4K render in the background. The manifest is updated dynamically as new resolutions become available — player seamlessly upgrades quality.

</details>

---

## 📝 Interview Q&A

**Q: How does adaptive bitrate streaming work?**
> A: The video is transcoded into multiple quality levels (144p to 4K), each split into 2-10 second segments. A manifest file (HLS .m3u8 or DASH .mpd) lists segment URLs for all quality levels. The player initially estimates bandwidth, requests segments at an appropriate quality, and continuously monitors download speed. If bandwidth drops, the next segment is requested at a lower quality — switching happens at segment boundaries with no rebuffering. If bandwidth improves, quality upgrades seamlessly. The CDN serves whichever segments the player requests.

**Q: How would you optimize storage costs for YouTube's scale?**
> A: (1) **Lazy transcoding**: Only transcode popular resolutions immediately (720p); add 4K/AV1 only if the video crosses a view threshold. 90% of videos never need 4K. (2) **Lifecycle policies**: Move raw uploads to Glacier after 30 days (creator can still request re-processing). (3) **Deduplication**: Detect duplicate uploads via perceptual hashing before transcoding. (4) **Efficient codecs**: AV1 produces 30% smaller files than H.264 — gradually re-encode popular back-catalog videos in AV1 to reduce CDN bandwidth costs.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/CDN.md](../BuildingBlocks/CDN.md)** — CDN architecture is the backbone of video streaming delivery
2. **[BuildingBlocks/Blob_Storage.md](../BuildingBlocks/Blob_Storage.md)** — Object storage patterns for video segments
3. **[SystemDesignCaseStudies/DesignNetflix.md](./DesignNetflix.md)** — Similar streaming architecture with different content model

---

*[← Design Instagram](./DesignInstagram.md) | [Back to Case Studies](./README.md) | [Next: Design Google Search →](./DesignGoogleSearch.md)*
