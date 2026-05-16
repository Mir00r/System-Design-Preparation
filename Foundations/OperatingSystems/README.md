# 🖥️ Operating Systems: Processes, Threads, and Memory

> *"Your application runs on an OS. Understanding how the OS manages processes, threads, and memory explains why your service OOMs at 2 AM, why context switching kills throughput, and why the event-loop model works so well for I/O-bound services."*

**⏱️ Estimated Time**: 22 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Networking](../Networking/README.md)

---

## 🏗️ Processes vs Threads

```
PROCESS:
  ┌──────────────────────────────┐
  │ Process (isolated)           │
  │ ┌──────┐ ┌──────┐ ┌──────┐  │
  │ │Heap  │ │Stack │ │Code  │  │  ← Own memory space
  │ └──────┘ └──────┘ └──────┘  │
  │ PID: 1234                    │  ← Own PID
  │ File descriptors, signals    │  ← Own resources
  └──────────────────────────────┘
  • Isolation: crash in one process doesn't affect others
  • Expensive to create (fork: copy memory, file descriptors)
  • IPC needed to communicate (pipes, sockets, shared memory)
  
THREAD (lightweight process):
  ┌──────────────────────────────────────────┐
  │ Process                                  │
  │ ┌────────────────────────────────────┐   │
  │ │ Shared: Heap, Code, File Descs     │   │  ← Shared memory!
  │ └────────────────────────────────────┘   │
  │ ┌────────┐ ┌────────┐ ┌────────┐        │
  │ │Thread 1│ │Thread 2│ │Thread 3│        │  ← Own stack only
  │ │Stack   │ │Stack   │ │Stack   │        │
  │ └────────┘ └────────┘ └────────┘        │
  └──────────────────────────────────────────┘
  • Share memory: fast communication (but needs synchronization!)
  • Cheap to create (~8KB stack vs full process memory)
  • One thread crash can kill entire process
  • Concurrency bugs: race conditions, deadlocks

SYSTEM DESIGN IMPLICATIONS:
  Multi-process (Nginx workers, Redis): Isolation, no shared state bugs
  Multi-threaded (Java Tomcat): Efficient memory, complex synchronization
  Event loop (Node.js, Netty): Single thread, async I/O, no locks needed
```

---

## ⚡ Concurrency Models

```
MODEL 1: THREAD-PER-REQUEST (Traditional Java/Tomcat)
  Request → spawn thread → block on I/O → resume → respond
  
  ✅ Simple mental model (sequential code)
  ❌ Threads are expensive (~1MB stack each)
  ❌ Context switching overhead at high concurrency
  ❌ 10K threads = ~10GB memory just for stacks
  
  Typical: 200-500 concurrent requests per server

MODEL 2: EVENT LOOP (Node.js, Nginx, Netty)
  Request → register callback → event loop handles I/O → callback fires
  
  ┌─────────────────────────────────────────┐
  │ Event Loop (single thread)              │
  │                                         │
  │  ← poll for events                      │
  │  → handle event (non-blocking)          │
  │  ← poll for events                      │
  │  → handle event (non-blocking)          │
  │                                         │
  │  I/O operations: delegated to OS (epoll)│
  └─────────────────────────────────────────┘
  
  ✅ Handles 10K-100K concurrent connections
  ✅ Minimal memory (no thread stacks)
  ❌ CPU-bound work blocks entire loop
  ❌ Callback hell (mitigated by async/await)
  
  Typical: 10K-50K concurrent connections per server

MODEL 3: VIRTUAL THREADS (Java 21+ / Project Loom)
  Best of both: sequential code + cheap "threads"
  
  Virtual threads: millions possible, ~few KB each
  Blocking I/O → JVM parks virtual thread → schedules another
  No callback hell, no reactive complexity
  
  ✅ Sequential code (easy to read/debug)
  ✅ Millions of concurrent connections
  ✅ No thread pool tuning needed
```

---

## 💾 Memory Management

