# 🪶 Flyweight Pattern: Share to Save Memory! 🎯

> **"Why store 'Arial, 12pt, Black' for EVERY character when they all share the same formatting?"**

---

## 🎬 The Story

### 📝 Text Editor Problem

```
A document with 1,000,000 characters.
Each character object stores: character, font, size, color, position

WITHOUT FLYWEIGHT:
1,000,000 × (char + font + size + color + x + y) = 💀 MASSIVE MEMORY!

WITH FLYWEIGHT:
• INTRINSIC (shared): font, size, color → only 26 objects (one per letter style!)
• EXTRINSIC (unique): position → stored externally

Result: 95% memory reduction! 🎉
```

---

## 💡 Key Concept

```
FLYWEIGHT splits object data into:

INTRINSIC STATE (shared, immutable):
→ Data that's the SAME across many objects
→ Stored INSIDE the flyweight
→ Example: font, color, texture

EXTRINSIC STATE (unique, varies):
→ Data that's UNIQUE to each context
→ Stored OUTSIDE (passed in by client)
→ Example: position, index, context-specific data
```

---

## 💻 Java Implementation

```java
// 🌳 Flyweight: shared tree type data
public class TreeType {
    private final String name;     // "Oak", "Pine" — shared!
    private final String color;    // "Green" — shared!
    private final String texture;  // Large texture data — shared!
    
    public TreeType(String name, String color, String texture) {
        this.name = name;
        this.color = color;
        this.texture = texture;
    }
    
    public void draw(int x, int y) { // x, y = extrinsic state, passed in
        System.out.printf("🌳 Drawing %s tree at (%d,%d)%n", name, x, y);
    }
}

// 🏭 Flyweight Factory — ensures sharing
public class TreeFactory {
    private static final Map<String, TreeType> cache = new HashMap<>();
    
    public static TreeType getTreeType(String name, String color, String texture) {
        String key = name + "_" + color;
        return cache.computeIfAbsent(key, k -> {
            System.out.println("🆕 Creating new TreeType: " + name);
            return new TreeType(name, color, texture);
        });
    }
    
    public static int getCacheSize() { return cache.size(); }
}

// 🌲 Context — stores extrinsic state + flyweight reference
public class Tree {
    private final int x, y;              // Extrinsic (unique per tree)
    private final TreeType type;         // Flyweight (shared!)
    
    public Tree(int x, int y, TreeType type) {
        this.x = x; this.y = y; this.type = type;
    }
    
    public void draw() { type.draw(x, y); }
}

// 🎮 Usage — 1 million trees, only ~5 TreeType objects!
public class Forest {
    private final List<Tree> trees = new ArrayList<>();
    
    public void plantTree(int x, int y, String name, String color, String texture) {
        TreeType type = TreeFactory.getTreeType(name, color, texture); // Shared!
        trees.add(new Tree(x, y, type));
    }
    
    public static void main(String[] args) {
        Forest forest = new Forest();
        for (int i = 0; i < 1_000_000; i++) {
            forest.plantTree(rand(), rand(), "Oak", "Green", "oak_texture.png");
        }
        System.out.println("Trees planted: 1,000,000");
        System.out.println("TreeType objects: " + TreeFactory.getCacheSize()); // Just 1! 🎉
    }
}
```

---

## 🌍 Real-World Examples

```java
// ☕ Java's Integer.valueOf() — caches -128 to 127!
Integer a = Integer.valueOf(42); // Shared flyweight!
Integer b = Integer.valueOf(42); // Same object!
System.out.println(a == b);      // true! (same reference)

// 📝 String Pool — Java's built-in Flyweight!
String s1 = "hello"; // Flyweight from pool
String s2 = "hello"; // Same object!

// 🎮 Game engines — particle systems (millions of particles, few textures)
// 🌐 Browser — DOM elements sharing CSS styles
```

---

## ⚡ When to Use

```
✅ Millions of similar objects consuming too much RAM
✅ Objects have large shared (intrinsic) state
✅ Extrinsic state can be computed or stored externally
✅ Identity of individual objects doesn't matter

❌ Few objects — overhead isn't worth it
❌ All state is unique (nothing to share)
❌ Objects need to be mutable
```

---

## 🏆 Achievement: Progress: 11/23 patterns ███████████░

---

*← [Facade](./Facade.md) | [Proxy →](./Proxy.md)*
