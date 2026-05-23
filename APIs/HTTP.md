# 🌐 HTTP: The Language of the Web — A Complete Deep Dive

> *"HTTP is not just a protocol — it's the heartbeat of the internet. Every web page you load, every API you call, every image you see travels over HTTP. Understanding it deeply transforms you from someone who uses the web into someone who can architect it."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium → 🔴 Advanced | **🔗 Prerequisites**: [Networking Fundamentals](../Foundations/Networking/README.md), [RESTful APIs](./RESTful.md)

---

## 📋 Table of Contents

1. [What is HTTP?](#1-what-is-http)
2. [The Evolution: HTTP/0.9 → 1.0 → 1.1 → 2 → 3](#2-the-evolution)
3. [Anatomy of an HTTP Request & Response](#3-anatomy-of-an-http-request--response)
4. [HTTP Methods — The Verbs of the Web](#4-http-methods)
5. [Status Codes — The Server's Reply](#5-status-codes)
6. [HTTP Headers — The Metadata Layer](#6-http-headers)
7. [Connection Management](#7-connection-management)
8. [HTTP/2 Deep Dive — Multiplexing & Binary Framing](#8-http2-deep-dive)
9. [HTTP/3 & QUIC — The UDP Revolution](#9-http3--quic)
10. [Caching — The Performance Superpower](#10-caching)
11. [Security — HTTPS, HSTS, CORS, CSP](#11-security)
12. [Java / Spring Boot Implementation](#12-java--spring-boot-implementation)
13. [Performance Optimization Patterns](#13-performance-optimization-patterns)
14. [Real-World Use Cases & Interesting Projects](#14-real-world-use-cases--interesting-projects)
15. [Common Pitfalls](#15-common-pitfalls)
16. [Interview Q&A — The Questions That Actually Get Asked](#16-interview-qa)
17. [Cheat Sheet](#17-cheat-sheet)

---

## 1. What is HTTP?

**HTTP (HyperText Transfer Protocol)** is an application-layer protocol that defines how clients and servers communicate. It sits on top of TCP (or UDP for HTTP/3) and is the foundation of all data exchange on the Web.

```
THE BIG PICTURE:

 ┌─────────────────────────────────────────────────────────────────┐
 │  APPLICATION LAYER    HTTP, HTTPS, WebSocket, gRPC              │
 │  ─────────────────────────────────────────────────────────────  │
 │  TRANSPORT LAYER      TCP (HTTP/1.1, HTTP/2)  UDP (HTTP/3)      │
 │  ─────────────────────────────────────────────────────────────  │
 │  INTERNET LAYER       IP (packet routing)                       │
 │  ─────────────────────────────────────────────────────────────  │
 │  LINK LAYER           Ethernet, Wi-Fi                           │
 └─────────────────────────────────────────────────────────────────┘

HTTP IS:
 ✅ Stateless         — each request is independent
 ✅ Text-based        — HTTP/1.1 is human-readable
 ✅ Extensible        — headers let you add metadata
 ✅ Request/Response  — client asks, server answers
 
HTTP IS NOT:
 ❌ A transport       — it rides on TCP/UDP
 ❌ Persistent        — by default, no memory between requests
 ❌ Secure            — HTTPS = HTTP + TLS
 ❌ Real-time         — WebSocket is built ON TOP of HTTP
```

### The Request-Response Cycle (Conceptual)

```
 ┌────────────────┐                        ┌────────────────┐
 │   YOUR BROWSER │                        │   WEB SERVER   │
 │   (client)     │                        │   (nginx/etc.) │
 └───────┬────────┘                        └───────┬────────┘
         │                                         │
         │  1. DNS Lookup: google.com → IP         │
         │  2. TCP Connect (3-way handshake)        │
         │  3. TLS Handshake (if HTTPS)             │
         │                                         │
         │  ──── GET /search?q=http HTTP/1.1 ────→ │
         │  ←─── HTTP/1.1 200 OK + HTML body ───── │
         │                                         │
         │  ──── GET /images/logo.png ───────────→ │
         │  ←─── HTTP/1.1 200 OK + PNG bytes ───── │
         │                                         │
         │  (connection closed or kept alive)       │
```

---

## 2. The Evolution

### HTTP/0.9 (1991) — The One-Line Protocol

```
Request:  GET /page.html
Response: <html>...</html>
(connection closes immediately)

LIMITATIONS:
• GET only — no other methods
• No headers — no metadata
• No status codes — no error signaling
• HTML only — no binary data
```

### HTTP/1.0 (1996) — Growing Up

```
Request:
  GET /page.html HTTP/1.0
  User-Agent: NCSA_Mosaic/2.0

Response:
  HTTP/1.0 200 OK
  Content-Type: text/html
  Content-Length: 1234
  
  <html>...</html>

NEW IN 1.0:
  ✅ Versioning (HTTP/1.0 in request line)
  ✅ Status codes (200, 404, 500...)
  ✅ Headers (Content-Type, Content-Length...)
  ✅ Multiple content types (not just HTML)
  ✅ POST method added
  
STILL BROKEN:
  ❌ One request per connection — close & reopen for every resource
  ❌ 10 images on a page = 10 TCP handshakes 💸
```

### HTTP/1.1 (1997) — The Workhorse (Still Everywhere!)

```
KILLER FEATURES:

1. PERSISTENT CONNECTIONS (Keep-Alive):
   One TCP connection → multiple requests → massive speedup
   
   Before (1.0):                  After (1.1):
   TCP connect                    TCP connect
   GET /page    ← round trip      GET /page    ──┐
   TCP close                      GET /style.css  │ all on
   TCP connect                    GET /logo.png   │ one TCP
   GET /style   ← round trip      GET /script.js ─┘
   TCP close                      TCP close (eventually)
   
2. CHUNKED TRANSFER ENCODING:
   Server can start sending before knowing total size
   
   HTTP/1.1 200 OK
   Transfer-Encoding: chunked
   
   5\r\n           ← chunk size in hex
   Hello\r\n       ← chunk data
   6\r\n
    World\r\n
   0\r\n           ← terminal chunk (size=0)
   \r\n
   
3. PIPELINING (rarely used):
   Send multiple requests without waiting for responses
   BUT: head-of-line blocking kills it in practice
   
4. HOST HEADER (mandatory):
   Enables virtual hosting — multiple sites on one IP!
   
   GET /page HTTP/1.1
   Host: www.google.com      ← server knows which site you want

THE HEAD-OF-LINE BLOCKING PROBLEM:
  
  ┌─────┐  req1 ──→  ┌─────┐
  │     │  req2 ──→  │     │  req1 takes 2 seconds (slow DB query)
  │     │  req3 ──→  │     │  req2 and req3 WAIT, even if ready
  │     │  ←── resp1 │     │  This is head-of-line blocking!
  │     │  ←── resp2 │     │
  │     │  ←── resp3 │     │
  └─────┘             └─────┘
  
  Browser workaround: open 6 TCP connections per domain
  (6 parallel "lanes" = 6x more traffic)
```

### HTTP/2 (2015) — The Multiplexing Revolution

```
BINARY FRAMING LAYER:
  HTTP/1.1 is text-based:      HTTP/2 is binary:
  "GET /page HTTP/1.1\r\n"     [ HEADERS frame  | len=20 | stream=1 ]
  "Host: example.com\r\n"      [ DATA frame      | len=0  | stream=1 ]
  "\r\n"
  
  Binary = faster to parse, less error-prone, more compact

MULTIPLEXING — The Game Changer:
  
  HTTP/1.1 (one at a time per connection):
  ──[req1]──────────────────[resp1]──
  ──────────────[req2]────────────────────────[resp2]──
  
  HTTP/2 (all at once, one connection):
  ──[req1]─────────────────────────[resp1]──────────────
  ──[req2]──────────[resp2]─────────────────────────────
  ──[req3]────────────────────────────────[resp3]────────
  ──[req4]──[resp4]─────────────────────────────────────
       all interleaved on ONE TCP connection!
  
HEADER COMPRESSION (HPACK):
  HTTP/1.1 repeats headers on every request (wasteful!):
  "User-Agent: Mozilla/5.0 (same 60 bytes every request)"
  
  HPACK uses a shared header table + Huffman coding:
  First request: sends full headers, adds to table
  Later requests: sends index number instead of full string
  Result: 85-88% header compression
  
SERVER PUSH:
  Server sends resources before client asks
  
  Client: GET /page
  Server: [200 OK for /page] 
        + [PUSH /style.css — you'll need this!]
        + [PUSH /script.js — you'll need this too!]
```

### HTTP/3 (2022) — UDP Takes Over

```
THE PROBLEM WITH HTTP/2 ON TCP:
  HTTP/2 solved application-layer head-of-line blocking
  BUT: TCP still has transport-layer head-of-line blocking
  
  TCP packet lost? ALL HTTP/2 streams wait for retransmit!
  (TCP doesn't know about HTTP/2 streams — it just sees bytes)

HTTP/3 SOLUTION: Replace TCP with QUIC (over UDP)

 HTTP/1.1  → TCP  → IP  → Ethernet
 HTTP/2    → TCP  → IP  → Ethernet
 HTTP/3    → QUIC → UDP → IP → Ethernet  ← different transport!

QUIC SUPERPOWERS:
  ┌──────────────────────────────────────────────────────────┐
  │  QUIC = HTTP/2 multiplexing + TLS 1.3 + UDP transport   │
  └──────────────────────────────────────────────────────────┘
  
  1. 0-RTT connection for repeat visits:
     TCP + TLS 1.3: 1 RTT handshake
     QUIC (repeat):  0 RTT — sends data in first packet!
  
  2. No transport-layer head-of-line blocking:
     Stream 1 packet lost → only Stream 1 waits
     Streams 2,3,4 continue unaffected
  
  3. Connection migration:
     Switch from Wi-Fi → LTE?
     HTTP/1.1/2: connection breaks (IP address changed)
     HTTP/3: connection CONTINUES (QUIC uses Connection ID, not IP)
  
  4. Built-in encryption:
     QUIC mandates TLS 1.3 — unencrypted HTTP/3 doesn't exist

ADOPTION:
  Google: all properties since 2019
  Facebook/Meta: 75%+ of traffic on QUIC
  YouTube: significant QUIC usage
  Cloudflare: default for all sites
```

### Evolution Summary

```
VERSION  YEAR  TRANSPORT  KEY FEATURE                    STILL USED?
──────────────────────────────────────────────────────────────────────
0.9      1991  TCP        GET only, no headers           ❌ Dead
1.0      1996  TCP        Status codes, headers, POST    ⚠️ Rare
1.1      1997  TCP        Keep-alive, chunked, Host      ✅ Everywhere
2        2015  TCP        Multiplexing, binary, HPACK    ✅ ~65% of sites
3        2022  UDP/QUIC   0-RTT, no HOL blocking         ✅ Growing fast
```

---

## 3. Anatomy of an HTTP Request & Response

### Request Structure

```
REQUEST LINE:    [METHOD] [REQUEST-TARGET] [HTTP-VERSION]
                  GET      /search?q=http   HTTP/1.1

HEADERS:         [Field-Name]: [Field-Value]
                  Host: www.example.com
                  User-Agent: Mozilla/5.0 ...
                  Accept: text/html,application/xhtml+xml
                  Accept-Encoding: gzip, deflate, br
                  Accept-Language: en-US,en;q=0.9
                  Cookie: session=abc123; pref=dark
                  Connection: keep-alive
                  Cache-Control: no-cache

BLANK LINE:      \r\n  (separates headers from body)

BODY:            (only for POST, PUT, PATCH)
                  {"username": "alice", "password": "..."}

COMPLETE EXAMPLE — POST request:
─────────────────────────────────────────────────────────
POST /api/orders HTTP/1.1\r\n
Host: api.shop.com\r\n
Content-Type: application/json\r\n
Content-Length: 47\r\n
Authorization: Bearer eyJhbGci...\r\n
Accept: application/json\r\n
\r\n
{"productId":"SKU-123","quantity":2,"note":"gift"}
─────────────────────────────────────────────────────────
```

### Response Structure

```
STATUS LINE:     [HTTP-VERSION] [STATUS-CODE] [REASON-PHRASE]
                  HTTP/1.1       200            OK

HEADERS:         Content-Type: application/json
                  Content-Length: 238
                  Content-Encoding: gzip
                  Cache-Control: no-store
                  X-Request-Id: a1b2c3d4
                  Set-Cookie: session=xyz; HttpOnly; Secure; SameSite=Strict
                  Strict-Transport-Security: max-age=31536000
                  Access-Control-Allow-Origin: https://shop.com

BLANK LINE:      \r\n

BODY:            {"orderId":"ORD-789","status":"confirmed","total":49.99}

COMPLETE EXAMPLE:
─────────────────────────────────────────────────────────
HTTP/1.1 201 Created\r\n
Content-Type: application/json; charset=utf-8\r\n
Content-Length: 58\r\n
Location: /api/orders/ORD-789\r\n
X-Request-Id: 7f3a2d1e\r\n
\r\n
{"orderId":"ORD-789","status":"confirmed","total":49.99}
─────────────────────────────────────────────────────────
```

### URL Anatomy

```
https://api.example.com:8443/v2/users/123/orders?status=pending&limit=10#section2
│      │                │    │   │     │   │      │                        │
│      │                │    │   │     │   │      │                        └── Fragment (client-only)
│      │                │    │   │     │   │      └── Query String
│      │                │    │   │     │   └── Resource
│      │                │    │   │     └── Resource ID
│      │                │    │   └── Collection
│      │                │    └── API Version (Path-based versioning)
│      │                └── Port (optional, defaults: 80=HTTP, 443=HTTPS)
│      └── Host (domain name → DNS → IP address)
└── Scheme (protocol)
```

---

## 4. HTTP Methods

### The Complete Method Reference

```
METHOD   SAFE?  IDEMPOTENT?  HAS BODY?  USE CASE
──────────────────────────────────────────────────────────────────────────
GET      ✅     ✅           ❌         Retrieve resource
HEAD     ✅     ✅           ❌         Check resource metadata (no body)
POST     ❌     ❌           ✅         Create resource / trigger action
PUT      ❌     ✅           ✅         Replace full resource (create or update)
PATCH    ❌     ❌*          ✅         Partial update
DELETE   ❌     ✅           optional   Remove resource
OPTIONS  ✅     ✅           ❌         Discover allowed methods (CORS preflight)
TRACE    ✅     ✅           ❌         Debug: echo request back (disable in prod!)
CONNECT  ❌     ❌           ❌         Tunnel for HTTPS through proxy

SAFE    = No side effects (doesn't modify server state)
IDEMPOTENT = Multiple identical requests = same result as one
*PATCH can be idempotent if designed carefully
```

### Deep Dive: PUT vs POST vs PATCH

```
PUT — Replace Entire Resource:
  PUT /api/users/123
  Body: {"name":"Alice","email":"alice@new.com","age":30,"role":"admin"}
  
  IDEMPOTENT: Call it 10 times → same result as calling once
  ALL fields required (missing fields are nulled/removed)
  Used by: S3 (upload object replaces completely)

POST — Create or Non-Idempotent Action:
  POST /api/users
  Body: {"name":"Alice","email":"alice@example.com"}
  
  NOT IDEMPOTENT: Call 10 times → 10 user records created!
  Server assigns the ID (response: 201 Created + Location: /api/users/123)
  Also used for actions: POST /api/orders/123/cancel

PATCH — Partial Update:
  PATCH /api/users/123
  Body: {"email":"alice@new.com"}   ← only what changed
  
  Only specified fields are updated (name, age, role unchanged)
  More efficient than PUT for large resources
  Standard: JSON Patch (RFC 6902) or JSON Merge Patch (RFC 7396)

JSON PATCH EXAMPLE (RFC 6902):
  PATCH /api/users/123
  Content-Type: application/json-patch+json
  
  [
    {"op": "replace", "path": "/email", "value": "alice@new.com"},
    {"op": "add",     "path": "/tags/0", "value": "premium"},
    {"op": "remove",  "path": "/tempToken"}
  ]
```

---

## 5. Status Codes

### Complete Reference with Real-World Context

```
1xx — INFORMATIONAL (rarely seen, mostly internal)
────────────────────────────────────────────────────
100 Continue       — Server says "send the request body" (used with Expect header)
101 Switching Protocols — WebSocket upgrade acknowledgment
102 Processing     — WebDAV: request received, still processing (long operations)

2xx — SUCCESS
────────────────────────────────────────────────────
200 OK             — Standard success (GET, PUT, PATCH responses)
201 Created        — POST that creates a resource (include Location header!)
202 Accepted       — Async: request queued, will process later
204 No Content     — Success but no body (DELETE, some PUT/PATCH)
206 Partial Content — Range request (video seeking, resumable downloads)

3xx — REDIRECTION
────────────────────────────────────────────────────
301 Moved Permanently  — URL changed forever. Browser caches → next request goes directly.
                          SEO: link equity transfers. Use when changing domain/URL structure.
302 Found (temporary)  — Temporary redirect. Browser doesn't cache.
                          Use for: login redirect, A/B testing, maintenance page
303 See Other          — After POST, redirect to GET (Post-Redirect-Get pattern)
304 Not Modified       — Caching: your cached copy is still fresh, use it
307 Temporary Redirect — Like 302 but MUST preserve method (POST stays POST)
308 Permanent Redirect — Like 301 but MUST preserve method

RULE: 301/308 = permanent (cached) | 302/303/307 = temporary (not cached)
RULE: 301/302 = may change method | 307/308 = preserve method always

4xx — CLIENT ERRORS (client did something wrong)
────────────────────────────────────────────────────
400 Bad Request        — Malformed syntax, invalid params, validation failure
401 Unauthorized       — Not authenticated (no token or bad token)
                          Name is misleading — should be "Unauthenticated"
403 Forbidden          — Authenticated but NOT authorized (no permission)
404 Not Found          — Resource doesn't exist (or hide it for security)
405 Method Not Allowed — Used wrong HTTP method (DELETE on read-only resource)
408 Request Timeout    — Server gave up waiting for request
409 Conflict           — State conflict (duplicate email, optimistic lock fail)
410 Gone               — Resource existed but permanently deleted (vs 404)
413 Payload Too Large  — Request body exceeds limit
415 Unsupported Media  — Wrong Content-Type (XML when JSON expected)
422 Unprocessable Entity — Syntax valid but semantics fail (invalid JSON values)
429 Too Many Requests  — Rate limit exceeded (include Retry-After header!)

5xx — SERVER ERRORS (our fault)
────────────────────────────────────────────────────
500 Internal Server Error — Generic unhandled exception (check logs!)
501 Not Implemented    — Server doesn't support the method
502 Bad Gateway        — Proxy received invalid response from upstream
503 Service Unavailable — Server down for maintenance or overloaded (Retry-After!)
504 Gateway Timeout    — Proxy timed out waiting for upstream
```

### Status Code Decision Flow

```
                      ┌─────────────────────┐
                      │  What happened?      │
                      └──────────┬──────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            ↓                    ↓                    ↓
    Request problem        Server problem         Success
    (4xx)                  (5xx)                  (2xx/3xx)
            │                    │                    │
     ┌──────┴──────┐      ┌──────┴──────┐    ┌───────┴───────┐
     ↓             ↓      ↓             ↓    ↓               ↓
 Not found?    Auth?   Upstream?  Unhandled  Created?    Redirect?
   (404)    401/403    502/504      (500)     (201)       (301/302)
```

---

## 6. HTTP Headers

### Request Headers Explained

```java
// Spring Boot — Reading request headers
@GetMapping("/api/data")
public ResponseEntity<Data> getData(
        @RequestHeader("Authorization") String auth,
        @RequestHeader(value = "Accept-Language", defaultValue = "en") String lang,
        @RequestHeader("X-Request-ID") String requestId,
        HttpServletRequest request) {
    
    // Access all headers
    Enumeration<String> headerNames = request.getHeaderNames();
    // ...
}

/*
COMMON REQUEST HEADERS:

Host:                 example.com                  — required in HTTP/1.1
Accept:               application/json             — preferred response format
Accept-Encoding:      gzip, deflate, br            — compression support
Accept-Language:      en-US,en;q=0.9,fr;q=0.8     — language preference (q = quality)
Authorization:        Bearer eyJhbGci...            — authentication token
Cookie:               sessionId=abc; pref=dark      — sent back to origin
Cache-Control:        no-cache, no-store            — caching instructions
If-None-Match:        "abc123"                      — conditional GET (ETag)
If-Modified-Since:    Mon, 20 Jan 2025 12:00:00 GMT — conditional GET (date)
Content-Type:         application/json              — body format (with POST/PUT)
Content-Length:       47                            — body size in bytes
Origin:               https://app.example.com       — CORS origin
Referer:              https://google.com/search     — previous page (note: misspelled!)
User-Agent:           Mozilla/5.0 (...)             — client software
X-Forwarded-For:      203.0.113.1, 198.51.100.1    — original IP through proxies
X-Request-ID:         7f3a2d1e-...                  — request correlation ID
Idempotency-Key:      a1b2c3d4-unique               — safe retry for POST
*/
```

### Response Headers Explained

```java
// Spring Boot — Setting response headers
@GetMapping("/api/data")
public ResponseEntity<Data> getData() {
    return ResponseEntity.ok()
        .header("Cache-Control", "public, max-age=3600")
        .header("ETag", "\"abc123\"")
        .header("X-Request-Id", UUID.randomUUID().toString())
        .header("X-RateLimit-Limit", "1000")
        .header("X-RateLimit-Remaining", "998")
        .header("X-RateLimit-Reset", "1621234567")
        .body(data);
}

/*
COMMON RESPONSE HEADERS:

Content-Type:                application/json; charset=utf-8
Content-Length:              238
Content-Encoding:            gzip                   — body is compressed
Cache-Control:               max-age=3600, public
ETag:                        "abc123def"             — resource fingerprint
Last-Modified:               Mon, 20 Jan 2025 ...
Location:                    /api/orders/ORD-789     — with 201 or 301/302
Set-Cookie:                  session=xyz; HttpOnly; Secure; SameSite=Strict
Strict-Transport-Security:   max-age=31536000        — force HTTPS (HSTS)
Access-Control-Allow-Origin: https://app.example.com — CORS
Content-Security-Policy:     default-src 'self'      — XSS mitigation
X-Content-Type-Options:      nosniff                 — MIME sniffing protection
X-Frame-Options:             DENY                    — clickjacking protection
Retry-After:                 120                     — with 429 or 503
*/
```

---

## 7. Connection Management

### HTTP/1.1 Keep-Alive vs Close

```
WITHOUT Keep-Alive (HTTP/1.0 style):
  ──TCP handshake──  req1  resp1  ──TCP close──
  ──TCP handshake──  req2  resp2  ──TCP close──
  ──TCP handshake──  req3  resp3  ──TCP close──
  Cost: 3 TCP handshakes × ~10ms = 30ms wasted

WITH Keep-Alive (HTTP/1.1 default):
  ──TCP handshake──  req1  resp1  req2  resp2  req3  resp3  ──TCP close──
  Cost: 1 TCP handshake × 10ms = 10ms total
  
  Connection: keep-alive        ← client requests
  Keep-Alive: timeout=5, max=100  ← server: idle timeout 5s, max 100 requests
  Connection: close             ← either side can close
  
NGINX CONFIGURATION:
  keepalive_timeout 65;       # close idle connections after 65s
  keepalive_requests 100;     # close connection after 100 requests
```

### HTTP/1.1 Pipelining (The Broken Promise)

```
THE IDEA (good on paper):
  Send all requests without waiting for responses:
  req1 → req2 → req3 ─────→ ← resp1 ← resp2 ← resp3
  
  vs sequential: req1 ──← resp1 ── req2 ──← resp2 ──
  
THE REALITY (broken):
  1. Head-of-line blocking: resp1 slow → resp2, resp3 wait
  2. Proxy incompatibilities: many proxies don't support it
  3. Virtually disabled in all browsers
  
  HTTP/2 SOLVES this properly with multiplexing.
```

---

## 8. HTTP/2 Deep Dive

### Binary Framing Layer

```
HTTP/2 communication unit = FRAME

FRAME STRUCTURE:
  ┌─────────────────────────────────────────────────┐
  │  Length (24 bits) │ Type (8 bits) │ Flags (8 bits) │
  │  Stream Identifier (31 bits, R flag 1 bit)        │
  │  Frame Payload (0 to 2^24 - 1 bytes)              │
  └─────────────────────────────────────────────────┘
  
FRAME TYPES:
  DATA         — request/response body
  HEADERS      — HTTP headers (compressed with HPACK)
  PRIORITY     — stream priority
  RST_STREAM   — cancel a stream
  SETTINGS     — connection parameters
  PUSH_PROMISE — server push announcement
  PING         — connection health check
  GOAWAY       — graceful connection shutdown
  WINDOW_UPDATE — flow control
  CONTINUATION — large header block continuation
```

### Streams, Messages, Frames

```
Connection = 1 TCP connection
  Stream 1 = 1 request-response pair (odd numbers: client, even: server push)
    Message = complete HTTP request or response
      Frame = smallest unit of communication
  Stream 3 = another request-response (concurrently!)
  Stream 5 = another request-response (concurrently!)

VISUAL:

TCP Connection:
  ─────────────────────────────────────────────────────────────────
  [HEADERS S1] [HEADERS S3] [DATA S1] [HEADERS S5] [DATA S3] [DATA S5]
  ─────────────────────────────────────────────────────────────────
         │            │          │           │           │        │
    Stream 1    Stream 3   Stream 1(body)  Stream 5  Stream3(body) Stream5(body)
  
  All interleaved! No head-of-line blocking at application level.

STREAM STATES:
  idle → open → half-closed (local) → half-closed (remote) → closed
```

### Server Push

```
SEQUENCE (with HTTP/2 Server Push):

  Client ──── GET /page.html ──────────────────────────────→ Server
         ←─── PUSH_PROMISE /style.css (stream 2) ──────────
         ←─── PUSH_PROMISE /app.js (stream 4) ─────────────
         ←─── HEADERS + DATA for /page.html ────────────────
         ←─── DATA for /style.css (stream 2) ────────────────
         ←─── DATA for /app.js (stream 4) ─────────────────

  vs WITHOUT Server Push:
  Client ──── GET /page.html ──────────────────────────────→ Server
         ←─── HTML body (references /style.css, /app.js) ──
         ──── GET /style.css ───────────────────────────────→
         ←─── CSS body ──────────────────────────────────────
         ──── GET /app.js ──────────────────────────────────→
         ←─── JS body ───────────────────────────────────────
  
  Server Push eliminates 2 extra round trips.
  
  SPRING BOOT — Server Push:
  
  @GetMapping("/page")
  public String page(HttpServletRequest request) {
      PushBuilder pushBuilder = request.newPushBuilder();
      if (pushBuilder != null) {
          pushBuilder.path("/css/style.css").push();
          pushBuilder.path("/js/app.js").push();
      }
      return "page";
  }
```

---

## 9. HTTP/3 & QUIC

### QUIC Deep Dive

```
QUIC (Quick UDP Internet Connections) — developed by Google, standardized as RFC 9000

QUIC ARCHITECTURE:
  ┌─────────────────────────────────────────────────────────┐
  │  APPLICATION:  HTTP/3                                   │
  │  ─────────────────────────────────────────────────────  │
  │  QUIC STREAMS: Multiplexed, independent, encrypted      │
  │  ─────────────────────────────────────────────────────  │
  │  QUIC TRANSPORT: Congestion control, flow control       │
  │  ─────────────────────────────────────────────────────  │
  │  TLS 1.3 (integrated, not separate layer)               │
  │  ─────────────────────────────────────────────────────  │
  │  UDP                                                    │
  └─────────────────────────────────────────────────────────┘

0-RTT CONNECTION:
  FIRST VISIT (1 RTT):
  Client ─── Initial (ClientHello + QUIC params) ──→ Server
  Client ←── ServerHello + Certificate + Finished ── Server
  Client ─── Finished + [HTTP/3 Request] ──────────→ Server
  Client ←── [HTTP/3 Response] ────────────────────── Server
  Total: 1 RTT before data (vs 2-3 for HTTP/1.1 + TLS)
  
  REPEAT VISIT (0 RTT):
  Client ─── 0-RTT data + Early Data ──────────────→ Server
             (sends cached session ticket from last visit)
  Server ─── Response immediately ─────────────────→ Client
  Total: 0 additional RTT (data in very first packet!)
  
  ⚠️ 0-RTT is vulnerable to replay attacks for non-idempotent requests
     Use only for safe/idempotent (GET) — protect POST behind 1-RTT

CONNECTION MIGRATION:
  Scenario: User on WiFi → switches to 4G
  
  HTTP/2 (TCP):   WiFi IP changes → TCP connection breaks → reconnect
  HTTP/3 (QUIC):  Connection identified by 64-bit Connection ID (not IP:port)
                   IP changes → same Connection ID → connection continues!
  
  This is huge for mobile users who constantly switch networks.
```

---

## 10. Caching

### The Caching Model

```
THE GOAL: Avoid sending the same data twice when it hasn't changed.

CACHE TYPES:
  Browser Cache    — per-user, fastest, private
  Shared Proxy     — ISP/corporate proxy, shared between users
  CDN Cache        — geographically distributed, shared
  Origin Cache     — server-side (Redis, Varnish)

CACHE-CONTROL DIRECTIVE SYSTEM:
  
  ┌─────────────────────────────────────────────────────────────────┐
  │  public     — can be cached by anyone (CDN, proxy, browser)    │
  │  private    — only browser cache (not CDN — user-specific data) │
  │  no-cache   — revalidate before using (ETag/Last-Modified check)│
  │  no-store   — NEVER cache (payments, medical records)           │
  │  max-age=N  — fresh for N seconds from response time           │
  │  s-maxage=N — CDN/proxy max-age (overrides max-age for CDN)    │
  │  must-revalidate — MUST check server when stale                 │
  │  immutable  — content will NEVER change (hashed assets!)        │
  │  stale-while-revalidate — serve stale, refresh in background   │
  └─────────────────────────────────────────────────────────────────┘

PRACTICAL EXAMPLES:
  CSS/JS with hash: Cache-Control: public, max-age=31536000, immutable
  HTML page:        Cache-Control: no-cache  (always revalidate)
  API user data:    Cache-Control: private, max-age=60
  Payment page:     Cache-Control: no-store  (never cache!)
  Public API data:  Cache-Control: public, max-age=300, s-maxage=3600
```

### Conditional Requests (Validation Caching)

```
FLOW WITH ETAG:

  First Request:
  Client ──── GET /api/products/123 ─────────────────────→ Server
  Client ←─── 200 OK + ETag: "v2.1" + body (full data) ─── Server
              (browser stores: resource + ETag + timestamp)

  Next Request (browser has cached copy):
  Client ──── GET /api/products/123 ─────────────────────→ Server
              If-None-Match: "v2.1"    ← "is this still current?"
  
  IF NOT CHANGED:
  Client ←─── 304 Not Modified (NO BODY!) ───────────────── Server
              Browser uses cached copy → saved bandwidth!
  
  IF CHANGED:
  Client ←─── 200 OK + ETag: "v2.2" + new body ─────────── Server

JAVA IMPLEMENTATION:
```

```java
@RestController
public class ProductController {
    
    @GetMapping("/api/products/{id}")
    public ResponseEntity<Product> getProduct(
            @PathVariable Long id,
            @RequestHeader(value = "If-None-Match", required = false) String ifNoneMatch) {
        
        Product product = productService.findById(id);
        String currentETag = "\"" + product.getVersion() + "\"";
        
        // ETag matches — resource unchanged
        if (currentETag.equals(ifNoneMatch)) {
            return ResponseEntity.status(HttpStatus.NOT_MODIFIED)
                                 .eTag(currentETag)
                                 .build();
        }
        
        return ResponseEntity.ok()
                .eTag(currentETag)
                .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS).cachePublic())
                .lastModified(product.getUpdatedAt())
                .body(product);
    }
}
```

### Stale-While-Revalidate (Modern Pattern)

```
Cache-Control: max-age=60, stale-while-revalidate=300

BEHAVIOUR:
  t=0:    Cache miss → fetch from server → cache it (fresh for 60s)
  t=30:   Cache hit! (fresh) → serve immediately
  t=70:   Cache stale → serve stale copy IMMEDIATELY (user sees fast response)
          + trigger background revalidation
  t=71:   Background fetch completes → cache updated
  t=75:   Next request gets fresh copy

  vs normal max-age=60:
  t=70:   Cache stale → WAIT for server response → user sees slow response
  
  PERFECT FOR: News feeds, product listings, any data that can tolerate
               seconds-old data. Huge latency improvement!
```

---

## 11. Security

### HTTPS = HTTP + TLS

```
TLS HANDSHAKE (TLS 1.3, simplified):

  Client                                           Server
    │                                                │
    │── ClientHello ──────────────────────────────→ │
    │   (TLS version, cipher suites, random1,        │
    │    supported groups, key_share)                │
    │                                                │
    │ ←─ ServerHello + EncryptedExtensions ──────── │
    │    + Certificate + CertificateVerify           │
    │    + Finished (encrypted)                      │
    │    (random2, key_share, chosen cipher)         │
    │                                                │
    │── Finished ──────────────────────────────────→ │
    │   [Application Data] ────────────────────────→ │  ← starts here!
    │                                                │
  1 RTT total for new connections, 0-RTT for resumption
```

### CORS (Cross-Origin Resource Sharing)

```
THE PROBLEM:
  Your frontend (https://app.example.com) calls API (https://api.example.com)
  
  SAME ORIGIN: same scheme + host + port
  CROSS ORIGIN: different scheme, host, OR port → CORS kicks in
  
  Browser security: "I won't send this request without permission"

SIMPLE REQUESTS (GET, POST with simple headers):
  Browser automatically adds:
  Origin: https://app.example.com
  
  Server must respond with:
  Access-Control-Allow-Origin: https://app.example.com  (or *)
  
  Without this → browser blocks the response (request DID go through!)

PREFLIGHT (for complex requests — PUT, DELETE, custom headers):
  Browser first sends OPTIONS preflight:
  
  OPTIONS /api/orders HTTP/1.1
  Origin: https://app.example.com
  Access-Control-Request-Method: DELETE
  Access-Control-Request-Headers: X-Custom-Header
  
  Server must respond:
  Access-Control-Allow-Origin: https://app.example.com
  Access-Control-Allow-Methods: GET, POST, PUT, DELETE
  Access-Control-Allow-Headers: X-Custom-Header
  Access-Control-Max-Age: 86400  ← cache preflight for 24h
  
  Then browser sends the real request.
```

```java
// Spring Boot CORS configuration
@Configuration
public class CorsConfig {
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://app.example.com")); // NEVER use * in production with credentials
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Request-ID"));
        config.setAllowCredentials(true);  // required for cookies/auth headers
        config.setMaxAge(86400L);          // cache preflight 24h
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### Security Headers

```java
// Spring Security — essential security headers
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            // Prevent MIME type sniffing
            .contentTypeOptions(Customizer.withDefaults())
            // Prevent clickjacking
            .frameOptions(frames -> frames.deny())
            // Force HTTPS for 1 year
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)
                .includeSubDomains(true))
            // Control which resources can be loaded
            .contentSecurityPolicy(csp -> csp
                .policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self' https://trusted.cdn.com; " +
                    "img-src 'self' data: https:; " +
                    "style-src 'self' 'unsafe-inline'"))
        );
        return http.build();
    }
}

/*
RESULTING SECURITY HEADERS:
  Strict-Transport-Security: max-age=31536000; includeSubDomains
  X-Content-Type-Options: nosniff
  X-Frame-Options: DENY
  Content-Security-Policy: default-src 'self'; script-src 'self' https://trusted.cdn.com; ...
  Cache-Control: no-cache, no-store (for auth pages)
  X-XSS-Protection: 0  (deprecated — use CSP instead)
*/
```

---

## 12. Java / Spring Boot Implementation

### Complete HTTP Client (WebClient)

```java
@Service
public class ProductApiClient {
    
    private final WebClient webClient;
    
    public ProductApiClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("https://api.external.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .codecs(config -> config.defaultCodecs().maxInMemorySize(16 * 1024 * 1024)) // 16MB
            .build();
    }
    
    // GET with timeout, retry, error handling
    public Mono<Product> getProduct(String id) {
        return webClient
            .get()
            .uri("/products/{id}", id)
            .header("X-Request-ID", UUID.randomUUID().toString())
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response ->
                response.bodyToMono(ErrorResponse.class)
                    .flatMap(err -> Mono.error(new ClientException(err.getMessage()))))
            .onStatus(HttpStatus::is5xxServerError, response ->
                Mono.error(new ServerException("Upstream server error")))
            .bodyToMono(Product.class)
            .timeout(Duration.ofSeconds(3))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                .filter(ex -> ex instanceof ServerException));
    }
    
    // POST with body
    public Mono<Order> createOrder(CreateOrderRequest request) {
        return webClient
            .post()
            .uri("/orders")
            .header("Idempotency-Key", request.getIdempotencyKey()) // safe retry
            .bodyValue(request)
            .retrieve()
            .bodyToMono(Order.class);
    }
    
    // Conditional GET with ETag
    public Mono<ResponseEntity<Catalog>> getCatalog(String cachedETag) {
        return webClient
            .get()
            .uri("/catalog")
            .header(HttpHeaders.IF_NONE_MATCH, cachedETag)
            .retrieve()
            .toEntity(Catalog.class);
        // Returns 304 with null body if not modified
        // Caller checks: if (response.getStatusCode() == NOT_MODIFIED) use cache
    }
}
```

### HTTP/2 Configuration in Spring Boot

```yaml
# application.yml
server:
  http2:
    enabled: true          # Enable HTTP/2
  ssl:
    enabled: true          # HTTP/2 requires HTTPS in browsers
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12

# application-http2.yml (with Undertow for H2C/cleartext)
server:
  undertow:
    threads:
      worker: 200
      io: 4
```

```java
// RestTemplate with HTTP/2 (via OkHttp)
@Configuration
public class HttpClientConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        OkHttpClient okHttpClient = new OkHttpClient.Builder()
            .protocols(List.of(Protocol.H2, Protocol.HTTP_1_1)) // prefer H2
            .connectTimeout(5, TimeUnit.SECONDS)
            .readTimeout(10, TimeUnit.SECONDS)
            .connectionPool(new ConnectionPool(10, 5, TimeUnit.MINUTES))
            .addInterceptor(new LoggingInterceptor())
            .build();
        
        ClientHttpRequestFactory factory = new OkHttp3ClientHttpRequestFactory(okHttpClient);
        return new RestTemplate(factory);
    }
}
```

### Custom Filter for Request Logging

```java
@Component
@Order(1)
public class RequestLoggingFilter implements Filter {
    private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) resp;
        
        String requestId = Optional.ofNullable(request.getHeader("X-Request-ID"))
                                   .orElse(UUID.randomUUID().toString());
        
        response.setHeader("X-Request-ID", requestId);
        MDC.put("requestId", requestId);
        
        long start = System.currentTimeMillis();
        try {
            chain.doFilter(req, resp);
        } finally {
            long elapsed = System.currentTimeMillis() - start;
            log.info("{} {} → {} ({}ms) [{}]",
                request.getMethod(), request.getRequestURI(),
                response.getStatus(), elapsed, requestId);
            MDC.clear();
        }
    }
}
```

---

## 13. Performance Optimization Patterns

### Connection Pooling (Critical for Microservices!)

```
WITHOUT pooling: each request = new TCP + TLS handshake = 10-50ms overhead
WITH pooling: reuse existing connections = ~0ms overhead

                    WITHOUT POOL               WITH POOL
  100 req/sec:      100 × 30ms handshake      30ms once + 0ms × 100
                  = 3000ms wasted/sec         = 30ms wasted/sec
  
  100x improvement for I/O bound services!

SPRING BOOT CONNECTION POOL TUNING:
```

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20      # max connections in pool
      minimum-idle: 5            # keep 5 warm
      idle-timeout: 600000       # remove idle after 10 min
      connection-timeout: 30000  # fail fast if pool exhausted

# For HTTP client pools:
http:
  client:
    max-connections: 500         # total connections
    max-connections-per-route: 100  # per host
    connection-ttl: 300s         # rotate old connections
```

### Response Compression

```java
// Spring Boot — Enable gzip/brotli compression
// application.yml:
// server:
//   compression:
//     enabled: true
//     min-response-size: 1024  # only compress if > 1KB
//     mime-types: application/json, text/html, text/css, application/javascript

@GetMapping("/api/large-dataset")
public ResponseEntity<List<Record>> getLargeDataset() {
    List<Record> data = fetchMillionRecords();
    
    return ResponseEntity.ok()
        .header(HttpHeaders.VARY, "Accept-Encoding") // important for CDN caching
        .body(data);
    // Spring auto-compresses based on Accept-Encoding header
    // JSON: ~5-10x compression, HTML: ~3-5x compression
}
```

---

## 14. Real-World Use Cases & Interesting Projects

### Use Cases by Industry

| Industry | HTTP Pattern | Key Headers/Features |
|---|---|---|
| E-Commerce | REST + Cache | ETag, Cache-Control, CDN |
| Banking | REST + no caching | no-store, HSTS, CSP |
| Video Platform | Range requests, CDN | Content-Range, Accept-Ranges |
| Search Engine | HTTP/2 Multiplexing | Server Push for assets |
| IoT Dashboard | SSE or long-poll | Transfer-Encoding: chunked |
| Social Media | HTTP/2 + H3 | 0-RTT, connection reuse |

### 🚀 Interesting Project Concepts

#### Project 1: HTTP/2 vs HTTP/1.1 Performance Visualizer
```
BUILD: A web app that loads the same 50 resources twice:
  Tab 1: HTTP/1.1 (sequential, 6 connections limit)
  Tab 2: HTTP/2 (multiplexed, 1 connection)
  
Visualize: waterfall diagram showing the difference
Tools: Wireshark for packet capture, custom Java server
Learning: See multiplexing in action with real numbers
```

#### Project 2: Smart HTTP Cache Layer
```java
@Service
public class SmartCacheService {
    private final Map<String, CachedResponse> cache = new ConcurrentHashMap<>();
    
    public ResponseEntity<?> serveWithCaching(
            String url, String ifNoneMatch, Instant ifModifiedSince) {
        
        CachedResponse cached = cache.get(url);
        
        if (cached != null) {
            // ETag validation
            if (cached.etag.equals(ifNoneMatch)) {
                return ResponseEntity.status(304).eTag(cached.etag).build();
            }
            // Last-Modified validation
            if (cached.lastModified.compareTo(ifModifiedSince) <= 0) {
                return ResponseEntity.status(304).lastModified(cached.lastModified).build();
            }
        }
        
        // Fetch fresh data
        var response = fetchFromOrigin(url);
        cache.put(url, new CachedResponse(response, generateETag(response), Instant.now()));
        return response;
    }
}
```

#### Project 3: HTTP Traffic Analyzer
```
BUILD: Proxy server that intercepts and analyzes HTTP/HTTPS traffic:
- Count requests per origin
- Show response time percentiles (p50, p95, p99)
- Detect unnecessary cache misses
- Identify oversized response bodies
- Flag missing security headers

Tools: Java + Netty (raw HTTP parsing), React for dashboard
Learning: Deep understanding of HTTP wire format
```

#### Project 4: HTTP/3 QUIC Load Tester
```
BUILD: Compare latency on different protocols:
  - HTTP/1.1 over TCP
  - HTTP/2 over TCP  
  - HTTP/3 over QUIC
  
Inject packet loss to see QUIC's advantage (no TCP HOL blocking)
Generate: latency distribution charts, connection reuse statistics
Tools: Java 19+ (Incubator HTTP client with HTTP/2), QUIC libraries
```

---

## 15. Common Pitfalls

**1. Using GET for state-changing operations** — `GET /api/deleteUser?id=123` is cached by browsers, indexed by search engines, and called by link prefetchers. Reads must be idempotent and side-effect free. Use POST/DELETE for mutations.

**2. Ignoring Idempotency for POST** — Payment or order creation APIs must handle duplicate requests (network retry, user double-click). Add `Idempotency-Key` header; server stores result keyed by that ID and returns same result on retry without re-processing.

**3. Caching sensitive data publicly** — `Cache-Control: public` on user-specific responses (bank balance, medical records) means CDN serves Alice's data to Bob. Always use `private` or `no-store` for user-specific data.

**4. CORS wildcard with credentials** — `Access-Control-Allow-Origin: *` with `Access-Control-Allow-Credentials: true` is rejected by browsers. You CANNOT use wildcard when sending cookies/auth headers. Must specify exact origin.

**5. Missing security headers** — Shipping to production without `Strict-Transport-Security`, `Content-Security-Policy`, and `X-Content-Type-Options` is like leaving your front door open. These protect against MITM, XSS, and MIME sniffing attacks.

**6. Not handling 429 / 503 with Retry-After** — When your client receives rate limiting or service unavailable, it should respect the `Retry-After` header. Blind retries amplify the overload problem — this is how cascading failures start.

**7. Large response bodies without pagination** — `GET /api/users` returning 500K users is an OOM waiting to happen. Always paginate, use cursor-based pagination for large datasets, and add streaming for bulk exports.

---

## 16. Interview Q&A

**Q: What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?**
> A: **HTTP/1.1**: text-based, one request at a time per connection (head-of-line blocking), persistent connections, workaround = 6 parallel connections per domain. **HTTP/2**: binary framing, true multiplexing (multiple streams on one TCP connection), header compression (HPACK), server push, solves app-layer HOL blocking but TCP layer HOL blocking remains. **HTTP/3**: uses QUIC over UDP instead of TCP, solves transport-layer HOL blocking (stream isolation), 0-RTT for repeat connections, connection migration (survives IP changes), built-in TLS 1.3.

---

**Q: What happens when you type `https://example.com` in a browser?**
> A: (1) **DNS resolution**: browser cache → OS cache → recursive resolver → root → TLD → authoritative NS → IP address. (2) **TCP handshake**: SYN → SYN-ACK → ACK (1 RTT). (3) **TLS handshake**: ClientHello + key share → ServerHello + certificate + Finished → Finished (1 RTT for TLS 1.3). (4) **HTTP request**: GET / HTTP/1.1 with Host, Accept, etc. (5) **Server processes**: LB → app → possibly DB. (6) **Response**: 200 OK + HTML. (7) **Subresource loading**: parse HTML, discover CSS/JS/images, request them (ideally over same HTTP/2 connection). (8) **Rendering**: DOM + CSSOM → layout → paint.

---

**Q: What is the difference between 401 and 403?**
> A: **401 Unauthorized** means the client is not authenticated — no credentials provided or credentials are invalid. The name is misleading (should be "Unauthenticated"). Response should include `WWW-Authenticate` header explaining how to authenticate. **403 Forbidden** means the client IS authenticated but lacks permission — they're not allowed to access this resource. 401: "I don't know who you are." 403: "I know who you are, but you can't do this."

---

**Q: Explain Cache-Control: no-cache vs no-store**
> A: **`no-cache`**: the response CAN be stored, but must be revalidated with the server before serving (using ETag or Last-Modified). If server confirms unchanged (304 Not Modified), the cached copy is served — saves bandwidth but not the round-trip. **`no-store`**: NEVER cache this response anywhere. Every request must go to the server. Use for sensitive data (payments, medical records). Many developers confuse these — `no-cache` doesn't mean "don't cache," it means "always check first."

---

**Q: What is CORS and how does it work?**
> A: CORS (Cross-Origin Resource Sharing) is a browser security mechanism that restricts cross-origin HTTP requests. When `app.example.com` calls `api.other.com`, the browser enforces CORS. For simple requests (GET, HEAD, simple POST), the browser adds an `Origin` header; the server must respond with `Access-Control-Allow-Origin`. For complex requests (PUT, DELETE, custom headers), the browser first sends a preflight `OPTIONS` request; the server must respond with allowed methods/headers. CORS is enforced by the browser, NOT the server — the request goes through regardless; only the browser blocks the response if CORS headers are missing.

---

**Q: How would you design an API to handle retries safely?**
> A: Three strategies: (1) **Idempotency keys**: client generates a unique UUID per logical operation and includes it as `Idempotency-Key` header. Server stores result by this key; retries return the same result without re-executing. (2) **Idempotent methods**: design mutations as idempotent — `PUT /orders/ORD-123/status` with `{"status":"shipped"}` is idempotent (can repeat safely), unlike `POST /orders/123/ship`. (3) **Conditional updates**: use `If-Match: "etag"` to ensure the resource hasn't changed since last read — optimistic locking. For transient errors (503, network timeout), clients should retry with exponential backoff + jitter to prevent thundering herd.

---

**Q: What are HTTP/2 server push and when should you avoid it?**
> A: Server push allows servers to proactively send resources to the client before it requests them, eliminating extra round trips for critical assets. The server sends `PUSH_PROMISE` frames. **Avoid when**: (1) the client already has the resource cached (wasted bandwidth — the client must cancel the push); (2) resources aren't on the critical rendering path; (3) you can't reliably know what the client needs. Modern alternative: **`103 Early Hints`** (send `Link` headers with preload hints while server processes the main request) is now preferred — it works with caches and doesn't waste bandwidth for already-cached resources.

---

**Q: Explain the difference between chunked transfer encoding and content-length.**
> A: **Content-Length**: server knows the exact response size upfront, sends it in the header, and the client knows when the response is complete. **Chunked Transfer Encoding** (`Transfer-Encoding: chunked`): server doesn't know the total size upfront (dynamically generated content, large file streaming), sends data in chunks with the size of each chunk prefixed. Each chunk: `size-in-hex\r\n`, `data\r\n`. Final chunk: `0\r\n\r\n`. This enables streaming — server starts sending before the full response is ready, and clients can start processing immediately (useful for large exports, server-sent events, live content).

---

**Q: What is HTTP Long Polling and how does it compare to SSE and WebSocket?**
> A: **Long polling**: client sends request, server HOLDS it open until data is available (or timeout), then responds and client immediately sends another request. Simulates push with request-response. **SSE** (Server-Sent Events): single long-lived HTTP connection, server pushes text events. Built-in reconnection, event IDs for resume. One-direction (server→client). **WebSocket**: protocol upgrade from HTTP, full-duplex persistent connection. Both sides can send anytime, binary support. Long polling: simplest, works everywhere. SSE: right for one-way push, HTTP/2 friendly. WebSocket: right for interactive real-time (chat, gaming, collaborative editing).

---

## 17. Cheat Sheet

```
HTTP METHOD QUICK REFERENCE:
  GET    — Read, safe, idempotent, no body, cacheable
  POST   — Create/action, not idempotent, has body, not cacheable
  PUT    — Replace all, idempotent, has body
  PATCH  — Update partial, ideally idempotent, has body
  DELETE — Remove, idempotent, optional body
  HEAD   — Like GET but no body (check headers/existence)
  OPTIONS— Discover methods, CORS preflight

STATUS CODE CHEAT SHEET:
  200 OK, 201 Created, 202 Accepted, 204 No Content, 206 Partial
  301 Permanent Redirect, 302 Temporary, 304 Not Modified
  400 Bad Request, 401 Not Authenticated, 403 Forbidden, 404 Not Found
  405 Wrong Method, 409 Conflict, 422 Invalid Value, 429 Rate Limited
  500 Server Error, 502 Bad Gateway, 503 Unavailable, 504 Timeout

CACHE-CONTROL PATTERNS:
  Static assets (hashed): public, max-age=31536000, immutable
  HTML pages:             no-cache (always revalidate)
  User data:              private, max-age=60
  Sensitive data:         no-store
  Public API:             public, max-age=300, s-maxage=3600

HTTP VERSION COMPARISON:
  1.1: text, keep-alive, HOL blocking at both layers
  2:   binary, multiplex, HPACK, server push, HOL at TCP
  3:   QUIC/UDP, 0-RTT, no HOL at any layer, connection migration

SECURITY HEADER CHECKLIST:
  ✅ Strict-Transport-Security: max-age=31536000; includeSubDomains
  ✅ Content-Security-Policy: default-src 'self'; ...
  ✅ X-Content-Type-Options: nosniff
  ✅ X-Frame-Options: DENY
  ✅ Referrer-Policy: strict-origin-when-cross-origin
  ✅ Permissions-Policy: geolocation=(), microphone=()
```

---

## 🔗 What to Read Next

1. **[APIs/WebSockets.md](./WebSockets.md)** — When HTTP request-response isn't enough
2. **[APIs/WebRTC.md](./WebRTC.md)** — Peer-to-peer real-time communication
3. **[APIs/gRPC.md](./gRPC.md)** — Binary HTTP/2 RPC framework
4. **[Security/TLS_SSL_HTTPS.md](../Security/TLS_SSL_HTTPS.md)** — TLS deep dive
5. **[Foundations/Networking/README.md](../Foundations/Networking/README.md)** — TCP, DNS, protocols

---

*[← API Design Guidelines](./API_Design_Guidelines.md) | [Back to Index](../INDEX.md) | [Next: WebRTC →](./WebRTC.md)*
