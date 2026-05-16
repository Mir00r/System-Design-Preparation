# 🧠 Memory Management: Leaks, Pooling & Off-Heap Memory

> *"Memory leaks in Java don't mean you forgot to free memory — they mean you're holding references to objects you'll never use again. The GC can't collect what's still reachable."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [JVM Tuning](./JVM_Tuning.md)

---

## 📋 Table of Contents
1. [Memory Leaks in Java](#-memory-leaks)
2. [Detecting Leaks](#-detecting-leaks)
3. [Object Pooling](#-object-pooling)
4. [Off-Heap Memory](#-off-heap-memory)
5. [Common Pitfalls](#-common-pitfalls)
6. [Interview Q&A](#-interview-qa)

---

## 💧 Memory Leaks in Java

```
COMMON LEAK PATTERNS:

1. COLLECTIONS THAT ONLY GROW (never remove entries)
   // Cache without eviction — grows forever!
   private static Map<String, Object> cache = new HashMap<>();
   public void addToCache(String key, Object value) {
       cache.put(key, value); // never removed → leak!
   }
   Fix: Use WeakHashMap, Caffeine cache with maxSize, or explicit eviction

2. LISTENERS NOT DEREGISTERED
   eventBus.register(this);  // registered on creation
   // Object "destroyed" but still referenced by eventBus → can't be GC'd
   Fix: Always unregister in cleanup/destroy lifecycle method

3. INNER CLASSES HOLDING OUTER CLASS REFERENCE
   class Outer {
       byte[] largeData = new byte[10_000_000]; // 10MB
       Runnable task = new Runnable() {  // holds implicit reference to Outer!
           public void run() { /* doesn't use largeData */ }
       };
   }
   Fix: Use static inner class or lambda (if not capturing outer state)

4. THREAD-LOCAL NOT CLEANED
   private static ThreadLocal<Connection> connHolder = new ThreadLocal<>();
   // In thread pool: thread is reused, ThreadLocal value persists!
   Fix: Always call threadLocal.remove() in finally block

5. UNCLOSED RESOURCES
   InputStream stream = new FileInputStream(file); // never closed!
   // Finalizer may eventually close it, but memory builds up
   Fix: try-with-resources (always)
```

---

## 🔎 Detecting Leaks

```
SIGNS OF A MEMORY LEAK:
  - Heap usage baseline rises after each GC cycle
  - Frequent Full GC events (old gen fills up)
  - Eventually: OutOfMemoryError
  - Memory usage graph looks like rising staircase (not sawtooth)

DIAGNOSIS TOOLS:
  1. Heap Dump:
     jcmd <PID> GC.heap_dump /tmp/heap.hprof
     # Or automatically on OOM: -XX:+HeapDumpOnOutOfMemoryError
     
  2. Analyze with Eclipse MAT (Memory Analyzer Tool):
     - Open heap dump
     - "Leak Suspects Report" (auto-detects likely leaks)
     - "Dominator Tree" (largest objects retaining memory)
     - "Histogram" (count of each class → what's accumulating?)
     
  3. Compare two heap dumps:
     - Take dump at T0 (healthy)
     - Take dump at T0+1hour (after leak accumulated)
     - Compare: what grew? That's your leak.

  4. JFR (Flight Recorder) — live monitoring:
     jcmd <PID> JFR.start duration=60s filename=recording.jfr
     # Shows allocation rates, object ages, GC behavior
```

---

## ♻️ Object Pooling

```
WHEN TO POOL:
  Objects expensive to create: DB connections, threads, SSL contexts
  Objects frequently created/destroyed in tight loops
  
  ✅ Pool: database connections (HikariCP)
  ✅ Pool: threads (ExecutorService)
  ✅ Pool: byte buffers (Netty ByteBufAllocator)
  ❌ Don't pool: simple objects (String, DTOs) — GC handles these fine

EXAMPLE: Custom object pool (rarely needed — use libraries)
  // Apache Commons Pool 2
  GenericObjectPool<ExpensiveObject> pool = new GenericObjectPool<>(
      new BasePooledObjectFactory<>() {
          @Override
          public ExpensiveObject create() { return new ExpensiveObject(); }
          @Override
          public PooledObject<ExpensiveObject> wrap(ExpensiveObject obj) {
              return new DefaultPooledObject<>(obj);
          }
          @Override
          public void passivateObject(PooledObject<ExpensiveObject> p) {
              p.getObject().reset(); // clean state before returning to pool
          }
      });
  pool.setMaxTotal(20);
  pool.setMaxIdle(10);
  
  // Usage
  ExpensiveObject obj = pool.borrowObject();
  try {
      obj.doWork();
  } finally {
      pool.returnObject(obj); // return to pool, not GC'd
  }
```

---

## 📦 Off-Heap Memory

```
OFF-HEAP (Direct) MEMORY:
  Allocated outside JVM heap → NOT managed by GC
  Used for: NIO buffers, memory-mapped files, large caches
  
  Benefits:
  - No GC pauses (GC doesn't scan off-heap)
  - Can be larger than heap (limited by OS memory)
  - Efficient I/O (no copy between heap and OS buffers)
  
  Risks:
  - Manual management (must explicitly free!)
  - No GC safety net (leak = OS memory exhaustion)
  - Harder to debug

  // Direct ByteBuffer (JDK standard)
  ByteBuffer buffer = ByteBuffer.allocateDirect(1024 * 1024); // 1MB off-heap
  // Used by Netty, Kafka, Cassandra for zero-copy I/O
  
  // Monitoring: -XX:MaxDirectMemorySize=256m (limit off-heap)

USE CASES:
  Netty:      Network I/O buffers (off-heap for zero-copy)
  Ehcache:    Off-heap cache tier (gigabytes without GC pressure)
  MapDB:      Memory-mapped file database
  Chronicle:  Low-latency inter-process messaging
```

---

## ⚠️ Common Pitfalls

1. **Unbounded caches** — `HashMap` as a cache without size limit or TTL is the #1 memory leak in Java. Always use Caffeine/Guava cache with `maximumSize()` and `expireAfterWrite()`.

2. **Large objects in session/request scope** — Storing large byte arrays or collections in HTTP session. Each user session accumulates memory. Set session timeouts and store large data in Redis/S3.

3. **Not monitoring native memory** — `jcmd <PID> VM.native_memory summary` shows native memory usage. Heap looks fine but process memory grows? Check native memory (Metaspace, thread stacks, direct buffers, JNI).

---

## 📝 Interview Q&A

**Q: How would you diagnose and fix an OutOfMemoryError in production?**
> A: (1) **Immediate**: ensure `-XX:+HeapDumpOnOutOfMemoryError` is set so you get a heap dump automatically. (2) **Identify OOM type**: `java.lang.OutOfMemoryError: Java heap space` (heap full) vs `Metaspace` (too many classes) vs `unable to create native thread` (thread limit). (3) **For heap OOM**: open heap dump in Eclipse MAT → check Leak Suspects → find the collection/object accumulating → trace back to code that adds without removing. (4) **Common fixes**: add cache eviction policy, fix collection leak, reduce object size (use primitives over wrappers), increase heap if data set legitimately grew. (5) **Prevent**: monitor heap usage trend (CloudWatch/Prometheus), alert when baseline rises, add `-XX:+ExitOnOutOfMemoryError` so orchestrator restarts the container immediately.

---

## 🔗 What to Read Next

1. **[Performance/Benchmarking.md](./Benchmarking.md)** — JMH microbenchmarks
2. **[Performance/JVM_Tuning.md](./JVM_Tuning.md)** — GC algorithms
3. **[Performance/Profiling_and_Optimization.md](./Profiling_and_Optimization.md)** — Finding bottlenecks

---

*[← JVM Tuning](./JVM_Tuning.md) | [Back to Index](../INDEX.md) | [Next: Benchmarking →](./Benchmarking.md)*
