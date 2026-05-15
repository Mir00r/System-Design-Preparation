# ⚡ Hash Tables: The Speed Demon of Data Structures! 🚀

---

## 🎯 Chapter 1: Hash Table Fundamentals

> **"Hash Tables: Where finding a needle in a haystack takes O(1) time!"**

---

## 🤔 What is a Hash Table?

A **Hash Table** (also called Hash Map) is a data structure that implements an **associative array** (key-value pairs) using a **hash function** to compute an index into an array of buckets, providing **O(1) average-time complexity** for search, insert, and delete operations.

### 🏗️ **Visual Representation**

```
Hash Table:
┌───────────────────────────────────────┐
│  Index  │  Key    │  Value            │
├───────────────────────────────────────┤
│    0    │  "cat"  →  "meow"           │
│    1    │   -     │   -               │
│    2    │  "dog"  →  "woof"           │
│    3    │  "cow"  →  "moo"            │
│    4    │   -     │   -               │
│    5    │  "pig"  →  "oink"           │
└───────────────────────────────────────┘
         ↑
    Hash Function: hash("cat") % 6 = 0
```

---

## 🎨 How Hash Functions Work

### 🔑 **The Magic Formula**

```
Index = hash(key) % array_size

Example:
hash("Alice") = 65498  
Index = 65498 % 10 = 8

┌─────┐
│  0  │
│  1  │
│  2  │
│  3  │
│  4  │
│  5  │
│  6  │
│  7  │
│  8  │ ← "Alice" stored here
│  9  │
└─────┘
```

### 📊 **Good Hash Function Properties**

✅ **Deterministic**: Same input → same output  
✅ **Uniform distribution**: Spreads keys evenly  
✅ **Fast to compute**: O(1) time  
✅ **Minimizes collisions**: Different keys → different indices  

---

## 💻 Java HashMap Implementation Essentials

### 🔧 **Internal Structure**

```java
// Simplified HashMap structure
class HashMap<K, V> {
    private Entry<K, V>[] table;  // Array of linked lists (buckets)
    private int size;
    private static final int DEFAULT_CAPACITY = 16;
    private static final float LOAD_FACTOR = 0.75f;
    
    static class Entry<K, V> {
        K key;
        V value;
        int hash;
        Entry<K, V> next; // For collision handling (chaining)
        
        Entry(K key, V value, int hash) {
            this.key = key;
            this.value = value;
            this.hash = hash;
        }
    }
    
    public HashMap() {
        table = new Entry[DEFAULT_CAPACITY];
    }
    
    private int hash(K key) {
        return key == null ? 0 : Math.abs(key.hashCode());
    }
    
    private int getIndex(int hash) {
        return hash % table.length;
    }
    
    // Put - O(1) average
    public V put(K key, V value) {
        int hash = hash(key);
        int index = getIndex(hash);
        
        Entry<K, V> entry = table[index];
        
        // Check if key exists (update)
        while (entry != null) {
            if (entry.hash == hash && entry.key.equals(key)) {
                V oldValue = entry.value;
                entry.value = value;
                return oldValue;
            }
            entry = entry.next;
        }
        
        // Insert new entry
        Entry<K, V> newEntry = new Entry<>(key, value, hash);
        newEntry.next = table[index];
        table[index] = newEntry;
        size++;
        
        // Resize if needed
        if (size > table.length * LOAD_FACTOR) {
            resize();
        }
        
        return null;
    }
    
    // Get - O(1) average
    public V get(K key) {
        int hash = hash(key);
        int index = getIndex(hash);
        
        Entry<K, V> entry = table[index];
        
        while (entry != null) {
            if (entry.hash == hash && entry.key.equals(key)) {
                return entry.value;
            }
            entry = entry.next;
        }
        
        return null;
    }
    
    // Remove - O(1) average
    public V remove(K key) {
        int hash = hash(key);
        int index = getIndex(hash);
        
        Entry<K, V> entry = table[index];
        Entry<K, V> prev = null;
        
        while (entry != null) {
            if (entry.hash == hash && entry.key.equals(key)) {
                if (prev == null) {
                    table[index] = entry.next;
                } else {
                    prev.next = entry.next;
                }
                size--;
                return entry.value;
            }
            prev = entry;
            entry = entry.next;
        }
        
        return null;
    }
    
    private void resize() {
        Entry<K, V>[] oldTable = table;
        table = new Entry[oldTable.length * 2];
        size = 0;
        
        for (Entry<K, V> entry : oldTable) {
            while (entry != null) {
                put(entry.key, entry.value);
                entry = entry.next;
            }
        }
    }
}
```

---

## ⚔️ Collision Resolution Strategies

### 🔗 **1. Chaining (Separate Chaining)**

```
Index 0: null
Index 1: null
Index 2: [("cat", "meow")] → [("bat", "screech")] → null
Index 3: [("dog", "woof")] → null
Index 4: null

// Both "cat" and "bat" hash to index 2 (collision)
// Stored in linked list at index 2
```

**Code Example**:
```java
// Java's HashMap uses chaining
HashMap<String, String> map = new HashMap<>();
map.put("cat", "meow");  // hash("cat") % 16 = 5
map.put("bat", "screech"); // hash("bat") % 16 = 5 (collision!)

// Internally:
// table[5] → [("cat", "meow")] → [("bat", "screech")] → null
```

