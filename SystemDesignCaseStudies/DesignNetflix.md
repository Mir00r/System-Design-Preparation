# 🎬 Design Netflix
## Video Streaming, Encoding Pipeline, and Global CDN at Scale

> *"Netflix serves 250 million subscribers in 190 countries. At peak, it accounts for 15% of global internet bandwidth. Every second of video you watch has been transcoded into 120+ different formats, cached on 17,000+ servers worldwide, and streamed via adaptive bitrate that adjusts to your exact connection speed in real time."*

**⏱️ Estimated Time**: 75 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [BuildingBlocks/CDN.md](../BuildingBlocks/CDN.md), [Design WhatsApp](./DesignWhatsApp.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Requirements (R)](#-requirements-r)
3. [API Design (A)](#-api-design-a)
4. [Data Model (D)](#-data-model-d)
5. [Infrastructure (I)](#-infrastructure-i)
6. [The Hard Part: Video Encoding Pipeline](#-the-hard-part-video-encoding-pipeline)
7. [Adaptive Bitrate Streaming (ABR)](#-adaptive-bitrate-streaming-abr)
8. [Open Connect CDN](#-open-connect-cdn)
9. [Optimization (O)](#-optimization-o)
10. [Code Examples](#-code-examples)
11. [Industry Examples](#-industry-examples)
12. [Common Pitfalls](#-common-pitfalls)
13. [Mini Challenge](#-mini-challenge)
14. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

Netflix uploads "Stranger Things Season 4." It's 9 episodes × 60 minutes = 9 hours of 4K video.

Before a single user can watch it, Netflix must:
1. **Transcode** 9 hours of raw video into **120+ versions** (4K, 1080p, 720p, 480p × multiple codecs × multiple audio tracks × subtitles)
2. **Distribute** those files to **17,000 servers** in ISP data centers worldwide
3. **Stream** it to users on every device (phone, TV, laptop) at the right quality for their current bandwidth — automatically adjusting mid-stream if their network fluctuates
4. **Handle** 250M subscribers potentially watching simultaneously at peak (Friday 9 PM)

The raw 4K master file for one episode = ~400 GB.
After transcoding to all variants = ~2-3 TB per episode.
For the full Stranger Things season = ~20 TB of transcoded video to distribute globally.

**Your job**: Design the system that does all of this at Netflix's scale.

---

## 📐 Requirements (R)

### Functional Requirements
```
Core features:
  ✅ Upload and process video content (admin/studio workflow)
  ✅ Stream video to users (smooth, adaptive quality)
  ✅ Support multiple devices (mobile, TV, web, game consoles)
  ✅ Resume playback (watch halfway, pick up on another device)
  ✅ Download for offline viewing
  ✅ Subtitles and multiple audio tracks

Out of scope:
  ❌ Recommendations / search (separate ML system)
  ❌ Billing / subscription management
  ❌ Live streaming (different architecture entirely)
```

### Non-Functional Requirements
```
Scale:
  - 250M subscribers, 100M+ streaming concurrently at peak
  - 15% of global internet bandwidth at peak
  - 36,000+ hours of content in catalog
  - ~15B hours watched per month

Performance:
  - Video start time: < 2 seconds (P99)
  - Seamless quality adaptation: < 500ms to switch bitrate
  - Zero buffering for users on stable connections

Reliability:
  - 99.99% availability for streaming
  - No video corruption or loss at any stage of the pipeline

Storage:
  - Each title = 2-3 TB (all variants)
  - 36,000 titles × 2.5 TB avg = ~90 PB total storage
```

---

## 🌐 API Design (A)

```
# Playback
GET    /v1/titles/{titleId}/playback-manifest?deviceProfile=TV_4K
  Response: {
    "manifestUrl": "https://cdn.nflxvideo.net/...",
    "licenseUrl": "https://drm.netflix.com/...",   // DRM
    "subtitleTracks": [...],
    "audioTracks": [...]
  }

# The manifest (HLS/DASH format, fetched from CDN)
GET    https://cdn.nflxvideo.net/stranger_things_s4e1/manifest.m3u8
  Response: HLS master playlist listing all quality variants and segment URLs

# Segments (actual video chunks)
GET    https://cdn.nflxvideo.net/stranger_things_s4e1/1080p/segment_001.ts
  Response: 2-4 second chunk of video (binary, ~2-5 MB)

# Watch history / resume
GET    /v1/users/{userId}/watch-progress/{titleId}
  Response: { "position": 1847, "deviceId": "...", "updatedAt": "..." }

POST   /v1/users/{userId}/watch-progress/{titleId}
  Request: { "position": 1847, "duration": 3600, "deviceId": "..." }
```

---

## 🗄️ Data Model (D)

### Storage Architecture

```
Content metadata   → MySQL (titles, episodes, genres, cast — relational)
Video segments     → S3 (object storage — 90 PB+ of video chunks)
CDN cache          → Open Connect (ISP-edge servers, populated from S3)
User data          → Cassandra (viewing history, watch progress — high write)
Search/Recommend   → Elasticsearch + custom ML pipeline
DRM licenses       → dedicated service (Widevine, FairPlay)
```

### Key Schemas

```sql
-- MySQL: Content catalog
CREATE TABLE titles (
    title_id     BIGINT PRIMARY KEY,
    title_name   VARCHAR(255) NOT NULL,
    title_type   ENUM('movie', 'series'),
    genres       JSON,
    rating       VARCHAR(10),    -- PG-13, R, etc.
    release_year INT,
    created_at   TIMESTAMP
);

CREATE TABLE episodes (
    episode_id   BIGINT PRIMARY KEY,
    title_id     BIGINT REFERENCES titles(title_id),
    season_num   INT,
    episode_num  INT,
    duration_sec INT,
    title        VARCHAR(255)
);

CREATE TABLE video_assets (
    asset_id       BIGINT PRIMARY KEY,
    episode_id     BIGINT REFERENCES episodes(episode_id),
    resolution     VARCHAR(10),  -- '4K', '1080p', '720p', '480p'
    codec          VARCHAR(20),  -- 'h264', 'h265', 'av1'
    bitrate_kbps   INT,
    manifest_url   TEXT,         -- HLS/DASH manifest S3 location
    status         ENUM('processing', 'ready', 'failed'),
    processed_at   TIMESTAMP
);
```

```
-- Cassandra: User viewing history (write-heavy, time-series)
CREATE TABLE watch_history (
    user_id      BIGINT,
    title_id     BIGINT,
    episode_id   BIGINT,
    progress_sec INT,          -- how far they watched (seconds)
    duration_sec INT,
    watched_at   TIMESTAMP,
    device_id    TEXT,
    PRIMARY KEY (user_id, watched_at)
) WITH CLUSTERING ORDER BY (watched_at DESC);

-- Watch progress (resume feature)
CREATE TABLE watch_progress (
    user_id      BIGINT,
    episode_id   BIGINT,
    position_sec INT,          -- where to resume
    updated_at   TIMESTAMP,
    PRIMARY KEY (user_id, episode_id)
);
```

---

## 🏗️ Infrastructure (I)

### High-Level System Architecture

```
                 ┌─────────────────────────────────┐
                 │          STUDIO UPLOAD           │
                 │  (Raw 4K master + audio/subs)    │
                 └──────────────┬──────────────────┘
                                │
                 ┌──────────────▼──────────────────┐
                 │     ENCODING PIPELINE           │
                 │   (AWS → Netflix's own infra)   │
                 │   Transcode → 120+ variants     │
                 └──────────────┬──────────────────┘
                                │
                 ┌──────────────▼──────────────────┐
                 │        AMAZON S3 (Master)       │
                 │   Origin Storage: 90 PB total   │
                 └──────────────┬──────────────────┘
                                │ proactively pushed
                 ┌──────────────▼──────────────────┐
                 │     OPEN CONNECT CDN            │
                 │  17,000 servers at ISPs         │
                 │  (last mile delivery)           │
                 └──────────────┬──────────────────┘
                                │ < 5ms to user
             ┌──────────────────┼──────────────────┐
    ┌─────────▼──────┐  ┌───────▼───────┐  ┌──────▼──────┐
    │    Netflix TV  │  │  Mobile App   │  │  Web Player │
    │                │  │               │  │             │
    └────────────────┘  └───────────────┘  └─────────────┘

    ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
    SEPARATE CONTROL PLANE (all non-video traffic):

    Client → AWS API Gateway → Microservices cluster
                → User Service     (auth, profile)
                → Catalog Service  (title metadata)
                → Playback Service (manifest URL generation)
                → Progress Service (watch history)
                → License Service  (DRM)
```

---

## 🎞️ The Hard Part: Video Encoding Pipeline

This is the most technically unique part of Netflix's architecture.

### Why 120+ Variants?

```
Dimensions to encode across:

  Resolution:  4K (3840×2160), 1080p, 720p, 480p, 360p, 240p
  Codec:       H.264 (universal compatibility)
               H.265/HEVC (50% better compression than H.264)
               AV1 (open-source, 30% better than H.265, for high-end devices)
               VP9 (Google's codec, used for Chrome/Android)
  Bitrate:     Multiple bitrate levels per resolution (for ABR)
  Audio:       5.1 surround, stereo, Dolby Atmos
  Languages:   Dubbed audio tracks (20+ languages for major titles)
  Subtitles:   Burned-in vs sidecar (50+ languages)

  Total: 6 resolutions × 4 codecs × 3 bitrates × 3 audio profiles = 200+ combinations
  (Netflix prunes combinations that are redundant → ~120 final variants per title)
```

### Encoding Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENCODING PIPELINE                            │
│                                                                 │
│  ┌──────────┐    ┌────────────┐    ┌──────────┐   ┌─────────┐  │
│  │  Studio  │───→│ Ingest     │───→│ Chunker  │──→│ Encoder │  │
│  │  Upload  │    │ Validator  │    │ (scenes) │   │ Workers │  │
│  └──────────┘    └────────────┘    └──────────┘   └────┬────┘  │
│                                                        │        │
│  ┌──────────────────────────────────────────────────────▼─────┐ │
│  │                 AWS/Netflix Compute Farm                   │ │
│  │   10,000+ encoding jobs run in parallel (AWS batch)        │ │
│  │   Each job = one (resolution × codec × audio) combination  │ │
│  └──────────────────────────────────────────────────────┬─────┘ │
│                                                         │        │
│  ┌──────────┐    ┌────────────┐    ┌──────────────────▼──────┐  │
│  │  Quality │←───│ Validation │←───│  Segment Packager       │  │
│  │  Check   │    │ (VMAF)     │    │  (MP4 → HLS/DASH segs)  │  │
│  └──────────┘    └────────────┘    └─────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Manifest Generator: creates .m3u8 master playlist       │   │
│  │  Upload all segments + manifest to S3                    │   │
│  │  Publish "title_ready" event → notify Open Connect       │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Netflix's Per-Shot Encoding Innovation

```
Traditional encoding:
  Encode each chunk with fixed quality settings → inefficient
  Action scene (complex motion) at same settings as static scene (simple) → wastes bits

Netflix's "Dynamic Optimizer" (2015 innovation):
  Analyze each shot independently
  Action shot → needs more bits → encode at higher bitrate
  Static scene → needs fewer bits → encode at lower bitrate
  
  Result: Same visual quality at 20% lower average file size
  At Netflix's scale: 20% of 90 PB = 18 PB saved in storage costs
```

---

## 📺 Adaptive Bitrate Streaming (ABR)

The most important concept for smooth video playback:

### How ABR Works

```
Video is NOT stored as one big file. It's chunked into 2-4 second segments:

  segment_001.ts  (0:00-0:04)
  segment_002.ts  (0:04-0:08)
  segment_003.ts  (0:08-0:12)
  ...
  segment_900.ts  (59:56-60:00)

Each segment exists in multiple quality levels:
  segment_001_4K.ts    (8 Mbps, 40 MB)
  segment_001_1080p.ts (5 Mbps, 25 MB)
  segment_001_720p.ts  (3 Mbps, 15 MB)
  segment_001_480p.ts  (1 Mbps,  5 MB)
```

### The ABR Algorithm (on the client)

```
Client continuously measures:
  - Current download speed (Mbps)
  - Buffer level (how many seconds of video are pre-buffered)

Decision logic (simplified):
  if buffer > 30 seconds AND speed > 8 Mbps:
      download next segment in 4K
  elif buffer > 15 seconds AND speed > 5 Mbps:
      download in 1080p
  elif buffer > 5 seconds AND speed > 3 Mbps:
      download in 720p
  else:
      download in 480p or 360p

Result:
  - Weak WiFi → plays at 480p smoothly (no buffering)
  - Strong connection → plays at 4K
  - Mid-stream network change → seamlessly downgrades/upgrades at next segment boundary
  - User never waits unless connection is extremely poor
```

### Video Start Time Optimization

```
Problem: First segment download + decode = startup latency

Netflix's solution:
  1. Prefetch manifest on title hover (before user clicks "Play")
  2. Start downloading first segment (in low-quality) immediately on click
  3. Begin playback as soon as 1-2 segments buffered (< 2 seconds)
  4. Upgrade quality as more segments buffer in background
```

---

## 🌍 Open Connect CDN

Netflix's most unique infrastructure component — they built their own CDN.

### Why Build a Custom CDN?

```
Problem with public CDNs (Akamai, CloudFront):
  - Netflix pays per-GB CDN costs
  - At 15% of global internet bandwidth, CDN costs would be enormous
  - Less control over delivery quality and cache behavior

Netflix's solution: Open Connect
  - 17,000+ servers deployed inside ISP data centers globally
  - ISPs get free hardware (Netflix's servers), faster video for their customers
  - Netflix pays zero transit costs for ISP-embedded traffic
  - Netflix controls cache placement, warming strategy, and routing

  "Netflix embedded a free server in your ISP's data center.
   Your video travels 1-2 hops instead of 15+"
```

### Open Connect Architecture

```
Traditional CDN:
  User → ISP → Internet Exchange → CDN PoP (100s of miles away) → Origin S3
  Latency: 50-200ms, multiple hops, transit costs

Netflix Open Connect:
  User → ISP → Open Connect Appliance (IN the ISP's building)
  Latency: < 5ms, 0-1 hops, no transit cost

Cache warming (proactive):
  - Netflix predicts what users will watch (based on day/time + recommendation data)
  - Night before: push expected popular content to nearby Open Connect appliances
  - By morning, Friday night's "most likely to be watched" content is already cached
  - Friday 9PM: 90%+ of content served from Open Connect, not S3
```

---

## ⚡ Optimization (O)

### Problem 1: 100M Concurrent Streams

```
100M concurrent streams × 5 Mbps avg = 500 Tbps of bandwidth

Solutions:
  - Open Connect handles 95%+ of this (no centralized bottleneck)
  - Remaining 5% (new content not yet on edge) served from S3 + CloudFront
  - AWS CloudFront and S3 together handle multi-Tbps without issue (AWS scale)
```

### Problem 2: Encoding Queue Time (New Content Release)

```
Problem: When Netflix releases a new season (200 hours of raw video),
         encoding must complete before the release date

Solution: Massively parallel encoding on AWS Batch
  - Spin up 10,000+ EC2 instances simultaneously
  - Each instance handles one encoding job (one resolution/codec combination)
  - 200 hours × 120 variants = 24,000 encoding jobs
  - On 10,000 instances: ~2.4 jobs per instance = done in ~4 hours
  - Cost: EC2 spot instances ~$0.10/hr × 10,000 = $1,000/hour × 4 hours = $4,000 total
```

### Problem 3: Resume Across Devices

```
User watches 40 minutes of Stranger Things on phone, switches to TV.
TV must resume at exactly 40 minutes.

Solution:
  - Client POSTs watch progress every 30 seconds to Progress Service
  - Progress Service writes to Cassandra (fast writes, user_id + episode_id = primary key)
  - On TV startup, client fetches watch progress
  - No conflict: last write wins (the most recent progress update is correct)
```

### Problem 4: Microservices Resilience at 250M Users

```
At this scale, any service will have failures every minute.
"Design for failure" — Netflix's core principle.

Hystrix (now deprecated) / Resilience4j:
  - Every service call wrapped in circuit breaker
  - Fallback: if recommendation service fails → show popular titles
  - Fallback: if subtitle service fails → show video without subtitles
  - Fallback: if progress service fails → start from beginning, sync later

Chaos Engineering (Netflix's innovation):
  - Netflix Chaos Monkey randomly kills production instances
  - Forces teams to build genuinely resilient services
  - "If it doesn't fail gracefully in testing, it will fail badly in production"
```

---

## 💻 Code Examples

### Playback Service — Generate Manifest URL

```java
@RestController
@RequestMapping("/v1/titles")
public class PlaybackController {

    @Autowired private ContentMetadataService contentService;
    @Autowired private CdnRoutingService cdnRoutingService;
    @Autowired private DrmService drmService;
    @Autowired private ProgressService progressService;

    @GetMapping("/{titleId}/playback-manifest")
    public ResponseEntity<PlaybackManifest> getPlaybackManifest(
            @PathVariable Long titleId,
            @RequestParam String deviceProfile,
            @RequestHeader("X-User-Id") Long userId,
            HttpServletRequest request) {

        // 1. Get content metadata (what assets are available)
        ContentInfo content = contentService.getContent(titleId);
        if (!content.isAvailable()) {
            return ResponseEntity.status(HttpStatus.NOT_FOUND).build();
        }

        // 2. Select appropriate CDN edge node based on user's IP
        String clientIp = request.getHeader("X-Forwarded-For");
        String cdnBaseUrl = cdnRoutingService.selectEdgeNode(clientIp);

        // 3. Get available video assets matching device profile
        List<VideoAsset> assets = contentService.getAssetsForProfile(titleId, deviceProfile);

        // 4. Generate DRM license URL if content is protected
        String licenseUrl = drmService.generateLicenseUrl(titleId, userId, deviceProfile);

        // 5. Build manifest (points to CDN URLs, not S3 directly)
        PlaybackManifest manifest = PlaybackManifest.builder()
                .manifestUrl(cdnBaseUrl + "/manifests/" + titleId + "/master.m3u8")
                .licenseUrl(licenseUrl)
                .availableQualities(assets.stream().map(VideoAsset::getResolution).collect(toList()))
                .subtitleTracks(contentService.getSubtitleTracks(titleId))
                .audioTracks(contentService.getAudioTracks(titleId))
                .build();

        // 6. Async: log playback start for recommendations + analytics
        progressService.logPlaybackStart(userId, titleId);

        return ResponseEntity.ok(manifest);
    }
}
```

### Encoding Coordinator — Dispatch Jobs

```java
@Service
public class EncodingCoordinator {

    @Autowired private AWSBatchClient batchClient;
    @Autowired private S3Client s3Client;
    @Autowired private VideoAssetRepository assetRepository;
    @Autowired private KafkaTemplate<String, EncodingEvent> kafka;

    public void dispatchEncodingJobs(Long episodeId, String rawS3Key) {
        List<EncodingSpec> specs = buildEncodingSpecs();  // 120+ combinations

        // Submit all encoding jobs to AWS Batch simultaneously
        List<CompletableFuture<Void>> futures = specs.stream()
                .map(spec -> CompletableFuture.runAsync(() -> {
                    submitBatchJob(episodeId, rawS3Key, spec);
                }))
                .collect(Collectors.toList());

        // Track completion
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                .thenRun(() -> {
                    generateMasterManifest(episodeId);
                    kafka.send("encoding.complete",
                            new EncodingEvent(episodeId, "COMPLETE"));
                });
    }

    private List<EncodingSpec> buildEncodingSpecs() {
        List<EncodingSpec> specs = new ArrayList<>();
        String[] resolutions = {"4K", "1080p", "720p", "480p", "360p"};
        String[] codecs = {"h264", "h265", "av1"};
        int[] bitrates = {8000, 5000, 3000, 1500, 800};  // kbps

        for (String res : resolutions) {
            for (String codec : codecs) {
                if (isCompatibleCombo(res, codec)) {
                    specs.add(new EncodingSpec(res, codec, getBitrateFor(res, codec)));
                }
            }
        }
        return specs;
    }
}
```

---

## 🏢 Industry Examples

| Company | CDN Strategy | Encoding Approach |
|---|---|---|
| **Netflix** | Open Connect (self-hosted in ISPs) | Per-shot optimization, AV1, 120+ variants |
| **YouTube** | Google's CDN (own fiber network) | VP9/AV1, transcoding on GCP, LIVE encoding in < 30s |
| **Twitch** | AWS CloudFront + own PoPs | Real-time H.264 transcoding (< 5s latency) |
| **Disney+** | AWS CloudFront | Similar to Netflix pipeline, uses Elemental MediaConvert |
| **Spotify (audio)** | Own CDN, 96kbps/320kbps Ogg Vorbis | Pre-encoded, simpler pipeline than video |

**Netflix public engineering facts**:
- Switched from AWS CDN to Open Connect in 2012, saved ~60% on CDN costs
- AV1 adoption (2020): 20% lower bandwidth vs H.265 on same quality
- Chaos Engineering (Chaos Monkey) invented by Netflix, now industry standard
- EVS (Encoding Video Service) processes 1+ million encoding jobs/day

---

## ⚠️ Common Pitfalls

1. **Designing video storage as a single S3 bucket** — At 90 PB, you need to think about multi-region replication, lifecycle policies (archive cold content to Glacier), and origin failover. Clarify this in the interview.

2. **Ignoring ABR** — Saying "just stream the video file" is wrong. Video is always chunked and streamed adaptively. Not knowing this signals you haven't thought about real streaming systems.

3. **Using a single encoding service** — "An encoding service transcodes the video" is too vague. Explain parallelism (120+ jobs simultaneously), the compute farm, and time-to-encode expectations.

4. **Not mentioning DRM** — Premium content must be DRM-protected. Widevine (Chrome, Android), FairPlay (Apple), PlayReady (Microsoft). The player must acquire a license before decrypting video — this is a separate service in the architecture.

5. **Forgetting the manifest/playlist** — The client doesn't directly download video. It first downloads an HLS/DASH manifest (.m3u8 or .mpd file) that lists all available quality levels and segment URLs. This manifest is the backbone of adaptive streaming.

---

## 🧩 Mini Challenge

**Design question**: Netflix is launching a "Netflix Live" feature for award shows (like the Oscars). How does this differ from on-demand streaming? What new components are needed?

<details>
<summary>💡 Click to reveal answer</summary>

**Key differences from on-demand**:

1. **No pre-encoding** — The video doesn't exist yet; it's being captured in real-time. You need live transcoding (not batch encoding). Software: AWS Elemental Live, FFmpeg, or NVIDIA GPU-based encoders.

2. **Ultra-low latency requirements** — HLS typically has 20-30 second latency. For live events, you want < 5 seconds. Use **LL-HLS (Low-Latency HLS)** with 200ms segments instead of 4-second segments.

3. **No pre-warming CDN** — Content doesn't exist to pre-warm. Open Connect can't proactively cache it. You fall back to cloud CDN (CloudFront) for live, accepting higher latency and cost.

4. **New components needed**:
   - **Live ingest service**: receives live video stream from broadcast truck via RTMP/SRT protocol
   - **Live transcoder farm**: real-time H.264 encoding in multiple bitrates (latency budget = 1-2 seconds)
   - **Live packager**: converts transcoded stream into LL-HLS segments in real-time
   - **Live origin server**: serves HLS segments to CDN; segments are 200ms and generated continuously
   - **DVR buffer**: stores last 4 hours of segments in S3 (allows "rewind" during live event)

5. **The tradeoff**: Live LL-HLS with 200ms segments = 200x more CDN requests than 40-second VOD segments. At 30M concurrent viewers × 1 request/200ms = 150M CDN requests/second — requires massive CDN capacity reserved in advance.

</details>

---

## 📝 Interview Q&A

**Q: How does Netflix handle DRM (Digital Rights Management)?**
> A: Netflix uses multi-DRM: Widevine for Android/Chrome, FairPlay for Apple devices, PlayReady for Microsoft. Each player downloads an encrypted manifest; to play, it must request a DRM license from Netflix's license server, which verifies subscription status and returns a decryption key. The key never leaves the secure enclave on the device. Netflix calls their DRM system "MSL" (Message Security Layer).

**Q: How does Netflix decide what content to pre-warm on Open Connect appliances?**
> A: Netflix's ML-based "Proactive Caching" system predicts what users will watch in each geographic region using: time of day, day of week, current trending content, geographic preferences, and upcoming content releases. The algorithm runs nightly and pushes predicted popular content to nearby appliances before the viewing window. 90%+ hit rate means most streams never touch origin S3.

**Q: What happens if a user's internet drops mid-stream?**
> A: The ABR player maintains a forward buffer (typically 30-60 seconds of pre-downloaded segments). A brief connection drop (< 30s) is invisible to the user. If the connection drops longer, playback pauses with a "Buffering" spinner. When connection restores, the client immediately downloads the next segment at the lowest quality level to minimize stall time, then gradually upgrades quality as the buffer refills.

**Q: How does Netflix implement "continue watching" across devices?**
> A: Every 30 seconds while watching, the Netflix client sends a progress heartbeat: `POST /playback/progress { titleId, episodeId, position: 1847, deviceId }`. This is written to Cassandra with `user_id + episode_id` as the primary key — so the latest write always wins. When you open Netflix on a new device, it fetches your progress and starts from that position. The 30-second granularity means maximum "lost progress" is 30 seconds.

**Q: Why does Netflix use Cassandra instead of MySQL for watch history?**
> A: Watch history is an append-heavy workload: 100M users × ~10 progress updates/hour = 1 billion writes/hour. MySQL would struggle with this write throughput and require complex sharding. Cassandra is designed for massive write throughput with linear horizontal scaling. The access pattern (write often, read by user_id) maps perfectly to Cassandra's partition key design.

---

## 🔗 What to Read Next

1. **[Design Uber](./DesignUber.md)** — Apply geospatial indexing and real-time matching for another "hard" case study
2. **[BuildingBlocks/CDN.md](../BuildingBlocks/CDN.md)** — Deep dive into CDN concepts powering Open Connect
3. **[Database/Sharding.md](../Database/Sharding.md)** — Understand how Cassandra horizontally scales for watch history

---

*[← Design WhatsApp](./DesignWhatsApp.md) | [Back to Case Studies](./README.md) | [Next: Design Uber →](./DesignUber.md)*
