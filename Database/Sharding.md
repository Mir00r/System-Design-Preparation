# 🚀 **The Ultimate Guide to Database Sharding: Interview Prep & Beyond**

Database sharding is a **scaling powerhouse** used by tech giants like Uber, Airbnb, and Netflix. This guide covers **everything**—from basic concepts to advanced strategies, Java examples, and real-world implementations—to help you **ace interviews** and **design scalable systems**.

---

## 🎯 **Table of Contents**
1. [What is Database Sharding?](#-what-is-database-sharding)
2. [Why Shard? Pros & Cons](#-why-shard-pros--cons)
3. [Sharding Strategies](#-sharding-strategies)
4. [Industry Use Cases](#-industry-use-cases)
5. [Best Practices](#-best-practices)
6. [How Big Companies Shard](#-how-big-companies-shard)
7. [Recommended Technologies](#-recommended-technologies)
8. [Java Code Examples](#-java-code-examples)
9. [Interview Q&A](#-interview-qa)
10. [Conclusion](#-conclusion)

---

## 🔍 **1. What is Database Sharding?**
**Definition:** Sharding is **horizontal partitioning** of a database into smaller, faster, more manageable pieces called **shards**, each stored on separate servers.

### **Key Concepts**
- **Horizontal vs. Vertical Partitioning:**
    - **Horizontal (Sharding):** Splits **rows** across servers (e.g., users 1-100M on Server A, 100M-200M on Server B).
    - **Vertical:** Splits **columns** (e.g., user_profile on Server A, user_settings on Server B).

- **Shard Key:** The column used to distribute data (e.g., `user_id`, `country`).

**Diagram:**
```
         [Database]
        /    |    \
   [Shard1][Shard2][Shard3]
   (USA)   (EU)    (Asia)
```  

---

## ⚖️ **2. Why Shard? Pros & Cons**

### **When to Shard?**
✅ **High write throughput** (e.g., social media posts).  
✅ **Geo-distributed users** (reduce latency).  
✅ **Data exceeds single-server capacity** (>1TB).

### **Pros & Cons**
| **Pros** | **Cons** |  
|----------|----------|  
| 📈 **Infinite scalability** | 🔄 **Complex joins across shards** |  
| ⚡ **Faster queries** (smaller datasets) | ⚠️ **No native consistency guarantees** |  
| 🌍 **Geo-distribution** | 🛠 **Hard to reshard** |  

**Industry Example:**
- **Uber:** Sharded by city (`city_id`) to isolate traffic data.

---

## 🧩 **3. Sharding Strategies**

### **1. Range-Based Sharding**
- Partitions data by ranges (e.g., `user_id 1-1000` → Shard A).
- **Use Case:** Time-series data (e.g., logs by date).

### **2. Hash-Based Sharding**
- Uses a hash function (e.g., `hash(user_id) % 4`) to assign shards.
- **Use Case:** Evenly distributed workloads (e.g., user profiles).

### **3. Directory-Based Sharding**
- Uses a lookup table to map shard keys to shards.
- **Use Case:** Flexible, dynamic resharding (e.g., multi-tenant apps).

**Comparison Table:**  
| **Strategy** | **Pros** | **Cons** |  
|-------------|---------|---------|  
| **Range** | Simple to implement | Hotspots (e.g., recent data) |  
| **Hash** | Even distribution | Hard to range queries |  
| **Directory** | Flexible | Extra lookup overhead |

---

## 🏢 **4. Industry Use Cases**
| **Company** | **Sharding Approach** |  
|------------|----------------------|  
| **Uber** | City-based sharding (`city_id`) |  
| **Airbnb** | Listing geo-sharding (`country_code`) |  
| **Netflix** | User-based sharding (`user_id`) |  
| **Discord** | Snowflake ID + hash-based sharding |  

---

## 🛠 **5. Best Practices**
1. **Choose the right shard key** (avoid hotspots!).
2. **Plan for resharding** (use consistent hashing).
3. **Implement cross-shard queries carefully** (avoid if possible).
4. **Monitor shard balance** (tools like Vitess).

**Anti-Pattern:** Sharding too early (start with read replicas first!).

---

## 🌍 **6. How Big Companies Shard**

### **Uber’s Architecture**
1. **Pre-Sharding:** Single PostgreSQL → Performance issues.
2. **Post-Sharding:**
    - Split by `city_id`.
    - Used Schemaless (custom sharding layer).

### **Instagram’s Approach**
- Sharded by `user_id` using Django.
- Each shard = separate PostgreSQL instance.

---

## 💻 **7. Recommended Technologies**
| **Tech** | **Use Case** |  
|----------|-------------|  
| **Vitess** | MySQL sharding (used by YouTube) |  
| **MongoDB** | Auto-sharding for NoSQL |  
| **Citus** | PostgreSQL extension for sharding |  
| **ShardingSphere** | JDBC-based sharding (Java) |  

---

## ☕ **8. Java Code Examples**

### **Sharding with Spring Boot + ShardingSphere**
```java
// 1. Define sharding rule (hash-based on user_id)
@Bean
public DataSource dataSource() throws SQLException {
    Map<String, DataSource> dataSourceMap = new HashMap<>();
    dataSourceMap.put("ds0", createDataSource("jdbc:mysql://db0:3306/db"));
    dataSourceMap.put("ds1", createDataSource("jdbc:mysql://db1:3306/db"));

    ShardingRuleConfiguration ruleConfig = new ShardingRuleConfiguration();
    ruleConfig.getTableRuleConfigs().add(
        new TableRuleConfiguration("users", "ds${0..1}.users_${0..1}"));

    // Hash sharding by user_id
    ruleConfig.setDefaultDatabaseShardingStrategyConfig(
        new InlineShardingStrategyConfiguration("user_id", "ds${user_id % 2}"));

    return ShardingDataSourceFactory.createDataSource(dataSourceMap, ruleConfig, null);
}

// 2. Query transparently (ShardingSphere routes to correct shard)
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    @Query("SELECT * FROM users WHERE user_id = :userId") // Auto-routed!
    User findByUserId(@Param("userId") long userId);
}
```

---

## ❓ **9. Interview Q&A**

### **Q1: What’s the difference between sharding and partitioning?**
**A:**
- **Sharding:** Distributes data **across servers**.
- **Partitioning:** Divides data **within a single server**.

### **Q2: How do you handle joins across shards?**
**A:**
1. **Denormalize data** (avoid joins).
2. **Use edge caching** (Redis).
3. **Batch queries + merge in app** (slow!).

### **Q3: What’s a hotspot and how do you avoid it?**
**A:**
- **Hotspot:** Uneven load (e.g., all traffic to Shard A).
- **Fix:** Choose a better shard key (e.g., `hash(user_id)` instead of `country`).

### **Q4: When would you NOT shard?**
**A:**  
❌ Small datasets (<1TB).  
❌ Heavy cross-shard transactions.  
❌ No DevOps team to manage complexity.

### **Q5: How does resharding work?**
**A:**
1. **Double-write** to old + new shards.
2. **Backfill** historical data.
3. **Migrate reads** → **Migrate writes** → **Retire old shards**.

---

## 🎯 **10. Conclusion**
- **Sharding = Ultimate scalability** for write-heavy apps.
- **Choose strategy wisely** (hash vs. range vs. directory).
- **Big tech uses it** (Uber, Airbnb, Netflix).
- **Java libraries** like ShardingSphere simplify implementation.

🚀 **Pro Tip:** In interviews, discuss trade-offs (e.g., "Sharding improves writes but complicates reads").

---

**📢 Share this guide!** #Database #Sharding #InterviewPrep

🔗 **Further Reading:**
- [Vitess Documentation](https://vitess.io/)
- [MongoDB Sharding Guide](https://www.mongodb.com/docs/manual/sharding/)

Now go **scale those databases!** 💪
