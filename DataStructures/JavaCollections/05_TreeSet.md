# 🌳 Java TreeSet: The Sorted Set Powerhouse 🚀

---

## 🎯 Chapter 5: TreeSet & Sorted Collections

> **"TreeSet is the librarian of collections — it keeps everything alphabetically sorted at all times, so finding the 'nearest' item is instant, whether you want the book before or after a given title."**

---

## 📋 Table of Contents

1. [What is TreeSet?](#what-is-treeset)
2. [Internal Architecture — Red-Black Tree](#internal-architecture)
3. [SortedSet Interface](#sortedset-interface)
4. [NavigableSet Interface](#navigableset-interface)
5. [All Methods Explained](#all-methods-explained)
6. [Natural vs Custom Ordering](#natural-vs-custom-ordering)
7. [Real-World Analogies](#real-world-analogies)
8. [When to USE TreeSet](#when-to-use-treeset)
9. [When to AVOID TreeSet](#when-to-avoid-treeset)
10. [Time & Space Complexity](#time--space-complexity)
11. [Common Pitfalls](#common-pitfalls)
12. [Industry Applications](#industry-applications)
13. [Practice Problems](#practice-problems)
14. [Interview Tips](#interview-tips)

---

## 🤔 What is TreeSet?

### 📖 **Definition**

`TreeSet<E>` is a `NavigableSet` implementation backed by a **Red-Black Tree**. Elements are stored in **sorted order** (natural ordering or custom Comparator), enabling efficient range operations.

```
TreeSet characteristics:
✅ Elements kept SORTED at all times
✅ O(log n) add, remove, contains
✅ Rich navigation: floor, ceiling, headSet, tailSet, subSet
✅ NO duplicates (Set contract)
❌ NO null elements (compareTo throws NullPointerException)
❌ Slower than HashSet (O(log n) vs O(1))
❌ NOT thread-safe
❌ Elements MUST be Comparable OR a Comparator must be provided

Introduced: Java 1.2 (1998)
Backed by: TreeMap<E, Object> — same as HashSet/HashMap relationship!
```

---

## 🏗️ Internal Architecture — Red-Black Tree

### 🌳 **What is a Red-Black Tree?**

A Red-Black Tree is a **self-balancing Binary Search Tree (BST)** with the following invariants:

```
RED-BLACK TREE RULES:
1. Every node is either RED or BLACK
2. The root is always BLACK
3. Every leaf (null node) is BLACK
4. If a node is RED, both its children are BLACK
   (no two consecutive red nodes on any path)
5. All paths from any node to leaf nodes have the same number of BLACK nodes
   (black-height property)

These rules GUARANTEE: height ≤ 2 × log₂(n+1)
→ All operations remain O(log n) even in worst case!
```

```
TreeSet<Integer> set = new TreeSet<>(Set.of(7, 3, 18, 10, 22, 8, 11, 26));

Red-Black Tree visualization:
              7(B)
             / \
           3(B)  18(R)
                /    \
             10(B)   22(B)
            /   \       \
           8(R) 11(R)  26(R)

B=Black, R=Red
In-order traversal gives SORTED sequence: 3, 7, 8, 10, 11, 18, 22, 26
```

### 🔧 **TreeSet is Backed by TreeMap**

```java
// Actual TreeSet source code:
public class TreeSet<E> extends AbstractSet<E>
        implements NavigableSet<E>, Cloneable, Serializable {
    
    private transient NavigableMap<E, Object> m; // THE BACKING TreeMap!
    private static final Object PRESENT = new Object(); // dummy value
    
    public TreeSet() {
        this(new TreeMap<>()); // natural ordering
    }
    
    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator)); // custom ordering
    }
    
    public boolean add(E e) {
        return m.put(e, PRESENT) == null; // element = TreeMap key
    }
    
    public boolean contains(Object o) {
        return m.containsKey(o);
    }
    
    // Navigation methods delegate to TreeMap:
    public E floor(E e)   { return m.floorKey(e); }
    public E ceiling(E e) { return m.ceilingKey(e); }
    public E lower(E e)   { return m.lowerKey(e); }
    public E higher(E e)  { return m.higherKey(e); }
}
```

---

## 📐 SortedSet Interface

### 🔧 **SortedSet Methods**

```java
TreeSet<Integer> ts = new TreeSet<>(Set.of(1, 5, 10, 15, 20, 25, 30));

// ── SortedSet interface ──

// First and last element
int first = ts.first();    // 1 — minimum element
int last  = ts.last();     // 30 — maximum element

// Comparator
Comparator<Integer> comp = ts.comparator(); // null if natural ordering

// SUB-RANGES — returns LIVE VIEWS (backed by original set!)
SortedSet<Integer> head = ts.headSet(15);      // [1, 5, 10] — elements < 15
SortedSet<Integer> tail = ts.tailSet(15);      // [15, 20, 25, 30] — elements >= 15
SortedSet<Integer> sub  = ts.subSet(5, 20);    // [5, 10, 15] — 5 <= x < 20
//                                                INCLUSIVE ^ EXCLUSIVE ^

// IMPORTANT: These return VIEWS — modifying them modifies the original!
head.add(3);    // adds 3 to ts!
ts.add(13);     // visible in sub view if 5 <= 13 < 20
ts.add(25);     // NOT visible in sub (25 >= 20)
```

---

## 🧭 NavigableSet Interface

### 🔧 **Navigation Methods (The Power of TreeSet)**

```java
TreeSet<Integer> ts = new TreeSet<>(Set.of(10, 20, 30, 40, 50));

// ── FIND NEAREST ELEMENT ──

// floor(e) — GREATEST element <= e (or null if none)
ts.floor(35);  // 30 (≤ 35, as close as possible)
ts.floor(30);  // 30 (exactly 30)
ts.floor(5);   // null (no element ≤ 5)

// ceiling(e) — SMALLEST element >= e (or null if none)  
ts.ceiling(35); // 40 (≥ 35, as close as possible)
ts.ceiling(30); // 30 (exactly 30)
ts.ceiling(55); // null (no element ≥ 55)

// lower(e) — GREATEST element STRICTLY < e (or null if none)
ts.lower(30);   // 20 (< 30, strictly less)
ts.lower(35);   // 30 (< 35, strictly less)

// higher(e) — SMALLEST element STRICTLY > e (or null if none)
ts.higher(30);  // 40 (> 30, strictly greater)
ts.higher(35);  // 40 (> 35, strictly greater)

// ── POLLINGRESULTS (DESTRUCTIVE) ──

// pollFirst() — remove and return SMALLEST element
int min = ts.pollFirst(); // 10, ts = {20,30,40,50}

// pollLast() — remove and return LARGEST element
int max = ts.pollLast();  // 50, ts = {20,30,40}

// ── INCLUSIVE/EXCLUSIVE RANGES (NavigableSet extends SortedSet) ──

// subSet with inclusivity control
NavigableSet<Integer> sub = ts.subSet(20, true, 40, true);  // [20, 30, 40]
NavigableSet<Integer> sub2 = ts.subSet(20, false, 40, false); // [30]
NavigableSet<Integer> sub3 = ts.subSet(20, true, 40, false); // [20, 30]

// headSet with inclusivity
NavigableSet<Integer> head = ts.headSet(30, true);  // [20, 30] (inclusive)
NavigableSet<Integer> head2 = ts.headSet(30, false); // [20] (exclusive)

// tailSet with inclusivity
NavigableSet<Integer> tail = ts.tailSet(30, true);  // [30, 40]
NavigableSet<Integer> tail2 = ts.tailSet(30, false); // [40]

// ── REVERSED VIEW ──
NavigableSet<Integer> desc = ts.descendingSet(); // [40, 30, 20] — reversed view
Iterator<Integer> descIt = ts.descendingIterator(); // iterate in reverse
```

---

## 🛠️ All Methods Explained

```java
TreeSet<String> ts = new TreeSet<>();

// ── BASIC OPERATIONS ──
ts.add("banana");     // O(log n)
ts.add("apple");
ts.add("cherry");
ts.add("apple");      // false — duplicate, not added
// Current: [apple, banana, cherry]

ts.remove("banana");  // O(log n)
ts.contains("apple"); // true — O(log n)
ts.size();            // 2
ts.isEmpty();         // false
ts.clear();           // empty the set

// ── ADDING MULTIPLE ──
ts.addAll(Set.of("mango", "grape", "kiwi")); // add all

// ── SET OPERATIONS (inherited) ──
TreeSet<String> other = new TreeSet<>(Set.of("grape", "peach"));
ts.addAll(other);     // union (in place)
ts.retainAll(other);  // intersection (in place)
ts.removeAll(other);  // difference (in place)
ts.containsAll(other); // subset check

// ── NAVIGATION ──
// (see NavigableSet section above)

// ── ITERATION (always in sorted order!) ──
for (String s : ts) { System.out.println(s); }     // sorted ascending
ts.descendingSet().forEach(System.out::println);     // descending
ts.stream().forEach(System.out::println);            // sorted stream

// ── CONVERSION ──
Object[] arr = ts.toArray();
String[] strArr = ts.toArray(new String[0]);

// ── Java 21 (SequencedSet) ──
String f = ts.getFirst();  // = ts.first()
String l = ts.getLast();   // = ts.last()
ts.addFirst("aaa");        // adds at "front" (smallest) — same as add() since TreeSet sorts
ts.reversed();             // returns reversed view
```

---

## 🎨 Natural vs Custom Ordering

### 🔧 **Natural Ordering (Comparable)**

```java
// Works with types that implement Comparable:
// String, Integer, Long, Double, Date, LocalDate, etc.

TreeSet<String> strings = new TreeSet<>();
strings.addAll(List.of("banana", "Apple", "cherry", "Date"));
// Natural ordering for String: case-sensitive lexicographic (uppercase < lowercase)
System.out.println(strings); // [Apple, Date, banana, cherry]
// 'A' (65) < 'D' (68) < 'b' (98) < 'c' (99)

TreeSet<Integer> ints = new TreeSet<>(Set.of(5, 2, 8, 1, 9, 3));
System.out.println(ints); // [1, 2, 3, 5, 8, 9]

// Custom class must implement Comparable<T>:
class Student implements Comparable<Student> {
    String name;
    double gpa;
    
    @Override
    public int compareTo(Student other) {
        return Double.compare(other.gpa, this.gpa); // descending by GPA
        // return this.name.compareTo(other.name);  // ascending by name
    }
    
    // IMPORTANT: equals must be consistent with compareTo!
    // If compareTo returns 0, equals should return true
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Student s)) return false;
        return gpa == s.gpa && name.equals(s.name);
    }
    
    @Override
    public int hashCode() { return Objects.hash(name, gpa); }
}
```

### 🔧 **Custom Ordering (Comparator)**

```java
// When you CANNOT modify the class, or want DIFFERENT ordering:

// Sort by string LENGTH, then alphabetically
Comparator<String> byLength = Comparator.comparingInt(String::length)
                                         .thenComparing(Comparator.naturalOrder());

TreeSet<String> byLengthSet = new TreeSet<>(byLength);
byLengthSet.addAll(List.of("banana", "kiwi", "apple", "fig", "pear", "plum"));
System.out.println(byLengthSet); // [fig, kiwi, pear, plum, apple, banana]

// Case-insensitive TreeSet
TreeSet<String> caseInsensitive = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
caseInsensitive.add("Apple");
caseInsensitive.add("apple"); // DUPLICATE (case-insensitive!) — not added
System.out.println(caseInsensitive); // [Apple]

// Reverse natural ordering
TreeSet<Integer> desc = new TreeSet<>(Comparator.reverseOrder());
desc.addAll(List.of(3, 1, 4, 1, 5, 9));
System.out.println(desc); // [9, 5, 4, 3, 1]

// Multi-level sort: sort employees by department, then by salary descending
TreeSet<Employee> employees = new TreeSet<>(
    Comparator.comparing(Employee::getDepartment)
              .thenComparingDouble(Employee::getSalary).reversed()
);
```

### ⚠️ **Comparator Consistency Warning**

```java
// CRITICAL: Comparator must be CONSISTENT WITH EQUALS if used in TreeSet
// i.e., if comparator.compare(a, b) == 0, then a.equals(b) should be true

// ❌ PROBLEMATIC: Comparator by salary only — two employees with same salary
//    are considered "equal" by TreeSet (will not add both!)
TreeSet<Employee> salarySet = new TreeSet<>(Comparator.comparingDouble(Employee::getSalary));
Employee alice = new Employee("Alice", 50000);
Employee bob = new Employee("Bob", 50000);  // same salary!
salarySet.add(alice);
salarySet.add(bob);   // NOT added! comparator returns 0 → considered duplicate
System.out.println(salarySet.size()); // 1, not 2!

// ✅ FIX: Include a tiebreaker that makes all elements unique
TreeSet<Employee> fixedSet = new TreeSet<>(
    Comparator.comparingDouble(Employee::getSalary)
              .thenComparing(Employee::getName) // name as tiebreaker
);
fixedSet.add(alice);
fixedSet.add(bob);    // added! different name means comparator != 0
System.out.println(fixedSet.size()); // 2 ✅
```

---

## 🌍 Real-World Analogies

### 📚 **The Sorted Library Catalog**

```
TreeSet is like a library card catalog (sorted, searchable):

CATALOG:
  ...
  [Biology]
  [Chemistry]         ← floor("Cooking") = "Chemistry"
  [Cooking]           ← exact match
  [Economics]         ← ceiling("Data Science") = "Economics"
  [Fiction]
  [Geography]
  ...

✅ "What book is just before 'Cooking'?" → floor() = Chemistry
✅ "What book is just after 'Data Science'?" → ceiling() = Economics
✅ "All books between D and G?" → subSet("D", "G") = [Economics, Fiction, Geography]
✅ Always sorted → no explicit sorting needed!
❌ "Give me the 3rd book" → must iterate (no index access)
```

### 🏆 **Leaderboard (Top K)**

```java
// Top 5 high scores using TreeSet:
TreeSet<ScoreEntry> leaderboard = new TreeSet<>(
    Comparator.comparingInt(ScoreEntry::getScore).reversed()
              .thenComparing(ScoreEntry::getPlayerName) // tiebreaker
);
final int MAX_SIZE = 5;

public void addScore(ScoreEntry newScore) {
    leaderboard.add(newScore);
    if (leaderboard.size() > MAX_SIZE) {
        leaderboard.pollLast(); // remove the lowest score
    }
}

// Result: always sorted, always top 5
```

---

## ✅ When to USE TreeSet

```
✅ USE TreeSet WHEN:
─────────────────────────────────────────────────────────────────────
✅ You need elements SORTED at all times
   Example: sorted word list, ranking system, sorted timestamps

✅ You need RANGE QUERIES: subSet, headSet, tailSet
   Example: "Find all events between 9AM and 5PM"
   "Find all products priced between $10 and $50"

✅ You need NEAREST ELEMENT queries: floor, ceiling, lower, higher
   Example: "What is the nearest available appointment slot?"
   "Find the closest sensor reading to a target value"

✅ You need MINIMUM or MAXIMUM efficiently: first(), last()
   Example: Always know the current min/max without sorting

✅ You need REVERSE ITERATION: descendingIterator()
   Example: Process events from newest to oldest

REAL EXAMPLES:
  - Word frequency sorted by count: TreeSet<WordCount>
  - Sliding window maximum: TreeSet used as sorted multiset
  - Calendar / scheduling: TreeSet<LocalDateTime> appointments
  - Stock price feed: TreeSet<PricePoint> for range queries
  - Auto-complete: TreeSet<String> with tailSet(prefix)
```

---

## ❌ When to AVOID TreeSet

```
❌ AVOID TreeSet WHEN:
─────────────────────────────────────────────────────────────────────
❌ YOU DON'T NEED SORTING — just use HashSet (3-5x faster)
   HashSet: O(1) add/remove/contains
   TreeSet: O(log n) — logarithmic overhead for no benefit

❌ HIGH-FREQUENCY WRITES with large sets
   Every add/remove requires O(log n) tree rebalancing
   For 10M elements: ~23 comparisons per operation

❌ NULL ELEMENTS
   TreeSet.add(null) → NullPointerException (compareTo(null) fails)
   Use HashSet if you need null support

❌ ELEMENTS WITHOUT NATURAL ORDERING and no Comparator
   Will throw ClassCastException at runtime

❌ MULTI-THREADED CONCURRENT ACCESS
   TreeSet is not thread-safe
   Use: Collections.synchronizedSortedSet(new TreeSet<>())
        or ConcurrentSkipListSet (concurrent, sorted, O(log n))

❌ WHEN YOU JUST NEED A SORTED LIST
   If you need index-based access to sorted elements,
   just use a sorted ArrayList (ArrayList + Collections.sort())
```

---

## ⏱️ Time & Space Complexity

```
OPERATION          │ TreeSet    │ HashSet    │ Notes
───────────────────┼────────────┼────────────┼──────────────────────────────
add(E)             │ O(log n)   │ O(1)★      │ Tree rebalancing on every add
remove(Object)     │ O(log n)   │ O(1)       │
contains(Object)   │ O(log n)   │ O(1)       │
first() / last()   │ O(log n)   │ N/A        │ Traverse to leftmost/rightmost
floor(e)           │ O(log n)   │ N/A        │ BST search
ceiling(e)         │ O(log n)   │ N/A        │
lower(e)           │ O(log n)   │ N/A        │
higher(e)          │ O(log n)   │ N/A        │
pollFirst()        │ O(log n)   │ N/A        │
pollLast()         │ O(log n)   │ N/A        │
headSet/tailSet    │ O(log n)   │ N/A        │ Returns view (no copy)
subSet             │ O(log n)   │ N/A        │ Returns view (no copy)
iteration          │ O(n)       │ O(n)       │ TreeSet iterates in sorted order
size()             │ O(1)       │ O(1)       │
───────────────────┴────────────┴────────────┴──────────────────────────────
SPACE: O(n) — each node has: data, left ref, right ref, parent ref, color bit
              ~48 bytes per node (vs ~48 bytes for HashMap.Entry)
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Comparator Not Consistent with equals**

```java
// ❌ PROBLEM — comparator says they're equal but equals() says different
TreeSet<String> set = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
set.add("Hello");
set.add("hello"); // comparator.compare("Hello","hello") == 0 → treated as duplicate!
System.out.println(set); // ["Hello"] — only ONE element!

// This is CORRECT behavior per TreeSet contract:
// "A set contains no two elements e1 and e2 such that comparator.compare(e1, e2) == 0"
// But it may SURPRISE you if you expected both strings to be present.
```

### 💣 **Pitfall 2: subSet/headSet/tailSet Views Allow Range-Violating Operations**

```java
TreeSet<Integer> ts = new TreeSet<>(Set.of(1, 5, 10, 20, 30));
SortedSet<Integer> sub = ts.subSet(5, 20); // {5, 10, 15} but 20 exclusive

// DANGER: Adding through the view!
sub.add(15);   // OK — 5 <= 15 < 20, within range
sub.add(25);   // ❌ IllegalArgumentException: key out of range!
sub.add(5);    // OK — boundary is inclusive
sub.add(20);   // ❌ IllegalArgumentException: 20 is the exclusive upper bound
```

### 💣 **Pitfall 3: Mutating Elements After Adding**

```java
// Same as HashSet — DO NOT mutate elements after adding!
class Score implements Comparable<Score> {
    int value;
    Score(int v) { this.value = v; }
    public int compareTo(Score o) { return Integer.compare(this.value, o.value); }
}

TreeSet<Score> ts = new TreeSet<>();
Score s = new Score(50);
ts.add(s);
s.value = 10;  // MUTATED! Tree structure is now CORRUPTED!
ts.contains(s); // May return false or throw exception
// The Score is in the wrong position in the tree (was sorted as 50, now acts as 10)
```

---

## 🏢 Industry Applications

```
COMPANY          USE CASE                              COLLECTION
──────────────────────────────────────────────────────────────────────────────
Uber             Find nearest driver to a location    TreeSet<Driver> by distance
Stock Exchange   Order book (sorted by price+time)   TreeSet<Order>
Google Calendar  Find free/busy time slots            TreeSet<TimeSlot>
LinkedIn         Sorted job search results            TreeSet<Job> by relevance
Kafka            Message offset tracking              TreeSet<Long> offsets
Spring Batch     Sorted job execution order           TreeSet<Job>
Redis (Java)     Sorted sets (ZSET) — similar concept TreeMap / TreeSet internally
MongoDB          Index ranges ($gt, $lt queries)      B-Tree (similar to RBTree)
```

### 💻 **Real-World: Appointment Scheduler**

```java
@Service
public class AppointmentScheduler {
    
    // Always-sorted appointment set
    private final TreeSet<Appointment> appointments = new TreeSet<>(
        Comparator.comparing(Appointment::getStartTime)
                  .thenComparing(Appointment::getId) // tiebreaker for unique keys
    );
    
    // Find next available slot after a given time
    public Optional<Appointment> nextAvailableAfter(LocalDateTime time) {
        Appointment probe = new Appointment(time, null); // probe object
        Appointment higher = appointments.higher(probe);
        return Optional.ofNullable(higher);
    }
    
    // Find all appointments in a time range
    public SortedSet<Appointment> getAppointmentsInRange(
            LocalDateTime from, LocalDateTime to) {
        Appointment fromProbe = new Appointment(from, null);
        Appointment toProbe = new Appointment(to, null);
        return appointments.subSet(fromProbe, true, toProbe, false);
        // Returns a LIVE VIEW — O(log n) to find bounds, O(k) to iterate results
    }
    
    // Always know the next upcoming appointment
    public Optional<Appointment> nextUpcoming() {
        LocalDateTime now = LocalDateTime.now();
        Appointment probe = new Appointment(now, null);
        return Optional.ofNullable(appointments.ceiling(probe));
    }
    
    // Find conflicting appointments
    public boolean hasConflict(Appointment newAppt) {
        Appointment floor = appointments.floor(newAppt);
        if (floor != null && floor.overlaps(newAppt)) return true;
        
        Appointment ceiling = appointments.ceiling(newAppt);
        if (ceiling != null && ceiling.overlaps(newAppt)) return true;
        
        return false;
    }
}
```

### 💻 **Auto-Complete with TreeSet**

```java
public class AutoComplete {
    private final TreeSet<String> words = new TreeSet<>();
    
    public void addWord(String word) {
        words.add(word.toLowerCase());
    }
    
    // Find all words starting with prefix — O(log n + k) where k = results
    public List<String> suggest(String prefix, int maxSuggestions) {
        String lower = prefix.toLowerCase();
        // nextKey after all strings with this prefix: increment last char
        String upper = lower.substring(0, lower.length() - 1)
                     + (char)(lower.charAt(lower.length() - 1) + 1);
        
        return new ArrayList<>(
            words.subSet(lower, true, upper, false)
                 .stream()
                 .limit(maxSuggestions)
                 .collect(Collectors.toList())
        );
    }
}

// Usage:
AutoComplete ac = new AutoComplete();
ac.addWord("apple"); ac.addWord("application"); ac.addWord("apply");
ac.addWord("apt"); ac.addWord("banana");

ac.suggest("app", 10); // ["apple", "application", "apply"]
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | TreeSet Pattern |
|---|---------|-----------|-----------------|
| 1 | [Contains Duplicate III](https://leetcode.com/problems/contains-duplicate-iii/) | 🔴 Hard | TreeSet floor/ceiling for range |
| 2 | [My Calendar I](https://leetcode.com/problems/my-calendar-i/) | 🟡 Medium | TreeSet for interval conflicts |
| 3 | [My Calendar II](https://leetcode.com/problems/my-calendar-ii/) | 🟡 Medium | Double booking detection |
| 4 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | 🔴 Hard | TreeMap/TreeSet as sliding window |
| 5 | [Find Smallest Letter Greater Than Target](https://leetcode.com/problems/find-smallest-letter-greater-than-target/) | 🟢 Easy | ceiling() pattern |
| 6 | [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | 🟡 Medium | TreeSet with custom comparator |
| 7 | [Count of Range Sum](https://leetcode.com/problems/count-of-range-sum/) | 🔴 Hard | TreeSet range counting |
| 8 | [Design In-Memory File System](https://leetcode.com/problems/design-in-memory-file-system/) | 🔴 Hard | TreeSet for sorted directory listing |

---

## 💡 Interview Tips

**Q1: What is the internal data structure of TreeSet?**
```
TreeSet is backed by TreeMap which uses a Red-Black Tree.
Red-Black Tree is a self-balancing BST guaranteeing height ≤ 2×log₂(n+1).
This guarantees O(log n) for all basic operations.
The tree stays balanced by using color (red/black) and rotations on insert/delete.
```

**Q2: When would you use TreeSet over a sorted ArrayList?**
```
TreeSet wins when:
  - Frequent insertions/deletions while maintaining sort: O(log n) vs O(n)
  - Range queries (subSet, headSet): TreeSet returns a live view in O(log n)
  - Finding nearest element (floor/ceiling): O(log n) vs O(log n) binary search
  - Duplicate-free guarantee: TreeSet enforces it automatically

Sorted ArrayList wins when:
  - Build once, read many: sort once with Collections.sort(), then binarySearch
  - Need index access: list.get(i) is O(1) vs O(log n) traversal
  - Memory matters: ArrayList uses less memory per element
```

**Q3: How does TreeSet.floor() differ from TreeSet.lower()?**
```
floor(e):   Returns GREATEST element ≤ e (includes e if present)
lower(e):   Returns GREATEST element STRICTLY < e (excludes e even if present)

ceiling(e): Returns SMALLEST element ≥ e (includes e if present)
higher(e):  Returns SMALLEST element STRICTLY > e (excludes e even if present)

Example with set {10, 20, 30}:
  floor(20)   = 20  (exactly 20)
  lower(20)   = 10  (strictly less than 20)
  ceiling(20) = 20  (exactly 20)
  higher(20)  = 30  (strictly greater than 20)
```

**Q4: Can you have a sorted collection with duplicates?**
```java
// TreeSet does NOT allow duplicates (Set contract).
// If you need SORTED with duplicates, you have 3 options:

// Option 1: TreeMap<Value, Integer> as frequency map
TreeMap<Integer, Integer> sortedMultiset = new TreeMap<>();
sortedMultiset.merge(5, 1, Integer::sum); // add 5
sortedMultiset.merge(5, 1, Integer::sum); // add another 5

// Option 2: List + Collections.sort() (for static data)
List<Integer> sorted = new ArrayList<>(Arrays.asList(5,3,5,1,3));
Collections.sort(sorted); // [1,3,3,5,5]

// Option 3: TreeMap<Value, List<Value>> or Guava's TreeMultimap
```

---

*Previous: [HashSet & LinkedHashSet ←](./04_HashSet_LinkedHashSet.md) | Next: [HashMap Deep Dive →](./06_HashMap_Deep_Dive.md)*
