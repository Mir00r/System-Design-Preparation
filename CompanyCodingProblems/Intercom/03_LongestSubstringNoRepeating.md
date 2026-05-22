# 🟣 Intercom | Problem 03 — Longest Substring Without Repeating Characters

> **Source:** Intercom Coding Round · LeetCode 3
> **Difficulty:** 🟡 Medium
> **Topics:** Sliding Window · HashMap · Two Pointers · String

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Given a string, find the length of the longest contiguous substring that contains **no duplicate characters**.

**Clarifying Questions to Ask:**
- "Is the input ASCII only, or can it include Unicode characters?"
- "Is an empty string a valid input? Should we return 0?"
- "Is the comparison case-sensitive? Is `'A'` different from `'a'`?"
- "Should we return the length only, or also the actual substring?"
- "Can the string contain spaces or special characters?"

**Confirmed Assumptions:**
- Input is a string of ASCII characters (printable + space)
- Case-sensitive: `'A' ≠ 'a'`
- Return the **length** of the longest valid substring
- Empty string → return 0
- String of length 1 → return 1 (trivially no repeats)

**Walking Through the Example:**
```
Input: "abcabcbb"

We want the longest window [left..right] with all unique chars.

  a b c a b c b b
  ^               left=0, right=0 → "a",      len=1
  ^   ^           left=0, right=2 → "abc",    len=3
      ^ ^         'a' seen at 0 → move left to 1 → "bca",   len=3
        ^ ^       'b' seen at 1 → move left to 2 → "cab",   len=3
          ^ ^     'c' seen at 2 → move left to 3 → "abc",   len=3
            ^ ^   'b' seen at 4 → move left to 5 → "cb",    len=2
              ^ ^ 'b' seen at 6 → move left to 7 → "b",     len=1

Answer: 3 ("abc")
```

**More Examples:**
```
"bbbbb"  → 1  ("b")
"pwwkew" → 3  ("wke")
""       → 0
"au"     → 2  ("au")
" "      → 1  (space is a valid character)
"dvdf"   → 3  ("vdf")
```

---

### **2️⃣ Breaking Down the Solution**

**Brute Force — O(n³):**
> Check every possible substring (O(n²) pairs), for each check if it has all unique chars (O(n)). Total: O(n³).

```
for i in range(n):
    for j in range(i+1, n+1):
        if all_unique(s[i:j]):  # O(n) with a set
            max_len = max(max_len, j - i)
```

**Better — O(n²) with Sliding Window + Set:**
> Expand right; if duplicate found, shrink left one-by-one until the window is valid again.

**Optimal — O(n) with Sliding Window + HashMap:**
> Instead of shrinking one step at a time, **jump left directly** to `last_seen_index + 1` when a duplicate is found.
> This avoids re-scanning the left portion.

```
HashMap: char → index of its LAST occurrence

When s[right] is already in the map AND its last index >= left:
    left = map[s[right]] + 1    ← jump left past the duplicate

Always: map[s[right]] = right
```

| Approach | Time | Space | Key Idea |
|----------|------|-------|----------|
| Brute Force (all substrings) | O(n³) | O(n) | Check every pair |
| Sliding window + Set | O(n²) worst case | O(k) | Shrink one step at a time |
| **Sliding window + HashMap** | **O(n)** | **O(k)** | Jump left directly |

> k = size of the character set (26 for lowercase, 128 for ASCII, 256 for extended ASCII)

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Python Solution

```python
def lengthOfLongestSubstring(s: str) -> int:
    """
    Find the length of the longest substring without repeating characters.

    Approach: Sliding window with HashMap
      - right pointer expands the window
      - left pointer jumps past any duplicate found at right
      - char_last_index tracks the most recent index of each character

    Time:  O(n)    — each character visited at most twice (once by right, once by left jump)
    Space: O(k)    — k = alphabet size (at most 128 for ASCII)
    """
    char_last_index: dict[str, int] = {}  # char → last seen index
    max_len = 0
    left = 0

    for right, char in enumerate(s):
        # If char was seen before AND is inside the current window [left..right]
        if char in char_last_index and char_last_index[char] >= left:
            # Jump left past the previous occurrence
            left = char_last_index[char] + 1

        # Update (or insert) last seen index
        char_last_index[char] = right

        # Update max window size
        max_len = max(max_len, right - left + 1)

    return max_len
```

---

#### ✅ Java Solution

```java
import java.util.HashMap;
import java.util.Map;

public class Solution {

    /**
     * Find the length of the longest substring without repeating characters.
     *
     * Approach: Sliding window with HashMap
     *   - right pointer expands the window character by character
     *   - left pointer jumps to charLastIndex[char] + 1 on duplicate
     *   - charLastIndex stores the most recent position of each character
     *
     * Time:  O(n)   — each char processed once
     * Space: O(k)   — k = alphabet size (128 for ASCII)
     */
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> charLastIndex = new HashMap<>();
        int maxLen = 0;
        int left = 0;

        for (int right = 0; right < s.length(); right++) {
            char c = s.charAt(right);

            // If char is inside current window, jump left past its last occurrence
            if (charLastIndex.containsKey(c) && charLastIndex.get(c) >= left) {
                left = charLastIndex.get(c) + 1;
            }

            charLastIndex.put(c, right);
            maxLen = Math.max(maxLen, right - left + 1);
        }

        return maxLen;
    }
}
```

