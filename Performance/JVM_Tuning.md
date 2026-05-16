# ☕ JVM Tuning: Garbage Collection, Heap Sizing & JVM Flags

> *"The JVM is a marvel of engineering — but default settings are designed for general use. Production workloads need tuned GC, proper heap sizing, and monitoring to achieve consistent low-latency performance."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Performance README](./README.md), Java memory model basics

---

## 📋 Table of Contents
1. [JVM Memory Model](#-memory-model)
2. [Garbage Collectors](#-garbage-collectors)
3. [Heap Sizing](#-heap-sizing)
4. [Essential JVM Flags](#-essential-flags)
5. [GC Tuning Strategy](#-gc-tuning)
6. [Interview Q&A](#-interview-qa)

---

## 🧱 Memory Model

```
JVM MEMORY LAYOUT:
  ┌─────────────────────────────────────────────────────────┐
  │                    JVM PROCESS                          │
  ├─────────────────────────────────────────────────────────┤
  │  HEAP (objects live here — GC manages this)             │
  │  ┌─────────────────┬────────────────────────────────┐  │
  │  │  Young Gen      │         Old Gen                │  │
  │  │  ┌─────┬──┬──┐ │                                │  │
  │  │  │Eden │S0│S1│ │  (long-lived objects)           │  │
  │  │  └─────┴──┴──┘ │                                │  │
  │  │ (new objects)   │                                │  │
  │  └─────────────────┴────────────────────────────────┘  │
  ├─────────────────────────────────────────────────────────┤
  │  NON-HEAP                                               │
  │  ┌──────────┬────────────┬───────────┬──────────────┐  │
  │  │Metaspace │ Code Cache │ Thread    │  Direct      │  │
  │  │(classes) │ (JIT code) │ Stacks    │  Buffers     │  │
  │  └──────────┴────────────┴───────────┴──────────────┘  │
  └─────────────────────────────────────────────────────────┘

OBJECT LIFECYCLE:
  New object → Eden (young gen)
  Survives minor GC → Survivor space (S0 ↔ S1)
  Survives N minor GCs → Promoted to Old Gen
  Old Gen fills up → Major GC (Full GC) — EXPENSIVE!
```

---

## 🗑️ Garbage Collectors

```
┌───────────────────┬──────────────────┬─────────────────┬──────────────────┐
│ Collector         │ Pause Time       │ Throughput       │ Best For         │
├───────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ Serial (-XX:+Use  │ Long (100s ms)   │ Single-threaded │ Tiny heaps (<1GB)│
│  SerialGC)        │                  │                 │ Containers       │
├───────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ Parallel (-XX:+Use│ Medium (100s ms) │ HIGH            │ Batch processing │
│  ParallelGC)      │                  │ (multi-thread)  │ Throughput apps  │
├───────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ G1 (-XX:+UseG1GC) │ Low (10-50ms)    │ Good            │ DEFAULT (JDK 9+) │
│                   │ (targets pause)  │                 │ General purpose  │
├───────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ ZGC (-XX:+UseZGC) │ ULTRA-LOW (<1ms) │ Good            │ Low-latency apps │
│                   │ (concurrent)     │                 │ Large heaps (TB) │
├───────────────────┼──────────────────┼─────────────────┼──────────────────┤
│ Shenandoah       │ ULTRA-LOW (<1ms) │ Good            │ Low-latency      │
│                   │ (concurrent)     │                 │ (Red Hat JDK)    │
└───────────────────┴──────────────────┴─────────────────┴──────────────────┘

RECOMMENDATION (JDK 21+):
  Web APIs / Microservices:  G1 (default) or ZGC (if p99 latency critical)
  Batch processing:          Parallel GC (maximize throughput)
  Large heaps (>8GB):        ZGC (handles TB-size heaps with <1ms pauses)
  Containers (small heap):   Serial GC or G1

G1 vs ZGC:
  G1:  targets 200ms max pause, good balance
  ZGC: targets <1ms pauses, uses more CPU (concurrent marking)
       Ideal when: p99 latency matters more than raw throughput
```

---

## 📐 Heap Sizing

```
HEAP SIZING STRATEGY:

  Too small: Frequent GC → high pause times → OOM crashes
  Too large: Infrequent but LONG GC pauses → wasted memory
  Right size: GC runs infrequently with short pauses

RULES OF THUMB:
  -Xms = -Xmx (set min = max to avoid heap resizing)
  Live data set × 3-4 = good heap size
  
  Example: App's live objects use 500MB → set heap to 1.5-2GB
  
  For containers:
    Container memory limit: 2GB
    Heap: 75% of container = 1.5GB (-Xmx1536m)
    Non-heap (metaspace, threads, native): remaining 25%
    
    -XX:MaxRAMPercentage=75.0  (let JVM calculate from container limit)

MONITORING HEAP:
  Healthy heap usage pattern:
    ╱╲  ╱╲  ╱╲  ╱╲  ← sawtooth (objects created, GC cleans them)
    after GC, usage returns to baseline (e.g., 30% of heap)
    
  UNHEALTHY (memory leak):
    ╱╲╱╲╱╲╱╲╱╲╱╲╱╲╱ ← baseline RISING after each GC
    Eventually: OOM! Baseline never decreases.
```

---

## 🏁 Essential Flags

```bash
# PRODUCTION JVM FLAGS (Spring Boot microservice, JDK 21)

java \
  # Heap sizing
  -Xms2g -Xmx2g \                    # Fixed heap (no resize overhead)
  -XX:MaxMetaspaceSize=256m \          # Limit metaspace (class loading)
  
  # GC selection (choose one)
  -XX:+UseG1GC \                       # G1 (default, good balance)
  # -XX:+UseZGC \                      # ZGC (ultra-low latency)
  
  # G1 tuning
  -XX:MaxGCPauseMillis=100 \           # Target max pause (G1 will try)
  -XX:G1HeapRegionSize=16m \           # Region size (for large heaps)
  
  # GC logging (ALWAYS enable in production!)
  -Xlog:gc*:file=/var/log/gc.log:time,level,tags:filecount=5,filesize=50m \
  
  # OOM handling
  -XX:+HeapDumpOnOutOfMemoryError \    # Dump heap on OOM (for analysis)
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -XX:+ExitOnOutOfMemoryError \        # Exit (let orchestrator restart)
  
  # Container awareness (JDK 8u191+)
  -XX:+UseContainerSupport \           # Respect container memory limits
  -XX:MaxRAMPercentage=75.0 \          # Use 75% of container memory
  
  # Performance
  -XX:+AlwaysPreTouch \                # Pre-allocate heap pages (faster startup after)
  -XX:+UseStringDeduplication \        # G1 only: dedup identical strings
  
  -jar app.jar
```

---

## 🎯 GC Tuning

```
GC TUNING WORKFLOW:

1. Enable GC logging (ALWAYS, even in production — low overhead)
2. Analyze with tools: GCViewer, GCEasy.io, JDK Mission Control
3. Look for:
   - Full GC events (should be RARE or NEVER)
   - Pause time > SLO (target: < 50ms for web apps)
   - GC frequency (too frequent = heap too small or too much allocation)
   - Promotion rate (objects moving young → old too fast)

COMMON SCENARIOS:

  Problem: Frequent minor GC (every few seconds)
  Cause:   Young gen too small OR too many short-lived objects
  Fix:     -XX:NewSize=512m (increase young gen) or reduce allocations
  
  Problem: Occasional long Full GC (500ms+)
  Cause:   Old gen filling up, triggering full compaction
  Fix:     Switch to ZGC (concurrent), or increase heap, or fix memory leak
  
  Problem: OOM despite "enough" memory
  Cause:   Memory leak — objects accumulating in old gen
  Fix:     Heap dump → analyze with Eclipse MAT → find leak source
```

---

## ⚠️ Common Pitfalls

1. **Not setting -Xms = -Xmx** — If min < max, JVM resizes heap at runtime (causes pauses). In production, always set both equal. Pre-allocate with `-XX:+AlwaysPreTouch`.

2. **Ignoring GC logs** — GC logs are FREE performance monitoring. Always enable them. When an incident happens, GC logs are your first diagnostic tool. Without them, you're flying blind.

3. **Tuning without measuring** — Don't copy GC flags from Stack Overflow. Every application has different allocation patterns. Measure YOUR app's GC behavior, then tune based on data.

---

## 📝 Interview Q&A

**Q: How would you choose between G1 and ZGC for a microservice?**
> A: **G1** (default): good for most microservices. Targets a max pause time (default 200ms, configurable). Suitable for heaps 2-8GB. Well-understood, mature, good throughput. Use when: p99 latency target is > 50ms, heap is moderate size. **ZGC**: concurrent collector with < 1ms pauses regardless of heap size. Uses more CPU (concurrent marking). Use when: p99 latency must be < 10ms (trading, real-time bidding), heap is very large (>16GB), or any Full GC would violate SLOs. In practice: start with G1 (default), monitor GC pauses. If GC pauses violate your SLOs, switch to ZGC. Most standard web APIs are fine with G1.

---

## 🔗 What to Read Next

1. **[Performance/Memory_Management.md](./Memory_Management.md)** — Memory leaks and optimization
2. **[Performance/Profiling_and_Optimization.md](./Profiling_and_Optimization.md)** — Profiling techniques
3. **[Performance/Benchmarking.md](./Benchmarking.md)** — JMH microbenchmarks

---

*[← Database Performance](./Database_Performance.md) | [Back to Index](../INDEX.md) | [Next: Memory Management →](./Memory_Management.md)*
