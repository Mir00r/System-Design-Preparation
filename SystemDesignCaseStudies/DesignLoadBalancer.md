# ⚖️ Design a Load Balancer

> *"Every request to Google goes through a load balancer. Not one — LAYERS of them! From L4 switching at wire speed to L7 intelligent routing, load balancers are the traffic cops of the internet. Understanding how they work under the hood (not just 'use AWS ALB') is what separates a developer from a system architect."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [Load Balancing](../BuildingBlocks/LoadBalancing.md), [Networking](../Foundations/Networking/TCP_vs_UDP.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Load Balancing Algorithms](#-load-balancing-algorithms)
3. [L4 vs L7 Load Balancing](#-l4-vs-l7-load-balancing)
4. [High-Level Architecture](#-high-level-architecture)
5. [Health Checking](#-health-checking)
6. [Session Persistence](#-session-persistence)
7. [Scaling the Load Balancer](#-scaling-the-load-balancer)
8. [Java Implementation](#-java-implementation)
9. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Distribute incoming traffic across healthy backend servers
  • Health check backends (detect and remove unhealthy!)
  • Support multiple routing algorithms
  • SSL/TLS termination (optional, L7)
  • Session stickiness when needed
  
NON-FUNCTIONAL:
  • Ultra-low latency (< 1ms added overhead!)
  • High throughput (millions of connections/sec!)
  • High availability (LB down = EVERYTHING down!)
  • Graceful failover (no dropped connections!)

SCALE:
  • Handle 1M+ concurrent connections
  • Process 100K+ new connections per second
  • Zero downtime during backend scaling
```

---

## 🎲 Load Balancing Algorithms

```
┌─────────────────────────────────────────────────────────────────────┐
│  Algorithm           │  How It Works             │  Best For         │
├─────────────────────────────────────────────────────────────────────┤
│  Round Robin         │  A → B → C → A → B → C   │  Uniform servers  │
│  Weighted RR         │  A,A,A → B,B → C          │  Mixed hardware   │
│  Least Connections   │  Send to server with      │  Long-lived conns │
│                      │  fewest active connections │  (WebSocket, DB)  │
│  Least Response Time │  Send to fastest server   │  Mixed latencies  │
│  IP Hash             │  hash(client_IP) % N      │  Session stickiness│
│  Random              │  Pick random server       │  Large pools       │
│  Consistent Hash     │  hash(key) → ring         │  Caching proxies   │
│  Power of Two Choices│  Pick 2 random, choose    │  High throughput!  │
│                      │  the less loaded one       │                   │
└─────────────────────────────────────────────────────────────────────┘

POWER OF TWO RANDOM CHOICES (modern favorite!):
  Instead of checking ALL servers (expensive!) or random (bad luck):
  1. Pick 2 servers randomly
  2. Send to the one with fewer connections!
  
  Mathematically proven: exponentially better than pure random!
  O(1) decision time vs O(N) for least-connections!
  Used by: Nginx (EWMA variant), Envoy, HAProxy

WEIGHTED LEAST CONNECTIONS:
  score = active_connections / weight
  Send to server with lowest score!
  
  Server A: weight=5, connections=10 → score = 2.0
  Server B: weight=3, connections=3  → score = 1.0 ← choose B!
  Server C: weight=2, connections=5  → score = 2.5
```

---

## 🔀 L4 vs L7 Load Balancing

```
L4 (Transport Layer — TCP/UDP):
  ┌──────┐         ┌──────┐        ┌──────────┐
  │Client│─TCP SYN─►│ L4 LB│──SYN──►│ Backend  │
  └──────┘         └──────┘        └──────────┘
  
  • Sees: IP addresses, ports, TCP flags
  • Cannot see: HTTP headers, URLs, cookies!
  • Speed: MILLIONS of packets/sec (wire speed!)
  • Decision: based on IP/port only
  • Connection: either NAT or DSR (Direct Server Return)
  • Used for: Database connections, non-HTTP protocols, gaming
  • Examples: AWS NLB, Linux IPVS, F5 hardware LB

L7 (Application Layer — HTTP/gRPC):
  ┌──────┐   HTTP    ┌──────┐   HTTP   ┌──────────┐
  │Client│──request──►│ L7 LB│─request──►│ Backend  │
  └──────┘           └──────┘          └──────────┘
  
  • Sees: EVERYTHING! URL, headers, cookies, body!
  • Can: route by path, modify headers, rewrite URLs
  • Speed: slower (must parse HTTP) — but still 100K+ req/sec
  • Features: SSL termination, compression, caching!
  • Used for: Web apps, APIs, microservices
  • Examples: AWS ALB, Nginx, HAProxy, Envoy, Traefik

WHEN TO USE WHICH:
  ┌───────────────────────────────────────────────────────────────┐
  │  Need                         │  L4        │  L7              │
  ├───────────────────────────────────────────────────────────────┤
  │  Route by URL path            │  ❌        │  ✅ /api → svc A │
  │  SSL termination              │  ❌        │  ✅              │
  │  Maximum throughput           │  ✅ (fastest)│ ❌             │
  │  Non-HTTP protocols           │  ✅        │  ❌              │
  │  WebSocket routing            │  ✅        │  ✅ (with upgrade)│
  │  Header-based routing         │  ❌        │  ✅ A/B testing! │
  │  Connection multiplexing      │  ❌        │  ✅ HTTP/2       │
  └───────────────────────────────────────────────────────────────┘
```

---

## 🏗️ High-Level Architecture

```
PRODUCTION LOAD BALANCING SETUP:

Internet
    │
    ▼ (Anycast → nearest PoP)
┌──────────────────────────────────────────────────────┐
│  DNS + Global Load Balancer (GeoDNS / Anycast)       │
│  Route to nearest region!                             │
└────────────────────┬─────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────┐
│  L4 Load Balancer (NLB)                               │
│  • TCP connection balancing                           │
│  • Ultra-fast (millions of conns!)                    │
│  • HA pair: Active-Passive with VRRP                  │
└────────────────────┬─────────────────────────────────┘
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │  L7 LB   │ │  L7 LB   │ │  L7 LB   │  (Nginx/Envoy pool!)
  │  (Nginx) │ │  (Nginx) │ │  (Nginx) │
  └─────┬────┘ └─────┬────┘ └─────┬────┘
        │             │             │
        │  URL-based routing:       │
        │  /api/* → API servers     │
        │  /static/* → CDN/cache    │
        │  /ws/* → WebSocket servers│
        ▼             ▼             ▼
  ┌──────────┐ ┌──────────┐ ┌──────────┐
  │ App Svr 1│ │ App Svr 2│ │ App Svr 3│  (your code!)
  └──────────┘ └──────────┘ └──────────┘

HA FOR THE LOAD BALANCER ITSELF:
  The LB is the most critical piece! If it fails = total outage!
  
  Solution 1: Active-Passive (VRRP/Keepalived)
  ┌────────┐         ┌────────┐
  │ LB-1   │ VIP ←── │ LB-2   │  (standby, heartbeat!)
  │(active)│         │(passive)│
  └────────┘         └────────┘
  LB-1 dies → LB-2 claims the Virtual IP (< 3s failover!)
  
  Solution 2: Active-Active (Anycast/ECMP)
  Both LBs active! Traffic split via BGP ECMP routing.
  One dies → BGP withdraws route → other absorbs traffic!
  
  Solution 3: Cloud managed (AWS ALB/NLB)
  Multi-AZ, auto-scaling, auto-healing. AWS handles HA!
```

---

## 🏥 Health Checking

```
DON'T SEND TRAFFIC TO DEAD SERVERS!

TYPES OF HEALTH CHECKS:
  ┌───────────────────────────────────────────────────────────────┐
  │  Type      │  How                    │  Detects               │
  ├───────────────────────────────────────────────────────────────┤
  │  TCP       │  Can connect to port?   │  Process crashed       │
  │  HTTP      │  GET /health → 200?     │  App errors, stuck     │
  │  Deep      │  Check DB, cache, deps  │  Dependency failures   │
  │  Custom    │  Business logic check   │  Logical errors        │
  └───────────────────────────────────────────────────────────────┘

HEALTH CHECK PARAMETERS:
  • Interval: 5-30 seconds (how often to check)
  • Timeout: 2-5 seconds (how long to wait for response)
  • Unhealthy threshold: 3 failures → mark DOWN
  • Healthy threshold: 2 successes → mark UP again
  
  Why thresholds? Avoid flapping!
  One failed check might be a network blip, not a dead server!

GRACEFUL REMOVAL (Connection Draining):
  1. Server signals "shutting down" (or health check fails)
  2. LB marks server as DRAINING (no NEW connections!)
  3. Existing connections finish (drain timeout: 30-300s)
  4. After drain: remove from pool completely
  → Zero dropped connections during deployment!
  
  Without draining: BANG! Active requests get RST! Users see errors!
```

---

## 🍪 Session Persistence

```
PROBLEM: Stateful apps need requests from same user → same server!

  Request 1: User logs in → Server A (session stored locally!)
  Request 2: Same user → Server B → "Who are you?!" 😱

SOLUTIONS:
  ┌───────────────────────────────────────────────────────────────┐
  │  Method          │  How                │  Trade-off           │
  ├───────────────────────────────────────────────────────────────┤
  │  Source IP hash  │  hash(IP) → server  │  Breaks behind NAT!  │
  │  Cookie-based    │  LB sets routing    │  Most common! ✅     │
  │                  │  cookie (SERVERID)  │                      │
  │  URL encoding    │  Encode server ID   │  Ugly URLs           │
  │                  │  in URL path        │                      │
  │  Shared session  │  Store in Redis!    │  Best for new apps!  │
  │  store           │  Any server works!  │  Stateless servers!  │
  └───────────────────────────────────────────────────────────────┘

BEST PRACTICE: Make servers STATELESS!
  Store session in Redis → any server can handle any request!
  No stickiness needed → better load distribution!
  Server dies → user doesn't notice (session still in Redis!)
```

---

## 📈 Scaling the Load Balancer

```
WHAT IF THE LOAD BALANCER ITSELF BECOMES THE BOTTLENECK?

  Single LB: ~100K connections (software), 1M+ (hardware)
  Need more? Scale the LB layer!

OPTION 1: DNS Round Robin (poor man's LB scaling)
  DNS returns multiple LB IPs:
  lb1.example.com → 1.2.3.4, 5.6.7.8
  Clients pick one randomly → distributes across LBs!
  
  Downside: DNS caching means slow failover.

OPTION 2: ECMP (Equal-Cost Multi-Path)
  Router distributes packets across multiple LBs at network layer!
  Same IP → multiple paths → hardware does the distribution!
  
  Used by: Google (Maglev), Facebook (Katran)

OPTION 3: Cloud Auto-Scaling
  AWS ALB/NLB auto-scales to handle traffic!
  You don't manage capacity — AWS does!
  
  Caveat: "pre-warming" needed for sudden traffic spikes.
  (Tell AWS in advance if expecting huge traffic event!)

GOOGLE'S APPROACH (Maglev):
  Each datacenter has MANY Maglev machines.
  Anycast IP → ECMP → any Maglev machine.
  Consistent hashing ensures connection affinity ACROSS Maglevs!
  If one Maglev dies → connections smoothly move to others!
  Handles 10M+ packets/second per machine!
```

---

## 💻 Java Implementation

### Simple Round-Robin Load Balancer

```java
public class LoadBalancer {
    
    private final List<Backend> backends = new CopyOnWriteArrayList<>();
    private final AtomicInteger counter = new AtomicInteger(0);
    private final ScheduledExecutorService healthChecker = 
        Executors.newScheduledThreadPool(1);
    
    public LoadBalancer(List<Backend> backends) {
        this.backends.addAll(backends);
        startHealthChecks();
    }
    
    /**
     * Round-robin selection among healthy backends.
     */
    public Backend nextBackend() {
        List<Backend> healthy = backends.stream()
            .filter(Backend::isHealthy)
            .collect(Collectors.toList());
        
        if (healthy.isEmpty()) {
            throw new NoHealthyBackendException("All backends down!");
        }
        
        int idx = Math.abs(counter.getAndIncrement() % healthy.size());
        return healthy.get(idx);
    }
    
    /**
     * Weighted round-robin: servers with higher weight get more traffic.
     */
    public Backend nextWeighted() {
        List<Backend> healthy = backends.stream()
            .filter(Backend::isHealthy)
            .collect(Collectors.toList());
        
        int totalWeight = healthy.stream()
            .mapToInt(Backend::getWeight).sum();
        int random = ThreadLocalRandom.current().nextInt(totalWeight);
        
        int cumulative = 0;
        for (Backend b : healthy) {
            cumulative += b.getWeight();
            if (random < cumulative) return b;
        }
        return healthy.get(healthy.size() - 1);
    }
    
    /**
     * Least connections: send to least-loaded backend.
     */
    public Backend leastConnections() {
        return backends.stream()
            .filter(Backend::isHealthy)
            .min(Comparator.comparingInt(Backend::getActiveConnections))
            .orElseThrow();
    }
    
    /**
     * Health check: periodically verify backends are alive.
     */
    private void startHealthChecks() {
        healthChecker.scheduleAtFixedRate(() -> {
            for (Backend backend : backends) {
                boolean healthy = checkHealth(backend);
                if (!healthy && backend.isHealthy()) {
                    backend.incrementFailCount();
                    if (backend.getFailCount() >= 3) { // threshold!
                        backend.setHealthy(false);
                        log.warn("Backend {} marked DOWN!", backend.getUrl());
                    }
                } else if (healthy) {
                    backend.resetFailCount();
                    backend.setHealthy(true);
                }
            }
        }, 0, 10, TimeUnit.SECONDS); // Check every 10s
    }
    
    private boolean checkHealth(Backend backend) {
        try {
            HttpURLConnection conn = (HttpURLConnection) 
                new URL(backend.getUrl() + "/health").openConnection();
            conn.setConnectTimeout(2000);
            conn.setReadTimeout(2000);
            return conn.getResponseCode() == 200;
        } catch (Exception e) {
            return false;
        }
    }
}
```

### Spring Boot Health Endpoint

```java
@RestController
public class HealthController {
    
    @Autowired private DataSource dataSource;
    @Autowired private RedisTemplate<String, String> redis;
    
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        Map<String, String> status = new HashMap<>();
        boolean allHealthy = true;
        
        // Check database
        try {
            dataSource.getConnection().isValid(2);
            status.put("database", "UP");
        } catch (Exception e) {
            status.put("database", "DOWN");
            allHealthy = false;
        }
        
        // Check Redis
        try {
            redis.getConnectionFactory().getConnection().ping();
            status.put("redis", "UP");
        } catch (Exception e) {
            status.put("redis", "DOWN");
            allHealthy = false;
        }
        
        status.put("status", allHealthy ? "UP" : "DEGRADED");
        return ResponseEntity
            .status(allHealthy ? 200 : 503)
            .body(status);
    }
}
```

---

## ❓ Interview Q&A

**Q1: What happens when a load balancer fails?**
> HA setup is mandatory! (1) Active-Passive with VRRP: standby LB monitors heartbeats from primary, claims the Virtual IP on failure (< 3s failover), (2) Active-Active with ECMP: network routes traffic to multiple LBs, if one fails router detects and redirects, (3) Cloud-managed: AWS ALB/NLB runs across multiple AZs, AWS handles failures transparently. In all cases: the LB is stateless (just routing decisions) so failover is seamless — no session state to lose.

**Q2: How would you implement zero-downtime deployment with a load balancer?**
> (1) Deploy new version to subset of servers, (2) LB health check passes on new servers, (3) Gradually shift traffic (canary: 5% → 25% → 100%), (4) For the old servers: mark as "draining" — stop sending NEW connections but let existing ones finish (connection draining timeout: 30-60s), (5) Once drained: shut down old servers. Key: the LB's health check gives you safe rollback — if new version is unhealthy, LB automatically stops sending traffic!

**Q3: L4 vs L7 — when do you use each?**
> L4 when: you need raw speed (millions of conns/sec), protocol doesn't matter (TCP pass-through), or non-HTTP protocols (database proxying, game servers). L7 when: you need content-based routing (/api → service A, /images → CDN), SSL termination, request modification (add headers, rewrite URLs), or HTTP/2 multiplexing. Most architectures: L4 in front (for HA and speed) → L7 behind (for smart routing). Google/Facebook: L4 at edge (Maglev/Katran) → L7 internally (Envoy).

**Q4: How does consistent hashing help with load balancing?**
> In caching proxies: consistent hashing ensures the same URL always goes to the same cache server (maximizes hit ratio!). In connection-based LB: ensures client X always reaches the same backend (session affinity without cookies!). Key benefit during scaling: adding/removing a backend only redistributes ~1/N keys instead of reshuffling everything. Used by: Maglev (Google's LB), Nginx upstream consistent hash, Envoy ring hash.

---

## 🎮 Mini Challenge

Design the load balancing strategy for a real-time multiplayer game:
- 100K concurrent players
- Players in same game room must connect to same server
- Servers have different capacities (4-core vs 16-core)
- Graceful migration when a server is overloaded

*Hint: Consistent hashing for room→server mapping + weighted least-connections for new room placement + migration protocol for rebalancing!*

---

## 🔗 Related Topics
- [Load Balancing](../BuildingBlocks/LoadBalancing.md) — Fundamentals
- [Service Discovery](../BuildingBlocks/ServiceDiscovery.md) — Finding backends
- [Reverse Proxy](../BuildingBlocks/Proxy_ReverseProxy.md) — Related concept
- [Rate Limiting](../BuildingBlocks/RateLimiting.md) — Often colocated with LB

---

*"A load balancer is the one component that, if it goes down, takes EVERYTHING with it. That's why we obsess over making it the most reliable, fastest, most boring piece of infrastructure possible. Boring infrastructure is GOOD infrastructure." — SRE at Google* ⚖️
