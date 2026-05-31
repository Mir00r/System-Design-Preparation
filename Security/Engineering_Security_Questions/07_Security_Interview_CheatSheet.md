# 📋 Security Interview Cheatsheet — Your Last-Night-Before-Interview Reference

> *"You don't need to memorize everything. You need to know the right FRAMEWORKS for thinking about security problems. This cheatsheet gives you the mental models, decision trees, and one-liner answers that will carry you through any security interview question."*

**⏱️ Estimated Time**: 15 minutes (skim) / 45 minutes (study) | **🎯 Difficulty**: 🟢 Quick Reference

---

## ⚡ The 30-Second Security Framework

When asked ANY security question, structure your answer with:

```
1. IDENTIFY the threat (what could go wrong?)
2. CLASSIFY using CIA/STRIDE (which security property is at risk?)
3. DEFEND with layered controls (never just one defense)
4. VALIDATE (how do you verify the defense works?)
```

---

## 🧠 Core Concepts — One-Liner Definitions

| Concept | One-Liner | Interview Tip |
|---------|-----------|---------------|
| **CIA Triad** | Confidentiality (who sees it), Integrity (is it tampered), Availability (can you access it) | Every security decision maps back to one of these |
| **Defense in Depth** | Multiple layered controls — if one fails, others still protect | "I'd implement N defenses because..." |
| **Least Privilege** | Give minimum access needed, nothing more | Database users, IAM roles, API scopes |
| **Zero Trust** | Never trust, always verify — even inside the network | "Assume the network is already compromised" |
| **Fail Secure** | When something breaks, deny access (don't open up) | If auth service is down → block, don't allow-all |
| **Security by Design** | System is secure even if attacker knows everything except the key | Kerckhoffs's Principle (1883) |
| **Blast Radius** | Limit damage when (not if) something is compromised | Per-service secrets, network segmentation |
| **Threat Modeling** | Systematically identify threats during DESIGN (not after deploy) | STRIDE framework |

---

## 🔑 Authentication vs Authorization Quick Reference

```
AUTHENTICATION (AuthN) = "Who are you?"
  → 401 Unauthorized = identity not proven
  → Mechanisms: Password, JWT, mTLS, MFA, Biometrics
  
AUTHORIZATION (AuthZ) = "What can you do?"
  → 403 Forbidden = identity known, but no permission
  → Mechanisms: RBAC, ABAC, ACLs, OAuth2 scopes

MODELS:
  RBAC: Role → Permissions (Admin can delete, User can read)
  ABAC: Attributes → Permissions (if department=finance AND time=business-hours)
  ACL:  Per-resource permission list (File X: Alice=read, Bob=write)
```

---

## 🛡️ Attack ↔ Defense Quick Map

| Attack | What It Does | Defense |
|--------|-------------|---------|
| **SQL Injection** | Modify DB queries via user input | Parameterized queries (PreparedStatement) |
| **XSS (Reflected)** | Inject script via URL, executes in victim's browser | Output encoding + CSP |
| **XSS (Stored)** | Persist script in DB, affects all viewers | Input sanitization + output encoding + CSP |
| **CSRF** | Trick browser into making unwanted authenticated requests | CSRF token + SameSite cookie |
| **SSRF** | Trick server into fetching internal resources | URL allowlist + block internal IPs |
| **IDOR** | Access other users' data by changing IDs | Ownership verification on every request |
| **Brute Force** | Try many passwords/tokens | Rate limiting + account lockout + MFA |
| **MitM** | Intercept communication between two parties | TLS/HTTPS + certificate pinning |
| **Deserialization** | Execute code via crafted serialized objects | Don't deserialize untrusted data; use JSON |
| **Path Traversal** | Access files outside intended directory | Canonicalize path + verify within base dir |
| **Clickjacking** | Invisible iframe overlay tricks user clicks | X-Frame-Options: DENY + frame-ancestors |
| **Session Fixation** | Force victim to use attacker's session ID | Regenerate session ID after login |
| **Log Injection** | Inject fake log entries via input | Sanitize newlines in log messages |
| **ReDoS** | Exponential regex backtracking | Avoid nested quantifiers, set timeout |

---

## 🔐 Cryptography Decision Tree

```
What do you need?
  │
  ├── Store passwords? → bcrypt (cost 12+) or Argon2id
  │                       NEVER: MD5, SHA-256, plaintext, reversible encryption
  │
  ├── Encrypt data at rest? → AES-256-GCM + envelope encryption (KMS for key)
  │                            NEVER: ECB mode, hardcoded keys, DES
  │
  ├── Encrypt in transit? → TLS 1.3 (ECDHE + AES-256-GCM)
  │                          NEVER: SSL 3.0, TLS 1.0/1.1, self-signed in prod
  │
  ├── Verify integrity? → HMAC-SHA256 (if shared secret) or Digital Signature (if public verification)
  │
  ├── Sign tokens? → RS256/ES256 for distributed systems, HS256 only if single service
  │
  └── Generate random? → SecureRandom.getInstanceStrong()
                          NEVER: Math.random(), new Random(), predictable seeds
```

---

## 🌐 Security Headers — Copy-Paste Reference

```
# THE ESSENTIAL HEADERS (implement ALL of these)

Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
Cache-Control: no-store  (for sensitive pages)

# COOKIES
Set-Cookie: session=abc; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=1800
```

---

## ☕ Java/Spring Security Quick Patterns

### Password Storage
```java
// ✅ Store: BCryptPasswordEncoder(12).encode(password)
// ✅ Verify: encoder.matches(raw, stored)
// ❌ NEVER: MD5, SHA-256, Base64, plaintext
```

### SQL Injection Prevention
```java
// ✅ PreparedStatement with ? placeholders
// ✅ JPA @Query with :param
// ✅ Spring Data findByXxx() methods
// ❌ NEVER: "SELECT * FROM x WHERE y = '" + input + "'"
```

### Secure Random
```java
// ✅ SecureRandom.getInstanceStrong()
// ❌ NEVER: new Random(), Math.random()
```

### Input Validation
```java
// ✅ @Valid @RequestBody DTO with @NotNull, @Size, @Pattern, @Positive
// ✅ Allowlist regex: ^[a-zA-Z0-9_]{3,20}$
// ❌ NEVER: Denylist (always has bypasses)
```

### IDOR Prevention
```java
// ✅ @PreAuthorize("@security.isOwner(#id, authentication)")
// ✅ Query: WHERE id = :id AND userId = :currentUser
// ❌ NEVER: Trust path variable without ownership check
```

### Error Handling
```java
// ✅ Generic to client: "An error occurred" + requestId
// ✅ Full details in internal logs only
// ❌ NEVER: Stack traces, SQL errors, file paths to client
```

---

## 📊 STRIDE Threat Model Template

Use this for ANY "design a secure system" interview question:

| STRIDE | Question to Ask | Common Defense |
|--------|----------------|----------------|
| **S**poofing | Can someone pretend to be another user/service? | Authentication (JWT, mTLS, MFA) |
| **T**ampering | Can someone modify data in transit or at rest? | Integrity checks (HMAC, signatures, TLS) |
| **R**epudiation | Can someone deny an action? | Audit logs, digital signatures |
| **I**nfo Disclosure | Can sensitive data leak? | Encryption, access controls, response filtering |
| **D**enial of Service | Can the system be overwhelmed? | Rate limiting, auto-scaling, circuit breakers |
| **E**levation | Can someone gain higher privileges? | RBAC, least privilege, input validation |

---

## 🏢 Company-Specific Focus Areas

| Company | What They Care About | Key Topics |
|---------|---------------------|------------|
| **Google** | Secure design, scalable security | Threat modeling, BeyondCorp (Zero Trust), supply chain |
| **Amazon** | Shared responsibility, IAM | Least privilege, encryption, S3 security, network |
| **Meta** | Privacy, data protection | Data minimization, consent, access controls, E2E encryption |
| **Netflix** | Distributed security, resilience | mTLS, secrets rotation, chaos engineering, zero trust |
| **Stripe** | Payment security, API security | PCI-DSS, tokenization, idempotency, key management |
| **Microsoft** | SDL, threat modeling, crypto | STRIDE, security development lifecycle, code review |
| **Apple** | Privacy, device security | E2E encryption, secure enclave, differential privacy |

---

## 🎯 Top 10 Interview Questions (Quick Answers)

### Q1: "How would you secure a REST API?"
> **Framework**: Authentication → Authorization → Input Validation → Rate Limiting → Encryption → Monitoring

> **Answer**: TLS everywhere, OAuth2/JWT for authentication, RBAC for authorization, input validation (Bean Validation), rate limiting (per-user/per-IP), response field filtering (DTOs), security headers (CORS, CSP), audit logging, WAF at edge.

### Q2: "What is the CIA Triad?"
> Confidentiality (only authorized access), Integrity (not tampered), Availability (accessible when needed). Every security control maps to protecting one or more of these.

### Q3: "Explain SQL Injection and prevention."
> Attacker inserts SQL code via user input that modifies query logic. Only reliable fix: parameterized queries (PreparedStatement) — separates code from data at the database level. Sanitization/escaping always has bypasses.

### Q4: "How do you store passwords?"
> Slow adaptive hash (bcrypt cost 12 or Argon2id), auto-salted, never reversible. NEVER: plaintext, MD5, SHA-256, encryption. Slow = brute-force infeasible. Salt = rainbow tables useless.

### Q5: "What's Zero Trust?"
> Never trust, always verify. Even internal network traffic must be authenticated and authorized. Assumes the network is already breached. Principles: verify explicitly, least-privilege access, assume breach.

### Q6: "CSRF vs XSS — difference?"
> CSRF: tricks browser into making unwanted requests using user's cookies. XSS: injects scripts that run in user's browser. CSRF can't read responses; XSS can do anything the user can do. JWT APIs immune to CSRF (no cookies); both vulnerable to XSS.

### Q7: "How does TLS work?"
> 1-RTT handshake (TLS 1.3): Client sends supported ciphers + key share → Server responds with cert + key share → Both derive shared secret (ECDHE) → Data encrypted with AES-256-GCM. Provides: authentication (certificates), confidentiality (encryption), integrity (AEAD), forward secrecy (ephemeral keys).

### Q8: "What would you do if you found a critical vulnerability in production?"
> Assess scope → Contain (block attack vector immediately) → Communicate (alert team, management) → Fix (deploy patch) → Recover (reset credentials, restore data) → Review (post-mortem, improve detection).

### Q9: "Encoding vs Hashing vs Encryption?"
> Encoding: reversible without key (Base64) — NOT security. Hashing: one-way, irreversible (SHA-256, bcrypt) — integrity/passwords. Encryption: reversible WITH key (AES, RSA) — confidentiality. Critical: Base64 is NOT encryption.

### Q10: "How do you handle secrets in a distributed system?"
> Never in code/git. Use secrets manager (Vault/AWS SM), inject at runtime via env vars or mounted volumes, rotate regularly, scope per-service (least privilege), audit access, revoke immediately on exposure.

---

## 🚦 Common Mistakes to Avoid in Interviews

```
❌ DON'T say: "Use a WAF to prevent SQL injection"
✅ DO say: "Use parameterized queries to prevent SQL injection; WAF is defense-in-depth"

❌ DON'T say: "Encrypt passwords with AES"
✅ DO say: "Hash passwords with bcrypt — you never need to reverse a password"

❌ DON'T say: "Use Base64 to secure data"
✅ DO say: "Base64 is encoding, not security — use AES-256-GCM for encryption"

❌ DON'T say: "Block localhost to prevent SSRF"
✅ DO say: "Resolve DNS first, check against IP allowlist, and block all internal ranges"

❌ DON'T say: "Use a denylist to prevent XSS"  
✅ DO say: "Output encode for context + CSP header + HttpOnly cookies"

❌ DON'T say: "MD5 is fine for non-critical hashing"
✅ DO say: "SHA-256 for integrity checks, bcrypt for passwords — MD5 is broken"

❌ DON'T say: "Security by obscurity works in some cases"
✅ DO say: "Obscurity can be a layer, never the primary control"
```

---

## 📐 System Design Security Checklist

Use this when answering "Design X system" questions:

```
□ Authentication: How do users/services prove identity?
□ Authorization: How do you enforce who can do what?
□ Input Validation: How do you reject malicious data?
□ Encryption: Data at rest? In transit? Key management?
□ Rate Limiting: How do you prevent abuse/DDoS?
□ Logging/Monitoring: How do you detect attacks?
□ Secrets: Where are credentials stored? How rotated?
□ Network: Segmented? mTLS between services?
□ Supply Chain: Dependencies audited? Containers scanned?
□ Incident Response: What happens when breached?
```

---

## 🎮 The Security Mindset Mantra

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║   "What's the worst thing that could happen?"                    ║
║   "How would I attack this system?"                              ║
║   "What if this component is compromised?"                       ║
║   "What's the blast radius?"                                     ║
║   "Am I trusting something I shouldn't?"                         ║
║                                                                  ║
║   Ask these questions about EVERY feature you build.             ║
║   The engineer who thinks about security by DEFAULT              ║
║   is the engineer companies fight to hire.                       ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔗 Full Tutorial Series

| # | Tutorial | Time | Go Here If... |
|---|---------|------|---------------|
| 1 | [Security Fundamentals](./01_Security_Fundamentals_QA.md) | 55 min | Need to understand security principles |
| 2 | [Cryptography Essentials](./02_Cryptography_Essentials_QA.md) | 60 min | Need crypto knowledge |
| 3 | [Web Security](./03_Web_Security_QA.md) | 65 min | Need XSS/CSRF/SQLi knowledge |
| 4 | [Secure Coding Practices](./04_Secure_Coding_Practices_QA.md) | 55 min | Need coding-level security patterns |
| 5 | [Java Security Deep Dive](./05_Java_Security_Deep_Dive_QA.md) | 60 min | Need Java/Spring-specific security |
| 6 | [Security Scenarios & Puzzles](./06_Security_Scenarios_And_Puzzles.md) | 75 min | Want to practice with realistic challenges |

---

*[← Security Scenarios](./06_Security_Scenarios_And_Puzzles.md) | [Back to Security Index](../README.md)*
