# 🗺️ System Design Preparation — Master Index

> **Your single entry point to everything. Start here. Always.**

```
┌─────────────────────────────────────────────────────────────────────┐
│         THE SINGLE-POINT SYSTEM DESIGN PREPARATION HUB              │
│                                                                     │
│  For ANY engineer type:  Junior • Senior • Staff • Principal        │
│  For ANY domain:         Backend • Frontend • Full-Stack • DevOps   │
│  For ANY tech stack:     Java • Python • Go • Node.js • .NET        │
│  For ANY goal:           FAANG • Startup • Enterprise • Research    │
│  For ANY level:          Beginner • Intermediate • Advanced         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🧭 Quick-Start Wizard

**Answer 3 questions to find your perfect starting point:**

### Q1: What's your experience level?
| Answer | Go Here |
|---|---|
| 🌱 Student / 0-1 years | [Track 5: CS Student Path](#-track-5--cs-student--bootcamp-grad) |
| ⚡ Junior / 1-3 years | [Track 1: Junior Engineer Path](#-track-1--junior-engineer) |
| 🔥 Mid-Level / 3-5 years | [Track 2: Mid-Level Engineer Path](#-track-2--mid-level-engineer) |
| 🚀 Senior / 5+ years | [Track 3: Senior Engineer Path](#-track-3--senior-engineer) |
| 🛠️ DevOps / Platform | [Track 4: DevOps Engineer Path](#-track-4--devopsplatform-engineer) |

### Q2: What's your immediate goal?
| Goal | Go Here |
|---|---|
| 🎯 Crack a system design interview next month | [SystemDesignCaseStudies/](./SystemDesignCaseStudies/README.md) |
| 📐 Master Data Structures & Algorithms | [DataStructures/](./DataStructures/README.md) |
| 🏗️ Understand distributed systems deeply | [KeyConcepts/](./KeyConcepts/) → [BuildingBlocks/](./BuildingBlocks/) |
| 🔐 Level up on security | [Security/](./Security/) |
| ☁️ Learn cloud architecture | [Cloud/](./Cloud/README.md) |
| 🧪 Improve software design skills | [DesignPattern/](./DesignPattern/README.md) |

### Q3: How much time do you have?
| Time | Go Here |
|---|---|
| 🔥 1 week (sprint) | [7-Day Crash Plan](./InterviewPrep/30_Day_Plan.md) |
| 📅 1 month | [30-Day Plan](./InterviewPrep/30_Day_Plan.md) |
| 📆 2 months | [60-Day Plan](./InterviewPrep/60_Day_Plan.md) |
| 🗓️ 3 months (thorough) | [90-Day Plan](./InterviewPrep/90_Day_Plan.md) |

---

## 📚 Domain Directory

### 🔑 Core Distributed Systems Theory
> *The "why" behind every design decision*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [CAPTheorem.md](./KeyConcepts/CAPTheorem.md) | Consistency vs Availability vs Partition Tolerance | 🟡 Medium | ✅ |
| [ACID_Transaction.md](./KeyConcepts/ACID_Transaction.md) | Database transaction guarantees | 🟢 Easy | ✅ |
| [Scalability.md](./KeyConcepts/Scalability.md) | Horizontal vs vertical scaling | 🟢 Easy | ✅ |
| [Availability.md](./KeyConcepts/Availability.md) | Uptime, SLA, redundancy | 🟢 Easy | ✅ |
| [Consistent_Hashing.md](./KeyConcepts/Consistent_Hashing.md) | Distributed hash rings | 🔴 Hard | ✅ |
| [Replication.md](./KeyConcepts/Replication.md) | Leader-follower, multi-master | 🟡 Medium | 🔄 Planned |
| [Consensus.md](./KeyConcepts/Consensus.md) | Paxos, Raft algorithms | 🔴 Hard | 🔄 Planned |
| [FaultTolerance.md](./KeyConcepts/FaultTolerance.md) | Redundancy, failover, chaos | 🟡 Medium | 🔄 Planned |
| [LatencyVsThroughput.md](./KeyConcepts/LatencyVsThroughput.md) | The fundamental trade-off | 🟡 Medium | 🔄 Planned |

---

### 🧩 Building Blocks
> *The primitives every large-scale system is assembled from*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [Caching.md](./BuildingBlocks/Caching.md) | In-memory caching fundamentals | 🟢 Easy | ✅ |
| [CachingStrategies.md](./BuildingBlocks/CachingStrategies.md) | Cache-aside, write-through, read-through | 🟡 Medium | ✅ |
| [LoadBalancing.md](./BuildingBlocks/LoadBalancing.md) | L4/L7, algorithms, health checks | 🟡 Medium | ✅ New |
| [CDN.md](./BuildingBlocks/CDN.md) | Content delivery, edge caching | 🟡 Medium | ✅ New |
| [RateLimiting.md](./BuildingBlocks/RateLimiting.md) | Token bucket, sliding window | 🟡 Medium | ✅ New |
| [APIGateway.md](./BuildingBlocks/APIGateway.md) | Routing, auth, rate limiting, observability | 🟡 Medium | ✅ New |
| [ServiceDiscovery.md](./BuildingBlocks/ServiceDiscovery.md) | Consul, Eureka, k8s DNS | 🟡 Medium | ✅ New |
| [CircuitBreaker.md](./BuildingBlocks/CircuitBreaker.md) | Resilience4j, Hystrix, patterns | 🟡 Medium | ✅ New |
| [Proxy_ReverseProxy.md](./BuildingBlocks/Proxy_ReverseProxy.md) | Forward/reverse proxy, Nginx, Envoy | 🟢 Easy | 🔄 Planned |
| [Blob_Storage.md](./BuildingBlocks/Blob_Storage.md) | S3, GCS, Azure Blob | 🟢 Easy | 🔄 Planned |
| [SearchIndex.md](./BuildingBlocks/SearchIndex.md) | Elasticsearch, inverted index | 🔴 Hard | 🔄 Planned |
| [MessageQueues.md](./BuildingBlocks/MessageQueues.md) | Overview: Kafka vs RabbitMQ vs SQS | 🟡 Medium | 🔄 Planned |

---

### 🗄️ Database
> *How data is stored, indexed, scaled, and queried*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [SQL_Vs_NoSQL.md](./Database/SQL_Vs_NoSQL.md) | Choosing the right database | 🟢 Easy | ✅ |
| [ACID.md](./Database/ACID.md) | Transaction guarantees | 🟢 Easy | ✅ |
| [Indexing.md](./Database/Indexing.md) | B-tree, hash, composite indexes | 🟡 Medium | ✅ |
| [Sharding.md](./Database/Sharding.md) | Horizontal partitioning strategies | 🔴 Hard | ✅ |
| [Normalization.md](./Database/Normalization.md) | 1NF → 3NF → BCNF | 🟡 Medium | ✅ |
| [ConnectionPooling.md](./Database/ConnectionPooling.md) | Pool sizing, HikariCP | 🟡 Medium | ✅ |
| [Deadlocks.md](./Database/Deadlocks.md) | Detection, prevention, resolution | 🟡 Medium | ✅ |
| [QueryOptimization.md](./Database/QueryOptimization.md) | EXPLAIN, index hints, joins | 🔴 Hard | ✅ |
| [MaterializedViews.md](./Database/MaterializedViews.md) | Pre-computed query results | 🟡 Medium | ✅ |
| [PostgreSQL_Deep_Dive.md](./Database/PostgreSQL_Deep_Dive.md) | MVCC, JSONB, partitioning | 🔴 Hard | 🔄 Planned |
| [Redis_Deep_Dive.md](./Database/Redis_Deep_Dive.md) | Data structures, persistence, clustering | 🔴 Hard | 🔄 Planned |
| [MongoDB_Deep_Dive.md](./Database/MongoDB_Deep_Dive.md) | Document model, aggregation, sharding | 🔴 Hard | 🔄 Planned |
| [Cassandra_Deep_Dive.md](./Database/Cassandra_Deep_Dive.md) | Wide-column, ring topology, tunable consistency | 🔴 Hard | 🔄 Planned |
| [Elasticsearch_Deep_Dive.md](./Database/Elasticsearch_Deep_Dive.md) | Inverted index, search, analytics | 🔴 Hard | 🔄 Planned |
| [Database_Selection_Guide.md](./Database/Database_Selection_Guide.md) | Decision tree for any use case | 🟡 Medium | 🔄 Planned |

---

### 📡 APIs
> *How services communicate with the world and each other*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [RESTful.md](./APIs/RESTful.md) | REST principles, HTTP verbs, status codes | 🟢 Easy | ✅ |
| [GraphQL.md](./APIs/GraphQL.md) | Queries, mutations, subscriptions | 🟡 Medium | ✅ |
| [gRPC.md](./APIs/gRPC.md) | Protocol Buffers, streaming, performance | 🟡 Medium | ✅ |
| [Richardson_Maturity_Model_(RMM).md](./APIs/Richardson_Maturity_Model_(RMM).md) | REST maturity levels | 🟢 Easy | ✅ |
| [WebSockets.md](./APIs/WebSockets.md) | Real-time bidirectional communication | 🟡 Medium | 🔄 Planned |
| [ServerSentEvents.md](./APIs/ServerSentEvents.md) | Server push, event streams | 🟢 Easy | 🔄 Planned |
| [Webhooks.md](./APIs/Webhooks.md) | Push-based integrations | 🟢 Easy | 🔄 Planned |
| [API_Versioning.md](./APIs/API_Versioning.md) | URI, header, query param strategies | 🟡 Medium | 🔄 Planned |

---

### 🏛️ Architectures
> *The big picture blueprints for entire systems*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [Overview.md](./Architectures/Overview.md) | Comparison of all 9 architectures | 🟡 Medium | ✅ |
| [Clean.md](./Architectures/Clean.md) | Clean Architecture by Uncle Bob | 🟡 Medium | ✅ |
| [CQRS.md](./Architectures/CQRS.md) | Command Query Responsibility Segregation | 🔴 Hard | ✅ |
| [Event_Driven.md](./Architectures/Event_Driven.md) | Pub/sub, event sourcing | 🔴 Hard | ✅ |
| [Hexagonal.md](./Architectures/Hexagonal.md) | Ports and adapters | 🟡 Medium | ✅ |
| [Layered_N-Tier.md](./Architectures/Layered_N-Tier.md) | Classic MVC/N-tier | 🟢 Easy | ✅ |
| [Microkernel.md](./Architectures/Microkernel.md) | Plugin-based architecture | 🟡 Medium | ✅ |
| [Modular.md](./Architectures/Modular.md) | Modular monolith | 🟡 Medium | ✅ |
| [Onion.md](./Architectures/Onion.md) | Domain-centric layers | 🟡 Medium | ✅ |
| [SOA.md](./Architectures/SOA.md) | Service-Oriented Architecture | 🟡 Medium | ✅ |
| [Serverless.md](./Architectures/Serverless.md) | FaaS, BaaS, Lambda patterns | 🟡 Medium | 🔄 Planned |

---

### 🔬 Microservices
> *Decomposing systems and managing the complexity that follows*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [DistributedSystem.md](./Microservices/DistributedSystem.md) | Core distributed systems concepts | 🔴 Hard | ✅ |
| [Designing_Guidline.md](./Microservices/Designing_Guidline.md) | Design principles and guidelines | 🟡 Medium | ✅ |
| [ExceptionHandling.md](./Microservices/ExceptionHandling.md) | Error propagation, retry strategies | 🟡 Medium | ✅ |
| [Securing.md](./Microservices/Securing.md) | Auth in microservices | 🔴 Hard | ✅ |
| [TestingMechanism.md](./Microservices/TestingMechanism.md) | Contract, integration, chaos testing | 🟡 Medium | ✅ |
| [Microservice_Failure_Situation_Handle.md](./Microservices/Microservice_Failure_Situation_Handle.md) | Failure handling patterns | 🔴 Hard | ✅ |
| [Slower_Microservice_Detection.md](./Microservices/Slower_Microservice_Detection.md) | Performance monitoring | 🟡 Medium | ✅ |
| [ServiceMesh.md](./Microservices/ServiceMesh.md) | Istio, Linkerd, sidecar proxy | 🔴 Hard | 🔄 Planned |
| [EventSourcing.md](./Microservices/EventSourcing.md) | Event store, projections | 🔴 Hard | 🔄 Planned |
| [Saga_Pattern_Deep_Dive.md](./Microservices/Saga_Pattern_Deep_Dive.md) | Choreography vs orchestration | 🔴 Hard | 🔄 Planned |
| [BulkheadPattern.md](./Microservices/BulkheadPattern.md) | Resource isolation | 🟡 Medium | 🔄 Planned |
| [Strangler_Fig_Pattern.md](./Microservices/Strangler_Fig_Pattern.md) | Monolith migration | 🟡 Medium | 🔄 Planned |

---

### 📨 Messaging & Event Streaming
> *Async communication, decoupling, and event-driven systems*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [Kafka.md](./MessagingQ/Kafka.md) | Topics, partitions, consumer groups | 🔴 Hard | ✅ |
| [RabbitMQ.md](./MessagingQ/RabbitMQ.md) | Exchanges, queues, routing | 🟡 Medium | 🔄 Planned |
| [AWS_SQS_SNS.md](./MessagingQ/AWS_SQS_SNS.md) | Managed queue and pub/sub | 🟡 Medium | 🔄 Planned |
| [Redis_Streams.md](./MessagingQ/Redis_Streams.md) | Lightweight event streaming | 🟡 Medium | 🔄 Planned |
| [MessageQueue_Comparison.md](./MessagingQ/MessageQueue_Comparison.md) | Kafka vs RabbitMQ vs SQS | 🟡 Medium | 🔄 Planned |

---

### 🔐 Security
> *Protecting systems, data, and users*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [OAuth2.md](./Security/OAuth2.md) | Authorization framework | 🟡 Medium | ✅ |
| [JWT_Deep_Dive.md](./Security/JWT_Deep_Dive.md) | Tokens, claims, signing | 🟡 Medium | 🔄 Planned |
| [TLS_SSL_HTTPS.md](./Security/TLS_SSL_HTTPS.md) | Certificate lifecycle, TLS handshake | 🟡 Medium | 🔄 Planned |
| [Authentication_vs_Authorization.md](./Security/Authentication_vs_Authorization.md) | Identity vs permission | 🟢 Easy | 🔄 Planned |
| [OWASP_Top10.md](./Security/OWASP_Top10.md) | Most critical web vulnerabilities | 🟡 Medium | 🔄 Planned |
| [ZeroTrust_Architecture.md](./Security/ZeroTrust_Architecture.md) | Never trust, always verify | 🔴 Hard | 🔄 Planned |
| [mTLS.md](./Security/mTLS.md) | Mutual TLS for service-to-service | 🔴 Hard | 🔄 Planned |
| [Secrets_Management.md](./Security/Secrets_Management.md) | Vault, AWS Secrets Manager | 🟡 Medium | 🔄 Planned |
| [OWASP_Top10.md](./Security/OWASP_Top10.md) | API security best practices | 🟡 Medium | 🔄 Planned |

---

### 👁️ Observability
> *If you can't measure it, you can't improve it*

| File | Topic | Difficulty | Status |
|---|---|---|---|
| [README.md](./Observability/README.md) | Three Pillars overview | 🟢 Easy | 🔄 Planned |
| [Logging_Best_Practices.md](./Observability/Logging_Best_Practices.md) | Structured logging, ELK, Loki | 🟡 Medium | 🔄 Planned |
| [Metrics_Monitoring.md](./Observability/Metrics_Monitoring.md) | Prometheus, Grafana, Datadog | 🟡 Medium | 🔄 Planned |
| [Distributed_Tracing.md](./Observability/Distributed_Tracing.md) | Jaeger, Zipkin, OpenTelemetry | 🔴 Hard | 🔄 Planned |
| [Alerting_SLO_SLA_SLI.md](./Observability/Alerting_SLO_SLA_SLI.md) | SLI/SLO/SLA/Error Budgets | 🟡 Medium | 🔄 Planned |

---

### 🎨 System Design Case Studies
> *End-to-end system design — the most-asked interview questions*

| File | System | Key Concepts | Difficulty | Status |
|---|---|---|---|---|
| [How_To_Approach.md](./SystemDesignCaseStudies/How_To_Approach_System_Design.md) | Framework & methodology | RADIO framework | 🟡 Medium | 🔄 Planned |
| [DesignURLShortener.md](./SystemDesignCaseStudies/DesignURLShortener.md) | URL Shortener (bit.ly) | Hashing, caching, DB choice | 🟡 Medium | 🔄 Planned |
| [DesignTwitter.md](./SystemDesignCaseStudies/DesignTwitter.md) | Twitter/X | Fan-out, timelines, search | 🔴 Hard | 🔄 Planned |
| [DesignNetflix.md](./SystemDesignCaseStudies/DesignNetflix.md) | Netflix | CDN, encoding, availability | 🔴 Hard | 🔄 Planned |
| [DesignUber.md](./SystemDesignCaseStudies/DesignUber.md) | Uber/Lyft | Geospatial, real-time, pricing | 🔴 Hard | 🔄 Planned |
| [DesignWhatsApp.md](./SystemDesignCaseStudies/DesignWhatsApp.md) | WhatsApp | WebSockets, message guarantees | 🔴 Hard | 🔄 Planned |
| [DesignInstagram.md](./SystemDesignCaseStudies/DesignInstagram.md) | Instagram | Feed, media storage, graph | 🔴 Hard | 🔄 Planned |
| [DesignYouTube.md](./SystemDesignCaseStudies/DesignYouTube.md) | YouTube | Video pipeline, CDN | 🔴 Hard | 🔄 Planned |
| [DesignNotificationSystem.md](./SystemDesignCaseStudies/DesignNotificationSystem.md) | Notification System | Fan-out, delivery guarantees | 🟡 Medium | 🔄 Planned |
| [DesignPaymentSystem.md](./SystemDesignCaseStudies/DesignPaymentSystem.md) | Payment System | ACID, idempotency, ledger | 🔴 Hard | 🔄 Planned |
| [DesignRateLimiter.md](./SystemDesignCaseStudies/DesignRateLimiter.md) | Rate Limiter | Algorithms, distributed coordination | 🟡 Medium | 🔄 Planned |

---

### 📐 Data Structures ✅
> *The DNA of efficient algorithms*

| Section | Topics | Status |
|---|---|---|
| [Arrays/](./DataStructures/Arrays/) | Fundamentals, Two Pointers, Sliding Window | ✅ |
| [LinkedList/](./DataStructures/LinkedList/) | Fast/Slow pointers, reversal, merge | ✅ |
| [Trees/](./DataStructures/Trees/) | Binary Tree, BST, traversals | ✅ |
| [Graphs/](./DataStructures/Graphs/) | BFS, DFS, shortest path | ✅ |
| [Heaps/](./DataStructures/Heaps/) | Priority queues, Top K | ✅ |
| [HashTables/](./DataStructures/HashTables/) | Collision handling, patterns | ✅ |
| [StacksAndQueues/](./DataStructures/StacksAndQueues/) | LIFO/FIFO, monotonic stack | ✅ |

👉 **[Full Data Structures Guide →](./DataStructures/README.md)**

---

### 🧮 Algorithms ✅
> *Patterns and techniques that unlock any coding problem*

| Section | Topics | Status |
|---|---|---|
| [Sorting/](./Algorithms/Sorting/) | Bubble → Radix, O(n log n) mastery | ✅ |
| [Searching/](./Algorithms/Searching/) | Binary search & all variations | ✅ |
| [Recursion/](./Algorithms/Recursion/) | N-Queens, Sudoku, backtracking | ✅ |
| [DynamicProgramming/](./Algorithms/DynamicProgramming/) | 10 core DP patterns | ✅ |
| [Greedy/](./Algorithms/Greedy/) | Activity selection, Huffman, MST | ✅ |
| [GraphAlgorithms/](./Algorithms/GraphAlgorithms/) | Dijkstra, Bellman-Ford, topological sort | ✅ |
| [StringAlgorithms/](./Algorithms/StringAlgorithms/) | KMP, Rabin-Karp, Z-algorithm | ✅ |
| [BitManipulation/](./Algorithms/BitManipulation/) | XOR tricks, bit masking | ✅ |

👉 **[Full Algorithms Guide →](./Algorithms/README.md)**

---

### 🎭 Design Patterns ✅
> *Proven recipes for recurring software problems*

| Category | Patterns | Status |
|---|---|---|
| [Creational/](./DesignPattern/Creational/) | Singleton, Factory, Builder, Prototype, Abstract Factory | ✅ |
| [Structural/](./DesignPattern/Structural/) | Adapter, Decorator, Facade, Proxy, Composite, Bridge, Flyweight | ✅ |
| [Behavioral/](./DesignPattern/Behavioral/) | Strategy, Observer, Command, Template, State, Chain, Iterator, Visitor, Mediator, Memento, Interpreter | ✅ |

👉 **[Full Design Patterns Guide →](./DesignPattern/README.md)**

---

### 🌱 Principles
> *The foundational laws of good software*

| File | Topic | Status |
|---|---|---|
| [SOLID.md](./Principles/SOLID.md) | 5 principles of OO design | ✅ |
| [OOP.md](./Principles/OOP.md) | Encapsulation, inheritance, polymorphism | ✅ |
| [DRY_KISS_YAGNI.md](./Principles/DRY_KISS_YAGNI.md) | Don't Repeat Yourself, Keep It Simple | ✅ |
| [CQRS.md](./Principles/CQRS.md) | Command Query Responsibility Segregation | ✅ |
| [SAGA.md](./Principles/SAGA.md) | Distributed transaction management | ✅ |
| [DomainDrivenDesign.md](./Principles/DomainDrivenDesign.md) | DDD bounded contexts, aggregates | 🔄 Planned |
| [Clean_Code_Principles.md](./Principles/Clean_Code_Principles.md) | CUPID, Clean Code rules | 🔄 Planned |
| [TheoriesAndLaws.md](./Principles/TheoriesAndLaws.md) | Conway's Law, Amdahl's Law, PACELC | 🔄 Planned |

---

### 🛠️ DevOps ✅
> *Build, ship, run — the modern engineering lifecycle*

| Section | Topics | Status |
|---|---|---|
| [Docker/](./DevOps/Docker/) | Fundamentals, Networking, Volumes, Compose | ✅ |
| [Kubernetes/](./DevOps/Kubernetes/) | Pods, Deployments, Services, Networking | ✅ |
| [CI-CD/](./DevOps/CI-CD/) | GitHub Actions, GitLab CI, Jenkins | ✅ |
| [Git/](./DevOps/Git/) | Branching, rebasing, workflows | ✅ |
| [LinuxCommands/](./DevOps/LinuxCommands/) | Essential commands cheatsheet | ✅ |

---

### ☕ Java & Spring Boot
> *Deep dives for Java engineers*

| Section | Topics | Status |
|---|---|---|
| [SpringBoot/](./SpringBoot/) | AOP, DI/IoC, Hibernate, Batch, Security | ✅ |
| [javaTpoints/](./javaTpoints/) | Collections, Streams, Threads, Records, Sealed | ✅ |

---

### ☁️ Cloud
> *AWS, GCP, and cloud architecture patterns*

| File | Topic | Status |
|---|---|---|
| [AWS/Core_Services.md](./Cloud/AWS/Core_Services.md) | EC2, S3, RDS, Lambda overview | 🔄 Planned |
| [AWS/Compute_EC2_Lambda.md](./Cloud/AWS/Compute_EC2_Lambda.md) | Compute services deep dive | 🔄 Planned |
| [Cloud_Comparison.md](./Cloud/Cloud_Comparison.md) | AWS vs GCP vs Azure | 🔄 Planned |

---

### 🧪 Testing
> *Quality assurance from unit to production*

| File | Topic | Status |
|---|---|---|
| [Testing_Pyramid.md](./Testing/Testing_Pyramid.md) | Unit → Integration → E2E | 🔄 Planned |
| [Contract_Testing.md](./Testing/Contract_Testing.md) | Pact, consumer-driven contracts | 🔄 Planned |
| [TDD_BDD.md](./Testing/TDD_BDD.md) | Test-first development | 🔄 Planned |

---

### ⚡ Performance
> *Making systems fast and keeping them fast*

| File | Topic | Status |
|---|---|---|
| [JVM_Tuning.md](./Performance/JVM_Tuning.md) | GC, heap, thread tuning | 🔄 Planned |
| [Profiling_and_Optimization.md](./Performance/Profiling_and_Optimization.md) | Flamegraphs, bottleneck analysis | 🔄 Planned |

---

### 🎤 Behavioral
> *Interviews are half technical, half human — be ready for both*

| File | Topic | Status |
|---|---|---|
| [Behavioral.md](./Behavioral/Behavioral.md) | STAR method Q&A — 10 questions | ✅ |
| [Behavioral2.md](./Behavioral/Behavioral2.md) | Additional behavioral scenarios | ✅ |
| [SelfIntroduction.md](./Behavioral/SelfIntroduction.md) | How to introduce yourself | ✅ |
| [HandleHighVolumeEnv.md](./Behavioral/HandleHighVolumeEnv.md) | Handling high-pressure situations | ✅ |

---

### 🏆 Interview Prep
> *Structured plans to get interview-ready*

| File | Topic | Status |
|---|---|---|
| [30_Day_Plan.md](./InterviewPrep/30_Day_Plan.md) | Intensive 30-day sprint | 🔄 Planned |
| [60_Day_Plan.md](./InterviewPrep/60_Day_Plan.md) | Thorough 60-day prep | 🔄 Planned |
| [90_Day_Plan.md](./InterviewPrep/90_Day_Plan.md) | Complete 90-day mastery | 🔄 Planned |
| [FAANG_Guide.md](./InterviewPrep/Company_Specific/FAANG_Guide.md) | FAANG-specific preparation | 🔄 Planned |
| [How_To_Crack_System_Design.md](./InterviewPrep/How_To_Crack_System_Design.md) | System design interview framework | 🔄 Planned |

---

## 📊 Overall Progress Dashboard

```
PHASE 1 — Foundation                          ████████░░ 75%
  ✅ INDEX.md (this file)
  ✅ BuildingBlocks: LoadBalancing, CDN, RateLimiting, APIGateway
  ✅ BuildingBlocks: ServiceDiscovery, CircuitBreaker
  🔄 ROADMAPS.md
  🔄 Standardize existing READMEs

