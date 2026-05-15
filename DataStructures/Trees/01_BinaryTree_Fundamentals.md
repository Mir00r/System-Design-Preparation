# 🌳 Binary Trees: The Hierarchical Powerhouse! 🚀

---

## 🎯 Chapter 1: Binary Tree Fundamentals

> **"Trees grow downward in computer science!"**

---

## 🤔 What is a Binary Tree?

A **Binary Tree** is a hierarchical data structure where each node has **at most two children** (left and right).

### 🏗️ **Visual Representation**

```
         1        ← Root
       /   \
      2     3     ← Level 1
     / \   / \
    4   5 6   7   ← Level 2 (Leaves)

Height = 2 (levels from root to deepest leaf)
Depth of node 4 = 2
```

---

## 📚 Tree Terminology

```
         10 (Root)
       /    \
      5      15 (Right child of 10)
     / \    / \
    3   7  12  20
   /
  1

✅ Root: Top node (10)
✅ Parent: Node with children (5 is parent of 3 and 7)
✅ Child: Node below parent (3, 7 are children of 5)
✅ Leaf: Node with no children (1, 7, 12, 20)
✅ Internal Node: Node with at least one child (10, 5, 15)
✅ Height: Longest path from root to leaf (3)
✅ Depth: Distance from root (depth of 1 = 2)
✅ Level: All nodes at same depth
```

---

## 💻 Java Implementation

### 🔧 **Node Structure**

```java
class TreeNode {
    int val;
    TreeNode left;
    TreeNode right;
    
    public TreeNode(int val) {
        this.val = val;
        this.left = null;
        this.right = null;
    }
}
```

### 🌳 **Binary Tree Class**

```java
public class BinaryTree {
    TreeNode root;
    
    // Inorder Traversal: Left → Root → Right
    public void inorder(TreeNode node) {
        if (node == null) return;
        
        inorder(node.left);
        System.out.print(node.val + " ");
        inorder(node.right);
    }
    
    // Preorder Traversal: Root → Left → Right
    public void preorder(TreeNode node) {
        if (node == null) return;
        
        System.out.print(node.val + " ");
        preorder(node.left);
        preorder(node.right);
    }
    
    // Postorder Traversal: Left → Right → Root
    public void postorder(TreeNode node) {
        if (node == null) return;
        
        postorder(node.left);
        postorder(node.right);
        System.out.print(node.val + " ");
    }
    
    // Level Order Traversal (BFS)
    public void levelOrder() {
        if (root == null) return;
        
        Queue<TreeNode> queue = new LinkedList<>();
        queue.offer(root);
        
        while (!queue.isEmpty()) {
            TreeNode current = queue.poll();
            System.out.print(current.val + " ");
            
            if (current.left != null) queue.offer(current.left);
            if (current.right != null) queue.offer(current.right);
        }
    }
    
    // Height of tree
    public int height(TreeNode node) {
        if (node == null) return -1;
        
        int leftHeight = height(node.left);
        int rightHeight = height(node.right);
        
        return Math.max(leftHeight, rightHeight) + 1;
    }
    
    // Count nodes
    public int countNodes(TreeNode node) {
        if (node == null) return 0;
        return 1 + countNodes(node.left) + countNodes(node.right);
    }
}
```

---

## 🎯 Tree Traversals Explained

### 🔥 **1. Inorder (Left → Root → Right)**

```
Tree:      1
          / \
         2   3
        / \
       4   5

Inorder: 4 → 2 → 5 → 1 → 3
(For BST: gives sorted order!)
```

### 🔥 **2. Preorder (Root → Left → Right)**

```
Preorder: 1 → 2 → 4 → 5 → 3
(Used to copy tree structure)
```

### 🔥 **3. Postorder (Left → Right → Root)**

```
Postorder: 4 → 5 → 2 → 3 → 1
(Used to delete tree)
```

### 🔥 **4. Level Order (BFS)**

```
Level Order: 1 → 2 → 3 → 4 → 5
(Level by level, left to right)
```

---

## 🎯 Common Tree Patterns

### 🔥 **Pattern 1: Maximum Depth**

```java
public int maxDepth(TreeNode root) {
    if (root == null) return 0;
    
    int leftDepth = maxDepth(root.left);
    int rightDepth = maxDepth(root.right);
    
    return Math.max(leftDepth, rightDepth) + 1;
}
```

### 🔥 **Pattern 2: Check if Balanced**

```java
public boolean isBalanced(TreeNode root) {
    return checkHeight(root) != -1;
}

private int checkHeight(TreeNode node) {
    if (node == null) return 0;
    
    int leftHeight = checkHeight(node.left);
    if (leftHeight == -1) return -1;
    
    int rightHeight = checkHeight(node.right);
    if (rightHeight == -1) return -1;
    
    if (Math.abs(leftHeight - rightHeight) > 1) return -1;
    
    return Math.max(leftHeight, rightHeight) + 1;
}
```

### 🔥 **Pattern 3: Path Sum**

```java
public boolean hasPathSum(TreeNode root, int targetSum) {
    if (root == null) return false;
    
    if (root.left == null && root.right == null) {
        return targetSum == root.val;
    }
    
    return hasPathSum(root.left, targetSum - root.val) ||
           hasPathSum(root.right, targetSum - root.val);
}
```

### 🔥 **Pattern 4: Lowest Common Ancestor**

```java
public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null || root == p || root == q) {
        return root;
    }
    
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    
    if (left != null && right != null) return root;
    
    return left != null ? left : right;
}
```

---

## 🏢 Real-World Applications

### 🔷 **File System**

```
C:/
├── Program Files/
│   ├── Java/
│   └── Python/
├── Users/
│   ├── Alice/
│   └── Bob/
└── Windows/
```

### 🔷 **Organization Hierarchy**

```
           CEO
         /     \
      CTO       CFO
     /   \
   Dev   QA
```

### 🔷 **HTML DOM**

```html
<html>
  <head>
    <title>Page</title>
  </head>
  <body>
    <div>Content</div>
  </body>
</html>
```

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. Maximum Depth of Binary Tree (LeetCode #104)
2. Invert Binary Tree (LeetCode #226)
3. Same Tree (LeetCode #100)

### 🟡 **Medium**
4. Validate Binary Search Tree (LeetCode #98)
5. Binary Tree Level Order Traversal (LeetCode #102)
6. Construct Binary Tree from Preorder and Inorder (LeetCode #105)

### 🔴 **Hard**
7. Binary Tree Maximum Path Sum (LeetCode #124)
8. Serialize and Deserialize Binary Tree (LeetCode #297)

---

## 🎯 Key Takeaways

✅ Binary trees have **at most 2 children** per node  
✅ **4 traversals**: Inorder, Preorder, Postorder, Level Order  
✅ **Recursion** is natural for tree problems  
✅ Used in **file systems, DOM, compilers**  

---

## 🚀 What's Next?

➡️ **[Binary Search Trees: Sorted Power](./02_BST_Deep_Dive.md)**  
➡️ **[Graphs: Network Power](../Graphs/01_Graph_Fundamentals.md)**

**🎮 Achievement Unlocked: Binary Tree Mastery!** 🏆