```
VIRTUAL MEMORY:
  ┌─────────────────────────────────────┐
  │ Process Virtual Address Space        │
  │                                      │
  │ ┌──────────┐ 0xFFFF... (high)       │
  │ │  Stack   │ ↓ grows down            │
  │ │          │                         │
  │ │  (free)  │                         │
  │ │          │                         │
  │ │  Heap    │ ↑ grows up              │
  │ ├──────────┤                         │
  │ │  Data    │ (global variables)      │
  │ ├──────────┤                         │
  │ │  Code    │ (read-only)             │
  │ └──────────┘ 0x0000... (low)        │
  └─────────────────────────────────────┘
  
  Virtual memory → Physical RAM (via page table)
  If physical RAM full → swap to disk (EXTREMELY slow)
  
SYSTEM DESIGN IMPLICATIONS:
  • Container memory limits: if process exceeds → OOM kill
  • JVM heap: -Xmx sets max heap; GC pauses when heap is large
  • Memory-mapped files: map file into virtual memory → OS handles caching
  • Copy-on-write: fork() shares pages until write (cheap process creation)

OOM (Out of Memory) SCENARIOS:
  1. Memory leak: objects never garbage collected → heap grows → OOM
  2. Container limit: process tries to allocate beyond cgroup limit → killed
  3. Too many connections: each connection has buffers → memory exhaustion
  
  PREVENTION: Monitor RSS, set container limits, use connection pooling
```

---

## 📊 Context Switching Cost

```
CONTEXT SWITCH: OS saves state of current thread, loads state of next

  Cost per switch: 1-10 microseconds
  Seems small, but at 100K switches/sec: 10% CPU just for switching!

  WHAT GETS SAVED/RESTORED:
  • CPU registers (general purpose, flags, instruction pointer)
  • Stack pointer
  • Page table pointer (if process switch)
  • TLB flush (if process switch) ← expensive!

  SYSTEM DESIGN LESSONS:
  Thread pool size ≈ number of CPU cores (for CPU-bound work)
  Too many threads → constant switching → thrashing
  
  OPTIMAL THREAD POOL SIZE:
  • CPU-bound: threads = number of cores
  • I/O-bound: threads = cores × (1 + wait_time/service_time)
  • Example: 8 cores, I/O takes 200ms, processing takes 20ms
    → threads = 8 × (1 + 200/20) = 88 threads
```

---

## ⚠️ Common Pitfalls

1. **Thread pool too large** — If your service does 50ms CPU + 200ms I/O per request, 200 threads on an 8-core machine means 25 threads compete per core. Context switching overhead negates the parallelism. Right-size your pools.

2. **Ignoring container memory limits** — JVM allocates heap based on `-Xmx`, but total memory includes: heap + metaspace + thread stacks + native buffers + OS overhead. Set container limit = JVM max heap × 1.5-2x.

3. **Deadlocks in production** — Two threads lock resources in different order. Prevention: always acquire locks in consistent order, use timeouts, prefer lock-free data structures (ConcurrentHashMap).

---

## 📝 Interview Q&A

**Q: Why does Node.js handle more concurrent connections than a traditional Java server?**
> A: Node.js uses a single-threaded event loop with non-blocking I/O. Each connection is just an event handler (~few KB), not a thread (~1MB stack). So Node.js can handle 10K-100K connections on one server while Java/Tomcat with thread-per-request handles 200-500. However, Node.js can't parallelize CPU-bound work on that single thread. Modern Java (virtual threads / Project Loom) bridges this gap: sequential code with millions of lightweight threads.

---

## 🔗 What to Read Next

1. **[Foundations/HowInternetWorks/README.md](../HowInternetWorks/README.md)** — Full request lifecycle
2. **[Performance/JVM_Tuning.md](../../Performance/JVM_Tuning.md)** — JVM memory and tuning
3. **[Performance/Memory_Management.md](../../Performance/Memory_Management.md)** — Memory optimization

---

*[← Networking](../Networking/README.md) | [Back to Index](../../INDEX.md) | [Next: How the Internet Works →](../HowInternetWorks/README.md)*
