# 🌳 Binary Search Trees: Sorted Tree Power! 🚀

---

## 🎯 Chapter 2: BST Deep Dive

> **"A BST is a binary tree where left children are smaller and right children are larger!"**

---

## 🤔 What is a Binary Search Tree (BST)?

A **Binary Search Tree** is a binary tree with a special property:
- **Left subtree**: All values < node value
- **Right subtree**: All values > node value
- **Both subtrees**: Also BSTs (recursive property)

### 🏗️ **Visual Representation**

```
Valid BST:
         8
       /   \
      3     10
     / \      \
    1   6      14
       / \     /
      4   7   13

✅ Left of 8: {3,1,6,4,7} all < 8
✅ Right of 8: {10,14,13} all > 8
✅ Subtrees also follow BST property

Invalid BST:
         8
       /   \
      3     10
     / \      \
    1   12     14  ❌ 12 > 8, should be in right subtree
```

---

## 🎨 BST Property Verification

### ✅ **Valid BST Examples**

```
Example 1:      Example 2:      Example 3:
     5              10              20
   /   \           /              /  \
  3     7         5             10    30
 / \   / \         \           /  \
1   4 6   9         8        5    15
```

### ❌ **Invalid BST Examples**

```
Example 1:      Example 2:
     5              10
   /   \           /  \
  3     7         5    15
 / \   / \           /
1   6 4   9        3  ❌ (3 < 10, should be in left)
    ❌ (6 > 5)
```

---

## 💻 BST Implementation

### 🔧 **Complete BST Class**

```java
public class BinarySearchTree {
    private Node root;
    
    private class Node {
        int val;
        Node left;
        Node right;
        
        Node(int val) {
            this.val = val;
        }
    }
    
    // Insert - O(log n) average, O(n) worst
    public void insert(int val) {
        root = insertRec(root, val);
    }
    
    private Node insertRec(Node node, int val) {
        if (node == null) {
            return new Node(val);
        }
        
        if (val < node.val) {
            node.left = insertRec(node.left, val);
        } else if (val > node.val) {
            node.right = insertRec(node.right, val);
        }
        // If val == node.val, don't insert duplicates
        
        return node;
    }
    
    // Search - O(log n) average, O(n) worst
    public boolean search(int val) {
        return searchRec(root, val);
    }
    
    private boolean searchRec(Node node, int val) {
        if (node == null) {
            return false;
        }
        
        if (val == node.val) {
            return true;
        } else if (val < node.val) {
            return searchRec(node.left, val);
        } else {
            return searchRec(node.right, val);
        }
    }
    
    // Iterative search (more efficient)
    public boolean searchIterative(int val) {
        Node current = root;
        
        while (current != null) {
            if (val == current.val) {
                return true;
            } else if (val < current.val) {
                current = current.left;
            } else {
                current = current.right;
            }
        }
        
        return false;
    }
    
    // Delete - O(log n) average, O(n) worst
    public void delete(int val) {
        root = deleteRec(root, val);
    }
    
    private Node deleteRec(Node node, int val) {
        if (node == null) {
            return null;
        }
        
        if (val < node.val) {
            node.left = deleteRec(node.left, val);
        } else if (val > node.val) {
            node.right = deleteRec(node.right, val);
        } else {
            // Node to be deleted found
            
            // Case 1: Leaf node
            if (node.left == null && node.right == null) {
                return null;
            }
            
            // Case 2: One child
            if (node.left == null) {
                return node.right;
            }
            if (node.right == null) {
                return node.left;
            }
            
            // Case 3: Two children
            // Find inorder successor (smallest in right subtree)
            Node successor = findMin(node.right);
            node.val = successor.val;
            node.right = deleteRec(node.right, successor.val);
        }
        
        return node;
    }
    
    private Node findMin(Node node) {
        while (node.left != null) {
            node = node.left;
        }
        return node;
    }
    
    private Node findMax(Node node) {
        while (node.right != null) {
            node = node.right;
        }
        return node;
    }
    
    // Find minimum value - O(log n)
    public int getMin() {
        if (root == null) {
            throw new IllegalStateException("Tree is empty");
        }
        return findMin(root).val;
    }
    
    // Find maximum value - O(log n)
    public int getMax() {
        if (root == null) {
            throw new IllegalStateException("Tree is empty");
        }
        return findMax(root).val;
    }
    
    // Inorder traversal (gives sorted order) - O(n)
    public void inorder() {
        inorderRec(root);
        System.out.println();
    }
    
    private void inorderRec(Node node) {
        if (node != null) {
            inorderRec(node.left);
            System.out.print(node.val + " ");
            inorderRec(node.right);
        }
    }
    
    // Validate if tree is BST - O(n)
    public boolean isValidBST() {
        return isValidBSTRec(root, Long.MIN_VALUE, Long.MAX_VALUE);
    }
    
    private boolean isValidBSTRec(Node node, long min, long max) {
        if (node == null) {
            return true;
        }
        
        if (node.val <= min || node.val >= max) {
            return false;
        }
        
        return isValidBSTRec(node.left, min, node.val) &&
               isValidBSTRec(node.right, node.val, max);
    }
}
```

