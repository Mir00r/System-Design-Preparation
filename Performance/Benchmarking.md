# 📏 Benchmarking: JMH, Load Testing & Measuring Performance Correctly

> *"Benchmarking on the JVM is treacherous. JIT compilation, GC pauses, CPU frequency scaling, and branch prediction can make naive measurements meaningless. JMH handles all these pitfalls so you can trust your results."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Profiling](./Profiling_and_Optimization.md), Java basics

---

## 📋 Table of Contents
1. [Why Benchmarking is Hard](#-why-hard)
2. [JMH (Java Microbenchmark Harness)](#-jmh)
3. [Common Benchmark Patterns](#-patterns)
4. [Load Testing vs Microbenchmarks](#-comparison)
5. [Common Pitfalls](#-common-pitfalls)
6. [Interview Q&A](#-interview-qa)

---

## 🤔 Why Benchmarking is Hard

```
NAIVE BENCHMARK (WRONG!):
  long start = System.nanoTime();
  for (int i = 0; i < 1000000; i++) {
      result = myMethod(input);
  }
  long elapsed = System.nanoTime() - start;
  System.out.println("Time: " + elapsed / 1000000 + "ms");

PROBLEMS:
  1. JIT hasn't compiled the method yet (first runs are slow/interpreted)
  2. Dead code elimination: JIT sees 'result' is never used → removes the loop!
  3. Constant folding: if input is constant, JIT pre-computes the answer
  4. GC pauses included in measurement (random spikes)
  5. CPU frequency scaling: processor may throttle during benchmark
  6. No statistical rigor: one measurement means nothing

JMH SOLVES ALL OF THESE:
  - Warmup iterations (JIT stabilizes)
  - Blackhole (prevents dead code elimination)
  - State objects (prevents constant folding)
  - Fork (fresh JVM per benchmark — isolated GC)
  - Statistical output (mean, std dev, percentiles)
```

---

## 🔬 JMH

```java
// JMH Benchmark example
@BenchmarkMode(Mode.AverageTime)           // measure average execution time
@OutputTimeUnit(TimeUnit.NANOSECONDS)       // report in nanoseconds
@Warmup(iterations = 5, time = 1)          // 5 warmup iterations (JIT stabilizes)
@Measurement(iterations = 10, time = 1)     // 10 measurement iterations
@Fork(2)                                    // 2 fresh JVM forks (reproducibility)
@State(Scope.Benchmark)                     // shared state across iterations
public class StringConcatBenchmark {
    
    private String firstName = "John";
    private String lastName = "Doe";
    private int age = 30;
    
    @Benchmark
    public String concatWithPlus() {
        return firstName + " " + lastName + " (" + age + ")";
    }
    
    @Benchmark
    public String concatWithBuilder() {
        return new StringBuilder()
            .append(firstName).append(" ")
            .append(lastName).append(" (")
            .append(age).append(")")
            .toString();
    }
    
    @Benchmark
    public String concatWithFormat() {
        return String.format("%s %s (%d)", firstName, lastName, age);
    }
    
    public static void main(String[] args) throws Exception {
        Options opt = new OptionsBuilder()
            .include(StringConcatBenchmark.class.getSimpleName())
            .build();
        new Runner(opt).run();
    }
}

// RESULTS (example):
// Benchmark                         Mode  Cnt   Score   Error  Units
// concatWithPlus                    avgt   20   25.3 ± 0.8  ns/op  ← fastest (JDK 9+ optimizes)
// concatWithBuilder                 avgt   20   28.1 ± 1.2  ns/op
// concatWithFormat                  avgt   20  385.7 ± 12.4 ns/op  ← 15x slower!
```

---

## 🏗️ Patterns

```java
// PATTERN 1: Comparing collection implementations
@State(Scope.Benchmark)
public class MapBenchmark {
    private Map<String, Integer> hashMap;
    private Map<String, Integer> treeMap;
    private Map<String, Integer> concurrentMap;
    
    @Setup
    public void setup() {
        hashMap = new HashMap<>();
        treeMap = new TreeMap<>();
        concurrentMap = new ConcurrentHashMap<>();
        for (int i = 0; i < 10000; i++) {
            String key = "key-" + i;
            hashMap.put(key, i);
            treeMap.put(key, i);
            concurrentMap.put(key, i);
        }
    }
    
    @Benchmark
    public Integer hashMapGet() { return hashMap.get("key-5000"); }
    
    @Benchmark
    public Integer treeMapGet() { return treeMap.get("key-5000"); }
    
    @Benchmark
    public Integer concurrentMapGet() { return concurrentMap.get("key-5000"); }
}

// PATTERN 2: Parameterized benchmarks
@State(Scope.Benchmark)
public class SortBenchmark {
    @Param({"100", "1000", "10000", "100000"})
    private int size;
    
    private int[] data;
    
    @Setup(Level.Invocation)  // fresh data each invocation (sorting mutates!)
    public void setup() {
        data = ThreadLocalRandom.current().ints(size).toArray();
    }
    
    @Benchmark
    public int[] arraySort() {
        Arrays.sort(data);
        return data;
    }
    
    @Benchmark
    public int[] parallelSort() {
        Arrays.parallelSort(data);
        return data;
    }
}

// PATTERN 3: Serialization comparison
@Benchmark
public byte[] jacksonSerialize() throws Exception {
    return objectMapper.writeValueAsBytes(order);
}

@Benchmark
public byte[] protobufSerialize() {
    return orderProto.toByteArray();
}
```

---

## ⚖️ Load Testing vs Microbenchmarks

| | Microbenchmark (JMH) | Load Test (Gatling/k6) |
|---|---|---|
| Scope | Single method/operation | Full system end-to-end |
| Environment | Isolated (single JVM) | Production-like (network, DB) |
| Measures | ns/op, ops/sec | Latency, throughput, errors |
| Answers | "Which algorithm is faster?" | "Can system handle 10K users?" |
| Concurrency | Usually single-threaded | Multi-user simulation |
| When | Choosing between implementations | Before production deployment |

---

## ⚠️ Common Pitfalls

1. **Dead code elimination** — If benchmark result is unused, JIT removes the computation. Always return the result or use `Blackhole.consume(result)`. JMH's `@Benchmark` method return handles this automatically.

2. **Benchmarking with default JVM settings** — Production uses different GC, heap size, JIT settings. Run benchmarks with production-like JVM flags. Or better: benchmark the system end-to-end (load test) for realistic results.

3. **Drawing conclusions from one benchmark** — "HashMap is O(1) so always use HashMap" ignores: cache locality, memory overhead, thread safety requirements, iteration performance. Benchmark YOUR actual use case with YOUR actual data sizes.

4. **Not accounting for warm-up** — JVM performance improves dramatically after JIT compilation (~10,000 method invocations). Always include warmup iterations. Never trust the first measurement.

---

## 📝 Interview Q&A

**Q: How would you prove that your optimization actually improved performance?**
> A: (1) **Before**: run JMH benchmark (or load test) with current implementation — record mean, p95, p99 with confidence intervals. (2) **Apply optimization**: one change at a time. (3) **After**: run SAME benchmark, same environment, same data — compare results. (4) **Statistical significance**: JMH reports error margins (±). If the improvement is within error margin, it's not a real improvement. (5) **Multiple runs**: use JMH's fork parameter (2+) to get results across different JVM instances. (6) **Production validation**: microbenchmark confirms code-level improvement, but also validate with load test against staging to confirm system-level improvement (sometimes micro-optimization doesn't show in end-to-end latency because the bottleneck is elsewhere).

---

## 🔗 What to Read Next

1. **[Performance/Profiling_and_Optimization.md](./Profiling_and_Optimization.md)** — Finding what to benchmark
2. **[Performance/JVM_Tuning.md](./JVM_Tuning.md)** — JVM flags for benchmarks
3. **[Testing/Performance_Testing.md](../Testing/Performance_Testing.md)** — System-level load testing

---

*[← Memory Management](./Memory_Management.md) | [Back to Index](../INDEX.md)*
