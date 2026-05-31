# 🔤 String Algorithms: Master Text Processing! 📝

> **"Strings are everywhere - from DNA sequences to web search. Master strings, master data!"**

Master **string algorithms** - the essential techniques powering Google Search, DNA matching, text editors, and every text-processing system! Learn pattern matching, tries, and more! 🚀

---

## 📋 Table of Contents

1. [String Fundamentals](#-string-fundamentals)
2. [KMP Algorithm](#-kmp-algorithm)
3. [Rabin-Karp Algorithm](#-rabin-karp-algorithm)
4. [Trie Data Structure](#-trie-data-structure)
5. [Z-Algorithm](#-z-algorithm)
6. [Practice Problems](#-practice-problems)

---

## 🎯 String Fundamentals

### **Key Operations**

```java
String s = "hello";

// Length
s.length();  // 5

// Character access
s.charAt(0);  // 'h'

// Substring
s.substring(1, 4);  // "ell"

// Comparison
s.equals("hello");  // true
s.compareTo("world");  // negative

// Concatenation
s + " world";  // "hello world"

// StringBuilder (mutable)
StringBuilder sb = new StringBuilder();
sb.append("hello").append(" world");
```

---

## 🔍 KMP Algorithm

### **Problem**: Find pattern in text efficiently

**Naive Approach**: O(nm)  
**KMP**: O(n + m) ✅

### **Key Idea**: Use failure function to avoid re-checking characters

```java
public class KMP {
    // Build failure function (LPS array)
    private int[] computeLPS(String pattern) {
        int m = pattern.length();
        int[] lps = new int[m];
        int len = 0;  // Length of previous longest prefix suffix
        
        for (int i = 1; i < m; i++) {
            while (len > 0 && pattern.charAt(i) != pattern.charAt(len)) {
                len = lps[len - 1];
            }
            
            if (pattern.charAt(i) == pattern.charAt(len)) {
                len++;
            }
            lps[i] = len;
        }
        
        return lps;
    }
    
    // KMP search
    public List<Integer> search(String text, String pattern) {
        List<Integer> result = new ArrayList<>();
        int n = text.length();
        int m = pattern.length();
        
        int[] lps = computeLPS(pattern);
        
        int i = 0;  // Index for text
        int j = 0;  // Index for pattern
        
        while (i < n) {
            if (text.charAt(i) == pattern.charAt(j)) {
                i++;
                j++;
            }
            
            if (j == m) {
                result.add(i - j);  // Match found
                j = lps[j - 1];
            } else if (i < n && text.charAt(i) != pattern.charAt(j)) {
                if (j != 0) {
                    j = lps[j - 1];  // Use failure function
                } else {
                    i++;
                }
            }
        }
        
        return result;
    }
}
```

**Example**:
```
Text:    "ABABDABACDABABCABAB"
Pattern: "ABABCABAB"

LPS: [0, 0, 1, 2, 0, 1, 2, 3, 4]

Process:
- Match "ABAB", mismatch at D
- Jump to lps[3] = 2 (skip "AB")
- Continue matching...

Found at index: 10 ✅
```

**Complexity**: O(n + m) time, O(m) space

---

## 🎲 Rabin-Karp Algorithm

### **Key Idea**: Use rolling hash for fast comparison

```java
public class RabinKarp {
    private static final int PRIME = 101;
    
    public List<Integer> search(String text, String pattern) {
        List<Integer> result = new ArrayList<>();
        int n = text.length();
        int m = pattern.length();
        
        if (m > n) return result;
        
        // Calculate hash of pattern
        long patternHash = hash(pattern, m);
        long textHash = hash(text.substring(0, m), m);
        
        // Calculate h = pow(256, m-1) % PRIME for rolling hash
        long h = 1;
        for (int i = 0; i < m - 1; i++) {
            h = (h * 256) % PRIME;
        }
        
        // Slide pattern over text
        for (int i = 0; i <= n - m; i++) {
            // Check hash first (fast)
            if (patternHash == textHash) {
                // Verify character by character (avoid false positives)
                if (text.substring(i, i + m).equals(pattern)) {
                    result.add(i);
                }
            }
            
            // Calculate hash for next window (rolling hash)
            if (i < n - m) {
                textHash = (256 * (textHash - text.charAt(i) * h) + 
                           text.charAt(i + m)) % PRIME;
                
                if (textHash < 0) {
                    textHash += PRIME;
                }
            }
        }
        
        return result;
    }
    
    private long hash(String s, int m) {
        long hash = 0;
        for (int i = 0; i < m; i++) {
            hash = (hash * 256 + s.charAt(i)) % PRIME;
        }
        return hash;
    }
}
```

**Rolling Hash Visual**:
```
Text: "ABCDE"
Pattern: "BC" (length = 2)

Window 1: "AB" → hash(A)*256 + hash(B)
Window 2: "BC" → (hash("AB") - hash(A)*256)*256 + hash(C)
         ↑ Remove A    ↑ Shift     ↑ Add C

This avoids recalculating entire hash! ✅
```

**Complexity**: O(n + m) average, O(nm) worst case

---

## 🌲 Trie Data Structure

### **Use Cases**:
✅ **Autocomplete** (Google search)  
✅ **Spell checker**  
✅ **IP routing**  
✅ **Word games** (Boggle, Scrabble)  

```java
class TrieNode {
    TrieNode[] children = new TrieNode[26];
    boolean isEndOfWord = false;
}

public class Trie {
    private TrieNode root;
    
    public Trie() {
        root = new TrieNode();
    }
    
    // Insert word
    public void insert(String word) {
        TrieNode node = root;
        
        for (char c : word.toCharArray()) {
            int index = c - 'a';
            if (node.children[index] == null) {
                node.children[index] = new TrieNode();
            }
            node = node.children[index];
        }
        
        node.isEndOfWord = true;
    }
    
    // Search word
    public boolean search(String word) {
        TrieNode node = searchNode(word);
        return node != null && node.isEndOfWord;
    }
    
    // Check prefix
    public boolean startsWith(String prefix) {
        return searchNode(prefix) != null;
    }
    
    private TrieNode searchNode(String word) {
        TrieNode node = root;
        
        for (char c : word.toCharArray()) {
            int index = c - 'a';
            if (node.children[index] == null) {
                return null;
            }
            node = node.children[index];
        }
        
        return node;
    }
    
    // Autocomplete suggestions
    public List<String> autocomplete(String prefix) {
        List<String> results = new ArrayList<>();
        TrieNode node = searchNode(prefix);
        
        if (node != null) {
            dfs(node, prefix, results);
        }
        
        return results;
    }
    
    private void dfs(TrieNode node, String current, List<String> results) {
        if (node.isEndOfWord) {
            results.add(current);
        }
        
        for (int i = 0; i < 26; i++) {
            if (node.children[i] != null) {
                dfs(node.children[i], current + (char)('a' + i), results);
            }
        }
    }
}
```

**Visual**:
```
Trie with words: "cat", "car", "dog"

        root
       /    \
      c      d
      |      |
      a      o
     / \     |
    t   r    g
   ✓   ✓    ✓

Search "car": root → c → a → r ✓
Search "can": root → c → a → (n not found) ✗
Prefix "ca": root → c → a → found! (suggests "cat", "car")
```

**Complexity**: Insert/Search = O(m) where m = word length

---

## ⚡ Z-Algorithm

### **Problem**: Find all occurrences of pattern in text

**Key Idea**: Z-array stores length of longest substring starting from i that matches prefix

```java
public int[] zAlgorithm(String s) {
    int n = s.length();
    int[] z = new int[n];
    int l = 0, r = 0;
    
    for (int i = 1; i < n; i++) {
        if (i > r) {
            l = r = i;
            while (r < n && s.charAt(r - l) == s.charAt(r)) {
                r++;
            }
            z[i] = r - l;
            r--;
        } else {
            int k = i - l;
            if (z[k] < r - i + 1) {
                z[i] = z[k];
            } else {
                l = i;
                while (r < n && s.charAt(r - l) == s.charAt(r)) {
                    r++;
                }
                z[i] = r - l;
                r--;
            }
        }
    }
    
    return z;
}

public List<Integer> searchPattern(String text, String pattern) {
    String combined = pattern + "$" + text;
    int[] z = zAlgorithm(combined);
    
    List<Integer> result = new ArrayList<>();
    int m = pattern.length();
    
    for (int i = 0; i < z.length; i++) {
        if (z[i] == m) {
            result.add(i - m - 1);  // Found at this index in text
        }
    }
    
    return result;
}
```

**Example**:
```
String: "aabaab"
Z-array: [0, 1, 0, 3, 1, 0]

Explanation:
Index 0: "aabaab" (entire string)
Index 1: "a" matches prefix (length 1)
Index 2: "b" doesn't match (0)
Index 3: "aab" matches prefix (length 3)
Index 4: "a" matches prefix (length 1)
Index 5: "b" doesn't match (0)
```

**Complexity**: O(n) time

---

## 🏢 Real-World Applications

### **Google Search** 🔍
- Pattern matching for search queries
- Autocomplete with tries

### **DNA Sequencing** 🧬
- KMP/Z-algorithm for finding gene patterns
- Longest common substring

### **Text Editors** 📝
- Find & Replace (Rabin-Karp)
- Syntax highlighting (pattern matching)

### **Plagiarism Detection** 📄
- String similarity algorithms
- Longest common subsequence

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Implement strStr()](https://leetcode.com/problems/implement-strstr/)
2. [Longest Common Prefix](https://leetcode.com/problems/longest-common-prefix/)
3. [Valid Palindrome](https://leetcode.com/problems/valid-palindrome/)

### 🟡 **Medium**
4. [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)
5. [Longest Repeating Character Replacement](https://leetcode.com/problems/longest-repeating-character-replacement/)
6. [Group Anagrams](https://leetcode.com/problems/group-anagrams/)
7. [Find All Anagrams](https://leetcode.com/problems/find-all-anagrams-in-a-string/)
8. [Implement Trie](https://leetcode.com/problems/implement-trie-prefix-tree/)

### 🔴 **Hard**
9. [Shortest Palindrome](https://leetcode.com/problems/shortest-palindrome/)
10. [Word Search II](https://leetcode.com/problems/word-search-ii/)

---

## 💡 Pro Tips

✅ **KMP** for exact pattern matching  
✅ **Rabin-Karp** for multiple patterns  
✅ **Trie** for prefix-based searches  
✅ **Z-algorithm** for linear-time pattern matching  
✅ **StringBuilder** for string concatenation in loops  

---

## 🎯 Key Takeaways

1. **KMP** = O(n+m) pattern matching with failure function
2. **Rabin-Karp** = Rolling hash for fast comparison
3. **Trie** = Tree for prefix searches (autocomplete)
4. **Z-algorithm** = Linear pattern matching
5. **Choose algorithm** based on use case

---

## 🚀 What's Next?

➡️ **[Bit Manipulation](../BitManipulation/01_Bit_Fundamentals.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**  

---

**🎮 Achievement Unlocked: String Master!** 🏆

*Remember: "Strings connect characters, algorithms connect ideas!"* 🔤✨

---

*Previous: [← Greedy Fundamentals](../Greedy/01_Greedy_Fundamentals.md) | Next: [Bit Manipulation →](../BitManipulation/01_Bit_Fundamentals.md)*

