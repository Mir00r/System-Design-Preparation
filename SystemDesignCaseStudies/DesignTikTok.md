# 🎬 Design TikTok (Short Video Platform)

> *"TikTok cracked the code that eluded every social platform before it: how to make complete strangers go viral. Unlike Instagram (follow-based) or YouTube (search-based), TikTok's 'For You' feed is powered by a recommendation engine so good that users watch 90 minutes per day on average. Behind the addictive scroll is a system that ingests millions of video uploads, transcodes them into multiple resolutions, serves them from a global CDN, and trains ML models on billions of user interactions — all in real-time."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [CDN](../BuildingBlocks/CDN.md), [Message Queues](../BuildingBlocks/MessageQueues.md), [Blob Storage](../BuildingBlocks/Blob_Storage.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Video Upload & Processing Pipeline](#-video-upload--processing-pipeline)
4. [Recommendation Engine (For You Feed)](#-recommendation-engine-for-you-feed)
5. [Video Serving & Streaming](#-video-serving--streaming)
6. [Engagement & Interaction](#-engagement--interaction)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Upload short videos (15s - 10min!)
  • "For You" feed — personalized recommendations!
  • Follow users, see their content in "Following" tab
  • Like, comment, share, save videos
  • Video effects, filters, music overlay (client-side!)
  • Duets/Stitches (interact with other videos!)
  • Hashtag challenges and trending page
  • Creator analytics dashboard
  • Live streaming!

NON-FUNCTIONAL:
  • 1B daily active users
  • 10M video uploads per day
  • Feed latency: < 200ms (instant scroll!)
  • Video start time: < 500ms (no buffering!)
  • Upload to viewable: < 5 minutes
  • 99.9% availability (users are VERY intolerant of downtime!)
  • Global reach (serve from nearest edge!)

SCALE:
  • 500 PB total video storage!
  • 5M concurrent video streams!
  • 100B+ user interactions per day (views, likes, watch time!)
  • Feed refresh: 1M requests/second globally!
```

---

## 🏗️ High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                           TIKTOK ARCHITECTURE                               │
│                                                                             │
│  ┌──────────┐         ┌──────────────────────────────────────────┐        │
│  │  Mobile  │────────►│            GLOBAL CDN                     │        │
│  │   App    │◄────────│  (video serving from nearest PoP!)        │        │
│  └──────────┘         └──────────────────────────────────────────┘        │
│       │ API calls                                                          │
│       ▼                                                                    │
│  ┌──────────────┐                                                          │
│  │  API Gateway │  (rate limiting, auth, routing!)                         │
│  └──────┬───────┘                                                          │
│         │                                                                   │
│    ┌────┴─────────┬──────────────┬──────────────┬──────────────┐          │
│    ▼              ▼              ▼              ▼              ▼           │
│ ┌────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│ │ Upload │  │   Feed   │  │  User    │  │Engagement│  │  Search  │     │
│ │Service │  │  Service │  │ Service  │  │ Service  │  │ Service  │     │
│ └───┬────┘  └────┬─────┘  └──────────┘  └──────────┘  └──────────┘     │
│     │             │                                                        │
│     ▼             ▼                                                        │
│ ┌────────────┐  ┌──────────────────┐                                      │
│ │ Video      │  │  Recommendation  │  ← THE SECRET SAUCE! 🧪             │
│ │ Processing │  │     Engine       │                                      │
│ │ Pipeline   │  │  (ML models!)    │                                      │
│ └────────────┘  └──────────────────┘                                      │
│                                                                            │
│  DATA STORES:                                                              │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │  S3/Blob│  │PostgreSQL│  │  Redis  │  │  Kafka  │  │ Feature │      │
│  │ (videos)│  │(metadata)│  │ (cache) │  │(events) │  │  Store  │      │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘      │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## 📤 Video Upload & Processing Pipeline

```
UPLOAD FLOW (10M videos/day = ~115 uploads/second!):

  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 1: UPLOAD                                                 │
  │  Client → Upload Service → S3 (raw video!)                     │
  │  • Chunked upload (resumable if network drops!)                 │
  │  • Pre-signed URL: client uploads directly to S3!               │
  │    (bypasses your servers — saves bandwidth!)                    │
  │  • Metadata stored in PostgreSQL                                │
  └─────────────────────────────────────┬──────────────────────────┘
                                        │ S3 event notification!
                                        ▼
  ┌────────────────────────────────────────────────────────────────┐
  │  STEP 2: PROCESSING PIPELINE (async via message queue!)         │
  │                                                                 │
  │  ┌─────────────┐                                                │
  │  │ Validation  │ → Check format, duration, file size            │
  │  │             │ → Virus scan! (malware in video files!)        │
  │  └──────┬──────┘                                                │
  │         ▼                                                       │
  │  ┌─────────────┐                                                │
  │  │ Transcoding │ → Multiple resolutions: 360p, 480p, 720p, 1080p│
  │  │ (FFmpeg!)   │ → Multiple bitrates (adaptive streaming!)      │
  │  │             │ → Generate HLS/DASH segments!                  │
  │  │             │ → Extract thumbnail at key frames!             │
  │  └──────┬──────┘                                                │
  │         ▼                                                       │
  │  ┌─────────────┐                                                │
  │  │ Content     │ → Nudity/violence detection (ML model!)       │
  │  │ Moderation  │ → Copyright check (audio fingerprint!)         │
  │  │             │ → Hate speech detection (NLP on captions!)     │
  │  └──────┬──────┘                                                │
  │         ▼                                                       │
  │  ┌─────────────┐                                                │
  │  │ ML Feature  │ → Extract visual features (CNN!)               │
  │  │ Extraction  │ → Identify objects, scenes, faces              │
  │  │             │ → Audio classification (music genre!)          │
  │  │             │ → Generate embedding vector for recommendation!│
  │  └──────┬──────┘                                                │
  │         ▼                                                       │
  │  ┌─────────────┐                                                │
  │  │ CDN Push    │ → Push to edge locations worldwide!            │
  │  │             │ → Pre-warm popular creator's content!          │
  │  └─────────────┘                                                │
  └────────────────────────────────────────────────────────────────┘

TRANSCODING AT SCALE:
  10M videos/day × 4 resolutions × 3 min avg = 120M encoding minutes/day!
  
  Solution: DISTRIBUTED ENCODING FARM
  • Split video into chunks (GOP-aligned segments!)
  • Encode chunks in parallel across worker fleet!
  • Merge encoded chunks back!
  • 1 video transcoding: 10 seconds (parallel!) vs 5 minutes (sequential!)
```

---

## 🧠 Recommendation Engine (For You Feed)

```
THE "FOR YOU" ALGORITHM (TikTok's moat!):

SIGNALS (what the model uses):
  ┌─────────────────────────────────────────────────────────────────┐
  │  STRONG SIGNALS (high weight!):                                  │
  │  • Watch time percentage (watched 95% vs 10%! HUGE signal!)     │
  │  • Rewatches (watched same video 3x = LOVE it!)                 │
  │  • Shares (strongest signal! "I want others to see this!")      │
  │  • Comments (engagement = interest!)                             │
  │                                                                  │
  │  MEDIUM SIGNALS:                                                 │
  │  • Likes                                                         │
  │  • Follow after watching                                         │
  │  • Profile visits after video                                    │
  │  • Video completion (didn't swipe away!)                         │
  │                                                                  │
  │  WEAK SIGNALS:                                                   │
  │  • Device type, language, location                               │
  │  • Time of day                                                   │
  │  • Video metadata (hashtags, sounds, captions)                   │
  │                                                                  │
  │  NEGATIVE SIGNALS:                                               │
  │  • "Not interested" button (STRONG negative!)                   │
  │  • Swipe away quickly (< 2 seconds = bad match!)                │
  │  • Hide/report                                                   │
  └─────────────────────────────────────────────────────────────────┘

ARCHITECTURE:
  ┌──────────────────────────────────────────────────────────────────┐
  │                                                                   │
  │  User opens app → Feed Service → Recommendation Engine:          │
  │                                                                   │
  │  1. RECALL (find candidates! — 1M videos → 10K candidates!)     │
  │     • Collaborative filtering (users like you watched X!)        │
  │     • Content-based (you watched cooking → more cooking!)        │
  │     • Trending (popular in your region!)                          │
  │     • Social graph (friends liked X!)                             │
  │                                                                   │
  │  2. RANKING (score candidates! — 10K → top 100!)                │
  │     • Deep neural network predicts:                              │
  │       P(watch) × watch_time_weight +                             │
  │       P(like) × like_weight +                                    │
  │       P(share) × share_weight +                                  │
  │       P(comment) × comment_weight                                │
  │     • Personalized per user (user embedding!)                    │
  │                                                                   │
  │  3. DIVERSIFICATION (avoid filter bubble!)                       │
  │     • Mix in exploration (10-20% videos outside your pattern!)   │
  │     • Ensure variety (not 100 cooking videos in a row!)          │
  │     • Spread creators (don't show same creator 3x in a row!)    │
  │     • Balance content types (funny, educational, music...)       │
  │                                                                   │
  │  4. SERVE (return top 50 videos with prefetch URLs!)             │
  │     • Client prefetches next 3 videos while watching current!    │
  │     • Infinite scroll: request next batch when 5 videos remain!  │
  └──────────────────────────────────────────────────────────────────┘

COLD START (new user with no history!):
  • First 10 videos: popular/trending in user's country!
  • After 10 videos: model starts learning from watch time!
  • After 50 videos: personalization kicks in noticeably!
  • After 200 videos: highly personalized feed!
  
  TikTok's advantage: short videos = MORE data points per session!
  Instagram: user might post 1 photo per week.
  TikTok: user watches 200 videos in ONE session = 200 data points!

NEW VIDEO COLD START:
  • Show to ~500 random users first (test group!)
  • Measure: completion rate, likes, shares
  • High engagement? → Expand audience to 5K, 50K, 500K...
  • Low engagement? → Stop showing (never goes viral!)
  • This gives EVERY video a chance (not just popular creators!)
```

---

## 📡 Video Serving & Streaming

```
VIDEO DELIVERY (5M concurrent streams!):

ADAPTIVE BITRATE STREAMING (ABR):
  ┌────────────────────────────────────────────────────────────────┐
  │  Video stored as multiple quality levels:                       │
  │  • 360p @ 400 kbps  (3G / poor wifi)                          │
  │  • 480p @ 800 kbps  (decent mobile)                           │
  │  • 720p @ 1.5 Mbps  (good wifi)                               │
  │  • 1080p @ 3 Mbps   (home wifi/5G)                            │
  │                                                                 │
  │  Client measures bandwidth in real-time!                        │
  │  Bandwidth drops? → Switch to lower quality mid-video!         │
  │  No rebuffering! (smooth degradation!)                          │
  │                                                                 │
  │  Protocol: HLS (HTTP Live Streaming)                            │
  │  • Video split into 2-second segments                           │
  │  • Client downloads segment-by-segment                          │
  │  • Can switch quality at each segment boundary!                 │
  └────────────────────────────────────────────────────────────────┘

CDN STRATEGY (global, multi-layer!):
  ┌────────────────────────────────────────────────────────────────┐
  │                                                                 │
  │  User (Mumbai) → CDN PoP (Mumbai) → CDN PoP (Singapore)       │
  │                        → Origin (US)                            │
  │                                                                 │
  │  Cache hit rate target: 95%+ (short videos = easy to cache!)   │
  │                                                                 │
  │  Smart pre-caching:                                             │
  │  • Trending videos → push to ALL edge locations!               │
  │  • New video from India creator → push to India edges!         │
  │  • Viral detection: once video hits 10K views → global push!   │
  │                                                                 │
  │  Short videos advantage:                                        │
  │  • 30 second video @ 720p = ~5 MB                              │
  │  • Entire video fits in one HTTP request!                       │
  │  • No complex streaming protocol needed for most!              │
  │  • Can pre-download next 3 videos in background!               │
  └────────────────────────────────────────────────────────────────┘

PREFETCHING (why TikTok feels so SMOOTH):
  While watching video #1:
  • Download video #2 (100% certain user will see it — next in feed!)
  • Download video #3 (high probability)
  • Pre-fetch thumbnail of video #4-#10
  
  Result: ZERO loading time between videos! Instant swipe! 🚀
```

---

## 💻 Java Implementation

### Feed Service

```java
@Service
public class FeedService {
    
    @Autowired private RecommendationEngine recommender;
    @Autowired private RedisTemplate<String, String> redis;
    @Autowired private VideoMetadataService videoMetadata;
    @Autowired private CdnUrlService cdnService;
    
    /**
     * Generate personalized "For You" feed.
     * Returns video IDs with pre-signed CDN URLs!
     */
    public FeedResponse getForYouFeed(String userId, String cursor, int count) {
        // Check cached feed first (avoid re-computing!)
        String feedCacheKey = "feed:foryou:" + userId;
        List<String> cachedVideoIds = redis.opsForList()
            .range(feedCacheKey, 0, -1);
        
        if (cachedVideoIds == null || cachedVideoIds.isEmpty()) {
            // Generate fresh recommendations!
            List<String> videoIds = recommender.recommend(userId, 200);
            
            // Filter out already-seen videos
            Set<String> seen = getRecentlySeenVideos(userId);
            videoIds = videoIds.stream()
                .filter(id -> !seen.contains(id))
                .collect(Collectors.toList());
            
            // Cache for 30 minutes (refresh on next open!)
            redis.opsForList().rightPushAll(feedCacheKey, videoIds);
            redis.expire(feedCacheKey, Duration.ofMinutes(30));
            
            cachedVideoIds = videoIds;
        }
        
        // Paginate from cursor
        int startIdx = cursor == null ? 0 : Integer.parseInt(cursor);
        int endIdx = Math.min(startIdx + count, cachedVideoIds.size());
        List<String> pageVideoIds = cachedVideoIds.subList(startIdx, endIdx);
        
        // Enrich with metadata + CDN URLs
        List<FeedItem> items = pageVideoIds.stream()
            .map(videoId -> {
                VideoMetadata meta = videoMetadata.get(videoId);
                String cdnUrl = cdnService.getStreamUrl(videoId, 
                    meta.getUserRegion());
                return FeedItem.builder()
                    .videoId(videoId)
                    .creatorId(meta.getCreatorId())
                    .creatorName(meta.getCreatorName())
                    .description(meta.getDescription())
                    .musicInfo(meta.getMusicInfo())
                    .likeCount(meta.getLikeCount())
                    .commentCount(meta.getCommentCount())
                    .shareCount(meta.getShareCount())
                    .streamUrl(cdnUrl)
                    .thumbnailUrl(meta.getThumbnailUrl())
                    .duration(meta.getDuration())
                    .build();
            })
            .collect(Collectors.toList());
        
        // Prefetch next batch (warm CDN cache!)
        prefetchNextBatch(cachedVideoIds, endIdx);
        
        String nextCursor = endIdx < cachedVideoIds.size() 
            ? String.valueOf(endIdx) : null;
        
        return new FeedResponse(items, nextCursor);
    }
    
    /**
     * Record watch event (CRITICAL for recommendation model!).
     */
    public void recordWatchEvent(String userId, String videoId, 
                                  WatchEvent event) {
        // This is the MOST important signal!
        double watchPercentage = (double) event.getWatchDuration() 
            / event.getVideoDuration();
        
        UserInteraction interaction = UserInteraction.builder()
            .userId(userId)
            .videoId(videoId)
            .watchPercentage(watchPercentage)
            .rewatched(event.isRewatched())
            .liked(event.isLiked())
            .shared(event.isShared())
            .commented(event.isCommented())
            .timestamp(Instant.now())
            .build();
        
        // Send to Kafka for ML model training + feature store update!
        kafka.send("user-interactions", userId, interaction);
        
        // Update "seen" set (don't recommend same video again!)
        redis.opsForSet().add("seen:" + userId, videoId);
        redis.expire("seen:" + userId, Duration.ofDays(30));
    }
}
```

### Video Upload Service

```java
@Service
public class VideoUploadService {
    
    @Autowired private S3Client s3;
    @Autowired private KafkaTemplate<String, VideoProcessingJob> kafka;
    @Autowired private VideoRepository videoRepo;
    
    /**
     * Initiate video upload — return pre-signed URL for direct S3 upload!
     */
    public UploadInitResponse initiateUpload(String userId, 
                                              VideoUploadRequest request) {
        // Validate
        if (request.getDurationSeconds() > 600) {
            throw new ValidationException("Max video duration: 10 minutes!");
        }
        
        String videoId = UUID.randomUUID().toString();
        String s3Key = "raw/" + userId + "/" + videoId + ".mp4";
        
        // Generate pre-signed URL (client uploads directly to S3!)
        String presignedUrl = s3.presignPutObject(PutObjectPresignRequest.builder()
            .signatureDuration(Duration.ofMinutes(30))
            .putObjectRequest(PutObjectRequest.builder()
                .bucket("tiktok-raw-videos")
                .key(s3Key)
                .contentType("video/mp4")
                .build())
            .build())
            .url().toString();
        
        // Save metadata
        Video video = Video.builder()
            .id(videoId)
            .creatorId(userId)
            .status(VideoStatus.UPLOADING)
            .description(request.getDescription())
            .hashtags(request.getHashtags())
            .musicId(request.getMusicId())
            .rawS3Key(s3Key)
            .createdAt(Instant.now())
            .build();
        videoRepo.save(video);
        
        return new UploadInitResponse(videoId, presignedUrl);
    }
    
    /**
     * Called by S3 event notification when upload completes!
     * Triggers processing pipeline.
     */
    @SqsListener("video-upload-complete")
    public void onUploadComplete(S3Event event) {
        String s3Key = event.getRecords().get(0).getS3().getObject().getKey();
        String videoId = extractVideoId(s3Key);
        
        // Update status
        videoRepo.updateStatus(videoId, VideoStatus.PROCESSING);
        
        // Trigger processing pipeline!
        kafka.send("video-processing", videoId, VideoProcessingJob.builder()
            .videoId(videoId)
            .rawS3Key(s3Key)
            .steps(List.of(
                ProcessingStep.VALIDATE,
                ProcessingStep.TRANSCODE,
                ProcessingStep.MODERATE,
                ProcessingStep.EXTRACT_FEATURES,
                ProcessingStep.CDN_PUSH))
            .build());
    }
}
```

---

## ❓ Interview Q&A

**Q1: How does TikTok's recommendation differ from YouTube's?**
> Key difference: TikTok optimizes for DISCOVERY (showing content from strangers!), YouTube optimizes for ENGAGEMENT (showing familiar content!). TikTok's "For You" page: 90% videos from accounts you DON'T follow. YouTube Home: mostly from subscriptions + similar creators. Technical implication: TikTok needs much stronger content-based signals (what's IN the video) vs YouTube's heavy reliance on social graph (who you follow, what they watch). TikTok's short format also means more data points per session (200 videos viewed vs 5 on YouTube), enabling faster model convergence.

**Q2: How would you handle a video going viral (0 to 10M views in 1 hour)?**
> Multi-layer CDN approach: (1) Video starts on single origin, (2) As views increase: automatic tier promotion — at 1K views: push to regional edges, at 100K: push to ALL global PoPs, at 1M: add to "mega-cache" (in-memory at edge!). Traffic management: (3) CDN can handle spikes natively (that's their purpose!), (4) Origin protection: shield layer between CDN and origin (collapse 1M requests into 1 origin fetch!), (5) Rate limit non-cached paths. Metadata: video metadata in Redis (handle read spikes without DB pressure). Counter updates: Redis INCRBY (not DB for each view!), batch flush every 5 seconds.

**Q3: How do you solve the cold-start problem for new users?**
> Three phases: (1) **Zero data**: show trending + popular in user's country/language (use registration signals: location, language, age if provided), (2) **First session (0-50 videos)**: A/B test different content categories, observe watch time per category — within 10 videos, model knows if you prefer comedy vs dance vs cooking, (3) **Early model (50-200 videos)**: fine-tune user embedding based on interactions so far. The secret: TikTok's short format means 50 videos = 15 minutes of watching, giving the model enough signal in ONE session! YouTube needs days/weeks for the same signal density.

**Q4: How do you ensure video upload doesn't fail on poor mobile networks?**
> Resumable chunked uploads: (1) Split video into 1MB chunks, (2) Upload each chunk separately with chunk_id, (3) If network drops: resume from last successful chunk (no re-upload!), (4) Server reassembles chunks after all received. Implementation: use TUS protocol (open standard for resumable uploads!) or multipart S3 upload with pre-signed URLs per part. Client-side: compress video before upload (client-side encoding at lower quality → smaller file → faster upload!). Queue upload: if fully offline → queue locally, upload when connection restored!

---

## 🔗 Related Topics
- [CDN Design](./DesignCDN.md) — Video delivery at scale
- [Design Spotify](./DesignSpotify.md) — Similar content streaming
- [Blob Storage](../BuildingBlocks/Blob_Storage.md) — Video storage
- [Push vs Pull](../Tradeoffs/Push_vs_Pull_Architecture.md) — Feed architecture

---

*"TikTok's engineering brilliance is in making complex ML feel effortless to users. The recommendation engine, the video processing pipeline, the global CDN — it all happens invisibly. Users just think 'this app knows me.' That's the highest compliment to a system's design: when users don't even realize there IS a system."* 🎬
