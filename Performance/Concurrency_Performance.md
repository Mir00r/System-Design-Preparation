# 🧵 Concurrency & Threading Performance: Parallelism Done Right

> *"Concurrency is not parallelism. Concurrency is dealing with multiple things at once. Parallelism is DOING multiple things at once. You need both — but incorrectly done, they create the hardest bugs in software."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [JVM Tuning](./JVM_Tuning.md), Java threading basics

---

## 📋 Table of Contents
1. [Thread Pool Sizing Strategy](#-thread-pool-sizing)
2. [Virtual Threads (Java 21+)](#-virtual-threads)
3. [Async Patterns for Performance](#-async-patterns)
4. [Lock Contention & Solutions](#-lock-contention)
5. [Common Concurrency Bottlenecks](#-common-bottlenecks)
6. [Interview Q&A](#-interview-qa)
7. [Boss Battle](#-boss-battle)

---

## 🧵 Thread Pool Sizing

```
THE FUNDAMENTAL QUESTION: How many threads should I have?

FOR CPU-BOUND WORK (computation, no I/O):
  Optimal threads = Number of CPU cores
  Why: More threads = context switching overhead with no benefit
  Example: Image processing, encryption, sorting

FOR I/O-BOUND WORK (DB calls, HTTP calls, file I/O):
  Optimal threads = CPU cores × (1 + Wait_Time / Service_Time)
  Why: While one thread waits for I/O, others can use the CPU
  
  Example: 4 cores, 200ms DB wait, 10ms processing
  threads = 4 × (1 + 200/10) = 4 × 21 = 84 threads
  
  Simplified rule: threads = CPU cores × 2 for typical web apps
  (because average I/O wait ≈ service time for most APIs)

FOR MIXED WORKLOADS:
  ❌ One pool for everything (CPU tasks starve during I/O burst)
  ✅ Separate pools: 
     - Small CPU pool (cores × 1)
     - Large I/O pool (cores × 10-20)
     
TOMCAT DEFAULT = 200 threads
  Fine for: typical web APIs with moderate I/O
  Too many for: CPU-heavy processing (context switch overhead)
  Too few for: high-concurrency WebSocket/streaming apps
```

### Thread Pool Configuration (Spring Boot)

```java
@Configuration
public class ThreadPoolConfig {
    
    // For I/O-bound async tasks (DB calls, HTTP clients)
    @Bean("ioExecutor")
    public Executor ioExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("io-");
        executor.setRejectedExecutionHandler(new CallerRunsPolicy());
        return executor;
    }
    
    // For CPU-bound tasks (calculations, transformations)
    @Bean("cpuExecutor")
    public Executor cpuExecutor() {
        int cores = Runtime.getRuntime().availableProcessors();
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(cores);
        executor.setMaxPoolSize(cores); // Never exceed cores for CPU work!
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("cpu-");
        return executor;
    }
}
```

---

## 🚀 Virtual Threads (Java 21+)

```
TRADITIONAL THREADS (Platform Threads):
  1 Java thread = 1 OS thread = ~1MB stack memory
  10,000 threads = 10GB memory + OS scheduling overhead
  Limit: ~10K concurrent connections before running out of threads

VIRTUAL THREADS (Project Loom):
  1 Virtual thread = ~few KB memory (1000x lighter!)
  1,000,000 concurrent virtual threads = totally fine!
  Mounted onto platform threads only when CPU work needed
  Unmounted during blocking I/O (await DB, HTTP, etc.)

GAME CHANGER FOR I/O-BOUND APPS:
  Before: 200 Tomcat threads → max 200 concurrent requests
  After:  1,000,000 virtual threads → limited only by CPU/memory!

ENABLE IN SPRING BOOT 3.2+:
  spring.threads.virtual.enabled=true
  That's it! All request handling uses virtual threads.
```

### When NOT to Use Virtual Threads

```
❌ CPU-bound work (virtual threads don't help — still need real CPU)
❌ Synchronized blocks (pins the carrier thread — defeats purpose!)
❌ ThreadLocal-heavy code (each virtual thread gets own copy — memory!)
❌ When you need predictable thread count (prefer bounded pools)

✅ PERFECT FOR:
  - HTTP request handling (waiting for DB, downstream services)
  - Batch processing with many concurrent I/O operations
  - WebSocket connections (1M simultaneous connections!)
  - File I/O operations
```

---

## ⚡ Async Patterns

### CompletableFuture — Compose Async Operations

```java
// Sequential (SLOW): total = 200ms + 150ms + 100ms = 450ms
User user = userService.getUser(id);           // 200ms
List<Order> orders = orderService.getOrders(id); // 150ms
int points = loyaltyService.getPoints(id);       // 100ms

// Parallel (FAST): total = max(200ms, 150ms, 100ms) = 200ms!
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(
    () -> userService.getUser(id), ioExecutor);
CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(
    () -> orderService.getOrders(id), ioExecutor);
CompletableFuture<Integer> pointsFuture = CompletableFuture.supplyAsync(
    () -> loyaltyService.getPoints(id), ioExecutor);

// Wait for all and combine
CompletableFuture.allOf(userFuture, ordersFuture, pointsFuture).join();
return new UserProfile(userFuture.get(), ordersFuture.get(), pointsFuture.get());

// Speed improvement: 450ms → 200ms (2.25x faster!)
```

---

## 🔒 Lock Contention

```
LOCK CONTENTION = Multiple threads fighting for the same lock

SYMPTOMS:
  - Threads in BLOCKED state (visible in thread dumps)
  - CPU is low but response is slow (threads waiting, not working!)
  - Throughput plateaus as you add more threads

COMMON CAUSES IN JAVA:
  1. synchronized(sharedMap) { ... }  ← One thread at a time!
  2. Database row-level locks (SELECT FOR UPDATE)
  3. Single-threaded connection pool acquisition
  4. Logging frameworks with synchronous file writes

SOLUTIONS:
  1. ConcurrentHashMap → replaces synchronized HashMap (lock striping!)
  2. ReadWriteLock → many readers, exclusive writer
  3. Atomic variables → CAS instead of locking (AtomicLong, AtomicReference)
  4. Lock-free algorithms → compare-and-swap operations
  5. Reduce lock scope → hold lock for minimum time
  6. Partition work → each thread works on its own partition
```

### Lock-Free Counter Example

```java
// ❌ SLOW: Lock contention at high concurrency
class CounterSlow {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int get() { return count; }
}

// ✅ FAST: Lock-free with CAS (Compare-And-Swap)
class CounterFast {
    private final AtomicLong count = new AtomicLong(0);
    public void increment() { count.incrementAndGet(); } // No lock!
    public long get() { return count.get(); }
}

// ✅✅ FASTEST: For high contention (Java 8+)
class CounterFastest {
    private final LongAdder count = new LongAdder(); // Striped counters!
    public void increment() { count.increment(); }
    public long get() { return count.sum(); }
}
// LongAdder > AtomicLong when many threads increment simultaneously
// Because: each thread increments its own stripe, sum() combines them
```

---

## 🐌 Common Bottlenecks

| Bottleneck | Symptom | Fix |
|-----------|---------|-----|
| Thread pool exhaustion | 503 errors, requests queued | Increase pool OR add async |
| Lock contention | Low CPU, high latency, BLOCKED threads | ConcurrentHashMap, reduce scope |
| GC pauses under load | Periodic latency spikes | Tune GC, reduce allocation |
| Connection pool starvation | Threads waiting for DB connection | Increase pool OR reduce hold time |
| Thundering herd | Traffic spike → all threads wake | Stagger, jitter, backpressure |

---

## 🎓 Interview Q&A

### Q1: "How do you size a thread pool?"
**A**: For CPU-bound work, use number of cores. For I/O-bound work, use `cores × (1 + wait_time/service_time)`. In practice, start with `cores × 2` for web apps, monitor thread utilization, and adjust. Use separate pools for different work types.

### Q2: "What are virtual threads and when should you use them?"
**A**: Virtual threads (Java 21+) are lightweight threads managed by the JVM, not the OS. They're ideal for I/O-bound workloads where traditional threads are wasted waiting. They allow millions of concurrent connections with minimal memory. Not suitable for CPU-bound or synchronized-heavy code.

### Q3: "How do you detect lock contention?"
**A**: Thread dumps show threads in BLOCKED state waiting for monitors. JFR (Java Flight Recorder) can show lock contention hotspots. async-profiler with `-e lock` event profiles lock waits. In production, monitor thread states via JMX metrics.

---

## 🎲 Boss Battle: The Slow Dashboard 🕵️

> **Scenario**: A dashboard API aggregates data from 5 downstream services (user profile, orders, recommendations, notifications, loyalty points). Each call takes 100-200ms. The dashboard response is 800ms. CPU is only 5%.
>
> **Challenge**: How do you make it respond in <250ms?
>
> <details>
> <summary>🔓 Click to reveal answer</summary>
>
> **Problem**: 5 sequential I/O calls: 150ms × 5 = 750ms + overhead = 800ms
>
> **Solution**: Parallelize all 5 calls!
> ```java
> CompletableFuture.allOf(profileF, ordersF, recsF, notifsF, loyaltyF).join();
> ```
> **Result**: max(150ms, 120ms, 200ms, 100ms, 80ms) = 200ms + overhead = ~220ms ✅
>
> **Additional optimizations:**
> 1. Cache profile + loyalty (change rarely) → 2ms from cache
> 2. Set timeouts on each call (300ms max) with fallback defaults
> 3. Non-critical calls (notifications) can be optional (skip if slow)
> 4. With virtual threads: no thread pool sizing needed!
>
> **Result**: 800ms → 200ms (4x improvement!) 🚀
> </details>

---

👉 **[Next: Network & I/O Performance →](./Network_IO_Performance.md)**  
👉 **[Back to Performance Overview →](./README.md)**
