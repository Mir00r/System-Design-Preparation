# 📹 WebRTC: Browser-to-Browser Real-Time Communication — The Complete Guide

> *"WebRTC is one of the most technically fascinating things the browser can do. It lets two people on opposite sides of the planet stream live video to each other — directly, without any of that data touching a server — using nothing but a browser. No plugins. No apps. Pure internet magic."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [WebSockets](./WebSockets.md), [HTTP](./HTTP.md), [Networking Fundamentals](../Foundations/Networking/README.md)

---

## 📋 Table of Contents

1. [What is WebRTC?](#1-what-is-webrtc)
2. [The Architecture — P2P vs Relay vs SFU/MCU](#2-the-architecture)
3. [The Three Core APIs](#3-the-three-core-apis)
4. [How Two Browsers Find Each Other — ICE, STUN, TURN](#4-how-two-browsers-find-each-other)
5. [Signaling — The Dating App for Browsers](#5-signaling)
6. [SDP — The Negotiation Language](#6-sdp--the-negotiation-language)
7. [Complete Connection Flow — Step by Step](#7-complete-connection-flow)
8. [Media — Audio, Video, Screen Sharing](#8-media)
9. [Data Channels — Binary & Text P2P Transfer](#9-data-channels)
10. [Codecs — VP8/VP9/H.264/AV1 & Opus](#10-codecs)
11. [Security — DTLS, SRTP, Fingerprints](#11-security)
12. [Scaling — Mesh, SFU, MCU](#12-scaling)
13. [Java / Spring Boot — Signaling Server](#13-java--spring-boot-signaling-server)
14. [JavaScript — Complete Client Implementation](#14-javascript-complete-client-implementation)
15. [Real-World Use Cases & Architecture Patterns](#15-real-world-use-cases--architecture-patterns)
16. [Interesting Projects to Build](#16-interesting-projects-to-build)
17. [Common Pitfalls & Debugging](#17-common-pitfalls--debugging)
18. [Performance Tuning](#18-performance-tuning)
19. [Interview Q&A](#19-interview-qa)
20. [Cheat Sheet](#20-cheat-sheet)

---

## 1. What is WebRTC?

**WebRTC (Web Real-Time Communication)** is an open standard (W3C + IETF) that enables real-time audio, video, and data communication directly between browsers (and native apps) — without a media server in the middle.

```
THE PROBLEM WEBRTC SOLVES:

BEFORE WebRTC (2011):
  ┌──────────┐  video ──→  ┌──────────┐  video ──→  ┌──────────┐
  │  Alice   │             │  Server  │             │   Bob    │
  │ Browser  │             │ (Flash!) │             │ Browser  │
  └──────────┘             └──────────┘             └──────────┘
  
  • Required Flash, Silverlight, or native plugins
  • All video routed through central server (bandwidth cost!)
  • High latency (server decode → re-encode → send)
  • Flash security nightmares, constant updates

WITH WebRTC (2012+):
  ┌──────────┐                                       ┌──────────┐
  │  Alice   │ ←══════ direct P2P (encrypted) ══════→ │   Bob    │
  │ Browser  │                                       │ Browser  │
  └──────────┘                                       └──────────┘
       ↕                                                  ↕
  ┌──────────┐  (only for signaling, not media)     ┌──────────┐
  │ Signaling│←──────────── WebSocket ───────────────│ Signaling│
  │  Server  │                                       │  Server  │
  └──────────┘                                       └──────────┘
  
  • Browser-native, no plugins
  • P2P: media never touches your server (save massive bandwidth)
  • Sub-100ms latency (vs 200-500ms for server-routed)
  • Mandatory encryption (DTLS + SRTP)

WHAT WEBRTC CAN DO:
  ✅ Video calling (1:1 or group via SFU)
  ✅ Audio calling
  ✅ Screen sharing
  ✅ P2P file transfer (no size limit!)
  ✅ Multiplayer gaming (low-latency data channels)
  ✅ Remote desktop / IoT control
  ✅ Collaborative whiteboarding
  ✅ Live streaming (with modifications)

WHO USES WEBRTC:
  • Google Meet — WebRTC at its purest
  • Discord — voice channels
  • Snapchat / Instagram — video calling
  • Amazon Chime, Zoom (browser version)
  • Facebook Messenger (browser)
  • Cloudflare Stream — ingest pipeline
```

---

## 2. The Architecture

### The Three Deployment Models

```
MODEL 1: PEER-TO-PEER MESH (1:1 or small groups up to ~4)
  
  Alice ────────────────────────────── Bob
        ╲                          ╱
         ╲                        ╱
          Charlie ─────────── Dave
  
  Every peer connects to every other peer.
  N peers = N×(N-1)/2 connections
  
  Pros:  No media server cost, lowest latency, true P2P privacy
  Cons:  Each client uploads N-1 streams (CPU/bandwidth explosion)
         4 people → each uploads 3 streams
         10 people → each uploads 9 streams 💥 unusable
  
  USE: 1:1 calls, small groups (≤4), IoT device control

─────────────────────────────────────────────────────────────────

MODEL 2: SFU (Selective Forwarding Unit) — Most Common for Video Conferencing
  
  Alice ──→╗                     ╔══→ Alice
  Bob ────→║   SFU Media Server  ║══→ Bob
  Charlie ─║   (Janus, Mediasoup)║══→ Charlie
  Dave ───→╝                     ╚══→ Dave
  
  Each participant sends ONE stream to SFU.
  SFU forwards appropriate streams to each participant.
  
  Pros:  Each client uploads only 1 stream (scalable!)
         SFU is just forwarding (no decode/encode — low cost)
         Selective forwarding: skip streams user can't see
  Cons:  Requires a media server (cost)
         Media DOES touch server (privacy consideration)
  
  USE: Video conferencing (Google Meet, Zoom, Twitch), 2-1000 participants

─────────────────────────────────────────────────────────────────

MODEL 3: MCU (Multipoint Control Unit) — Legacy, Expensive
  
  Alice ──→╗                     ╔══→ Alice (single mixed stream)
  Bob ────→║   MCU Media Server  ║══→ Bob
  Charlie ─║  (decode all,        ║══→ Charlie
  Dave ───→╚   mix, re-encode)    ╚══→ Dave
  
  MCU decodes all streams, mixes into one, re-encodes, sends.
  
  Pros:  Clients receive only 1 stream (works on slow mobile)
  Cons:  EXTREMELY expensive (decode + transcode in real-time)
         Highest latency (decode+mix+encode delay)
  
  USE: Legacy enterprise video conferencing, WebEx old-style

─────────────────────────────────────────────────────────────────

CHOOSING THE RIGHT MODEL:

  Participants    Recommended
  ─────────────────────────────────
  1:1              P2P Mesh (free, private, lowest latency)
  2-4              P2P Mesh or SFU
  5-50             SFU (Janus, Mediasoup, LiveKit)
  50-1000          SFU with simulcast
  1000+            CDN + SFU hybrid (broadcast model)
```

---

## 3. The Three Core APIs

```
WEBRTC is built on exactly THREE browser APIs:

┌─────────────────────────────────────────────────────────────────┐
│                      WebRTC API Surface                         │
│                                                                 │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐ │
│  │  getUserMedia /       │  │  RTCPeerConnection               │ │
│  │  getDisplayMedia      │  │                                  │ │
│  │                       │  │  • The core of WebRTC            │ │
│  │  • Capture camera     │  │  • Manages peer connection       │ │
│  │  • Capture microphone │  │  • Handles ICE negotiation       │ │
│  │  • Screen share       │  │  • Manages codecs & SDP          │ │
│  │                       │  │  • Sends/receives media          │ │
│  │  Returns: MediaStream │  │  • Handles encryption            │ │
│  └──────────────────────┘  └──────────────────────────────────┘ │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  RTCDataChannel                                          │   │
│  │                                                          │   │
│  │  • P2P data (not media) — text, binary, JSON, files      │   │
│  │  • Based on SCTP (not UDP, not TCP — something better!)  │   │
│  │  • Configurable: reliable vs unreliable, ordered vs not  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. How Two Browsers Find Each Other

### The NAT Problem

```
THE FUNDAMENTAL CHALLENGE:
  Your computer isn't on the public internet. It's behind a NAT router.
  
  Alice's Network:          Internet:          Bob's Network:
  ┌─────────────────────┐              ┌───────────────────────┐
  │ Alice: 192.168.1.5  │              │ Bob: 192.168.1.3       │
  │ (private IP)        │   NAT Router │ (private IP)           │
  │      ↕              │     ↕        │       ↕                │
  │ Router: 67.x.x.x    │←──Public──→ │  Router: 98.x.x.x      │
  │ (public IP)         │             │  (public IP)            │
  └─────────────────────┘             └───────────────────────┘
  
  Alice only knows her private IP (192.168.1.5)
  Bob only knows his private IP (192.168.1.3)
  
  How does Alice connect directly to Bob?
  Private IPs are not routable on the internet!
  
  SOLUTION: ICE (Interactive Connectivity Establishment)
```

### ICE Candidates — Three Types

```
ICE CANDIDATE TYPES (in order of preference):

1. HOST CANDIDATE (fastest, works on LAN):
   IP: 192.168.1.5:54321  ← private IP, direct connection
   Works only if Alice and Bob are on same network (rare)

2. SERVER REFLEXIVE (works in most cases):
   IP: 67.x.x.x:54321     ← Alice's public IP as seen by STUN server
   
   STUN server tells Alice her public IP/port:
   Alice → STUN server: "What's my public IP?"
   STUN server → Alice: "You appear as 67.x.x.x:54321"
   
   Alice includes 67.x.x.x:54321 as a candidate.
   Bob tries connecting to 67.x.x.x:54321 → NAT router punches hole → Alice!
   
   Works: ~86% of NAT configurations (Full Cone, Restricted Cone)
   Fails: Symmetric NAT (enterprise firewalls!) 

3. RELAY CANDIDATE (always works, but indirect):
   IP: 52.y.y.y:3478      ← TURN server's IP
   
   When P2P fails (Symmetric NAT), traffic relays through TURN server.
   NOT P2P anymore, but still encrypted end-to-end.
   
   Cost: TURN server bandwidth = about the same as what you're sending
   Typical: 10-20% of calls end up using TURN

STUN vs TURN:
  ┌──────────────────────────────────────────────────────────────┐
  │  STUN (Session Traversal Utilities for NAT)                  │
  │  • Purpose: Discover your public IP and port                 │
  │  • Lightweight: just tell you your public address            │
  │  • Free public servers: stun.l.google.com:19302              │
  │  • Bandwidth: near zero (just a discovery query)             │
  │                                                              │
  │  TURN (Traversal Using Relays around NAT)                    │
  │  • Purpose: Relay media when P2P impossible                  │
  │  • Heavyweight: all media flows through it                   │
  │  • Cost: you pay for bandwidth (media relay)                 │
  │  • Required for: Symmetric NAT, strict enterprise firewalls  │
  │  • Must run your own or pay provider (Twilio, Xirsys)        │
  └──────────────────────────────────────────────────────────────┘

ICE CANDIDATE GATHERING PROCESS:
  
  1. Browser asks OS: "What network interfaces do I have?"
     → host candidate: 192.168.1.5:54321
  
  2. Browser contacts STUN server: "What's my public IP?"
     → srflx candidate: 67.x.x.x:54321
  
  3. Browser contacts TURN server: "Allocate a relay for me"
     → relay candidate: 52.y.y.y:3478
  
  4. Browser sends all candidates to remote peer (via signaling)
  
  5. ICE connectivity checks: both sides try each candidate pair
     Test A: local ↔ remote host candidates
     Test B: local ↔ remote srflx candidates
     Test C: local srflx ↔ remote relay
     ...
  
  6. Best working candidate pair selected → connection!
  
  This whole process: 100-3000ms (STUN: fast, TURN allocation: slower)
```

---

## 5. Signaling

### The Dating App Analogy

```
ANALOGY: Two people want to date but don't have each other's contact info.
  → They both post profiles on a dating app (signaling server)
  → The app connects them: "Hey Alice, Bob wants to chat"
  → They exchange contact details via the app
  → They meet in person (P2P connection) — the app is no longer involved!

WebRTC = same idea:
  → Signaling server helps browsers "find" each other
  → They exchange SDP offers/answers and ICE candidates via signaling
  → Direct P2P connection established — signaling server no longer involved for media!

WHAT SIGNALING SERVER DOES:
  ✅ Route SDP offer from Alice to Bob
  ✅ Route SDP answer from Bob to Alice
  ✅ Route ICE candidates between peers
  ✅ Handle room management (who's in the call?)
  ✅ Handle call state (ringing, connected, ended)

WHAT SIGNALING SERVER DOES NOT DO:
  ❌ Handle audio/video media (that's P2P!)
  ❌ See the contents of media (encrypted)
  ❌ Stay involved once P2P connection is established

WEBRTC DOESN'T SPECIFY SIGNALING PROTOCOL:
  You can use: WebSocket ✅ (most common)
               HTTP REST  ✅ (polling-based, simple but less real-time)
               XMPP       ✅ (enterprise)
               SIP        ✅ (telecom integration)
               Custom     ✅ (anything that gets messages from A to B)
```

---

## 6. SDP — The Negotiation Language

### Understanding SDP (Session Description Protocol)

```
SDP IS:
  A text format that describes a multimedia session:
  - What media tracks am I sending? (audio, video, data)
  - What codecs can I use? (VP8, H.264, Opus...)
  - What encryption keys to use?
  - What ICE credentials to use?
  - What bandwidth am I willing to use?

EXAMPLE SDP OFFER:
─────────────────────────────────────────────────────────────────
v=0                               ← version
o=- 123456789 0 IN IP4 0.0.0.0   ← origin
s=-                               ← session name
t=0 0                             ← timing (0=unbounded)
a=group:BUNDLE audio video        ← bundle A+V on same connection
a=msid-semantic: WMS stream1      ← media stream identifier

m=audio 9 UDP/TLS/RTP/SAVPF 111  ← audio media line, port 9 (ICE will give real port)
c=IN IP4 0.0.0.0                  ← connection (ICE will override)
a=rtcp-mux                        ← multiplex RTP and RTCP
a=ice-ufrag:XyZw                  ← ICE username fragment
a=ice-pwd:abc123securepassword    ← ICE password
a=fingerprint:sha-256 AB:CD:...   ← DTLS fingerprint (anti-MITM)
a=setup:actpass                   ← DTLS role negotiation
a=mid:audio                       ← media ID
a=rtpmap:111 opus/48000/2         ← codec: Opus, 48kHz, stereo
a=fmtp:111 minptime=10;useinbandfec=1 ← Opus params
a=sendrecv                        ← I can send AND receive audio

m=video 9 UDP/TLS/RTP/SAVPF 96 97 ← video media line
a=rtpmap:96 VP8/90000             ← VP8 codec, 90kHz clock
a=rtpmap:97 H264/90000            ← H.264 as fallback
a=rtcp-fb:96 nack                 ← request retransmit on loss
a=rtcp-fb:96 ccfb                 ← congestion control feedback
a=fmtp:97 profile-level-id=42e01f ← H.264 profile
a=sendrecv
─────────────────────────────────────────────────────────────────

OFFER/ANSWER MODEL:
  Alice creates OFFER SDP:
    "I can send audio (Opus) + video (VP8, H.264)"
    "I can receive audio + video"
    "My ICE credentials: ufrag=XyZw, pwd=abc123"
  
  Bob creates ANSWER SDP:
    "I'll receive audio with Opus ✅ (both support it)"
    "I'll receive video with VP8 ✅ (both support it)"
    "H.264: I don't support that one, removing it"
    "My ICE credentials: ufrag=AbCd, pwd=xyz789"
  
  NEGOTIATED: Opus for audio, VP8 for video
  This is called "codec negotiation" — intersection of both capabilities
```

---

## 7. Complete Connection Flow

### The Full Sequence Diagram

```
Alice                 Signaling Server              Bob
  │                         │                        │
  │──createRoom("room1")──→ │                        │
  │                         │──joinRoom("room1")────→│
  │                         │ ←─────────────────────│
  │                         │ "Bob joined!"           │
  │ ←──"Bob joined!" ──────│                        │
  │                         │                        │
  │══ PHASE 1: MEDIA SETUP ══════════════════════════│
  │                         │                        │
  │─getUserMedia()          │                        │
  │ (camera + mic granted)  │                        │
  │                         │         getUserMedia() │
  │                         │     (camera + mic)─────│
  │                         │                        │
  │══ PHASE 2: SDP OFFER ════════════════════════════│
  │                         │                        │
  │─new RTCPeerConnection() │                        │
  │─addTrack(videoStream)   │                        │
  │─addTrack(audioStream)   │                        │
  │─createOffer()           │                        │
  │─setLocalDescription(offer)                       │
  │──────────────────────────────────────────────────│
  │  signal({type:"offer",  │                        │
  │   to:"Bob", sdp:...})──→│──────────send offer──→ │
  │                         │                        │
  │══ PHASE 3: SDP ANSWER ═══════════════════════════│
  │                         │                        │
  │                         │   new RTCPeerConnection│
  │                         │   addTrack(video)      │
  │                         │   addTrack(audio)      │
  │                         │   setRemoteDescription │
  │                         │     (Alice's offer)    │
  │                         │   createAnswer()       │
  │                         │   setLocalDescription  │
  │                         │     (answer)           │
  │ ←──────────────────────│ ←────send answer───────│
  │ setRemoteDescription   │                         │
  │   (Bob's answer)        │                        │
  │                         │                        │
  │══ PHASE 4: ICE EXCHANGE ═════════════════════════│
  │                         │                        │
  │ onicecandidate fires    │                        │
  │ (host, srflx, relay)    │                        │
  │──signal(candidate)─────→│──forward candidate────→│
  │                         │                        │
  │                         │ ←─onicecandidate fires─│
  │                         │   (host, srflx, relay) │
  │ ←──────────────────────│ ←──signal(candidate)───│
  │ addIceCandidate()       │                        │
  │                         │                        │
  │══ PHASE 5: ICE CONNECTIVITY CHECKS ═════════════ │
  │                                                   │
  │ ←───────────────── STUN binding requests ────────→│
  │                   (checking all candidate pairs)  │
  │                                                   │
  │══ PHASE 6: DTLS HANDSHAKE (encryption) ══════════│
  │                                                   │
  │ ←─────────────── DTLS handshake ─────────────────→│
  │                   (verify fingerprints, exchange keys)  │
  │                                                   │
  │══ PHASE 7: MEDIA FLOWS ══════════════════════════ │
  │                                                   │
  │ ←══════════════ SRTP audio/video ════════════════→│
  │          (encrypted, peer-to-peer, real-time!)    │
  │                                                   │
  │ ontrack event fires on both sides                 │
  │ Attach stream to <video> element                  │
  │ 🎉 VIDEO CALL LIVE!                               │
  │                                                   │

Total time: ~1-3 seconds (depending on ICE, STUN, network)
```

---

## 8. Media

### getUserMedia — Capturing Camera & Microphone

```javascript
// Basic camera + microphone
const stream = await navigator.mediaDevices.getUserMedia({
    audio: {
        echoCancellation: true,      // remove echo (essential!)
        noiseSuppression: true,      // remove background noise
        autoGainControl: true,       // normalize volume
        sampleRate: 48000,           // 48kHz (Opus optimal)
        channelCount: 2              // stereo
    },
    video: {
        width: { ideal: 1280, max: 1920 },   // prefer 720p, accept 1080p
        height: { ideal: 720, max: 1080 },
        frameRate: { ideal: 30, max: 60 },
        facingMode: "user"                   // front camera on mobile
    }
});

// Constrain to available devices
const devices = await navigator.mediaDevices.enumerateDevices();
const cameras = devices.filter(d => d.kind === 'videoinput');
const microphones = devices.filter(d => d.kind === 'audioinput');
```

### Screen Sharing

```javascript
// Screen share (requires user gesture)
const screenStream = await navigator.mediaDevices.getDisplayMedia({
    video: {
        displaySurface: "monitor",   // "monitor", "window", "browser"
        width: { max: 1920 },
        height: { max: 1080 },
        frameRate: { max: 30 }
    },
    audio: {
        suppressLocalAudioPlayback: false,   // share system audio
        echoCancellation: false,             // don't process system audio
        noiseSuppression: false
    },
    selfBrowserSurface: "exclude"            // don't let them share our tab
});

// Replace camera track with screen track mid-call
const videoSender = peerConnection.getSenders()
    .find(s => s.track?.kind === 'video');
await videoSender.replaceTrack(screenStream.getVideoTracks()[0]);

// Listen for user stopping screen share
screenStream.getVideoTracks()[0].onended = () => {
    // Switch back to camera
    videoSender.replaceTrack(cameraStream.getVideoTracks()[0]);
};
```

### Track Management

```javascript
// Mute/unmute (disable track — still sends silence/black frames)
// Preferred over removing track (avoids SDP renegotiation)
const audioTrack = localStream.getAudioTracks()[0];
audioTrack.enabled = false; // mute
audioTrack.enabled = true;  // unmute

// Check remote stream quality
peerConnection.getStats().then(stats => {
    stats.forEach(report => {
        if (report.type === 'inbound-rtp' && report.kind === 'video') {
            console.log(`Video: ${report.frameWidth}x${report.frameHeight}
                         FPS: ${report.framesPerSecond}
                         Packets lost: ${report.packetsLost}
                         Jitter: ${report.jitter}ms`);
        }
    });
});
```

---

## 9. Data Channels

### SCTP — Better Than TCP or UDP for This

```
DATA CHANNEL PROTOCOL STACK:
  ┌─────────────────────────────────────────────────────┐
  │  Your Application Data                              │
  │  ───────────────────────────────────────────────    │
  │  SCTP (Stream Control Transmission Protocol)        │
  │  ───────────────────────────────────────────────    │
  │  DTLS (encryption, same keys as SRTP)               │
  │  ───────────────────────────────────────────────    │
  │  ICE / UDP                                          │
  └─────────────────────────────────────────────────────┘

WHY SCTP (not TCP, not UDP)?
  
  TCP: reliable, ordered, but head-of-line blocking
  UDP: unreliable, unordered, fast
  SCTP: configurable per message!
  
  SCTP DATA CHANNEL MODES:
  
  Mode 1: Reliable + Ordered (like TCP)
    → Use for: chat messages, file transfers
    → Retransmits on loss, delivers in order
  
  Mode 2: Reliable + Unordered
    → Use for: game events that must arrive but order doesn't matter
    → No head-of-line blocking!
  
  Mode 3: Unreliable + Ordered (partial reliability)
    → Use for: typing indicators (lose a few, no big deal)
    → maxRetransmits: 0 (or maxPacketLifeTime in ms)
  
  Mode 4: Unreliable + Unordered (like UDP over DTLS)
    → Use for: game position updates, real-time sensor data
    → Fastest, drop old data and send new
```

```javascript
// Reliable ordered data channel (chat)
const chatChannel = peerConnection.createDataChannel("chat", {
    ordered: true,        // messages arrive in order
    maxRetransmits: null  // unlimited retransmits = reliable
});

// Unreliable unordered data channel (game positions)
const gameChannel = peerConnection.createDataChannel("gamestate", {
    ordered: false,       // don't care about order
    maxRetransmits: 0     // drop if lost, don't retry
});

// File transfer channel
const fileChannel = peerConnection.createDataChannel("file", {
    ordered: true,
    maxRetransmits: null
});

fileChannel.binaryType = 'arraybuffer';

// Sending a file P2P (no server involved!)
async function sendFile(file) {
    const CHUNK_SIZE = 16384; // 16KB chunks
    const buffer = await file.arrayBuffer();
    
    // Send metadata first
    fileChannel.send(JSON.stringify({
        name: file.name, size: file.size, type: file.type
    }));
    
    // Send file in chunks
    let offset = 0;
    while (offset < buffer.byteLength) {
        // Respect flow control (don't flood the channel)
        if (fileChannel.bufferedAmount > fileChannel.bufferedAmountLowThreshold) {
            await new Promise(resolve => fileChannel.onbufferedamountlow = resolve);
        }
        const chunk = buffer.slice(offset, offset + CHUNK_SIZE);
        fileChannel.send(chunk);
        offset += CHUNK_SIZE;
    }
}
```

---

## 10. Codecs

### Video Codecs

```
VP8 (2010, Google):
  • Open source, royalty-free
  • Excellent browser support (all browsers)
  • Good quality at 500kbps-2Mbps
  • Used by: Google Meet, Discord
  
VP9 (2013, Google):
  • 50% better compression than VP8 at same quality
  • Open source, royalty-free
  • Higher encode CPU cost
  • Excellent for: low bandwidth users
  
H.264/AVC (2003):
  • Industry standard, hardware acceleration on ALL devices
  • Royalty-bearing (though WebRTC use is typically free)
  • Best for: iOS Safari (mandatory for Apple devices)
  • Profile: Baseline/Main/High (High = best quality)
  
AV1 (2018, Alliance for Open Media):
  • Next-gen: 50% better than VP9 at same quality
  • Open source, royalty-free
  • Very high encode cost (software only in many cases)
  • Growing hardware support (2023+)
  • Future: will replace VP9

CODEC SELECTION STRATEGY:
  if (Safari/iOS) → H.264 (mandatory)
  else if (budget limited) → VP9 (best quality/bandwidth ratio)
  else → VP8 (maximum compatibility) or H.264
  Future: AV1 when hardware encode is universal

RESOLUTION/BITRATE GUIDELINES:
  Quality      Resolution  Bitrate
  ─────────────────────────────────
  Thumbnail    160×90      60 kbps
  Low          320×180     150 kbps
  Medium       640×360     300 kbps
  HD (720p)    1280×720    1-2 Mbps
  Full HD      1920×1080   2-4 Mbps
  4K screen    3840×2160   10-20 Mbps (not typical for calling)
```

### Audio Codecs

```
Opus (2012, IETF):
  • THE standard for WebRTC audio
  • Variable bitrate: 6kbps (voice) to 510kbps (music)
  • Built-in: echo cancellation, noise reduction, PLC (packet loss concealment)
  • Handles: voice (SILK mode), music (CELT mode), and transitions
  • Typical: 16-32 kbps for voice (excellent quality)

G.711 (1972!):
  • Legacy PSTN codec
  • Required for interop with phone systems (SIP/PSTN)
  • Poor quality by modern standards, 64kbps fixed rate
  • Use only when connecting to telephone networks

G.722:
  • HD voice for PSTN/SIP interop
  • Better than G.711 but worse than Opus
  • Use for enterprise telephony

ALWAYS USE OPUS for browser-to-browser. Only switch if doing PSTN/SIP interop.
```

---

## 11. Security

### Mandatory Encryption

```
WEBRTC SECURITY IS MANDATORY (not optional!):

1. DTLS (Datagram TLS):
   • Encrypts ALL data channel and SRTP key exchange
   • Like TLS but for UDP (handles packet loss, reordering)
   • Version: DTLS 1.2 minimum (DTLS 1.3 increasingly used)
   • Certificate fingerprints exchanged via SDP (verified in DTLS handshake)
   
   FINGERPRINT VERIFICATION (prevents MITM):
   SDP contains: a=fingerprint:sha-256 AB:CD:EF:12:...
   During DTLS: "Is the certificate you're presenting the one in the SDP?"
   If fingerprint doesn't match → connection refused!
   
   This means: even if someone intercepts the signaling server and modifies
   the SDP, they can't change the fingerprint (they don't have the private key)
   
2. SRTP (Secure Real-time Transport Protocol):
   • Encrypts all audio and video media
   • Keys derived from DTLS handshake
   • Algorithm: AES-128-CM or AES-256-CM
   • SRTCP: encrypted RTCP (control/stats channel)
   
3. ICE CREDENTIALS:
   • ufrag + password authenticate ICE connectivity checks
   • Prevents unauthorized parties from injecting into the connection

RESULT: WebRTC media is encrypted end-to-end between peers.
Even the signaling server cannot decrypt media.
(Exception: TURN relay sees encrypted packets but can't decrypt them)
```

---

## 12. Scaling

### Simulcast — Efficient Multi-Quality Streaming

```
PROBLEM: In a 10-person call, participants have different bandwidths.
  Alice on fiber: can receive 1080p from everyone
  Bob on mobile: can only receive 360p from everyone

SIMULCAST SOLUTION:
  Each sender sends MULTIPLE quality layers simultaneously:
  
  Alice's camera
       │
       ├──→ Layer 0: 180p @ 150kbps  (for poor connections)
       ├──→ Layer 1: 360p @ 500kbps  (for medium connections)  
       └──→ Layer 2: 720p @ 1.5Mbps  (for good connections)
  
  SFU receives all 3 layers, forwards appropriate one to each receiver:
  Bob (mobile) ←── Layer 0 (180p) ──── SFU ──── Layer 2 (720p) ──→ Carol (fiber)

SPRING BOOT — Configure simulcast in SDP:

// The SFU negotiates simulcast via SDP extensions
// a=simulcast:send h;m;l (send 3 layers: high, medium, low)
// a=rid:h send
// a=rid:m send
// a=rid:l send
// a=extmap:1 urn:ietf:params:rtp-hdrext:sdes:mid
```

### SFU Architecture at Scale

```
LARGE-SCALE VIDEO CONFERENCING:

                              ┌─────────────────────────────┐
  Alice ──────────────────→  │  Edge SFU (Region: US-East) │
  Bob ────────────────────→  │  Janus / Mediasoup / LiveKit│
  Charlie ────────────────→  │  (Kubernetes, auto-scale)   │
                              └──────────────┬──────────────┘
                                             │ SFU cascade (forward media)
                              ┌──────────────↓──────────────┐
  Dave ───────────────────→  │  Edge SFU (Region: EU-West) │
  Eve ────────────────────→  │  Handles EU participants     │
  Frank ──────────────────→  └─────────────────────────────┘

POPULAR SFU SERVERS:
  Mediasoup  — Node.js + C++, extremely performant, low-level API
  Janus      — C, highly configurable, battle-tested
  LiveKit    — Go, batteries-included, open source, Kubernetes-native
  Pion       — Go library for building custom SFUs
  Kurento    — Java-based, good Spring Boot integration
  Amazon Chime SDK — managed SFU service
```

---

## 13. Java / Spring Boot — Signaling Server

### Complete Signaling Server with WebSocket

```java
// ─────────────────────────────────────────────────────────
// WebSocket Configuration
// ─────────────────────────────────────────────────────────
@Configuration
@EnableWebSocketMessageBroker
public class WebRTCSignalingConfig implements WebSocketMessageBrokerConfigurer {
    
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/signaling")
                .setAllowedOriginPatterns("*")
                .withSockJS(); // fallback for environments without WS
    }
    
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/app");
        registry.enableSimpleBroker("/topic", "/queue");
        registry.setUserDestinationPrefix("/user");
    }
}

// ─────────────────────────────────────────────────────────
// Room Management Service
// ─────────────────────────────────────────────────────────
@Service
public class RoomService {
    // roomId → Set of participant session IDs
    private final ConcurrentHashMap<String, Set<String>> rooms = new ConcurrentHashMap<>();
    // sessionId → userId
    private final ConcurrentHashMap<String, String> sessions = new ConcurrentHashMap<>();
    
    public Set<String> joinRoom(String roomId, String sessionId, String userId) {
        sessions.put(sessionId, userId);
        rooms.computeIfAbsent(roomId, k -> ConcurrentHashMap.newKeySet()).add(sessionId);
        return getParticipants(roomId);
    }
    
    public void leaveRoom(String sessionId) {
        rooms.values().forEach(participants -> participants.remove(sessionId));
        sessions.remove(sessionId);
    }
    
    public Set<String> getParticipants(String roomId) {
        return rooms.getOrDefault(roomId, Set.of());
    }
    
    public String findRoomBySession(String sessionId) {
        return rooms.entrySet().stream()
            .filter(e -> e.getValue().contains(sessionId))
            .map(Map.Entry::getKey)
            .findFirst().orElse(null);
    }
}

// ─────────────────────────────────────────────────────────
// Signaling Message Models
// ─────────────────────────────────────────────────────────
public sealed interface SignalingMessage permits
    JoinMessage, OfferMessage, AnswerMessage, 
    IceCandidateMessage, LeaveMessage {}

public record JoinMessage(String roomId, String userId) implements SignalingMessage {}
public record OfferMessage(String to, String from, String sdp) implements SignalingMessage {}
public record AnswerMessage(String to, String from, String sdp) implements SignalingMessage {}
public record IceCandidateMessage(
    String to, String from,
    String candidate, String sdpMid, int sdpMLineIndex
) implements SignalingMessage {}
public record LeaveMessage(String roomId) implements SignalingMessage {}

// ─────────────────────────────────────────────────────────
// Signaling Controller
// ─────────────────────────────────────────────────────────
@Controller
@RequiredArgsConstructor
public class SignalingController {
    
    private final RoomService roomService;
    private final SimpMessagingTemplate messaging;
    
    @MessageMapping("/join")
    public void handleJoin(JoinMessage msg, SimpMessageHeaderAccessor headerAccessor) {
        String sessionId = headerAccessor.getSessionId();
        Set<String> existing = roomService.joinRoom(msg.roomId(), sessionId, msg.userId());
        
        // Notify the new participant about existing peers
        messaging.convertAndSendToUser(sessionId, "/queue/participants",
            Map.of("type", "room-joined", "peers", existing, "roomId", msg.roomId()));
        
        // Notify existing participants that someone joined
        existing.stream()
            .filter(id -> !id.equals(sessionId))
            .forEach(peerId -> messaging.convertAndSendToUser(peerId, "/queue/signals",
                Map.of("type", "peer-joined", "peerId", sessionId, "userId", msg.userId())));
        
        log.info("User {} joined room {} (session: {})", msg.userId(), msg.roomId(), sessionId);
    }
    
    @MessageMapping("/offer")
    public void handleOffer(OfferMessage msg) {
        // Forward SDP offer to the target peer
        messaging.convertAndSendToUser(msg.to(), "/queue/signals",
            Map.of("type", "offer", "from", msg.from(), "sdp", msg.sdp()));
    }
    
    @MessageMapping("/answer")
    public void handleAnswer(AnswerMessage msg) {
        // Forward SDP answer to the target peer
        messaging.convertAndSendToUser(msg.to(), "/queue/signals",
            Map.of("type", "answer", "from", msg.from(), "sdp", msg.sdp()));
    }
    
    @MessageMapping("/ice-candidate")
    public void handleIceCandidate(IceCandidateMessage msg) {
        // Forward ICE candidate to the target peer
        messaging.convertAndSendToUser(msg.to(), "/queue/signals",
            Map.of("type", "ice-candidate", "from", msg.from(),
                   "candidate", msg.candidate(),
                   "sdpMid", msg.sdpMid(),
                   "sdpMLineIndex", msg.sdpMLineIndex()));
    }
    
    @EventListener
    public void handleDisconnect(SessionDisconnectEvent event) {
        String sessionId = event.getSessionId();
        String roomId = roomService.findRoomBySession(sessionId);
        
        if (roomId != null) {
            // Notify room that this peer left
            roomService.getParticipants(roomId).forEach(peerId ->
                messaging.convertAndSendToUser(peerId, "/queue/signals",
                    Map.of("type", "peer-left", "peerId", sessionId)));
        }
        roomService.leaveRoom(sessionId);
    }
}
```

### TURN Server Configuration (Coturn)

```java
// Serve ICE configuration to clients (STUN + TURN with auth)
@RestController
@RequestMapping("/api/webrtc")
public class WebRTCConfigController {
    
    @Value("${webrtc.turn.host}")
    private String turnHost;
    
    @Value("${webrtc.turn.secret}")
    private String turnSecret;
    
    @GetMapping("/ice-configuration")
    public IceConfiguration getIceConfiguration(Principal principal) {
        // Generate time-limited TURN credentials (HMAC-based)
        // Expires in 1 hour — prevents credential abuse
        long timestamp = (System.currentTimeMillis() / 1000) + 3600;
        String username = timestamp + ":" + principal.getName();
        String credential = generateHmacCredential(username, turnSecret);
        
        return new IceConfiguration(
            List.of(
                new IceServer("stun:stun.l.google.com:19302"),   // free public STUN
                new IceServer("stun:" + turnHost + ":3478"),     // your STUN
                new IceServer("turn:" + turnHost + ":3478", username, credential),  // TURN UDP
                new IceServer("turns:" + turnHost + ":5349", username, credential)  // TURN TLS (HTTPS firewall bypass)
            )
        );
    }
    
    private String generateHmacCredential(String username, String secret) {
        try {
            Mac mac = Mac.getInstance("HmacSHA1");
            mac.init(new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), "HmacSHA1"));
            byte[] hash = mac.doFinal(username.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate TURN credentials", e);
        }
    }
}
```

---

## 14. JavaScript — Complete Client Implementation

### Full Video Call Application

```javascript
class VideoCallClient {
    constructor(signalingUrl) {
        this.socket = new SockJS(signalingUrl);
        this.stompClient = Stomp.over(this.socket);
        this.peerConnections = new Map(); // peerId → RTCPeerConnection
        this.localStream = null;
        this.iceConfig = null;
    }
    
    async initialize() {
        // 1. Fetch ICE configuration from server
        const res = await fetch('/api/webrtc/ice-configuration');
        this.iceConfig = await res.json();
        
        // 2. Get local media
        this.localStream = await navigator.mediaDevices.getUserMedia({
            video: { width: 1280, height: 720, frameRate: 30 },
            audio: { echoCancellation: true, noiseSuppression: true }
        });
        
        // Show local preview
        document.getElementById('local-video').srcObject = this.localStream;
        
        // 3. Connect to signaling server
        await this.connectSignaling();
    }
    
    connectSignaling() {
        return new Promise(resolve => {
            this.stompClient.connect({}, () => {
                // Subscribe to personal signal channel
                this.stompClient.subscribe('/user/queue/signals', (msg) => {
                    this.handleSignal(JSON.parse(msg.body));
                });
                this.stompClient.subscribe('/user/queue/participants', (msg) => {
                    this.handleRoomJoined(JSON.parse(msg.body));
                });
                resolve();
            });
        });
    }
    
    async joinRoom(roomId, userId) {
        this.stompClient.send('/app/join', {}, JSON.stringify({ roomId, userId }));
    }
    
    async handleRoomJoined({ peers }) {
        // For each existing peer, create a connection and send an offer
        for (const peerId of peers) {
            if (peerId !== this.mySessionId) {
                await this.createOffer(peerId);
            }
        }
    }
    
    async createPeerConnection(peerId) {
        const pc = new RTCPeerConnection(this.iceConfig);
        this.peerConnections.set(peerId, pc);
        
        // Add local tracks to the connection
        this.localStream.getTracks().forEach(track => pc.addTrack(track, this.localStream));
        
        // Handle remote tracks (when remote sends their video/audio)
        pc.ontrack = (event) => {
            const remoteVideo = this.getOrCreateRemoteVideo(peerId);
            if (remoteVideo.srcObject !== event.streams[0]) {
                remoteVideo.srcObject = event.streams[0];
            }
        };
        
        // ICE candidate ready — send to peer via signaling
        pc.onicecandidate = (event) => {
            if (event.candidate) {
                this.stompClient.send('/app/ice-candidate', {}, JSON.stringify({
                    to: peerId,
                    from: this.mySessionId,
                    candidate: event.candidate.candidate,
                    sdpMid: event.candidate.sdpMid,
                    sdpMLineIndex: event.candidate.sdpMLineIndex
                }));
            }
        };
        
        // Connection state monitoring
        pc.onconnectionstatechange = () => {
            console.log(`Connection to ${peerId}: ${pc.connectionState}`);
            if (pc.connectionState === 'failed') {
                this.restartIce(peerId); // ICE restart on failure
            }
        };
        
        // Bandwidth/quality stats
        pc.oniceconnectionstatechange = () => {
            if (pc.iceConnectionState === 'connected') {
                this.startStatsMonitoring(pc, peerId);
            }
        };
        
        return pc;
    }
    
    async createOffer(peerId) {
        const pc = await this.createPeerConnection(peerId);
        
        // Create and send offer
        const offer = await pc.createOffer({
            offerToReceiveAudio: true,
            offerToReceiveVideo: true
        });
        await pc.setLocalDescription(offer);
        
        this.stompClient.send('/app/offer', {}, JSON.stringify({
            to: peerId,
            from: this.mySessionId,
            sdp: offer.sdp
        }));
    }
    
    async handleSignal({ type, from, sdp, candidate, sdpMid, sdpMLineIndex }) {
        switch (type) {
            case 'peer-joined':
                // New peer joined — they'll send us an offer
                break;
                
            case 'offer':
                // Received offer — create answer
                const pc = await this.createPeerConnection(from);
                await pc.setRemoteDescription({ type: 'offer', sdp });
                const answer = await pc.createAnswer();
                await pc.setLocalDescription(answer);
                this.stompClient.send('/app/answer', {}, JSON.stringify({
                    to: from, from: this.mySessionId, sdp: answer.sdp
                }));
                break;
                
            case 'answer':
                // Received answer — complete the offer/answer exchange
                const connection = this.peerConnections.get(from);
                await connection.setRemoteDescription({ type: 'answer', sdp });
                break;
                
            case 'ice-candidate':
                // Add received ICE candidate
                const conn = this.peerConnections.get(from);
                if (conn) {
                    await conn.addIceCandidate({ candidate, sdpMid, sdpMLineIndex });
                }
                break;
                
            case 'peer-left':
                // Peer disconnected — clean up
                this.removePeer(from);
                break;
        }
    }
    
    async restartIce(peerId) {
        const pc = this.peerConnections.get(peerId);
        if (!pc) return;
        
        // ICE restart: generates new ICE credentials, triggers new gathering
        const offer = await pc.createOffer({ iceRestart: true });
        await pc.setLocalDescription(offer);
        this.stompClient.send('/app/offer', {}, JSON.stringify({
            to: peerId, from: this.mySessionId, sdp: offer.sdp
        }));
    }
    
    startStatsMonitoring(pc, peerId) {
        setInterval(async () => {
            const stats = await pc.getStats();
            stats.forEach(report => {
                if (report.type === 'candidate-pair' && report.state === 'succeeded') {
                    console.log(`RTT to ${peerId}: ${report.currentRoundTripTime * 1000}ms`);
                }
            });
        }, 5000);
    }
    
    getOrCreateRemoteVideo(peerId) {
        let video = document.getElementById(`video-${peerId}`);
        if (!video) {
            video = document.createElement('video');
            video.id = `video-${peerId}`;
            video.autoplay = true;
            video.playsinline = true;
            document.getElementById('remote-videos').appendChild(video);
        }
        return video;
    }
    
    removePeer(peerId) {
        const pc = this.peerConnections.get(peerId);
        if (pc) { pc.close(); }
        this.peerConnections.delete(peerId);
        document.getElementById(`video-${peerId}`)?.remove();
    }
    
    async toggleMute() {
        const audioTrack = this.localStream.getAudioTracks()[0];
        audioTrack.enabled = !audioTrack.enabled;
        return !audioTrack.enabled; // returns true if muted
    }
    
    async toggleCamera() {
        const videoTrack = this.localStream.getVideoTracks()[0];
        videoTrack.enabled = !videoTrack.enabled;
        return !videoTrack.enabled;
    }
    
    hangUp() {
        this.peerConnections.forEach(pc => pc.close());
        this.peerConnections.clear();
        this.localStream.getTracks().forEach(t => t.stop());
        this.stompClient.disconnect();
    }
}
```

---

## 15. Real-World Use Cases & Architecture Patterns

### Industry Examples

```
GOOGLE MEET:
  Architecture: SFU (own infrastructure)
  Scale: millions of concurrent participants
  Special: Cascade SFU (multiple SFU regions linked)
  Interesting: Real-time noise cancellation via ML in browser
  Codec: VP9 (Google owns it, royalty-free, better than VP8)

DISCORD:
  Architecture: SFU + Voice Activity Detection
  Scale: hundreds of thousands concurrent voice channels
  Special: Persistent voice channels (different from call model)
  Interesting: Voice-only mode uses Opus at 64kbps (very efficient)

CLOUDFLARE STREAM LIVE:
  Architecture: WebRTC ingest → transcode → HLS/DASH delivery
  Use: Ultra-low latency live streaming from browser
  Interesting: WebRTC is NOT for delivery to millions (use HLS)
               WebRTC is used for the sub-second ingest from broadcaster

ZOOM (browser version):
  Architecture: MCU + SFU hybrid depending on meeting size
  Special: Zoom's custom codec optimizations
  Interesting: Zoom's desktop app does NOT use WebRTC
               (their own proprietary stack)
               Zoom.us browser tab uses WebRTC

FACEBOOK MESSENGER ROOMS:
  Architecture: SFU (Mediasoup-based)
  Scale: up to 50 participants
  Interesting: Uses simulcast aggressively for mobile users
```

---

## 16. Interesting Projects to Build

### 🚀 Project 1: P2P File Sharing — "AirDrop for Web"

```
CONCEPT: Send ANY file directly browser-to-browser, zero server storage
TECH: WebRTC Data Channels, chunk-based transfer
UNIQUE: Works across different LANs, encrypted, no file size limit

ARCHITECTURE:
  Sender                    Signaling Server          Receiver
    │─── Creates room ──────────────────────→           │
    │                       │←──── Joins room ──────────│
    │───── SDP Offer ───────→────── Forwarded ─────────→│
    │←──── SDP Answer ──────←────── Forwarded ──────────│
    │←════ ICE candidates ══════════════════════════════│
    │                                                    │
    │══════ RTCDataChannel established (P2P) ══════════→│
    │──── file metadata JSON ──────────────────────────→│
    │──── ArrayBuffer chunk 1 (16KB) ──────────────────→│
    │──── ArrayBuffer chunk 2 (16KB) ──────────────────→│
    │──── [BufferedAmount flow control] ─────────────────│
    │──── ArrayBuffer chunk N ──────────────────────────→│
    │                       "Transfer complete: 1.2GB in 45s at 22 MB/s"

INTERESTING TWIST: 
  - Add QR code for the room link (phone to desktop file sharing!)
  - Show real-time transfer progress bar
  - Support batch file sending (send entire folder as zip)
```

### 🚀 Project 2: Multiplayer Browser Game with WebRTC Data Channels

```java
// Game state synchronization service
@Component
public class GameStateSyncService {
    
    // Each player's game loop sends position 60 times per second
    // Data channel: unreliable + unordered (latest state only)
    
    record PlayerState(String playerId, double x, double y, 
                       double velocityX, double velocityY, 
                       long timestamp) {}
    
    // Server authoritative state (for cheat prevention)
    // WebRTC peer handles the actual transport
}
```

```javascript
// Client: send player position over unreliable data channel
const gameChannel = peerConnection.createDataChannel("game", {
    ordered: false,
    maxRetransmits: 0    // drop old positions, send new ones
});

// 60fps game loop
function gameLoop() {
    if (gameChannel.readyState === 'open') {
        gameChannel.send(JSON.stringify({
            x: player.x, y: player.y,
            vx: player.vx, vy: player.vy,
            t: performance.now()
        }));
    }
    requestAnimationFrame(gameLoop);
}
```

### 🚀 Project 3: Real-Time Collaborative Code Editor

```
CONCEPT: VS Code Live Share, but in-browser with WebRTC data channels
TECH: WebRTC Data Channel + CRDT (Yjs) for conflict-free edits
UNIQUE: Zero operational transformation server needed — pure P2P sync

Each keystroke:
  Editor → CRDT operation → Data Channel → Remote peer → CRDT apply → Editor
  
CRDT ensures: even if messages arrive out of order, final state is consistent
Library: Yjs + y-webrtc provider (does exactly this!)
Use case: pair programming, technical interviews, code review
```

### 🚀 Project 4: WebRTC Network Quality Monitor

```
BUILD: Visualize your WebRTC connection quality in real-time
METRICS TO SHOW:
  - ICE connection path (Host P2P / STUN relay / TURN relay)
  - Round-trip time (RTT) ms — from getStats()
  - Packet loss % — packetsLost / packetsSent
  - Jitter — variation in packet arrival time
  - Available bandwidth estimate
  - Frame drops (framesDropped metric)
  - Audio levels (audioLevel metric)

WHY IT'S USEFUL:
  Identify when you need TURN, diagnose quality issues,
  understand how different networks affect WebRTC
  
TOOLS: WebRTC getStats() API, Chart.js for visualization
```

### 🚀 Project 5: WebRTC Recording Service

```java
@Service
public class CallRecordingService {
    // WebRTC doesn't have built-in recording
    // Solution: MediaRecorder API (client-side) OR server-side (Kurento/GStreamer)
    
    // Client-side approach: record your own outgoing stream
    // Server-side: route through Kurento media server, record there
    
    // For compliance recording: MUST be server-side
    // GDPR consideration: inform users they're being recorded!
}
```

---

## 17. Common Pitfalls & Debugging

**1. Race condition: ICE candidates before setRemoteDescription** — ICE candidates can arrive before the remote description is set. Queue them: `if (!pc.remoteDescription) { pendingCandidates.push(candidate); return; }` Then apply queued candidates after `setRemoteDescription` succeeds.

**2. Not handling ICE failures** — `iceConnectionState === 'failed'` happens in ~10-15% of connections (strict firewalls, symmetric NAT without TURN). Must implement ICE restart: `pc.createOffer({ iceRestart: true })`. Without this, your app silently breaks for enterprise users.

**3. Not deploying TURN** — 10-20% of real-world connections REQUIRE TURN (symmetric NAT, corporate firewalls). Apps without TURN show "connecting forever..." for those users. Always deploy a TURN server.

**4. Memory leaks from unremoved tracks** — Call `localStream.getTracks().forEach(t => t.stop())` on hangup. Otherwise camera light stays on (track still active) and memory leaks.

**5. Safari/iOS compatibility** — Safari requires explicit `playsinline` attribute on video elements (`<video playsinline autoplay>`). Safari may require H.264 codec (add to SDP). iOS requires a user gesture to start getUserMedia.

**6. Mobile battery drain** — Camera + video encoding drains battery fast. Offer users the option to turn off video when on battery. Consider reducing resolution on mobile (`facingMode: "user", width: 640`).

### Debugging Tools

```
BROWSER DEVTOOLS:
  chrome://webrtc-internals/          ← GOLD MINE for debugging
  about:webrtc (Firefox)
  
  Shows: SDP negotiation, ICE candidates, stats over time,
         codec negotiation, connection state timeline

NETWORK DEBUGGING:
  Wireshark with WebRTC/DTLS dissector
  Filter: udp.port == 3478  (STUN/TURN traffic)
  
STATS API DEBUGGING:
  const report = await pc.getStats();
  // Look for:
  // - candidate-pair.state === 'succeeded' → which path is used
  // - inbound-rtp.jitter > 50ms → poor network
  // - inbound-rtp.packetsLost > 5% → significant packet loss
  // - outbound-rtp.qualityLimitationReason → CPU/bandwidth limiting
```

---

## 18. Performance Tuning

### Bandwidth Management

```javascript
// Limit sender bitrate (prevent overwhelming slow receivers)
async function setBandwidthLimit(pc, maxKbps) {
    const sender = pc.getSenders().find(s => s.track.kind === 'video');
    const params = sender.getParameters();
    
    if (!params.encodings) params.encodings = [{}];
    params.encodings[0].maxBitrate = maxKbps * 1000; // in bps
    
    await sender.setParameters(params);
}

// Adaptive bitrate based on network quality
pc.onconnectionstatechange = async () => {
    const stats = await pc.getStats();
    stats.forEach(report => {
        if (report.type === 'candidate-pair' && report.state === 'succeeded') {
            const rtt = report.currentRoundTripTime * 1000;
            if (rtt > 300) {
                setBandwidthLimit(pc, 300);   // poor network: reduce to 300kbps
            } else if (rtt < 50) {
                setBandwidthLimit(pc, 2000);  // good network: allow 2Mbps
            }
        }
    });
};
```

---

## 19. Interview Q&A

**Q: Explain how WebRTC establishes a peer-to-peer connection between two browsers behind NATs.**
> A: (1) **Signaling**: Both peers connect to a signaling server (via WebSocket). Alice creates an SDP offer (codec capabilities, ICE credentials) and sends it via signaling. Bob creates an answer and sends it back. (2) **ICE gathering**: Each browser discovers its candidates — host (private IP), server reflexive (public IP from STUN), and relay (TURN server). These are sent via signaling. (3) **ICE connectivity checks**: Both sides perform STUN binding requests to all candidate pairs. The NAT router punches a hole when it sees outbound requests, allowing inbound replies. (4) **Best pair selected**: highest priority working candidate pair (P2P if possible, TURN if NAT is symmetric). (5) **DTLS handshake**: encrypt the connection, verify certificate fingerprints from SDP. (6) **SRTP**: media flows encrypted peer-to-peer. Total: ~1-3 seconds.

---

**Q: What is the difference between STUN and TURN? When does TURN get used?**
> A: **STUN** (Session Traversal Utilities for NAT) tells a client its public IP and port as seen from the internet. It's a lightweight discovery service — just a query/response. Used by ~80-85% of WebRTC connections to discover their server-reflexive candidate. Free public servers exist. **TURN** (Traversal Using Relays Around NAT) is a relay server — when P2P fails, media flows through TURN encrypted. It's used when: (1) Symmetric NAT maps outgoing connections to different ports per destination, making STUN reflexive candidates unusable for incoming connections, or (2) corporate firewalls block UDP entirely (TURN over TCP/TLS bypasses this). ~10-20% of real-world connections require TURN. TURN costs money (you relay all media bandwidth), so good ICE implementation tries P2P first and only falls back to TURN.

---

**Q: What is SDP and what does the offer/answer model mean?**
> A: **SDP (Session Description Protocol)** is a text format that describes a multimedia session — what media types, codecs, security keys, and ICE credentials the peer wants to use. The **offer/answer model**: the calling party (Alice) creates an offer SDP listing everything it supports ("I can do VP8, VP9, H.264 video; Opus, G.722 audio"). The receiving party (Bob) creates an answer SDP that is the intersection: "I support VP8 and Opus; let's use those." This negotiation determines the codecs used for the call. The SDP also contains DTLS fingerprints (certificate hash) which are verified during the DTLS handshake to prevent MITM attacks — even if the signaling server is compromised, it can't change the fingerprint without the private key.

---

**Q: What is the difference between SFU and MCU? Which would you choose for Google Meet-scale?**
> A: **MCU (Multipoint Control Unit)**: receives all streams, decodes them, mixes into a single composed stream, and re-encodes for each participant. Pro: clients receive only 1 stream regardless of participant count. Con: extremely CPU-intensive (decode+transcode in real-time), adds encoding latency. **SFU (Selective Forwarding Unit)**: receives all streams, forwards the appropriate quality layer to each participant without decoding. Pro: very efficient (just routing), each participant sends 1 stream. Con: clients receive N-1 streams (mitigated by simulcast + bandwidth adaptation). **For Google Meet scale**: SFU every time. MCU CPU cost makes it economically impossible at millions of calls. SFUs like Janus, Mediasoup, LiveKit, Pion handle 100s of participants per server. Simulcast (sending 3 quality layers) + SFU quality selection solves the bandwidth problem.

---

**Q: How would you design a video conferencing system for 1000 concurrent participants?**
> A: (1) **Architecture**: SFU (not MCU, not P2P mesh). Each participant sends 1 stream to the nearest SFU. SFU forwards streams selectively. (2) **Simulcast**: each sender sends 3 resolution layers (180p/360p/720p). SFU sends appropriate layer based on receiver bandwidth. (3) **SFU cascade**: multiple SFU instances per region, connected via server-side links. EU participants connect to EU SFU, US to US SFU. (4) **Visible tile limit**: show maximum 25 tiles — don't forward streams for off-screen participants (SFU selective forwarding). (5) **TURN**: deploy TURN servers in each region for users behind strict NAT. (6) **Signaling**: WebSocket-based, horizontally scalable via pub/sub (Redis). (7) **Codecs**: VP9 with simulcast. (8) **Monitoring**: track RTT, packet loss, jitter per participant for quality adaptation.

---

**Q: How is WebRTC secured? Can the signaling server decrypt the video?**
> A: No. WebRTC mandates two encryption layers: **DTLS** encrypts the data channel and negotiates SRTP keys. **SRTP** encrypts all audio/video media. The key exchange happens directly between peers via DTLS — the signaling server only sees the SDP, which contains certificate fingerprints, not the actual keys. Since the signaling server doesn't have the private keys, it cannot decrypt the media stream. Additionally, fingerprints in the SDP are verified during the DTLS handshake — if a MITM modifies the SDP, the fingerprint won't match the actual certificate, and the connection is rejected. TURN servers only relay encrypted SRTP packets — they can't decrypt them either. End-to-end encryption for group calls (E2EE) requires additional key management on top of this basic WebRTC security model.

---

## 20. Cheat Sheet

```
WEBRTC COMPONENT QUICK REFERENCE:
  getUserMedia     → Capture camera, mic, screen
  RTCPeerConnection→ Core P2P connection, ICE, DTLS, SRTP
  RTCDataChannel   → P2P data (text, binary, files)
  
ICE CANDIDATE TYPES:
  host    → private IP (LAN only)
  srflx   → public IP via STUN (works most of the time)
  relay   → TURN server (always works, not P2P)
  
CONNECTION FLOW:
  createOffer → setLocalDescription → [signaling] →
  setRemoteDescription → createAnswer → setLocalDescription → [signaling] →
  setRemoteDescription → ICE exchange → DTLS → SRTP → 🎉 LIVE!
  
SFU vs MCU vs Mesh:
  P2P Mesh: ≤4 people, free, private, full P2P
  SFU:      2-1000 people, efficient, media touches server
  MCU:      legacy, expensive, high CPU, single stream per client

CODEC CHOICES:
  Video: VP8 (compat), VP9 (quality), H.264 (Safari/iOS), AV1 (future)
  Audio: Opus always (6-510kbps, built-in noise cancel)

DATA CHANNEL MODES:
  ordered+reliable     → chat, file transfer
  ordered+unreliable   → game events (must arrive, order matters)
  unordered+unreliable → game positions, sensor data (fastest)

COMMON ISSUES:
  "Connecting forever" → Need TURN server
  "Works LAN not WAN" → STUN server needed
  "Works Chrome not Safari" → H.264 codec + playsinline attribute
  "Leaking memory/camera light stays on" → track.stop() on hangup
  "Failed after 30s" → ICE failure, need ICE restart logic
```

---

## 🔗 What to Read Next

1. **[APIs/WebSockets.md](./WebSockets.md)** — For cases where full P2P isn't needed
2. **[APIs/ServerSentEvents.md](./ServerSentEvents.md)** — One-way streaming alternative
3. **[APIs/HTTP.md](./HTTP.md)** — HTTP fundamentals (WebRTC uses it for signaling)
4. **[BuildingBlocks/CDN.md](../BuildingBlocks/CDN.md)** — For distributing WebRTC app assets globally
5. **[SystemDesignCaseStudies/DesignWhatsApp.md](../SystemDesignCaseStudies/DesignWhatsApp.md)** — See WebRTC in a complete system design

---

*[← WebSockets](./WebSockets.md) | [Back to Index](../INDEX.md) | [Next: Server-Sent Events →](./ServerSentEvents.md)*
