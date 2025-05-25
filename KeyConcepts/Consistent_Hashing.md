# **Consistent Hashing Explained: The Secret Sauce Behind Scalable Distributed Systems**

## **🚀 Introduction: Why Traditional Hashing Fails in Distributed Systems**

Imagine you're running a high-traffic web application with **5 backend servers (S0-S4)**. You distribute user requests using a simple hashing strategy:

```python
server_index = hash(user_ip) % 5
```  

This works perfectly—until you **add or remove a server**. Suddenly, `hash(user_ip) % 6` reassigns **most requests** to new servers, causing:

- **Session disconnections** 🔌
- **Cache misses** 🗑️
- **Database overload** 🏋️

**Traditional hashing is brittle in dynamic systems.** Enter **consistent hashing**—a game-changer for distributed databases (DynamoDB, Cassandra) and load balancers.

---

## **🔍 Problem: The Pitfalls of Modulo-Based Hashing**

### **Scenario 1: Adding a Server (S5)**
- Original mapping: `hash(ip) % 5`
- New mapping: `hash(ip) % 6`

**Result:**
- User A’s request shifts from **S1 → S5**
- User B’s request shifts from **S0 → S5**
- **👉 80% of users get reassigned!**

### **Scenario 2: Removing a Server (S4)**
- Original mapping: `hash(ip) % 5`
- New mapping: `hash(ip) % 4`

**Result:**
- User E’s request shifts from **S4 → S2**
- **👉 60% of users are remapped!**

**Pain Points:**
- **Massive rehashing** 🔄
- **Cache invalidation** ❌
- **Poor scalability** 📉

---

## **🎯 Solution: Consistent Hashing**

### **✨ How It Works**
1. **Hash Ring Construction** 🎡
    - Servers and keys are mapped to a **virtual ring** (e.g., `0` to `2³²-1`).
    - Each server (e.g., `S0`) is placed at `Hash(S0)` on the ring.

2. **Key Assignment** 🔑
    - A key (e.g., user IP) is hashed and placed on the ring.
    - The **next server clockwise** owns the key.

**Example:**
- `Hash(User A) = 11` → Assigned to **S1** (next server after 11).
- `Hash(User D) = 17` → Assigned to **S2**.