---

## 🎯 BST Operations Explained

### 🔥 **1. Insertion Visual**

```
Insert 15 into BST:

Step 1: Start at root (20)
         20
        /  \
      10    30

Step 2: 15 > 10, go right
         20
        /  \
      10    30
        \
        (15?)

Step 3: Insert as right child of 10
         20
        /  \
      10    30
        \
        15  ✅
```

---

### 🔥 **2. Deletion Cases**

```java
// Case 1: Delete leaf (no children)
Delete 15:
     20           20
    /  \         /  \
   10   30  →   10   30
     \
     15

// Case 2: Delete node with one child
Delete 10:
     20           20
    /  \         /  \
   10   30  →   15   30
     \
     15

// Case 3: Delete node with two children
Delete 20:
     20              25
    /  \            /  \
   10   30    →   10    30
       /  \             /  \
      25  40           27  40
        \
        27

Find inorder successor (25), replace, delete successor
```

---

### 🔥 **3. Kth Smallest Element**

```java
public int kthSmallest(Node root, int k) {
    int[] result = new int[2]; // [count, value]
    kthSmallestHelper(root, k, result);
    return result[1];
}

private void kthSmallestHelper(Node node, int k, int[] result) {
    if (node == null || result[0] >= k) {
        return;
    }
    
    // Inorder: left → root → right
    kthSmallestHelper(node.left, k, result);
    
    result[0]++; // Increment count
    if (result[0] == k) {
        result[1] = node.val;
        return;
    }
    
    kthSmallestHelper(node.right, k, result);
}

// Example: BST = [5, 3, 7, 1, 4, 6, 9]
// Inorder: 1, 3, 4, 5, 6, 7, 9
// kthSmallest(root, 3) → 4
```

---

### 🔥 **4. Lowest Common Ancestor (BST Specific)**

```java
public Node lowestCommonAncestor(Node root, Node p, Node q) {
    // Leverage BST property
    if (p.val < root.val && q.val < root.val) {
        // Both in left subtree
        return lowestCommonAncestor(root.left, p, q);
    } else if (p.val > root.val && q.val > root.val) {
        // Both in right subtree
        return lowestCommonAncestor(root.right, p, q);
    } else {
        // Split point (one left, one right) or one equals root
        return root;
    }
}

// Example:
//       6
//      / \
//     2   8
//    / \ / \
//   0  4 7  9
//     / \
//    3   5

// LCA(2, 8) = 6
// LCA(2, 4) = 2
```

---

## 🎯 Common BST Patterns

### 🔥 **Pattern 1: Validate BST**

```java
public boolean isValidBST(TreeNode root) {
    return validate(root, null, null);
}

private boolean validate(TreeNode node, Integer min, Integer max) {
    if (node == null) return true;
    
    if ((min != null && node.val <= min) || 
        (max != null && node.val >= max)) {
        return false;
    }
    
    return validate(node.left, min, node.val) &&
           validate(node.right, node.val, max);
}
```

---

### 🔥 **Pattern 2: Convert Sorted Array to BST**

```java
public TreeNode sortedArrayToBST(int[] nums) {
    return buildBST(nums, 0, nums.length - 1);
}

private TreeNode buildBST(int[] nums, int left, int right) {
    if (left > right) return null;
    
    int mid = left + (right - left) / 2;
    TreeNode root = new TreeNode(nums[mid]);
    
    root.left = buildBST(nums, left, mid - 1);
    root.right = buildBST(nums, mid + 1, right);
    
    return root;
}

// Example: [1, 2, 3, 4, 5, 6, 7]
//       4
//      / \
//     2   6
//    / \ / \
//   1  3 5  7
```

---

### 🔥 **Pattern 3: BST Iterator**

```java
class BSTIterator {
    private Stack<TreeNode> stack;
    
    public BSTIterator(TreeNode root) {
        stack = new Stack<>();
        pushLeft(root);
    }
    
    private void pushLeft(TreeNode node) {
        while (node != null) {
            stack.push(node);
            node = node.left;
        }
    }
    
    public int next() {
        TreeNode node = stack.pop();
        pushLeft(node.right);
        return node.val;
    }
    
    public boolean hasNext() {
        return !stack.isEmpty();
    }
}

// Usage: iterates BST in sorted order
// BSTIterator iterator = new BSTIterator(root);
// while (iterator.hasNext()) {
//     System.out.println(iterator.next()); // Sorted order
// }
```

---

### 🔥 **Pattern 4: Range Sum Query**

