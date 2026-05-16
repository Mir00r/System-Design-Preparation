# 📦 AWS Storage: S3, EBS & EFS

> *"S3 stores unlimited objects with 99.999999999% durability (11 nines). EBS provides persistent block storage for EC2. EFS gives you shared file systems. Understanding which to use is critical for any AWS architecture."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [AWS Core Services](./Core_Services.md)

---

## 📋 Table of Contents
1. [Storage Types Comparison](#-storage-types)
2. [S3 (Simple Storage Service)](#-s3)
3. [EBS (Elastic Block Store)](#-ebs)
4. [EFS (Elastic File System)](#-efs)
5. [Decision Guide](#-decision-guide)
6. [Interview Q&A](#-interview-qa)

---

## 🗂️ Storage Types

```
┌────────────────────────────────────────────────────────────────────┐
│                    AWS STORAGE TYPES                                │
├───────────────┬───────────────┬────────────────┬──────────────────┤
│               │ S3            │ EBS            │ EFS              │
├───────────────┼───────────────┼────────────────┼──────────────────┤
│ Type          │ Object store  │ Block store    │ File store (NFS) │
│ Analogy       │ Dropbox       │ Hard drive     │ Shared NAS       │
│ Access        │ HTTP/API      │ Mounted to EC2 │ Mounted (NFS)    │
│ Sharing       │ Any service   │ ONE EC2 (mostly)│ Multiple EC2s   │
│ Scalability   │ Unlimited     │ Up to 64 TB    │ Petabytes        │
│ Latency       │ 50-100ms      │ <1ms (SSD)     │ Low ms           │
│ Durability    │ 11 nines      │ 99.999%        │ 11 nines         │
│ Use case      │ Files, backups│ OS disks, DBs  │ Shared content   │
│ Cost          │ $0.023/GB/mo  │ $0.08/GB/mo    │ $0.30/GB/mo      │
└───────────────┴───────────────┴────────────────┴──────────────────┘
```

---

## 🪣 S3

```
WHAT: Unlimited object storage accessible via HTTP
  - Store anything: images, videos, backups, logs, static websites
  - Objects: up to 5TB each (use multipart upload for >100MB)
  - Buckets: globally unique name, region-specific
  - 11 nines durability = if you store 10M objects, expect to lose 1 every 10,000 years

STORAGE CLASSES (cost optimization):
  Standard:          Frequent access ($0.023/GB)
  Intelligent-Tier:  Auto-moves between tiers (small monitoring fee)
  Standard-IA:       Infrequent access ($0.0125/GB, retrieval fee)
  Glacier Instant:   Archive with ms retrieval ($0.004/GB)
  Glacier Flexible:  Archive, minutes-hours retrieval ($0.0036/GB)
  Glacier Deep:      Cheapest, 12-48hr retrieval ($0.00099/GB)
  
  LIFECYCLE RULE EXAMPLE:
    Day 0-30:   Standard (frequent access)
    Day 31-90:  Standard-IA (less frequent)
    Day 91-365: Glacier Instant (rarely accessed)
    Day 366+:   Glacier Deep Archive (compliance retention)

KEY FEATURES:
  Versioning:     Keep all versions of an object (accidental delete protection)
  Encryption:     SSE-S3, SSE-KMS, SSE-C (server-side), client-side
  Replication:    Cross-Region Replication (CRR) for DR
  Event notifications: S3 → Lambda/SQS/SNS on upload/delete
  Static website hosting: serve HTML/CSS/JS directly from S3
  Pre-signed URLs: temporary secure access (download/upload)
```

---

## 💾 EBS

```
WHAT: Block storage volumes attached to EC2 instances
  - Like a hard drive for your virtual machine
  - Persist independently of EC2 (survives instance stop/terminate)
  - Snapshot to S3 for backup (incremental, cross-region copy)

VOLUME TYPES:
  gp3 (General Purpose SSD): 
    3000 IOPS baseline, up to 16,000 IOPS
    125 MB/s baseline, up to 1000 MB/s
    Best for: boot volumes, dev/test, small-medium DBs
    
  io2 Block Express (Provisioned IOPS SSD):
    Up to 256,000 IOPS, 4000 MB/s
    Best for: large databases, latency-sensitive workloads
    
  st1 (Throughput Optimized HDD):
    500 MB/s, 500 IOPS max
    Best for: big data, data warehousing, log processing
    
  sc1 (Cold HDD):
    250 MB/s, cheapest
    Best for: infrequent access, lowest cost

LIMITATIONS:
  - Single AZ (if AZ fails, volume unavailable until AZ recovers)
  - Attached to ONE EC2 at a time (io2 supports multi-attach for up to 16)
  - Must be same AZ as EC2 instance
  - Max 64 TB per volume
```

---

## 📁 EFS

```
WHAT: Managed NFS file system shared across multiple EC2 instances
  - Multiple instances read/write simultaneously
  - Auto-scales (no pre-provisioning capacity)
  - Cross-AZ by default (highly available)
  - POSIX-compliant (standard file system interface)

USE CASES:
  - Shared content management (CMS, media files)
  - Machine learning training data (shared across GPU instances)
  - Container storage (ECS/EKS persistent volumes)
  - Home directories for development teams
  - Web serving (shared document root across instances)

PERFORMANCE MODES:
  General Purpose: low latency, most workloads
  Max I/O: higher throughput, higher latency (big data, parallel processing)

COST OPTIMIZATION:
  Standard: $0.30/GB/month (frequently accessed)
  IA (Infrequent Access): $0.025/GB/month + retrieval fee
  Lifecycle policy: auto-move files not accessed in 30 days to IA
```

---

## 🧭 Decision Guide

```
"Where should I store this?"

  User-uploaded images/files  → S3 (unlimited, cheap, CDN-friendly)
  Database data files         → EBS (low latency, attached to DB server)
  Application logs            → S3 (after shipping with Kinesis/Fluentd)
  Shared config across servers→ EFS or S3 (depends on access pattern)
  Machine learning datasets   → S3 (storage) + EFS (training - shared access)
  Static website              → S3 + CloudFront
  Docker container storage    → EBS (ephemeral) or EFS (persistent shared)
  Database backups            → S3 (EBS snapshots automatically go to S3)
  Video transcoding input     → S3 (trigger Lambda on upload)
```

---

## ⚠️ Common Pitfalls

1. **S3 bucket naming and access** — Bucket names are globally unique. Use company prefix. Enable "Block Public Access" at account level. Use bucket policies + IAM (not ACLs) for access control.

2. **EBS not backed up** — EBS volumes are single-AZ. Create automated snapshots using AWS Backup or DLM (Data Lifecycle Manager). Without snapshots, AZ failure = data loss.

3. **EFS cost surprise** — EFS at $0.30/GB is 13x more expensive than S3. Enable lifecycle policies to move cold data to EFS-IA. Don't use EFS for archival storage.

---

## 📝 Interview Q&A

**Q: How would you design a system to handle image uploads for a social media app?**
> A: (1) Client uploads to **S3** via pre-signed URL (direct upload, bypasses your servers). (2) S3 event notification triggers **Lambda** for image processing (resize, thumbnail, format conversion). (3) Lambda stores processed versions back to S3 (different prefix/bucket). (4) **CloudFront** CDN serves images globally with low latency. (5) **DynamoDB** stores metadata (user_id, image_id, S3 key, dimensions, upload_date). Benefits: unlimited storage, no server bottleneck on upload, auto-processing pipeline, global delivery via CDN, pay only for storage used.

---

## 🔗 What to Read Next

1. **[Cloud/AWS/Databases_RDS_DynamoDB.md](./Databases_RDS_DynamoDB.md)** — Managed databases
2. **[Cloud/AWS/Networking_VPC.md](./Networking_VPC.md)** — Network architecture
3. **[BuildingBlocks/CDN.md](../../BuildingBlocks/CDN.md)** — CDN + S3 patterns

---

*[← Compute](./Compute_EC2_Lambda.md) | [Back to Index](../../INDEX.md) | [Next: Databases →](./Databases_RDS_DynamoDB.md)*