![Hash Ring](https://i.imgur.com/xyz123.png)

---

## **🔄 Handling Server Changes Gracefully**

### **Case 1: Adding a Server (S5)**
- S5 is placed between S1 and S2.
- **Only keys between S1 and S5 are reassigned** (e.g., User D moves from S2 → S5).
- **Minimal disruption!**

### **Case 2: Removing a Server (S4)**
- Keys owned by S4 are reassigned to **S3** (next server).
- **Only S4’s keys are affected.**

**Advantages:**
- **O(1/N) keys remapped** (vs. O(N) in traditional hashing).
- **No cache thrashing** 🚫🗑️

---

## **⚖️ Virtual Nodes: Solving Load Imbalance**

**Problem:** Basic consistent hashing can create **hotspots** if servers are unevenly spaced.

**Solution:** **Virtual nodes (VNodes)**!
- Each physical server (e.g., `S1`) is represented by **multiple virtual nodes** (e.g., `S1-1`, `S1-2`).
- Distributes keys more evenly.

**Example:**
- Without VNodes:
    - `S1` at position 10 → Overloaded if adjacent keys pile up.
- With VNodes:
    - `S1-1` at 10, `S1-2` at 70, `S1-3` at 120 → **Balanced load**.

![Virtual Nodes](https://i.imgur.com/abc456.png)

---

## **💻 Code Implementation (Java)**

```java
import java.util.*;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class ConsistentHashing {
  private final int numReplicas; // Number of virtual nodes per server
  private final TreeMap<Long, String> ring; // Hash ring (key: hash value, value: server)
  private final Set<String> servers; // Physical servers

  public ConsistentHashing(List<String> servers, int numReplicas) {
    this.numReplicas = numReplicas;
    this.ring = new TreeMap<>();
    this.servers = new HashSet<>();

    for (String server : servers) {
      addServer(server);
    }
  }

  // MD5 hash function (returns a long)
  private long hash(String key) {
    try {
      MessageDigest md = MessageDigest.getInstance("MD5");
      byte[] digest = md.digest(key.getBytes());
      return ((long) (digest[0] & 0xFF) << 56) |
        ((long) (digest[1] & 0xFF) << 48) |
        ((long) (digest[2] & 0xFF) << 40) |
        ((long) (digest[3] & 0xFF) << 32) |
        ((long) (digest[4] & 0xFF) << 24) |
        ((long) (digest[5] & 0xFF) << 16) |
        ((long) (digest[6] & 0xFF) << 8) |
        ((long) (digest[7] & 0xFF));
    } catch (NoSuchAlgorithmException e) {
      throw new RuntimeException("MD5 algorithm not found", e);
    }
  }

  // Add a server and its virtual nodes to the ring
  public void addServer(String server) {
    servers.add(server);
    for (int i = 0; i < numReplicas; i++) {
      String replica = server + "-" + i;
      long hash = hash(replica);
      ring.put(hash, server);
    }
  }

  // Remove a server and its virtual nodes from the ring
  public void removeServer(String server) {
    if (!servers.contains(server)) {
      return;
    }
    servers.remove(server);
    for (int i = 0; i < numReplicas; i++) {
      String replica = server + "-" + i;
      long hash = hash(replica);
      ring.remove(hash);
    }
  }

  // Get the server responsible for a key
  public String getServer(String key) {
    if (ring.isEmpty()) {
      return null;
    }
    long hash = hash(key);
    Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
    if (entry == null) {
      // Wrap around to the first server
      entry = ring.firstEntry();
    }
    return entry.getValue();
  }

  public static void main(String[] args) {
    List<String> servers = Arrays.asList("S0", "S1", "S2", "S3", "S4");
    ConsistentHashing ch = new ConsistentHashing(servers, 3); // 3 virtual nodes per server

    // Example usage
    String userIp = "192.168.1.10";
    String assignedServer = ch.getServer(userIp);
    System.out.println("Request from " + userIp + " is routed to: " + assignedServer);

    // Add a new server
    ch.addServer("S5");
    System.out.println("Added S5. Reassigned requests minimized.");

    // Remove a server
    ch.removeServer("S2");
    System.out.println("Removed S2. Only S2's keys were reassigned.");
  }
}
```

**Key Features:**
- **Virtual nodes** for load balancing.
- **O(log N) lookup** using binary search (`bisect`).

---

## **🚀 Real-World Applications**
1. **Distributed Caches** (Redis, Memcached)
2. **Database Sharding** (Cassandra, DynamoDB)
3. **Load Balancers** (NGINX, HAProxy)

---

## 📊 Performance Comparison

| Operation       | Traditional Hashing | Consistent Hashing |
|-----------------|---------------------|--------------------|
| Add 1 server   | O(k) keys move      | O(k/n) keys move   |
| Remove 1 server | O(k) keys move      | O(k/n) keys move   |
| Lookup         | O(1)                | O(log n)           |

## 🚀 Pro Tips

1. **Choose a good hash function** (MD5, SHA-1 are common)
2. **Tune virtual nodes** based on your load balancing needs
3. **Monitor distribution** to detect hotspots
4. **Consider weighted distribution** for heterogeneous servers

---

## **🎯 Key Takeaways for Interviews**
✔ **Problem:** Traditional hashing (`mod N`) fails with dynamic servers.  
✔ **Solution:** Consistent hashing remaps **only `K/N` keys** on changes.  
✔ **Optimization:** Virtual nodes prevent hotspots.  
✔ **Use Cases:** Databases, CDNs, load balancers.

**Next Steps:** Explore **rendezvous hashing** or **weighted consistent hashing** for advanced scenarios!

---

**💬 Got questions? Ask below!**  
**🔗 Share this with your network if you found it helpful!**

---

### **📚 Further Reading**
- [Amazon’s Dynamo Paper](https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)
- [Consistent Hashing in Cassandra](https://cassandra.apache.org/)

**Credits:** Inspired by [Ashish Pratap Singh](https://blog.algomaster.io/p/consistent-hashing-explained).

---

This guide combines **theory**, **visuals**, and **code** to master consistent hashing—essential for **system design interviews** and **scalable architectures**. 🚀