```java
public int rangeSumBST(TreeNode root, int low, int high) {
    if (root == null) return 0;
    
    int sum = 0;
    
    // Current node in range
    if (root.val >= low && root.val <= high) {
        sum += root.val;
    }
    
    // Explore left if possible
    if (root.val > low) {
        sum += rangeSumBST(root.left, low, high);
    }
    
    // Explore right if possible
    if (root.val < high) {
        sum += rangeSumBST(root.right, low, high);
    }
    
    return sum;
}

// BST: [10, 5, 15, 3, 7, null, 18]
// rangeSumBST(root, 7, 15) → 7 + 10 + 15 = 32
```

---

## 🏢 Real-World Applications

### 🔷 **Database Indexing**

```java
public class DatabaseIndex {
    private BST<Integer, Record> index;
    
    public void insertRecord(int id, Record record) {
        index.insert(id, record);
    }
    
    public Record findRecord(int id) {
        return index.search(id); // O(log n)
    }
    
    public List<Record> findRecordsInRange(int low, int high) {
        return index.rangeQuery(low, high);
    }
}
```

### 🔷 **File System (Balanced BST)**

```java
public class FileSystem {
    private TreeMap<String, File> files; // TreeMap uses Red-Black Tree (BST)
    
    public void addFile(String name, File file) {
        files.put(name, file); // O(log n)
    }
    
    public File getFile(String name) {
        return files.get(name); // O(log n)
    }
    
    public List<String> getFilesInRange(String start, String end) {
        return new ArrayList<>(files.subMap(start, end).keySet());
    }
}
```

### 🔷 **Auto-Complete System**

```java
public class AutoComplete {
    private TreeSet<String> dictionary; // Sorted set using BST
    
    public List<String> getSuggestions(String prefix) {
        // Get all words >= prefix
        SortedSet<String> suggestions = dictionary.tailSet(prefix);
        
        List<String> result = new ArrayList<>();
        for (String word : suggestions) {
            if (word.startsWith(prefix)) {
                result.add(word);
            } else {
                break; // No more matches
            }
        }
        
        return result;
    }
}
```

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Search in BST](https://leetcode.com/problems/search-in-a-binary-search-tree/) (LeetCode #700)
2. [Minimum Absolute Difference in BST](https://leetcode.com/problems/minimum-absolute-difference-in-bst/) (LeetCode #530)
3. [Range Sum of BST](https://leetcode.com/problems/range-sum-of-bst/) (LeetCode #938)
4. [Convert Sorted Array to BST](https://leetcode.com/problems/convert-sorted-array-to-binary-search-tree/) (LeetCode #108)

### 🟡 **Medium**
5. [Validate BST](https://leetcode.com/problems/validate-binary-search-tree/) (LeetCode #98)
6. [Kth Smallest Element in BST](https://leetcode.com/problems/kth-smallest-element-in-a-bst/) (LeetCode #230)
7. [Delete Node in BST](https://leetcode.com/problems/delete-node-in-a-bst/) (LeetCode #450)
8. [BST Iterator](https://leetcode.com/problems/binary-search-tree-iterator/) (LeetCode #173)
9. [Inorder Successor in BST](https://leetcode.com/problems/inorder-successor-in-bst/) (LeetCode #285)
10. [Insert into BST](https://leetcode.com/problems/insert-into-a-binary-search-tree/) (LeetCode #701)

### 🔴 **Hard**
11. [Recover BST](https://leetcode.com/problems/recover-binary-search-tree/) (LeetCode #99)
12. [Count of Smaller Numbers After Self](https://leetcode.com/problems/count-of-smaller-numbers-after-self/) (LeetCode #315)

---

## ⚡ BST vs Hash Table

| Feature | BST | Hash Table |
|---------|-----|------------|
| **Search** | O(log n) | O(1) average |
| **Sorted Order** | ✅ Yes | ❌ No |
| **Range Query** | O(log n + k) | O(n) |
| **Space** | O(n) | O(n) |
| **Worst Case** | O(n) unbalanced | O(n) collisions |

**Use BST when you need**:
- Sorted data
- Range queries
- Predecessor/successor operations
- Ordered iteration

**Use Hash Table when you need**:
- Fastest lookup
- No ordering required
- Exact match queries

---

## 🎯 Key Takeaways

✅ BST maintains **sorted order**  
✅ Operations: **O(log n)** average (balanced), **O(n)** worst (skewed)  
✅ **Inorder traversal** gives sorted sequence  
✅ Perfect for **range queries, ordered data**  
✅ Balanced variants (AVL, Red-Black) guarantee O(log n)  

---

## 🚀 What's Next?

➡️ **[Balanced Trees: AVL & Red-Black](./03_Balanced_Trees.md)**  
➡️ **[Trie: Prefix Trees](./04_Trie_Prefix_Trees.md)**

---

**🎮 Achievement Unlocked: BST Mastery!** 🏆

*Remember: "In BST, everything has its place—left is less, right is more!"* 🌳

