**System Design Interview Preparation Notes**

---

### 💡 1. Client-Server Architecture

**What is it?**  
A communication model where a client sends requests to a server, which processes and returns the appropriate response.

**👉 Why do we need this?**  
- Separates client interface from backend logic.
- Centralizes data and services.

**🎯 Use Case:**  
Web applications, mobile apps, cloud services.

**💬 Real-Life Example:**  
A browser requests product data from an e-commerce server.

**💪 Advantages:**
- Scalable and maintainable.
- Centralized control and updates.

**🚨 Disadvantages / Limitations:**
- Server is a single point of failure.
- Network latency impacts performance.

**🧠 Alternatives:**
- Peer-to-Peer networks.
- Microservices architecture.

**📋 Visual Representation:**
```
Client --> Server
```

---

### 💡 2. IP Address

**What is it?**  
A unique numeric address assigned to devices on a network for identification and communication.

**👉 Why do we need this?**  
- To locate and route traffic to devices across networks.

**🎯 Use Case:**  
When accessing any website, your device resolves its IP address.

**💬 Real-Life Example:**  
`google.com` resolves to `142.250.68.78`.

**💪 Advantages:**
- Enables global network communication.

**🚨 Disadvantages / Limitations:**
- IPv4 address exhaustion.

**🧠 Alternatives:**
- IPv6 for more address space.
- Hostnames (resolved by DNS).

**📋 Visual Representation:**
```
DNS Lookup --> IP Address
```

---

### 💡 3. DNS (Domain Name System)

**What is it?**  
A system that translates human-friendly domain names into IP addresses.

**👉 Why do we need this?**  
- Makes it easy to access servers without remembering numerical IPs.

**🎯 Use Case:**  
Every time you type a URL in your browser.

**💬 Real-Life Example:**  
`www.github.com` resolves to its server IP via DNS.

**💪 Advantages:**
- Simplifies navigation on the internet.

**🚨 Disadvantages / Limitations:**
- Susceptible to DNS attacks.

**🧠 Alternatives:**
- Local hosts file for manual mappings.

**📋 Visual Representation:**
```
Browser --> DNS Resolver --> IP Address
```

---

### 💡 4. Proxy / Reverse Proxy

**What is it?**  
A server that sits between clients and backend servers to forward requests.

**👉 Why do we need this?**  
- Load balancing, security, caching.

**🎯 Use Case:**  
Web traffic routing and protection.

**💬 Real-Life Example:**  
Nginx as a reverse proxy.

**💪 Advantages:**
- Improves performance and security.

**🚨 Disadvantages / Limitations:**
- Can become a bottleneck.

**🧠 Alternatives:**
- Direct server access.

**📋 Visual Representation:**
```
Client --> Proxy --> Server
```

---

### 💡 5. Latency

**What is it?**  
The delay between a client sending a request and receiving a response.

**👉 Why do we need to reduce this?**  
- Faster response improves user experience.

**🎯 Use Case:**  
Video streaming, online gaming.

**💬 Real-Life Example:**  
Buffering delays when streaming.

**💪 Advantages:**
- Low latency = better real-time interaction.

**🚨 Disadvantages / Limitations:**
- High latency degrades user experience.

**🧠 Alternatives:**
- Use of CDNs and caching.

**📋 Visual Representation:**
```
Request --> 🌐 --> Server --> 🌐 --> Response
```

---

### 💡 6. HTTP / HTTPS

**What is it?**  
Protocols for transferring data on the web, with HTTPS securing the transfer.

**👉 Why do we need this?**  
- Enables communication and ensures security over the internet.

**🎯 Use Case:**  
Accessing websites, APIs, online transactions.

**💬 Real-Life Example:**  
Secure banking websites use HTTPS.

**💪 Advantages:**
- HTTPS provides encrypted, secure communication.

**🚨 Disadvantages / Limitations:**
- HTTP is insecure by design.

**🧠 Alternatives:**
- gRPC for microservices.

**📋 Visual Representation:**
```
Client --> HTTPS --> Server
```

---

### 💡 7. APIs

**What is it?**  
A set of rules and protocols that allow different software systems to communicate with each other.

**👉 Why do we need this?**  
- Enables integration between services.
- Simplifies client-server communication.

**🎯 Use Case:**  
Mobile apps accessing backend data or third-party services.

