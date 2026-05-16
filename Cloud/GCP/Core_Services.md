# 🔵 GCP Core Services: Google Cloud Platform Overview

> *"GCP excels at data engineering, Kubernetes (they invented it), and machine learning. While AWS has breadth, GCP often has depth in developer experience and managed infrastructure."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Cloud README](../README.md)

---

## 📋 GCP Service Map

```
┌─────────────────────────────────────────────────────────────────┐
│                   GCP CORE SERVICES                              │
├─────────────────────────────────────────────────────────────────┤
│ COMPUTE          │ Compute Engine, Cloud Run, GKE, Cloud Func  │
│ STORAGE          │ Cloud Storage, Persistent Disk, Filestore    │
│ DATABASE         │ Cloud SQL, Spanner, Firestore, Bigtable     │
│ NETWORKING       │ VPC, Cloud Load Balancing, Cloud CDN, DNS   │
│ MESSAGING        │ Pub/Sub, Cloud Tasks, Eventarc              │
│ BIG DATA         │ BigQuery, Dataflow, Dataproc, Composer      │
│ AI/ML            │ Vertex AI, AutoML, TPUs, Gemini API         │
│ SECURITY         │ IAM, KMS, Secret Manager, Binary Auth       │
│ DEVOPS           │ Cloud Build, Artifact Registry, Deploy      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏆 GCP Differentiators

```
WHERE GCP EXCELS:

1. KUBERNETES (GKE):
   - Google invented Kubernetes (donated to CNCF)
   - GKE Autopilot: fully managed K8s (no node management)
   - Best K8s experience of any cloud (multi-cluster, Anthos)

2. DATA & ANALYTICS (BigQuery):
   - Serverless data warehouse (PB-scale, SQL interface)
   - No infrastructure to manage, pay per query
   - Separation of storage and compute
   - ML built-in (BigQuery ML — run ML in SQL!)

3. SERVERLESS (Cloud Run):
   - Run containers serverlessly (any language, any framework)
   - Scale to zero (pay nothing when no traffic)
   - No cold start issues like Lambda (containers stay warm)
   - Standard Docker containers (no vendor lock-in)

4. GLOBAL DATABASE (Spanner):
   - Only globally distributed database with strong consistency
   - SQL + horizontal scaling (previously impossible)
   - 99.999% availability (5 nines — 5 min downtime/year)
   - Used by Google internally (AdWords, Google Play)

5. AI/ML:
   - TPUs (Tensor Processing Units — custom ML hardware)
   - Vertex AI (managed ML platform)
   - Pre-trained APIs (Vision, Speech, NLP, Translation)
   - Gemini API (LLM access)
```

---

## 🔄 AWS ↔ GCP Service Mapping

```
┌──────────────────────┬────────────────────┬────────────────────────┐
│ Category             │ AWS                │ GCP                    │
├──────────────────────┼────────────────────┼────────────────────────┤
│ VMs                  │ EC2                │ Compute Engine         │
│ Serverless functions │ Lambda             │ Cloud Functions        │
│ Serverless containers│ Fargate            │ Cloud Run              │
│ Kubernetes           │ EKS                │ GKE                   │
│ Object storage       │ S3                 │ Cloud Storage          │
│ Block storage        │ EBS                │ Persistent Disk        │
│ Managed SQL          │ RDS                │ Cloud SQL              │
│ Serverless NoSQL     │ DynamoDB           │ Firestore              │
│ Cache                │ ElastiCache        │ Memorystore            │
│ Global DB            │ Aurora Global      │ Spanner                │
│ Data warehouse       │ Redshift           │ BigQuery               │
│ Message queue        │ SQS                │ Cloud Tasks            │
│ Pub/Sub              │ SNS                │ Pub/Sub                │
│ Stream processing    │ Kinesis            │ Dataflow               │
│ CDN                  │ CloudFront         │ Cloud CDN              │
│ DNS                  │ Route53            │ Cloud DNS              │
│ Load balancer        │ ALB/NLB            │ Cloud Load Balancing   │
│ IAM                  │ IAM                │ Cloud IAM              │
│ Secrets              │ Secrets Manager    │ Secret Manager         │
│ CI/CD                │ CodePipeline       │ Cloud Build            │
│ Monitoring           │ CloudWatch         │ Cloud Monitoring       │
│ IaC                  │ CloudFormation     │ Deployment Manager     │
└──────────────────────┴────────────────────┴────────────────────────┘
```

---

## 💻 GCP Unique Strengths in Code

```java
// Cloud Run — deploy any Spring Boot app as serverless container
// Dockerfile (standard — no GCP-specific code!)
// FROM eclipse-temurin:21-jre
// COPY target/app.jar app.jar
// CMD ["java", "-jar", "app.jar"]
// 
// Deploy: gcloud run deploy my-service --image gcr.io/project/app
// Result: HTTPS URL, auto-scales 0→1000 instances, pay per request

// BigQuery — run SQL on petabytes
// No cluster to manage, no indexes to create
// SELECT 
//   DATE(created_at) as day,
//   COUNT(*) as orders,
//   SUM(total) as revenue
// FROM `project.dataset.orders`
// WHERE created_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
// GROUP BY day
// ORDER BY day DESC
// → Scans 1TB in seconds, costs ~$5

// Spanner — globally consistent SQL at scale
// Automatic sharding + strong consistency + SQL
// Like having a PostgreSQL that scales to millions of TPS globally
```

---

## ⚠️ Common Pitfalls

1. **Choosing GCP just for BigQuery** — BigQuery is incredible but can be used cross-cloud (BigQuery Omni). Don't choose your entire cloud provider for one service. Consider: where is your team's expertise?

2. **GCP's smaller community** — Fewer Stack Overflow answers, fewer third-party integrations, fewer consulting partners than AWS. Factor in learning curve and support ecosystem.

3. **Egress costs** — Like all clouds, GCP charges for data leaving their network. Design to minimize cross-region and internet egress. Use CDN for content delivery.

---

## 📝 Interview Q&A

**Q: When would you recommend GCP over AWS?**
> A: Choose GCP when: (1) **Data/analytics** is core — BigQuery is unmatched for serverless data warehousing. (2) **Kubernetes-first** — GKE Autopilot is the best managed K8s. (3) **ML/AI workloads** — TPUs + Vertex AI + pre-trained models. (4) **Simplicity** — Cloud Run makes serverless containers trivial (vs AWS's ECS/Fargate complexity). (5) **Global consistency** — Spanner is the only horizontally-scalable strongly-consistent SQL DB. Choose AWS when: breadth of services matters, enterprise compliance requirements, existing team expertise, or you need services GCP doesn't offer (IoT, some niche services).

---

## 🔗 What to Read Next

1. **[Cloud/Cloud_Comparison.md](../Cloud_Comparison.md)** — AWS vs GCP vs Azure decision guide
2. **[Cloud/AWS/Core_Services.md](../AWS/Core_Services.md)** — AWS services for comparison
3. **[Database/Database_Selection_Guide.md](../../Database/Database_Selection_Guide.md)** — Choosing databases

---

*[← AWS Well-Architected](../AWS/AWS_Well_Architected.md) | [Back to Index](../../INDEX.md) | [Next: Cloud Comparison →](../Cloud_Comparison.md)*