---

### 🎯 **2. Open Addressing (Linear Probing)**

```
// If collision occurs, find next empty slot

Index 0: null
Index 1: ("cat", "meow")   ← hash("cat") = 1
Index 2: ("bat", "screech") ← hash("bat") = 1, but occupied, so try 2
Index 3: null
Index 4: null
```

**Code Example**:
```java
// Put with linear probing
public void put(K key, V value) {
    int index = hash(key) % capacity;
    
    while (table[index] != null && !table[index].key.equals(key)) {
        index = (index + 1) % capacity; // Linear probing
    }
    
    table[index] = new Entry<>(key, value);
}
```

---

## 🎯 Common HashMap Patterns

### 🔥 **Pattern 1: Two Sum Problem**

```java
public int[] twoSum(int[] nums, int target) {
    HashMap<Integer, Integer> map = new HashMap<>();
    
    for (int i = 0; i < nums.length; i++) {
        int complement = target - nums[i];
        
        if (map.containsKey(complement)) {
            return new int[] {map.get(complement), i};
        }
        
        map.put(nums[i], i);
    }
    
    return new int[] {-1, -1};
}

// Example: nums = [2, 7, 11, 15], target = 9
// map = {2: 0}
// complement = 9 - 7 = 2 (found at index 0)
// return [0, 1]
```

---

### 🔥 **Pattern 2: Frequency Counter**

```java
public char firstNonRepeating(String s) {
    HashMap<Character, Integer> freqMap = new HashMap<>();
    
    // Count frequencies
    for (char c : s.toCharArray()) {
        freqMap.put(c, freqMap.getOrDefault(c, 0) + 1);
    }
    
    // Find first non-repeating
    for (char c : s.toCharArray()) {
        if (freqMap.get(c) == 1) {
            return c;
        }
    }
    
    return '\0';
}

// Example: "leetcode" → 'l'
```

---

### 🔥 **Pattern 3: Group Anagrams**

```java
public List<List<String>> groupAnagrams(String[] strs) {
    HashMap<String, List<String>> map = new HashMap<>();
    
    for (String str : strs) {
        char[] chars = str.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        
        map.putIfAbsent(key, new ArrayList<>());
        map.get(key).add(str);
    }
    
    return new ArrayList<>(map.values());
}

// Example: ["eat","tea","tan","ate","nat","bat"]
// Result: [["bat"],["nat","tan"],["ate","eat","tea"]]
```

---

### 🔥 **Pattern 4: Longest Substring Without Repeating Characters**

```java
public int lengthOfLongestSubstring(String s) {
    HashMap<Character, Integer> map = new HashMap<>();
    int maxLength = 0;
    int start = 0;
    
    for (int end = 0; end < s.length(); end++) {
        char c = s.charAt(end);
        
        if (map.containsKey(c)) {
            start = Math.max(start, map.get(c) + 1);
        }
        
        map.put(c, end);
        maxLength = Math.max(maxLength, end - start + 1);
    }
    
    return maxLength;
}

// Example: "abcabcbb" → 3 ("abc")
```

---

## 🏢 Real-World Applications

### 🔷 **Database Indexing**

```java
public class Database {
    private HashMap<Integer, User> userIndex; // Primary key index
    private HashMap<String, User> emailIndex; // Email index
    
    public User findById(int id) {
        return userIndex.get(id); // O(1)
    }
    
    public User findByEmail(String email) {
        return emailIndex.get(email); // O(1)
    }
}
```

### 🔷 **Caching (LRU Cache uses HashMap + DoublyLinkedList)**

```java
class LRUCache {
    private HashMap<Integer, Node> map;
    private DoublyLinkedList dll;
    private int capacity;
    
    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        
        Node node = map.get(key);
        dll.moveToFront(node);
        return node.value;
    }
    
    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.value = value;
            dll.moveToFront(node);
        } else {
            if (map.size() >= capacity) {
                Node removed = dll.removeLast();
                map.remove(removed.key);
            }
            
            Node newNode = new Node(key, value);
            dll.addToFront(newNode);
            map.put(key, newNode);
        }
    }
}
```

### 🔷 **Phone Book**

```java
public class PhoneBook {
    private HashMap<String, String> contacts;
    
    public void addContact(String name, String number) {
        contacts.put(name, number);
    }
    
    public String findNumber(String name) {
        return contacts.get(name);
    }
}
```

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. Two Sum (LeetCode #1)
2. Contains Duplicate (LeetCode #217)
3. Valid Anagram (LeetCode #242)

### 🟡 **Medium**
4. Group Anagrams (LeetCode #49)
5. Top K Frequent Elements (LeetCode #347)
6. Longest Substring Without Repeating Characters (LeetCode #3)

### 🔴 **Hard**
7. LRU Cache (LeetCode #146)
8. First Missing Positive (LeetCode #41)

---

## 🎯 Key Takeaways

✅ HashMap provides **O(1) average** search, insert, delete  
✅ **Hash collisions** handled via chaining or open addressing  
✅ **Load factor** (0.75) determines when to resize  
✅ Perfect for **counting, lookup, caching**  

---

## 🚀 What's Next?

➡️ **[Trees: Hierarchical Power](../Trees/01_BinaryTree_Fundamentals.md)**

**🎮 Achievement Unlocked: Hash Table Mastery!** 🏆