**💬 Real-Life Example:**  
Google Maps API to embed maps in websites and apps.

**💪 Advantages:**
- Standardized communication.
- Reusability and scalability.

**🚨 Disadvantages / Limitations:**
- Versioning issues.
- Security vulnerabilities if mismanaged.

**🧠 Alternatives:**
- SDKs.
- Direct database queries (not recommended for distributed systems).

**📋 Visual Representation:**
```
Client --> API --> Server
```

---

### 💡 8. REST API

**What is it?**  
A web service that follows REST principles using HTTP methods like GET, POST, PUT, DELETE.

**👉 Why do we need this?**  
- Stateless communication.
- Simple and scalable web service architecture.

**🎯 Use Case:**  
CRUD operations for user profiles, products, or orders.

**💬 Real-Life Example:**  
Twitter's RESTful API for fetching tweets.

**💪 Advantages:**
- Language agnostic.
- Easy caching using HTTP standards.

**🚨 Disadvantages / Limitations:**
- Over-fetching or under-fetching data.
- Multiple round-trips for nested data.

**🧠 Alternatives:**
- GraphQL.
- gRPC.

**📋 Visual Representation:**
```
GET /users
POST /users
```

---

### 💡 9. GraphQL

**What is it?**  
A query language for APIs allowing clients to request only the data they need.

**👉 Why do we need this?**  
- Reduces over-fetching and under-fetching.
- Flexible and efficient.

**🎯 Use Case:**  
Mobile apps with dynamic and complex data requirements.

**💬 Real-Life Example:**  
Facebook uses GraphQL for its data layer.

**💪 Advantages:**
- Customizable queries.
- Strong typing via schemas.

**🚨 Disadvantages / Limitations:**
- Complex server setup.
- Caching can be challenging.

**🧠 Alternatives:**
- REST.
- gRPC.

**📋 Visual Representation:**
```
query {
  user(id: "1") {
    name
    email
  }
}
```

---

### 💡 10. Databases

**What is it?**  
Structured storage systems used for data management, retrieval, and persistence.

**👉 Why do we need this?**  
- Reliable storage and retrieval of data.
- Ensures data consistency and integrity.

**🎯 Use Case:**  
Storing user information, orders, logs, and configurations.

**💬 Real-Life Example:**  
MySQL, MongoDB, PostgreSQL.

**💪 Advantages:**
- Optimized querying.
- Structured data storage.

**🚨 Disadvantages / Limitations:**
- Schema rigidity in SQL databases.
- Scaling requires proper design.

**🧠 Alternatives:**
- Flat files.
- In-memory stores like Redis.

**📋 Visual Representation:**
```
Table: Users
+----+--------+----------+
| ID | Name   | Email    |
+----+--------+----------+
```

**System Design Interview Preparation Notes**

---

### 💡 11. SQL vs NoSQL

**What is it?**  
Two types of databases: SQL (Structured Query Language) for relational databases and NoSQL for flexible schema or unstructured data.

**👉 Why do we need this?**  
- SQL: When strong consistency and relationships between data are required.
- NoSQL: When flexibility, scalability, or high write throughput is a priority.

**🎯 Use Case:**  
- SQL: Banking systems, ERP applications.
- NoSQL: Social media feeds, real-time analytics.

**💬 Real-Life Example:**  
- SQL: MySQL, PostgreSQL.
- NoSQL: MongoDB, Cassandra.

**💪 Advantages:**
- SQL: ACID guarantees and structured data integrity.
- NoSQL: Schema-less design and horizontal scaling.

**🚨 Disadvantages / Limitations:**
- SQL: Vertical scaling limits.
- NoSQL: Eventual consistency and complex queries can be tricky.

**🧠 Alternatives:**
- NewSQL (CockroachDB, Google Spanner).

**📋 Visual Representation:**
```
SQL -> Tables, Rows, Joins
NoSQL -> Key-Value, Document, Graph, Column Store
```

---

### 💡 12. Vertical Scaling

**What is it?**  
Upgrading an existing machine by adding more CPU, RAM, or storage.

**👉 Why do we need this?**  
- Quick way to handle increased load without redesigning the system.

**🎯 Use Case:**  
Database servers and legacy applications.

**💬 Real-Life Example:**  
Upgrading an AWS EC2 instance from `t3.medium` to `m5.4xlarge`.

