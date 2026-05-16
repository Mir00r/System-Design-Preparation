# 🔀 Proxy & Reverse Proxy: The Invisible Middlemen of the Internet

> *"Every time you load a webpage, between 3 and 7 proxies silently handle your request before it ever reaches the origin server. Understanding proxies is understanding how the internet actually works."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [Load Balancing](./LoadBalancing.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Forward Proxy vs Reverse Proxy](#-forward-proxy-vs-reverse-proxy)
3. [How Forward Proxies Work](#-how-forward-proxies-work)
4. [How Reverse Proxies Work](#-how-reverse-proxies-work)
5. [Real-World Analogy](#-real-world-analogy)
6. [Common Use Cases](#-common-use-cases)
7. [Nginx Reverse Proxy Configuration](#-nginx-reverse-proxy-configuration)
8. [Performance & Trade-offs](#-performance--trade-offs)
9. [Industry Examples](#-industry-examples)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

Without proxies, every client talks directly to every server:

```
Direct connection problems:
  - Client IP is exposed to every server (privacy risk)
  - No caching layer → every identical request hits origin
  - No SSL termination → every backend must manage TLS certs
  - No load distribution → single server handles all traffic
  - No access control gateway → no central point to enforce policies
  - DDoS traffic reaches your app servers directly
```

---

## 💡 Forward Proxy vs Reverse Proxy

```
FORWARD PROXY (sits in front of CLIENTS):

  [Client A] ──┐                    
  [Client B] ──┼──▶ [Forward Proxy] ──▶ [Internet] ──▶ [Server]
  [Client C] ──┘    (hides clients)

  Server sees: proxy's IP (not client IPs)
  Use case: corporate firewalls, content filtering, anonymity (VPN/Tor)


REVERSE PROXY (sits in front of SERVERS):

  [Client] ──▶ [Internet] ──▶ [Reverse Proxy] ──┬──▶ [Server A]
                               (hides servers)    ├──▶ [Server B]
                                                  └──▶ [Server C]

  Client sees: proxy's IP (not server IPs)
  Use case: load balancing, SSL termination, caching, WAF
```

### Key Difference

| Aspect | Forward Proxy | Reverse Proxy |
|---|---|---|
| **Protects** | Clients (hides client identity) | Servers (hides server identity) |
| **Configured by** | Client side (corporate IT) | Server side (infrastructure team) |
| **Client knows?** | Yes (must configure proxy) | No (transparent to client) |
| **Server knows?** | No (sees proxy IP only) | Yes (knows it's behind proxy) |
| **Examples** | Squid, corporate proxy, VPN | Nginx, HAProxy, Cloudflare, Envoy |

---

## 🏗️ How Forward Proxies Work

```
Corporate network example:

Employee browser → [Squid Proxy] → Internet
                      │
                      ├── Cache: if CNN.com was fetched 2 min ago, serve cached version
                      ├── Filter: block access to gambling/social media sites
                      ├── Log: record all employee web activity
                      └── Anonymize: server sees corporate IP, not employee's machine
```

### Use Cases for Forward Proxy

1. **Content filtering** — Block access to unauthorized websites (corporate, schools)
2. **Caching** — Cache frequently-accessed content, save bandwidth
3. **Anonymity** — Hide client identity from servers (Tor, VPN)
4. **Access control** — Authenticate users before allowing internet access
5. **Geo-unblocking** — Access region-restricted content

---

## 🏗️ How Reverse Proxies Work

```
Production architecture:

[Internet Users] ──▶ [Cloudflare CDN/WAF]     ← Layer 1: DDoS protection + edge cache
                          │
                     [Nginx Reverse Proxy]      ← Layer 2: SSL termination + routing
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
        [App Server 1] [App Server 2] [App Server 3]
              │           │           │
              └───────────┼───────────┘
                          ▼
                    [Database]

Reverse proxy responsibilities:
  1. TLS termination (decrypt HTTPS, forward HTTP internally)
  2. Load balancing (distribute across app servers)
  3. Caching (serve static assets without hitting app servers)
  4. Compression (gzip/brotli responses)
  5. Request routing (/api → app servers, /static → CDN)
  6. Rate limiting (block abusive clients)
  7. Health checking (remove unhealthy backends)
  8. Request/response modification (add headers, rewrite URLs)
```

---

## 🌍 Real-World Analogy

**Forward Proxy = A lawyer representing you in court.**
You (client) don't speak directly to the judge (server). Your lawyer speaks on your behalf. The judge knows the lawyer, not you personally. You can change lawyers without the judge knowing.

**Reverse Proxy = A hotel reception desk.**
Guests (clients) don't go directly to housekeeping, kitchen, or maintenance (servers). They go to the front desk, which routes their requests to the right department. Guests don't know how many staff are in the back or which specific person handles their request.

---

## ⚙️ Nginx Reverse Proxy Configuration

### Basic Reverse Proxy

```nginx
# /etc/nginx/conf.d/app.conf
upstream backend_servers {
    # Load balancing with health checks
    server 10.0.1.10:8080 weight=3;   # gets 3x traffic
    server 10.0.1.11:8080 weight=2;
    server 10.0.1.12:8080 weight=1 backup;  # only if others are down
    
    # Keepalive connections to backend (connection pooling)
    keepalive 64;
}

server {
    listen 443 ssl http2;
    server_name api.example.com;

    # TLS termination (HTTPS ends here, HTTP to backends)
    ssl_certificate     /etc/ssl/certs/example.com.pem;
    ssl_certificate_key /etc/ssl/private/example.com.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Proxy to backend servers
    location /api/ {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;          # pass real client IP
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
        proxy_send_timeout 10s;
        
        # Buffer settings
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }

    # Static files served directly (no backend hit)
    location /static/ {
        root /var/www;
        expires 30d;
        add_header Cache-Control "public, immutable";
    }

    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
    location /api/search {
        limit_req zone=api_limit burst=20 nodelay;
        proxy_pass http://backend_servers;
    }
}
```

### Spring Boot Behind a Reverse Proxy

```yaml
# application.yaml: trust proxy headers
server:
  forward-headers-strategy: framework   # trust X-Forwarded-* headers from proxy
  tomcat:
    remote-ip-header: X-Real-IP
    protocol-header: X-Forwarded-Proto
```

```java
// Access real client IP (not proxy IP) in Spring Boot
@GetMapping("/api/resource")
public ResponseEntity<?> getResource(HttpServletRequest request) {
    // Without proxy config: returns 10.0.1.1 (nginx IP)
    // With forward-headers-strategy: returns actual client IP
    String clientIp = request.getRemoteAddr();
    log.info("Request from: {}", clientIp);
    return ResponseEntity.ok(resource);
}
```

---

## 📊 Performance & Trade-offs

| Aspect | Without Reverse Proxy | With Reverse Proxy |
|---|---|---|
| **SSL management** | Every backend handles TLS | Centralized at proxy |
| **Latency** | Direct (lower) | +1-5ms per hop |
| **Caching** | No shared cache | Shared cache across all backends |
| **Scalability** | Hard to add/remove servers | Transparent backend changes |
| **Security** | Backends exposed to internet | Backends hidden, proxy filters traffic |
| **Complexity** | Simpler (fewer components) | One more thing to operate |

### When to Use

```
✅ USE reverse proxy when:
  - Multiple backend instances (load balancing needed)
  - Need SSL termination (don't want certs on every backend)
  - Public-facing API (security layer needed)
  - Microservices architecture (routing needed)
  - Static content can be cached (save backend compute)

❌ SKIP when:
  - Single backend server, low traffic, internal only
  - Ultra-low latency requirement where 1ms matters (e.g., HFT)
  - Simple prototype/MVP (add later when scaling)
```

---

## 🏢 Industry Examples

| Company | Proxy Usage | Tool |
|---|---|---|
| **Cloudflare** | Reverse proxy for 30% of internet traffic | Custom (based on Nginx) |
| **Netflix** | Zuul/Spring Cloud Gateway for API routing | Custom Java proxy |
| **Uber** | Envoy sidecar proxy for all service-to-service communication | Envoy |
| **Airbnb** | Nginx for SSL termination + routing to microservices | Nginx |
| **Google** | Google Front End (GFE) terminates all external connections | Custom |

---

## ⚠️ Common Pitfalls

1. **Not forwarding the real client IP** — Without `X-Forwarded-For` / `X-Real-IP` headers, your backend sees the proxy's IP for every request. Rate limiting, geolocation, and audit logging all break. Always configure proxy headers.

2. **Single point of failure** — One Nginx instance = one failure point. Run at least 2 proxy instances behind a virtual IP (keepalived/VRRP) or use cloud load balancers (AWS ALB) in front.

3. **Proxy buffer too small** — Large responses (file downloads, big JSON) fail with `502 Bad Gateway` when proxy buffers are undersized. Monitor `proxy_buffering` errors and increase `proxy_buffers` if needed.

4. **Timeout mismatch** — Proxy timeout shorter than backend processing time causes premature 504 errors. Set `proxy_read_timeout` longer than your slowest endpoint's P99 latency.

5. **Over-trusting `X-Forwarded-For`** — Clients can spoof this header. Only trust it from known proxy IPs. Use `set_real_ip_from` in Nginx to specify trusted proxy addresses and reject forged headers from external sources.

---

## 🧩 Mini Challenge

**Scenario**: Your API is served by 3 backend instances. You notice that one instance handles 70% of traffic while the other two handle 15% each. You're using round-robin load balancing in your Nginx reverse proxy.

**What could cause this uneven distribution?**

<details>
<summary>💡 Click to reveal answer</summary>

**Likely causes**:

1. **HTTP keep-alive connections**: Round-robin distributes new connections, not requests. If one client opens a keep-alive connection to Server A, all subsequent requests from that client go to Server A without round-robin redistributing. A client making many requests over one connection skews the distribution.

2. **Slow backends**: If Server B and C are slow (responding in 2s vs 100ms for A), Nginx's round-robin still sends to them equally, but connections pile up. If you're looking at active request count (not connection count), slow servers appear to have fewer active requests because they're finishing slower — so the counts don't look uneven, but Server A completes 70% of total requests.

3. **Session stickiness enabled accidentally**: Check for `ip_hash` or `sticky` directive in the upstream config — these pin clients to specific backends.

**Fix**: Use `least_conn` instead of default round-robin — it sends new requests to the backend with fewest active connections, naturally balancing even with keep-alive and varying response times:

```nginx
upstream backend_servers {
    least_conn;
    server 10.0.1.10:8080;
    server 10.0.1.11:8080;
    server 10.0.1.12:8080;
}
```

</details>

---

## 📝 Interview Q&A

**Q: What is the difference between a forward proxy and a reverse proxy?**
> A: A forward proxy sits in front of clients — it hides client identity from servers (clients configure it explicitly). Examples: corporate proxy, VPN. A reverse proxy sits in front of servers — it hides server identity from clients (clients don't know it exists). Examples: Nginx, Cloudflare. The key distinction: forward proxy protects clients; reverse proxy protects servers.

**Q: Why do we need a reverse proxy if we already have a load balancer?**
> A: A load balancer distributes traffic (one function). A reverse proxy does load balancing PLUS: SSL termination, caching, compression, request routing (path-based), rate limiting, security filtering (WAF), header manipulation, and health checking. In practice, tools like Nginx and Envoy serve as both load balancer and reverse proxy. Cloud ALBs (AWS ALB) are reverse proxies that also load balance.

**Q: How does a reverse proxy handle SSL/TLS termination?**
> A: The reverse proxy holds the TLS certificate and private key. It decrypts incoming HTTPS connections from clients, then forwards plain HTTP to backend servers over the internal network. Benefits: backends don't need TLS certificates or crypto overhead; certificate management is centralized; TLS protocol upgrades happen in one place. For sensitive data, you can optionally re-encrypt (proxy → backend over HTTPS), adding end-to-end encryption at the cost of extra latency and certificate management.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/LoadBalancing.md](./LoadBalancing.md)** — Load balancing algorithms that work inside a reverse proxy
2. **[BuildingBlocks/CDN.md](./CDN.md)** — CDNs are geographically distributed reverse proxies
3. **[BuildingBlocks/APIGateway.md](./APIGateway.md)** — API Gateways are reverse proxies with authentication and rate limiting built in

---

*[← Circuit Breaker](./CircuitBreaker.md) | [Back to BuildingBlocks](../INDEX.md) | [Next: Blob Storage →](./Blob_Storage.md)*
