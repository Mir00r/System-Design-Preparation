# 🔵 Java HashSet & LinkedHashSet: The Set Masterclass 🚀

---

## 🎯 Chapter 4: Set Interface, HashSet & LinkedHashSet

> **"A Set is like a guest list for an exclusive party — each person can only be on the list ONCE, and trying to add them again is silently ignored."**

---

## 📋 Table of Contents

1. [The Set Interface Contract](#the-set-interface-contract)
2. [What is HashSet?](#what-is-hashset)
3. [Internal Architecture — HashSet IS a HashMap](#internal-architecture)
4. [HashSet All Methods](#hashset-all-methods)
5. [The equals() and hashCode() Contract](#equals-and-hashcode-contract)
6. [What is LinkedHashSet?](#what-is-linkedhashset)
7. [LinkedHashSet vs HashSet](#linkedhashset-vs-hashset)
8. [EnumSet — The Performance Champion](#enumset)
9. [Real-World Analogies](#real-world-analogies)
10. [When to USE HashSet vs LinkedHashSet](#when-to-use)
11. [When to AVOID](#when-to-avoid)
12. [Time & Space Complexity](#time--space-complexity)
13. [Common Pitfalls](#common-pitfalls)
14. [Industry Applications](#industry-applications)
15. [Practice Problems](#practice-problems)
16. [Interview Tips](#interview-tips)

---

## 🔌 The Set Interface Contract

### 📖 **What Set Guarantees**

The `Set<E>` interface extends `Collection<E>` and adds exactly **one contract**: **no duplicate elements**.

```
A Set is a Collection that:
✅ Contains NO DUPLICATE elements
✅ At most ONE null element (for HashSet/LinkedHashSet)
✅ Operations: add, remove, contains, size, isEmpty, iterator
❌ NO index-based access (no get(i))
❌ NO positional insertion (no add(index, element))

The add(E e) method returns:
  true  → element was added (new unique element)
  false → element was NOT added (already existed, no change)
```

```java
// java.util.Set<E>
public interface Set<E> extends Collection<E> {
    // From Collection — but with stricter contract:
    boolean add(E e);           // returns false if already present
    boolean addAll(Collection<? extends E> c); // adds only non-duplicates
    
    // Same as Collection: remove, contains, size, isEmpty, iterator, etc.
    
    // Java 9+ immutable factory:
    static <E> Set<E> of(E... elements); // THROWS if duplicates!
    static <E> Set<E> copyOf(Collection<? extends E> c); // removes duplicates
}
```

---

## 🤔 What is HashSet?

### 📖 **Definition**

`HashSet<E>` is the most commonly used Set implementation. It stores elements using a **hash table** (backed by a `HashMap`) with no guarantees about **iteration order**.

```
HashSet characteristics:
✅ O(1) add, remove, contains (amortized)
✅ Allows ONE null element
❌ No guaranteed order (can change between runs, JVM versions!)
❌ NOT thread-safe
❌ No index-based access

Introduced: Java 1.2 (1998)
Backed by: HashMap<E, Object> (a HashMap where all values are a dummy object)
```

---

## 🏗️ Internal Architecture — HashSet IS a HashMap

### 🔧 **The Surprising Truth**

```java
// Actual HashSet source code (OpenJDK):
public class HashSet<E> extends AbstractSet<E>
        implements Set<E>, Cloneable, Serializable {

    private transient HashMap<E, Object> map;  // ← THE BACKING MAP!
    
    // Dummy value to associate with every key in the backing Map
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();
    }

    public boolean add(E e) {
        return map.put(e, PRESENT) == null;  // element becomes the MAP KEY!
        // map.put returns null if key was new → return true (added)
        // map.put returns old value (PRESENT) if key existed → return false (duplicate)
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;  // removes from backing HashMap
    }

    public boolean contains(Object o) {
        return map.containsKey(o);  // delegates to HashMap
    }

    public int size() {
        return map.size();
    }

    public Iterator<E> iterator() {
        return map.keySet().iterator(); // iterates HashMap keys
    }
}
```

```
MEMORY LAYOUT:
┌──────────────────────────────────────────────────────────────────┐
│  HashSet { "Alice", "Bob", "Charlie" }                           │
│                                                                  │
│  map → HashMap {                                                 │
│    "Alice"   → PRESENT (dummy Object @0x1234)                   │
│    "Bob"     → PRESENT (same dummy Object @0x1234)              │
│    "Charlie" → PRESENT (same dummy Object @0x1234)              │
│  }                                                               │
│                                                                  │
│  NOTE: All values point to the SAME PRESENT object!             │
│        Memory-efficient design.                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 📊 **How HashSet Determines Duplicates**

```
STEP 1: Compute hash of the element
  hash = element.hashCode()
  
STEP 2: Find bucket in internal array
  bucketIndex = hash & (capacity - 1)  // faster than modulo
  
STEP 3: Check all elements in that bucket
  For each existing element at bucketIndex:
    If existingElement.hashCode() == newElement.hashCode()
    AND (existingElement == newElement OR existingElement.equals(newElement))
    → DUPLICATE! Return false, do NOT add.
  
STEP 4: If no duplicate found → add to bucket chain

TWO CONDITIONS MUST BOTH BE TRUE for a duplicate:
  1. hashCode() values are EQUAL
  2. equals() returns true
  
If only hashCode() matches but equals() returns false → NOT a duplicate (hash collision)
```

---

## 🛠️ HashSet All Methods

```java
HashSet<String> set = new HashSet<>();

// ── ADDING ──
boolean added = set.add("Alice");    // true (new)
boolean again = set.add("Alice");    // false (duplicate, set unchanged!)
set.addAll(Set.of("Bob", "Charlie", "Alice")); // only Bob and Charlie added

// ── REMOVING ──
boolean removed = set.remove("Bob"); // true (was present)
boolean notThere = set.remove("Dave"); // false (wasn't there)
set.removeAll(Set.of("Charlie"));    // remove all matching
set.retainAll(Set.of("Alice"));      // keep only Alice (removes others)
set.removeIf(s -> s.startsWith("A")); // Java 8+ — remove matching
set.clear();                         // empty the set

// ── QUERYING ──
boolean has = set.contains("Alice"); // true — O(1)
int sz = set.size();                 // number of elements
boolean empty = set.isEmpty();       // true if size == 0

// ── SET OPERATIONS ──
Set<String> a = new HashSet<>(Set.of("1","2","3"));
Set<String> b = new HashSet<>(Set.of("2","3","4"));

// UNION
Set<String> union = new HashSet<>(a);
union.addAll(b);                     // {1, 2, 3, 4}

// INTERSECTION
Set<String> intersection = new HashSet<>(a);
intersection.retainAll(b);           // {2, 3}

// DIFFERENCE (a - b)
Set<String> difference = new HashSet<>(a);
difference.removeAll(b);             // {1}

// SYMMETRIC DIFFERENCE (a XOR b)
Set<String> symDiff = new HashSet<>(a);
symDiff.addAll(b);
Set<String> common = new HashSet<>(a);
common.retainAll(b);
symDiff.removeAll(common);           // {1, 4}

// SUBSET CHECK
boolean isSubset = a.containsAll(Set.of("1","2")); // true

// ── CONVERSION ──
Object[] arr = set.toArray();
String[] strArr = set.toArray(new String[0]);
List<String> list = new ArrayList<>(set); // Set → List (order not guaranteed)

// ── ITERATION ──
for (String s : set) { System.out.println(s); } // order not guaranteed!
set.forEach(System.out::println);
set.stream().filter(...).collect(Collectors.toSet());
```

---

## 🔑 The equals() and hashCode() Contract

### 📖 **The Most Important Rule in Java**

```
GOLDEN CONTRACT:
If a.equals(b) is true, then a.hashCode() MUST equal b.hashCode()

(The reverse is NOT required — same hashCode doesn't mean equal)

WHAT BREAKS IF YOU VIOLATE THIS:
```

```java
// ❌ BROKEN CLASS — only overrides equals, not hashCode
class BadKey {
    String name;
    BadKey(String name) { this.name = name; }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof BadKey)) return false;
        return name.equals(((BadKey) o).name);
    }
    // ❌ hashCode() NOT overridden → uses Object.hashCode() = memory address!
}

Set<BadKey> set = new HashSet<>();
set.add(new BadKey("Alice"));
set.contains(new BadKey("Alice")); // FALSE! 💥 Different hashCode → different bucket!

// HashSet's logic:
// hashCode("Alice"@0x1234) = some hash of memory address
// hashCode("Alice"@0x5678) = DIFFERENT hash (different object!)
// Goes to different bucket → never finds the equal element!
```

```java
// ✅ CORRECT CLASS — overrides BOTH equals AND hashCode
class GoodKey {
    String name;
    GoodKey(String name) { this.name = name; }
    
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof GoodKey g)) return false;
        return Objects.equals(name, g.name);
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(name); // consistent with equals
    }
}

Set<GoodKey> set = new HashSet<>();
set.add(new GoodKey("Alice"));
set.contains(new GoodKey("Alice")); // TRUE ✅

// Java 16+ Records automatically implement equals+hashCode correctly:
record PersonKey(String name, int age) {}  // auto-generated equals+hashCode
Set<PersonKey> personSet = new HashSet<>();
personSet.add(new PersonKey("Alice", 30));
personSet.contains(new PersonKey("Alice", 30)); // TRUE ✅
```

### 💡 **IDE-Generated equals/hashCode (Best Practice)**

```java
// Java 7+ — use Objects.hash() and Objects.equals():
class Product {
    private Long id;
    private String sku;
    private String name;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Product p)) return false;
        return Objects.equals(id, p.id) && Objects.equals(sku, p.sku);
        // Use ONLY the fields that define logical equality (usually ID/key fields)
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(id, sku);  // SAME fields as equals!
    }
}

// Spring Boot / JPA entities — use @EqualsAndHashCode from Lombok:
@Entity
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
class Order {
    @Id @EqualsAndHashCode.Include
    private Long id;
    // ... other fields excluded from equals/hashCode
}
```

---

## 🔵 What is LinkedHashSet?

### 📖 **Definition**

`LinkedHashSet<E>` extends `HashSet<E>` and maintains **insertion order** of elements. It is backed by `LinkedHashMap` instead of `HashMap`.

```
LinkedHashSet characteristics:
✅ O(1) add, remove, contains (amortized)
✅ Maintains INSERTION ORDER — iterates in order elements were added
✅ Allows ONE null element
❌ Slightly more memory than HashSet (doubly-linked list overhead)
❌ NOT thread-safe

Introduced: Java 1.4 (2002)
Backed by: LinkedHashMap<E, Object>
```

```java
// HashSet — NO order guarantee:
Set<String> hashSet = new HashSet<>(List.of("banana", "apple", "cherry"));
System.out.println(hashSet); // might print: [banana, cherry, apple] (any order!)

// LinkedHashSet — INSERTION ORDER:
Set<String> linkedSet = new LinkedHashSet<>(List.of("banana", "apple", "cherry"));
System.out.println(linkedSet); // ALWAYS prints: [banana, apple, cherry]

// Adding more elements:
linkedSet.add("date");
linkedSet.add("apple");    // duplicate — ignored! Order not affected.
System.out.println(linkedSet); // [banana, apple, cherry, date]
// "apple" stays at its ORIGINAL position, not moved to end!
```

---

## 🔄 LinkedHashSet vs HashSet

```
FEATURE              │ HashSet              │ LinkedHashSet
─────────────────────┼──────────────────────┼─────────────────────────────
Iteration order      │ ❌ No guarantee       │ ✅ Insertion order
Memory               │ Less                 │ More (+linked list overhead)
Performance          │ Slightly faster      │ Slightly slower (pointer update)
Backed by            │ HashMap              │ LinkedHashMap
Null allowed         │ ✅ One null           │ ✅ One null
Use case             │ O(1) dedup, no order │ Dedup + preserve order
```

```java
// Use case: Remove duplicates while preserving order
List<String> withDuplicates = List.of("c", "a", "b", "a", "c", "d");

// Approach 1: HashSet — deduplicates but loses order
Set<String> deduped = new HashSet<>(withDuplicates); // {a, b, c, d} — unordered

// Approach 2: LinkedHashSet — deduplicates AND preserves first-occurrence order
Set<String> orderedDeduped = new LinkedHashSet<>(withDuplicates); // [c, a, b, d]

// Approach 3: Java 8 Stream (also preserves order)
List<String> result = withDuplicates.stream()
    .distinct()
    .collect(Collectors.toList()); // [c, a, b, d]
```

---

## ⚡ EnumSet — The Performance Champion

### 📖 **Special Set for Enum Types**

`EnumSet` is the FASTEST Set implementation when working with enum values. It uses a **bit vector** internally (a long or array of longs).

```java
enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

// EnumSet — backed by bit manipulation!
EnumSet<Day> weekdays = EnumSet.of(Day.MON, Day.TUE, Day.WED, Day.THU, Day.FRI);
EnumSet<Day> weekend = EnumSet.of(Day.SAT, Day.SUN);
EnumSet<Day> allDays = EnumSet.allOf(Day.class);
EnumSet<Day> noDays = EnumSet.noneOf(Day.class);
EnumSet<Day> midWeek = EnumSet.range(Day.TUE, Day.THU); // {TUE, WED, THU}
EnumSet<Day> complement = EnumSet.complementOf(weekdays); // {SAT, SUN}

// Operations:
weekdays.contains(Day.MON);  // O(1) — single bit check!
weekdays.add(Day.SAT);
weekdays.remove(Day.FRI);

// PERFORMANCE:
// EnumSet.contains(): single bit check → FASTER than any other Set
// EnumSet.add():      single bit set   → FASTER than HashSet (no hashing!)
// Memory: 1 long (8 bytes) for up to 64 enum values!
```

```
WHEN TO USE EnumSet:
✅ Working with enum constants
✅ Need to represent a subset of enum values
✅ Access patterns (days of week, permissions, feature flags)

Example: Permission system
  enum Permission { READ, WRITE, DELETE, ADMIN, AUDIT }
  Set<Permission> userPerms = EnumSet.of(Permission.READ, Permission.WRITE);
  if (userPerms.contains(Permission.DELETE)) { ... }  // instant bit check
```

---

## 🌍 Real-World Analogies

### 🎭 **Concert Guest List (HashSet)**

```
BOUNCER with HashSet:
"Is Alice on the list?" → hash("Alice") → check bucket 7 → YES → allow in
"Add Bob to list" → hash("Bob") → check bucket 3 → not there → add
"Add Alice to list" → hash("Alice") → check bucket 7 → already there! → reject

✅ Instant lookup (O(1)) — bouncer can check ANY name instantly
❌ List has no ordering — was Bob added before or after Charlie? Unknown!
```

### 📝 **Ordered Register (LinkedHashSet)**

```
SECRETARY with LinkedHashSet:
Registration log (preserves signup order):
  1. Alice (first)
  2. Charlie
  3. Bob
  4. Dave

"Add Alice again" → She's already registered! No change. Still at position 1.
"Who was 2nd?" → Can iterate in order: Alice, Charlie, Bob, Dave

✅ Fast lookup AND remembers order of first registration
```

---

## ✅ When to USE HashSet vs LinkedHashSet

```
USE HashSet WHEN:
✅ You need fast deduplication (no order needed)
   Example: Unique user IDs, visited URLs, processed order IDs
✅ You only need contains/add/remove — no ordering required
✅ Memory is a concern (HashSet uses less memory than LinkedHashSet)
✅ Maximum performance priority

Real examples:
  - "Have we processed this order ID before?" → HashSet<Long> processedIds
  - Unique words in a document → Set<String> uniqueWords = new HashSet<>()
  - Remove duplicates from large list (if order doesn't matter)
  - Graph visited nodes in BFS/DFS

──────────────────────────────────────────────────────────────────────

USE LinkedHashSet WHEN:
✅ You need deduplication AND insertion order preservation
   Example: Recent items list (no duplicates, in order visited)
✅ You need a predictable iteration order (for testing, display)
✅ Implementing a uniqueness filter that preserves first occurrence

Real examples:
  - User's recently viewed products (no duplicates, show in order visited)
  - Unique tags on a blog post (insertion order = author's order)
  - De-duplicate a stream of events while preserving order
  - Any "first seen" collection

──────────────────────────────────────────────────────────────────────

USE TreeSet WHEN:
✅ You need elements sorted (see next chapter!)
```

---

## ❌ When to AVOID

```
AVOID HashSet WHEN:
❌ You need elements sorted → use TreeSet
❌ You need insertion order → use LinkedHashSet
❌ Multi-threaded access → use Collections.synchronizedSet(new HashSet<>())
   or ConcurrentHashMap.newKeySet()
❌ Your elements don't properly implement equals/hashCode
   (adds will appear to work but contains() will always return false!)

AVOID LinkedHashSet WHEN:
❌ Order doesn't matter (pay memory overhead for nothing)
❌ You need elements sorted → use TreeSet
```

---

## ⏱️ Time & Space Complexity

```
OPERATION      │ HashSet    │ LinkedHashSet │ TreeSet    │ Notes
───────────────┼────────────┼───────────────┼────────────┼────────────────────────
add(E)         │ O(1)★      │ O(1)★         │ O(log n)   │ ★ amortized
remove(E)      │ O(1)       │ O(1)          │ O(log n)   │
contains(E)    │ O(1)       │ O(1)          │ O(log n)   │
size()         │ O(1)       │ O(1)          │ O(1)       │
iteration      │ O(n)       │ O(n)          │ O(n)       │ LL is slightly slower
retainAll      │ O(n)       │ O(n)          │ O(n log n) │
addAll         │ O(k)★      │ O(k)★         │ O(k log n) │ k = added elements
───────────────┴────────────┴───────────────┴────────────┴────────────────────────
SPACE: O(n) for all.
  HashSet:       ~48 bytes per entry (HashMap.Entry)
  LinkedHashSet: ~56 bytes per entry (LinkedHashMap.Entry + prev/next)
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Mutable Object as Set Element**

```java
Set<List<Integer>> set = new HashSet<>();
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
set.add(list);

System.out.println(set.contains(list)); // true

list.add(4);   // MUTATE the list! hashCode changes!

System.out.println(set.contains(list)); // FALSE! 💥
// The set still has the list reference, but can't FIND it
// because the hashCode has changed and it's in the wrong bucket!
// This is an ORPHANED ELEMENT — it exists in the set but is unreachable!

// RULE: NEVER use mutable objects as Set elements (or Map keys)!
// Use: String, Integer, UUID, records, or other immutable types
```

### 💣 **Pitfall 2: Set.of() Throws on Duplicates**

```java
// Java 9+
Set<String> s = Set.of("a", "b", "a"); // ❌ IllegalArgumentException: duplicate element

// Use this instead to silently ignore duplicates:
Set<String> s = new HashSet<>(List.of("a", "b", "a")); // OK, size=2
```

### 💣 **Pitfall 3: Null in TreeSet**

```java
TreeSet<String> ts = new TreeSet<>();
ts.add("a");
ts.add(null);  // ❌ NullPointerException!
               // TreeSet uses compareTo() which fails on null

HashSet<String> hs = new HashSet<>();
hs.add(null);  // ✅ OK — one null allowed
```

---

## 🏢 Industry Applications

```
COMPANY           USE CASE                              IMPLEMENTATION
──────────────────────────────────────────────────────────────────────────────
Amazon            Recently viewed products (no dup)    LinkedHashSet<ProductId>
Google Search     De-duplicate URL results             HashSet<String> uniqueUrls
Netflix           Unique content IDs per user feed     HashSet<Long> contentIds
Twitter/X         Unique follower IDs                  HashSet<UserId> followers
Kafka             Deduplicate event processing         Set<String> processedKeys
Spring Security   Role-based authorization             Set<GrantedAuthority>
                    EnumSet<Permission> userPermissions
Hibernate         Bidirectional @ManyToMany            Set<Entity> (recommended over List!)
```

### 💻 **Real Spring Boot Example**

```java
@Service
public class ProductRecommendationService {

    // De-duplicate viewed products, preserve order of first view
    public List<Product> getUniqueViewedProducts(String userId) {
        List<Long> viewHistory = userActivityRepository.getViewHistory(userId);
        
        // LinkedHashSet: no duplicates, preserves first-view order
        Set<Long> uniqueIds = new LinkedHashSet<>(viewHistory);
        
        return productRepository.findAllById(uniqueIds);
    }
    
    // Fast duplicate check for real-time event processing
    private final Set<String> processedEventIds = 
        Collections.newSetFromMap(new ConcurrentHashMap<>());
    
    public void processEvent(Event event) {
        if (!processedEventIds.add(event.getId())) {
            return; // duplicate event, skip
        }
        // process event...
    }
    
    // Permission check using EnumSet
    public boolean hasPermission(User user, Permission required) {
        EnumSet<Permission> userPerms = EnumSet.copyOf(user.getPermissions());
        return userPerms.contains(required); // O(1) bit check
    }
}
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | Set Usage |
|---|---------|-----------|-----------|
| 1 | [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) | 🟢 Easy | HashSet.add() returns false for dup |
| 2 | [Happy Number](https://leetcode.com/problems/happy-number/) | 🟢 Easy | HashSet to detect cycle |
| 3 | [Intersection of Two Arrays](https://leetcode.com/problems/intersection-of-two-arrays/) | 🟢 Easy | retainAll or separate sets |
| 4 | [First Unique Character in a String](https://leetcode.com/problems/first-unique-character-in-a-string/) | 🟢 Easy | HashSet for duplicates |
| 5 | [Longest Consecutive Sequence](https://leetcode.com/problems/longest-consecutive-sequence/) | 🟡 Medium | HashSet for O(n) sequence finding |
| 6 | [Word Pattern](https://leetcode.com/problems/word-pattern/) | 🟢 Easy | Two HashSets |
| 7 | [4Sum II](https://leetcode.com/problems/4sum-ii/) | 🟡 Medium | HashMap + Set tricks |
| 8 | [Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | 🟡 Medium | HashMap + ArrayList for O(1) random |

---

## 💡 Interview Tips

**Q1: How does HashSet detect duplicates internally?**
```
HashSet is backed by HashMap where elements are keys.
When add(e) is called:
  1. Compute hash: h = e.hashCode(), then spread: h ^ (h >>> 16)
  2. Find bucket: index = h & (capacity - 1)
  3. Check all entries in bucket:
     if entry.hash == h && (entry.key == e || entry.key.equals(e)):
       → DUPLICATE, map.put() returns old PRESENT value, add() returns false
  4. If no duplicate found: add new entry, return true

So duplicates require:
  - SAME hashCode() AND
  - equals() returns true
```

**Q2: What is the difference between Set.of() and new HashSet<>()?**
```java
Set<String> immutable = Set.of("a", "b", "c");
// ✅ Null-free, immutable, throws on add/remove/set
// ✅ Throws IllegalArgumentException if duplicate elements provided
// ✅ Compact JVM-optimized representation (not actually a HashSet!)
// ❌ Cannot modify

Set<String> mutable = new HashSet<>(List.of("a", "b", "c"));
// ✅ Can add/remove
// ✅ Silently ignores duplicates on construction
// ✅ Is an actual HashSet (HashMap-backed)
```

**Q3: Why should you use Set for @ManyToMany in Hibernate?**
```
Using List for @ManyToMany causes the "HHH90003004 warning":
  Hibernate loads the ENTIRE collection to check for duplicates
  when adding an element, and uses delete-all + re-insert strategy.

Using Set: Hibernate checks by primary key (equals/hashCode on ID),
  can add/remove individual elements without loading everything.
  Much more efficient for large collections.

Rule: In JPA, @OneToMany and @ManyToMany should use Set<Entity>,
      and entities MUST override equals/hashCode based on their ID.
```

---

*Previous: [Stack, Vector & Legacy ←](./03_Stack_Vector_Legacy.md) | Next: [TreeSet Deep Dive →](./05_TreeSet.md)*
