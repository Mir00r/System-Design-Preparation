# ⚖️ Load Balancing: The Art of Distributing Work Without Breaking a Sweat

> **"A chain is only as strong as its weakest link. A load balancer ensures no single link ever carries the full load."**

---

## 🎯 What You'll Learn
- Why load balancing is non-negotiable in production systems
- L4 vs L7 load balancing — the real difference
- 6 load balancing algorithms and when to use each
- Health checks, session persistence, and failure handling
- How FAANG companies use load balancers at massive scale

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium  
**🔗 Prerequisites**: [Scalability](../KeyConcepts/Scalability.md) | [CAP Theorem](../KeyConcepts/CAPTheorem.md)  
**🔗 Related Topics**: [CDN](./CDN.md) | [API Gateway](./APIGateway.md) | [Service Discovery](./ServiceDiscovery.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [What Is a Load Balancer?](#-what-is-a-load-balancer)
3. [L4 vs L7 Load Balancing](#-l4-vs-l7-load-balancing)
4. [Load Balancing Algorithms](#-load-balancing-algorithms)
5. [Health Checks](#-health-checks)
6. [Session Persistence (Sticky Sessions)](#-session-persistence-sticky-sessions)
7. [Code Examples](#-code-examples)
8. [Tools & Technologies](#-tools--technologies)
9. [Industry Examples](#-industry-examples)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)
13. [What to Read Next](#-what-to-read-next)

---

## 🤔 The Problem

It's Black Friday. Your e-commerce site is getting 50,000 requests per second. You have one server.

```
❌ WITHOUT LOAD BALANCER:

50,000 users → [Single Server 🔥💀]
                 ↓
            Server crashes.
            $2M lost in 1 hour.
            Your phone won't stop ringing.
```

You add more servers. But now:
- How do users find the right server?
- What if one server crashes? Do users get errors?
- What if one server is slow?

```
✅ WITH LOAD BALANCER:

50,000 users → [Load Balancer]
                 ↓  ↓  ↓  ↓
              [S1][S2][S3][S4]
              
              Even distribution. 
              Zero downtime when S3 crashes.
              Users don't know or care.
```

> 💡 **Key Insight**: Load balancing isn't just about performance — it's about **reliability**. A load balancer is what transforms a fragile single server into a fault-tolerant cluster.

---

## 💡 What Is a Load Balancer?

A **load balancer** sits between clients and a pool of servers, distributing incoming requests across multiple servers to:

1. **Maximize throughput** — all servers share the work
2. **Minimize response time** — no server gets overwhelmed
3. **Ensure availability** — if one server fails, others take over
4. **Enable horizontal scaling** — add/remove servers transparently

```
LOAD BALANCER ARCHITECTURE:

Internet
    │
    ▼
┌───────────────┐
│  Load Balancer │  ← Single entry point
│  (VIP/Anycast) │
└───────┬───────┘
        │
   ┌────┼────┐
   ▼    ▼    ▼
 [S1]  [S2]  [S3]   ← Server pool (can be 2 or 2000)
  │     │     │
  └─────┴─────┘
       ▼
   Database
```

---

## 🌍 Real-World Analogy

Think of a load balancer like a **supermarket checkout manager**:

```
SUPERMARKET ANALOGY:

Customers (requests) arrive at the entrance.
The manager (load balancer) looks at all checkout lanes:

Lane 1: 3 people in queue → "Don't go here"
Lane 2: 0 people → "Go to lane 2!" 
Lane 3: 5 people → "Avoid"
Lane 4: 1 person → "Also good"

Result: No lane gets overwhelmed.
        Service is fast for everyone.
        If one cashier calls sick, others cover.
```

---

## 🏗️ L4 vs L7 Load Balancing

This is one of the most common interview topics. Know the difference cold.

```
OSI MODEL REMINDER:
Layer 7 = Application (HTTP, HTTPS, WebSocket)
Layer 4 = Transport   (TCP, UDP)
Layer 3 = Network     (IP)
```

### Layer 4 (Transport Layer) Load Balancer

Routes traffic based on **IP address + port** only. Does NOT inspect packet content.

```
L4 ROUTING:
Client sends: TCP packet to 203.0.113.1:443
LB sees:      "Source IP: 1.2.3.4, Dest Port: 443"
LB decides:   Forward to server 10.0.0.2:8443
LB does NOT:  Look at URL path, cookies, or headers
```

**Pros**: Ultra-fast (no packet inspection), works for any TCP/UDP protocol  
**Cons**: Can't route by URL, can't do header-based routing, can't terminate SSL  
**Tools**: AWS NLB, HAProxy (TCP mode), hardware LBs  

### Layer 7 (Application Layer) Load Balancer

Routes traffic based on **HTTP headers, URL paths, cookies, content type**.

```
L7 ROUTING EXAMPLES:

/api/users/*    → User Service servers
/api/orders/*   → Order Service servers
/images/*       → CDN or image servers

Host: mobile.example.com  → Mobile backend
Host: api.example.com     → API backend

Cookie: session=abc123    → Sticky to Server 2
```

**Pros**: Intelligent routing, SSL termination, request inspection, A/B testing  
**Cons**: Slightly slower (packet inspection overhead), more complex config  
**Tools**: AWS ALB, Nginx, Traefik, Envoy, HAProxy (HTTP mode)  

### L4 vs L7 Comparison

| Feature | L4 | L7 |
|---|---|---|
| Routing basis | IP + Port | URL, headers, cookies, content |
| Protocol awareness | TCP/UDP only | HTTP/HTTPS aware |
| SSL termination | ❌ | ✅ |
| URL-based routing | ❌ | ✅ |
| Performance | ⚡ Faster | Slightly slower |
| Use case | Raw TCP, non-HTTP, gaming | HTTP APIs, microservices |
| Cost | Lower | Higher |

> 💡 **Interview Tip**: In practice, most web applications use **L7**. L4 is used for non-HTTP workloads (databases, gaming, IoT) or when you need maximum throughput.

---

## 🧮 Load Balancing Algorithms

### 1. 🔄 Round Robin (Default)

Requests are distributed sequentially to each server in turn.

```
Request 1 → Server 1
Request 2 → Server 2
Request 3 → Server 3
Request 4 → Server 1  (cycle repeats)
Request 5 → Server 2
...
```

**Best for**: Servers with equal capacity and similar request processing times  
**Problem**: Doesn't account for server load or request weight

---

### 2. ⚖️ Weighted Round Robin

Like round robin, but servers with more capacity get more requests.

```
Server 1: weight=3 (powerful)
Server 2: weight=1 (weak)

Distribution: S1, S1, S1, S2, S1, S1, S1, S2...
```

**Best for**: Heterogeneous server pools (different hardware specs)

---

### 3. 📊 Least Connections

New requests go to the server with the **fewest active connections**.

```
Server 1: 50 active connections
Server 2: 20 active connections  ← New request goes here
Server 3: 35 active connections
```

**Best for**: Long-lived connections (WebSockets, database connections, file uploads)  
**Problem**: Doesn't account for connection weight (one heavy query vs many light ones)

---

### 4. ⏱️ Least Response Time

New requests go to the server with the **fastest average response time**.

```
Server 1: avg 200ms response
Server 2: avg 50ms response  ← New request goes here
Server 3: avg 180ms response
```

**Best for**: Heterogeneous workloads where some requests are heavier  
**Problem**: Requires response time monitoring overhead

---

### 5. 🔑 IP Hash (Sticky by Client)

Client IP is hashed to always route to the **same server**.

```
Client IP: 192.168.1.10  → hash(192.168.1.10) % 3 = 1 → Server 1 (always)
Client IP: 10.0.0.5      → hash(10.0.0.5)     % 3 = 2 → Server 2 (always)
```

**Best for**: Stateful applications where session data lives on a specific server  
**Problem**: Uneven distribution if client IPs are skewed; breaks if server count changes

---

### 6. 🎲 Random

Requests are distributed randomly.

```
Server pool: [S1, S2, S3]
Request arrives → pick random server
```

**Best for**: Stateless applications where any server can handle any request  
**Problem**: Can lead to uneven distribution in small pools

---

### Algorithm Decision Tree

```
Is your app stateless?
├── YES → Round Robin or Least Connections
│         └── Are servers identical hardware?
│              ├── YES → Round Robin
│              └── NO  → Weighted Round Robin
└── NO (session-based) → IP Hash or Sticky Sessions
    └── Are requests long-lived (WebSockets)?
         ├── YES → Least Connections
         └── NO  → Round Robin is fine
```

---

## 🩺 Health Checks

A load balancer must know which servers are **alive and healthy** before routing to them.

### Passive Health Checks
Monitor actual traffic. If a server returns too many errors → mark unhealthy.
```
Server returns 5xx errors for 3 consecutive requests → Remove from pool
```

### Active Health Checks
LB periodically sends probe requests to each server.

```
Every 10 seconds, LB sends:
GET /health HTTP/1.1
Host: server1.internal

Expected response:
HTTP/1.1 200 OK
{"status": "healthy", "uptime": "99.9%"}

If timeout or non-200 → Mark server unhealthy → Stop routing
When server recovers → Add back to pool
```

```java
// Spring Boot health endpoint (standard pattern)
@RestController
public class HealthController {
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        // Check DB, cache, critical dependencies
        Map<String, String> status = new HashMap<>();
        status.put("status", "UP");
        status.put("database", checkDatabase() ? "UP" : "DOWN");
        return ResponseEntity.ok(status);
    }
}
```

---

## 🍪 Session Persistence (Sticky Sessions)

Some applications store session state on the server (shopping cart, user session). 
The same user must always hit the same server — this is called **sticky sessions**.

```
WITHOUT STICKY SESSIONS (stateful app):
Request 1 → Server 1 (user logs in, session stored on S1)
Request 2 → Server 2 (user session NOT found → logged out!)
Request 3 → Server 3 (same problem)

WITH STICKY SESSIONS:
Request 1 → Server 1 (session stored)
Request 2 → Server 1 (LB reads cookie → always S1)
Request 3 → Server 1 ✅
```

**Implementation**: LB injects a cookie (`SERVERID=server1`) in the first response. Subsequent requests include this cookie, and LB routes accordingly.

> ⚠️ **Warning**: Sticky sessions create a single-point-of-failure. If Server 1 goes down, all its sessions are lost. **Preferred solution**: Use a **distributed session store** (Redis) so any server can handle any request.

```
✅ BETTER APPROACH:
All servers share session state via Redis

Request 1 → Server 1 → reads/writes session from Redis
Request 2 → Server 3 → reads/writes SAME session from Redis
Request 3 → Server 2 → still works! 

No sticky sessions needed.
```

---

## 💻 Code Examples

### Nginx L7 Load Balancer Config (Round Robin)
```nginx
# /etc/nginx/nginx.conf

upstream app_servers {
    # Round Robin (default — no keyword needed)
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    server 10.0.0.3:8080;
    
    # Health check
    keepalive 32;
}

server {
    listen 80;
    
    location /api/ {
        proxy_pass http://app_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        
        # Timeout settings
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

### Nginx with Weighted + Least Connections
```nginx
upstream app_servers {
    least_conn;  # Least connections algorithm
    
    server 10.0.0.1:8080 weight=3;  # Powerful server gets 3x requests
    server 10.0.0.2:8080 weight=1;
    server 10.0.0.3:8080 backup;    # Only used if others are down
}
```

### Spring Cloud LoadBalancer (Java — client-side)
```java
@Configuration
public class LoadBalancerConfig {
    
    @Bean
    @LoadBalanced  // Spring will intercept and load balance
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class OrderService {
    
    @Autowired
    private RestTemplate restTemplate;
    
    public Order getOrder(String orderId) {
        // "order-service" is resolved by load balancer to an actual host
        return restTemplate.getForObject(
            "http://order-service/api/orders/" + orderId,
            Order.class
        );
    }
}
```

---

## 🛠️ Tools & Technologies

| Tool | Type | Best For |
|---|---|---|
| **AWS ALB** (Application Load Balancer) | L7 Managed | HTTP/HTTPS apps on AWS, ECS, EKS |
| **AWS NLB** (Network Load Balancer) | L4 Managed | TCP/UDP, extreme performance, static IP |
| **Nginx** | L7 Software | General purpose, reverse proxy, high customization |
| **HAProxy** | L4+L7 Software | High performance, bare metal, complex ACLs |
| **Traefik** | L7 Software | Kubernetes-native, dynamic config, auto-discovery |
| **Envoy** | L7 Software | Service mesh sidecar, advanced observability |
| **Kubernetes Ingress** | L7 Managed | K8s workloads (backed by Nginx, Traefik, etc.) |

---

## 🏢 Industry Examples

```
NETFLIX:
- Uses AWS ALB for HTTP API traffic
- Uses Zuul (their own LB/API Gateway) for fine-grained routing
- Handles 2.5+ billion edge requests per day

TWITTER/X:
- Uses HAProxy as L7 LB, handling millions of TPS
- Fan-out service uses consistent hashing for routing

UBER:
- Uses Nginx + custom routing layer
- Geographic load balancing — route to nearest data center first

GOOGLE:
- Built their own "Maglev" software load balancer
- Processes 1 million+ requests/second on a single machine
- Published paper on it: "Maglev: A Fast and Reliable Software Network Load Balancer"
```

---

## ⚠️ Common Pitfalls

### 1. The Load Balancer Becomes the Bottleneck
```
❌ BAD: Single load balancer handles all 50K rps → it crashes
✅ GOOD: Active-passive or active-active LB pair with VIP failover
         OR use DNS-based load balancing + multiple LB instances
```

### 2. Not Implementing Health Checks
```
❌ BAD: LB keeps routing to crashed server → users get errors
✅ GOOD: Active health checks with proper /health endpoint
         Remove failed servers within 10-30 seconds
```

### 3. Sticky Sessions with No Session Replication
```
❌ BAD: Sticky sessions + no Redis = data loss when server dies
✅ GOOD: Stateless app + distributed session store (Redis)
```

### 4. Ignoring SSL Termination
```
❌ BAD: Each server manages its own SSL cert → cert management nightmare
✅ GOOD: SSL termination at LB → internal traffic is HTTP
         Reduces CPU load on app servers by 10-30%
```

### 5. Long Health Check Intervals
```
❌ BAD: Health check every 60 seconds → 60s of errors before removing bad server
✅ GOOD: Health check every 5-10 seconds with 2-3 failures threshold
```

---

## 🧩 Mini Challenge

```
🎲 SCENARIO (3 minutes):

You're designing a video streaming service.
You have two types of requests:
  - API requests (fast, stateless, ~50ms each)  
  - Video chunk requests (slow, large files, ~5 seconds each)

Your current setup: 10 servers behind one Round Robin LB.

PROBLEM: Users report API calls are slow during peak hours.

QUESTION: What's wrong, and how do you fix it?
```

<details>
<summary>💡 Click to reveal answer</summary>

**Problem**: Video chunk requests hold connections for ~5 seconds each. During peak hours, all 10 servers are busy with video requests. New API requests (which take 50ms) are queued behind 5-second video requests.

Round Robin doesn't know about request weight — it just distributes equally.

**Fix 1**: Separate server pools
```nginx
upstream api_servers {
    server api1:8080;
    server api2:8080;
}

upstream video_servers {
    least_conn;  # Critical — video requests are long-lived
    server video1:8080;
    server video2:8080;
}

location /api/ { proxy_pass http://api_servers; }
location /video/ { proxy_pass http://video_servers; }
```

**Fix 2**: For video servers, switch from Round Robin to **Least Connections** — so new video requests go to servers with the fewest active streams.

**Lesson**: L7 LB + request-type-aware routing + correct algorithm per workload type.
</details>

---

## 📝 Interview Q&A

**Q1: What is the difference between L4 and L7 load balancing?**
> L4 routes based on IP/port (TCP/UDP), doesn't inspect content, faster but less flexible. L7 routes based on HTTP headers/URL/cookies, slower but can do path-based routing, SSL termination, and A/B testing.

**Q2: What happens if the load balancer itself goes down?**
> Use an Active-Passive pair: primary LB handles traffic, standby monitors via heartbeat. If primary fails, standby takes over via a Virtual IP (VIP) that both share. Alternatively, use DNS-based failover or anycast routing.

**Q3: How does a load balancer handle a server that crashes mid-request?**
> Active health checks detect the failure within the configured interval (typically 10-30s). The LB stops routing new requests to the failed server. In-flight requests may fail — clients should retry with exponential backoff. Some LBs support connection draining to let existing requests complete before removal.

**Q4: What is the difference between a load balancer and a reverse proxy?**
> A reverse proxy forwards client requests to backend servers (one backend server typically). A load balancer distributes requests across multiple backend servers. In practice, they often overlap — Nginx is both.

**Q5: When would you use Least Connections over Round Robin?**
> Least Connections when request processing time varies significantly (WebSockets, long-lived connections, file uploads). Round Robin when requests are short and roughly equal in processing time (REST API calls, stateless requests).

**Q6: What is connection draining?**
> When a server is removed from the pool (for maintenance or failure), connection draining allows existing active requests to complete before the server is fully removed. New requests are not sent to it, but open connections are given time to finish (typically 30-300 seconds).

---

## 🔗 What to Read Next

| Topic | Why You Need It |
|---|---|
| [CDN](./CDN.md) | LBs handle your origin — CDNs handle your static content and global distribution |
| [API Gateway](./APIGateway.md) | API Gateways are specialized L7 LBs with auth, rate limiting, and observability built in |
| [Service Discovery](./ServiceDiscovery.md) | In microservices, servers register themselves — LBs need service discovery to know the pool |

---

*[← Back to Building Blocks](../BuildingBlocks/) | [← Index](../INDEX.md)*
