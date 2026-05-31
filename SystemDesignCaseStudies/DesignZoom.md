# 🎥 Design Zoom / Video Conferencing System

> *"Zoom handles 300 MILLION daily meeting participants. Each video stream is 1-4 Mbps. A 10-person meeting means routing 10 video + 10 audio streams simultaneously with < 150ms latency (humans notice lag above 150ms!). Add screen sharing, recording, chat, breakout rooms, virtual backgrounds, and you have one of the most complex real-time systems ever built."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [WebSockets](../../APIs/WebSockets.md), [WebRTC](../../APIs/WebRTC.md), [CDN](../../BuildingBlocks/CDN.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Challenges of Real-Time Video](#-challenges-of-real-time-video)
3. [High-Level Architecture](#-high-level-architecture)
4. [Signaling Service](#-signaling-service)
5. [Media Routing: SFU vs MCU](#-media-routing-sfu-vs-mcu)
6. [Adaptive Bitrate & Quality](#-adaptive-bitrate--quality)
7. [Recording Service](#-recording-service)
8. [Screen Sharing & Chat](#-screen-sharing--chat)
9. [Scalability & Global Deployment](#-scalability--global-deployment)
10. [Java Implementation](#-java-implementation)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ 1-on-1 video calls
✅ Group calls (up to 100 video, 1000 audio-only)
✅ Screen sharing
✅ Text chat during meetings
✅ Meeting recording (cloud storage)
✅ Virtual backgrounds
✅ Breakout rooms
✅ Waiting room & host controls
✅ Join via link (no account required for guests)
```

### Non-Functional Requirements
```
✅ Ultra-low latency: < 150ms end-to-end (real-time feel!)
✅ Adaptive quality (adjust to network conditions)
✅ High availability: 99.99% (meetings can't "just stop")
✅ Encrypt all media (E2E encryption option)
✅ Scale: support 300M+ participants/day
✅ Cross-platform: web, desktop, mobile
✅ Bandwidth efficiency (not everyone has fast internet!)
```

### Scale Estimation
```
Daily participants: 300 million
Average meeting duration: 40 minutes
Concurrent meetings (peak): 10 million
Average participants per meeting: 5

Bandwidth per video stream: 1.5 Mbps (720p)
Per meeting (5 people): 5 × 1.5 = 7.5 Mbps upload
                        Each receives 4 × 1.5 = 6 Mbps download

Total bandwidth at peak: millions of Gbps! 
→ This is why media servers are distributed globally!
```

---

## ⚡ Challenges of Real-Time Video

```
WHY IS VIDEO CONFERENCING SO HARD?

  1. LATENCY BUDGET (only 150ms total!):
     ┌──────────────────────────────────────────────────────┐
     │  Camera capture:           10ms                       │
     │  Encoding (H.264/VP8):     15ms                       │
     │  Network transmission:      50-100ms                  │
     │  Server processing:         5-10ms                    │
     │  Decoding:                  15ms                       │
     │  Rendering:                 5ms                        │
     ├──────────────────────────────────────────────────────┤
     │  TOTAL BUDGET:             100-155ms ← barely fits!   │
     └──────────────────────────────────────────────────────┘
     
  2. PACKET LOSS: Internet isn't reliable!
     • Lost video frame → glitch/freeze
     • Lost audio packet → gap in sound (MORE annoying!)
     • Solution: Forward Error Correction (FEC) + jitter buffer
     
  3. BANDWIDTH VARIABILITY:
     • User's bandwidth changes every second (WiFi!)
     • Must adapt quality in real-time
     • Simulcast: send multiple quality levels
     
  4. N² PROBLEM:
     • 10 participants = each sends 1 stream, receives 9!
     • 100 participants peer-to-peer = impossible!
     • Need server-based media routing
     
  5. CLOCK SYNCHRONIZATION:
     • Audio must sync with video (lip sync!)
     • Multiple participants must be in sync
     • RTCP timestamps for synchronization
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      PARTICIPANTS                                │
│  👤 Camera + Mic → Encode → Send                               │
│  👤 Receive → Decode → Display                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │ (WebRTC / Custom UDP)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MEDIA LAYER                                    │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐      │
│  │ SFU Server 1 │  │ SFU Server 2 │  │ SFU Server N     │      │
│  │ (Region: US) │  │ (Region: EU) │  │ (Region: Asia)   │      │
│  └──────────────┘  └──────────────┘  └──────────────────┘      │
│           ↕ (inter-region media relay for global meetings)       │
└─────────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────────┐
│                   CONTROL PLANE                                   │
│                                                                  │
│  ┌────────────┐ ┌────────────┐ ┌──────────┐ ┌──────────────┐   │
│  │ Signaling  │ │ Meeting    │ │ Auth     │ │ Recording    │   │
│  │ Service    │ │ Service    │ │ Service  │ │ Service      │   │
│  └────────────┘ └────────────┘ └──────────┘ └──────────────┘   │
│  ┌────────────┐ ┌────────────┐ ┌──────────┐                    │
│  │ Chat       │ │ Presence   │ │ Analytics│                    │
│  │ Service    │ │ Service    │ │ Service  │                    │
│  └────────────┘ └────────────┘ └──────────┘                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📡 Signaling Service

```
SIGNALING: How participants discover each other and negotiate media

  PROTOCOL: WebSocket connection (persistent, bidirectional)
  
  JOINING A MEETING:
  
  1. User clicks meeting link: https://zoom.us/j/123456789
  2. Client → Signaling Server: "I want to join meeting 123456789"
  3. Server authenticates, checks meeting exists
  4. Server assigns user to an SFU media server (nearest region!)
  5. Server sends SFU address + ICE candidates to client
  6. Client establishes WebRTC connection to SFU
  7. Server notifies other participants: "New person joined!"
  8. Client sends SDP offer (codec negotiation)
  9. SFU responds with SDP answer
  10. Media starts flowing!

  ┌────────────┐         ┌────────────┐         ┌────────────┐
  │  Client A  │         │  Signaling │         │  Client B  │
  └─────┬──────┘         └─────┬──────┘         └─────┬──────┘
        │  "Join meeting"      │                       │
        │─────────────────────►│                       │
        │  "Here's your SFU"   │                       │
        │◄─────────────────────│                       │
        │                      │  "A joined meeting!"  │
        │                      │──────────────────────►│
        │        WebRTC media  │                       │
        │══════════════════════╪══════════════════════►│
        │◄═════════════════════╪═══════════════════════│
        
  SDP (Session Description Protocol):
    "I support H.264 and VP8 video, Opus audio, 
     my IP is X, I'm behind NAT, here are my STUN results..."
```

---

## 🎬 Media Routing: SFU vs MCU

```
THREE APPROACHES:

1. PEER-TO-PEER (P2P):
   Each participant sends to every other participant directly.
   
   4 people = each sends 3 streams, receives 3
   Upload: 3 × 1.5 Mbps = 4.5 Mbps per person
   
   ✅ Lowest latency (no server hop!)
   ✅ No server cost!
   ❌ Doesn't scale beyond 4-5 people
   ❌ Bandwidth nightmare for participants
   
   → Used only for 1-on-1 calls!

2. SFU (Selective Forwarding Unit): ← ZOOM'S APPROACH!
   Participants send ONE stream to server.
   Server FORWARDS streams selectively to each participant.
   
   4 people: each uploads 1 stream, downloads 3
   Upload: 1.5 Mbps (just one stream!)
   Download: 3 × quality (server can downscale!)
   
   ┌─────┐     ┌─────────────────┐     ┌─────┐
   │  A  │────►│                 │────►│  B  │
   │     │◄────│    SFU Server   │◄────│     │
   └─────┘     │                 │     └─────┘
   ┌─────┐     │  Routes streams │     ┌─────┐
   │  C  │────►│  No transcoding │────►│  D  │
   │     │◄────│  Very efficient!│◄────│     │
   └─────┘     └─────────────────┘     └─────┘
   
   ✅ Low server CPU (just forwarding packets!)
   ✅ Scales to 50-100 participants
   ✅ Client can choose quality per stream
   ❌ Download bandwidth = (N-1) × stream
   
   → Used by Zoom, Google Meet, Teams for most meetings!

3. MCU (Multipoint Control Unit):
   Server MIXES all streams into ONE composite stream.
   
   4 people: upload 1, download 1 (mixed!)
   
   ✅ Minimal download bandwidth (just 1 stream!)
   ✅ Works for very large meetings
   ❌ MASSIVE server CPU (decode + composite + re-encode!)
   ❌ Higher latency (processing time)
   ❌ All participants see same layout
   
   → Used for webinars, 100+ participant "view-only" meetings

ZOOM'S HYBRID:
  1-4 participants: P2P (direct, lowest latency)
  5-49 participants: SFU (server routes, adaptive)
  50+ participants: SFU with "active speaker" optimization
  Webinar mode: MCU for viewers, SFU for panelists
```

---

## 📊 Adaptive Bitrate & Quality

```
SIMULCAST (Key to adaptive quality!):
  Each participant encodes and sends MULTIPLE quality levels:
  
  High:   720p @ 1.5 Mbps (main video)
  Medium: 360p @ 500 Kbps (gallery view)
  Low:    180p @ 150 Kbps (thumbnail/speaker only)
  
  SFU decides which quality to forward to each viewer:
  
  Viewer watching SPEAKER → gets High quality
  Viewer in GALLERY VIEW → gets Medium for all
  Viewer on BAD NETWORK → gets Low quality
  Participant is MUTED/camera off → gets nothing (saves bandwidth!)

CONGESTION CONTROL:
  Problem: Network bandwidth fluctuates!
  
  Detection methods:
  • RTCP feedback: receiver reports packet loss %
  • Google's GCC (Google Congestion Control): delay-based
  • Transport-wide CC: measures one-way delay trends
  
  Response:
  Packet loss > 5%? → Drop to lower simulcast layer
  Packet loss > 15%? → Audio only! (video paused)
  Bandwidth improving? → Gradually increase quality
  
  PRIORITY: Audio > Screen sharing > Active speaker video > Gallery

JITTER BUFFER:
  Packets arrive at irregular intervals (network jitter!)
  
  Without buffer: choppy audio/video (packets played as they arrive)
  With buffer: collect packets, reorder, play at steady rate
  
  Buffer size trade-off:
  Small (30ms): less latency, more glitches
  Large (200ms): smooth playback, more delay
  Adaptive: start small, grow if jitter detected!
```

---

## 🎙️ Recording Service

```
RECORDING ARCHITECTURE:

  ┌─────────────────────────────────────────────────┐
  │  SFU Server (meeting in progress)              │
  │                                                 │
  │  Streams: [A:video] [B:video] [C:video]        │
  │           [A:audio] [B:audio] [C:audio]        │
  └─────────────────────┬─────────────────────────┘
                        │ (media fork to recorder)
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Recording Worker                                │
  │  1. Receive all media streams                    │
  │  2. Composite video (layout: speaker + gallery)  │
  │  3. Mix audio (all tracks → single track)        │
  │  4. Encode to MP4 (H.264 + AAC)                 │
  │  5. Upload segments to Object Storage            │
  └─────────────────────┬─────────────────────────┘
                        │
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Object Storage (S3/GCS)                         │
  │  meeting_123/segment_001.mp4                    │
  │  meeting_123/segment_002.mp4                    │
  │  ...                                            │
  └─────────────────────────────────────────────────┘
                        │ (meeting ends)
                        ▼
  ┌─────────────────────────────────────────────────┐
  │  Post-Processing Pipeline                        │
  │  • Concatenate segments → final.mp4             │
  │  • Generate transcript (Speech-to-Text)         │
  │  • Create chapters (detect speaker changes)     │
  │  • Generate thumbnail                           │
  │  • Notify host: "Recording ready!"             │
  └─────────────────────────────────────────────────┘

LIVE RECORDING CHALLENGES:
  • Must record IN REAL-TIME (can't wait for meeting to end!)
  • Participant joins/leaves → layout changes mid-recording
  • Screen sharing overlays video → compositing logic
  • Network issues → gaps in stream → fill with last frame
  • 1-hour meeting ≈ 500 MB - 2 GB storage
```

---

## 🖥️ Screen Sharing & Chat

```
SCREEN SHARING:
  Different from camera video!
  
  Characteristics:
  • High resolution (1080p-4K) but low motion
  • Text/code must be SHARP (no blur!)
  • Frame rate can be lower (5-15 fps vs 30 fps for video)
  • Compression: favor quality over framerate
  
  Encoding strategy:
  • Use VP9/AV1 (better for static content)
  • Higher resolution, lower framerate
  • Detect motion regions: only encode changed areas!
  • Full screen capture vs window capture vs browser tab
  
  Bandwidth: 1-3 Mbps (depends on content motion)

IN-MEETING CHAT:
  Simpler subsystem but needs:
  • Real-time delivery (WebSocket alongside media)
  • Message history (persisted for meeting duration)
  • File sharing (upload → CDN → share link)
  • Direct messages vs group chat
  • Reactions/emojis (emoji overlay on video!)
  
  Architecture:
  Client → WebSocket → Chat Service → Redis Pub/Sub → Other clients
                                    → PostgreSQL (persistence)
```

---

## 🌍 Scalability & Global Deployment

```
GLOBAL MEDIA ROUTING:
  Meeting with participants across the world:
  
  Alice (New York) + Bob (London) + Charlie (Tokyo)
  
  Bad approach: Single SFU in US
    Alice → US SFU: 10ms
    Bob → US SFU: 80ms (transatlantic!)
    Charlie → US SFU: 150ms (transpacific!)
    
  Good approach: Regional SFUs with relay!
    Alice → US SFU: 10ms
    Bob → EU SFU: 10ms  
    Charlie → Asia SFU: 10ms
    US SFU ↔ EU SFU ↔ Asia SFU: dedicated backbone
    
  ┌──────────┐         ┌──────────┐         ┌──────────┐
  │  US SFU  │◄═══════►│  EU SFU  │◄═══════►│Asia SFU  │
  │  Alice   │         │   Bob    │         │ Charlie  │
  └──────────┘         └──────────┘         └──────────┘
        Media relay between SFUs on dedicated fiber!
        (Not public internet → consistent low latency!)

SFU SERVER ALLOCATION:
  When meeting starts:
  1. First participant joins → assigned to nearest SFU
  2. Second participant joins from same region → same SFU
  3. Participant from different region → cascade to regional SFU
  4. Meeting grows → may need multiple SFUs in same region!
  
  Load balancing: 
  Each SFU handles ~1000 concurrent streams
  Beyond capacity → spin up new SFU, redistribute

CAPACITY PLANNING:
  Single SFU server:
  • CPU: decode/forward ~1000 streams
  • Bandwidth: 10 Gbps NIC handles ~6000 × 1.5Mbps streams
  • Memory: minimal (just forwarding, not storing!)
  
  Global deployment:
  • 40+ PoPs (Points of Presence) worldwide
  • Auto-scaling: spin up SFUs based on concurrent meetings
  • Health checks: if SFU unhealthy → migrate meeting to backup
```

---

## 💻 Java Implementation

### Meeting Service

```java
@Service
public class MeetingService {
    
    @Autowired private MeetingRepository meetingRepo;
    @Autowired private SFUAllocator sfuAllocator;
    @Autowired private ParticipantRegistry participantRegistry;
    
    public MeetingSession createMeeting(CreateMeetingRequest request) {
        Meeting meeting = Meeting.builder()
            .id(generateMeetingId()) // 9-digit numeric ID
            .hostUserId(request.getHostId())
            .title(request.getTitle())
            .scheduledStart(request.getStartTime())
            .settings(MeetingSettings.builder()
                .waitingRoom(request.isWaitingRoom())
                .maxParticipants(request.getMaxParticipants())
                .recordingEnabled(request.isRecording())
                .e2eEncryption(request.isE2EEncryption())
                .build())
            .status(MeetingStatus.CREATED)
            .build();
        
        meetingRepo.save(meeting);
        
        return MeetingSession.builder()
            .meeting(meeting)
            .joinUrl("https://meet.example.com/j/" + meeting.getId())
            .hostKey(generateHostKey())
            .build();
    }
    
    public JoinResult joinMeeting(String meetingId, String userId, 
                                   GeoLocation userLocation) {
        Meeting meeting = meetingRepo.findById(meetingId)
            .orElseThrow(() -> new MeetingNotFoundException(meetingId));
        
        // Check capacity
        int currentParticipants = participantRegistry.count(meetingId);
        if (currentParticipants >= meeting.getSettings().getMaxParticipants()) {
            throw new MeetingFullException(meetingId);
        }
        
        // Allocate SFU server (nearest to user!)
        SFUServer sfu = sfuAllocator.allocate(meetingId, userLocation);
        
        // Generate ICE candidates for WebRTC connection
        List<IceCandidate> iceCandidates = sfu.getIceCandidates();
        
        // Register participant
        Participant participant = Participant.builder()
            .userId(userId)
            .meetingId(meetingId)
            .sfuServerId(sfu.getId())
            .joinedAt(Instant.now())
            .mediaState(MediaState.AUDIO_VIDEO)
            .build();
        
        participantRegistry.add(participant);
        
        // Notify other participants
        signalingService.broadcast(meetingId, 
            new ParticipantJoinedEvent(userId, participant));
        
        return JoinResult.builder()
            .sfuAddress(sfu.getAddress())
            .iceCandidates(iceCandidates)
            .existingParticipants(participantRegistry.list(meetingId))
            .meetingSettings(meeting.getSettings())
            .build();
    }
}
```

### SFU Allocator (Region-Aware)

```java
@Service
public class SFUAllocator {
    
    @Autowired private SFURegistry sfuRegistry;
    
    /**
     * Allocate best SFU server for a meeting participant.
     * Strategy: closest region with available capacity.
     */
    public SFUServer allocate(String meetingId, GeoLocation userLocation) {
        // 1. Check if meeting already has an SFU in user's region
        String userRegion = resolveRegion(userLocation);
        Optional<SFUServer> existingSfu = sfuRegistry
            .findByMeetingAndRegion(meetingId, userRegion);
        
        if (existingSfu.isPresent() && existingSfu.get().hasCapacity()) {
            return existingSfu.get(); // Same SFU, same region!
        }
        
        // 2. Find least-loaded SFU in user's region
        SFUServer regionSfu = sfuRegistry
            .findLeastLoadedInRegion(userRegion)
            .orElseThrow(() -> new NoSFUAvailableException(userRegion));
        
        // 3. If meeting has SFUs in OTHER regions, set up cascade
        List<SFUServer> meetingSfus = sfuRegistry.findByMeeting(meetingId);
        if (!meetingSfus.isEmpty()) {
            // Create media relay between regions
            SFUServer primarySfu = meetingSfus.get(0);
            setupCascade(primarySfu, regionSfu, meetingId);
        }
        
        // 4. Register this SFU for the meeting
        sfuRegistry.assignToMeeting(regionSfu.getId(), meetingId);
        
        return regionSfu;
    }
    
    private void setupCascade(SFUServer primary, SFUServer secondary, 
                              String meetingId) {
        // Establish media relay between SFUs
        // Uses dedicated backbone network (not public internet!)
        CascadeLink link = CascadeLink.builder()
            .meetingId(meetingId)
            .sourceSfu(primary.getId())
            .targetSfu(secondary.getId())
            .protocol(TransportProtocol.SRTP_OVER_TCP) // Reliable relay
            .build();
        
        primary.establishCascade(link);
        secondary.establishCascade(link.reverse());
    }
}
```

### Adaptive Quality Controller

```java
@Component
public class AdaptiveQualityController {
    
    /**
     * Called periodically (every 2 seconds) per participant.
     * Adjusts which simulcast layer each viewer receives.
     */
    public void adjustQuality(String meetingId, String viewerId) {
        ParticipantMetrics metrics = getMetrics(viewerId);
        List<String> senders = getActiveSenders(meetingId);
        String activeSpeaker = getActiveSpeaker(meetingId);
        
        for (String senderId : senders) {
            SimulcastLayer targetLayer = determineLayer(
                senderId, viewerId, activeSpeaker, metrics);
            
            sfuServer.setForwardingLayer(
                meetingId, senderId, viewerId, targetLayer);
        }
    }
    
    private SimulcastLayer determineLayer(String senderId, String viewerId,
                                          String activeSpeaker,
                                          ParticipantMetrics metrics) {
        // Rule 1: Bad network → everything low
        if (metrics.getPacketLoss() > 15) {
            return SimulcastLayer.AUDIO_ONLY; // Pause video!
        }
        if (metrics.getPacketLoss() > 5) {
            return SimulcastLayer.LOW; // 180p thumbnails only
        }
        
        // Rule 2: Active speaker gets high quality
        if (senderId.equals(activeSpeaker)) {
            return metrics.getAvailableBandwidth() > 1_500_000
                ? SimulcastLayer.HIGH    // 720p
                : SimulcastLayer.MEDIUM; // 360p
        }
        
        // Rule 3: Gallery view → medium for all
        int participantCount = getParticipantCount(viewerId);
        if (participantCount <= 9) {
            return SimulcastLayer.MEDIUM; // 360p grid
        }
        
        // Rule 4: Large meeting → low for non-speakers
        return SimulcastLayer.LOW; // 180p thumbnail
    }
    
    enum SimulcastLayer {
        HIGH(1_500_000, 720, 30),    // 1.5 Mbps, 720p, 30fps
        MEDIUM(500_000, 360, 25),    // 500 Kbps, 360p, 25fps
        LOW(150_000, 180, 15),       // 150 Kbps, 180p, 15fps
        AUDIO_ONLY(64_000, 0, 0);    // 64 Kbps audio only
        
        final int bitrate;
        final int height;
        final int fps;
        
        SimulcastLayer(int bitrate, int height, int fps) {
            this.bitrate = bitrate;
            this.height = height;
            this.fps = fps;
        }
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Active Speaker Detection

When 50 people are in a meeting, how do you detect WHO is currently speaking to highlight their video and give them high-quality stream?

<details>
<summary>🔑 Answer</summary>

**Audio Energy Analysis:**
```java
public class ActiveSpeakerDetector {
    
    // Track audio energy levels per participant
    private final Map<String, SlidingWindow> energyWindows = 
        new ConcurrentHashMap<>();
    
    // Called every 20ms (audio packet interval)
    public void onAudioPacket(String participantId, byte[] audioData) {
        double energy = calculateRMSEnergy(audioData);
        energyWindows.computeIfAbsent(participantId, 
            k -> new SlidingWindow(50)) // 1 second window
            .add(energy);
    }
    
    // Called every 500ms to determine active speaker
    public String getActiveSpeaker(String meetingId) {
        return energyWindows.entrySet().stream()
            .filter(e -> isInMeeting(e.getKey(), meetingId))
            .filter(e -> e.getValue().getAverage() > SILENCE_THRESHOLD)
            .max(Comparator.comparing(e -> e.getValue().getAverage()))
            .map(Map.Entry::getKey)
            .orElse(null); // No one speaking
    }
    
    // Smooth transitions: don't switch on every syllable!
    // Require 500ms+ of dominant audio before switching speaker
    // "Dominant speaker" algorithm used by Ooh (used in Ooh)
}
```

**Key considerations:**
- Don't switch on every syllable (add hysteresis: must be dominant for 500ms+)
- Handle interruptions gracefully (show both speakers briefly)
- Consider "dominant speaker" algorithm (Ooh, used in Ooh and WebRTC)
- Audio energy is computed on SFU server (not client!) for efficiency
</details>

---

## ❓ Interview Q&A

**Q1: Why use SFU over MCU for most meetings?**
> SFU just forwards packets — no decoding/encoding on server (10x less CPU!). Each client controls their own layout and can request different quality per stream. MCU creates single composite — less flexible, massive CPU cost. At Zoom's scale (millions of concurrent meetings), MCU would require 10x more servers. SFU scales linearly with participants; MCU scales quadratically with CPU.

**Q2: How do you handle a participant with bad internet?**
> Simulcast + adaptive forwarding! Sender uploads 3 quality layers (720p, 360p, 180p). SFU monitors receiver's RTCP feedback (packet loss, delay). When loss exceeds 5%: switch to lower layer. Above 15%: audio-only mode. When network improves: gradually step up quality. The sender doesn't change anything — SFU handles per-receiver adaptation!

**Q3: How would you implement end-to-end encryption?**
> E2E means server can't decrypt media! Approach: (1) Each participant generates ephemeral key pair on join, (2) Key exchange via "Messaging Layer Security" (MLS) protocol, (3) Shared meeting key distributed to all participants, (4) Client encrypts media before sending (SFU sees encrypted blobs), (5) Challenge: SFU can't do simulcast layer switching on encrypted content → "SFrame" format allows encrypted payload with unencrypted routing headers.

**Q4: How do you handle meetings with 1000+ participants?**
> Tiered architecture: (1) Panelists (10-20): full bidirectional SFU, send and receive, (2) Viewers (1000+): receive-only, get composite of active speakers, (3) MCU creates single high-quality composite for viewers, (4) CDN edge distribution for viewers (like live streaming!), (5) Latency for viewers: 2-5 seconds (acceptable for webinar), (6) Q&A/reactions flow through separate lightweight WebSocket channel.

**Q5: How do you detect and handle echo/feedback?**
> Acoustic Echo Cancellation (AEC): (1) Know what audio the speaker is PLAYING (reference signal), (2) When their mic picks up sound, subtract the reference signal, (3) Adaptive filter learns room acoustics in real-time, (4) This is done CLIENT-SIDE (not server), (5) Also: auto-mute when not speaking, noise suppression ML models (like Krisp/RNNoise), comfort noise generation during silence.

---

## 🔗 Related Topics
- [WebRTC](../../APIs/WebRTC.md) — Protocol for real-time media
- [WebSockets](../../APIs/WebSockets.md) — Signaling channel
- [CDN](../../BuildingBlocks/CDN.md) — Recording distribution & webinar streams
- [Load Balancing](../../BuildingBlocks/LoadBalancing.md) — SFU server allocation

---

*"The hardest part of video conferencing isn't the video. It's making 50 people feel like they're in the same room when they're scattered across 12 time zones with internet ranging from fiber to 3G on a moving train." — WebRTC Engineer* 🎥