PHASE 2 — Core Expansion                      ░░░░░░░░░░  0%
  🔄 SystemDesignCaseStudies/ domain (13 files)
  🔄 Security expansion (9 more files)
  🔄 Observability/ domain (7 files)

PHASE 3 — Database & Cloud                    ░░░░░░░░░░  0%
  🔄 NoSQL deep dives (Redis, MongoDB, Cassandra, ES)
  🔄 Cloud/AWS/ section
  🔄 KeyConcepts expansion (4 more)
  🔄 MessagingQ expansion (4 more)

PHASE 4 — Refinement & Polish                 ░░░░░░░░░░  0%
  🔄 Cross-references on all articles
  🔄 Testing/ domain
  🔄 InterviewPrep/ plans

PHASE 5 — Advanced Content                    ░░░░░░░░░░  0%
  🔄 Performance/ domain
  🔄 Advanced case studies
  🔄 DDD, Clean Code, Laws

OVERALL: ██░░░░░░░░ ~15% (Phase 1 in progress)
```

---

## 🏷️ Topic Tags

Find content by concept:

| Tag | Topics |
|---|---|
| `#scaling` | [Scalability](./KeyConcepts/Scalability.md), [Sharding](./Database/Sharding.md), [Load Balancing](./BuildingBlocks/LoadBalancing.md), [CDN](./BuildingBlocks/CDN.md), [Consistent Hashing](./KeyConcepts/Consistent_Hashing.md) |
| `#caching` | [Caching](./BuildingBlocks/Caching.md), [Caching Strategies](./BuildingBlocks/CachingStrategies.md), [CDN](./BuildingBlocks/CDN.md), [Redis](./Database/Redis_Deep_Dive.md) |
| `#database` | [SQL vs NoSQL](./Database/SQL_Vs_NoSQL.md), [Indexing](./Database/Indexing.md), [ACID](./Database/ACID.md), [Sharding](./Database/Sharding.md) |
| `#security` | [OAuth2](./Security/OAuth2.md), [JWT](./Security/JWT_Deep_Dive.md), [TLS](./Security/TLS_SSL_HTTPS.md), [OWASP](./Security/OWASP_Top10.md) |
| `#microservices` | [Distributed System](./Microservices/DistributedSystem.md), [Circuit Breaker](./BuildingBlocks/CircuitBreaker.md), [Service Discovery](./BuildingBlocks/ServiceDiscovery.md), [Kafka](./MessagingQ/Kafka.md) |
| `#interview` | [System Design Cases](./SystemDesignCaseStudies/), [Behavioral](./Behavioral/), [CAP Theorem](./KeyConcepts/CAPTheorem.md) |
| `#algorithms` | [Algorithms/](./Algorithms/), [DataStructures/](./DataStructures/), [ProblemSets](./Algorithms/ProblemSets/) |
| `#devops` | [Docker](./DevOps/Docker/), [Kubernetes](./DevOps/Kubernetes/), [CI-CD](./DevOps/CI-CD/) |
| `#java` | [SpringBoot/](./SpringBoot/), [javaTpoints/](./javaTpoints/), [JVM Tuning](./Performance/JVM_Tuning.md) |
| `#observability` | [Logging](./Observability/Logging_Best_Practices.md), [Metrics](./Observability/Metrics_Monitoring.md), [Tracing](./Observability/Distributed_Tracing.md) |