**💪 Advantages:**
- Simple to implement.
- Less complexity compared to horizontal scaling.

**🚨 Disadvantages / Limitations:**
- Limited by hardware specs.
- Single point of failure remains.

**🧠 Alternatives:**
- Horizontal Scaling (add more servers).

**📋 Visual Representation:**
```
Scale Up: Server -> Bigger Server
```

---

### 💡 13. Horizontal Scaling

**What is it?**  
Adding more servers to handle increased load, distributing the traffic among them.

**👉 Why do we need this?**  
- Infinite scalability potential.
- Improves availability and fault tolerance.

**🎯 Use Case:**  
Large web platforms, microservices, cloud-native apps.

**💬 Real-Life Example:**  
Netflix adds new instances when traffic spikes.

**💪 Advantages:**
- Fault tolerance.
- Elastic scaling.

**🚨 Disadvantages / Limitations:**
- More complex to manage.
- Requires load balancing.

**🧠 Alternatives:**
- Vertical Scaling.

**📋 Visual Representation:**
```
Users --> Load Balancer --> Multiple Servers
```

---

### 💡 14. Load Balancer

**What is it?**  
A component that distributes network traffic across multiple servers to ensure system reliability and efficiency.

**👉 Why do we need this?**  
- Prevents any single server from being overloaded.

**🎯 Use Case:**  
E-commerce sites, social media apps.

**💬 Real-Life Example:**  
AWS Elastic Load Balancer (ELB).

**💪 Advantages:**
- Improved fault tolerance.
- Smooth traffic distribution.

**🚨 Disadvantages / Limitations:**
- Single point of failure if not configured redundantly.

**🧠 Alternatives:**
- DNS Round Robin.
- Client-side load balancing.

**📋 Visual Representation:**
```
Clients -> Load Balancer -> Servers
```

---

### 💡 15. Database Indexing

**What is it?**  
A technique to optimize database performance by reducing lookup time for queries.

**👉 Why do we need this?**  
- Improves read/query speed significantly.

**🎯 Use Case:**  
E-commerce sites improving product search speed.

**💬 Real-Life Example:**  
Indexing `username` in the login table for fast user lookups.

**💪 Advantages:**
- Speeds up queries.
- Reduces CPU load for frequent searches.

**🚨 Disadvantages / Limitations:**
- Slower write operations.
- Increased storage usage.

**🧠 Alternatives:**
- Caching for frequently accessed data.

**📋 Visual Representation:**
```
Index: B-Tree / Hash Map
```

---

### 💡 16. Replication

**What is it?**  
Copying data from one database to another to ensure redundancy and improve availability.

**👉 Why do we need this?**  
- Read scalability.
- Fault tolerance and backup.

**🎯 Use Case:**  
Web applications distributing read queries to replicas.

**💬 Real-Life Example:**  
MySQL Master-Slave replication.

**💪 Advantages:**
- Improved availability.
- Supports disaster recovery.

**🚨 Disadvantages / Limitations:**
- Replication lag.
- Eventual consistency.

**🧠 Alternatives:**
- Sharding for data distribution.

**📋 Visual Representation:**
```
Primary DB -> Secondary Replica(s)
```

---

### 💡 17. Sharding

**What is it?**  
Splitting a database into smaller pieces (shards) distributed across different machines.

**👉 Why do we need this?**  
- Allows horizontal scalability for large datasets.

**🎯 Use Case:**  
Social networks storing billions of user records.

**💬 Real-Life Example:**  
MongoDB uses shard keys to distribute documents.

**💪 Advantages:**
- Efficient for huge datasets.
- Reduces query time for partitioned data.

**🚨 Disadvantages / Limitations:**
- Complex balancing.
- Risk of uneven data distribution.

**🧠 Alternatives:**
- Vertical partitioning.

**📋 Visual Representation:**
```
Shard1: IDs 1-1000 | Shard2: IDs 1001-2000
```

---

### 💡 18. Vertical Partitioning

**What is it?**  
Splitting database tables by columns to isolate frequently accessed data.

**👉 Why do we need this?**  
- To improve I/O efficiency and reduce table size.

**🎯 Use Case:**  
Splitting user profile fields from user authentication fields.

**💬 Real-Life Example:**  
User table split into: `user_auth` and `user_profile`.

**💪 Advantages:**
- Faster queries for subsets of data.

