# 🌐 Networking Fundamentals: How Data Travels

> *"Every system design decision — caching, load balancing, microservices, CDNs — is fundamentally a networking decision. Understanding TCP, HTTP, and DNS isn't optional knowledge; it's the foundation everything else is built on."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: Basic CS knowledge

---

## 🏗️ The Network Stack (OSI Simplified)

```
APPLICATION LAYER:  HTTP, gRPC, WebSocket, DNS, SMTP
                    ↕ (what developers interact with)
TRANSPORT LAYER:    TCP, UDP
                    ↕ (reliable vs fast delivery)
NETWORK LAYER:      IP (routing packets across networks)
                    ↕
LINK LAYER:         Ethernet, Wi-Fi (physical delivery)

A REQUEST'S JOURNEY:
  Browser → DNS → TCP handshake → TLS handshake → HTTP request → 
  → Server processes → HTTP response → renders page
```

---

## 📡 TCP vs UDP

```
TCP (Transmission Control Protocol):
  ┌────────┐  SYN ──────────→  ┌────────┐
  │ Client │  ←──── SYN+ACK    │ Server │   3-way handshake
  │        │  ACK ──────────→  │        │
  │        │                   │        │
  │        │  Data ────────→   │        │   Reliable, ordered
  │        │  ←──── ACK       │        │   delivery guaranteed
  │        │  Data ────────→   │        │
  │        │  ←──── ACK       │        │
  └────────┘                   └────────┘
  
  ✅ Reliable (retransmits lost packets)
  ✅ Ordered (reassembles in sequence)
  ✅ Flow control (doesn't overwhelm receiver)
  ✅ Congestion control (adapts to network)
  ❌ Higher latency (handshake + ACKs)
  ❌ Head-of-line blocking (one lost packet blocks all)
  
  USE: HTTP, email, file transfer, database connections

UDP (User Datagram Protocol):
  ┌────────┐  Data ────────→  ┌────────┐
  │ Client │  Data ────────→  │ Server │   No handshake!
  │        │  Data ────────→  │        │   Fire and forget
  └────────┘                   └────────┘
  
  ✅ Low latency (no handshake, no ACKs)
  ✅ No head-of-line blocking
  ✅ Supports multicast/broadcast
  ❌ Unreliable (packets can be lost)
  ❌ Unordered (can arrive out of sequence)
  ❌ No congestion control (can flood network)
  
  USE: Video streaming, gaming, DNS queries, VoIP
```

---

## 🌍 HTTP Evolution

```
HTTP/1.0 (1996):
  One request per TCP connection → close → reopen
  Extremely slow for modern web (10+ resources per page)

HTTP/1.1 (1997):
  Keep-alive: reuse TCP connection for multiple requests
  BUT: head-of-line blocking (requests processed sequentially)
  Workaround: browsers open 6 parallel connections per domain

HTTP/2 (2015):
  Multiplexing: multiple requests on ONE TCP connection simultaneously
  Server push: server sends resources before client asks
  Header compression (HPACK): reduces overhead
  Binary framing: more efficient parsing
  BUT: TCP head-of-line blocking still exists at transport layer

HTTP/3 (2022):
  Built on QUIC (UDP-based, not TCP!)
  No TCP head-of-line blocking (streams are independent)
  0-RTT connection resumption (instant for repeat visits)
  Built-in encryption (TLS 1.3 integrated)
  Connection migration (survives IP changes, e.g., WiFi→cellular)
  
  ┌─────────────────────────────────────────────┐
  │ HTTP/1.1:  ■■■■□□□□□□  (sequential)        │
  │ HTTP/2:    ■■■■■■■■■■  (multiplexed/TCP)   │
  │ HTTP/3:    ■■■■■■■■■■  (multiplexed/QUIC)  │
  └─────────────────────────────────────────────┘
```

---

## 🔐 TLS Handshake

