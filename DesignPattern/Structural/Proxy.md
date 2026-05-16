# 🛡️ Proxy Pattern: Control Access Through a Stand-In! 🎯

> **"You don't talk directly to the celebrity. You talk to their agent (proxy) who controls access."**

---

## 🎬 The Story

```
TYPES OF PROXIES:
━━━━━━━━━━━━━━━━━

🏦 VIRTUAL PROXY (Lazy Loading):
   "Don't load that 10MB image until user scrolls to it"
   → YouTube thumbnails load → full video only on click

🔒 PROTECTION PROXY (Access Control):
   "Only admins can delete records"
   → Spring Security checking permissions before method execution

📡 REMOTE PROXY (Network):
   "Call this method as if the object is local"
   → Java RMI, gRPC stubs

📝 LOGGING PROXY:
   "Log every method call without modifying the original"
   → Spring AOP, debug interceptors

💾 CACHING PROXY:
   "Return cached result if available"
   → Redis cache layer, Hibernate second-level cache
```

---

## 💻 Java Implementation

```java
// 🖼️ Subject interface
public interface Image {
    void display();
    int getSize();
}

// 🎨 Real Subject (heavy, expensive)
public class HighResImage implements Image {
    private final String filename;
    private byte[] data; // 10MB!
    
    public HighResImage(String filename) {
        this.filename = filename;
        loadFromDisk(); // 🐌 Expensive!
    }
    
    private void loadFromDisk() {
        System.out.println("⏳ Loading " + filename + " from disk... (3 seconds)");
        this.data = new byte[10_000_000]; // 10MB
    }
    
    @Override public void display() { System.out.println("🖼️ Displaying " + filename); }
    @Override public int getSize() { return data.length; }
}

// 🛡️ PROXY — Virtual Proxy (Lazy Loading)
public class ImageProxy implements Image {
    private final String filename;
    private HighResImage realImage; // Created only when needed!
    
    public ImageProxy(String filename) {
        this.filename = filename;
        // ✅ NO loading happens here! Fast!
    }
    
    @Override
    public void display() {
        if (realImage == null) {
            realImage = new HighResImage(filename); // Load on first use!
        }
        realImage.display();
    }
    
    @Override
    public int getSize() {
        if (realImage == null) return 0; // Don't load just for size check
        return realImage.getSize();
    }
}

// 🎮 Usage
Image img = new ImageProxy("vacation_photo.jpg"); // ⚡ Instant! No loading!
// ... user scrolls down ...
img.display(); // NOW it loads (only when needed)
```

### Protection Proxy

```java
// 🔒 Access control proxy
public class SecureDocumentProxy implements Document {
    private final Document realDocument;
    private final User currentUser;
    
    public SecureDocumentProxy(Document doc, User user) {
        this.realDocument = doc;
        this.currentUser = user;
    }
    
    @Override
    public void read() {
        if (currentUser.hasPermission("READ")) {
            realDocument.read();
        } else {
            throw new AccessDeniedException("🚫 No read permission!");
        }
    }
    
    @Override
    public void write(String content) {
        if (currentUser.hasPermission("WRITE")) {
            realDocument.write(content);
        } else {
            throw new AccessDeniedException("🚫 No write permission!");
        }
    }
}
```

---

## 🌍 Real-World Examples

```java
// 🌱 Spring AOP — every @Transactional, @Cacheable creates a PROXY!
@Service
public class UserService {
    @Transactional  // Spring creates a proxy that manages transactions!
    public void createUser(User user) { /* ... */ }
    
    @Cacheable("users") // Proxy checks cache before calling real method!
    public User findById(Long id) { /* ... */ }
}

// 🗄️ Hibernate Lazy Loading — related entities are PROXIES!
@Entity
class Order {
    @ManyToOne(fetch = FetchType.LAZY) // Returns a proxy, loads on access!
    private Customer customer;
}

// 🌐 Java's java.lang.reflect.Proxy — dynamic proxies!
// 🔌 Java RMI — remote method invocation uses proxy stubs
```

---

## ⚡ When to Use

```
✅ Lazy initialization (virtual proxy) — expensive objects loaded on demand
✅ Access control (protection proxy) — permission checks
✅ Caching (caching proxy) — avoid repeated expensive operations
✅ Logging/monitoring — track method calls transparently
✅ Remote objects (remote proxy) — hide network complexity

❌ Simple objects with no access control needs
❌ When proxy adds latency to performance-critical paths
```

---

## 🎯 Key Interview Point

**Proxy vs Decorator:**
- **Proxy** controls ACCESS (same interface, manages lifecycle)
- **Decorator** adds BEHAVIOR (same interface, enhances functionality)
- Proxy usually creates/manages the real object's lifecycle
- Decorator receives the object to wrap (doesn't create it)

---

## 🏆 Achievement Unlocked!

```
╔══════════════════════════════════════╗
║  🏗️ ALL STRUCTURAL PATTERNS DONE!   ║
║  Progress: 12/23 patterns ████████████░ ║
╚══════════════════════════════════════╝
```

---

*← [Flyweight](./Flyweight.md) | [Behavioral Patterns →](../Behavioral/00_Behavioral_Overview.md)*
