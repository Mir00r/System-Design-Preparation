# ⚡ AWS Compute: EC2, Lambda, ECS & Fargate

> *"EC2 gives you full control over virtual machines. Lambda lets you run code without thinking about servers. ECS/Fargate runs containers at scale. Choosing the right compute option is the first decision in any AWS architecture."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [AWS Core Services](./Core_Services.md)

---

## 📋 Table of Contents
1. [Compute Options Overview](#-compute-options-overview)
2. [EC2 (Elastic Compute Cloud)](#-ec2)
3. [Lambda (Serverless Functions)](#-lambda)
4. [ECS & Fargate (Containers)](#-ecs--fargate)
5. [Decision Matrix](#-decision-matrix)
6. [Interview Q&A](#-interview-qa)

---

## 🗺️ Compute Options Overview

```
┌───────────────────────────────────────────────────────────────────────┐
│             MORE CONTROL ◀──────────────────────▶ LESS MANAGEMENT     │
├───────────────────────────────────────────────────────────────────────┤
│  EC2            │  ECS on EC2     │  ECS Fargate    │  Lambda         │
│  Full VM        │  Containers on  │  Serverless     │  Serverless     │
│  You manage:    │  your EC2s      │  containers     │  functions      │
│  - OS patches   │  You manage:    │  You manage:    │  You manage:    │
│  - scaling      │  - EC2 fleet    │  - container def│  - code only    │
│  - networking   │  - cluster      │  - task config  │                 │
│  - everything   │  Provider:      │  Provider:      │  Provider:      │
│  Provider:      │  - orchestration│  - servers      │  - everything   │
│  - hardware     │                 │  - scaling      │  - scaling      │
│                 │                 │  - patching     │  - runtime      │
├───────────────────────────────────────────────────────────────────────┤
│  Best for:      │  Best for:      │  Best for:      │  Best for:      │
│  Legacy apps    │  GPU workloads  │  Microservices  │  Event handlers │
│  Custom OS/HW   │  Cost optimize  │  General apps   │  APIs < 15min  │
│  Compliance     │  Specific AMIs  │  Simple ops     │  Glue logic     │
└───────────────────────────────────────────────────────────────────────┘
```

---

## 🖥️ EC2

```
WHAT: Virtual machines in the cloud
  - Choose instance type (CPU, memory, GPU, storage optimized)
  - Choose OS (Amazon Linux, Ubuntu, Windows, etc.)
  - Full root access, install anything
  - Persistent (runs until you stop it)

INSTANCE FAMILIES:
  t3/t4g  — Burstable (web servers, dev environments) — cheapest
  m5/m6g  — General purpose (balanced CPU/memory)
  c5/c6g  — Compute optimized (CPU-intensive: encoding, ML inference)
  r5/r6g  — Memory optimized (in-memory DBs, caches)
  p4/g5   — GPU (ML training, video rendering)
  i3/d3   — Storage optimized (databases, data warehousing)
  
  "g" suffix = ARM/Graviton (20% cheaper, same performance)

AUTO SCALING:
  ┌─────────────────────────────────────────────────┐
  │  Auto Scaling Group                             │
  │  Min: 2  |  Desired: 4  |  Max: 10             │
  │                                                 │
  │  [EC2] [EC2] [EC2] [EC2]  ← currently running  │
  │                                                 │
  │  Scale OUT when: CPU > 70% for 5 min            │
  │  Scale IN when:  CPU < 30% for 10 min           │
  └─────────────────────────────────────────────────┘

PRICING:
  On-demand:  $0.0116/hr (t3.micro) — most flexible
  Reserved:   ~40-60% cheaper (1-3 year commitment)
  Spot:       ~60-90% cheaper (can be interrupted with 2-min warning!)
  Savings Plan: flexible discount (commit $/hr, not instance type)
```

---

## ⚡ Lambda

```
WHAT: Run code without managing servers
  - Upload function code (Java, Python, Node.js, Go, .NET)
  - Triggered by events (API Gateway, S3, SQS, CloudWatch, etc.)
  - Pay per invocation + execution time (per millisecond!)
  - Auto-scales to thousands of concurrent executions

LIMITS:
  Timeout:           15 minutes max
  Memory:            128 MB to 10 GB
  Package size:      50 MB (zipped) / 250 MB (unzipped)
  Concurrent:        1000 default (can request increase)
  Ephemeral storage: 512 MB to 10 GB (/tmp)

COLD START:
  First invocation → provision container → load code → execute
  Subsequent → reuse warm container (much faster)
  
  Cold start latency:
    Python/Node.js: 100-500ms
    Java/Spring:    3-10 seconds! (JVM startup)
    
  Mitigations:
    - Provisioned concurrency (pre-warm N instances)
    - SnapStart (Java — snapshot after init, restore on invoke)
    - Smaller packages, lazy initialization

EVENT SOURCES:
  Synchronous:  API Gateway → Lambda → response to caller
  Asynchronous: S3 event → Lambda (retries on failure)
  Stream:       Kinesis/DynamoDB Streams → Lambda (ordered batches)
  Queue:        SQS → Lambda (batch processing)

PRICING EXAMPLE:
  1M invocations/month × 200ms × 256MB = ~$0.63/month
  vs EC2 t3.micro running 24/7 = ~$8.50/month
  (Lambda cheaper for low/spiky traffic; EC2 cheaper for steady high traffic)
```

---

## 🐳 ECS & Fargate

```
ECS (Elastic Container Service): AWS-native container orchestration
  
  ECS on EC2:    You manage the EC2 instances (more control, cheaper at scale)
  ECS on Fargate: AWS manages the servers (serverless containers)

CONCEPTS:
  Task Definition: blueprint (Docker image, CPU, memory, ports, env vars)
  Task:           running instance of a task definition (= running container)
  Service:        maintains desired count of tasks, handles load balancing
  Cluster:        logical grouping of tasks/services

FARGATE vs EC2 LAUNCH TYPE:
  ┌─────────────────────┬──────────────────┬──────────────────┐
  │                     │ Fargate          │ EC2 Launch Type  │
  ├─────────────────────┼──────────────────┼──────────────────┤
  │ Server management   │ None (serverless)│ You manage EC2s  │
  │ Scaling             │ Per-task         │ EC2 + task level │
  │ Cost (low traffic)  │ Cheaper          │ More expensive   │
  │ Cost (high traffic) │ More expensive   │ Cheaper (spot)   │
  │ GPU support         │ No               │ Yes              │
  │ SSH to host         │ No               │ Yes              │
  │ Startup time        │ 30-60s           │ Seconds (warm)   │
  └─────────────────────┴──────────────────┴──────────────────┘

EKS (Elastic Kubernetes Service):
  - Managed Kubernetes control plane
  - Use when: team already knows K8s, need portability, complex workloads
  - More powerful but more complex than ECS
```

---

## 📊 Decision Matrix

| Criteria | EC2 | Lambda | ECS Fargate |
|---|---|---|---|
| Startup time | Minutes (new) | Seconds | 30-60s |
| Max runtime | Unlimited | 15 min | Unlimited |
| Scaling speed | Minutes | Seconds | 30-60s |
| Cost model | Per hour | Per invocation | Per second |
| Best workload | Steady, long-running | Event-driven, spiky | Microservices |
| Ops burden | High | Minimal | Low |
| Customization | Full | Limited | Container-level |

---

## ⚠️ Common Pitfalls

1. **Lambda cold starts in Java** — Spring Boot on Lambda has 5-10 second cold starts. Use SnapStart, GraalVM native image, or lightweight frameworks (Micronaut, Quarkus) for Java Lambda. Or accept cold starts for async workloads where latency doesn't matter.

2. **EC2 without Auto Scaling** — Running fixed-size EC2 fleets wastes money during low traffic and crashes during peaks. Always use Auto Scaling Groups with proper scaling policies.

3. **Fargate for cost-sensitive high-throughput** — Fargate is convenient but ~20-30% more expensive than well-managed EC2 at scale. For stable high-volume workloads, ECS on Spot EC2 instances is significantly cheaper.

---

## 📝 Interview Q&A

**Q: When would you choose Lambda over containers (ECS/Fargate)?**
> A: Choose **Lambda** when: workload is event-driven (S3 upload → process), execution < 15 min, traffic is spiky/unpredictable (pay nothing at zero traffic), functions are stateless and independent. Choose **containers** when: need long-running processes, require specific runtime/OS dependencies, need consistent low latency (no cold starts), traffic is steady (containers cheaper), need more than 10GB memory, or microservices with persistent connections (gRPC, WebSocket). **Hybrid approach**: Use Lambda for glue logic and event processing, containers for core APIs and stateful services.

---

## 🔗 What to Read Next

1. **[Cloud/AWS/Storage_S3_EBS_EFS.md](./Storage_S3_EBS_EFS.md)** — Storage options
2. **[Cloud/AWS/Networking_VPC.md](./Networking_VPC.md)** — Network your compute
3. **[Architectures/Serverless.md](../../Architectures/Serverless.md)** — Serverless architecture patterns

---

*[← AWS Core Services](./Core_Services.md) | [Back to Index](../../INDEX.md) | [Next: Storage →](./Storage_S3_EBS_EFS.md)*
