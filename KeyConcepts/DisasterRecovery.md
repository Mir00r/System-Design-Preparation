# 🛡️ Disaster Recovery & Business Continuity

> *"On October 2, 2021, Facebook went offline for 6 HOURS — all of Facebook, Instagram, WhatsApp, Messenger. 3.5 billion people couldn't communicate. The cause? A routine BGP configuration change that disconnected their data centers from the internet. Even the engineers couldn't access their own systems to fix it (the door badges ran on Facebook's network!). Disaster recovery isn't about IF something will go catastrophically wrong — it's about WHEN."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Reliability](../../KeyConcepts/Reliability.md), [Failover](../../KeyConcepts/Failover.md)

---

## 📋 Table of Contents
1. [Core Concepts: RTO & RPO](#-core-concepts-rto--rpo)
2. [Disaster Recovery Strategies](#-disaster-recovery-strategies)
3. [Multi-Region Architecture](#-multi-region-architecture)
4. [Data Backup & Recovery](#-data-backup--recovery)
5. [Chaos Engineering](#-chaos-engineering)
6. [Runbooks & Incident Response](#-runbooks--incident-response)
7. [Java/Spring Boot DR Patterns](#-javaspring-boot-dr-patterns)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## ⏱️ Core Concepts: RTO & RPO

```
TWO CRITICAL METRICS:

  RPO (Recovery Point Objective):
  "How much DATA can we afford to lose?"
  
  ──── Data Loss ────┤ Disaster! ├──── Recovery ────
  Last backup        │           │     System back up
  ◄─── RPO ─────────►           │
  (e.g., 1 hour)               │
  "We'll lose at most 1 hour   │
   of data"                     │
   
  RPO Examples:
  • Banking: RPO = 0 (ZERO data loss — synchronous replication!)
  • E-commerce: RPO = 5 minutes (recent orders might be lost)
  • Analytics: RPO = 1 hour (can re-process from source)
  • Dev environment: RPO = 24 hours (daily backups fine)

  RTO (Recovery Time Objective):
  "How quickly must we be BACK ONLINE?"
  
  ──── Running ────┤ Disaster! ├──── Downtime ────┤ Recovered!
                   │           │                   │
                   │◄───────── RTO ──────────────►│
                   │  (e.g., 15 minutes)          │
                   │  "System back in 15 min"     │
   
  RTO Examples:
  • Payment systems: RTO = 0 (active-active, no downtime!)
  • SaaS product: RTO = 15 minutes (users notice > 5 min)
  • Internal tools: RTO = 4 hours (during business hours)
  • Batch processing: RTO = 24 hours (nightly job can wait)

COST vs. RTO/RPO:
  ┌────────────────────────────────────────────────────────────┐
  │  $$$$$  │  RTO = 0, RPO = 0    │  Active-Active multi-region│
  │  $$$$   │  RTO = min, RPO = min │  Hot standby (auto-failover)│
  │  $$$    │  RTO = hrs, RPO = min │  Warm standby (manual)     │
  │  $$     │  RTO = hrs, RPO = hrs │  Pilot light (minimal)     │
  │  $      │  RTO = day, RPO = day │  Backup & restore          │
  └────────────────────────────────────────────────────────────┘
  
  Lower RTO/RPO = MUCH more expensive!
  Most companies: different tiers for different services.
```

---

## 🏗️ Disaster Recovery Strategies

```
STRATEGY 1: BACKUP & RESTORE (Cold)
  RTO: hours-days | RPO: hours | Cost: $ (cheapest!)
  
  How: Periodic backups to different region. On disaster: restore.
  
  ┌──────────┐   backup    ┌──────────────┐
  │ Primary  │ ──────────► │  S3 Backup   │
  │ Region   │  (every 1h) │  (other      │
  └──────────┘             │   region)    │
                           └──────────────┘
                           On disaster: spin up new infra, restore!
  
  ✅ Cheapest option
  ❌ Long recovery time (provision infra + restore data!)
  ❌ Up to 1 hour of data loss (since last backup)
  
  Good for: Dev/staging, non-critical internal apps

STRATEGY 2: PILOT LIGHT (Minimal)
  RTO: 30min-hours | RPO: minutes | Cost: $$
  
  How: Core infrastructure running (DB replicated), app servers OFF.
  
  Primary Region:          DR Region:
  ┌──────────────┐        ┌──────────────┐
  │ App Servers  │        │ (OFF - not   │
  │ (running)    │        │  running!)   │
  ├──────────────┤        ├──────────────┤
  │ Database     │──repl─►│ Database     │ ← Always synced!
  │ (primary)    │        │ (replica)    │
  └──────────────┘        └──────────────┘
  
  On disaster: Start app servers in DR! Database already has data!
  
  ✅ Faster than cold (DB already replicated!)
  ❌ Still need to start app servers (5-30 minutes)
  Good for: Medium-importance production services

STRATEGY 3: WARM STANDBY
  RTO: minutes | RPO: seconds | Cost: $$$
  
  How: Full environment running at MINIMUM scale in DR.
  
  Primary Region:          DR Region:
  ┌──────────────┐        ┌──────────────┐
  │ App × 10     │        │ App × 2      │ ← Running! (minimal)
  ├──────────────┤        ├──────────────┤
  │ Database     │──repl─►│ Database     │ ← Sync replica!
  │ (primary)    │        │ (read-only)  │
  └──────────────┘        └──────────────┘
  
  On disaster: Scale UP DR region + promote DB + route traffic!
  
  ✅ Minutes to recover (already running, just scale!)
  ❌ Paying for always-on infrastructure (even if unused)
  Good for: Important production services (SaaS, APIs)

STRATEGY 4: ACTIVE-ACTIVE (Hot)
  RTO: ~0 | RPO: 0 | Cost: $$$$$ (most expensive!)
  
  How: BOTH regions serving traffic simultaneously!
  
  Region A:               Region B:
  ┌──────────────┐        ┌──────────────┐
  │ App × 10     │◄─────►│ App × 10     │ ← Both active!
  ├──────────────┤        ├──────────────┤
  │ Database     │◄─repl─►│ Database     │ ← Multi-primary!
  │ (primary)    │        │ (primary)    │
  └──────────────┘        └──────────────┘
       ▲                        ▲
       │    DNS/Global LB       │
       └────────┬───────────────┘
                │
            Users (routed to nearest!)
  
  On disaster: Traffic automatically routes to surviving region!
  No manual intervention! Users might not even notice!
  
  ✅ Zero downtime, zero data loss
  ❌ EXPENSIVE (2x infrastructure!)
  ❌ Complex: conflict resolution for multi-primary writes!
  Good for: Payment systems, messaging, critical SaaS
```

---

## 🌍 Multi-Region Architecture

```
GLOBAL TRAFFIC ROUTING:

  DNS-based failover (Route 53 health checks):
  
  api.example.com → Health check: is US healthy?
    YES → route to US endpoint (closest!)
    NO  → route to EU endpoint (failover!)
  
  Failover detection time: 30-60 seconds (DNS TTL!)
  For faster: use Global Load Balancer (Cloudflare, AWS Global Accelerator)
  → Failover in < 10 seconds!

DATA REPLICATION PATTERNS:

  Pattern 1: SINGLE-PRIMARY (simpler)
  ┌───────────────────────────────────────┐
  │  US (Primary)      EU (Replica)       │
  │  ┌──────────┐     ┌──────────┐       │
  │  │  Writes  │────►│  Reads   │       │
  │  │  & Reads │     │  only!   │       │
  │  └──────────┘     └──────────┘       │
  │                                       │
  │  Failover: Promote EU to primary      │
  │  Risk: Seconds of unreplicated data   │
  └───────────────────────────────────────┘
  
  Pattern 2: MULTI-PRIMARY (complex!)
  ┌───────────────────────────────────────┐
  │  US (Primary)      EU (Primary)       │
  │  ┌──────────┐     ┌──────────┐       │
  │  │  Writes  │◄───►│  Writes  │       │
  │  │  & Reads │     │  & Reads │       │
  │  └──────────┘     └──────────┘       │
  │                                       │
  │  Challenge: CONFLICTS!                │
  │  Same user updated in both regions    │
  │  simultaneously → which write wins?   │
  │  Solutions: Last-Write-Wins, CRDTs    │
  └───────────────────────────────────────┘

WHICH SERVICES TO REPLICATE:
  Must replicate: Authentication, core business logic, database
  Can be regional: Analytics, batch jobs, search (rebuild from primary)
  Don't need: Dev tools, internal dashboards, non-critical services
```

---

## 💾 Data Backup & Recovery

```
BACKUP TYPES:

  Full Backup: Copy EVERYTHING (large, slow, complete)
  Incremental: Only changes since last backup (fast, small)
  Differential: Changes since last FULL backup (medium)
  
  Strategy: Full weekly + Incremental hourly
  
  Monday 2AM: Full backup (100 GB)
  Monday 3AM: Incremental (500 MB changes)
  Monday 4AM: Incremental (300 MB changes)
  ...
  Tuesday 2AM: Full backup (101 GB)

  Recovery: Restore full + replay all incrementals since!

DATABASE BACKUP STRATEGIES:

  PostgreSQL:
  • pg_dump: Logical backup (SQL statements). Slow but portable.
  • pg_basebackup: Physical backup (file copy). Fast, point-in-time!
  • WAL archiving: Continuous, point-in-time recovery to any second!
    → RPO ≈ 0! Can recover to exact timestamp of failure!
  
  MongoDB:
  • mongodump: Logical backup
  • Filesystem snapshots (EBS snapshots on AWS)
  • Oplog-based: continuous replication (like WAL!)

THE 3-2-1 RULE:
  3 copies of your data
  2 different storage media (disk + cloud)
  1 offsite (different geographic region!)
  
  Example:
  Copy 1: Production database (live!)
  Copy 2: Replica in same region (real-time sync)
  Copy 3: Backup in different region (S3 cross-region replication!)

TESTING BACKUPS:
  A backup you've never tested restoring is NOT a backup!
  
  Monthly drill:
  1. Restore from backup to test environment
  2. Verify data integrity (row counts, checksums)
  3. Run application against restored DB
  4. Measure actual RTO (how long did it take?)
  5. Document and improve!
```

---

## 🐒 Chaos Engineering

```
"THE BEST TIME TO FIND OUT YOUR DR PLAN DOESN'T WORK IS NOT 
DURING AN ACTUAL DISASTER!" — Netflix (inventors of Chaos Engineering)

NETFLIX'S SIMIAN ARMY:
  ┌────────────────────────────────────────────────────────────┐
  │  Tool              │  What It Does                          │
  ├────────────────────────────────────────────────────────────┤
  │  Chaos Monkey      │  Kills random production instances!    │
  │  Chaos Kong        │  Takes down entire AWS region!         │
  │  Latency Monkey    │  Adds artificial network delays        │
  │  Conformity Monkey │  Finds instances not following rules   │
  │  Doctor Monkey     │  Health checks + auto-remediation      │
  └────────────────────────────────────────────────────────────┘

CHAOS ENGINEERING PRINCIPLES:
  1. Define "steady state" (what does normal look like?)
  2. Hypothesize: "If X fails, system should do Y"
  3. Introduce failure: kill server, add latency, corrupt data
  4. Observe: Did the system behave as expected?
  5. Fix: Found a gap? Fix it! Then test again!

WHAT TO TEST:
  • Kill a server → does auto-scaling replace it?
  • Database failover → does app reconnect automatically?
  • Network partition → do services degrade gracefully?
  • Disk full → does alerting fire? Does cleanup work?
  • DNS failure → does caching prevent total outage?
  • Dependency down → does circuit breaker activate?

GAME DAY EXERCISES:
  Schedule quarterly "game days":
  • Announce to team: "Today we're simulating region failure"
  • Actually fail over to DR region
  • Time the recovery
  • Document what went wrong
  • Fix issues, improve runbooks
```

---

## 💻 Java/Spring Boot DR Patterns

### Database Failover Configuration

```java
@Configuration
public class DatabaseFailoverConfig {
    
    /**
     * Multi-datasource with automatic failover.
     * Primary fails → switch to replica automatically!
     */
    @Bean
    public DataSource dataSource() {
        HikariDataSource primary = createDataSource(
            "jdbc:postgresql://primary-db.us-east-1:5432/myapp");
        HikariDataSource replica = createDataSource(
            "jdbc:postgresql://replica-db.eu-west-1:5432/myapp");
        
        // Use routing datasource that switches on failure
        return new FailoverDataSource(primary, replica);
    }
}

@Component
public class FailoverDataSource extends AbstractRoutingDataSource {
    
    private final DataSource primary;
    private final DataSource secondary;
    private final AtomicBoolean primaryHealthy = new AtomicBoolean(true);
    
    @Scheduled(fixedRate = 5000) // Health check every 5s
    public void checkPrimaryHealth() {
        try {
            primary.getConnection().isValid(2); // 2s timeout
            primaryHealthy.set(true);
        } catch (Exception e) {
            log.error("Primary database unreachable! Failing over...");
            primaryHealthy.set(false);
            alertService.trigger("DB_FAILOVER_ACTIVATED");
        }
    }
    
    @Override
    protected Object determineCurrentLookupKey() {
        return primaryHealthy.get() ? "primary" : "secondary";
    }
}
```

### Circuit Breaker for External Dependencies

```java
@Service
public class PaymentServiceWithDR {
    
    @Autowired private PaymentGateway primaryGateway;  // Stripe
    @Autowired private PaymentGateway fallbackGateway; // Adyen (backup!)
    
    @CircuitBreaker(name = "payment", fallbackMethod = "processFallback")
    @Retry(name = "payment", maxRetries = 2)
    public PaymentResult processPayment(PaymentRequest request) {
        return primaryGateway.charge(request);
    }
    
    /**
     * Fallback: primary payment gateway is down!
     * Use backup gateway (different provider for redundancy!)
     */
    public PaymentResult processFallback(PaymentRequest request, Exception e) {
        log.warn("Primary payment gateway failed, using fallback", e);
        alertService.trigger("PAYMENT_GATEWAY_FAILOVER");
        
        // Use different payment provider!
        return fallbackGateway.charge(request);
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Scenario: Design DR for an E-commerce Platform

Your e-commerce platform (US-East primary) must survive a complete region failure. Requirements: RPO < 1 minute, RTO < 5 minutes for the ordering system. What's your strategy?

<details>
<summary>🔑 Answer</summary>

**Architecture:**
```
US-East (Primary):        EU-West (DR):
┌────────────────┐       ┌────────────────┐
│ App Servers ×20│       │ App Servers ×5 │ ← Warm standby!
│ (auto-scaled)  │       │ (min running)  │
├────────────────┤       ├────────────────┤
│ PostgreSQL     │──sync─►│ PostgreSQL     │ ← Sync replication!
│ (primary)      │  repl  │ (hot standby) │   (RPO ≈ 0!)
├────────────────┤       ├────────────────┤
│ Redis Cluster  │──async►│ Redis Cluster  │ ← Async (cache is lossy)
├────────────────┤       ├────────────────┤
│ Elasticsearch  │       │ Elasticsearch  │ ← Rebuild from DB
└────────────────┘       └────────────────┘
```

**Failover steps (automated!):**
1. Health check fails for US-East (3 consecutive failures, 15 seconds)
2. Automated runbook triggers:
   a. Promote EU PostgreSQL to primary (< 30 seconds)
   b. Scale EU app servers from 5 → 20 (< 2 minutes)
   c. Update DNS/Global LB to route to EU (< 30 seconds)
   d. Invalidate CDN caches pointing to old region
3. Alert on-call team (parallel to automated failover!)
4. Total RTO: ~3 minutes ✅

**RPO guarantee:** Synchronous replication → zero data loss for committed transactions. Trade-off: 50-100ms added write latency (cross-region sync).
</details>

---

## ❓ Interview Q&A

**Q1: What's the difference between High Availability and Disaster Recovery?**
> HA handles component failures WITHIN a region (server dies → auto-replace, DB failover to replica). DR handles REGION-LEVEL failures (entire data center down, natural disaster, massive network outage). HA is automatic, sub-second failover. DR may be manual or semi-automatic, minutes-to-hours recovery. You need BOTH: HA for daily reliability + DR for catastrophic events.

**Q2: How do you achieve RPO = 0 (zero data loss)?**
> Synchronous replication: every write must be confirmed by the DR replica before returning success to the client. Trade-off: increased write latency (cross-region round trip: 50-200ms). Implementation: PostgreSQL synchronous_commit=remote_apply, or database-level multi-region writes (CockroachDB, Spanner). Alternative: if slightly lossy is OK, async replication + WAL shipping gives RPO of seconds.

**Q3: How do you test your disaster recovery plan?**
> (1) Chaos engineering: regularly inject failures (kill servers, simulate network partitions), (2) Quarterly game days: actually trigger full region failover with the team, (3) Backup restore tests: monthly restore from backup, verify data integrity, (4) Runbook rehearsals: walk through procedures without actual failure, (5) Monitoring drill: verify alerting works end-to-end. The #1 reason DR fails in real disasters: it was never actually tested.

**Q4: How do you handle database failover without losing transactions?**
> For synchronous replication: no transactions are lost (by definition — writes wait for DR confirmation). For async: some in-flight transactions may be lost. Mitigation: (1) Application-level idempotency (retry without side effects), (2) WAL-based recovery (replay un-shipped WAL segments when old primary recovers), (3) Queue pending writes during failover window and replay, (4) Accept the loss for non-critical operations (analytics, cache writes).

**Q5: Active-active vs active-passive — when to use each?**
> Active-active: both regions serve traffic simultaneously. Best when: zero downtime required, traffic is globally distributed anyway, data model supports multi-primary (CRDTs, last-write-wins OK). Challenges: conflict resolution, increased complexity. Active-passive: one region handles traffic, other waits. Best when: strong consistency required, simpler to reason about, cost matters. Most companies start active-passive, evolve to active-active for critical paths only.

---

## 🔗 Related Topics
- [Reliability](../../KeyConcepts/Reliability.md) — Building reliable systems
- [Failover](../../KeyConcepts/Failover.md) — Automatic failover mechanisms
- [SPOF](../../KeyConcepts/SPOF.md) — Eliminating single points of failure
- [Circuit Breaker](../../BuildingBlocks/CircuitBreaker.md) — Graceful degradation

---

*"Everyone has a disaster recovery plan until they actually need it. Then they discover the plan assumes the person who wrote it is available, the wiki is accessible, and the backup actually has data in it. None of these things are true during a real disaster." — Every On-Call Engineer at 3 AM* 🛡️