---

#### ✅ Variant — Return the Actual Substring

```python
def longestSubstringWithoutRepeating(s: str) -> str:
    """Returns the actual substring, not just the length."""
    char_last_index: dict[str, int] = {}
    max_start, max_len = 0, 0
    left = 0

    for right, char in enumerate(s):
        if char in char_last_index and char_last_index[char] >= left:
            left = char_last_index[char] + 1
        char_last_index[char] = right

        if right - left + 1 > max_len:
            max_len = right - left + 1
            max_start = left

    return s[max_start : max_start + max_len]
```

---

### **4️⃣ Explaining the Code to the Interviewer**

**Step-by-step walkthrough with `"dvdf"`:**

```
char_last_index = {}, left = 0, max_len = 0

right=0, char='d':  not in map      → map{'d':0}, window=[0,0]="d",  len=1
right=1, char='v':  not in map      → map{'d':0,'v':1}, window="dv", len=2
right=2, char='d':  in map, idx=0 >= left=0
                    → left = 0+1 = 1
                    → map{'d':2,'v':1}, window=[1,2]="vd", len=2
right=3, char='f':  not in map      → map{...,'f':3}, window=[1,3]="vdf", len=3

max_len = 3  ✓
```

**Why `char_last_index[char] >= left` matters:**
```
Input: "abba"

right=0, char='a' → map{'a':0}, left=0, len=1
right=1, char='b' → map{'a':0,'b':1}, left=0, len=2
right=2, char='b' → in map, idx=1 >= left=0 → left=2, map{'a':0,'b':2}, len=1
right=3, char='a' → in map, idx=0 < left=2 → DON'T move left (already past 'a')
                    → map{'a':3,'b':2}, window=[2,3]="ba", len=2

max_len = 2  ✓

Without the >= left check, left would wrongly jump back to idx=1, shrinking the window.
```

**Complexity Analysis:**

| | Value | Explanation |
|--|-------|------------|
| Time | O(n) | Right pointer moves n steps; left pointer only moves forward |
| Space | O(k) | At most k=128 entries in HashMap (ASCII alphabet) |
| Space (lowercase only) | O(26) | Effectively O(1) constant space |

---

### **5️⃣ Discussing Edge Cases & Testing**

#### Test Table

| Input | Expected | Edge Case Type |
|-------|----------|----------------|
| `"abcabcbb"` | `3` | Normal — repeats after window |
| `"bbbbb"` | `1` | All same character |
| `"pwwkew"` | `3` | Repeat in middle |
| `""` | `0` | Empty string |
| `"a"` | `1` | Single character |
| `"au"` | `2` | Two unique characters |
| `"abba"` | `2` | Left boundary jump test |
| `"dvdf"` | `3` | Tricky: stale map entry for 'd' |
| `" "` | `1` | Space is a valid character |
| `"aab"` | `2` | Repeat at start |
| All 26 distinct letters | `26` | Maximum for lowercase |

#### Off-By-One Checks
- Window length = `right - left + 1` (inclusive on both ends)
- Jump: `left = char_last_index[char] + 1` (move PAST the duplicate, not to it)
- Guard: `char_last_index[char] >= left` (ignore stale entries before the window)

---

### **6️⃣ Debugging & Fixing Mistakes**

**Common Mistakes:**

❌ **Mistake 1: Not checking if the stored index is inside the current window**
```python
# WRONG — causes left to jump backward on stale entries
if char in char_last_index:
    left = char_last_index[char] + 1

# RIGHT
if char in char_last_index and char_last_index[char] >= left:
    left = char_last_index[char] + 1
```

❌ **Mistake 2: Off-by-one in window length**
```python
# WRONG
max_len = max(max_len, right - left)

# RIGHT — window is [left..right] inclusive
max_len = max(max_len, right - left + 1)
```

❌ **Mistake 3: Using a Set instead of a Map (causes O(n) shrink)**
```python
# Works but O(n) left shrink in worst case → O(n²) total
seen = set()
while s[right] in seen:
    seen.remove(s[left])
    left += 1
seen.add(s[right])
```

**Debug Print:**
```python
for right, char in enumerate(s):
    print(f"right={right} char={char!r} left={left} window={s[left:right+1]!r}")
```

---

## 🚀 Best Practices Demonstrated

✅ **Start with brute force** — Stated O(n³) → O(n²) → O(n) progression before coding  
✅ **Guard condition** — `char_last_index[char] >= left` prevents left from moving backward  
✅ **Single pass** — Right pointer never revisits characters  
✅ **Edge cases verified** — Empty string, single char, all-same, tricky "stale entry" case  
✅ **Variant offered** — Can return the actual substring with one extra tracked variable  

---

## 📎 Related Problems

| # | Problem | Connection |
|---|---------|-----------|
| [LeetCode 159](https://leetcode.com/problems/longest-substring-with-at-most-two-distinct-characters/) | At Most Two Distinct | Same sliding window, different constraint |
| [LeetCode 340](https://leetcode.com/problems/longest-substring-with-at-most-k-distinct-characters/) | At Most K Distinct | Generalization of this problem |
| [LeetCode 76](https://leetcode.com/problems/minimum-window-substring/) | Minimum Window Substring | Sliding window with frequency map |
| [LeetCode 567](https://leetcode.com/problems/permutation-in-string/) | Permutation in String | Fixed-size sliding window |
