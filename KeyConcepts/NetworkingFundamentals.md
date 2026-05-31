# 🌐 Networking Fundamentals: TCP, HTTP, DNS & Beyond 🚀

> **"You can't design systems you don't understand. And all systems are networks. So if you don't understand networking, you don't understand systems."**

**⏱️ Estimated Time**: 45 minutes  
**🎯 Difficulty**: 🟡 Intermediate  
**🔗 Prerequisites**: [RESTful APIs](../APIs/RESTful.md) | [CAP Theorem](./CAPTheorem.md) | [Scalability](./Scalability.md)

---

## 📋 Table of Contents

1. [The OSI Model — A Mental Map](#-the-osi-model)
2. [TCP vs UDP — The Core Choice](#-tcp-vs-udp)
3. [TCP Deep Dive — Connection, Flow, Congestion](#-tcp-deep-dive)
4. [HTTP/1.1 vs HTTP/2 vs HTTP/3](#-http-versions)
5. [DNS — The Internet's Phone Book](#-dns)
6. [CDN — Content Delivery Networks](#-cdn)
7. [Load Balancing at the Network Level](#-load-balancing)
8. [WebSockets and Long-Lived Connections](#-websockets)
9. [Network Latency — Causes and Solutions](#-network-latency)
10. [Interview Q&A](#-interview-qa)
11. [Mini Challenge](#-mini-challenge)

---

## 🗺️ The OSI Model

Think of the OSI model as a **postal system**: layers of packaging that ensure a letter gets from one address to another, regardless of the underlying postal network:

```
OSI MODEL — 7 Layers:
────────────────────────────────────────────────────────────────────────────────
Layer 7  APPLICATION    HTTP, gRPC, DNS, SMTP           You write code here
Layer 6  PRESENTATION   TLS/SSL, JSON encoding           Data encryption/format
Layer 5  SESSION        TCP sessions, WebSocket          Connection management
Layer 4  TRANSPORT      TCP, UDP, QUIC                   Port numbers, reliability
Layer 3  NETWORK        IP, ICMP, BGP                    IP addresses, routing
Layer 2  DATA LINK      Ethernet, WiFi (802.11)          MAC addresses
Layer 1  PHYSICAL       Cables, fiber, radio waves       The actual wire/air

When you type google.com in your browser:
  L7: Your browser constructs an HTTP GET request
  L6: TLS encrypts the HTTP request
  L4: TCP wraps it with port 443 (source), port 80/443 (dest)
  L3: IP wraps it with source IP (your machine) + dest IP (Google's server)
  L2: Ethernet frame targets your router's MAC address
  L1: Bits fly over your WiFi radio
```

**Interview shortcut:** FAANG interviews typically focus on L3 (IP routing), L4 (TCP/UDP), and L7 (HTTP/gRPC/WebSocket). You rarely need L1-L2 depth.

---

## 🔄 TCP vs UDP

```
TCP                                   UDP
──────────────────────────────────────────────────────────────────────────────
Connection-oriented (3-way handshake) Connectionless (no setup)
Reliable delivery (ACK + retransmit)  Best-effort (no ACK)
Ordered packets                       Unordered (app handles ordering)
Flow control + congestion control     No flow control
Slower (overhead)                     Fast (minimal overhead)
~20-60 byte header                    8 byte header

USE TCP FOR:                          USE UDP FOR:
  Web (HTTP/1, HTTP/2)                  Video streaming (Netflix, YouTube)
  File transfer (FTP, SFTP)             Voice/Video calls (WebRTC, VoIP)
  Email (SMTP, IMAP)                    Online gaming (real-time position)
  Database connections                  DNS lookups (small, latency-sensitive)
  APIs (REST, gRPC over HTTP/2)         IoT sensor data (loss acceptable)
```

### The 3-Way Handshake

```
TCP CONNECTION ESTABLISHMENT:

Client                              Server
  │                                   │
  │─────── SYN (seq=x) ───────────────→│  "Hey, I want to connect"
  │                                   │
  │←─── SYN-ACK (seq=y, ack=x+1) ────│  "OK, I'm ready. Here's my seq"
  │                                   │
  │────── ACK (ack=y+1) ──────────────→│  "Got it. Connection open!"
  │                                   │

Total: 1.5 round trips = 1.5 × RTT latency before first data byte.

TCP CONNECTION TEARDOWN (4-way):
Client ──FIN──→ Server  (client done sending)
Client ←──ACK── Server  (server acknowledges)
Client ←──FIN── Server  (server done sending)
Client ──ACK──→ Server  (client acknowledges)

TIME_WAIT state: Client waits 2×MSL (~2 minutes) before port can be reused.
High-traffic servers use SO_REUSEPORT to avoid port exhaustion.
```

### Head-of-Line Blocking

```
TCP HEAD-OF-LINE BLOCKING PROBLEM:

Requests: [A, B, C, D] sent over one TCP connection

Packet for B is lost:
  A arrives: ✅ delivered
  B arrives: ❌ missing
  C arrives: ✅ received but HELD (waiting for B)
  D arrives: ✅ received but HELD (waiting for B)

  ↳ B retransmitted...
  ↳ B arrives: ✅
  ↳ C and D finally delivered (delayed!)

Impact: One packet loss delays ALL subsequent requests on that connection.
Solution: HTTP/3 uses QUIC (over UDP) — each stream has independent loss recovery.
```

---

## 🚀 TCP Deep Dive

### Flow Control and Congestion Control

```
FLOW CONTROL — prevents sender overwhelming receiver:

Receiver advertises its "receive window" (rwnd) in every ACK:
  ACK, window=65535  → "I have 64KB of buffer space"
  ACK, window=0      → "My buffer is full, STOP sending"

Sender can have at most min(cwnd, rwnd) bytes "in flight" (unACKed).

CONGESTION CONTROL — prevents sender overwhelming the NETWORK:

Slow Start:          cwnd starts at 1 MSS, doubles every RTT until ssthresh
Congestion Avoidance: cwnd += 1 MSS per RTT (linear growth) after ssthresh
Fast Retransmit:     3 duplicate ACKs → retransmit immediately (don't wait for timeout)
Fast Recovery:       After fast retransmit, halve cwnd instead of resetting to 1
```

### TCP Keep-Alive and Connection Pooling

```java
// Configuring TCP keep-alive for database connections (JDBC):
@Bean
public DataSource dataSource() {
    HikariConfig config = new HikariConfig();
    config.setJdbcUrl("jdbc:postgresql://localhost/mydb");
    
    // Connection pool settings:
    config.setMaximumPoolSize(20);            // Max active connections
    config.setMinimumIdle(5);                 // Keep 5 warm connections
    config.setIdleTimeout(600_000);           // Remove idle connections after 10 min
    config.setConnectionTimeout(30_000);      // Wait max 30s for a connection
    config.setKeepaliveTime(60_000);          // TCP keepalive every 60s
    
    // Validate connections before using (detect network drops):
    config.setConnectionTestQuery("SELECT 1");
    
    return new HikariDataSource(config);
}
```

---

## 🌐 HTTP Versions

```
HTTP VERSION TIMELINE:
─────────────────────────────────────────────────────────────────────────────
HTTP/0.9 (1991)   One line: GET /index.html  Response: raw HTML only
HTTP/1.0 (1996)   Request + Response headers. New TCP connection per request!
HTTP/1.1 (1997)   Keep-Alive, Host header, chunked transfer, pipelining (broken)
HTTP/2  (2015)    Binary, multiplexing, server push, header compression (HPACK)
HTTP/3  (2022)    QUIC (UDP), 0-RTT, independent stream loss recovery
```

### HTTP/1.1 Problems

```
HTTP/1.1 "WORKAROUNDS" (signs of a broken protocol):

Problem: One request at a time per connection (pipelining was buggy)
Workaround: Browsers open 6-8 parallel TCP connections per domain (wasteful!)

Problem: Large HTTP headers sent with every request (cookies, user-agent, etc.)
Workaround: Domain sharding (static.cdn.com vs api.mysite.com) to split connections

Problem: Server can't push resources proactively
Workaround: HTTP Long Polling (client polls), JavaScript inlining
```

### HTTP/2 Improvements

```
HTTP/2 MULTIPLEXING:

One TCP connection, multiple streams (requests/responses interleaved):

Connection:  ──────────────────────────────────────────────────────→
Stream 1:        [GET /] ──────────→ [200 HTML ←────────]
Stream 2:           [GET /style.css]─→ [200 CSS ←───]
Stream 3:               [GET /app.js]─────→ [200 JS ←──────────]

All happening simultaneously on ONE TCP connection.
Headers are HPACK-compressed (first request sends full headers, subsequent
requests send only DIFFERENCES → 85-95% reduction in header size).

HTTP/2 LIMITATIONS:
  - TCP head-of-line blocking still applies at the packet level
  - All streams share one TCP stream → one packet loss = all streams stall
```

### HTTP/3 with QUIC

```
HTTP/3 + QUIC:
─────────────────────────────────────────────────────────────────────
Built on UDP, not TCP (QUIC implements reliability at the application layer)

0-RTT Connection Resume:
  First connection: 1-RTT (like TLS 1.3 handshake)
  Subsequent connections: 0-RTT (session ticket from previous session)
  Impact: ~100ms faster reconnection for returning users

Independent Stream Loss Recovery:
  QUIC streams are independent at the packet level
  Packet loss in Stream 1 does NOT block Stream 2 or Stream 3
  (This is the TCP head-of-line blocking fix)

Connection Migration:
  Connection ID is NOT tied to IP:port (unlike TCP)
  Mobile user switches from WiFi to LTE? Same connection continues seamlessly!
  
ADOPTION: 25-30% of web traffic uses HTTP/3 (2024). Google, Facebook, Cloudflare.
```

```java
// Spring Boot: Enable HTTP/2 (supports HTTP/3 via Undertow or Netty):
// application.yml:
server:
  http2:
    enabled: true
  ssl:
    enabled: true  # HTTP/2 requires HTTPS
  # For HTTP/3 (requires Netty or Undertow + ALPN):
  # Add Alt-Svc header to advertise HTTP/3 support
```

---

## 🗄️ DNS

```
DNS RESOLUTION — What happens when you type "api.github.com":
──────────────────────────────────────────────────────────────────────────────
1. Browser DNS Cache: "Have I looked up api.github.com recently?" → Miss
2. OS DNS Cache:      Check /etc/hosts, then OS cache → Miss
3. Recursive Resolver: Query your ISP's or 8.8.8.8 resolver
4. Root Nameservers:   "Who handles .com?" → a.gtld-servers.net
5. TLD Nameservers:    "Who handles github.com?" → ns-520.awsdns-01.net
6. Authoritative NS:   "api.github.com = 140.82.114.6"
7. Cache response for TTL (Time To Live) seconds

TOTAL: 4-8 DNS lookups, 20-120ms for first resolution
CACHED: <1ms for subsequent resolutions (within TTL)
```

```
DNS RECORD TYPES:
─────────────────────────────────────────────────────────────────────
A Record      mysite.com        → 192.168.1.1       (IPv4 address)
AAAA Record   mysite.com        → 2001:db8::1        (IPv6 address)
CNAME         www.mysite.com    → mysite.com         (alias/redirect)
MX Record     mysite.com        → mail.mysite.com    (email server)
TXT Record    mysite.com        → "v=spf1 include:..."(SPF, DKIM verification)
NS Record     mysite.com        → ns1.cloudflare.com (nameservers)
SOA Record    mysite.com        → authority info      (zone info)
SRV Record    _grpc._tcp.svc    → host:port:priority  (service discovery)
```

```
DNS FOR SYSTEM DESIGN — Key TTL tradeoffs:

LOW TTL (30s):  Fast failover (change IP quickly), high DNS query load
HIGH TTL (300s+): Low DNS load, slow to failover

DNS-BASED LOAD BALANCING (Round Robin):
  api.mysite.com → [1.2.3.4, 1.2.3.5, 1.2.3.6]
  Each client gets a different IP in rotation
  
  Problems:
  - TTL caching breaks round-robin (clients cache and reuse one IP)
  - No health checking (returns dead servers)
  - Can't do weighted routing based on server capacity

  Better: Use a load balancer (L4/L7) + single DNS entry, or use Anycast
```

---

## 📦 CDN

```
CDN (Content Delivery Network) ARCHITECTURE:
──────────────────────────────────────────────────────────────────────────────
WITHOUT CDN:                          WITH CDN:
  User in Tokyo                         User in Tokyo
  ────────────────────────            ─────────────────────────
  Request → Sydney origin (150ms)     Request → Tokyo CDN PoP (5ms)
  Response: 150ms × 2 = 300ms         Cache HIT → 10ms total
                                       Cache MISS → 5ms + 150ms origin fetch
                                                → cached for next user

CDN CACHES:
  Static assets: JS, CSS, images → long TTL (1 year with content hash URLs)
  API responses: product catalog, user profiles → short TTL (1-60 min)
  Dynamic content: personalized pages → usually NOT cached (or edge computing)

CDN INVALIDATION:
  Option 1: URL versioning: /app.v1.2.3.js → /app.v1.2.4.js (deploy-triggered)
  Option 2: Explicit purge API: cdn.purge("product:12345") on update
  Option 3: Short TTL (accept stale data) → no invalidation needed
```

---

## ⚖️ Load Balancing

```
L4 vs L7 LOAD BALANCING:
──────────────────────────────────────────────────────────────────────────────
L4 (Transport Layer):                   L7 (Application Layer):
  Operates on TCP/UDP                     Operates on HTTP/gRPC/WebSocket
  Routing by IP:port                      Routing by URL path, headers, cookies
  No TLS termination                      TLS termination here
  Extremely fast (hardware ASICs)         Slower (parse HTTP headers)
  Cannot do sticky sessions by content    Can do sticky by session cookie
  
L4 USE CASES:                           L7 USE CASES:
  TCP proxy (database connections)        HTTP APIs
  UDP game servers                        A/B testing (route by header)
  High-throughput streaming               Blue/Green deployments
                                          Rate limiting (inspect body/path)
```

```
LOAD BALANCING ALGORITHMS:
─────────────────────────────────────────────────────────────────────
Round Robin:       1→2→3→1→2→3  Simple, works when servers are equal
Weighted RR:       1→1→1→2→3→1  Server 1 is 3x stronger → 3x more traffic
Least Connections: Route to server with fewest active connections (best for variable request duration)
IP Hash:           hash(client_ip) % servers → same client → same server (sticky)
Random:            Random server selection (surprisingly good, especially "2 random choices")
Consistent Hash:   Used for caching (route by key → same server = cache locality)
```

---

## 🔗 WebSockets

```
WEBSOCKET CONNECTION LIFECYCLE:
──────────────────────────────────────────────────────────────────────────────
1. HTTP Upgrade Handshake:
   Client → GET /ws HTTP/1.1
             Upgrade: websocket
             Connection: Upgrade
             Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==

   Server → HTTP/1.1 101 Switching Protocols
             Upgrade: websocket
             Connection: Upgrade
             Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=

2. Bidirectional Frame Exchange:
   Client ──[frame: "subscribe:AAPL"]──→ Server
   Server ←─[frame: "price:182.45"]───── Server  (server-initiated push)
   Client ──[frame: ping]───────────────→ Server
   Server ←─[frame: pong]───────────────         (keepalive)

3. Termination:
   Either side sends Close frame → other echoes Close → TCP FIN
```

```java
// Spring Boot WebSocket endpoint:
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(stockTickerHandler(), "/ws/stocks")
                .setAllowedOrigins("https://myapp.com");  // CORS for WebSocket
    }
}

@Component
public class StockTickerHandler extends TextWebSocketHandler {
    private final ConcurrentHashSet<WebSocketSession> sessions = new ConcurrentHashSet<>();
    
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        sessions.add(session);
    }
    
    // Broadcast to all connected clients:
    public void broadcastPrice(String symbol, double price) {
        String msg = "{\"symbol\":\"%s\",\"price\":%.2f}".formatted(symbol, price);
        sessions.removeIf(session -> {
            try {
                session.sendMessage(new TextMessage(msg));
                return false; // Keep in set
            } catch (IOException e) {
                return true; // Remove dead session
            }
        });
    }
    
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        sessions.remove(session);
    }
}
```

---

## ⏱️ Network Latency

```
TYPICAL LATENCY NUMBERS (2024 estimates):
────────────────────────────────────────────────────────────────────────────────
L1 Cache reference              0.5 ns
Branch mispredict               5 ns
L2 Cache reference              7 ns
Mutex lock/unlock               25 ns
Main memory reference           100 ns         = 0.1 µs
Compress 1K with Snappy         3,000 ns       = 3 µs
Send 2K bytes over 1 Gbps net   20,000 ns      = 20 µs    ← this is the NETWORK
SSD random read                 150,000 ns     = 0.15 ms
Read 1 MB sequentially from SSD 1,000,000 ns  = 1 ms
Round trip in same datacenter   500,000 ns     = 0.5 ms
Read 1 MB sequentially from HDD 20,000,000 ns = 20 ms
Send packet CA→Netherlands→CA   150,000,000 ns = 150 ms  ← transcontinental RTT

KEY TAKEAWAYS:
  Memory: nanoseconds
  SSD:    microseconds to milliseconds
  Network (local DC): sub-millisecond
  Network (cross-region): 10-100ms
  Network (cross-continent): 100-300ms
```

```
LATENCY REDUCTION STRATEGIES:
────────────────────────────────────────────────────────────────────────────────
Problem                        Solution
─────────────────────────────────────────────────────────────────────────────
Many small requests            Batch requests (GraphQL, bulk APIs)
Sequential API calls           Parallelize with CompletableFuture / reactor
Large payloads                 Compression (gzip, brotli), pagination
High TCP connection overhead   HTTP/2 (reuse connections), connection pools
Geographic distance            CDN for static, multi-region deployment for APIs
DNS lookup time                DNS prefetch, local resolver, long TTL
TLS handshake latency          TLS session resumption, HTTP/3 0-RTT
Cold start (Lambda/container)  Provisioned concurrency, keep-warm pings
```

---

## 💡 Interview Q&A

**Q1: What happens when you type "google.com" in your browser?**
```
This is the "classic" network interview question. Hit all these points:

1. DNS Resolution:
   - Check browser/OS/router cache for google.com → IP
   - If miss: query recursive resolver → root NS → .com TLD NS → google's NS
   - Returns IP (e.g., 142.250.80.46), cached for TTL seconds

2. TCP 3-Way Handshake:
   - SYN → SYN-ACK → ACK (adds ~1.5 RTT latency)

3. TLS Handshake (HTTPS):
   - ClientHello (supported cipher suites) → ServerHello (chosen cipher, cert)
   - Certificate verification (against CA roots)
   - Key exchange (ECDHE), session keys derived
   - Adds ~1 RTT (TLS 1.3) or 2 RTT (TLS 1.2)

4. HTTP Request:
   - GET / HTTP/2 (if HTTP/2 negotiated via ALPN during TLS)
   - Headers: Host, Accept, Cookie, User-Agent

5. Load Balancer → Server:
   - Google's Maglev L4 LB → Google Front End (L7) → Backend

6. Response:
   - HTML returned, browser parses, fetches sub-resources (CSS/JS/images)
   - Sub-resources served from same connection (HTTP/2) or CDN

Bonus: Mention CDN for static assets, HTTP/3 for mobile, HSTS for HTTPS enforcement
```

**Q2: TCP vs UDP — when would you use UDP in a backend system?**
```
UDP is appropriate when:
1. Loss tolerance exists: video frames (one bad frame = brief glitch, not crash)
2. Latency > reliability: gaming position updates (stale position = teleport, not disaster)
3. Stateless protocols: DNS (small query/response, easy to retry)
4. High-frequency small messages: IoT sensors (10k sensors × 10 readings/sec)
5. Multicast: UDP can fan out to many recipients; TCP cannot

Don't use UDP when:
  - Data must arrive complete (file transfer, financial transactions)
  - Ordering matters and is hard to reconstruct
  - You'd end up reimplementing TCP anyway (just use TCP)

Note: HTTP/3 uses QUIC (over UDP) but QUIC implements reliability — you get UDP's
HOL-blocking benefits without losing reliability.
```

**Q3: How does HTTP/2 multiplexing improve performance over HTTP/1.1?**
```
HTTP/1.1 pipelining was supposed to solve this but was widely disabled due to bugs.
Real HTTP/1.1 behavior: browsers open 6-8 parallel connections per domain.

HTTP/2 multiplexing:
- ONE TCP connection, many concurrent streams (requests/responses)
- Streams are independent at the HTTP level (not at TCP packet level)
- Header compression (HPACK) — 85-95% size reduction for headers
- Server push — send CSS/JS before browser requests it

Performance impact:
- Eliminates connection setup overhead (3-way handshake × N connections)
- Eliminates TLS handshake overhead × N connections
- Enables priority hints (tell server which resources matter most)

Caveat: TCP HOL blocking still exists at packet level — HTTP/3 fixes this.
```

---

## 🎲 Mini Challenge

> 🎲 **CHALLENGE** (5 minutes):
> A mobile app makes 20 separate API calls when a user opens it (home screen, user profile, notifications, ads, product catalog sections...). The app feels slow. Users on 4G are OK but users on 3G (150ms RTT) complain it takes 4+ seconds. How would you solve this?

<details>
<summary>💡 Click to reveal solution</summary>

**Root Cause Analysis:**

```
20 sequential API calls × 2 × 150ms RTT = 6,000ms = 6 seconds
(each call: send request + receive response = 2× RTT minimum)

Even parallel calls:
  HTTP/1.1: 6 connections × ceil(20/6) = ~4 round trips × 300ms = 1.2s + server time
  HTTP/2:   1 connection, all 20 in parallel → 1 round trip = 300ms + server time
```

**Solution Layers:**

**Layer 1 — Network Protocol**: Upgrade to HTTP/2
- All 20 requests in ONE TCP connection simultaneously
- ~300ms instead of 6000ms for the requests alone

**Layer 2 — API Aggregation (Backend for Frontend)**:
```java
// Instead of 20 API calls, 1 call to a BFF:
@GetMapping("/home-screen")
public HomeScreenData getHomeScreen(@AuthenticationPrincipal User user) {
    // Execute all in parallel:
    var userFuture = userService.getProfileAsync(user.getId());
    var notifFuture = notificationService.getUnreadAsync(user.getId());
    var productsFuture = productService.getFeaturedAsync();
    var adsFuture = adService.getHomeAdsAsync(user.getSegment());
    
    return new HomeScreenData(
        userFuture.join(),
        notifFuture.join(),
        productsFuture.join(),
        adsFuture.join()
    );
    // 1 network hop from app + parallel server-side processing
}
```

**Layer 3 — Caching**:
- Cache product catalog (changes hourly) → serve from CDN (5ms vs 150ms)
- Cache user profile locally in app (stale-while-revalidate pattern)
- Delta sync: only send what CHANGED since last fetch (ETags, If-None-Match)

**Layer 4 — Prefetching**:
- On login, pre-warm the home screen cache in the background
- Push notifications/data changes via WebSocket (remove polling)

Result: 6+ seconds → ~300ms (network) + ~50ms (server) = **~350ms**

</details>

---

## 🏢 Industry Applications

| Company | Networking Innovation | Impact |
|---------|----------------------|--------|
| Google | QUIC protocol (became HTTP/3 standard) | 8% fewer retransmits on mobile |
| Facebook | TCP BBR congestion control rollout | 20% throughput improvement on long-distance |
| Netflix | Open Connect CDN (own CDN in ISP DCs) | 95%+ of traffic served from within ISP |
| Cloudflare | Anycast routing for DDoS mitigation | Traffic absorbed at nearest PoP |
| Discord | Moved from HTTP to WebSocket + Protobuf | 50ms → 15ms message delivery |

---

## 🔗 What to Read Next

| Article | Why |
|---------|-----|
| [RESTful APIs](../APIs/RESTful.md) | HTTP in action |
| [gRPC](../APIs/gRPC.md) | HTTP/2 for microservice communication |
| [Consistent Hashing](./Consistent_Hashing.md) | Routing at the application layer |
| [Scalability](./Scalability.md) | How networking limits scale |
| [CAP Theorem](./CAPTheorem.md) | Network partitions and their meaning |

---

*Previous: [← Consistent Hashing](./Consistent_Hashing.md) | Next: [Scalability →](./Scalability.md)*
