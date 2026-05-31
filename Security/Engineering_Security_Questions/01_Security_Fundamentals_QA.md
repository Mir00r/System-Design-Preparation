# 🧱 Security Fundamentals Q&A — The Bedrock Every Engineer Must Know

> *"Imagine you're building a house. You wouldn't start picking curtain colors before laying the foundation, right? Security works the same way. Before you touch OAuth, JWT, or Spring Security — you need to understand WHY we need security and WHAT we're protecting against. This is the chapter that separates engineers who 'do security' from engineers who 'understand security'."*

**⏱️ Estimated Time**: 55 minutes | **🎯 Difficulty**: 🟢 Easy → 🟡 Medium | **🔗 Prerequisites**: None — this IS where you start

---

## 📋 Table of Contents
1. [Why Security Matters — The Wake-Up Call](#-why-security-matters--the-wake-up-call)
2. [The CIA Triad — Security's Holy Trinity](#-the-cia-triad--securitys-holy-trinity)
3. [Defense in Depth — The Onion Model](#-defense-in-depth--the-onion-model)
4. [Principle of Least Privilege](#-principle-of-least-privilege)
5. [Threat Modeling — Think Like an Attacker](#-threat-modeling--think-like-an-attacker)
6. [Attack Surface & Attack Vectors](#-attack-surface--attack-vectors)
7. [Security by Design vs Security by Obscurity](#-security-by-design-vs-security-by-obscurity)
8. [The Human Factor — Social Engineering](#-the-human-factor--social-engineering)
9. [Risk Assessment Framework](#-risk-assessment-framework)
10. [Gamification: Security Architect Challenge](#-gamification-security-architect-challenge)
11. [Interview Q&A — 25 Must-Know Questions](#-interview-qa--25-must-know-questions)
12. [What's Next](#-whats-next)

---

## 🚨 Why Security Matters — The Wake-Up Call

### 💰 The Cost of Getting It Wrong

```
╔══════════════════════════════════════════════════════════════════╗
║                    THE BREACH HALL OF SHAME                       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  🏦 Equifax (2017)      → 147M records → $700M fine             ║
║     Root cause: Unpatched Apache Struts (known vuln for 2 months)║
║                                                                  ║
║  🎯 Target (2013)       → 40M credit cards → $292M total cost   ║
║     Root cause: HVAC vendor had network access to POS systems    ║
║                                                                  ║
║  🔑 SolarWinds (2020)   → 18,000 orgs → $100B+ estimated       ║
║     Root cause: Compromised build system, trusted update channel ║
║                                                                  ║
║  📱 T-Mobile (2023)     → 37M customers → ongoing lawsuits      ║
║     Root cause: API with no rate limiting, sequential user IDs   ║
║                                                                  ║
║  🏥 Change Healthcare   → $22B company paralyzed for weeks      ║
║  (2024)                   Root cause: No MFA on Citrix portal    ║
║                                                                  ║
║  💡 Pattern: These aren't sophisticated zero-day attacks.        ║
║     They're BASIC security failures that any engineer should     ║
║     have caught during design or code review.                    ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 Quick Quiz: Would YOU Have Caught It?

> **Scenario**: Your team deploys an API endpoint `GET /api/users/{userId}/profile`. The userId is a sequential integer. There's no rate limiting. Authentication is required but there's no check that the authenticated user owns the requested profile.

<details>
<summary>🤔 What vulnerabilities exist here? (Click to reveal)</summary>

1. **IDOR (Insecure Direct Object Reference)** — Any logged-in user can access any other user's profile by incrementing the ID
2. **Enumeration Attack** — Sequential IDs let attackers iterate through ALL profiles
3. **No Rate Limiting** — Attacker can scrape the entire user database
4. **Information Disclosure** — Profile data exposed without ownership verification

**This is exactly what happened at Optus (2022)** — 10M records exposed through an unauthenticated API with sequential IDs.

**Fix**: UUID instead of sequential IDs + ownership check + rate limiting + response field filtering
</details>

---

## 🔺 The CIA Triad — Security's Holy Trinity

> 🎮 **Think of it as a video game**: Every security system protects three health bars. If ANY bar drops to zero, you lose.

```
                    🔺 CIA TRIAD
                   /            \
                  /              \
                 /                \
     CONFIDENTIALITY         INTEGRITY
     "Only the right        "Data hasn't been
      people can see it"     tampered with"
                 \                /
                  \              /
                   \            /
                    AVAILABILITY
                   "It's there when
                    you need it"
```

### 🔐 Confidentiality — "Who can see this?"

**Real-life analogy**: Your diary has a lock. Only you have the key. If your sibling reads it, confidentiality is breached.

| Attack | Example | Defense |
|--------|---------|---------|
| Eavesdropping | Man-in-the-middle reads HTTP traffic | TLS/HTTPS encryption |
| Data breach | SQL injection dumps user table | Access controls, encryption at rest |
| Shoulder surfing | Someone reads your screen | Screen locks, privacy filters |
| Social engineering | Phishing email tricks you into sharing credentials | Training, MFA |

**Java Example — Confidentiality Violation**:
```java
// ❌ BAD: Logging sensitive data
logger.info("User login: email={}, password={}", email, password);
logger.info("Processing payment: card={}", creditCardNumber);

// ✅ GOOD: Never log secrets
logger.info("User login: email={}", email);
logger.info("Processing payment: card=****{}", last4Digits);
```

### ✅ Integrity — "Has this been tampered with?"

**Real-life analogy**: You send a sealed letter. If someone opens it, changes the amount on a check from $100 to $10,000, and reseals it — integrity is breached.

| Attack | Example | Defense |
|--------|---------|---------|
| Data tampering | Modify price in request body from $10 to $0.01 | Server-side validation, signed requests |
| Man-in-the-middle | Alter API response data in transit | TLS, message signing |
| SQL injection | `UPDATE users SET role='admin' WHERE id=attacker_id` | Parameterized queries |
| Cache poisoning | Inject malicious content into CDN cache | Cache validation, signed content |

**Java Example — Integrity Check**:
```java
// Verify data integrity with HMAC
public boolean verifyWebhookSignature(String payload, String signature, String secret) {
    Mac mac = Mac.getInstance("HmacSHA256");
    mac.init(new SecretKeySpec(secret.getBytes(), "HmacSHA256"));
    String computed = Base64.getEncoder().encodeToString(mac.doFinal(payload.getBytes()));
    // ✅ Constant-time comparison prevents timing attacks
    return MessageDigest.isEqual(computed.getBytes(), signature.getBytes());
}
```

### 🌐 Availability — "Can I access it when I need it?"

**Real-life analogy**: The hospital's emergency room door is always open. If someone chains it shut, availability is breached — even though nothing was stolen or modified.

| Attack | Example | Defense |
|--------|---------|---------|
| DDoS | 10M requests/sec overwhelm servers | CDN, rate limiting, auto-scaling |
| Ransomware | Encrypt all data, demand payment | Backups, network segmentation |
| Resource exhaustion | Regex bomb, zip bomb, billion laughs XML | Input validation, resource limits |
| Dependency failure | Single database goes down, entire system fails | Redundancy, circuit breakers |

**Java Example — Availability Protection**:
```java
// ❌ BAD: Unbounded query — one user can consume all DB connections
@GetMapping("/search")
public List<Product> search(@RequestParam String query) {
    return productRepo.findByNameContaining(query); // Could return 10M rows
}

// ✅ GOOD: Bounded query with pagination
@GetMapping("/search")
public Page<Product> search(@RequestParam String query,
                            @RequestParam(defaultValue = "0") int page,
                            @RequestParam(defaultValue = "20") int size) {
    if (size > 100) size = 100; // Hard limit
    return productRepo.findByNameContaining(query, PageRequest.of(page, size));
}
```

### 🎮 CIA Triad Game: Classify the Attack!

For each scenario, identify which CIA principle is PRIMARILY violated:

| # | Scenario | Your Answer |
|---|----------|-------------|
| 1 | Attacker encrypts hospital records with ransomware | <details><summary>Reveal</summary>**Availability** — Data exists but can't be accessed</details> |
| 2 | Employee forwards customer data to personal email | <details><summary>Reveal</summary>**Confidentiality** — Unauthorized access to sensitive data</details> |
| 3 | Attacker modifies bank transfer amount in transit | <details><summary>Reveal</summary>**Integrity** — Data was altered without authorization</details> |
| 4 | DDoS takes down an e-commerce site on Black Friday | <details><summary>Reveal</summary>**Availability** — Service can't be accessed by legitimate users</details> |
| 5 | DNS poisoning redirects users to a fake bank website | <details><summary>Reveal</summary>**Integrity** + **Confidentiality** — DNS records tampered AND credentials stolen</details> |

---

## 🧅 Defense in Depth — The Onion Model

> 🎮 **Think of it as a castle**: The enemy doesn't just face one wall — they face a moat, outer wall, inner wall, guards, locked doors, and a vault. If one layer fails, the next stops them.

```
╔══════════════════════════════════════════════════════════════════╗
║                    DEFENSE IN DEPTH LAYERS                        ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║   Layer 7: DATA         → Encryption at rest, tokenization       ║
║   Layer 6: APPLICATION  → Input validation, OWASP, secure code   ║
║   Layer 5: HOST         → OS hardening, patching, antivirus      ║
║   Layer 4: NETWORK      → Firewalls, IDS/IPS, segmentation       ║
║   Layer 3: PERIMETER    → WAF, DDoS protection, edge security    ║
║   Layer 2: IDENTITY     → MFA, RBAC, zero trust                  ║
║   Layer 1: PHYSICAL     → Data center access, hardware security  ║
║                                                                  ║
║   ┌─────────────────────────────────────────────┐               ║
║   │  POLICIES & PROCEDURES (wrap everything)     │               ║
║   │  Training, incident response, auditing       │               ║
║   └─────────────────────────────────────────────┘               ║
╚══════════════════════════════════════════════════════════════════╝
```

### Real-World Example: How Netflix Does It

```
Request lifecycle through Netflix's defense layers:

[User] → [CloudFlare WAF] → [AWS Shield DDoS] → [API Gateway]
              Layer 3            Layer 3           Layer 4
                                                     │
[Rate Limiter] → [OAuth Token Validation] → [Service Mesh mTLS]
    Layer 6           Layer 2                   Layer 4
                                                     │
[Input Validation] → [Business Logic] → [Encrypted DB]
    Layer 6              Layer 6            Layer 7

If CloudFlare fails → AWS Shield catches DDoS
If token is stolen → Rate limiter limits blast radius
If service is compromised → mTLS prevents lateral movement
If DB is stolen → Data is encrypted at rest
```

### 🧩 Puzzle: Design the Layers

> **Challenge**: You're building a banking app. Design at least 5 defense layers for a money transfer endpoint `POST /api/transfer`.

<details>
<summary>💡 Hint: Think about what could go wrong at each stage</summary>

Consider: network layer (DDoS), identity layer (who's calling?), application layer (is the input valid?), business layer (is the transfer legitimate?), data layer (is it stored securely?)
</details>

<details>
<summary>✅ Sample Answer</summary>

```
Layer 1 — PERIMETER:     WAF blocks malicious payloads, DDoS protection
Layer 2 — IDENTITY:      OAuth2 + MFA for high-value transfers
Layer 3 — RATE LIMIT:    Max 5 transfers per minute per user
Layer 4 — INPUT:         Validate amount (positive, within limits), recipient exists
Layer 5 — BUSINESS:      Daily transfer limit, fraud detection ML model
Layer 6 — AUDIT:         Log every transfer attempt with full context
Layer 7 — DATA:          Encrypt account numbers, use database encryption at rest
Layer 8 — NETWORK:       Service-to-service mTLS, no direct DB access from internet
```
</details>

---

## 🔒 Principle of Least Privilege

> 🎮 **Hotel analogy**: Your room key opens YOUR room, not every room on the floor. The housekeeping master key opens all rooms on your floor, but not the manager's office. The general manager's key opens everything — but there's only ONE of those.

### The Principle

```
╔══════════════════════════════════════════════════════════════╗
║  LEAST PRIVILEGE: Every user, process, and system component  ║
║  should have ONLY the minimum access needed to perform its   ║
║  function — and NOTHING more.                                ║
╚══════════════════════════════════════════════════════════════╝
```

### ❌ Violations You See Every Day

```java
// ❌ BAD: Service account with full admin access
// "It works so let's just give it admin" — famous last words
@Bean
public DataSource dataSource() {
    return DataSourceBuilder.create()
        .username("root")              // 🚨 Root access!
        .password("admin123")          // 🚨 Weak password!
        .build();
}

// ✅ GOOD: Dedicated service account with minimal permissions
@Bean
public DataSource dataSource() {
    return DataSourceBuilder.create()
        .username("order_service_rw")  // Only this service's tables
        .password(secretsManager.get("db/order-service/password"))
        .build();
}
// DB user 'order_service_rw' has: SELECT, INSERT, UPDATE on orders table ONLY
// Cannot DROP, cannot access users table, cannot create tables
```

### Real-World Disasters from Privilege Violations

| Incident | What Happened | Least Privilege Fix |
|----------|--------------|---------------------|
| Capital One (2019) | WAF role had S3 full access → 100M records stolen | WAF role should only access its config bucket |
| Uber (2016) | Dev credentials in GitHub repo had access to S3 with 57M user records | Separate dev/prod credentials, no S3 access for app keys |
| Tesla (2018) | Kubernetes dashboard exposed without auth → crypto mining | Dashboard should require auth, pods should have resource limits |

### 🎮 Least Privilege Scorecard

Rate your current project (1-5 for each):

| Question | Score |
|----------|-------|
| Do service accounts have access to only their own tables? | ☐ |
| Are API keys scoped to specific endpoints/resources? | ☐ |
| Can developers access production databases? | ☐ |
| Are IAM roles per-service or shared? | ☐ |
| Do CI/CD pipelines have minimal deployment permissions? | ☐ |

**Score**: 20-25 = 🏆 Excellent | 15-19 = 🟡 Needs work | <15 = 🔴 High risk

---

## 🕵️ Threat Modeling — Think Like an Attacker

> 🎮 **Ocean's Eleven analogy**: Before robbing the casino, Danny Ocean's team studied every security camera, guard rotation, vault mechanism, and alarm system. Threat modeling is doing this to YOUR OWN system — before the bad guys do.

### STRIDE Framework

Microsoft's STRIDE is the most interview-relevant threat modeling framework:

```
S — Spoofing         → Pretending to be someone else
T — Tampering        → Modifying data or code
R — Repudiation      → Denying you did something (no audit trail)
I — Information Disclosure → Leaking sensitive data
D — Denial of Service → Making system unavailable  
E — Elevation of Privilege → Gaining unauthorized access level
```

### STRIDE in Action — E-Commerce Checkout

```
System: POST /api/checkout (user submits order)

┌─────────────────────────────────────────────────────────────┐
│ THREAT ANALYSIS                                              │
├──────────┬───────────────────────────────────────────────────┤
│ Spoofing │ Attacker uses stolen session token to buy items   │
│          │ → Defense: Short-lived tokens, device fingerprint │
├──────────┼───────────────────────────────────────────────────┤
│ Tampering│ Attacker modifies price in request body to $0.01  │
│          │ → Defense: Server-side price lookup, never trust  │
│          │   client-provided prices                          │
├──────────┼───────────────────────────────────────────────────┤
│ Repudiat.│ User claims "I never placed that order"           │
│          │ → Defense: Immutable audit logs, signed receipts  │
├──────────┼───────────────────────────────────────────────────┤
│ Info Disc│ Error message reveals DB schema or stack trace     │
│          │ → Defense: Generic errors to client, detailed     │
│          │   logs internally                                 │
├──────────┼───────────────────────────────────────────────────┤
│ DoS      │ Bot submits 10K orders/sec, overwhelming payment  │
│          │ → Defense: Rate limiting, CAPTCHA, queue          │
├──────────┼───────────────────────────────────────────────────┤
│ Elevation│ Regular user accesses admin discount endpoint      │
│          │ → Defense: RBAC, deny by default                  │
└──────────┴───────────────────────────────────────────────────┘
```

### 🎮 Threat Model This! (Practice Exercise)

> **Your turn**: Apply STRIDE to a **chat messaging system** (like Slack/WhatsApp). For each STRIDE category, identify one threat and one defense.

<details>
<summary>✅ Sample Answer</summary>

| STRIDE | Threat | Defense |
|--------|--------|---------|
| **S**poofing | Attacker impersonates a team member | E2E encryption with verified identities |
| **T**ampering | Message content modified in transit | Message signing + TLS |
| **R**epudiation | User sends inappropriate message, deletes it | Immutable audit logs, message hashes |
| **I**nfo Disclosure | Messages readable by server operators | End-to-end encryption (only sender/receiver can read) |
| **D**oS | Attacker floods channel with 100K messages/sec | Rate limiting per user, message size limits |
| **E**levation | Regular member gains admin/owner privileges | RBAC, privilege changes require MFA |
</details>

---

## 🎯 Attack Surface & Attack Vectors

### What's an Attack Surface?

> 🎮 **Think of your app as a house**: The attack surface is every door, window, chimney, and crack that a burglar could use to get in. The bigger the house, the more entry points.

```
ATTACK SURFACE OF A TYPICAL WEB APP:

  ┌─────────────────────────────────────────────────────────┐
  │                    YOUR APPLICATION                       │
  │                                                          │
  │  Entry Points (Attack Surface):                          │
  │                                                          │
  │  🚪 HTTP endpoints (REST APIs)                           │
  │  🚪 WebSocket connections                                │
  │  🚪 File upload forms                                    │
  │  🚪 Email processing (if applicable)                     │
  │  🚪 Third-party integrations (webhooks)                  │
  │  🚪 Admin panels                                         │
  │  🚪 Database ports (if exposed)                          │
  │  🚪 SSH/RDP access                                       │
  │  🚪 DNS (for DNS-based attacks)                          │
  │  🚪 CI/CD pipeline (supply chain)                        │
  │  🚪 Container registry                                   │
  │  🚪 Message queues (Kafka, RabbitMQ)                     │
  │  🚪 Monitoring endpoints (/actuator, /metrics)           │
  │                                                          │
  │  Each 🚪 = potential way in for an attacker              │
  └─────────────────────────────────────────────────────────┘
```

### Reducing Attack Surface

```
PRINCIPLE: Minimize what's exposed. If it doesn't need to be accessible, 
           it shouldn't be.

Before (Large Attack Surface):
  ├── /api/v1/* (public)
  ├── /api/v2/* (public)
  ├── /admin/* (public, password-protected)
  ├── /actuator/* (public, no auth!)     ← 🚨 DANGER
  ├── /swagger-ui/* (public)             ← 🚨 INFO DISCLOSURE
  ├── /h2-console/* (public)             ← 🚨 CRITICAL
  └── /debug/* (public, forgot to remove)← 🚨 CRITICAL

After (Minimized Attack Surface):
  ├── /api/v2/* (public, authenticated)
  ├── /admin/* (VPN-only, MFA required)
  ├── /actuator/health (public, limited info)
  ├── /actuator/* (internal network only)
  └── Everything else → 404
```

**Spring Boot — Reduce Attack Surface**:
```java
// application-prod.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus  # Only expose what's needed
  endpoint:
    health:
      show-details: never  # Don't expose internal details publicly

spring:
  h2:
    console:
      enabled: false  # NEVER in production
  
# Disable unnecessary auto-configurations
spring.autoconfigure.exclude:
  - org.springframework.boot.autoconfigure.h2.H2ConsoleAutoConfiguration
  - org.springframework.boot.devtools.autoconfigure.DevToolsDataSourceAutoConfiguration
```

---

## 🏗️ Security by Design vs Security by Obscurity

### Security by Obscurity (❌ DON'T DO THIS)

```
"Nobody will find our admin panel at /xK9mZ2_admin"
"Our encryption algorithm is proprietary — attackers won't figure it out"
"We use port 8443 instead of 443, so nobody will scan it"
"The database password is hidden in a config file called 'definitely_not_passwords.txt'"

Why it fails:
  → Attackers have automated scanners
  → Secrets eventually leak (GitHub, logs, error messages)
  → Decompilation reveals algorithms
  → "Security through obscurity is no security at all" — Auguste Kerckhoffs, 1883
```

### Security by Design (✅ DO THIS)

```
Kerckhoffs's Principle (1883):
  "A system should be secure even if everything about it
   is public knowledge — except the key."

Modern interpretation:
  → Assume the attacker has your source code (they probably will)
  → Assume the attacker knows your architecture
  → The ONLY secret should be the cryptographic keys/passwords
  → Everything else should be secure BY DESIGN

Examples:
  ✅ AES encryption: Algorithm is 100% public. Security comes from the key.
  ✅ bcrypt: Algorithm is open source. Security comes from computational cost.
  ✅ HTTPS: Protocol is completely documented. Security comes from certificates.
```

### 🎮 Spot the Obscurity! (Game)

Which of these rely on security by obscurity vs. security by design?

| Practice | Type |
|----------|------|
| Using a non-standard port for SSH (2222 instead of 22) | <details><summary>Reveal</summary>**Obscurity** — Scanner finds it in seconds. Use key-based auth instead.</details> |
| Encrypting data with AES-256-GCM | <details><summary>Reveal</summary>**By Design** — Algorithm is public, security from key strength</details> |
| Hiding admin API under random URL path | <details><summary>Reveal</summary>**Obscurity** — Will be found. Use proper authentication instead.</details> |
| Rate limiting + account lockout after 5 failed attempts | <details><summary>Reveal</summary>**By Design** — Works even if attacker knows the mechanism</details> |
| Using ROT13 to "encrypt" passwords in config | <details><summary>Reveal</summary>**Obscurity** — Not encryption at all, trivially reversible</details> |

---

## 🧠 The Human Factor — Social Engineering

> 🎮 **The weakest link in any security system isn't the code — it's the human operating it.**

### Why Social Engineering Works

```
╔══════════════════════════════════════════════════════════════╗
║  Kevin Mitnick (legendary hacker): "I was never really a    ║
║  hacker. I was a social engineer. I manipulated people      ║
║  into giving me access."                                    ║
╠══════════════════════════════════════════════════════════════╣
║                                                              ║
║  HUMAN VULNERABILITIES:                                      ║
║    • Authority bias: "The CEO asked for this urgently"       ║
║    • Helpfulness: "I'm from IT, I need your password to fix" ║
║    • Fear: "Your account will be closed in 24 hours"         ║
║    • Curiosity: "Click here to see who viewed your profile"  ║
║    • Urgency: "Transfer $50K NOW or the deal falls through"  ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### Common Social Engineering Attacks

| Attack | How It Works | Real Example |
|--------|-------------|--------------|
| **Phishing** | Fake email/website mimics trusted brand | Google Doc phishing (2017) — 1M users clicked |
| **Spear Phishing** | Targeted email to specific person | RSA breach (2011) — Excel file emailed to employees |
| **Vishing** | Voice call pretending to be support | Twitter hack (2020) — Called employees, got admin access |
| **Pretexting** | Made-up scenario to extract info | "Hi, I'm from the bank verifying your account..." |
| **Baiting** | Leave infected USB drives in parking lot | US DoD: 60% of dropped USBs were plugged in |

### Engineering Defense Against Social Engineering

```java
// The engineering perspective: defend against the HUMAN factor

// 1. MFA — even if password is phished, attacker can't log in
@Configuration
public class MfaConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        http.authorizeRequests()
            .antMatchers("/api/admin/**").access("hasRole('ADMIN') and @mfaVerifier.isVerified()")
            .antMatchers("/api/transfer/**").access("@mfaVerifier.isVerified()")
            // High-value actions ALWAYS require MFA
    }
}

// 2. Anomaly detection — flag unusual behavior even with valid credentials
public class LoginAnomalyDetector {
    public RiskScore assessLogin(LoginAttempt attempt) {
        // New device? New country? Unusual time? → Challenge with MFA
        if (isNewDevice(attempt) || isNewCountry(attempt)) {
            return RiskScore.HIGH; // Trigger step-up authentication
        }
    }
}
```

---

## ⚖️ Risk Assessment Framework

### Risk = Likelihood × Impact

```
                        RISK MATRIX
                        
        │ Negligible │   Low    │  Medium  │   High   │ Critical │
   ─────┼────────────┼──────────┼──────────┼──────────┼──────────┤
   Very  │    Low     │  Medium  │   High   │ Critical │ Critical │
   Likely│            │          │          │          │          │
   ─────┼────────────┼──────────┼──────────┼──────────┼──────────┤
   Likely│ Negligible │   Low    │  Medium  │   High   │ Critical │
        │            │          │          │          │          │
   ─────┼────────────┼──────────┼──────────┼──────────┼──────────┤
   Poss- │ Negligible │   Low    │  Medium  │  Medium  │   High   │
   ible  │            │          │          │          │          │
   ─────┼────────────┼──────────┼──────────┼──────────┼──────────┤
   Un-   │ Negligible │Negligible│   Low    │   Low    │  Medium  │
   likely│            │          │          │          │          │
   ─────┴────────────┴──────────┴──────────┴──────────┴──────────┘
              ← IMPACT (damage if exploit succeeds) →
```

### DREAD Model (Microsoft's Risk Scoring)

```
D — Damage:          How bad is it if exploited? (1-10)
R — Reproducibility: How easy to reproduce? (1-10)
E — Exploitability:  How easy to launch the attack? (1-10)
A — Affected Users:  How many users impacted? (1-10)
D — Discoverability: How easy to find the vulnerability? (1-10)

Risk Score = (D + R + E + A + D) / 5

Example: SQL Injection on login page
  D = 10 (full DB access)
  R = 10 (works every time)
  E = 8  (tools like sqlmap make it easy)
  A = 10 (all users affected)
  D = 9  (scanners find it automatically)
  Score = 47/5 = 9.4 → CRITICAL → Fix immediately
```

### 🎮 Risk Assessment Practice

Score these vulnerabilities using DREAD:

| Vulnerability | D | R | E | A | D | Score | Priority |
|--------------|---|---|---|---|---|-------|----------|
| SQL injection on search page | <details><summary>?</summary>9</details> | <details><summary>?</summary>10</details> | <details><summary>?</summary>8</details> | <details><summary>?</summary>10</details> | <details><summary>?</summary>9</details> | <details><summary>?</summary>9.2 Critical</details> | <details><summary>?</summary>Fix NOW</details> |
| CSRF on profile update | <details><summary>?</summary>4</details> | <details><summary>?</summary>8</details> | <details><summary>?</summary>6</details> | <details><summary>?</summary>7</details> | <details><summary>?</summary>6</details> | <details><summary>?</summary>6.2 Medium</details> | <details><summary>?</summary>Fix this sprint</details> |
| Missing rate limit on login | <details><summary>?</summary>6</details> | <details><summary>?</summary>10</details> | <details><summary>?</summary>9</details> | <details><summary>?</summary>10</details> | <details><summary>?</summary>8</details> | <details><summary>?</summary>8.6 High</details> | <details><summary>?</summary>Fix this week</details> |

---

## 🎮 Gamification: Security Architect Challenge

### 🏰 Level 1: Secure the Startup (Score: /100)

> **Scenario**: You've just been hired as the first security-minded engineer at a startup. The app is a Node.js/Java backend with React frontend, PostgreSQL database, deployed on AWS. There are 10,000 users and the app handles payments.
>
> **Current state**: No security measures exist. Design the security architecture.

**Your budget**: You can implement 10 security controls. Each costs "effort points":

| Control | Effort | Impact |
|---------|--------|--------|
| HTTPS everywhere | 2 pts | 🔒🔒🔒 |
| Input validation on all endpoints | 5 pts | 🔒🔒🔒🔒 |
| Password hashing (bcrypt) | 2 pts | 🔒🔒🔒 |
| Rate limiting | 3 pts | 🔒🔒🔒 |
| SQL parameterized queries | 3 pts | 🔒🔒🔒🔒🔒 |
| MFA for admin accounts | 3 pts | 🔒🔒🔒 |
| Secrets in environment variables (not code) | 2 pts | 🔒🔒🔒 |
| Audit logging | 4 pts | 🔒🔒 |
| WAF (Web Application Firewall) | 5 pts | 🔒🔒🔒 |
| Encryption at rest | 4 pts | 🔒🔒 |
| Security headers (CSP, HSTS, etc.) | 2 pts | 🔒🔒 |
| Dependency scanning | 3 pts | 🔒🔒🔒 |

**Budget**: 20 effort points. Which controls do you pick?

<details>
<summary>✅ Optimal Strategy (20 points, maximum security)</summary>

| Pick | Control | Effort | Reason |
|------|---------|--------|--------|
| ✅ | HTTPS everywhere | 2 | Table stakes — prevents all MitM |
| ✅ | SQL parameterized queries | 3 | Prevents the #1 most dangerous vuln |
| ✅ | Password hashing (bcrypt) | 2 | Protects users when (not if) DB is breached |
| ✅ | Input validation | 5 | Stops XSS, injection, business logic abuse |
| ✅ | Secrets in env vars | 2 | Prevents credential leaks in source code |
| ✅ | Rate limiting | 3 | Prevents brute force and scraping |
| ✅ | MFA for admin | 3 | Protects the keys to the kingdom |
| **Total** | | **20** | Core security posture established |

**Why not WAF first?** A WAF is a band-aid. If your code has SQL injection, the WAF helps but fixing the code is the real solution. **Fix the root cause first, then add layers.**
</details>

---

## 🎯 Interview Q&A — 25 Must-Know Questions

### Fundamentals (Questions 1-10)

**Q1: What is the CIA Triad and why does it matter?**

> **A**: CIA stands for Confidentiality (only authorized access), Integrity (data hasn't been tampered with), and Availability (system is accessible when needed). It matters because every security decision maps back to protecting one or more of these properties. When prioritizing security work, I classify threats by which CIA property they attack and the business impact of losing that property.

**Q2: Explain Defense in Depth with a real example.**

> **A**: Defense in Depth means layering multiple security controls so that if one fails, others still protect the system. Example: A banking API has (1) WAF to block known attack patterns, (2) authentication to verify identity, (3) authorization to check permissions, (4) input validation to reject malformed data, (5) parameterized queries to prevent injection even if validation misses something, (6) encryption at rest so stolen databases are useless, (7) audit logs to detect and investigate breaches. Each layer assumes the previous one failed.

**Q3: What is the Principle of Least Privilege? Give a concrete example.**

> **A**: Every entity should have only the minimum permissions needed. Concrete example: A microservice that reads from a product catalog should have a database user with SELECT permission on the products table only — not INSERT, not DELETE, not access to the users table. If that service is compromised, the attacker can only read products, not modify data or access credentials.

**Q4: What's the difference between a vulnerability, a threat, and a risk?**

> **A**: 
> - **Vulnerability**: A weakness in the system (e.g., unpatched Log4j library)
> - **Threat**: Something that could exploit the vulnerability (e.g., attacker scanning for Log4j)
> - **Risk**: The probability of the threat exploiting the vulnerability × the impact (e.g., high probability + critical impact = critical risk)
> 
> Analogy: A broken lock (vulnerability) + a neighborhood burglar (threat) = risk of robbery.

**Q5: Explain threat modeling. Which framework would you use?**

> **A**: Threat modeling is systematically identifying potential security threats to a system during design phase. I'd use STRIDE (Spoofing, Tampering, Repudiation, Info Disclosure, DoS, Elevation of Privilege) because it maps directly to security controls. Process: (1) Diagram the system with data flows, (2) For each component/flow, apply STRIDE categories, (3) Assess risk of each threat, (4) Design mitigations, (5) Validate mitigations are implemented.

**Q6: What is "Security by Design" and how does it differ from "Security by Obscurity"?**

> **A**: Security by Design means the system is secure even if the attacker knows everything about it except the keys — following Kerckhoffs's Principle. Security by Obscurity relies on keeping the design secret (hidden URLs, proprietary algorithms). Obscurity fails because secrets leak, code gets decompiled, and automated scanners don't care about hidden paths. Example: AES is public and secure (by design); ROT13 is "hidden" but trivially broken (by obscurity).

**Q7: How would you prioritize security work when resources are limited?**

> **A**: Use risk-based prioritization: (1) Identify assets and their business value, (2) Map threats using STRIDE, (3) Score using DREAD or CVSS, (4) Fix critical/high risks first — especially those that are easily exploitable and affect many users, (5) Accept low risks with documentation and monitoring. A SQL injection on a public endpoint that accesses PII is always higher priority than a theoretical DoS on an internal tool.

**Q8: What's the difference between authentication and authorization?**

> **A**: Authentication = "Who are you?" (proving identity — via password, token, certificate). Authorization = "What can you do?" (checking permissions — via RBAC, ABAC, policies). HTTP 401 means authentication failed; 403 means authentication succeeded but authorization failed. Common mistake: checking authentication but forgetting authorization — user is logged in but accessing other users' data.

**Q9: Explain the concept of "Zero Trust." Why is perimeter security insufficient?**

> **A**: Zero Trust means "never trust, always verify" — even traffic inside the corporate network must be authenticated and authorized. Perimeter security failed because once an attacker is inside (via phishing, compromised vendor, stolen VPN), they have full trusted access. SolarWinds, Target, and Colonial Pipeline all demonstrate this. Zero Trust principles: verify explicitly, use least-privilege access, assume breach has already occurred.

**Q10: What is the attack surface? How do you minimize it?**

> **A**: The attack surface is the sum of all points where an attacker can try to interact with the system — APIs, open ports, file uploads, admin panels, third-party integrations. Minimize by: (1) Disable unused features and endpoints, (2) Remove debug/actuator endpoints in production, (3) Close unnecessary ports, (4) Minimize third-party dependencies, (5) Use allowlists instead of denylists, (6) Remove unused API versions.

### Applied Security (Questions 11-20)

**Q11: How would you secure an API endpoint that handles sensitive data?**

> **A**: Layers: (1) TLS for transport encryption, (2) OAuth2/JWT for authentication, (3) RBAC for authorization, (4) Input validation and sanitization, (5) Rate limiting, (6) Response field filtering (don't return unnecessary fields), (7) Audit logging, (8) CORS policy to restrict origins. For highly sensitive data like PCI: add tokenization, field-level encryption, and WAF rules.

**Q12: Explain the concept of "fail secure" vs "fail open."**

> **A**: Fail secure (fail closed) = when a security component fails, deny access. Example: If the authentication service is down, reject all requests. Fail open = when it fails, allow access. Example: If auth is down, let everyone through. For security components, always fail secure. For availability-critical paths with low security risk (like a CDN serving public content), you might accept fail-open with monitoring.

**Q13: What is a side-channel attack? Give an engineering example.**

> **A**: A side-channel attack extracts information from the system's physical characteristics rather than direct security flaws — timing, power consumption, error messages, cache behavior. Engineering example: A login endpoint returns in 5ms for "user not found" but 200ms for "wrong password" (because it does bcrypt comparison). An attacker can enumerate valid usernames by measuring response time. Fix: Always execute the same code path regardless of whether user exists.

```java
// ❌ Timing attack vulnerable
public boolean authenticate(String username, String password) {
    User user = userRepo.findByUsername(username);
    if (user == null) return false;  // Fast path reveals user doesn't exist
    return bcrypt.matches(password, user.getPasswordHash());
}

// ✅ Constant-time (no information leakage)
public boolean authenticate(String username, String password) {
    User user = userRepo.findByUsername(username);
    String hashToCompare = (user != null) ? user.getPasswordHash() : DUMMY_HASH;
    boolean matches = bcrypt.matches(password, hashToCompare);
    return user != null && matches;
}
```

**Q14: What is input validation? What's the difference between allowlist and denylist?**

> **A**: Input validation ensures data matches expected format/constraints before processing. Allowlist (whitelist) defines what IS allowed — everything else is rejected. Denylist (blacklist) defines what ISN'T allowed — everything else passes. Always prefer allowlists because attackers constantly find new bypass patterns for denylists.

**Q15: How do you store passwords securely?**

> **A**: Never store plaintext or reversibly encrypted passwords. Use: (1) A slow, adaptive hash algorithm (bcrypt, scrypt, or Argon2id), (2) Unique salt per password (bcrypt does this automatically), (3) High cost factor (bcrypt cost 12+ means ~250ms per hash). Why slow? If DB is breached, slow hashing makes brute-force infeasible. MD5/SHA-256 are too fast — an attacker can try billions per second on GPUs.

**Q16: Explain CORS. Why does it exist and how can it be misconfigured?**

> **A**: CORS (Cross-Origin Resource Sharing) is a browser security mechanism that prevents JavaScript on `evil.com` from making requests to `your-api.com` and reading the response. Common misconfiguration: setting `Access-Control-Allow-Origin: *` with credentials, or reflecting the Origin header without validation — this lets any website make authenticated requests to your API on behalf of the user.

**Q17: What are security headers and which ones are essential?**

> **A**: HTTP headers that instruct browsers to enable security features. Essential ones: (1) `Strict-Transport-Security` — force HTTPS, (2) `Content-Security-Policy` — prevent XSS by whitelisting script sources, (3) `X-Content-Type-Options: nosniff` — prevent MIME sniffing, (4) `X-Frame-Options: DENY` — prevent clickjacking, (5) `Referrer-Policy` — control information leakage in referrer headers. These are low-effort, high-impact defenses.

**Q18: How do you handle secrets in a distributed system?**

> **A**: Never in code, never in config files committed to git. Use: (1) Secrets manager (HashiCorp Vault, AWS Secrets Manager), (2) Inject at runtime via environment variables or mounted volumes, (3) Rotate regularly (automated), (4) Audit access to secrets, (5) Scope secrets to specific services (least privilege). In Kubernetes: use sealed secrets or external secrets operator, never plain Kubernetes secrets (they're just base64-encoded).

**Q19: What is the difference between encryption, hashing, and encoding?**

> **A**: 
> - **Encryption**: Reversible with a key. Purpose: confidentiality. (AES, RSA)
> - **Hashing**: One-way, irreversible. Purpose: integrity verification, password storage. (SHA-256, bcrypt)
> - **Encoding**: Reversible without a key. Purpose: data format conversion, NOT security. (Base64, URL encoding)
> 
> Critical mistake: Using Base64 (encoding) thinking it's encryption. Base64 is NOT security — anyone can decode it.

**Q20: How would you implement rate limiting? What attacks does it prevent?**

> **A**: Rate limiting restricts how many requests a client can make in a time window. Implementation: Token bucket or sliding window algorithm, keyed by (user_id, IP, or API key). Prevents: brute-force attacks, credential stuffing, API scraping, DoS, enumeration attacks. Important: Apply at multiple levels — (1) Per-user for authenticated endpoints, (2) Per-IP for unauthenticated endpoints, (3) Global for overall capacity protection.

### Architecture & Design (Questions 21-25)

**Q21: How would you design security for a microservices architecture?**

> **A**: (1) API Gateway for edge authentication and rate limiting, (2) mTLS for service-to-service communication, (3) JWT with short expiry for request context propagation, (4) Service mesh (Istio) for network policies, (5) Per-service secrets with automated rotation, (6) Centralized logging for security events, (7) Network segmentation — services can only call specific other services, (8) Circuit breakers to contain failures.

**Q22: What is supply chain security? How do you protect against it?**

> **A**: Supply chain attacks target dependencies, build tools, or CI/CD pipelines rather than your code directly. Protection: (1) Pin dependency versions with lock files, (2) Use tools like Dependabot/Snyk to scan for known vulnerabilities, (3) Verify package integrity (checksums, signatures), (4) Minimize dependencies, (5) Use private registries, (6) Sign build artifacts, (7) Implement SBOM (Software Bill of Materials). SolarWinds and Log4Shell are examples.

**Q23: Explain the concept of "blast radius" in security architecture.**

> **A**: Blast radius is the extent of damage if one component is compromised. Design goal: minimize blast radius. Techniques: (1) Network segmentation — compromised service can't reach unrelated services, (2) Separate databases per service — breach of one doesn't expose all data, (3) Short-lived credentials — stolen token expires in minutes, (4) Principle of least privilege — compromised service can only access its own resources. Example: At Netflix, each microservice has its own AWS IAM role with minimal permissions.

**Q24: How do you balance security with developer productivity?**

> **A**: Security should be a "paved road" — make the secure way the easy way. (1) Provide secure defaults in frameworks/libraries (e.g., auto-escaping in templates), (2) Automated security scanning in CI/CD (shift left), (3) Pre-approved patterns/libraries (don't make devs reinvent auth), (4) Security guardrails (linters, pre-commit hooks) rather than gates, (5) Threat model during design (catch issues early when they're cheap to fix). Security that slows down development gets bypassed.

**Q25: You discover a critical vulnerability in production. Walk me through your incident response.**

> **A**: (1) **Assess**: Determine scope — is it being actively exploited? What data is at risk? (2) **Contain**: Mitigate immediately — block the attack vector (WAF rule, disable endpoint, revoke compromised credentials), (3) **Communicate**: Alert security team, management, potentially legal/compliance, (4) **Fix**: Deploy the patch, verify it resolves the issue, (5) **Recover**: Restore affected data/systems, reset compromised credentials, (6) **Review**: Post-mortem — how did it happen, how was it found, how do we prevent similar issues, what alerts should we add. Timeline: Contain within minutes, communicate within hours, fix within days (severity-dependent).

---

## 📊 Key Takeaways

```
╔══════════════════════════════════════════════════════════════════╗
║  SECURITY FUNDAMENTALS — REMEMBER THESE FOREVER                  ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1. CIA Triad: Every decision maps to Confidentiality,           ║
║     Integrity, or Availability                                   ║
║                                                                  ║
║  2. Defense in Depth: Never rely on a single control             ║
║                                                                  ║
║  3. Least Privilege: Give minimum access needed                  ║
║                                                                  ║
║  4. Threat Model Early: STRIDE during design, not after deploy   ║
║                                                                  ║
║  5. Assume Breach: Design systems that limit damage when         ║
║     (not if) a component is compromised                          ║
║                                                                  ║
║  6. Security by Design: Never rely on secrecy of mechanism       ║
║                                                                  ║
║  7. Fail Secure: When in doubt, deny access                      ║
║                                                                  ║
║  8. Tools change, principles don't: Master the WHY               ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Topic | Link | Why Read It? |
|-------|------|-------------|
| Cryptography Essentials | [02_Cryptography_Essentials_QA.md](./02_Cryptography_Essentials_QA.md) | Understand the math that protects everything |
| Web Security Q&A | [03_Web_Security_QA.md](./03_Web_Security_QA.md) | XSS, CSRF, injection — what interviewers ask most |
| OWASP Top 10 | [../OWASP_Top10.md](../OWASP_Top10.md) | The industry standard vulnerability list |

---

*[← Back to Security Index](../README.md) | [Next: Cryptography Essentials →](./02_Cryptography_Essentials_QA.md)*
