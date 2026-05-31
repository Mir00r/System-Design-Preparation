# 🗺️ Learning Roadmaps: Your Personalized Path to Mastery

> **"A goal without a plan is just a wish. Pick your track. Follow the map. Land the role."**

---

## 📋 Table of Contents

1. [How to Use These Roadmaps](#-how-to-use-these-roadmaps)
2. [Track 1 — Junior Engineer](#-track-1--junior-engineer-0-2-years)
3. [Track 2 — Mid-Level Engineer](#-track-2--mid-level-engineer-2-5-years)
4. [Track 3 — Senior / FAANG-Targeting](#-track-3--senior-engineer-5-years-faang-targeting)
5. [Track 4 — DevOps / Platform Engineer](#-track-4--devopsplatform-engineer)
6. [Track 5 — CS Student / Bootcamp Grad](#-track-5--cs-student--bootcamp-grad)
7. [Domain-First Tracks](#-domain-first-tracks)
8. [Quick Reference: Topic by Interview Round](#-quick-reference-topic-by-interview-round)

> **New sections added**: Tradeoffs Mastery, AI/ML for Engineers, Company Interview Prep tracks — see [Domain-First Tracks](#-domain-first-tracks)

---

## 🧭 How to Use These Roadmaps

```
STEP 1: Pick your track based on experience + goal
STEP 2: Bookmark this file — check off each week as you complete it
STEP 3: For each topic, read the article → do the challenge → move on
STEP 4: After every 2 weeks, do a mock interview or solve 5 problems
STEP 5: Adjust your pace — these are guides, not strict deadlines
```

> 💡 **Key Insight**: Consistency beats intensity. 1 hour daily beats 8 hours on weekends.

---

## 🌱 Track 1 — Junior Engineer (0-2 years)

**Goal**: Land your first engineering role or get promoted to mid-level  
**Duration**: 16 weeks (4 months)  
**Daily Commitment**: 1.5-2 hours  

```
COMPLEXITY:  ●●○○○  (2/5)
BREADTH:     ●●●●○  (4/5 — cover many topics)
DEPTH:       ●●○○○  (2/5 — focus on understanding)
```

### Week-by-Week Plan

| Week | Domain | Topics | Files to Read |
|---|---|---|---|
| **1** | 🌐 How Internet Works | DNS, HTTP, Client-Server, IP, TCP/UDP | [PreparationNotes.md](./PreparationNotes.md) §1-5 |
| **2** | 🌐 Foundations | REST APIs, HTTP methods, status codes | [APIs/RESTful.md](./APIs/RESTful.md) |
| **3** | 📐 Data Structures I | Arrays, ArrayList vs Arrays | [DataStructures/Arrays/01_Arrays_Fundamentals.md](./DataStructures/Arrays/01_Arrays_Fundamentals.md) |
| **4** | 📐 Data Structures II | LinkedList, Stacks, Queues | [DataStructures/LinkedList/](./DataStructures/LinkedList/) |
| **5** | 📐 Data Structures III | Trees — Binary Tree, BST | [DataStructures/Trees/01_BinaryTree_Fundamentals.md](./DataStructures/Trees/01_BinaryTree_Fundamentals.md) |
| **6** | 🧮 Algorithms I | Big-O, Sorting (Bubble, Merge, Quick) | [Algorithms/Sorting/](./Algorithms/Sorting/) |
| **7** | 🧮 Algorithms II | Binary Search, Two Pointers | [Algorithms/Searching/](./Algorithms/Searching/) |
| **8** | 🎭 Design Patterns I | Singleton, Factory, Builder | [DesignPattern/Creational/](./DesignPattern/Creational/) |
| **9** | 🎭 Design Patterns II | Strategy, Observer, Decorator | [DesignPattern/Behavioral/Strategy.md](./DesignPattern/Behavioral/Strategy.md) |
| **10** | 🗄️ Databases | SQL vs NoSQL, ACID, basic indexing | [Database/SQL_Vs_NoSQL.md](./Database/SQL_Vs_NoSQL.md) |
| **11** | 🌱 Principles | OOP, SOLID, DRY/KISS/YAGNI | [Principles/SOLID.md](./Principles/SOLID.md) |
| **12** | 🔑 Key Concepts | Scalability, Caching basics, CAP theorem intro | [KeyConcepts/Scalability.md](./KeyConcepts/Scalability.md) |
| **13** | 🛠️ DevOps I | Docker fundamentals, containers | [DevOps/Docker/Docker_Fundamentals.md](./DevOps/Docker/Docker_Fundamentals.md) |
| **14** | 🛠️ DevOps II | Git workflows, CI/CD basics | [DevOps/CI-CD/CI_CD_Fundamentals.md](./DevOps/CI-CD/CI_CD_Fundamentals.md) |
| **15** | 🎤 Behavioral | STAR method, self-introduction | [Behavioral/SelfIntroduction.md](./Behavioral/SelfIntroduction.md) |
| **16** | 🎤 Mock + Review | Solve 20 LeetCode Easy problems, 2 mock interviews | [DataStructures/ProblemSets/](./DataStructures/ProblemSets/) |

### 🏆 Milestone: Junior-Ready Checklist
```
✅ Can explain what happens when you type "google.com" in a browser
✅ Can implement arrays, linked lists, trees from scratch
✅ Knows Singleton, Factory, Observer patterns by heart
✅ Can write a CRUD REST API and explain its design choices
✅ Can pass most LeetCode Easy problems in 20 minutes
✅ Has a confident 2-minute self-introduction
```

---

## ⚡ Track 2 — Mid-Level Engineer (2-5 years)

**Goal**: Senior role, FAANG mid-level (E4/E5), or staff promotion  
**Duration**: 12 weeks (3 months)  
**Daily Commitment**: 2 hours  

```
COMPLEXITY:  ●●●○○  (3/5)
BREADTH:     ●●●●●  (5/5 — must know everything)
DEPTH:       ●●●○○  (3/5 — understand trade-offs)
```

### Week-by-Week Plan

| Week | Domain | Topics | Files to Read |
|---|---|---|---|
| **1** | 🔑 Core Theory | CAP Theorem, ACID, Availability, Consistent Hashing | [KeyConcepts/CAPTheorem.md](./KeyConcepts/CAPTheorem.md) |
| **2** | 🧩 Building Blocks I | Caching (strategies), Load Balancing, CDN | [BuildingBlocks/LoadBalancing.md](./BuildingBlocks/LoadBalancing.md), [BuildingBlocks/CDN.md](./BuildingBlocks/CDN.md) |
| **3** | 🧩 Building Blocks II | Rate Limiting, API Gateway, Service Discovery | [BuildingBlocks/RateLimiting.md](./BuildingBlocks/RateLimiting.md), [BuildingBlocks/APIGateway.md](./BuildingBlocks/APIGateway.md) |
| **4** | 🗄️ Database Deep Dive | Sharding, Indexing deep dive, Connection Pooling | [Database/Sharding.md](./Database/Sharding.md), [Database/Indexing.md](./Database/Indexing.md) |
| **5** | 🔬 Microservices | Distributed systems, Circuit Breaker, Saga Pattern | [Microservices/DistributedSystem.md](./Microservices/DistributedSystem.md), [BuildingBlocks/CircuitBreaker.md](./BuildingBlocks/CircuitBreaker.md) |
| **6** | 🎨 System Design I | URL Shortener (practice full design) | [SystemDesignCaseStudies/DesignURLShortener.md](./SystemDesignCaseStudies/DesignURLShortener.md) |
| **7** | 🎨 System Design II | Design Twitter (fan-out, timelines) | [SystemDesignCaseStudies/DesignTwitter.md](./SystemDesignCaseStudies/DesignTwitter.md) |
| **8** | 🔐 Security | OAuth2, JWT, HTTPS/TLS, Auth vs AuthZ | [Security/OAuth2.md](./Security/OAuth2.md), [Security/JWT_Deep_Dive.md](./Security/JWT_Deep_Dive.md) |
| **9** | 📨 Messaging | Kafka, RabbitMQ, async patterns | [MessagingQ/Kafka.md](./MessagingQ/Kafka.md) |
| **10** | 🎨 System Design III | Netflix + Uber (complex systems) | [SystemDesignCaseStudies/DesignNetflix.md](./SystemDesignCaseStudies/DesignNetflix.md) |
| **11** | 🧮 Algorithms Review | DP, Graphs, advanced patterns | [Algorithms/DynamicProgramming/](./Algorithms/DynamicProgramming/) |
| **12** | 🏆 Mock Interviews | 3 full system design mocks, 30 LeetCode Mediums | [InterviewPrep/Mock_Interviews.md](./InterviewPrep/Mock_Interviews.md) |

### 🏆 Milestone: Mid-Level Senior-Ready Checklist
```
✅ Can design a URL Shortener in 45 minutes without prompting
✅ Can explain CAP theorem and give a real example in 60 seconds
✅ Knows when to use Kafka vs RabbitMQ vs SQS and WHY
✅ Can identify the correct database for any use case
✅ Understands OAuth2 + JWT flow end-to-end
✅ Can implement a rate limiter (token bucket) from scratch
✅ Passes LeetCode Medium problems consistently in 30-40 minutes
```

---

## 🚀 Track 3 — Senior Engineer (5+ years, FAANG-Targeting)

**Goal**: Staff/Principal engineer, FAANG E6+, or senior system architect  
**Duration**: 8 weeks (intensive)  
**Daily Commitment**: 2-3 hours  

```
COMPLEXITY:  ●●●●●  (5/5)
BREADTH:     ●●●●●  (5/5)
DEPTH:       ●●●●●  (5/5 — master every trade-off)
```

### Week-by-Week Plan

| Week | Domain | Focus | Files |
|---|---|---|---|
| **1** | 🏛️ Architecture Mastery | All 9 patterns + when/why to choose each | [Architectures/Overview.md](./Architectures/Overview.md) + all individual files |
| **2** | 🔬 Distributed Systems Theory | Consensus (Paxos/Raft), Replication, Fault Tolerance | [KeyConcepts/Consensus.md](./KeyConcepts/Consensus.md), [KeyConcepts/Replication.md](./KeyConcepts/Replication.md) |
| **3** | 🎨 Hard System Designs | Google Search + Payment System | [SystemDesignCaseStudies/DesignGoogleSearch.md](./SystemDesignCaseStudies/DesignGoogleSearch.md), [SystemDesignCaseStudies/DesignPaymentSystem.md](./SystemDesignCaseStudies/DesignPaymentSystem.md) |
| **4** | 🔐 Advanced Security | Zero Trust, mTLS, Secrets Management, OWASP | [Security/ZeroTrust_Architecture.md](./Security/ZeroTrust_Architecture.md), [Security/mTLS.md](./Security/mTLS.md) |
| **5** | ☁️ Cloud Architecture | AWS Well-Architected, multi-region design | [Cloud/AWS/AWS_Well_Architected.md](./Cloud/AWS/AWS_Well_Architected.md) |
| **6** | ⚡ Performance Engineering | JVM tuning, profiling, database performance | [Performance/JVM_Tuning.md](./Performance/JVM_Tuning.md), [Performance/Profiling_and_Optimization.md](./Performance/Profiling_and_Optimization.md) |
| **7** | 🔬 Advanced Microservices | Service Mesh (Istio), Event Sourcing, Strangler Fig | [Microservices/ServiceMesh.md](./Microservices/ServiceMesh.md), [Microservices/EventSourcing.md](./Microservices/EventSourcing.md) |
| **8** | 🎤 Senior Behavioral | Leadership decisions, conflict, technical influence | [Behavioral/Behavioral.md](./Behavioral/Behavioral.md), [Behavioral/Behavioral2.md](./Behavioral/Behavioral2.md) |

### 🏆 Milestone: Senior/Staff-Ready Checklist
```
✅ Can design ANY of the 12 case studies from first principles in 45 min
✅ Can explain Raft consensus algorithm with a whiteboard diagram
✅ Can argue trade-offs between Kafka and Kinesis for a specific use case
✅ Can describe a Zero Trust migration plan for a microservices org
✅ Has opinions on Conway's Law and can give a real example
✅ Can explain JVM G1GC pauses and how to tune them
✅ Can solve LeetCode Hard problems with optimal solutions
✅ Can lead a technical interview and evaluate candidates
```

---

## 🛠️ Track 4 — DevOps / Platform Engineer

**Goal**: Senior DevOps, SRE, or Platform Engineering role  
**Duration**: 10 weeks  
**Daily Commitment**: 1.5-2 hours  

```
COMPLEXITY:  ●●●●○  (4/5)
BREADTH:     ●●●●○  (4/5)
DEPTH:       ●●●●○  (4/5 — deep on infra topics)
```

### Week-by-Week Plan

| Week | Domain | Topics | Files |
|---|---|---|---|
| **1** | 🐳 Docker Mastery | Fundamentals, Networking, Volumes, Security | [DevOps/Docker/](./DevOps/Docker/) |
| **2** | ☸️ Kubernetes I | Pods, Deployments, Services, ConfigMaps | [DevOps/Kubernetes/Kubernetes_Fundamentals.md](./DevOps/Kubernetes/Kubernetes_Fundamentals.md) |
| **3** | ☸️ Kubernetes II | Networking, Ingress, Persistent Volumes, RBAC | [DevOps/Kubernetes/Kubernetes_Services_Networking.md](./DevOps/Kubernetes/Kubernetes_Services_Networking.md) |
| **4** | 🔄 CI/CD | GitHub Actions, GitLab CI, Jenkins pipelines | [DevOps/CI-CD/](./DevOps/CI-CD/) |
| **5** | ☁️ Cloud (AWS) | EC2, S3, RDS, Lambda, VPC, IAM | [Cloud/AWS/Core_Services.md](./Cloud/AWS/Core_Services.md) |
| **6** | 👁️ Observability | Logging (ELK), Metrics (Prometheus/Grafana), Tracing | [Observability/](./Observability/) |
| **7** | 🔐 Security | TLS, mTLS, Secrets Management, RBAC | [Security/TLS_SSL_HTTPS.md](./Security/TLS_SSL_HTTPS.md), [Security/Secrets_Management.md](./Security/Secrets_Management.md) |
| **8** | 🧩 Infrastructure | Load Balancing, CDN, Service Discovery | [BuildingBlocks/LoadBalancing.md](./BuildingBlocks/LoadBalancing.md) |
| **9** | 🔬 SRE Concepts | SLI/SLO/SLA, Error Budgets, Incident Management | [Observability/Alerting_SLO_SLA_SLI.md](./Observability/Alerting_SLO_SLA_SLI.md) |
| **10** | 🏆 Mock + Review | Design a deployment pipeline for a microservices app | End-to-end exercise |

---

## 🎓 Track 5 — CS Student / Bootcamp Grad

**Goal**: First software engineering job  
**Duration**: 20 weeks (5 months)  
**Daily Commitment**: 1 hour  

```
COMPLEXITY:  ●○○○○  (1/5 — start from basics)
BREADTH:     ●●●○○  (3/5)
DEPTH:       ●○○○○  (1/5 — understand concepts)
```

### Week-by-Week Plan

| Week | Focus | Go Here |
|---|---|---|
| **1-3** | Data Structures Fundamentals | [DataStructures/00_GettingStarted.md](./DataStructures/00_GettingStarted.md) → Arrays → LinkedList |
| **4-6** | Trees + Hash Tables | [DataStructures/Trees/](./DataStructures/Trees/), [DataStructures/HashTables/](./DataStructures/HashTables/) |
| **7-9** | Algorithms Basics | [Algorithms/00_GettingStarted.md](./Algorithms/00_GettingStarted.md) → Sorting → Searching |
| **10-12** | OOP + Design Patterns | [Principles/OOP.md](./Principles/OOP.md), [DesignPattern/00_GettingStarted.md](./DesignPattern/00_GettingStarted.md) |
| **13-14** | REST APIs + HTTP | [APIs/RESTful.md](./APIs/RESTful.md) |
| **15-16** | Databases | [Database/SQL_Vs_NoSQL.md](./Database/SQL_Vs_NoSQL.md), [Database/ACID.md](./Database/ACID.md) |
| **17-18** | Docker + Git | [DevOps/Docker/Docker_Fundamentals.md](./DevOps/Docker/Docker_Fundamentals.md) |
| **19** | Behavioral Prep | [Behavioral/SelfIntroduction.md](./Behavioral/SelfIntroduction.md) |
| **20** | Mock Interviews + 20 LeetCode Easy | [DataStructures/ProblemSets/](./DataStructures/ProblemSets/) |

---

## 🎯 Domain-First Tracks

If you want to master one specific area fast:

### 🎨 System Design Interview in 2 Weeks
```
Day 1-2:   CAP Theorem + Scalability + Building Blocks overview
Day 3-4:   Load Balancing + CDN + Caching
Day 5-6:   Database selection (SQL vs NoSQL + Sharding)
Day 7:     Rate Limiting + API Gateway + Service Discovery
Day 8-9:   Design URL Shortener (full design)
Day 10-11: Design Twitter or WhatsApp
Day 12-13: Design Netflix or Uber
Day 14:    Mock interview practice
```

### 🔐 Security Mastery in 1 Week
```
Day 1: Authentication vs Authorization (basics)
Day 2: OAuth2 flow (authorization code, client credentials)
Day 3: JWT — structure, signing, validation, pitfalls
Day 4: TLS/HTTPS — handshake, certificates, HSTS
Day 5: OWASP Top 10 — injection, XSS, CSRF, etc.
Day 6: Zero Trust + mTLS (for microservices)
Day 7: Secrets Management + API security best practices
```

### 📐 LeetCode Patterns in 3 Weeks
```
Week 1: Arrays (Two Pointers, Sliding Window, Prefix Sum)
Week 2: Trees (DFS/BFS traversals, path problems)
Week 3: Graphs (BFS/DFS, shortest path, topological sort)
+ 2 problems per day throughout
```

### ☁️ AWS Essentials in 1 Week
```
Day 1: Core services overview (EC2, S3, RDS, Lambda)
Day 2: Networking (VPC, Subnets, Security Groups, Route 53)
Day 3: Compute deep dive (EC2 types, Auto Scaling, ELB)
Day 4: Storage (S3 classes, EBS, EFS, CloudFront)
Day 5: Databases (RDS, DynamoDB, ElastiCache)
Day 6: DevOps (CodePipeline, ECS, EKS)
Day 7: Security (IAM, KMS, CloudTrail) + review
```

### ⚖️ Tradeoffs Mastery in 1 Week
```
Day 1: Consistency vs Availability (CAP theorem applied)
Day 2: SQL vs NoSQL — when to pick each
Day 3: Sync vs Async — REST, queues, events
Day 4: Monolith vs Microservices — the real tradeoffs
Day 5: Latency vs Throughput — caching, batching, CDN
Day 6: Strong vs Eventual Consistency — read your Tradeoffs/ folder
Day 7: Practice — explain 3 real systems and their tradeoff choices
```
**Files**: [Tradeoffs/](./Tradeoffs/)

### 🤖 AI/ML for Engineers in 2 Weeks
```
Week 1: Foundations → ML fundamentals → Deep Learning basics
  Day 1-2: AI_Core_Concepts/Foundations/ (linear algebra, probability)
  Day 3-4: AI_Core_Concepts/MachineLearning/ (what is ML, supervised, unsupervised)
  Day 5-7: AI_Core_Concepts/DeepLearning/ (neural nets, backprop, transformers)

Week 2: Practical AI Engineering
  Day 8-9:  AI_Core_Concepts/NLP_LLMs/ (embeddings, LLMs, prompt engineering)
  Day 10-11: AI_Core_Concepts/AI_Design_Patterns/ (RAG, agents, chain-of-thought)
  Day 12-13: AI_Core_Concepts/AI_System_Design/ (ML pipelines)
  Day 14:   AI_Core_Concepts/AI_Java_Developers/ (Java AI ecosystem)
```
**Files**: [AI_Core_Concepts/README.md](./AI_Core_Concepts/README.md)

### 🏢 Company Interview Prep in 1 Week (Intercom example)
```
Day 1: 00_InterviewPrepGuide — understand format, read company research
Day 2: Solve problems 01-05 (coding: assignments, frequency, sliding window)
Day 3: Solve problems 06-09 (LLD: chat system, rate limiter, LRU cache)
Day 4: Solve problems 10-12 (system design: leaderboard, scheduling)
Day 5: 13_MinicomRoundPrep + 14_ValuesRound_BehavioralPrep
Day 6: 15_SystemDesign_DataModelling + 16_AdditionalCodingProblems
Day 7: 17_InterviewDay_CheatSheet + 18_CompanyKnowledge review
```
**Files**: [CompanyCodingProblems/Intercom/](./CompanyCodingProblems/Intercom/)

---

## 📋 Quick Reference: Topic by Interview Round

### Coding Round (DSA)
| Topic | Priority | Where |
|---|---|---|
| Arrays & Strings | 🔴 Must Know | [DataStructures/Arrays/](./DataStructures/Arrays/) |
| Trees & Binary Search | 🔴 Must Know | [DataStructures/Trees/](./DataStructures/Trees/) |
| Hash Tables | 🔴 Must Know | [DataStructures/HashTables/](./DataStructures/HashTables/) |
| Dynamic Programming | 🟡 Important | [Algorithms/DynamicProgramming/](./Algorithms/DynamicProgramming/) |
| Graphs | 🟡 Important | [DataStructures/Graphs/](./DataStructures/Graphs/) |
| Heaps / Priority Queue | 🟢 Good to Know | [DataStructures/Heaps/](./DataStructures/Heaps/) |

### System Design Round
| Topic | Priority | Where |
|---|---|---|
| Caching | 🔴 Must Know | [BuildingBlocks/Caching.md](./BuildingBlocks/Caching.md) |
| Load Balancing | 🔴 Must Know | [BuildingBlocks/LoadBalancing.md](./BuildingBlocks/LoadBalancing.md) |
| Database Choice | 🔴 Must Know | [Database/SQL_Vs_NoSQL.md](./Database/SQL_Vs_NoSQL.md) |
| CAP Theorem | 🔴 Must Know | [KeyConcepts/CAPTheorem.md](./KeyConcepts/CAPTheorem.md) |
| Sharding | 🟡 Important | [Database/Sharding.md](./Database/Sharding.md) |
| Consistent Hashing | 🟡 Important | [KeyConcepts/Consistent_Hashing.md](./KeyConcepts/Consistent_Hashing.md) |
| Message Queues | 🟡 Important | [MessagingQ/Kafka.md](./MessagingQ/Kafka.md) |
| CDN | 🟡 Important | [BuildingBlocks/CDN.md](./BuildingBlocks/CDN.md) |
| Rate Limiting | 🟡 Important | [BuildingBlocks/RateLimiting.md](./BuildingBlocks/RateLimiting.md) |
| API Gateway | 🟢 Good to Know | [BuildingBlocks/APIGateway.md](./BuildingBlocks/APIGateway.md) |

### Behavioral Round
| Situation | Where |
|---|---|
| Leadership & conflict | [Behavioral/Behavioral.md](./Behavioral/Behavioral.md) |
| Technical challenges | [Behavioral/ApproachComplexTechnicalChallenges.md](./Behavioral/ApproachComplexTechnicalChallenges.md) |
| Prioritization | [Behavioral/PrioritiseYourWork.md](./Behavioral/PrioritiseYourWork.md) |
| Self-introduction | [Behavioral/SelfIntroduction.md](./Behavioral/SelfIntroduction.md) |

---

> 💡 **Pro Tip**: Don't just read — **do**. After every article, solve one related problem or draw the architecture on paper. Passive reading gives you 20% retention. Active practice gives you 80%.

---

*[← Back to Index](./INDEX.md) | [Enhancement Roadmap](./ENHANCEMENT_ROADMAP.md)*
