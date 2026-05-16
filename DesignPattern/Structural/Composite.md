# 🌳 Composite Pattern: Treat One and Many the Same! 🎯

> **"A box can contain products AND other boxes. Treat them all as 'things in the shipment.'"**

---

## 🎬 The Story

### 📁 File System — Perfect Composite!

```
📁 Documents/
├── 📄 resume.pdf         (File — leaf)
├── 📄 cover_letter.doc   (File — leaf)
├── 📁 Projects/          (Folder — composite)
│   ├── 📄 design.png     (File — leaf)
│   └── 📁 Backend/       (Folder — composite)
│       ├── 📄 App.java   (File — leaf)
│       └── 📄 Test.java  (File — leaf)
└── 📁 Photos/            (Folder — composite)
    └── 📄 avatar.jpg     (File — leaf)

KEY INSIGHT:
• Files have a size
• Folders ALSO have a size (sum of contents)
• You treat them UNIFORMLY! folder.getSize() just works!
```

---

## 💡 The Solution

```
COMPOSITE PATTERN:
Compose objects into TREE structures to represent part-whole hierarchies.
Clients treat individual objects (leaves) and compositions (composites) UNIFORMLY.

Component (interface): defines operations for ALL objects in tree
├── Leaf: no children, implements actual operation
└── Composite: has children, delegates to them
```

---

## 🏗️ The Structure

```
              ┌────────────────────┐
              │  Component (intf)   │
              │ ────────────────── │
     ┌───────│ + operation()       │──────┐
     │        │ + add(Component)    │      │
     │        │ + remove(Component) │      │
     │        │ + getChild(int)     │      │
     │        └────────────────────┘      │
     │ implements                   implements
┌────▼─────┐                    ┌─────────▼────────┐
│   Leaf    │                    │    Composite      │
│ ──────── │                    │ ──────────────── │
│operation()│                    │ - children: List  │
│ → do work │                    │ + operation()     │
└───────────┘                    │   → for each child│
                                 │     child.op()    │
                                 │ + add/remove      │
                                 └──────────────────┘
```

---

## 💻 Java Implementation

```java
// 🧩 Component — common interface
public interface FileSystemItem {
    String getName();
    long getSize();
    void display(String indent);
}

// 🍃 Leaf — File
public class File implements FileSystemItem {
    private final String name;
    private final long size;
    
    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }
    
    @Override public String getName() { return name; }
    @Override public long getSize() { return size; }
    @Override public void display(String indent) {
        System.out.println(indent + "📄 " + name + " (" + size + " bytes)");
    }
}

// 🌳 Composite — Folder
public class Folder implements FileSystemItem {
    private final String name;
    private final List<FileSystemItem> children = new ArrayList<>();
    
    public Folder(String name) { this.name = name; }
    
    public void add(FileSystemItem item) { children.add(item); }
    public void remove(FileSystemItem item) { children.remove(item); }
    
    @Override public String getName() { return name; }
    
    @Override public long getSize() {
        return children.stream().mapToLong(FileSystemItem::getSize).sum(); // Recursive!
    }
    
    @Override public void display(String indent) {
        System.out.println(indent + "📁 " + name + " (" + getSize() + " bytes)");
        children.forEach(child -> child.display(indent + "  "));
    }
}

// 🎮 Usage
Folder root = new Folder("Documents");
root.add(new File("resume.pdf", 2048));
Folder projects = new Folder("Projects");
projects.add(new File("App.java", 4096));
projects.add(new File("Test.java", 1024));
root.add(projects);

root.display(""); // Treats everything uniformly!
System.out.println("Total size: " + root.getSize()); // Recursive sum!
```

---

## 🌍 Real-World Examples

```java
// 🎨 Java Swing/AWT — JPanel contains JButtons AND other JPanels!
JPanel panel = new JPanel();
panel.add(new JButton("Click"));
panel.add(new JPanel()); // Composite contains composite!

// 🌱 Spring — CompositeHealthIndicator
// Individual health checks compose into overall system health

// 📊 Organization structure
// CEO → VP → Manager → Engineer (each "reports" like a tree)
```

---

## ⚡ When to Use

```
✅ Tree/hierarchical structure (menus, org charts, file systems)
✅ Client should treat leaf and composite uniformly
✅ You want recursive composition

❌ Structure is flat, not hierarchical
❌ Leaf and composite have very different operations
```

---

## 🏆 Achievement Unlocked!

```
Progress: 8/23 patterns ████████░ 
```

---

*← [Bridge](./Bridge.md) | [Decorator →](./Decorator.md)*