---

## 🆕 Recently Added

| Date | File | Description |
|---|---|---|
| May 2026 | [INDEX.md](./INDEX.md) | This master navigation hub |
| May 2026 | [ROADMAPS.md](./ROADMAPS.md) | All learning tracks |
| May 2026 | [BuildingBlocks/LoadBalancing.md](./BuildingBlocks/LoadBalancing.md) | L4/L7, algorithms, production config |
| May 2026 | [BuildingBlocks/CDN.md](./BuildingBlocks/CDN.md) | Edge caching, push vs pull, CloudFront |
| May 2026 | [BuildingBlocks/RateLimiting.md](./BuildingBlocks/RateLimiting.md) | Token bucket, sliding window, distributed |
| May 2026 | [BuildingBlocks/APIGateway.md](./BuildingBlocks/APIGateway.md) | Routing, auth, observability, Kong/AWS |
| May 2026 | [BuildingBlocks/ServiceDiscovery.md](./BuildingBlocks/ServiceDiscovery.md) | Consul, Eureka, k8s DNS |
| May 2026 | [BuildingBlocks/CircuitBreaker.md](./BuildingBlocks/CircuitBreaker.md) | Resilience4j, states, bulkhead |
| May 2026 | [ENHANCEMENT_ROADMAP.md](./ENHANCEMENT_ROADMAP.md) | Full repo enhancement plan |

---

## 🔗 Useful Links

- 📌 [Enhancement Roadmap](./ENHANCEMENT_ROADMAP.md) — Full plan for what's being built
- 🗺️ [All Learning Tracks](./ROADMAPS.md) — Week-by-week study plans
- 📋 [Preparation Notes](./PreparationNotes.md) — Quick reference notes

---

*Last Updated: May 2026 | [Report an Issue](https://github.com) | [Enhancement Roadmap](./ENHANCEMENT_ROADMAP.md)*