**🚨 Disadvantages / Limitations:**
- More complex joins.

**🧠 Alternatives:**
- Horizontal partitioning.

**📋 Visual Representation:**
```
TableA: ID, Name
TableB: ID, Address, Preferences
```

---

### 💡 19. Caching

**What is it?**  
Temporarily storing frequently accessed data to reduce database or computation load.

**👉 Why do we need this?**  
- Reduces response time.
- Lowers backend stress.

**🎯 Use Case:**  
Session storage, query results, static content.

**💬 Real-Life Example:**  
Redis caching recent search queries.

**💪 Advantages:**
- Fast data retrieval.
- Scalability boost.

**🚨 Disadvantages / Limitations:**
- Stale data risk.
- Cache invalidation complexity.

**🧠 Alternatives:**
- Indexing.
- Pre-computation.

**📋 Visual Representation:**
```
Client -> Cache -> DB
```

---

### 💡 20. Denormalization

**What is it?**  
Introducing redundancy in a database to reduce the need for joins and optimize read speed.

**👉 Why do we need this?**  
- Read-heavy systems benefit from reduced join overhead.

**🎯 Use Case:**  
Reporting, dashboards, search indexes.

**💬 Real-Life Example:**  
Duplicating customer address into orders table to avoid joins.

**💪 Advantages:**
- Faster read performance.

**🚨 Disadvantages / Limitations:**
- Higher risk of data inconsistency.
- Storage overhead.

**🧠 Alternatives:**
- Proper indexing.

**📋 Visual Representation:**
```
Orders Table: OrderID, CustomerID, Address (duplicated)
```

**System Design Interview Preparation Notes**

---

### 💡 21. CAP Theorem

**What is it?**  
A principle stating a distributed system can only guarantee two out of three: Consistency, Availability, Partition Tolerance.

**👉 Why do we need this?**  
- Helps design trade-offs for distributed systems.

**🎯 Use Case:**  
Cloud-native databases, distributed file storage.

**💬 Real-Life Example:**  
MongoDB prioritizing Consistency and Partition Tolerance in default settings.

**💪 Advantages:**
- Simplifies architectural choices.
- Sets clear expectations for fault scenarios.

**🚨 Disadvantages / Limitations:**
- Can't have all three in a network partition scenario.

**🧠 Alternatives:**
- Eventual consistency for highly available systems.

**📋 Visual Representation:**
```
Pick any 2: Consistency, Availability, Partition Tolerance
```

---

### 💡 22. Blob Storage

**What is it?**  
Storage for large, unstructured data like media files and backups.

**👉 Why do we need this?**  
- Store and retrieve objects without size constraints.
- Low-cost and highly scalable.

**🎯 Use Case:**  
Storing user-uploaded images, videos, backups.

**💬 Real-Life Example:**  
Amazon S3, Azure Blob Storage.

**💪 Advantages:**
- Elastic scalability.
- Suitable for static and large objects.

**🚨 Disadvantages / Limitations:**
- Higher latency compared to in-memory storage.
- Limited direct query capabilities.

**🧠 Alternatives:**
- Distributed file systems.

**📋 Visual Representation:**
```
User Upload --> Blob Storage --> URL Access
```

---

### 💡 23. CDN (Content Delivery Network)

**What is it?**  
A globally distributed network of servers to cache and deliver content closer to users.

**👉 Why do we need this?**  
- Reduces latency and improves page load speed.
- Handles high traffic more effectively.

**🎯 Use Case:**  
Media websites, e-commerce platforms, streaming services.

**💬 Real-Life Example:**  
Cloudflare, Akamai, AWS CloudFront.

**💪 Advantages:**
- Faster content delivery.
- Offloads origin servers.

**🚨 Disadvantages / Limitations:**
- Cache invalidation complexity.

**🧠 Alternatives:**
- Edge computing.

**📋 Visual Representation:**
```
Client --> Nearest CDN Server --> Origin Server (on cache miss)
```

---

### 💡 24. WebSockets

**What is it?**  
A persistent connection protocol that allows real-time, two-way communication.

**👉 Why do we need this?**  
- Reduces overhead for continuous updates.
- Enables real-time applications.

**🎯 Use Case:**  
Chat apps, real-time notifications, multiplayer games.

**💬 Real-Life Example:**  
Slack and WhatsApp Web for instant message delivery.

