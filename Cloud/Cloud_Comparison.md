# ⚔️ Cloud Comparison: AWS vs GCP vs Azure Decision Guide

> *"There's no 'best' cloud provider — only the best for YOUR specific requirements. This guide helps you make informed decisions based on workload type, team expertise, and business constraints."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [AWS Core Services](./AWS/Core_Services.md), [GCP Core Services](./GCP/Core_Services.md)

---

## 📊 Market Position

```
MARKET SHARE (2024):
  AWS:    32% — First mover, broadest services, enterprise
  Azure:  22% — Microsoft ecosystem, hybrid, enterprise
  GCP:    11% — Data/ML/K8s, developer experience
  Others: 35% — Alibaba, Oracle, IBM, DigitalOcean

GROWTH RATE:
  GCP growing fastest (~25% YoY)
  Azure growing steadily (~20% YoY)
  AWS growing but maturing (~15% YoY)
```

---

## 🏆 Strengths by Category

```
┌──────────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Category             │ AWS (Winner?)   │ Azure           │ GCP             │
├──────────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ Breadth of services  │ ✅ 200+ services│ Good (150+)     │ Fewer but focused│
│ Enterprise adoption  │ Strong          │ ✅ MS ecosystem │ Growing          │
│ Kubernetes           │ Good (EKS)      │ Good (AKS)      │ ✅ Best (GKE)   │
│ Serverless containers│ Good (Fargate)  │ Good (ACI)      │ ✅ Best (CloudRun)│
│ Data warehouse       │ Good (Redshift) │ Good (Synapse)  │ ✅ Best(BigQuery)│
│ Machine Learning     │ Good (SageMaker)│ Good (Azure ML) │ ✅ Best (Vertex) │
│ Hybrid cloud         │ Outposts        │ ✅ Best (Arc)   │ Anthos           │
│ Gaming               │ Good            │ ✅ Best(PlayFab)│ Limited          │
│ IoT                  │ ✅ Best(IoTCore)│ Good (IoT Hub)  │ Limited          │
│ Startup credits      │ $100K           │ $150K (BizSpark)│ $100K           │
│ Open source friendly │ Moderate        │ Moderate        │ ✅ Most          │
│ Global network       │ Good            │ Good            │ ✅ Best (private)│
│ Pricing transparency │ Complex         │ Complex         │ ✅ Simpler      │
│ Free tier            │ 12 months       │ 12 months       │ Always free tier │
└──────────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

---

## 🧭 Decision Framework

```
CHOOSE AWS WHEN:
  ✅ Need widest range of services (200+)
  ✅ Team already has AWS expertise
  ✅ Need specific services (IoT, Outposts, unique managed services)
  ✅ Enterprise compliance (GovCloud, most compliance certifications)
  ✅ Largest partner ecosystem (consultants, tools, training)
  ✅ Most job market demand

CHOOSE AZURE WHEN:
  ✅ Already using Microsoft (Office 365, Active Directory, .NET)
  ✅ Hybrid cloud (Azure Arc — manage on-prem + cloud together)
  ✅ Enterprise with existing Microsoft contracts (EA discounts)
  ✅ .NET/Windows workloads
  ✅ Gaming (PlayFab, Xbox integration)
  ✅ Need to integrate with Microsoft Teams, Power Platform

