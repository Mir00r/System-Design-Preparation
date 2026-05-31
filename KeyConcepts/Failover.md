# 🔄 Failover: When Plan A Dies, Plan B Saves the Day

> *"In 2016, Delta Airlines' primary data center lost power. Their failover system kicked in... except it didn't work because it hadn't been tested in 3 years. Result: 2,000 flights cancelled, $150M loss, and 500,000 stranded passengers. Failover that isn't tested is failover that doesn't exist."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [SPOF](./SPOF.md), [Availability](./Availability.md), [Reliability](./Reliability.md)

---

## 📋 Table of Contents
1. [What is Failover?](#-what-is-failover)
2. [Failover Strategies](#-failover-strategies)
3. [Failover Detection — How to Know Something Died](#-failover-detection--how-to-know-something-died)
4. [Failover Challenges — The Hard Parts](#-failover-challenges--the-hard-parts)
5. [Database Failover Deep Dive](#-database-failover-deep-dive)
6. [DNS Failover](#-dns-failover)
7. [Real-World Failover Architectures](#-real-world-failover-architectures)
8. [Java Implementation](#-java-implementation)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Failover?

```
╔══════════════════════════════════════════════════════════════════╗
║  FAILOVER = Automatically switching to a backup system         ║
║             when the primary system fails.                     ║
║                                                                ║
║  Primary dies → Detect failure → Switch to backup → Continue   ║
╚══════════════════════════════════════════════════════════════════╝

Real-life analogy:
  🏥 Hospital generators:
     City power fails → Detect (< 1 sec) → Generator starts → Lights back on
     
  ✈️ Airplane engines:
     Engine 1 fails → Engine 2 provides full thrust → Safe landing
     
  🏦 ATM network:
     Primary processor down → Backup processor takes over → Transactions continue
```

### The Failover Timeline

```
         Detection     Switchover     Recovery
            │              │              │
  ──────────┼──────────────┼──────────────┼──────────────►
  │ Normal  │  Detecting   │  Switching   │  Normal      │
  │ ops     │  failure     │  to backup   │  ops (on B)  │
  
  ◄── RTO ──────────────────────────────►
  (Recovery Time Objective = total downtime)
  
  GOAL: Make this entire timeline as SHORT as possible
        Best-in-class: < 30 seconds (AWS Multi-AZ RDS)
        Good:          < 5 minutes
        Acceptable:    < 15 minutes
        Bad:           > 1 hour (manual failover)
```

---

## 🎯 Failover Strategies

### Strategy 1: Active-Passive (Cold/Warm Standby)

```
ACTIVE-PASSIVE:
  
  Normal operation:
  ┌──────────┐  heartbeat  ┌──────────┐
  │ PRIMARY  │────────────►│ STANDBY  │
  │ (active) │  "I'm alive"│ (passive)│
  │ handles  │  every 5sec │ waiting  │
  │ all traffic│            │ idle     │
  └──────────┘             └──────────┘
  
  After failover:
  ┌──────────┐             ┌──────────┐
  │ PRIMARY  │    ╳        │ STANDBY  │
  │  (DEAD)  │             │ (NOW     │
  │          │             │  ACTIVE!)│
  └──────────┘             └──────────┘

  Pros: Simple, lower cost (standby uses fewer resources)
  Cons: Wasted resources (standby doing nothing), longer failover time
  
  Use when: Cost matters more than speed (< 99.99% SLA)
```

### Strategy 2: Active-Active (Hot Standby)

```
ACTIVE-ACTIVE:

  Normal operation:
  ┌──────────┐             ┌──────────┐
  │ SERVER A │             │ SERVER B │
  │ (active) │◄───────────►│ (active) │
  │ 50% load │  sync data  │ 50% load │
  └──────────┘             └──────────┘
       │                        │
       └────────┬───────────────┘
                │
         ┌──────▼──────┐
         │ Load Balancer│
         └─────────────┘
  
  After failure of A:
  ┌──────────┐             ┌──────────┐
  │ SERVER A │    ╳        │ SERVER B │
  │  (DEAD)  │             │ (active) │
  │          │             │ 100% load│
  └──────────┘             └──────────┘

  Pros: Near-zero failover time, no wasted resources
  Cons: Complex (data sync), expensive, split-brain risk
  
  Use when: Zero-downtime requirement (> 99.99% SLA)
```

### Strategy 3: N+1 Redundancy

```
N+1 means: You need N servers to handle load + 1 extra for failover

  Example: You need 4 servers for full load → Deploy 5 servers
  
  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐
  │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │  ← 5 servers (N+1)
  └───┘ └───┘ └───┘ └───┘ └───┘
  Each handles 20% of load
  
  If server 3 dies:
  ┌───┐ ┌───┐ ┌───┐ ┌───┐
  │ 1 │ │ 2 │ │ 4 │ │ 5 │  ← 4 servers handle 25% each
  └───┘ └───┘ └───┘ └───┘
  Still within capacity! ✅

  For critical systems: N+2 (survives 2 simultaneous failures)
```

### Comparison Table

| Strategy | Failover Time | Cost | Complexity | Data Loss Risk |
|----------|--------------|------|-----------|---------------|
| Active-Passive (Cold) | 5-30 min | 💰 Low | Low | Medium |
| Active-Passive (Warm) | 30s-5 min | 💰💰 Medium | Medium | Low |
| Active-Active (Hot) | < 5 sec | 💰💰💰 High | High | Very Low |
| N+1 | Near-zero | 💰💰 Medium | Medium | None |

---

## 🔍 Failover Detection — How to Know Something Died

```
┌─────────────────────────────────────────────────────────────────┐
│                 DETECTION MECHANISMS                             │
├─────────────────┬───────────────────────────────────────────────┤
│  Heartbeat      │  Primary sends "I'm alive" every N seconds.  │
│  (Pull-based)   │  If 3 heartbeats missed → declared dead.     │
│                 │  Simple but can have false positives.         │
├─────────────────┼───────────────────────────────────────────────┤
│  Health Check   │  Monitor actively probes the primary:        │
│  (Push-based)   │  HTTP /health → 200 OK means alive.          │
│                 │  More reliable than heartbeat alone.          │
├─────────────────┼───────────────────────────────────────────────┤
│  Consensus      │  Multiple monitors vote on "is it dead?"      │
│  (Quorum)       │  Majority must agree → avoids false positives.│
│                 │  Used by: ZooKeeper, etcd, Consul             │
├─────────────────┼───────────────────────────────────────────────┤
│  Leader Election│  Nodes elect a leader using Raft/Paxos.       │
│                 │  If leader dies → new election → new leader.  │
│                 │  Used by: Kafka, Redis Sentinel, CockroachDB  │
└─────────────────┴───────────────────────────────────────────────┘
```

### 🎮 The Detection Dilemma

```
TOO SENSITIVE:                    TOO SLOW:
  Timeout = 1 second               Timeout = 60 seconds
  ↓                                ↓
  Network hiccup = false alarm     Real failure = 60 sec downtime
  ↓                                ↓
  Unnecessary failover!            Users suffer for a full minute!
  (split-brain risk 😱)           
  
SWEET SPOT:
  - 3 missed heartbeats (e.g., 3 × 5sec = 15 sec detection)
  - Confirm with secondary check before failover
  - Different timeout for different components
```

---

## 🧩 Failover Challenges — The Hard Parts

### Challenge 1: Split-Brain Problem 🧠

```
THE NIGHTMARE SCENARIO:

  Network partition between primary and standby:
  
  ┌──────────────────┐    ╳    ┌──────────────────┐
  │    DATA CENTER A  │  (cut)  │    DATA CENTER B  │
  │                  │         │                  │
  │  Primary DB      │         │  Standby DB      │
  │  "I'm primary!"  │         │  "Primary is dead!│
  │                  │         │   I'm primary now!"│
  └──────────────────┘         └──────────────────┘
  
  RESULT: TWO primaries accepting writes! 💀
  Different clients write to different "primaries"
  Data diverges → INCONSISTENCY → CORRUPTION
  
  SOLUTIONS:
  1. Fencing (STONITH: Shoot The Other Node In The Head)
     → Force-kill the old primary before promoting standby
  2. Quorum-based consensus (odd number of nodes vote)
     → Only majority side can be primary
  3. Lease-based leadership (primary must renew lease)
     → If lease expires, primary MUST stop accepting writes
```

### Challenge 2: Data Loss During Failover

```
Async replication gap:

  Primary:  [Write A] [Write B] [Write C] [Write D] ← CRASH!
  Standby:  [Write A] [Write B]                       ← Promoted
  
  LOST: Write C and Write D! 😰
  
  This is the RPO (Recovery Point Objective):
    RPO = Maximum acceptable data loss during failover
    
  Solutions:
    - Synchronous replication (RPO = 0, but slower)
    - Semi-synchronous (at least 1 replica has the data)
    - Transaction log shipping (minimize gap)
```

---

## 🗄️ Database Failover Deep Dive

### MySQL/PostgreSQL Failover

```
                    Automatic Failover (e.g., Patroni for PostgreSQL)
                    
  BEFORE:
  ┌─────────────┐     async/sync      ┌─────────────┐
  │  Primary    │─────────────────────►│  Replica 1  │
  │  (writes)   │                      └─────────────┘
  └─────────────┘─────────────────────►┌─────────────┐
                                       │  Replica 2  │
                                       └─────────────┘
         │
    ┌────▼────┐
    │  etcd   │  (stores leader info)
    └─────────┘
  
  AFTER PRIMARY DIES:
  ┌─────────────┐                      ┌─────────────┐
  │  Primary    │         ╳            │  Replica 1  │ ← NEW PRIMARY!
  │  (DEAD)     │                      │  (promoted) │
  └─────────────┘                      └─────────────┘
                                       ┌─────────────┐
                                       │  Replica 2  │ ← now follows
                                       └─────────────┘    Replica 1
  
  Timeline: Detection (10s) + Election (5s) + Promotion (5s) = ~20s total
```

### Redis Sentinel Failover

```
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │Sentinel 1│  │Sentinel 2│  │Sentinel 3│  (monitors)
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       │              │              │
  ┌────▼─────────────▼──────────────▼────┐
  │                                       │
  │  ┌────────┐    ┌────────┐            │
  │  │ Master │───►│Replica │            │
  │  └────────┘    └────────┘            │
  │                                       │
  └───────────────────────────────────────┘
  
  Failover process:
  1. Sentinel 1 can't reach Master → marks it as "SDOWN" (subjective down)
  2. Sentinel 2 also can't reach → quorum agrees → "ODOWN" (objective down)
  3. Sentinels elect a leader sentinel
  4. Leader promotes best replica to Master
  5. Other replicas reconfigured to follow new Master
  6. Clients notified of new Master address
```

---

## 🌐 DNS Failover

```
DNS-based failover (used by many CDNs):

  Normal:
    example.com → A record → 1.2.3.4 (US-East server)
    
  After US-East failure:
    example.com → A record → 5.6.7.8 (US-West server)
    
  ⚠️ PROBLEM: DNS caching!
    - ISPs cache DNS for hours (even days!)
    - TTL (Time To Live) must be set LOW for fast failover
    - Low TTL = more DNS lookups = slightly higher latency
    
  SOLUTION: Use low TTL (30-60 seconds) + Anycast routing
    - CloudFlare, Route53 health checks change DNS automatically
    - Anycast routes to nearest healthy endpoint at network level
```

---

## 🏢 Real-World Failover Architectures

### Netflix: Multi-Region Active-Active
```
  US-East-1           US-West-2           EU-West-1
  ┌─────────┐        ┌─────────┐        ┌─────────┐
  │ Full    │◄──────►│ Full    │◄──────►│ Full    │
  │ Stack   │  sync   │ Stack   │  sync   │ Stack   │
  └─────────┘        └─────────┘        └─────────┘
  
  - Each region handles its own traffic independently
  - If US-East fails → DNS routes all US traffic to US-West
  - Failover time: < 7 minutes (fully automated)
  - Regular "region evacuation" drills to test failover
```

### AWS RDS Multi-AZ Failover
```
  Availability Zone A         Availability Zone B
  ┌───────────────────┐      ┌───────────────────┐
  │                   │      │                   │
  │  ┌─────────────┐ │      │  ┌─────────────┐ │
  │  │ RDS Primary │──────────►│ RDS Standby │ │
  │  │             │ │sync   │  │             │ │
  │  └─────────────┘ │      │  └─────────────┘ │
  │                   │      │                   │
  └───────────────────┘      └───────────────────┘
  
  Automatic failover: ~30-60 seconds
  No data loss (synchronous replication)
  DNS endpoint stays the same (transparent to app)
```

---

## 💻 Java Implementation

### Spring Boot with Failover Data Source

```java
@Configuration
public class FailoverConfig {
    
    @Bean
    public DataSource failoverDataSource(
            @Value("${db.primary.url}") String primaryUrl,
            @Value("${db.secondary.url}") String secondaryUrl) {
        
        HikariConfig primary = new HikariConfig();
        primary.setJdbcUrl(primaryUrl);
        primary.setConnectionTimeout(5000); // Fast failure detection
        
        HikariConfig secondary = new HikariConfig();
        secondary.setJdbcUrl(secondaryUrl);
        
        return new FailoverDataSource(
            new HikariDataSource(primary),
            new HikariDataSource(secondary)
        );
    }
}

public class FailoverDataSource implements DataSource {
    private final DataSource primary;
    private final DataSource secondary;
    private final AtomicBoolean usePrimary = new AtomicBoolean(true);
    
    @Override
    public Connection getConnection() throws SQLException {
        if (usePrimary.get()) {
            try {
                Connection conn = primary.getConnection();
                if (conn.isValid(2)) return conn;
            } catch (SQLException e) {
                log.error("Primary DB failed, switching to secondary", e);
                usePrimary.set(false);
                scheduleRecoveryCheck();
            }
        }
        return secondary.getConnection();
    }
    
    private void scheduleRecoveryCheck() {
        // Periodically check if primary recovers
        scheduler.scheduleAtFixedRate(() -> {
            try (Connection conn = primary.getConnection()) {
                if (conn.isValid(2)) {
                    usePrimary.set(true);
                    log.info("Primary DB recovered! Switching back.");
                }
            } catch (SQLException ignored) {}
        }, 30, 30, TimeUnit.SECONDS);
    }
}
```

### Service-Level Failover with Resilience4j

```java
@Service
public class PaymentServiceWithFailover {
    
    @CircuitBreaker(name = "primaryPayment", fallbackMethod = "fallbackPayment")
    public PaymentResult processPayment(PaymentRequest request) {
        return primaryPaymentGateway.charge(request);
    }
    
    // Failover to secondary payment provider
    private PaymentResult fallbackPayment(PaymentRequest request, Exception ex) {
        log.warn("Primary payment gateway down, using failover: {}", ex.getMessage());
        return secondaryPaymentGateway.charge(request);
    }
}
```

---

## ⚠️ Common Pitfalls

| Pitfall | Why It's Dangerous | Fix |
|---------|-------------------|-----|
| 🔴 Never testing failover | Untested failover = no failover | Monthly failover drills |
| 🔴 Split-brain ignorance | Two primaries = data corruption | Fencing + quorum |
| 🔴 Failback without validation | Returning to recovered primary with stale data | Verify data integrity first |
| 🔴 DNS TTL too high | Failover takes hours due to caching | Set TTL to 30-60 seconds |
| 🟡 No alerting on failover | Team doesn't know failover happened | Alert immediately on promotion |
| 🟡 Capacity planning | Standby can't handle full load | Size standby for 100% traffic |

---

## 🎮 Mini Challenge

### 🧩 Design a Failover System

You're building a banking application. Requirements:
- Zero data loss (RPO = 0)
- Maximum 30 seconds downtime (RTO = 30s)
- Must survive entire data center failure
- 10,000 transactions per second

**Design questions:**
1. Active-Active or Active-Passive? Why?
2. Synchronous or asynchronous replication?
3. How do you prevent split-brain?
4. What's your failover detection mechanism?
5. How do you handle in-flight transactions during failover?

---

## ❓ Interview Q&A

**Q1: What is failover and why is it important?**
> Failover is the automatic switching from a failed primary system to a standby backup. It's critical for high availability — without it, any component failure means complete system outage until manual intervention.

**Q2: Explain the difference between Active-Active and Active-Passive failover.**
> Active-Passive: standby sits idle, takes over when primary fails (simpler, cheaper, but wasteful). Active-Active: both nodes handle traffic simultaneously, if one dies the other absorbs its load (faster failover, better resource usage, but complex data synchronization).

**Q3: What is the split-brain problem and how do you solve it?**
> Split-brain occurs when network partition causes both nodes to think the other is dead, so both promote themselves to primary. Solutions: (1) Quorum/voting (odd number of nodes), (2) STONITH fencing (force-kill ambiguous node), (3) Lease-based leadership (must hold valid lease to be primary).

**Q4: What are RPO and RTO?**
> RPO (Recovery Point Objective) = maximum acceptable data loss, measured in time. RTO (Recovery Time Objective) = maximum acceptable downtime. Example: RPO=0 means zero data loss (needs synchronous replication), RTO=30s means system must recover within 30 seconds.

**Q5: How does AWS RDS handle failover?**
> Multi-AZ RDS uses synchronous replication to a standby in a different AZ. On primary failure, AWS automatically updates the DNS endpoint to point to the standby (promoted to primary). Total failover time: 30-120 seconds. Application doesn't need code changes since the endpoint stays the same.

---

## 🔗 Related Topics
- [SPOF](./SPOF.md) — Failover eliminates single points of failure
- [Availability](./Availability.md) — Failover is the mechanism that delivers high availability
- [Reliability](./Reliability.md) — Reliable systems require working failover
- [Replication](./Replication.md) — Data replication enables failover
- [Consistent Hashing](./Consistent_Hashing.md) — Consistent hashing enables seamless failover in distributed caches

---

*"Everyone has a failover plan until they actually need to fail over." — Adapted from Mike Tyson* 🥊
