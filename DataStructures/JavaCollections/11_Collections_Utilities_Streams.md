# 🛠️ Java Collections Utilities & Streams: The Power Tools 🚀

---

## 🎯 Chapter 11: Collections Utilities & Stream API — Mastering the Toolbox

> **"Knowing a collection is good. Knowing how to sort, search, transform, and pipeline collections is what separates a junior developer from a senior one. These utilities are the Swiss Army knife of Java development."**

---

## 📋 Table of Contents

1. [Collections Utility Class](#collections-utility-class)
2. [Arrays Utility Class](#arrays-utility-class)
3. [Stream API with Collections](#stream-api)
4. [Collectors — The Power of collect()](#collectors)
5. [Java 9+ Immutable Factory Methods](#java-9-factory-methods)
6. [Java 10-21 Stream Evolution](#modern-stream-features)
7. [Unmodifiable vs Immutable Views](#unmodifiable-vs-immutable)
8. [Performance Guide: Loop vs Stream vs Parallel](#performance-guide)
9. [Real-World Analogies](#real-world-analogies)
10. [Time & Space Complexity](#time--space-complexity)
11. [Common Pitfalls](#common-pitfalls)
12. [Industry Applications](#industry-applications)
13. [Interview Tips](#interview-tips)

---

## 🧰 Collections Utility Class

### 📖 **Definition**

`java.util.Collections` is a **utility class** of only static methods that operate on or return collections. It complements the Collections Framework with algorithms and decorators.

```
java.util.Collections provides:
✅ Sorting algorithms (sort, reverseOrder, unmodifiableXxx)
✅ Search algorithms (binarySearch)
✅ Shuffling and rotation
✅ Min/max finding
✅ Frequency and disjoint tests
✅ Unmodifiable wrappers
✅ Synchronized wrappers
✅ Singleton/empty collections
✅ Type-checked (checked) collections
```

### 🔧 **Sorting**

```java
List<Integer> nums = new ArrayList<>(List.of(5, 2, 8, 1, 9, 3));

// ── SORT ──
Collections.sort(nums);                                      // ascending (natural order)
Collections.sort(nums, Comparator.reverseOrder());           // descending
Collections.sort(nums, (a, b) -> b - a);                    // descending (manual)

// Sort by field:
List<Employee> employees = getEmployees();
Collections.sort(employees, Comparator.comparing(Employee::getName));
Collections.sort(employees, Comparator.comparing(Employee::getSalary).reversed());
Collections.sort(employees,
    Comparator.comparing(Employee::getDept)
              .thenComparing(Employee::getName));           // multi-field

// ── REVERSE ── O(n)
Collections.reverse(nums);   // in-place reverse, does NOT sort

// ── ROTATE ── O(n)
List<String> letters = new ArrayList<>(List.of("a", "b", "c", "d", "e"));
Collections.rotate(letters, 2);  // right rotate by 2: [d, e, a, b, c]
Collections.rotate(letters, -1); // left rotate by 1:  [e, a, b, c, d]
// Rotate is useful for circular scheduling algorithms

// ── SHUFFLE ── O(n)
Collections.shuffle(nums);                     // uses internal Random
Collections.shuffle(nums, new Random(42));     // seeded for reproducibility
// Use case: randomize questions, card shuffling, A/B testing
```

### 🔧 **Binary Search** (requires sorted list!)

```java
List<Integer> sorted = List.of(1, 3, 5, 7, 9, 11);

int idx = Collections.binarySearch(sorted, 7);   // returns 3 (index)
int missing = Collections.binarySearch(sorted, 4);
// returns negative: -(insertion point) - 1
// For value 4: insertion point = 2 (between 3 and 5), returns -(2)-1 = -3

// Extract insertion point from negative result:
if (missing < 0) {
    int insertionPoint = -missing - 1; // 2
}

// Custom comparator:
List<String> words = new ArrayList<>(List.of("apple", "banana", "cherry"));
// Must be sorted by the SAME comparator you search with!
Collections.sort(words, Comparator.comparing(String::length));
int pos = Collections.binarySearch(words, "cherry", Comparator.comparing(String::length));
// ⚠️ WARNING: if multiple elements have same key, result is unspecified index
```

### 🔧 **Min, Max, Frequency, Disjoint**

```java
List<Integer> list = List.of(3, 1, 4, 1, 5, 9, 2, 6);

int min = Collections.min(list);              // 1
int max = Collections.max(list);              // 9
int freq = Collections.frequency(list, 1);   // 2 (appears twice)

// With comparator:
List<String> words = List.of("cat", "elephant", "bee");
String longest = Collections.max(words, Comparator.comparing(String::length)); // "elephant"

// ── DISJOINT — true if no elements in common ──
boolean disjoint = Collections.disjoint(List.of(1,2,3), List.of(4,5,6)); // true
boolean overlap  = Collections.disjoint(List.of(1,2,3), List.of(3,4,5)); // false
```

### 🔧 **Fill, Copy, Swap, nCopies**

```java
List<String> target = new ArrayList<>(Arrays.asList("a", "b", "c", "d", "e"));

// ── FILL — replace all elements ──
Collections.fill(target, "X");   // [X, X, X, X, X]

// ── SWAP ──
Collections.swap(target, 0, 4);  // swap index 0 and 4

// ── COPY — dest must be at least as large as src ──
List<String> source = List.of("1", "2", "3");
List<String> dest = new ArrayList<>(Arrays.asList("a", "b", "c", "d", "e"));
Collections.copy(dest, source);  // dest = [1, 2, 3, d, e] (only first 3 overwritten)

// ── nCopies — immutable list of N copies ──
List<String> tenXs = Collections.nCopies(10, "X");   // ["X","X",..."X"] (10 items)
List<Integer> zeros = Collections.nCopies(5, 0);     // [0,0,0,0,0]
// Backed by a single reference, memory-efficient! 
// ⚠️ Immutable — cannot be modified
// Use case: initialize a mutable list:
List<String> mutable = new ArrayList<>(Collections.nCopies(10, "default"));
```

### 🔧 **Unmodifiable Wrappers** — Runtime protection

```java
List<String> original = new ArrayList<>(List.of("a", "b", "c"));
List<String> readOnly = Collections.unmodifiableList(original);

readOnly.get(0);          // ✅ "a" — reads are allowed
readOnly.add("d");        // ❌ UnsupportedOperationException!
readOnly.remove(0);       // ❌ UnsupportedOperationException!

// ⚠️ IMPORTANT: It's a VIEW — changes to 'original' are reflected!
original.add("d");
System.out.println(readOnly); // [a, b, c, d] — reflected!

// For true immutability, copy before wrapping, or use List.of() (Java 9+)
List<String> trulyImmutable = Collections.unmodifiableList(new ArrayList<>(original));
// Or simply: List.of("a", "b", "c")  ← preferred in modern Java

// Other unmodifiable wrappers:
Set<String> roSet = Collections.unmodifiableSet(mySet);
Map<K,V> roMap = Collections.unmodifiableMap(myMap);
SortedSet<E> roSortedSet = Collections.unmodifiableSortedSet(mySortedSet);
SortedMap<K,V> roSortedMap = Collections.unmodifiableSortedMap(mySortedMap);
```

### 🔧 **Synchronized Wrappers**

```java
// Wrap any collection with synchronized methods:
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
Map<K,V> syncMap = Collections.synchronizedMap(new HashMap<>());

// Remember: iteration needs external sync!
synchronized (syncList) {
    for (String s : syncList) { process(s); }
}
```

### 🔧 **Singleton & Empty Collections**

```java
// ── EMPTY — type-safe, immutable, single instance (flyweight pattern) ──
List<String> empty = Collections.emptyList();      // <T>emptyList()
Set<Integer> emptySet = Collections.emptySet();
Map<K,V> emptyMap = Collections.emptyMap();
Iterator<E> emptyIter = Collections.emptyIterator();

// ✅ USE INSTEAD OF null (null-object pattern)
public List<Order> getOrders(String userId) {
    if (!userExists(userId)) return Collections.emptyList(); // not null!
    return orderRepo.findByUser(userId);
}

// ── SINGLETON — single-element immutable collections ──
List<String> single = Collections.singletonList("only-one");
Set<String> singleSet = Collections.singleton("only-one");
Map<K,V> singleMap = Collections.singletonMap("key", "value");
// Memory-efficient: no array overhead, just wraps one reference
```

---

## 📐 Arrays Utility Class

### 🔧 **Sorting**

```java
int[] arr = {5, 2, 8, 1, 9, 3};

// ── PRIMITIVE ARRAYS — dual-pivot quicksort ──
Arrays.sort(arr);              // [1, 2, 3, 5, 8, 9]
Arrays.sort(arr, 1, 4);       // sort only arr[1..3]: [5, 1, 2, 8, 9, 3]

// ── OBJECT ARRAYS — TimSort (stable! O(n log n)) ──
String[] words = {"banana", "apple", "cherry"};
Arrays.sort(words);                                  // natural order
Arrays.sort(words, Comparator.comparing(String::length));  // by length

// ── WHY DIFFERENT ALGORITHMS? ──
// Primitives: quicksort (O(n log n) avg, no stability needed — no object identity)
// Objects: TimSort (stable — equal objects preserve their relative order)
//          Also very fast on partially sorted data (common in real datasets)!
```

### 🔧 **Binary Search, Fill, Copy**

```java
int[] sorted = {1, 3, 5, 7, 9, 11};
int idx = Arrays.binarySearch(sorted, 7);  // 3 — must be sorted first!

// ── FILL ──
int[] filled = new int[5];
Arrays.fill(filled, 42);     // [42, 42, 42, 42, 42]
Arrays.fill(filled, 1, 3, 0); // fill index 1 to 2: [42, 0, 0, 42, 42]

// ── COPY ──
int[] original = {1, 2, 3, 4, 5};
int[] copy = Arrays.copyOf(original, 3);          // [1, 2, 3] (first 3)
int[] longer = Arrays.copyOf(original, 8);        // [1,2,3,4,5, 0,0,0] (padded)
int[] range = Arrays.copyOfRange(original, 1, 4); // [2, 3, 4] (index 1 to 3)

// ── EQUALS ──
Arrays.equals(new int[]{1,2,3}, new int[]{1,2,3}); // true
Arrays.deepEquals(new int[][]{{1,2},{3,4}}, new int[][]{{1,2},{3,4}}); // true (2D)

// ── TO STRING ──
System.out.println(Arrays.toString(original));    // [1, 2, 3, 4, 5]
int[][] matrix = {{1,2},{3,4}};
System.out.println(Arrays.deepToString(matrix));  // [[1, 2], [3, 4]]

// ── STREAM ──
int[] nums = {5, 2, 8, 1};
int sum = Arrays.stream(nums).sum();                        // 16
double avg = Arrays.stream(nums).average().orElse(0);       // 4.0
int max = Arrays.stream(nums).max().orElse(Integer.MIN_VALUE); // 8
long count = Arrays.stream(nums).filter(x -> x > 3).count(); // 2

// ── asList ── (creates FIXED-SIZE list backed by array!)
String[] arr2 = {"a", "b", "c"};
List<String> list = Arrays.asList(arr2); // FIXED-SIZE — can set(), can NOT add/remove!
list.set(0, "A"); // ✅ OK
list.add("d");    // ❌ UnsupportedOperationException!
// For mutable list: new ArrayList<>(Arrays.asList(arr2))
```

---

## 🌊 Stream API with Collections

### 📖 **Stream Pipeline Anatomy**

```
Source → Intermediate Operations → Terminal Operation
          (lazy, chainable)          (eager, triggers eval)

myList.stream()         ← source
      .filter(...)      ← intermediate (lazy)
      .map(...)         ← intermediate (lazy)
      .sorted(...)      ← intermediate (lazy, stateful)
      .collect(...)     ← terminal (triggers all of above)
```

### 🔧 **Creating Streams**

```java
// From collections:
List<String> list = List.of("a", "b", "c");
list.stream()              // sequential stream
list.parallelStream()      // parallel stream

// From arrays:
Arrays.stream(new int[]{1,2,3})      // IntStream
Arrays.stream(new String[]{"a","b"}) // Stream<String>

// From values:
Stream.of("a", "b", "c")
Stream.ofNullable(maybeNull)  // empty stream if null (Java 9+)
IntStream.range(0, 10)        // 0,1,...,9
IntStream.rangeClosed(0, 10)  // 0,1,...,10
IntStream.iterate(0, n -> n+2).limit(5) // 0,2,4,6,8
Stream.generate(Math::random).limit(3)  // 3 random doubles
```

### 🔧 **Intermediate Operations**

```java
List<String> words = List.of("hello", "world", "java", "stream", "api");

// ── filter — keep elements matching predicate ──
words.stream()
     .filter(w -> w.length() > 4)  // ["hello", "world", "stream"]
     .collect(Collectors.toList());

// ── map — transform each element ──
words.stream()
     .map(String::toUpperCase)      // ["HELLO", "WORLD", ...]
     .collect(Collectors.toList());

// ── mapToInt / mapToLong / mapToDouble — to primitive stream ──
words.stream().mapToInt(String::length).sum();  // total character count

// ── flatMap — flatten nested streams ──
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4), List.of(5));
List<Integer> flat = nested.stream()
                            .flatMap(Collection::stream)  // [1,2,3,4,5]
                            .collect(Collectors.toList());

// ── distinct ──
List<Integer> dupes = List.of(1,2,2,3,3,3,4);
dupes.stream().distinct().collect(Collectors.toList()); // [1,2,3,4]

// ── sorted ──
words.stream().sorted().collect(Collectors.toList());  // alphabetically
words.stream().sorted(Comparator.comparing(String::length)).collect(Collectors.toList()); // by length

// ── limit / skip ──
Stream.iterate(1, n -> n+1)
      .skip(5)    // skip first 5 (6,7,8,...)
      .limit(3)   // take only 3: [6,7,8]
      .collect(Collectors.toList());

// ── peek — debug without consuming ──
words.stream()
     .peek(w -> System.out.println("Before: " + w))
     .filter(w -> w.length() > 4)
     .peek(w -> System.out.println("After: " + w))
     .collect(Collectors.toList());
```

### 🔧 **Terminal Operations**

```java
List<Integer> nums = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// ── collect ── (see next section for full details)
List<Integer> evens = nums.stream()
                           .filter(n -> n % 2 == 0)
                           .collect(Collectors.toList());

// ── forEach ──
nums.stream().forEach(System.out::println); // no guaranteed order for parallel!
nums.stream().forEachOrdered(System.out::println); // preserves encounter order

// ── reduce ── fold all elements into one
int sum = nums.stream().reduce(0, Integer::sum);   // 55
Optional<Integer> max = nums.stream().reduce(Integer::max); // Optional[10]

// ── count ──
long count = nums.stream().filter(n -> n > 5).count(); // 5

// ── min / max ──
Optional<Integer> min = nums.stream().min(Integer::compareTo); // Optional[1]
Optional<Integer> maxNum = nums.stream().max(Integer::compareTo); // Optional[10]

// ── anyMatch / allMatch / noneMatch ── short-circuits!
boolean anyEven = nums.stream().anyMatch(n -> n % 2 == 0);  // true (stops at 2)
boolean allPos = nums.stream().allMatch(n -> n > 0);         // true
boolean noneNeg = nums.stream().noneMatch(n -> n < 0);       // true

// ── findFirst / findAny ──
Optional<Integer> first = nums.stream().filter(n -> n > 5).findFirst(); // Optional[6]
Optional<Integer> any = nums.parallelStream().filter(n -> n > 5).findAny(); // any >5

// ── toArray ──
Object[] arr = nums.stream().filter(n -> n % 2 == 0).toArray();
Integer[] typedArr = nums.stream().filter(n -> n%2==0).toArray(Integer[]::new);
```

---

## 🏺 Collectors — The Power of collect()

```java
List<Employee> employees = getEmployees();

// ── toList / toSet / toCollection ──
List<String> names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

Set<String> depts = employees.stream()
    .map(Employee::getDept)
    .collect(Collectors.toSet());

TreeSet<String> sortedDepts = employees.stream()
    .map(Employee::getDept)
    .collect(Collectors.toCollection(TreeSet::new));

// ── toMap ──
// Key: employee name, Value: salary
Map<String, Double> salaryMap = employees.stream()
    .collect(Collectors.toMap(
        Employee::getName,    // key mapper
        Employee::getSalary,  // value mapper
        (existing, replacement) -> existing // merge function (for duplicate keys)
    ));

// ── groupingBy — by department ──
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));
// {"Engineering": [Alice, Bob], "Marketing": [Carol], ...}

// groupingBy with downstream collector:
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept, Collectors.counting()));

Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.averagingDouble(Employee::getSalary)
    ));

Map<String, Optional<Employee>> highestPaidByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDept,
        Collectors.maxBy(Comparator.comparing(Employee::getSalary))
    ));

// ── partitioningBy — split into two groups ──
Map<Boolean, List<Employee>> seniorVsJunior = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getYearsExp() >= 5));
List<Employee> senior = seniorVsJunior.get(true);
List<Employee> junior = seniorVsJunior.get(false);

// ── joining — for strings ──
String allNames = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));     // "Alice, Bob, Carol"

String formatted = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", ", "[", "]")); // "[Alice, Bob, Carol]"

// ── counting ──
long engCount = employees.stream()
    .filter(e -> "Engineering".equals(e.getDept()))
    .collect(Collectors.counting()); // or just .count()

// ── summarizingInt — statistics ──
IntSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingInt(Employee::getYearsExp));
stats.getCount(); stats.getMin(); stats.getMax(); stats.getSum(); stats.getAverage();

// ── toUnmodifiableList/Set/Map (Java 10+) ──
List<String> immutableNames = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toUnmodifiableList());

// ── teeing — split to two collectors, merge results (Java 12+) ──
record MinMax(int min, int max) {}
MinMax minMax = IntStream.of(3,1,4,1,5,9,2,6)
    .boxed()
    .collect(Collectors.teeing(
        Collectors.minBy(Integer::compareTo),
        Collectors.maxBy(Integer::compareTo),
        (min, max) -> new MinMax(min.orElse(0), max.orElse(0))
    ));
```

---

## 🏭 Java 9+ Immutable Factory Methods

### 📖 **List.of(), Set.of(), Map.of()**

```java
// ── Java 9: Immutable factory methods ──
List<String> list = List.of("a", "b", "c");   // immutable, no nulls, no duplicate check
Set<Integer> set = Set.of(1, 2, 3, 4);         // immutable, no nulls, no duplicates!
Map<String, Integer> map = Map.of(             // immutable, no nulls
    "one", 1,
    "two", 2,
    "three", 3
);

// For maps with more than 10 entries, use Map.ofEntries():
Map<String, Integer> bigMap = Map.ofEntries(
    Map.entry("key1", 1),
    Map.entry("key2", 2),
    // ... up to any number of entries
    Map.entry("key20", 20)
);

// List.of() vs Collections.unmodifiableList():
List<String> mutable = new ArrayList<>(List.of("a","b","c"));
List<String> view = Collections.unmodifiableList(mutable); // view — changes if mutable changes!
List<String> copy = List.copyOf(mutable); // true snapshot — immune to original changes

// ── Java 10: copyOf ──
List<String> listCopy = List.copyOf(existingList);   // immutable copy
Set<String> setCopy = Set.copyOf(existingSet);
Map<K,V> mapCopy = Map.copyOf(existingMap);

// ── DIFFERENCES ──
// List.of():   immutable, CANNOT contain null, O(1) creation
// Arrays.asList(): FIXED-SIZE (set ok, add/remove not ok), CAN contain null
// new ArrayList<>(list): MUTABLE, CAN contain null, O(n) copy
// Collections.unmodifiableList(): read-only VIEW (still see original changes)
// List.copyOf(): immutable copy (independent snapshot)
```

---

## 🔮 Modern Stream Features (Java 10–21)

```java
// ── Java 16: Stream.toList() — simplest immutable collection ──
List<String> result = names.stream()
    .filter(n -> n.startsWith("A"))
    .toList(); // shorter than .collect(Collectors.toUnmodifiableList())

// ── Java 9: Stream.takeWhile / dropWhile ──
List<Integer> nums = List.of(1, 2, 3, 4, 5, 4, 3, 2, 1);
List<Integer> taken = nums.stream()
    .takeWhile(n -> n < 4)    // [1, 2, 3] — stops at first failure
    .toList();
List<Integer> dropped = nums.stream()
    .dropWhile(n -> n < 4)    // [4, 5, 4, 3, 2, 1] — drops until first failure
    .toList();

// ── Java 9: Stream.iterate with predicate ──
List<Integer> powers = Stream.iterate(1, n -> n < 1000, n -> n * 2)
    .toList();  // [1, 2, 4, 8, ..., 512]

// ── Java 9: Stream.ofNullable ──
String value = null;
long count = Stream.ofNullable(value).count(); // 0 (empty stream, not NPE!)
Stream.ofNullable("hello").forEach(System.out::println); // "hello"

// ── Java 12: Collectors.teeing ──
var result2 = Stream.of(1,2,3,4,5)
    .collect(Collectors.teeing(
        Collectors.summingInt(Integer::intValue),  // collector 1: sum
        Collectors.counting(),                     // collector 2: count
        (sum, count2) -> (double) sum / count2     // average = 3.0
    ));

// ── Java 22+: Stream.gather() (preview) ──
// More flexible mid-stream operations (custom stateful transformations)
```

---

## 🔒 Unmodifiable vs Immutable Views

```
                    UNMODIFIABLE          IMMUTABLE
─────────────────────────────────────────────────────────────────────────────
What it is          View of original      Independent copy
Mutations via self  ❌ throws             ❌ throws
Mutations via original ✅ seen by view   ❌ impossible
Null elements       Depends on original  ❌ for List.of/Set.of/Map.of
Creation            Collections.unmodifiable...()  List.of() / List.copyOf()
Memory              O(1) — wraps only    O(n) — full copy
Thread-safe?        Only if underlying   ✅ Yes (no writes possible)
                    collection is

EXAMPLES:
  Collections.unmodifiableList(mutable) → "read-only window" into mutable
  List.of(...)                          → truly immutable from creation
  List.copyOf(existing)                 → immutable snapshot at that moment
```

---

## ⚡ Performance Guide: Loop vs Stream vs Parallel

```
OPERATION TYPE        │ TRADITIONAL LOOP │ STREAM     │ PARALLEL STREAM
──────────────────────┼──────────────────┼────────────┼────────────────────────────────
Simple iteration      │ Fastest          │ Close      │ SLOWER (thread overhead)
Simple filter+collect │ Fast             │ Similar    │ Often SLOWER for small lists
Complex multi-step    │ Verbose          │ Cleaner    │ Can be faster for large data
Large data (>10K)     │ Fast             │ Fast       │ Often FASTER
CPU-bound transforms  │ Fast             │ Similar    │ Often FASTER (multi-core)
I/O-bound operations  │ Sequential       │ Sequential │ ⚠️ Usually WORSE (contention)
Primitives (int[])    │ Fastest          │ IntStream  │ IntStream.parallel() (best for math)
```

```java
// ── WHEN TO USE TRADITIONAL LOOP ──
// 1. Simple single iteration with no transformation:
for (String s : list) log(s); // clearest intent

// 2. Need early exit without exception tricks:
for (int i = 0; i < list.size(); i++) {
    if (found(list.get(i))) return i; // clean return
}

// 3. Collecting with index:
for (int i = 0; i < list.size(); i++) {
    result.add(i + ": " + list.get(i));
}

// ── WHEN TO USE STREAM ──
// 1. Multiple transformation steps (readable pipeline):
employees.stream()
    .filter(e -> e.getSalary() > 50000)
    .map(Employee::getName)
    .sorted()
    .collect(Collectors.toList());

// 2. Collectors: groupingBy, joining, partitioningBy
Map<String, List<Employee>> grouped = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDept));

// ── WHEN TO USE PARALLEL STREAM ──
// Only when ALL of these are true:
// 1. Large dataset (>10,000 elements typically)
// 2. CPU-intensive, stateless operations
// 3. No shared mutable state
// 4. No I/O or blocking operations

long count = largeList.parallelStream()
    .filter(this::complexPredicate)       // CPU-intensive check
    .mapToInt(this::expensiveCompute)     // expensive computation
    .sum();

// ❌ BAD parallel stream patterns:
// Modifying shared state:
List<String> result = new ArrayList<>(); // NOT thread-safe!
stream.parallel().forEach(s -> result.add(s)); // race condition!
// Fix: collect() to a new list (Collectors handle thread safety)
List<String> safe = stream.parallel().collect(Collectors.toList());

// Inherently sequential operations:
// sorted() on parallel stream → must merge all partial sorts → usually slower
```

---

## 🌍 Real-World Analogies

```
UTILITY            ANALOGY
─────────────────────────────────────────────────────────────────────
Collections.sort() Like hiring a professional organizer for your bookshelf
Collections.shuffle() Like shuffling a deck of cards before dealing
binarySearch()     Like looking up a word in a dictionary (not scanning page by page)
Collections.unmodifiableList() Like putting your documents in a display case — visible but untouchable
List.of()          Like a plaque cast in concrete — unchangeable from creation
Stream pipeline    Like a factory assembly line — raw material flows through stations
Collectors.groupingBy() Like a post office sorting mail into different PO boxes by destination
parallelStream()   Like splitting a sorting task between multiple workers
```

---

## ⏱️ Time & Space Complexity

```
OPERATION                          │ TIME        │ NOTES
───────────────────────────────────┼─────────────┼──────────────────────────────────────
Collections.sort(list)             │ O(n log n)  │ TimSort (stable)
Collections.binarySearch(list, k)  │ O(log n)    │ Requires pre-sorted list!
Collections.min/max(collection)    │ O(n)        │ Linear scan
Collections.frequency(coll, o)     │ O(n)        │ Linear scan
Collections.disjoint(c1, c2)       │ O(n)        │ Worst case
Collections.reverse(list)          │ O(n)        │ In-place swap
Collections.shuffle(list)          │ O(n)        │ Fisher-Yates algorithm
Collections.nCopies(n, obj)        │ O(1)        │ Single reference repeated!
Arrays.sort(primitiveArray)        │ O(n log n)  │ Dual-pivot quicksort
Arrays.sort(objectArray)           │ O(n log n)  │ TimSort (stable)
Arrays.binarySearch(arr, k)        │ O(log n)    │ Requires sorted array
Arrays.copyOf(arr, newLen)         │ O(n)        │ System.arraycopy
stream().collect(toList())         │ O(n)        │ Single pass
stream().sorted()                  │ O(n log n)  │ TimSort on stream buffer
stream().distinct()                │ O(n)        │ HashSet tracking
Collectors.groupingBy()            │ O(n)        │ HashMap bucketing
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Arrays.asList() Returns Fixed-Size List**

```java
String[] arr = {"a", "b", "c"};
List<String> list = Arrays.asList(arr); // Fixed-size! Backed by array.

list.set(0, "A");  // ✅ OK — modifies array too!
list.add("d");     // ❌ UnsupportedOperationException!
list.remove(0);    // ❌ UnsupportedOperationException!

arr[1] = "B";       // ✅ modifies list too (they share the array!)
System.out.println(list); // [A, B, c] — the change is reflected!

// ✅ For a truly independent mutable list:
List<String> mutable = new ArrayList<>(Arrays.asList(arr));
// Or in modern Java:
List<String> mutable2 = new ArrayList<>(List.of(arr));
```

### 💣 **Pitfall 2: Modifying a Collection During Stream**

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// ❌ Modifying source while streaming:
list.stream().forEach(s -> {
    if (s.equals("b")) list.remove(s); // ConcurrentModificationException!
});

// ✅ Collect removals first, then apply:
List<String> toRemove = list.stream()
    .filter(s -> s.equals("b"))
    .collect(Collectors.toList());
list.removeAll(toRemove);

// ✅ Or use removeIf (most elegant):
list.removeIf(s -> s.equals("b")); // direct, efficient, no stream needed
```

### 💣 **Pitfall 3: Streams Are One-Use**

```java
Stream<String> stream = List.of("a","b","c").stream();
long count = stream.count();           // ✅ OK, uses the stream
List<String> list = stream.collect(Collectors.toList()); // ❌ IllegalStateException!
// "stream has already been operated upon or closed"

// ✅ Create a new stream each time:
List<String> source = List.of("a","b","c");
long count2 = source.stream().count();
List<String> list2 = source.stream().collect(Collectors.toList());

// ✅ Or store the result, not the stream:
List<String> result = source.stream()
    .filter(s -> s.startsWith("a"))
    .collect(Collectors.toList());
```

### 💣 **Pitfall 4: Parallel Stream Side Effects**

```java
// ❌ NEVER modify external state in parallel streams:
List<String> results = new ArrayList<>(); // NOT thread-safe!
names.parallelStream()
     .filter(n -> n.length() > 3)
     .forEach(n -> results.add(n)); // RACE CONDITION — may lose elements!

// ✅ Use collect() for thread-safe aggregation:
List<String> safe = names.parallelStream()
    .filter(n -> n.length() > 3)
    .collect(Collectors.toList()); // always thread-safe!
```

---

## 🏢 Industry Applications

```
FRAMEWORK/USE CASE    UTILITY                           EXAMPLE
────────────────────────────────────────────────────────────────────────────────────────────
Spring Data JPA       Stream API                        Page results to streams, lazy loading
Spring Security       Collections.unmodifiableSet()     immutable granted authorities
Kafka Consumer        Collections.singletonList()       subscribe to single topic
REST Controller       Collectors.groupingBy()           aggregate response DTOs
Spring Batch          Collectors.partitioningBy()       split failed/succeeded items
Hibernate ORM         Collections.sort()                in-memory sort of lazy collections
Netflix Hystrix       Collections.emptyList()           fallback response for circuit break
JSON Serialization    Collectors.toMap()                entity → DTO field mapping
Spring Cache          Collections.unmodifiableMap()     read-only cache snapshot
Microservice gateway  Stream filtering + mapping        Request header transformation
```

### 💻 **Spring Boot: Full Analytics Pipeline**

```java
@Service
public class AnalyticsService {
    
    public DashboardReport buildReport(List<Order> orders) {
        
        // ── Revenue by category ──
        Map<String, Double> revenueByCategory = orders.stream()
            .collect(Collectors.groupingBy(
                Order::getCategory,
                Collectors.summingDouble(Order::getAmount)
            ));
        
        // ── Top 5 products by revenue ──
        List<String> top5Products = orders.stream()
            .collect(Collectors.groupingBy(
                Order::getProductName,
                Collectors.summingDouble(Order::getAmount)
            ))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(5)
            .map(Map.Entry::getKey)
            .toList();  // Java 16+ shorthand
        
        // ── Orders by status (completed/pending partition) ──
        Map<Boolean, List<Order>> byCompletion = orders.stream()
            .collect(Collectors.partitioningBy(
                o -> "COMPLETED".equals(o.getStatus())
            ));
        
        // ── Monthly order count ──
        Map<String, Long> monthlyCounts = orders.stream()
            .collect(Collectors.groupingBy(
                o -> o.getOrderDate().getMonth().toString(),
                Collectors.counting()
            ));
        
        // ── Summary statistics ──
        DoubleSummaryStatistics amountStats = orders.stream()
            .collect(Collectors.summarizingDouble(Order::getAmount));
        
        return new DashboardReport(
            revenueByCategory,
            top5Products,
            byCompletion.get(true).size(),
            byCompletion.get(false).size(),
            monthlyCounts,
            amountStats.getAverage(),
            amountStats.getMax()
        );
    }
}
```

---

## 💡 Interview Tips

**Q1: What is the difference between Collections.sort() and List.sort()?**
```
Both use TimSort (stable, O(n log n)), but:

Collections.sort(list):
  - Static method on Collections utility class
  - Available since Java 2
  - Internally calls list.sort() since Java 8

list.sort(comparator):
  - Instance method on List interface (default method since Java 8)
  - Slightly more natural to use
  - Can pass null for natural ordering

// Modern preferred style:
employees.sort(Comparator.comparing(Employee::getName));

// vs older style:
Collections.sort(employees, Comparator.comparing(Employee::getName));

Both produce identical results. list.sort() is the modern idiom.
```

**Q2: When should you use parallelStream()?**
```
Parallel streams use the ForkJoinPool.commonPool() by default.
They're beneficial ONLY when:

✅ Large dataset (>10,000 elements — benchmark for your case)
✅ CPU-intensive, stateless operations per element
✅ No I/O, no shared mutable state, no ordering requirements
✅ Computation time >> thread-coordination overhead

Common MISTAKES with parallel streams:
❌ Using on small lists → thread overhead dominates
❌ Modifying shared variables → race conditions
❌ Database/I/O calls → thread pool exhaustion
❌ Relying on encounter order → nondeterministic results

Rule of thumb: measure first, parallelize only if benchmarks show benefit.
Amdahl's Law limits parallel speedup: sequential portions don't benefit.
```

**Q3: What is the difference between map() and flatMap() in streams?**
```java
// map() → one-to-one transformation
Stream<String[]> words = Stream.of("hello world", "foo bar")
    .map(s -> s.split(" ")); // Stream<String[]> — nested!

// flatMap() → one-to-many transformation (flattens one level)
Stream<String> words2 = Stream.of("hello world", "foo bar")
    .flatMap(s -> Arrays.stream(s.split(" "))); // Stream<String> — flat!
// "hello", "world", "foo", "bar"

// Real example:
List<Order> orders = getOrders();
List<OrderItem> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream()) // flatten order → items
    .collect(Collectors.toList());
```

**Q4: Explain the difference between List.of() and Collections.unmodifiableList().**
```
List.of("a","b","c"):
  - Truly immutable from the start
  - Cannot contain nulls (throws NullPointerException)
  - Independent of any other list
  - Uses a space-efficient implementation (not ArrayList internally)
  - Thread-safe (immutability implies thread safety)

Collections.unmodifiableList(mutableList):
  - Returns a read-only VIEW of the original list
  - The view still CHANGES if the underlying list changes
  - Can contain nulls (if original does)
  - Wraps the original — memory-efficient (O(1))
  - Only the VIEW is read-only; you can still modify via original reference!

Example:
  List<String> src = new ArrayList<>(List.of("a","b"));
  List<String> view = Collections.unmodifiableList(src);
  src.add("c");
  System.out.println(view); // [a, b, c] ← view reflects the change!

For true immutable snapshots in modern Java, prefer:
  List.copyOf(src)  ← immutable independent copy
  List.of(...)      ← immutable from construction
```

---

*Previous: [Concurrent Collections ←](./10_Concurrent_Collections.md) | Next: [Back to DataStructures Overview →](../README.md)*