**💪 Advantages:**
- Bi-directional communication.
- Low latency.

**🚨 Disadvantages / Limitations:**
- Harder to scale compared to stateless HTTP.

**🧠 Alternatives:**
- HTTP long polling.

**📋 Visual Representation:**
```
Client <==> WebSocket Server (Persistent Connection)
```

---

### 💡 25. Webhooks

**What is it?**  
An event-driven callback mechanism triggered by external systems.

**👉 Why do we need this?**  
- Real-time data sync between systems.
- No need for constant polling.

**🎯 Use Case:**  
Payment gateways sending transaction notifications.

**💬 Real-Life Example:**  
Stripe webhooks notify e-commerce systems on payment completion.

**💪 Advantages:**
- Immediate event-driven updates.

**🚨 Disadvantages / Limitations:**
- Requires retry logic for network issues.

**🧠 Alternatives:**
- Polling or message queues.

**📋 Visual Representation:**
```
Event Occurs --> Webhook Fires --> Receiver Processes
```

---

### 💡 26. Microservices

**What is it?**  
Breaking large applications into smaller, independently deployable services.

**👉 Why do we need this?**  
- Easier to scale, develop, and maintain components.

**🎯 Use Case:**  
Large scale platforms like Netflix, Uber, Amazon.

**💬 Real-Life Example:**  
Netflix uses microservices for account, playback, and recommendations.

**💪 Advantages:**
- Independent scalability.
- Fault isolation.

**🚨 Disadvantages / Limitations:**
- More complex system design.

**🧠 Alternatives:**
- Monolithic or modular monolith architectures.

**📋 Visual Representation:**
```
Client --> API Gateway --> Microservices A / B / C
```

---

### 💡 27. Message Queues

**What is it?**  
A software component for asynchronous communication between services.

**👉 Why do we need this?**  
- Decouples systems for better fault tolerance and scalability.

**🎯 Use Case:**  
Order processing, notifications, logs.

**💬 Real-Life Example:**  
RabbitMQ, Apache Kafka, AWS SQS.

**💪 Advantages:**
- Handles traffic spikes.
- Increases system resilience.

**🚨 Disadvantages / Limitations:**
- Adds latency.
- Requires monitoring for queue overflow.

**🧠 Alternatives:**
- HTTP-based synchronous communication.

**📋 Visual Representation:**
```
Producer --> Queue --> Consumer
```

---

### 💡 28. Rate Limiting

**What is it?**  
Controlling the number of requests sent to a server in a given time window.

**👉 Why do we need this?**  
- Prevents abuse and DoS attacks.

**🎯 Use Case:**  
API usage, authentication systems.

**💬 Real-Life Example:**  
GitHub API limiting requests to 5000/hour per user.

**💪 Advantages:**
- Fair resource distribution.

**🚨 Disadvantages / Limitations:**
- Can block legitimate requests during bursts.

**🧠 Alternatives:**
- Circuit breakers, back-pressure.

**📋 Visual Representation:**
```
Client --> Rate Limiter --> Server
```

---

### 💡 29. API Gateway

**What is it?**  
A service that acts as a single entry point for client requests to microservices.

**👉 Why do we need this?**  
- Centralizes security, routing, and rate limiting.

**🎯 Use Case:**  
Microservices-based applications.

**💬 Real-Life Example:**  
AWS API Gateway, Kong.

**💪 Advantages:**
- Simplifies client-server interaction.

**🚨 Disadvantages / Limitations:**
- Single point of failure unless highly available.

**🧠 Alternatives:**
- Service meshes like Istio.

**📋 Visual Representation:**
```
Client --> API Gateway --> Services
```

---

### 💡 30. Idempotency

**What is it?**  
An operation that produces the same result if performed once or multiple times.

**👉 Why do we need this?**  
- Makes operations safe for retries and fault recovery.

**🎯 Use Case:**  
Payment processing, database updates.

**💬 Real-Life Example:**  
PUT `/user/123` should always result in the same user data.

**💪 Advantages:**
- Avoids unintended side effects during retries.

**🚨 Disadvantages / Limitations:**
- Complex design for non-idempotent scenarios.

**🧠 Alternatives:**
- Retry tokens or unique request identifiers.

**📋 Visual Representation:**
```
Same Input --> Same Output (even on multiple executions)
```