CHOOSE GCP WHEN:
  ✅ Data-intensive workloads (BigQuery, Dataflow, Pub/Sub)
  ✅ Kubernetes-first architecture (GKE is gold standard)
  ✅ Machine learning / AI (TPUs, Vertex AI, pre-trained models)
  ✅ Want simplicity (Cloud Run, fewer choices = less confusion)
  ✅ Need global database (Spanner — only option for global consistency)
  ✅ Open source preference (Kubernetes, TensorFlow, Go ecosystem)
  ✅ Network-intensive (Google's private global network)
```

---

## 💰 Cost Comparison

```
COMPUTE (similar size, monthly):
  AWS:   t3.medium (2 vCPU, 4GB) = ~$30/month
  Azure: B2s (2 vCPU, 4GB) = ~$30/month
  GCP:   e2-medium (2 vCPU, 4GB) = ~$25/month
  GCP wins: sustained use discounts (auto, no commitment)

STORAGE (per GB/month):
  AWS S3 Standard:    $0.023
  Azure Blob Hot:     $0.018
  GCP Cloud Storage:  $0.020
  Similar pricing, Azure slightly cheaper for storage

DATABASE (managed PostgreSQL, small):
  AWS RDS:    db.t3.micro = ~$15/month
  Azure:      Basic tier = ~$25/month
  GCP:        db-f1-micro = ~$8/month (shared core)
  
SERVERLESS FUNCTIONS (per 1M invocations):
  Lambda:          $0.20 + compute
  Azure Functions: $0.20 + compute
  Cloud Functions: $0.40 + compute (more expensive!)
  Cloud Run:       $0.00 (per container second — different model)

KEY INSIGHT: Costs are surprisingly similar across providers.
  The bigger savings come from ARCHITECTURE choices:
  - Serverless vs always-on
  - Reserved/committed vs on-demand
  - Right-sizing instances
  - Data transfer optimization
```

---

## 🚫 Vendor Lock-in Considerations

```
HIGH LOCK-IN (hard to migrate):
  AWS: DynamoDB, Lambda (event sources), Step Functions, SQS
  Azure: Cosmos DB, Azure Functions (bindings), Logic Apps
  GCP: BigQuery, Spanner, Cloud Dataflow, Firebase
  
LOW LOCK-IN (portable):
  Kubernetes: EKS ↔ AKS ↔ GKE (mostly portable)
  Containers: Docker runs anywhere
  Managed SQL: PostgreSQL on RDS ↔ Cloud SQL ↔ Azure DB
  Object storage: S3 API is de facto standard (most support it)
  Terraform: infrastructure code works across clouds

STRATEGY:
  Use managed services (accept some lock-in) for:
    - Databases (too painful to self-manage)
    - Queues/messaging (reliability matters)
    - ML/AI (accelerators are expensive)
    
  Stay portable for:
    - Application code (containers + K8s)
    - Infrastructure (Terraform/Pulumi)
    - CI/CD (GitHub Actions, Jenkins — cloud-agnostic)
```

---

## ⚠️ Common Pitfalls

1. **Multi-cloud for the wrong reasons** — "We'll avoid lock-in by using AWS AND GCP!" Reality: you double complexity, double expertise needed, and use lowest-common-denominator features of both. Use multi-cloud only when business justifies it (specific service, compliance, redundancy).

2. **Choosing based on one service** — Don't pick your entire cloud because BigQuery is cool. Consider: ecosystem, team skills, support, total cost, compliance requirements.

3. **Ignoring egress costs** — All clouds charge to move data OUT. Architect to minimize cross-region and cross-cloud data transfer. This is where multi-cloud gets expensive fast.

---

## 📝 Interview Q&A

**Q: How would you evaluate cloud providers for a new greenfield project?**
> A: Decision framework: (1) **Team expertise** — what does the team already know? (switching clouds = 6-month learning curve). (2) **Workload requirements** — compute-heavy? data-heavy? ML? The answer changes the provider. (3) **Compliance** — healthcare (HIPAA), finance (PCI), government (FedRAMP) may narrow options. (4) **Existing ecosystem** — Microsoft shop? Azure. AWS shop? Stick with it. Google shop (Workspace)? Consider GCP. (5) **Cost at scale** — model expected usage against all three. (6) **Exit strategy** — how locked-in will we be? Use containers + Kubernetes + Terraform for portability. For most teams: **stay with what you know** unless there's a compelling reason to switch. The productivity cost of learning a new cloud is higher than the feature difference.

---

## 🎲 Mini Challenge

> 🎲 **CHALLENGE** (5 minutes):
> You're a startup that just raised $2M. Your team of 6 engineers is building a real-time analytics platform
> for e-commerce (ingests purchase events, shows dashboards, trains recommendation models).
> The CTO wants to choose a cloud provider. What do you recommend and why?

<details>
<summary>💡 Click to reveal recommendation framework</summary>

**Analysis:**

```
REQUIREMENT              BEST MATCH    REASONING
─────────────────────────────────────────────────────────────────────
Real-time event ingestion AWS (Kinesis) or GCP (Pub/Sub)   Both excellent
Analytics / Dashboards   GCP (BigQuery)                    BQ's slot-based pricing
                                                            cheaper at startup scale
ML / Recommendations     GCP (Vertex AI) or AWS (SageMaker) GCP slightly ahead for ML
Team of 6 (productivity) GCP (Cloud Run, simpler IAM)       Gentler learning curve
Startup budget           GCP (always-free tier, $100K credits) Longest runway
```

**Recommendation: GCP**

Reasoning:
1. BigQuery — real-time streaming analytics with zero ops, pay-per-query pricing
2. Cloud Run — serverless containers, scales to zero (no idle costs for 6 engineers)
3. Vertex AI — pre-built recommendation models with minimal ML infrastructure
4. Always-free tier on many services (Pub/Sub 10GB/month free)
5. GCP's simpler IAM model reduces security configuration burden for a small team

**When AWS wins instead:**
- Team has existing AWS expertise → productivity beats feature differences
- Need a service that only exists on AWS (e.g., AWS IoT Core, DynamoDB specific features)
- Partners/investors are deep AWS shops

**Anti-pattern to avoid:**
Don't choose multi-cloud at 6 engineers. The operational overhead of managing two
cloud environments exceeds the flexibility benefit. Make a choice, own it.

</details>

---

## 💸 Detailed Cost Comparison (Running 1 Web Service 24/7)

| Resource | AWS | GCP | Azure |
|----------|-----|-----|-------|
| 1 vCPU / 2GB RAM VM | t3.small: $0.023/h ($17/mo) | e2-small: $0.017/h ($12/mo) | B1s: $0.012/h ($9/mo) |
| Managed PostgreSQL | RDS db.t3.micro: $27/mo | Cloud SQL: $25/mo | Azure DB Flex: $23/mo |
| 100 GB Object Storage | S3: $2.30/mo | GCS: $2.00/mo | Blob: $1.90/mo |
| 1TB Data Egress | $90/mo | $85/mo | $87/mo |
| Container Orchestration | EKS: $73/mo (control plane) | GKE: $74/mo | AKS: $0/mo (free) |
| Serverless (1M calls) | Lambda: $0.20 | Cloud Functions: $0.40 | Azure Functions: $0.20 |

**Key insight:** At small scale, differences are minor. At large scale (100+ VMs), AWS bulk discounts and Reserved Instance pricing can be 40-60% cheaper. GCP Sustained Use Discounts are automatic (no commitment required).

---

## 🔗 What to Read Next

1. **[Cloud/AWS/Core_Services.md](./AWS/Core_Services.md)** — AWS deep dive
2. **[Cloud/GCP/Core_Services.md](./GCP/Core_Services.md)** — GCP deep dive
3. **[DevOps/Kubernetes/](../DevOps/Kubernetes/)** — Container orchestration (portable across clouds)

---

*[← GCP Core Services](./GCP/Core_Services.md) | [Back to Index](../INDEX.md)*
