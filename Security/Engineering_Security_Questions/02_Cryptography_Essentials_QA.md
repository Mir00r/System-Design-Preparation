# 🔐 Cryptography Essentials Q&A — The Math That Protects the Internet

> *"Cryptography is the art of writing secrets. Every time you log in, make a payment, or send a message — cryptographic algorithms are working invisibly to keep you safe. The difference between a secure system and a breached one often comes down to whether the engineer understood hashing vs. encryption vs. encoding. Get these wrong, and no firewall in the world can save you."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Security Fundamentals](./01_Security_Fundamentals_QA.md)

---

## 📋 Table of Contents
1. [The Big Picture — Why Cryptography?](#-the-big-picture--why-cryptography)
2. [Encoding vs Hashing vs Encryption — The Critical Distinction](#-encoding-vs-hashing-vs-encryption--the-critical-distinction)
3. [Hashing Deep Dive](#-hashing-deep-dive)
4. [Symmetric Encryption](#-symmetric-encryption)
5. [Asymmetric Encryption](#-asymmetric-encryption)
6. [Digital Signatures](#-digital-signatures)
7. [Key Management](#-key-management)
8. [Common Cryptographic Mistakes](#-common-cryptographic-mistakes)
9. [Gamification: Crypto Puzzle Room](#-gamification-crypto-puzzle-room)
10. [Interview Q&A — 25 Must-Know Questions](#-interview-qa--25-must-know-questions)
11. [What's Next](#-whats-next)

---

## 🌍 The Big Picture — Why Cryptography?

```
WITHOUT CRYPTOGRAPHY:                    WITH CRYPTOGRAPHY:

[Alice] ──"Send $100 to Bob"──▶         [Alice] ──"a7f3...encrypted..."──▶
         │                                         │
    [Eve reads it]                            [Eve sees gibberish]
    [Eve changes it to $10,000]               [Eve can't modify it]
    [Eve pretends to be Alice]                [Eve can't impersonate]
         │                                         │
         ▼                                         ▼
[Bank processes $10,000]                    [Bank processes $100] ✅

CRYPTOGRAPHY GIVES US:
  🔒 Confidentiality → Encryption (only intended recipient reads it)
  ✅ Integrity       → Hashing/MAC (detects any modification)
  🪪 Authentication  → Digital Signatures (proves who sent it)
  📜 Non-repudiation → Signatures (sender can't deny sending it)
```

### 🎮 The Cryptography Family Tree

```
                    CRYPTOGRAPHY
                         │
          ┌──────────────┼──────────────┐
          │              │              │
     ENCODING       HASHING       ENCRYPTION
     (NOT crypto!)   (one-way)    (two-way)
          │              │              │
      Base64         ┌───┴───┐    ┌────┴────┐
      URL encode     │       │    │         │
      Hex            │       │    │         │
                  General  Password   Symmetric  Asymmetric
                  purpose  specific      │           │
                     │       │       AES-256     RSA-2048
                  SHA-256  bcrypt    ChaCha20    Ed25519
                  SHA-3    Argon2               ECDSA
                           scrypt
```

---

## ⚡ Encoding vs Hashing vs Encryption — The Critical Distinction

> 🎮 **The most asked interview question in security.** If you remember only ONE thing from this tutorial, let it be this table:

| Property | Encoding | Hashing | Encryption |
|----------|----------|---------|-----------|
| **Purpose** | Data format conversion | Integrity verification | Confidentiality |
| **Reversible?** | ✅ Yes (by anyone) | ❌ No (one-way) | ✅ Yes (with key only) |
| **Needs a key?** | ❌ No | ❌ No | ✅ Yes |
| **Security?** | ❌ NONE | ✅ Integrity only | ✅ Confidentiality |
| **Examples** | Base64, URL, Hex | SHA-256, bcrypt | AES, RSA |
| **Use case** | Transport data | Verify data, store passwords | Protect secrets |

### ❌ The #1 Mistake Engineers Make

```java
// 🚨 CRITICAL BUG — This is NOT security!
String encoded = Base64.getEncoder().encodeToString(password.getBytes());
// stored in DB: "cGFzc3dvcmQxMjM=" 
// This is just Base64 ENCODING — anyone can decode it!
// $ echo "cGFzc3dvcmQxMjM=" | base64 -d
// → password123   ← 🤦 OOPS

// ✅ CORRECT — Use proper password hashing
String hashed = BCrypt.hashpw(password, BCrypt.gensalt(12));
// stored in DB: "$2a$12$LJ3m4sMKfyE6...." — cannot be reversed!
```

### 🎮 Quick Quiz: Classify Each Operation

| Operation | Encoding / Hashing / Encryption? |
|-----------|----------------------------------|
| `btoa("hello")` in JavaScript | <details><summary>Reveal</summary>**Encoding** (Base64) — No security, reversible by anyone</details> |
| `SHA256("hello")` → fixed output | <details><summary>Reveal</summary>**Hashing** — One-way, can't get "hello" back from the hash</details> |
| `AES.encrypt("hello", key)` | <details><summary>Reveal</summary>**Encryption** — Reversible only with the key</details> |
| `URLEncoder.encode("hello world")` → "hello+world" | <details><summary>Reveal</summary>**Encoding** — Format conversion, no security</details> |
| `bcrypt("password123")` | <details><summary>Reveal</summary>**Hashing** (password-specific) — Slow, salted, one-way</details> |
| `jwt.sign(payload, secretKey)` | <details><summary>Reveal</summary>**Digital Signature** (uses HMAC or RSA under the hood)</details> |

---

## 🔨 Hashing Deep Dive

### What is Hashing?

```
INPUT (any size)    →    HASH FUNCTION    →    OUTPUT (fixed size)
"hello"             →       SHA-256       →    "2cf24dba5fb0a30e..."  (64 hex chars)
"hello world"       →       SHA-256       →    "b94d27b9934d3e08..."  (64 hex chars)
War and Peace       →       SHA-256       →    "8e2c5a12f4b3c..."    (64 hex chars)
(1,225 pages)                                   (still 64 hex chars!)

KEY PROPERTIES:
  1. Deterministic: Same input → ALWAYS same output
  2. Fixed size: Output is always the same length regardless of input
  3. One-way: Cannot reverse hash back to input (computationally infeasible)
  4. Avalanche effect: Tiny change in input → completely different output
  5. Collision resistant: Hard to find two inputs with the same hash
```

### The Avalanche Effect (Why Hashing is Powerful)

```
Input: "hello"         → SHA-256 → 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
Input: "Hello"         → SHA-256 → 185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969
Input: "hello."        → SHA-256 → 5891b5b522d5df086d0ff0b110fbd9d21bb4fc7163af34d08286a2e846f6be03

☝️ Change ONE character → completely different hash
   An attacker can't predict how a small change affects the hash
   This makes brute-force the only option (and slow hashes make brute-force infeasible)
```

### General-Purpose Hashes vs. Password Hashes

```
╔══════════════════════════════════════════════════════════════════╗
║  CRITICAL DISTINCTION — Get this right in interviews!            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  GENERAL-PURPOSE (SHA-256, SHA-3):                               ║
║    → Designed to be FAST (billions/sec on GPU)                   ║
║    → Use for: file integrity, checksums, data deduplication      ║
║    → ❌ NEVER for passwords (fast = easy to brute-force)        ║
║                                                                  ║
║  PASSWORD-SPECIFIC (bcrypt, Argon2id, scrypt):                   ║
║    → Designed to be SLOW (hundreds of ms per hash)               ║
║    → Built-in salt (prevents rainbow tables)                     ║
║    → Configurable cost (increase as hardware gets faster)        ║
║    → ✅ ALWAYS use for passwords                                ║
║                                                                  ║
║  WHY SLOW MATTERS:                                               ║
║    SHA-256:  10 billion hashes/sec on modern GPU                 ║
║    bcrypt:   ~70 hashes/sec with cost 12                         ║
║                                                                  ║
║    8-char password (lowercase + digits):                         ║
║      SHA-256: Cracked in ~0.3 seconds                            ║
║      bcrypt:  Cracked in ~29 years                               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Password Hashing — The Right Way (Java)

```java
// ❌ TERRIBLE: Plain text
userRepository.save(new User(email, password));

// ❌ BAD: MD5 (too fast, no salt, broken collision resistance)
String hash = DigestUtils.md5Hex(password);

// ❌ BAD: SHA-256 (too fast, no built-in salt)
String hash = DigestUtils.sha256Hex(password);

// ❌ BAD: SHA-256 + salt (still too fast — GPU cracks it)
String hash = DigestUtils.sha256Hex(salt + password);

// ✅ GOOD: bcrypt (slow, salted, adaptive)
String hash = BCrypt.hashpw(password, BCrypt.gensalt(12));
// Cost 12 ≈ 250ms per hash — fast enough for login, too slow for brute force

// ✅ BETTER: Argon2id (winner of Password Hashing Competition 2015)
Argon2PasswordEncoder encoder = new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
String hash = encoder.encode(password);
// Memory-hard — can't parallelize on GPUs as easily
```

### Salt: Why It Matters

```
WITHOUT SALT:
  password123 → SHA-256 → ef92b778ba... (same for ALL users with this password)
  
  Rainbow table attack:
    Pre-compute hash for top 10M passwords → instant lookup
    If hash "ef92b778ba..." found → password is "password123"
    Cracks ALL users with same password in ONE lookup

WITH SALT (random per user):
  User A: password123 + "x8Kj2m" → bcrypt → "$2a$12$x8Kj2m.LJ3m4..."
  User B: password123 + "p9Qr5n" → bcrypt → "$2a$12$p9Qr5n.Xk7n2..."
  
  ☝️ Same password → DIFFERENT hashes (because different salts)
  Rainbow tables are useless — each user needs individual brute-force
  bcrypt includes the salt IN the hash output (no separate storage needed)
```

---

## 🔑 Symmetric Encryption

> 🎮 **Analogy**: A lockbox with ONE key. Alice puts a message in, locks it. Bob has a copy of the SAME key, unlocks it. Problem: How do you give Bob the key without Eve intercepting it?

### How It Works

```
SYMMETRIC ENCRYPTION:

  Same key for encryption AND decryption
  
  [Plaintext] ──key──▶ [ENCRYPT] ──▶ [Ciphertext] ──key──▶ [DECRYPT] ──▶ [Plaintext]
  "Transfer $100"                     "x7Hf9k2Lm..."                       "Transfer $100"
  
  Algorithms:
    AES-256-GCM  ← Gold standard (used by governments, banks, everyone)
    ChaCha20-Poly1305 ← Alternative (better on devices without AES hardware)
    
  Key sizes:
    AES-128: Good (2^128 possible keys ≈ cannot be brute-forced)
    AES-256: Better (2^256 possible keys ≈ heat death of universe first)
```

### AES Modes — GCM vs CBC (Interview Favorite!)

```
╔══════════════════════════════════════════════════════════════════╗
║  AES MODES — Not all are created equal!                          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ECB (Electronic Codebook) — ❌ NEVER USE                       ║
║    Same plaintext block → same ciphertext block                  ║
║    Patterns in data are visible in ciphertext                    ║
║    Famous "ECB Penguin" — encrypt an image, penguin still visible║
║                                                                  ║
║  CBC (Cipher Block Chaining) — ⚠️ Legacy, needs HMAC           ║
║    Each block XORed with previous ciphertext block               ║
║    Provides confidentiality but NOT integrity                    ║
║    Vulnerable to padding oracle attacks (lucky13)                ║
║    If used: MUST add HMAC for integrity (Encrypt-then-MAC)       ║
║                                                                  ║
║  GCM (Galois/Counter Mode) — ✅ RECOMMENDED                    ║
║    Provides BOTH confidentiality AND integrity (AEAD)            ║
║    Authentication tag built in — detects any tampering           ║
║    Parallelizable — fast on modern hardware                      ║
║    Standard: AES-256-GCM (TLS 1.3, AWS, Google Cloud)           ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Java Implementation — AES-256-GCM

```java
public class AesGcmEncryption {
    private static final int KEY_SIZE = 256;
    private static final int IV_SIZE = 12;  // 96 bits recommended for GCM
    private static final int TAG_SIZE = 128; // Authentication tag

    // Generate a secure random key
    public static SecretKey generateKey() throws NoSuchAlgorithmException {
        KeyGenerator keyGen = KeyGenerator.getInstance("AES");
        keyGen.init(KEY_SIZE, SecureRandom.getInstanceStrong());
        return keyGen.generateKey();
    }

    // Encrypt with AES-256-GCM
    public static byte[] encrypt(byte[] plaintext, SecretKey key) throws Exception {
        // ✅ Generate fresh IV for EVERY encryption (CRITICAL!)
        byte[] iv = new byte[IV_SIZE];
        SecureRandom.getInstanceStrong().nextBytes(iv);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(TAG_SIZE, iv));
        
        byte[] ciphertext = cipher.doFinal(plaintext);
        
        // Prepend IV to ciphertext (IV is not secret, but must be unique)
        byte[] result = new byte[IV_SIZE + ciphertext.length];
        System.arraycopy(iv, 0, result, 0, IV_SIZE);
        System.arraycopy(ciphertext, 0, result, IV_SIZE, ciphertext.length);
        return result;
    }

    // Decrypt with AES-256-GCM
    public static byte[] decrypt(byte[] encryptedData, SecretKey key) throws Exception {
        // Extract IV from the beginning
        byte[] iv = Arrays.copyOfRange(encryptedData, 0, IV_SIZE);
        byte[] ciphertext = Arrays.copyOfRange(encryptedData, IV_SIZE, encryptedData.length);

        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(TAG_SIZE, iv));
        
        return cipher.doFinal(ciphertext); // Throws if tampered (integrity check!)
    }
}
```

### ⚠️ Critical Mistakes with Symmetric Encryption

```java
// ❌ MISTAKE 1: Reusing IV/Nonce (catastrophic for GCM!)
byte[] iv = "hardcoded_iv".getBytes(); // NEVER DO THIS
// With GCM, reusing IV with same key → attacker can recover plaintext

// ❌ MISTAKE 2: Using ECB mode
Cipher.getInstance("AES/ECB/PKCS5Padding"); // Patterns visible in ciphertext!

// ❌ MISTAKE 3: Hardcoding the key
private static final String KEY = "mySecretKey12345"; // 🚨 IN SOURCE CODE!

// ❌ MISTAKE 4: Using DES or 3DES (deprecated, too short key)
Cipher.getInstance("DES/CBC/PKCS5Padding"); // 56-bit key = crackable

// ✅ CORRECT: AES-256-GCM, random IV per operation, key from secrets manager
```

---

## 🔓 Asymmetric Encryption

> 🎮 **Analogy**: A mailbox. Anyone can DROP a letter in (public key = mail slot), but only the owner with the mailbox key can READ the letters (private key = mailbox key). You can publish your mail slot address to the world, but the reading key stays with you.

### How It Works

```
ASYMMETRIC ENCRYPTION (Public Key Cryptography):

  TWO keys: Public (shared with everyone) + Private (kept secret)
  
  ENCRYPTION (for confidentiality):
    [Plaintext] ──public key──▶ [ENCRYPT] ──▶ [Ciphertext]
    [Ciphertext] ──private key──▶ [DECRYPT] ──▶ [Plaintext]
    
    Anyone can encrypt with public key → only private key holder can decrypt
  
  SIGNING (for authentication):
    [Message] ──private key──▶ [SIGN] ──▶ [Signature]
    [Message + Signature] ──public key──▶ [VERIFY] ──▶ Valid/Invalid
    
    Only private key holder can sign → anyone with public key can verify

  ALGORITHMS:
    RSA-2048/4096     ← Most common, well-understood
    ECDSA (P-256)     ← Smaller keys, same security (used in Bitcoin, TLS)
    Ed25519           ← Modern, fast, resistant to timing attacks
    X25519            ← Key exchange (used in Signal, TLS 1.3)
```

### RSA vs. Elliptic Curve — When to Use Which

| Property | RSA-2048 | ECDSA P-256 | Ed25519 |
|----------|----------|-------------|---------|
| Key size | 2048 bits | 256 bits | 256 bits |
| Security level | 112-bit | 128-bit | 128-bit |
| Sign speed | Slower | Fast | Fastest |
| Verify speed | Fast | Slower | Fast |
| Key generation | Slow | Fast | Fast |
| Use case | Legacy systems, certificates | General purpose | Modern apps, SSH keys |
| Used by | Traditional CAs, PGP | Bitcoin, TLS 1.2+ | GitHub, Signal, WireGuard |

### The Key Exchange Problem (Solved!)

```
THE PROBLEM:
  Alice and Bob want to communicate securely over an insecure channel.
  They need a shared secret key for AES encryption.
  But how do they share the key without Eve intercepting it?

DIFFIE-HELLMAN KEY EXCHANGE (the beautiful solution):
  
  1. Alice and Bob agree on public parameters (p, g) — anyone can see these
  2. Alice picks secret 'a', computes A = g^a mod p, sends A publicly
  3. Bob picks secret 'b', computes B = g^b mod p, sends B publicly
  4. Alice computes: shared_secret = B^a mod p
  5. Bob computes:   shared_secret = A^b mod p
  6. BOTH get the SAME shared_secret! (math magic: g^(ab) = g^(ba))
  7. Eve saw A and B publicly, but can't compute g^(ab) without a or b
  
  This is used in EVERY TLS handshake you've ever done!
  
  Modern version: ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
    → Used in TLS 1.3
    → "Ephemeral" = new keys per session = perfect forward secrecy
```

---

## ✍️ Digital Signatures

> 🎮 **Analogy**: A wax seal on a letter in medieval times. Only the king has the signet ring (private key). Anyone can verify the seal pattern matches the king's ring (public key). If the seal is broken, you know the letter was tampered with.

### How Digital Signatures Work

```
SIGNING (sender proves authorship):
  
  1. Hash the message: hash = SHA-256(message)
  2. Encrypt hash with private key: signature = RSA_SIGN(hash, private_key)
  3. Send: message + signature
  
VERIFYING (receiver checks authenticity):
  
  1. Hash the received message: computed_hash = SHA-256(message)
  2. Decrypt signature with public key: claimed_hash = RSA_VERIFY(signature, public_key)
  3. Compare: computed_hash == claimed_hash?
     → YES: Message is authentic and unmodified ✅
     → NO:  Message was tampered with or not from claimed sender ❌

WHAT SIGNATURES GUARANTEE:
  ✅ Authentication: Only the private key holder could have signed it
  ✅ Integrity: Any modification invalidates the signature
  ✅ Non-repudiation: Signer can't deny signing (their private key was used)
```

### Real-World Uses of Digital Signatures

```
YOU USE DIGITAL SIGNATURES EVERY DAY:

  🌐 HTTPS/TLS:      Server certificate signed by CA → browser trusts it
  📦 Software:       macOS/Windows verify app signatures before installing
  🪙 Blockchain:     Every Bitcoin transaction signed by wallet's private key
  📧 Email:          DKIM signature proves email came from claimed domain
  🐳 Docker:         Container images signed → verify they haven't been tampered
  📱 Mobile Apps:    APK/IPA signed by developer → app store verifies
  🔑 JWT:            RS256 JWT = JSON payload signed with RSA private key
  📄 Git commits:    GPG-signed commits prove who wrote the code
```

### Java — Digital Signature Example

```java
public class DigitalSignatureExample {
    
    // Generate RSA key pair
    public static KeyPair generateKeyPair() throws Exception {
        KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
        gen.initialize(2048, SecureRandom.getInstanceStrong());
        return gen.generateKeyPair();
    }
    
    // Sign a message
    public static byte[] sign(byte[] message, PrivateKey privateKey) throws Exception {
        Signature signer = Signature.getInstance("SHA256withRSA");
        signer.initSign(privateKey);
        signer.update(message);
        return signer.sign();
    }
    
    // Verify a signature
    public static boolean verify(byte[] message, byte[] signature, PublicKey publicKey) 
            throws Exception {
        Signature verifier = Signature.getInstance("SHA256withRSA");
        verifier.initVerify(publicKey);
        verifier.update(message);
        return verifier.verify(signature);
    }
    
    // Usage
    public static void main(String[] args) throws Exception {
        KeyPair keys = generateKeyPair();
        
        byte[] message = "Transfer $100 to Bob".getBytes();
        byte[] signature = sign(message, keys.getPrivate());
        
        // Anyone with public key can verify
        boolean isValid = verify(message, signature, keys.getPublic());
        System.out.println("Signature valid: " + isValid); // true
        
        // If message is tampered...
        byte[] tampered = "Transfer $10000 to Eve".getBytes();
        boolean isTampered = verify(tampered, signature, keys.getPublic());
        System.out.println("Tampered valid: " + isTampered); // false! ❌
    }
}
```

---

## 🗝️ Key Management

> 🎮 **The lock is only as strong as where you keep the key.** The most sophisticated encryption is worthless if the key is in your Git repo.

### Key Management Hierarchy

```
╔══════════════════════════════════════════════════════════════════╗
║               KEY MANAGEMENT HIERARCHY                           ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Level 4: MASTER KEY (KEK — Key Encryption Key)                  ║
║    → Stored in HSM (Hardware Security Module)                    ║
║    → Never leaves the hardware                                   ║
║    → Used only to encrypt other keys                             ║
║           │                                                      ║
║  Level 3: DATA ENCRYPTION KEYS (DEK)                             ║
║    → Encrypted by the master key (envelope encryption)           ║
║    → Rotated frequently (30-90 days)                             ║
║    → One per service/purpose                                     ║
║           │                                                      ║
║  Level 2: SESSION KEYS                                           ║
║    → Generated per TLS session                                   ║
║    → Ephemeral (exist only during communication)                 ║
║    → Perfect forward secrecy                                     ║
║           │                                                      ║
║  Level 1: DERIVED KEYS                                           ║
║    → Application-specific keys derived from DEK                  ║
║    → Using KDF (Key Derivation Function)                         ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Envelope Encryption (How AWS/GCP/Azure Does It)

```
PROBLEM: You want to encrypt 10TB of data with AES-256.
         Storing the AES key securely is the challenge.

SOLUTION: Envelope Encryption

  1. Generate Data Encryption Key (DEK) — random AES-256 key
  2. Encrypt your 10TB of data with the DEK
  3. Encrypt the DEK itself with a Master Key (KEK) stored in KMS/HSM
  4. Store: [Encrypted Data] + [Encrypted DEK]
  5. Discard plaintext DEK from memory

  To decrypt:
  1. Send encrypted DEK to KMS/HSM → get plaintext DEK back
  2. Use plaintext DEK to decrypt data
  3. Discard plaintext DEK from memory

  Why?
  → Data never leaves your server (only the tiny DEK goes to KMS)
  → Master key NEVER leaves the HSM (hardware protection)
  → Easy to rotate: re-encrypt DEK with new master key (don't re-encrypt all data)
  → Audit trail: KMS logs every time the master key is used
```

### Key Rotation — Why and How

```
WHY ROTATE KEYS:
  → Limits exposure if a key is compromised (old data with old key)
  → Compliance (PCI-DSS requires annual key rotation minimum)
  → Reduces the value of stolen encrypted data
  → Cryptographic hygiene — keys used many times are more analyzable

HOW TO ROTATE WITHOUT DOWNTIME:
  
  Phase 1: [Key-v1 active] 
    → All encrypt and decrypt use Key-v1
    
  Phase 2: [Key-v2 active, Key-v1 decrypt-only]
    → New data encrypted with Key-v2
    → Old data still readable with Key-v1
    
  Phase 3: [Key-v2 active, re-encrypt old data]
    → Background job re-encrypts old data with Key-v2
    → Key-v1 still available for any missed data
    
  Phase 4: [Key-v2 only, Key-v1 destroyed]
    → All data now encrypted with Key-v2
    → Key-v1 securely deleted
```

---

## 💀 Common Cryptographic Mistakes

### The Hall of Crypto Shame

```
╔══════════════════════════════════════════════════════════════════╗
║  MISTAKE #1: Rolling Your Own Crypto                             ║
║  "I'll write my own encryption algorithm — how hard can it be?"  ║
║  Reality: Bruce Schneier: "Anyone can invent a cipher that       ║
║  they themselves cannot break." Use AES, not your own XOR magic. ║
╠══════════════════════════════════════════════════════════════════╣
║  MISTAKE #2: Using MD5 or SHA-1 for anything security-related    ║
║  MD5: Collisions found in 2004. Dead. Buried.                    ║
║  SHA-1: Collision demonstrated by Google in 2017. Don't use.     ║
╠══════════════════════════════════════════════════════════════════╣
║  MISTAKE #3: Hardcoding keys/IVs                                 ║
║  private static final String KEY = "0123456789abcdef";           ║
║  → Found in decompiled APK/JAR in 5 minutes                     ║
╠══════════════════════════════════════════════════════════════════╣
║  MISTAKE #4: Using encryption for passwords                      ║
║  "I'll encrypt passwords with AES so I can decrypt them later"   ║
║  → If attacker gets the key, ALL passwords exposed instantly     ║
║  → Use HASHING (bcrypt). You never need to "decrypt" a password. ║
╠══════════════════════════════════════════════════════════════════╣
║  MISTAKE #5: Random without SecureRandom                         ║
║  new Random().nextBytes(iv); // ❌ Predictable!                  ║
║  SecureRandom.getInstanceStrong().nextBytes(iv); // ✅ Secure    ║
╠══════════════════════════════════════════════════════════════════╣
║  MISTAKE #6: Not using authenticated encryption                  ║
║  AES-CBC without HMAC → attacker can modify ciphertext           ║
║  AES-GCM → built-in integrity check, detects tampering           ║
╠══════════════════════════════════════════════════════════════════╣
║  MISTAKE #7: Comparing hashes with == instead of constant-time   ║
║  if (hash.equals(expectedHash)) // ❌ Timing attack!             ║
║  if (MessageDigest.isEqual(a, b)) // ✅ Constant-time            ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🎮 Gamification: Crypto Puzzle Room

### 🧩 Puzzle 1: The Broken Login System

```
A startup stores passwords like this:
  password: "admin123"
  stored:   "YWRtaW4xMjM="

Your colleague says: "Don't worry, it's encrypted."
```

<details>
<summary>🔍 What's wrong? How would you fix it?</summary>

**Problem**: That's Base64 ENCODING, not encryption or hashing!
```bash
echo "YWRtaW4xMjM=" | base64 -d
# Output: admin123 ← Zero security!
```

**Fix**: Use bcrypt or Argon2id:
```java
String hash = BCrypt.hashpw("admin123", BCrypt.gensalt(12));
// "$2a$12$..." → cannot be reversed, even by the database admin
```
</details>

### 🧩 Puzzle 2: The JWT Vulnerability

```
A JWT header says: {"alg": "none", "typ": "JWT"}
The server accepts this token and grants admin access.
```

<details>
<summary>🔍 What attack is this? How do you prevent it?</summary>

**Attack**: Algorithm confusion / "none" algorithm attack.
The attacker changed the algorithm to "none" (no signature required), and the server blindly trusts the `alg` header.

**Prevention**:
```java
// ❌ BAD: Trust the alg header from the token
Algorithm algorithm = getAlgorithmFromTokenHeader(token); 

// ✅ GOOD: Server ENFORCES the expected algorithm
JWTVerifier verifier = JWT.require(Algorithm.RSA256(publicKey))
    .build();
// This will REJECT any token not signed with RS256
```
</details>

### 🧩 Puzzle 3: Spot All the Mistakes

```java
public class SecureStorage {
    private static final String KEY = "mySecretKey12345"; // 128-bit
    private static final String IV = "1234567890123456";  // 16 bytes
    
    public String encrypt(String data) {
        Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
        cipher.init(Cipher.ENCRYPT_MODE, new SecretKeySpec(KEY.getBytes(), "AES"));
        return Base64.encode(cipher.doFinal(data.getBytes()));
    }
    
    public String hashPassword(String password) {
        return DigestUtils.md5Hex(password);
    }
}
```

<details>
<summary>🔍 How many mistakes can you find? (There are 7)</summary>

1. **Hardcoded key** — Should come from secrets manager
2. **Hardcoded IV** — Should be random per encryption (though IV isn't even used with ECB!)
3. **ECB mode** — Patterns visible in ciphertext; use GCM
4. **No authentication** — Should use AEAD (AES-GCM) for integrity
5. **MD5 for passwords** — Broken, too fast; use bcrypt/Argon2id
6. **No salt** — MD5 without salt = rainbow table attack
7. **128-bit key from string** — Use `KeyGenerator` with `SecureRandom`, not a literal string

**Fixed version**: Use `AES/GCM/NoPadding`, `SecureRandom` for IV, `KeyGenerator` for key, `BCrypt` for passwords.
</details>

### 🧩 Puzzle 4: The Timing Attack

```java
public boolean verifyApiKey(String provided, String expected) {
    return provided.equals(expected);
}
```

<details>
<summary>🔍 What's the vulnerability? How does an attacker exploit it?</summary>

**Vulnerability**: Timing side-channel attack.

`String.equals()` returns `false` at the first mismatched character. An attacker can measure response time:
- "a000000000" → fails instantly (first char wrong) → ~1ms
- "c000000000" → fails slightly slower (first char matches) → ~1.1ms
- "co00000000" → even slower (two chars match) → ~1.2ms
- Continue until entire key is extracted one character at a time!

**Fix**: Use constant-time comparison:
```java
public boolean verifyApiKey(String provided, String expected) {
    return MessageDigest.isEqual(
        provided.getBytes(StandardCharsets.UTF_8),
        expected.getBytes(StandardCharsets.UTF_8)
    );
    // Always compares ALL bytes regardless of where mismatch is
}
```
</details>

---

## 🎯 Interview Q&A — 25 Must-Know Questions

### Fundamentals (Q1-Q10)

**Q1: What's the difference between encoding, hashing, and encryption?**

> **A**: Encoding is reversible format conversion (Base64) — NO security. Hashing is one-way transformation (SHA-256, bcrypt) — provides integrity, used for passwords and checksums. Encryption is reversible WITH a key (AES, RSA) — provides confidentiality. Critical mistake: using Base64 "encryption" for passwords. Base64 is not security.

**Q2: Why can't you use SHA-256 for password storage?**

> **A**: SHA-256 is designed to be fast — GPUs can compute 10 billion hashes/second. An 8-character password is cracked in under a second. Password hashes (bcrypt, Argon2id) are intentionally slow (~250ms each), making brute-force infeasible. They also include built-in salts to prevent rainbow table attacks.

**Q3: What is a salt? Why is it necessary?**

> **A**: A salt is a random value unique to each password, mixed in before hashing. Without salt: identical passwords produce identical hashes → one rainbow table cracks all of them. With salt: same password produces different hashes per user → attacker must individually brute-force each hash. bcrypt generates and stores the salt automatically within the hash output.

**Q4: Explain symmetric vs asymmetric encryption.**

> **A**: Symmetric uses ONE key for both encrypt and decrypt (AES) — fast but requires secure key exchange. Asymmetric uses a KEY PAIR: public key encrypts, private key decrypts (RSA, ECC) — solves key exchange but 100-1000x slower. In practice, we combine both: asymmetric for key exchange, then symmetric for bulk data (this is how TLS works).

**Q5: What is AES-GCM and why is it preferred over AES-CBC?**

> **A**: AES-GCM is an Authenticated Encryption with Associated Data (AEAD) mode. It provides both confidentiality AND integrity in one operation. AES-CBC provides only confidentiality — if an attacker modifies ciphertext, decryption succeeds with corrupted data (and padding oracle attacks are possible). GCM includes an authentication tag that detects any tampering.

**Q6: What is Perfect Forward Secrecy? Why does it matter?**

> **A**: PFS means that if a long-term private key is compromised in the future, past communications remain secure. Achieved using ephemeral keys — each TLS session generates a fresh key pair (ECDHE). Without PFS: attacker records encrypted traffic, later steals the server's private key, decrypts ALL past traffic. With PFS: each session's key is gone after the session ends.

**Q7: How does TLS 1.3 handshake work?**

> **A**: TLS 1.3 is a 1-RTT (one round-trip) handshake: (1) Client sends ClientHello with supported cipher suites AND key share, (2) Server responds with ServerHello, its key share, and its certificate (all encrypted after key exchange), (3) Both derive the shared secret using ECDHE. Improvements over TLS 1.2: faster (1-RTT vs 2-RTT), removed insecure algorithms (RSA key exchange, CBC, RC4), mandatory PFS.

**Q8: What is HMAC and when would you use it?**

> **A**: HMAC (Hash-based Message Authentication Code) is a MAC using a cryptographic hash and a secret key. It proves both integrity (not tampered) and authenticity (came from someone with the key). Use cases: API webhook verification (Stripe uses HMAC-SHA256), JWT HS256 signatures, session token validation, file integrity verification. Unlike a plain hash, HMAC requires the secret key — only holders can generate or verify it.

**Q9: What are the risks of using `Math.random()` or `Random` for security-sensitive operations?**

> **A**: `Math.random()` and `java.util.Random` use pseudo-random number generators (PRNGs) seeded from predictable values. An attacker who observes outputs can predict future values, compromising tokens, keys, or IVs. For security: use `SecureRandom.getInstanceStrong()` which draws from OS entropy sources (`/dev/urandom` on Linux). Never use non-cryptographic RNGs for session IDs, tokens, keys, IVs, or salts.

**Q10: What is envelope encryption? Why do cloud providers use it?**

> **A**: Envelope encryption wraps (encrypts) data encryption keys (DEKs) with a master key (KEK) stored in a Hardware Security Module (HSM). Benefits: (1) Large data never leaves your server (only the small DEK goes to KMS), (2) Master key never leaves hardware (FIPS 140-2 compliance), (3) Key rotation is fast — re-encrypt the DEK, not all data, (4) Audit trail on master key usage. Used by AWS KMS, GCP CMEK, Azure Key Vault.

### Applied Cryptography (Q11-Q20)

**Q11: How does HTTPS actually protect your data?**

> **A**: HTTPS = HTTP over TLS. Protection layers: (1) TLS handshake establishes identity (server certificate signed by trusted CA) and negotiates session keys via ECDHE, (2) All data encrypted with symmetric key (AES-256-GCM), (3) Integrity protected by GCM authentication tag, (4) PFS ensures past sessions stay secure even if server key leaks. What it does NOT protect: metadata (which server you're connecting to is visible via SNI — unless using Encrypted Client Hello).

**Q12: What's a man-in-the-middle attack? How does TLS prevent it?**

> **A**: MITM: Attacker intercepts communication, impersonates both parties. Alice thinks she's talking to Bob, but is actually talking to Eve who relays messages. TLS prevents it via: (1) Server presents certificate signed by trusted CA, (2) Client verifies certificate chain to trusted root, (3) Certificate binds server's public key to domain name, (4) ECDHE key exchange — even if Eve intercepts the exchange, she can't derive the shared secret without the server's private key.

**Q13: What's the difference between a digital signature and a MAC?**

> **A**: MAC (HMAC) uses a shared secret key — both parties can generate AND verify. It proves authenticity between two parties who share the key but doesn't provide non-repudiation (either could have generated it). Digital signature uses asymmetric keys — only the private key holder can sign, anyone with the public key can verify. Provides non-repudiation: signer can't deny creating the signature. Use MAC for internal service-to-service auth; use signatures for public verification (certificates, JWTs with RSA).

**Q14: How does bcrypt work internally?**

> **A**: bcrypt: (1) Generates 16-byte random salt, (2) Derives encryption key from password + salt using Blowfish key schedule (expensive), (3) Encrypts the string "OrpheanBeholderScryDoubt" 64 times with derived key, (4) Output format: `$2a$12$[22-char salt][31-char hash]`. The cost factor (12) means 2^12 = 4096 iterations of key schedule. Increasing cost by 1 doubles computation time. Why Blowfish? Its key schedule is inherently expensive — can't be optimized on GPUs easily.

**Q15: What is a rainbow table attack? How do salts prevent it?**

> **A**: A rainbow table is a precomputed lookup from hash → plaintext. Attacker hashes all common passwords once, then does instant lookup against stolen hashes. Salt prevents this because each user has a unique salt: `hash(password + unique_salt)`. The rainbow table would need to precompute ALL passwords × ALL possible salts — computationally infeasible. Without salt, one table cracks all users with the same password simultaneously.

**Q16: What is key stretching? When would you use it?**

> **A**: Key stretching converts a low-entropy input (like a password) into a cryptographic key by applying a computationally expensive derivation function. Examples: PBKDF2 (HMAC iterated 600,000+ times), Argon2id (memory-hard + CPU-hard). Use when: deriving encryption keys from passwords (e.g., encrypting a file with a password), storing passwords (bcrypt is essentially key stretching + encryption).

**Q17: What are the security implications of JWT RS256 vs HS256?**

> **A**: HS256 (HMAC-SHA256): Symmetric — same secret key signs and verifies. If any service needs to verify tokens, ALL services need the secret. Compromise of any service exposes the signing key. RS256 (RSA-SHA256): Asymmetric — private key signs, public key verifies. Only the auth server needs the private key. Microservices only need the public key (which is safe to distribute). Compromise of a microservice doesn't compromise token generation. **For distributed systems: always RS256/ES256.**

**Q18: How would you implement end-to-end encryption (like WhatsApp)?**

> **A**: Signal Protocol approach: (1) Each user generates identity key pair + pre-key bundles (published to server), (2) When Alice messages Bob: fetch Bob's pre-key → perform X3DH key agreement → derive shared secret, (3) Use Double Ratchet algorithm: new key per message, forward secrecy + break-in recovery, (4) Server only sees encrypted blobs — can't read content, (5) Group chats: sender encrypts separately for each member (sender keys for efficiency). Critical: server never has plaintext or decryption keys.

**Q19: What is a certificate chain and why do browsers trust it?**

> **A**: Certificate chain: Server cert → Intermediate CA → Root CA. Browsers/OS ship with pre-installed trusted Root CAs (~100-200). When you visit a site: (1) Server presents its certificate (signed by intermediate CA), (2) Browser traces the chain up to a Root CA it trusts, (3) If chain is valid and unrevoked → trust. If a CA is compromised (DigiNotar 2011), browsers remove it. Certificate Transparency logs make fraudulent certificates detectable.

**Q20: What is a padding oracle attack? How does it affect AES-CBC?**

> **A**: In CBC mode with PKCS#7 padding, if the server reveals whether padding is valid (different error messages, timing), an attacker can decrypt ciphertext without the key by manipulating blocks and observing responses. Each byte recovered with ~128 requests on average. Prevention: (1) Use AEAD (AES-GCM) — no separate padding, (2) If CBC required: always HMAC before decrypting (Encrypt-then-MAC), verify HMAC first, (3) Never reveal padding-specific errors.

### Architecture & Design (Q21-Q25)

**Q21: How do you choose between symmetric and asymmetric encryption for a system?**

> **A**: Decision tree: Need multiple parties to verify without sharing secrets? → Asymmetric (signatures, JWTs, certificates). Need to encrypt bulk data fast? → Symmetric (AES). Need to exchange keys over insecure channel? → Asymmetric for key exchange, then symmetric for data (hybrid — this is TLS). Need to encrypt data at rest? → Symmetric with envelope encryption (DEK encrypted by asymmetric KEK).

**Q22: Design a secrets rotation system with zero downtime.**

> **A**: (1) Use versioned secrets: `db_password_v3`, `db_password_v4`, (2) Deploy new secret alongside old one (both valid), (3) Update application to use new secret, (4) Verify all instances using new secret, (5) Revoke old secret. Automation: Vault with dynamic secrets — leases that auto-expire. For databases: dual-password support during rotation window. For API keys: grace period where both old and new are accepted.

**Q23: What is crypto agility and why does it matter?**

> **A**: Crypto agility is designing systems so cryptographic algorithms can be swapped without rewriting the application. Matters because algorithms weaken over time (MD5, SHA-1, DES all deprecated). Implementation: (1) Abstract crypto behind interfaces, (2) Store algorithm identifier alongside encrypted data, (3) Support multiple algorithms during migration, (4) Have a rotation plan. Post-quantum threat: RSA and ECC will break when quantum computers arrive — crypto-agile systems can migrate to post-quantum algorithms (CRYSTALS-Kyber, CRYSTALS-Dilithium).

**Q24: How would you detect if your encryption keys have been compromised?**

> **A**: Detection strategies: (1) Audit all key access via KMS/HSM logs — alert on unexpected access patterns, (2) Canary tokens — encrypt known values, alert if decrypted unexpectedly, (3) Monitor for data exfiltration patterns (unusual download volumes), (4) Certificate transparency monitoring for unauthorized certificates, (5) Honeypot keys — fake keys that trigger alerts if used, (6) Regular key usage audits — is this key being used from unexpected IPs/services?

**Q25: Quantum computing threatens current cryptography. How would you prepare?**

> **A**: (1) Inventory all cryptographic algorithms in use (crypto-SBOM), (2) Identify most vulnerable: RSA, ECDSA, DH (broken by Shor's algorithm); AES-256 and SHA-256 weakened but not broken (Grover's algorithm halves effective key length), (3) Plan migration to NIST post-quantum standards: CRYSTALS-Kyber (key exchange), CRYSTALS-Dilithium (signatures), (4) For highly sensitive data encrypted now: "harvest now, decrypt later" is a real threat — consider hybrid encryption (classical + post-quantum) today, (5) Ensure crypto agility in architecture.

---

## 📊 Cryptography Decision Flowchart

```
What do you need to do?
         │
    ┌────┴─────────────────────────────────────┐
    │                                           │
  PROTECT DATA                            VERIFY SOMETHING
    │                                           │
    ├── At rest? → AES-256-GCM (symmetric)     ├── Password? → bcrypt/Argon2id
    │             + envelope encryption         │
    │                                           ├── File unchanged? → SHA-256 checksum
    ├── In transit? → TLS 1.3                   │
    │               (ECDHE + AES-256-GCM)       ├── Message from trusted party? → HMAC
    │                                           │
    └── In use? → Homomorphic encryption        └── Document signed by someone? → Digital signature
                  (still experimental)                (RSA/ECDSA + public key verification)
```

---

## 🔗 What's Next?

| Topic | Link | Why Read It? |
|-------|------|-------------|
| Web Security Q&A | [03_Web_Security_QA.md](./03_Web_Security_QA.md) | Apply crypto knowledge to real attack/defense |
| JWT Deep Dive | [../JWT_Deep_Dive.md](../JWT_Deep_Dive.md) | See crypto in action with token security |
| TLS/SSL/HTTPS | [../TLS_SSL_HTTPS.md](../TLS_SSL_HTTPS.md) | Full TLS handshake deep dive |

---

*[← Security Fundamentals](./01_Security_Fundamentals_QA.md) | [Back to Security Index](../README.md) | [Next: Web Security →](./03_Web_Security_QA.md)*
