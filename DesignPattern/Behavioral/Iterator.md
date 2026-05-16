# 🔄 Iterator Pattern: Traverse Without Exposing Internals! 🎯

> **"You don't need to know if it's an array, linked list, or tree. You just say: next!"**

---

## 🎬 The Story

```
🎵 Music Playlist — You just press "Next Track"
Whether songs are stored in:
• An array (Spotify local cache)
• A linked list (shuffle queue)  
• A database query (streaming)
• A remote API (cloud library)

You don't care about the STORAGE. You care about: NEXT! ⏭️
```

---

## 💻 Java Implementation

```java
// 🔄 Iterator interface (Java already has java.util.Iterator!)
public interface Iterator<T> {
    boolean hasNext();
    T next();
}

// 📦 Collection interface
public interface IterableCollection<T> {
    Iterator<T> createIterator();
}

// 🌳 Custom Collection: Binary Tree with Iterator
public class BinaryTree<T> implements IterableCollection<T> {
    private Node<T> root;
    
    // ... tree operations ...
    
    @Override
    public Iterator<T> createIterator() {
        return new InOrderIterator<>(root); // Clients don't know it's a tree!
    }
}

// 🚶 Concrete Iterator: In-order traversal
public class InOrderIterator<T> implements Iterator<T> {
    private final Deque<Node<T>> stack = new ArrayDeque<>();
    
    public InOrderIterator(Node<T> root) {
        pushLeft(root);
    }
    
    private void pushLeft(Node<T> node) {
        while (node != null) {
            stack.push(node);
            node = node.left;
        }
    }
    
    @Override
    public boolean hasNext() { return !stack.isEmpty(); }
    
    @Override
    public T next() {
        Node<T> node = stack.pop();
        pushLeft(node.right);
        return node.value;
    }
}

// 🎮 Usage — Client doesn't know the internal structure!
BinaryTree<Integer> tree = new BinaryTree<>();
// ... add elements ...

Iterator<Integer> it = tree.createIterator();
while (it.hasNext()) {
    System.out.println(it.next()); // Just: next, next, next!
}

// Or with Java's Iterable + for-each:
for (Integer value : tree) { // If BinaryTree implements Iterable<T>
    System.out.println(value);
}
```

---

## 🌍 Real-World Examples

```java
// ☕ Java Collections — ALL use Iterator!
List<String> list = List.of("A", "B", "C");
Iterator<String> it = list.iterator();

// 🌊 Java Streams — lazy iterators!
Stream.of(1, 2, 3, 4, 5).filter(n -> n > 2).forEach(System.out::println);

// 🗄️ JDBC ResultSet — iterates over database rows!
while (resultSet.next()) { /* process row */ }

// 📄 BufferedReader.lines() — iterates over file lines lazily
// 🌱 Spring Data — Page<T> iterates over paginated results
```

---

## ⚡ When to Use

```
✅ Traverse complex structures (trees, graphs) without exposing them
✅ Support multiple traversal algorithms (in-order, pre-order, BFS, DFS)
✅ Provide uniform interface across different collection types
✅ Lazy evaluation — don't load all elements at once

❌ Simple arrays/lists — for-each loop is fine!
❌ Collection won't have multiple traversal strategies
```

---

## 🏆 Achievement: Progress: 17/23 patterns █████████████████░

---

*← [Template Method](./Template_Method.md) | [State →](./State.md)*
