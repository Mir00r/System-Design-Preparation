# 📦 Blob & Object Storage: Storing Files at Planetary Scale

> *"Netflix stores 100+ petabytes of video content. Instagram ingests 100 million photos daily. You can't put that in a relational database. Object storage is how the internet stores its files — infinitely scalable, incredibly cheap, and deceptively simple."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [CDN](./CDN.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [What Is Object/Blob Storage?](#-what-is-objectblob-storage)
3. [How It Works](#-how-it-works)
4. [S3 — The Gold Standard](#-s3--the-gold-standard)
5. [Pre-Signed URLs (Secure Direct Upload)](#-pre-signed-urls-secure-direct-upload)
6. [Storage Classes & Cost Optimization](#-storage-classes--cost-optimization)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [Performance & Trade-offs](#-performance--trade-offs)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
Why can't we just store files in the database?

Option A: Store file in PostgreSQL BYTEA column
  Problems:
    - Database backups become enormous (10TB of images in every backup)
    - DB queries slow down (IO bandwidth consumed by file reads)
    - Expensive storage (EBS is $0.10/GB vs S3 at $0.023/GB)
    - Can't serve files directly to browsers (need to read through app server)
    - Database replication copies ALL file data (network saturated)

Option B: Store file on local disk (/var/uploads/)
  Problems:
    - Server dies = files lost (no replication)
    - Can't scale horizontally (files are on ONE server)
    - No CDN integration (slow for global users)
    - Disk space limited to single machine

Option C: Object storage (S3, GCS, Azure Blob) ✅
    - Infinite capacity (Amazon manages storage for you)
    - 99.999999999% durability (11 nines — virtually indestructible)
    - Serve directly via CDN (sub-50ms globally)
    - Costs $0.023/GB/month (Standard) or $0.004/GB (Archive)
    - Built-in versioning, encryption, lifecycle policies
```

---

## 💡 What Is Object/Blob Storage?

```
Object Storage model:

  Bucket (container)
    └── Object (file + metadata)
          ├── Key: "users/12345/profile/avatar.jpg"  (path-like unique ID)
          ├── Data: [binary content of the image]
          ├── Metadata: { content-type: "image/jpeg", uploaded-by: "user-12345", size: 204800 }
          └── Tags: { department: "marketing", classification: "public" }

Key characteristics:
  - Flat namespace (no directories — "folders" are just key prefixes)
  - Immutable objects (update = upload new version)
  - Access via HTTP (GET/PUT to a URL)
  - Eventual consistency for overwrites (read-after-write for new objects on S3)
```

### Object Storage vs File System vs Block Storage

| Feature | Object Storage (S3) | File System (EFS/NFS) | Block Storage (EBS) |
|---|---|---|---|
| **Access** | HTTP API (REST) | POSIX (mount + read/write) | Mount as disk (fstab) |
| **Scale** | Infinite (managed) | TB scale (manually expand) | Single server (limited) |
| **Performance** | High throughput, higher latency | Lower latency, lower throughput | Lowest latency |
| **Cost** | Cheapest per GB ($0.023) | Medium ($0.30/GB) | Medium ($0.10/GB) |
| **Use case** | Images, videos, backups, logs | Shared file access (NFS mount) | Database volumes, OS disks |
| **Durability** | 11 nines (99.999999999%) | Depends on implementation | 99.999% (5 nines) |

---

## 🏗️ How It Works

```
Upload flow:

  [Client App] ──PUT──▶ [S3 API endpoint]
                              │
                ┌─────────────┼──────────────────┐
                ▼             ▼                   ▼
           [Shard 1]     [Shard 2]          [Shard N]
           (3 copies)    (3 copies)          (3 copies)

  S3 internally:
    1. Receives the file
    2. Splits into chunks if large (multipart upload)
    3. Replicates across 3+ availability zones
    4. Returns HTTP 200 with ETag (MD5 hash of content)
    5. Object is immediately readable (read-after-write consistency)

Download flow:

  [Browser] ──GET──▶ [CloudFront CDN] ──cache miss──▶ [S3]
                           │
                      cache hit → serve from edge (< 50ms globally)
```

---

## ☁️ S3 — The Gold Standard

Amazon S3 (Simple Storage Service) defines the industry standard API that most object stores implement:

```
S3 API basics:
  PUT    /bucket-name/key           → Upload object
  GET    /bucket-name/key           → Download object
  DELETE /bucket-name/key           → Delete object
  HEAD   /bucket-name/key           → Get metadata without downloading
  LIST   /bucket-name?prefix=users/ → List objects with prefix

S3-compatible alternatives:
  - Google Cloud Storage (GCS)
  - Azure Blob Storage
  - MinIO (self-hosted, S3-compatible API)
  - DigitalOcean Spaces
  - Backblaze B2
```

---

## 🔐 Pre-Signed URLs (Secure Direct Upload)

The most important pattern for file uploads in modern applications:

```
BAD pattern (file passes through your server):
  Client ──upload 100MB──▶ [Your App Server] ──upload 100MB──▶ [S3]
  Problem: your server is a bottleneck, uses bandwidth, memory, CPU

GOOD pattern (pre-signed URL — client uploads directly to S3):
  1. Client asks your server: "I want to upload avatar.jpg (2MB)"
  2. Server generates a pre-signed URL (valid for 5 minutes):
     https://bucket.s3.amazonaws.com/users/123/avatar.jpg
       ?X-Amz-Algorithm=AWS4-HMAC-SHA256
       &X-Amz-Credential=...
       &X-Amz-Expires=300
       &X-Amz-Signature=abc123...
  3. Client uploads DIRECTLY to S3 using that URL (no server in the path)
  4. S3 verifies the signature and accepts the upload
  5. Client notifies your server: "upload complete, key = users/123/avatar.jpg"
  6. Server stores the key in the database

  Security: URL is time-limited, size-limited, and content-type-restricted
  Performance: your server handles zero file bytes — only metadata
```

### Spring Boot Pre-Signed URL Example

```java
@RestController
public class FileUploadController {

    private final S3Presigner presigner;
    private final String bucketName = "my-app-uploads";

    @PostMapping("/api/upload-url")
    public Map<String, String> getUploadUrl(@RequestBody UploadRequest request) {
        // Validate: only allow images, max 10MB
        if (!Set.of("image/jpeg", "image/png", "image/webp").contains(request.contentType())) {
            throw new ResponseStatusException(HttpStatus.BAD_REQUEST, "Invalid content type");
        }

        String key = "users/" + getCurrentUserId() + "/uploads/" + UUID.randomUUID()
                     + getExtension(request.contentType());

        PutObjectRequest putRequest = PutObjectRequest.builder()
                .bucket(bucketName)
                .key(key)
                .contentType(request.contentType())
                .contentLength(request.fileSize())   // enforce exact size
                .build();

        PutObjectPresignRequest presignRequest = PutObjectPresignRequest.builder()
                .signatureDuration(Duration.ofMinutes(5))  // URL valid for 5 min
                .putObjectRequest(putRequest)
                .build();

        PresignedPutObjectRequest presigned = presigner.presignPutObject(presignRequest);

        return Map.of(
            "uploadUrl", presigned.url().toString(),
            "key", key,
            "expiresAt", Instant.now().plus(Duration.ofMinutes(5)).toString()
        );
    }
}
```

---

## 💰 Storage Classes & Cost Optimization

| Storage Class | Use Case | Cost/GB/month | Retrieval | Min Duration |
|---|---|---|---|---|
| **S3 Standard** | Frequently accessed data | $0.023 | Instant | None |
| **S3 Infrequent Access** | Accessed < 1x/month | $0.0125 | Instant (+ retrieval fee) | 30 days |
| **S3 Glacier Instant** | Archive, needs instant access | $0.004 | Instant | 90 days |
| **S3 Glacier Deep Archive** | Compliance archives, rarely accessed | $0.00099 | 12-48 hours | 180 days |

### Lifecycle Policies (Automate Cost Savings)

```json
{
  "Rules": [{
    "ID": "ArchiveOldUploads",
    "Filter": { "Prefix": "users/" },
    "Transitions": [
      { "Days": 30, "StorageClass": "STANDARD_IA" },
      { "Days": 365, "StorageClass": "GLACIER" }
    ],
    "Expiration": { "Days": 2555 }
  }]
}
// Day 0-30: Standard ($0.023/GB) — user actively viewing their photos
// Day 30-365: Infrequent Access ($0.0125/GB) — 46% cheaper
// Day 365+: Glacier ($0.004/GB) — 83% cheaper than Standard
// Day 2555 (7 years): Delete automatically
```

---

## ⚠️ Common Pitfalls

1. **Uploading through your application server** — Every file byte passes through your server, consuming bandwidth and memory. Use pre-signed URLs for direct client-to-S3 uploads. Your server should only handle metadata, never file bytes.

2. **Public bucket permissions** — S3 buckets are private by default. Accidentally setting `PublicRead` ACL exposes all objects. Use bucket policies with explicit denials and enable S3 Block Public Access at the account level.

3. **No content-type validation** — Accepting any file upload without validating content-type allows attackers to upload executable files. Validate both the Content-Type header AND file magic bytes on the server side.

4. **Storing keys in a flat structure** — Keys like `file1.jpg`, `file2.jpg` make listing slow and unmanageable. Use hierarchical prefixes: `users/{userId}/uploads/{year}/{month}/{uuid}.{ext}` for efficient prefix-based listing.

5. **Not enabling versioning** — Without versioning, an accidental overwrite or delete is permanent. Enable versioning on important buckets, and use lifecycle rules to expire old versions after 30 days.

---

## 🧩 Mini Challenge

**Design the file storage architecture for a chat application (like WhatsApp) where users can send images, videos, and documents (up to 100MB).**

Requirements: 10M messages/day with media, global users, files must be downloadable for 30 days after sending.

<details>
<summary>💡 Click to reveal answer</summary>

**Architecture**:

1. **Upload path**: Sender → pre-signed PUT URL → S3 (direct upload, bypass app server)
2. **Storage**: S3 Standard with 30-day lifecycle → auto-delete after 30 days
3. **Download path**: Receiver → pre-signed GET URL (time-limited, 1 hour expiry) → CloudFront CDN → S3
4. **CDN**: CloudFront for global latency < 100ms; signed URLs ensure only the recipient can access
5. **Key structure**: `media/{chatId}/{messageId}/{filename}` — easy to batch-delete when chat is deleted
6. **Thumbnails**: Lambda trigger on S3 upload → generate thumbnail → store as `media/{chatId}/{messageId}/thumb_{filename}`
7. **Virus scanning**: S3 event → Lambda → ClamAV scan → if infected, delete + notify user
8. **Encryption**: S3-SSE (server-side encryption) with KMS key for at-rest encryption
9. **Cost**: 10M messages/day × 30% have media × 5MB average = 15TB/day new storage. 30-day retention = 450TB active. At $0.023/GB = ~$10,350/month. With lifecycle to IA after 7 days: ~$7,000/month.

</details>

---

## 📝 Interview Q&A

**Q: Why use object storage instead of a database for files?**
> A: Object storage is purpose-built for large binary data: infinite scalability (no capacity planning), 11 nines durability (S3 stores 3+ copies across AZs), ~10x cheaper than database storage ($0.023/GB vs database EBS $0.10/GB+), direct HTTP access (serve via CDN without app server), and lifecycle management (auto-archive/delete). Databases are optimized for structured queries, transactions, and indexes — storing large blobs in them wastes expensive resources (RAM, IOPS, backup bandwidth) on data that doesn't benefit from those features.

**Q: How do you secure files in S3 so only authorized users can access them?**
> A: Never make buckets public. Use pre-signed URLs: your app server generates a time-limited signed URL (e.g., valid 1 hour) that grants access to a specific object. The client uses this URL to download directly from S3. The signature is verified by S3 — no way to tamper or extend expiry. For additional security: enable S3 Block Public Access at account level, use bucket policies to deny all except your app's IAM role, enable server-side encryption (SSE-KMS), and log all access with S3 Access Logs or CloudTrail.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/CDN.md](./CDN.md)** — CDNs cache objects at edge locations for fast global delivery
2. **[SystemDesignCaseStudies/DesignDropbox.md](../SystemDesignCaseStudies/DesignDropbox.md)** — File sync system design built on object storage
3. **[Security/Secrets_Management.md](../Security/Secrets_Management.md)** — Manage S3 credentials safely (never hardcode AWS keys)

---

*[← Proxy & Reverse Proxy](./Proxy_ReverseProxy.md) | [Back to BuildingBlocks](../INDEX.md) | [Next: Search Index →](./SearchIndex.md)*
