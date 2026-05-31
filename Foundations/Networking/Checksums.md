# 🧮 Checksums & Data Integrity

> *"Every time you download a file, how do you know it wasn't corrupted during transfer? Every time data is written to disk, how do you know it was stored correctly? Checksums — simple mathematical fingerprints that detect corruption — are the invisible guardians protecting every byte flowing through the internet. From TCP packets to Git commits to database pages, checksums are EVERYWHERE."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [TCP vs UDP](TCP_vs_UDP.md)

---

## 📋 Table of Contents
1. [What Is a Checksum?](#-what-is-a-checksum)
2. [Common Checksum Algorithms](#-common-checksum-algorithms)
3. [Where Checksums Are Used](#-where-checksums-are-used)
4. [Checksums in Distributed Systems](#-checksums-in-distributed-systems)
5. [Java Implementation](#-java-implementation)
6. [Interview Q&A](#-interview-qa)

---

## 🎯 What Is a Checksum?

```
A CHECKSUM is a small fixed-size value computed from data,
used to detect if data was altered or corrupted.

  Original data: "Hello World" → checksum: 0xAB3F
  Corrupted data: "Hello Worl!" → checksum: 0x7C12 (DIFFERENT!)
  
  DETECTION: If computed checksum ≠ expected checksum → DATA CORRUPTED!

ANALOGY:
  Sending a package via mail:
  • You weigh it: 2.5 kg (this is the "checksum")
  • You write "2.5 kg" on the label
  • Recipient weighs it: 2.5 kg? → Package intact! ✅
  • Recipient weighs it: 2.3 kg? → Something fell out! ❌

PROPERTIES OF A GOOD CHECKSUM:
  1. DETERMINISTIC: Same input → always same checksum
  2. FAST to compute: shouldn't slow down data transfer!
  3. SENSITIVE: Small change in input → different checksum
  4. Fixed size: Output is always same length regardless of input
  
  NOT A SECURITY MECHANISM! (checksums ≠ cryptographic hashes)
  CRC can detect accidental corruption but NOT malicious tampering!
  For security: use SHA-256, HMAC (cryptographic hashes!)
```

---

## 🔢 Common Checksum Algorithms

```
┌────────────────────────────────────────────────────────────────────┐
│  Algorithm  │  Size    │  Speed     │  Use Case                    │
├────────────────────────────────────────────────────────────────────┤
│  CRC-32     │  32 bits │  Very fast │  Network packets, Zip files  │
│  Adler-32   │  32 bits │  Fastest   │  zlib compression            │
│  MD5        │  128 bits│  Fast      │  File verification (legacy)  │
│  SHA-1      │  160 bits│  Medium    │  Git commits (being phased)  │
│  SHA-256    │  256 bits│  Slower    │  Security, blockchain        │
│  xxHash     │  64 bits │  Fastest   │  Hash tables, deduplication  │
│  MurmurHash │  128 bits│  Very fast │  Distributed systems, Bloom  │
└────────────────────────────────────────────────────────────────────┘

CRC-32 (Cyclic Redundancy Check):
  • Used in: Ethernet frames, ZIP files, PNG images
  • Detects: all single-bit errors, most multi-bit errors
  • NOT secure! Easy to craft collisions intentionally!
  
MD5 / SHA-256 (Cryptographic Hashes):
  • Used for: file integrity verification, digital signatures
  • MD5: broken for security! Still OK for integrity checks.
  • SHA-256: current standard for security applications
  
WHEN TO USE WHICH:
  Just detecting corruption → CRC-32 (fastest!)
  File verification downloads → SHA-256 (published hash on website)
  Distributed systems → MurmurHash / xxHash (fast + good distribution)
  Blockchain/security → SHA-256 / SHA-3 (cryptographically secure)
```

---

## 🌐 Where Checksums Are Used

```
EVERYWHERE! At every layer of the stack!

NETWORK LAYER:
  ┌─────────────────────────────────────────────────────────┐
  │  Ethernet frame: CRC-32 at end of each frame!          │
  │  IP packet: Header checksum (16 bits)                   │
  │  TCP segment: Checksum over header + data               │
  │  UDP datagram: Optional checksum (but usually included) │
  └─────────────────────────────────────────────────────────┘
  
  If checksum fails → packet DROPPED! Sender will retransmit (TCP).

STORAGE LAYER:
  • Hard drives: ECC (Error Correcting Code) on every sector!
  • SSDs: CRC per page (detect bit rot!)
  • RAID: Parity data reconstructs corrupted drives!
  • ZFS/Btrfs: Checksums every data block (detect silent corruption!)

APPLICATION LAYER:
  • Git: SHA-1 hash of every commit, tree, blob
    git commit abc123 → entire history is verified by hash chain!
  • Docker: Image layers identified by SHA-256 hash
  • Package managers: npm/pip verify package integrity via checksum
  • Databases: Page-level checksums (PostgreSQL data_checksums)
  • S3/GCS: ETag (MD5 of uploaded content) for verification
  • Kafka: CRC-32 per message batch (detect disk/network corruption)

IN DISTRIBUTED SYSTEMS:
  • HDFS: Checksum per 512-byte chunk (detects data node corruption!)
  • Cassandra: CRC-32 on commit log and SSTables
  • Merkle Trees: Tree of hashes for efficient comparison!
    (Used by Bitcoin, Cassandra anti-entropy, ZFS, Git)
```

---

## 🏗️ Checksums in Distributed Systems

```
MERKLE TREES (Hash Trees):
  The power tool for data integrity in distributed systems!
  
  Verify TERABYTES of data by comparing just ONE hash!
  
        H(root) = hash(H1 + H2)         ← Compare just this!
         /              \                   If same → ALL data matches!
     H1 = hash(A+B)   H2 = hash(C+D)    If different → drill down!
      /      \          /      \
   hash(A)  hash(B)  hash(C)  hash(D)
     |        |        |        |
   Data A   Data B   Data C   Data D

  USE CASE: Cassandra anti-entropy repair
  Node A and Node B: each has copy of data.
  Compare Merkle tree roots:
    Same? → Data is in sync! (verified terabytes in one comparison!)
    Different? → Drill into subtrees → find exact differing blocks!
    Transfer ONLY the blocks that differ! (not entire dataset!)

DATA INTEGRITY IN TRANSIT:
  Client uploads 5 GB file to S3:
  1. Client computes MD5 while uploading
  2. Sends "Content-MD5" header with the request
  3. S3 computes MD5 of received data
  4. Compares: client's MD5 == S3's MD5? 
     YES → file stored correctly!
     NO → reject upload! Network corrupted data!

IDEMPOTENCY KEYS AS CHECKSUMS:
  Not traditional checksums, but same concept!
  Request hash = SHA-256(payload + timestamp + user_id)
  If same hash seen before → duplicate request! Skip!
```

---

## 💻 Java Implementation

```java
import java.security.MessageDigest;
import java.util.zip.CRC32;

public class ChecksumUtils {
    
    /**
     * CRC-32: fastest, for non-security integrity checks.
     * Used in: network protocols, file formats, Kafka messages.
     */
    public static long crc32(byte[] data) {
        CRC32 crc = new CRC32();
        crc.update(data);
        return crc.getValue(); // 32-bit checksum
    }
    
    /**
     * SHA-256: cryptographically secure hash.
     * Used in: file verification, blockchain, digital signatures.
     */
    public static String sha256(byte[] data) {
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        byte[] hash = digest.digest(data);
        return bytesToHex(hash);
    }
    
    /**
     * Verify file integrity after download.
     * Compare computed hash with expected hash from trusted source.
     */
    public static boolean verifyFileIntegrity(
            Path filePath, String expectedSha256) throws IOException {
        
        MessageDigest digest = MessageDigest.getInstance("SHA-256");
        
        try (InputStream is = Files.newInputStream(filePath)) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                digest.update(buffer, 0, bytesRead);
            }
        }
        
        String actualHash = bytesToHex(digest.digest());
        return actualHash.equalsIgnoreCase(expectedSha256);
    }
    
    /**
     * Simple Merkle Tree for data block verification.
     */
    public static String merkleRoot(List<byte[]> dataBlocks) {
        List<String> hashes = dataBlocks.stream()
            .map(ChecksumUtils::sha256)
            .collect(Collectors.toList());
        
        while (hashes.size() > 1) {
            List<String> nextLevel = new ArrayList<>();
            for (int i = 0; i < hashes.size(); i += 2) {
                String left = hashes.get(i);
                String right = (i + 1 < hashes.size()) 
                    ? hashes.get(i + 1) : left;
                nextLevel.add(sha256((left + right).getBytes()));
            }
            hashes = nextLevel;
        }
        
        return hashes.get(0); // Root hash!
    }
}
```

---

## ❓ Interview Q&A

**Q1: What's the difference between a checksum and a hash?**
> Checksum (CRC-32, Adler-32): fast, detects accidental corruption, NOT secure (easy to create intentional collisions). Hash (SHA-256, MD5): slower, detects both accidental AND intentional tampering (cryptographically hard to create collisions). For data integrity in transit: CRC is sufficient and faster. For security (verifying downloads, signatures): use SHA-256.

**Q2: How do Merkle Trees help in distributed systems?**
> Merkle Trees allow comparing entire datasets by comparing just the root hash (O(1)!). If roots differ: drill into children to find exact differing blocks in O(log N) comparisons. Used in: Cassandra anti-entropy (sync replicas efficiently), Bitcoin (verify transactions without downloading entire blockchain), Git (content-addressable storage), HDFS (verify data blocks). Key benefit: only transfer data that actually differs!

**Q3: What is "bit rot" and how do filesystems handle it?**
> Bit rot: data on disk silently flips bits over time (cosmic rays, magnetic decay, firmware bugs). Drives DON'T report errors! Data looks fine but has corrupted bytes. Solution: ZFS/Btrfs compute checksums for every block. On read: verify checksum. If mismatch: automatically heal from redundant copy (mirror/RAID). PostgreSQL: enable data_checksums to detect corrupt pages. Without checksums: you serve corrupt data to users silently!

**Q4: How does TCP ensure data integrity?**
> TCP checksum: 16-bit ones' complement sum of header + data + pseudo-header (source/dest IP). Sender computes checksum, receiver recomputes and compares. If mismatch: segment discarded, sender retransmits. But TCP checksum is WEAK (16-bit, misses some multi-bit errors!). That's why applications add their own checksums (HDFS, Kafka, etc.) — TCP alone isn't sufficient for data integrity guarantees.

---

## 🔗 Related Topics
- [TCP vs UDP](TCP_vs_UDP.md) — Transport-layer checksums
- [Bloom Filters](../../Database/BloomFilters.md) — Hash-based data structures
- [Distributed Locking](../../Microservices/DistributedLocking.md) — Data consistency
- [Gossip Protocol](../../Microservices/GossipProtocol.md) — Merkle tree anti-entropy

---

*"A checksum is a guard at the gate of data integrity. It can't fix corruption, but it can TELL you something is wrong — and in distributed systems, knowing something is wrong before serving it to users is worth its weight in gold." — Storage Engineer* 🧮
