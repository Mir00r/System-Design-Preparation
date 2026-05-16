# 🟠 AWS Core Services: The Building Blocks of Amazon Web Services

> *"AWS offers 200+ services, but 80% of system designs use the same 15 core services. Master these, and you can architect almost anything."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Cloud README](../README.md)

---

## 📋 Table of Contents
1. [AWS Service Categories](#-service-categories)
2. [The 15 Must-Know Services](#-the-15-must-know-services)
3. [Service Selection Guide](#-service-selection-guide)
4. [Architecture Patterns on AWS](#-architecture-patterns)
5. [Interview Q&A](#-interview-qa)

---

## 🗂️ Service Categories

```
┌─────────────────────────────────────────────────────────────────┐
│                   AWS SERVICE MAP                                │
├─────────────────────────────────────────────────────────────────┤
│ COMPUTE          │ EC2, Lambda, ECS, Fargate, EKS              │
│ STORAGE          │ S3, EBS, EFS, Glacier                       │
│ DATABASE         │ RDS, DynamoDB, ElastiCache, Aurora           │
│ NETWORKING       │ VPC, Route53, CloudFront, ALB, API Gateway  │
│ MESSAGING        │ SQS, SNS, EventBridge, Kinesis              │
│ SECURITY         │ IAM, KMS, Secrets Manager, WAF              │
│ MONITORING       │ CloudWatch, X-Ray, CloudTrail               │
│ DEPLOYMENT       │ CodePipeline, CloudFormation, CDK           │
│ ML/AI            │ SageMaker, Rekognition, Comprehend          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏆 The 15 Must-Know Services

```
COMPUTE:
  EC2     — Virtual machines (any size, any OS, full control)
  Lambda  — Serverless functions (run code without managing servers)
  ECS     — Docker container orchestration (AWS-native)
  EKS     — Kubernetes on AWS (industry-standard containers)

STORAGE:
  S3      — Object storage (unlimited, 11 9s durability)
  EBS     — Block storage for EC2 (like a hard drive)
  
DATABASE:
  RDS     — Managed relational DB (PostgreSQL, MySQL, Oracle)
  DynamoDB— Serverless NoSQL (single-digit ms at any scale)
  ElastiCache — Managed Redis/Memcached (in-memory caching)

NETWORKING:
  VPC     — Virtual private network (isolation, subnets, routing)
  ALB     — Application Load Balancer (HTTP routing, health checks)
  Route53 — DNS service (domain routing, health-based failover)
  CloudFront — CDN (cache content at edge locations globally)

MESSAGING:
  SQS     — Message queue (decouple services, at-least-once delivery)
  SNS     — Pub/sub notifications (fan-out to multiple subscribers)
```

---

## 🧭 Service Selection Guide

```
"I need to run..." → Service Choice:

  A web application           → EC2 (control) / ECS (containers) / Beanstalk (PaaS)
  A background job on events  → Lambda (serverless, event-driven)
  A container workload        → ECS Fargate (serverless containers)
  Kubernetes                  → EKS
  
"I need to store..." → Service Choice:

  Files, images, backups      → S3
  Structured relational data  → RDS (PostgreSQL/MySQL) or Aurora
  Key-value at massive scale  → DynamoDB
  Session data / caching      → ElastiCache (Redis)
  Search                      → OpenSearch (Elasticsearch)
  
"I need to communicate..." → Service Choice:

  Async decoupling            → SQS (point-to-point queue)
  Fan-out to many subscribers → SNS (pub/sub)
  Real-time streaming         → Kinesis (ordered, replay)
  Event routing               → EventBridge (rules-based routing)
  
"I need to secure..." → Service Choice:

  Who can access what         → IAM (users, roles, policies)
  Encrypt data                → KMS (key management)
  Store secrets               → Secrets Manager / Parameter Store
  Protect web apps            → WAF (firewall) + Shield (DDoS)
```

---

## 🏗️ Architecture Patterns

```
PATTERN 1: Three-Tier Web App
  CloudFront → ALB → EC2 (Auto Scaling Group) → RDS (Multi-AZ)
  + S3 for static assets
  + ElastiCache for session/caching

PATTERN 2: Event-Driven Microservices
  API Gateway → Lambda → DynamoDB
  Lambda → SQS → Lambda (async processing)
  SNS for cross-service events

PATTERN 3: Container-Based Microservices
  ALB → ECS Fargate (multiple services)
  Each service → own RDS/DynamoDB
  Service mesh with App Mesh
  
PATTERN 4: Real-Time Data Pipeline
  Kinesis Data Streams → Lambda → DynamoDB
  Kinesis Firehose → S3 → Athena (analytics)
```

---

## ⚠️ Common Pitfalls

1. **Using EC2 when Lambda suffices** — If your workload is event-driven, runs < 15 minutes, and handles < 10GB memory, Lambda is cheaper and simpler. Don't manage servers when you don't need to.

2. **Public S3 buckets** — S3 defaults to private now, but misconfigured bucket policies have caused massive data leaks. Always enable S3 Block Public Access at the account level.

3. **No multi-AZ for databases** — Single-AZ RDS means an AZ failure = database downtime. Always use Multi-AZ for production databases (automatic failover).

---

## 📝 Interview Q&A

**Q: How would you design a scalable web application on AWS?**
> A: Typical architecture: (1) **CloudFront** CDN caches static content at edge. (2) **Route53** DNS routes to **ALB** (Application Load Balancer). (3) ALB distributes to **EC2 Auto Scaling Group** (or ECS Fargate for containers). (4) **ElastiCache** (Redis) for session management and caching. (5) **RDS Aurora** Multi-AZ for relational data (or DynamoDB for key-value). (6) **S3** for file uploads and static assets. (7) **SQS** for async tasks (email sending, image processing). (8) **CloudWatch** for monitoring + alarms. This handles millions of requests, auto-scales with demand, and survives AZ failures.

---

## 🔗 What to Read Next

1. **[Cloud/AWS/Compute_EC2_Lambda.md](./Compute_EC2_Lambda.md)** — Deep dive into compute options
2. **[Cloud/AWS/Storage_S3_EBS_EFS.md](./Storage_S3_EBS_EFS.md)** — Storage services explained
3. **[Cloud/AWS/Networking_VPC.md](./Networking_VPC.md)** — Networking fundamentals

---

*[← Cloud Overview](../README.md) | [Back to Index](../../INDEX.md) | [Next: Compute →](./Compute_EC2_Lambda.md)*
