# 💓 Heartbeats in Distributed Systems: The Pulse of Your Infrastructure

> *"In 2017, an AWS S3 outage lasted 4 hours because a health check script accidentally marked healthy servers as dead. Heartbeats are the simplest concept in distributed systems — and the most dangerous when they go wrong. A missed heartbeat can cascade into a full system collapse."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [SPOF](../KeyConcepts/SPOF.md), [Failover](../KeyConcepts/Failover.md)

---

## 📋 Table of Contents
1. [What are Heartbeats?](#-what-are-heartbeats)
2. [How Heartbeats Work](#-how-heartbeats-work)
3. [Types of Heartbeat Mechanisms](#-types-of-heartbeat-mechanisms)
4. [Failure Detection Strategies](#-failure-detection-strategies)
5. [The False Positive Problem](#-the-false-positive-problem)
6. [Heartbeats in Real Systems](#-heartbeats-in-real-systems)
7. [Java Implementation](#-java-implementation)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 What are Heartbeats?

```
╔══════════════════════════════════════════════════════════════════╗
║  HEARTBEAT = A periodic signal sent by a component to indicate ║
║  "I am alive and functioning."                                 ║
║                                                                ║
║  If heartbeats stop → component is assumed dead → action taken ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Real-Life Analogy

```
  A scuba diver has a "buddy check" every 5 minutes:
  
  🤿 "You OK?" → 👍 "I'm OK!" (heartbeat received)
  🤿 "You OK?" → 👍 "I'm OK!" (heartbeat received)
  🤿 "You OK?" → 😶 ... silence ... (heartbeat MISSED!)
  🤿 "You OK?" → 😶 ... silence ... (2nd miss!)
  🤿 "EMERGENCY! Buddy is unresponsive!" → Rescue initiated!

  Same in distributed systems:
  Monitor: "Server, are you alive?" → Server: "Yes!" ✅
  Monitor: "Server, are you alive?" → ... timeout ... ❌
  Monitor: "Server, are you alive?" → ... timeout ... ❌
  Monitor: "3 misses! Declaring server DEAD. Starting failover!"
```

---

## ⚙️ How Heartbeats Work

```
PUSH Model (most common):
  Server sends heartbeat TO monitor at regular intervals

  ┌────────────┐  ❤️ every 5s   ┌─────────────┐
  │  Server    │─────────────────►│  Monitor    │
  │            │  "I'm alive!"   │  (tracks    │
  │            │                 │   last seen) │
  └────────────┘                 └─────────────┘
  
  Monitor logic:
    if (now - lastHeartbeat > threshold):
        markAsDead(server)
        triggerFailover()

PULL Model:
  Monitor ASKS server "are you alive?"

  ┌─────────────┐  "ping?"   ┌────────────┐
  │  Monitor    │────────────►│  Server    │
  │             │◄────────────│            │
  │             │  "pong!"   │            │
  └─────────────┘            └────────────┘
```

### Timeline

```
Normal:
  T=0s  ❤️ received    (healthy)
  T=5s  ❤️ received    (healthy)
  T=10s ❤️ received    (healthy)
  T=15s ❤️ received    (healthy)

Failure detection:
  T=20s ❤️ received    (healthy)
  T=25s ❌ MISSED!     (suspicious...)
  T=30s ❌ MISSED!     (more suspicious...)
  T=35s ❌ MISSED!     (3 misses = DEAD! 💀)
  T=36s 🚨 FAILOVER TRIGGERED!
  
  Detection time: 3 × interval = 15 seconds
  (configurable: faster detection = more false positives)
```

---

## 📊 Types of Heartbeat Mechanisms

```
┌────────────────────┬──────────────────────────────────────────────┐
│  Type              │  How It Works                                │
├────────────────────┼──────────────────────────────────────────────┤
│  Simple Ping       │  "Are you alive?" → "Yes"                   │
│                    │  Checks: connectivity only                   │
├────────────────────┼──────────────────────────────────────────────┤
│  Health Check      │  HTTP GET /health → 200 OK + metrics        │
│                    │  Checks: app is functional, not just alive   │
├────────────────────┼──────────────────────────────────────────────┤
│  Gossip-based      │  Nodes exchange state with random peers      │
│                    │  Checks: cluster-wide health consensus       │
├────────────────────┼──────────────────────────────────────────────┤
│  Leader Election   │  Leader sends heartbeat to followers         │
│                    │  Miss = trigger new election                 │
├────────────────────┼──────────────────────────────────────────────┤
│  Application-level │  "I processed 1000 requests last minute"    │
│                    │  Checks: functional health, not just alive   │
└────────────────────┴──────────────────────────────────────────────┘
```

### Simple vs Rich Heartbeats

```
SIMPLE HEARTBEAT:
  { "status": "alive", "timestamp": 1706123456 }
  
  Pro: Lightweight, low bandwidth
  Con: Server could be "alive" but not processing requests (zombie!)

RICH HEARTBEAT:
  {
    "status": "alive",
    "timestamp": 1706123456,
    "cpu_usage": 72,
    "memory_usage": 85,
    "active_connections": 1234,
    "requests_per_second": 5000,
    "error_rate": 0.01,
    "disk_free_gb": 42,
    "last_successful_db_query": 1706123455
  }
  
  Pro: Full picture of health, can detect degradation
  Con: More bandwidth, more processing
```

---

## 🎯 Failure Detection Strategies

### Fixed Timeout

```
  Config: interval = 5s, threshold = 3 misses
  Dead after: 15 seconds of silence
  
  ✅ Simple
  ❌ Doesn't adapt to network conditions
  ❌ Same timeout for WiFi and cross-continent
```

### Adaptive (Phi Accrual Failure Detector)

```
  Used by: Cassandra, Akka
  
  Instead of binary "alive/dead", calculates PROBABILITY of failure:
  
  φ = -log10(probability that heartbeat is just late)
  
  φ = 1  → 10% chance it's dead    (probably just late)
  φ = 3  → 99.9% chance it's dead  (probably dead!)
  φ = 5  → 99.999% dead            (definitely dead!)
  
  Threshold: typically φ > 8 → declare dead
  
  WHY BETTER: Adapts to network variance!
    - Stable network: quick detection (low variance)
    - Unstable network: more tolerance (high variance)
```

### Quorum-Based Detection

```
  Multiple monitors vote on "is node X dead?"
  
  Monitor A: "Node X missed 3 heartbeats" → DEAD
  Monitor B: "Node X responded to me 2 seconds ago" → ALIVE
  Monitor C: "Node X missed heartbeats" → DEAD
  
  Quorum (2/3): DEAD
  
  ✅ Reduces false positives (network partition between one monitor
     and node doesn't cause false declaration)
```

---

## ⚠️ The False Positive Problem

```
THE DANGER: Declaring a node dead when it's actually alive!

CAUSES OF FALSE POSITIVES:
  1. Network hiccup (packet lost, temporary congestion)
  2. GC pause (JVM stops for 500ms during full GC)
  3. CPU spike (heartbeat thread starved)
  4. Network partition (monitor can't reach node, but node is fine)

CONSEQUENCES:
  ┌──────────────────────────────────────────────────────┐
  │  False positive → Unnecessary failover                │
  │  → New node takes over workload                       │
  │  → Old node ALSO still working (didn't actually die!)│
  │  → SPLIT BRAIN! Two nodes doing same work! 💀        │
  └──────────────────────────────────────────────────────┘

MITIGATION:
  1. Multiple missed heartbeats before declaration (3-5)
  2. Multiple monitors must agree (quorum)
  3. Adaptive timeouts (adjust to network conditions)
  4. Fencing: when declaring dead, FORCE-KILL the suspect node
     (STONITH: Shoot The Other Node In The Head)
```

---

## 🏢 Heartbeats in Real Systems

### Kubernetes: Kubelet Heartbeats
```
  Every 10 seconds, kubelet sends heartbeat to API server:
  
  Kubelet (on Node) → API Server: "I'm alive, here are my pods"
  
  If 40 seconds pass without heartbeat:
    Node marked as "NotReady"
    After 5 minutes: Pods rescheduled to other nodes
    
  This is why K8s pod recovery takes 5+ minutes by default!
```

### Apache Kafka: Broker Heartbeats (via ZooKeeper)
```
  Each broker maintains a ZooKeeper ephemeral node:
  
  /brokers/ids/0  (ephemeral — auto-deleted if session dies)
  /brokers/ids/1
  /brokers/ids/2
  
  Broker heartbeat = ZooKeeper session keepalive (every 6s)
  Session timeout: 18s (3 missed keepalives)
  
  Broker dies → Ephemeral node deleted → Controller notified
  → Partition leadership reassigned within seconds
```

### Redis Sentinel
```
  Sentinel instances ping Redis master every 1 second:
  
  Sentinel → PING → Redis Master → PONG ✅
  Sentinel → PING → Redis Master → ... (no response)
  
  After down-after-milliseconds (default 30s):
    Sentinel marks master as "SDOWN" (Subjectively Down)
    
  When quorum of sentinels agree:
    Marked as "ODOWN" (Objectively Down)
    Failover initiated!
```

---

## 💻 Java Implementation

### Simple Heartbeat Server

```java
@Component
public class HeartbeatService {
    
    private final Map<String, Instant> lastHeartbeat = new ConcurrentHashMap<>();
    private static final Duration TIMEOUT = Duration.ofSeconds(15);
    
    // Receive heartbeats from registered services
    @PostMapping("/heartbeat")
    public void receiveHeartbeat(@RequestParam String serviceId) {
        lastHeartbeat.put(serviceId, Instant.now());
    }
    
    // Check for dead services every 5 seconds
    @Scheduled(fixedRate = 5000)
    public void checkHeartbeats() {
        Instant now = Instant.now();
        
        lastHeartbeat.forEach((serviceId, lastSeen) -> {
            if (Duration.between(lastSeen, now).compareTo(TIMEOUT) > 0) {
                handleServiceDown(serviceId);
            }
        });
    }
    
    private void handleServiceDown(String serviceId) {
        log.error("Service {} has not sent heartbeat in {}s — marking as DOWN!",
            serviceId, TIMEOUT.getSeconds());
        
        // Remove from service registry
        serviceRegistry.deregister(serviceId);
        
        // Trigger failover
        failoverService.initiateFailover(serviceId);
        
        // Alert operations team
        alertService.sendAlert("SERVICE_DOWN", serviceId);
    }
}
```

### Heartbeat Sender (Client Side)

```java
@Component
public class HeartbeatSender {
    
    @Value("${app.service-id}")
    private String serviceId;
    
    @Autowired private RestTemplate restTemplate;
    
    @Scheduled(fixedRate = 5000) // Send heartbeat every 5 seconds
    public void sendHeartbeat() {
        try {
            HealthStatus status = collectHealthMetrics();
            
            restTemplate.postForEntity(
                "http://monitor/heartbeat?serviceId=" + serviceId,
                status, Void.class);
                
        } catch (Exception e) {
            log.warn("Failed to send heartbeat: {}", e.getMessage());
            // Continue running — we're still alive even if monitor is unreachable!
        }
    }
    
    private HealthStatus collectHealthMetrics() {
        Runtime runtime = Runtime.getRuntime();
        return HealthStatus.builder()
            .serviceId(serviceId)
            .timestamp(Instant.now())
            .cpuUsage(getCpuUsage())
            .memoryUsed(runtime.totalMemory() - runtime.freeMemory())
            .activeRequests(requestCounter.get())
            .build();
    }
}
```

---

## ⚠️ Common Pitfalls

| Pitfall | Why It's Dangerous | Fix |
|---------|-------------------|-----|
| 🔴 Single missed heartbeat = dead | Network hiccup causes unnecessary failover | Require 3+ misses before action |
| 🔴 Heartbeat on same network as data | Network saturation delays heartbeats | Separate network for heartbeats |
| 🔴 Fixed timeout for all environments | Same timeout for local and cross-region | Adaptive timeouts (Phi Accrual) |
| 🟡 Heartbeat says "alive" but service is broken | Zombie state — alive but not processing | Rich health checks, not just ping |
| 🟡 No alerting on heartbeat failures | Team doesn't know failover happened | Alert on every state change |

---

## 🎮 Mini Challenge

### 🧩 Design: Heartbeat System for Microservices

You have 200 microservice instances across 3 data centers. Design a heartbeat system that:
- Detects failures within 30 seconds
- Has < 0.1% false positive rate
- Survives monitor node failures
- Doesn't overwhelm the network

**Questions:**
1. Push or pull model? Why?
2. Heartbeat interval and failure threshold?
3. How many monitor nodes? What if a monitor dies?
4. How do you handle network partitions between DCs?

<details>
<summary>🔑 Answers</summary>

1. **Push model** — 200 services pulling every 5s = 40 req/s to each service (unnecessary load). Push: each service sends one HTTP call every 5s = 200 req/s to monitor (manageable).
2. **Interval: 5s, Threshold: 5 misses = 25s detection** — Under 30s requirement with buffer for network jitter.
3. **3 monitors (one per DC)** — Quorum-based detection. If a monitor dies, remaining 2 form majority. Services send heartbeats to ALL monitors.
4. **Per-DC detection** — If DC-1 monitor can't reach DC-2 services, don't declare them dead (it's probably a network partition). Only local monitors vote on local services. Cross-DC state shared via gossip.
</details>

---

## ❓ Interview Q&A

**Q1: What is a heartbeat in distributed systems?**
> A periodic signal (usually every 1-10 seconds) sent by a component to indicate it's alive and functioning. If heartbeats stop arriving within a timeout, the component is assumed dead and failover/recovery actions are triggered.

**Q2: How do you prevent false positives in heartbeat-based failure detection?**
> (1) Require multiple consecutive misses (3-5), (2) Use quorum — multiple monitors must agree, (3) Adaptive timeouts that adjust to network variance (Phi Accrual), (4) Separate heartbeat from data network to avoid congestion interference.

**Q3: What's the tradeoff between fast detection and false positives?**
> Shorter heartbeat interval = faster detection but more false positives (brief network blip triggers failover). Longer interval = fewer false positives but slower detection (real failures take longer to notice). Typical balance: 5s interval, 3-5 misses = 15-25s detection.

**Q4: How do heartbeats relate to leader election?**
> In leader-based systems (Raft, Kafka), the leader sends heartbeats to followers. If followers don't receive heartbeats within election timeout, they assume the leader is dead and start a new election. This is how Kafka, etcd, and ZooKeeper detect leader failures.

**Q5: What's the difference between liveness and readiness probes in Kubernetes?**
> Liveness probe: "Is the container alive?" (restart if failed). Readiness probe: "Can the container serve traffic?" (remove from load balancer if failed). A container can be alive but not ready (e.g., still warming up cache). Both use heartbeat-like mechanisms (HTTP check, TCP check, or exec command).

---

## 🔗 Related Topics
- [Failover](../KeyConcepts/Failover.md) — Triggered by heartbeat failure
- [Service Discovery](../BuildingBlocks/ServiceDiscovery.md) — Uses heartbeats for health tracking
- [Gossip Protocol](./GossipProtocol.md) — Peer-to-peer heartbeat alternative
- [Consensus](../KeyConcepts/Consensus.md) — Leader heartbeats in Raft/Paxos

---

*"A heartbeat is the simplest thing in distributed systems to implement, and the hardest thing to get right. Too sensitive = chaos. Too slow = downtime." — Distributed Systems Folklore* 💓