```
TLS 1.2 (2 round trips):
  Client → Server: ClientHello (supported ciphers, random)
  Server → Client: ServerHello + Certificate + KeyExchange
  Client → Server: KeyExchange + ChangeCipherSpec + Finished
  Server → Client: ChangeCipherSpec + Finished
  
  Total: 2 RTT before first byte of data
  If RTT = 50ms → 100ms added latency

TLS 1.3 (1 round trip):
  Client → Server: ClientHello + KeyShare
  Server → Client: ServerHello + KeyShare + Certificate + Finished
  Client → Server: Finished + [Application Data]
  
  Total: 1 RTT (50% faster!)
  0-RTT resumption: 0 RTT for repeat connections (pre-shared keys)

SYSTEM DESIGN IMPLICATION:
  Internal services (same datacenter): mTLS adds ~1ms overhead
  Cross-region: TLS adds 1-2 RTT (100-300ms per new connection)
  → Use connection pooling! Reuse established TLS connections.
```

---

## 🏢 DNS (Domain Name System)

```
DNS RESOLUTION:
  Browser cache → OS cache → Router → ISP Recursive Resolver → 
  → Root NS → TLD NS (.com) → Authoritative NS → IP address

  "google.com" → 142.250.80.46

DNS RECORD TYPES:
  A:     domain → IPv4 address
  AAAA:  domain → IPv6 address
  CNAME: alias → another domain name
  MX:    domain → mail server
  NS:    domain → authoritative nameserver
  TXT:   domain → arbitrary text (SPF, DKIM verification)
  SRV:   service discovery (host + port)

DNS IN SYSTEM DESIGN:
  • Global load balancing (GeoDNS → route to nearest datacenter)
  • Failover (health-check → remove unhealthy IPs from DNS)
  • Blue/green deployment (swap DNS to point to new environment)
  • TTL trade-off: low TTL = fast failover but more DNS queries
                   high TTL = cached longer but slow to update
```

---

## 📊 Key Numbers Every Developer Should Know

| Operation | Latency |
|---|---|
| L1 cache reference | 0.5 ns |
| L2 cache reference | 7 ns |
| Main memory (RAM) | 100 ns |
| SSD random read | 150 µs |
| HDD random read | 10 ms |
| Same datacenter round trip | 0.5 ms |
| Send 1 MB over 1 Gbps network | 10 ms |
| Cross-continent round trip | 150 ms |
| TCP handshake (same region) | 0.5 ms |
| TLS handshake (same region) | 1-2 ms |
| DNS lookup (uncached) | 20-120 ms |

---

## ⚠️ Common Pitfalls

1. **Ignoring latency budgets** — If your SLA is 200ms and a cross-region call takes 150ms, you have 50ms for everything else. Map your latency budget FIRST.

2. **Not using connection pooling** — Each new TCP+TLS connection costs 2-3 RTTs. Reuse connections (HTTP keep-alive, database connection pools, gRPC persistent connections).

3. **DNS as single point of failure** — If your DNS provider goes down (Dyn attack 2016), your entire service is unreachable. Use multiple DNS providers or anycast DNS.

---

## 📝 Interview Q&A

**Q: What happens when you type google.com in a browser?**
> A: (1) **DNS resolution**: browser cache → OS cache → recursive resolver → root → .com TLD → google.com authoritative NS → returns IP. (2) **TCP handshake**: SYN → SYN-ACK → ACK (3-way). (3) **TLS handshake**: ClientHello → ServerHello+Cert → key exchange (1-2 RTT). (4) **HTTP request**: GET / HTTP/2 with headers. (5) **Server processes**: load balancer → web server → application → maybe database. (6) **HTTP response**: status 200, HTML body, headers. (7) **Browser renders**: parse HTML → request CSS/JS/images → build DOM → paint.

---

## 🔗 What to Read Next

1. **[Foundations/OperatingSystems/README.md](../OperatingSystems/README.md)** — Processes and threads
2. **[Foundations/HowInternetWorks/README.md](../HowInternetWorks/README.md)** — Full request lifecycle
3. **[BuildingBlocks/CDN.md](../../BuildingBlocks/CDN.md)** — Content delivery networks

---

*[← Foundations Overview](../README.md) | [Back to Index](../../INDEX.md) | [Next: Operating Systems →](../OperatingSystems/README.md)*
