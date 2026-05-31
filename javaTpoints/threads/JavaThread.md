## Mastering Java Multithreading and Concurrency for Interviews 🚦

Acing Java interviews at top tech companies requires solid knowledge of multithreading and concurrency. This guide covers core concepts, advanced techniques, best practices, and Java code examples to give you an edge.

---

### 🔍 1. Understanding Threads and Processes
- **Thread:** A lightweight unit of execution within a process.
- **Process:** A self-contained execution environment (has its own memory).
- **Differences:** Threads share memory within a process, processes do not.
- **Multithreading Benefits:**
    - Efficient CPU utilization.
    - Better responsiveness in GUIs.
    - Asynchronous background tasks.
- **Context Switching:** CPU switches from one thread to another—overhead due to saving/restoring states.

---

### 🚀 2. Creating and Managing Threads

- **Runnable vs Thread:** Prefer `Runnable` for better OOP practices (allows inheritance).
```java
Runnable task = () -> System.out.println("Running in thread " + Thread.currentThread().getName());
new Thread(task).start();
```

- **start() vs run():**
    - `start()` launches a new thread.
    - `run()` runs on the current thread.

- **Thread Lifecycle:**
    - `NEW` -> `RUNNABLE` -> `RUNNING` -> `BLOCKED`/`WAITING` -> `TERMINATED`

- **Thread Priorities:**
    - Range: 1 (MIN_PRIORITY) to 10 (MAX_PRIORITY).
    - May be ignored by OS/thread scheduler.

- **Daemon vs User Threads:**
    - Daemon threads run in background (e.g., GC).
    - JVM exits when only daemon threads remain.

---

### 🛡️ 3. Synchronization and Avoiding Race Conditions

- **Shared Resources + Critical Sections = Race Conditions**
```java
synchronized (lockObject) {
   // critical section
}
```

- **Intrinsic Locks:** Every object has one. `synchronized` uses them.
- **Object-Level Locks:** Lock is tied to object instance.
- **Deadlock Conditions:** Mutual exclusion, hold & wait, no preemption, circular wait.
- **Avoiding Deadlock:** Lock ordering, timeout, `tryLock()`.

---

### 📬 4. Inter-Thread Communication

- **wait(), notify(), notifyAll():** Must be called inside a `synchronized` block.
- **Spurious Wakeups:** Always check conditions in a loop.
```java
synchronized (lock) {
  while (!condition) lock.wait();
  // proceed when condition is true
}
```

---

### ⚙️ 5. Concurrency Utilities (`java.util.concurrent`)

#### 🔄 Executors Framework
- `ExecutorService`, `ThreadPoolExecutor`, `ScheduledExecutorService`.
```java
ExecutorService executor = Executors.newFixedThreadPool(5);
executor.submit(() -> System.out.println("Async task"));
```
- **Thread Pool Types:**
    - Fixed: Limit threads.
    - Cached: Dynamic scaling.
    - Scheduled: Delay & periodic execution.
    - Single: Sequential tasks.
- **Future:**
```java
Future<Integer> result = executor.submit(() -> 42);
System.out.println(result.get());
```

#### 🔒 Locks
- `Lock` vs `synchronized`: More control.
- `ReentrantLock`: Manual lock/unlock, fair locks, `tryLock()`.
- `ReadWriteLock`: High read, low write scenarios.

#### 🧰 Concurrent Collections
- **Standard collections are not thread-safe.**
- Use:
    - `ConcurrentHashMap`
    - `CopyOnWriteArrayList`
    - `ConcurrentLinkedQueue`

#### 🧩 Synchronization Aids
- **CountDownLatch:** Wait for tasks to finish.
- **CyclicBarrier:** All threads wait at a barrier.
- **Semaphore:** Limit concurrent access.
- **Exchanger:** Thread data exchange.

#### 🔢 Atomic Variables
```java
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();
```

---

### 📛 6. Volatile Keyword

- Ensures visibility, avoids stale values.
- Prevents reordering (to a degree).
- **Limitation:** Does not ensure atomicity.
```java
volatile boolean flag = true;
```

---

### 🧵 7. ThreadLocal

- Provides each thread its own copy.
```java
ThreadLocal<Integer> threadId = ThreadLocal.withInitial(() -> 1);
```
- Useful in web apps, DB transactions.

---

### ✅ 8. Best Practices & Pitfalls

- Avoid shared mutable state.
- Minimize lock scope.
- Prefer high-level abstractions (Executors, ForkJoin).
- Handle thread exceptions (`UncaughtExceptionHandler`).
- Avoid blocking I/O inside synchronized blocks.
- Benchmark concurrency features before production use.

---

### 📘 Final Tips for Interviews

- Be ready to write thread-safe code.
- Explain trade-offs clearly (e.g., ReentrantLock vs synchronized).
- Use diagrams to explain thread lifecycle or deadlocks.
- Practice LeetCode concurrency problems.
- Be familiar with common pitfalls and how to debug them.

Master these concepts, and you’ll confidently breeze through any Java multithreading round! 💪
---

*Previous: [← Throwable](../Throwable.md) | Next: [Java Thread Part 2 →](./JavaThreadpart2.md)*
