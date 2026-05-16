# 🔒 OWASP Top 10
## The Most Critical Web Application Security Risks — With Java Examples

> *"The OWASP Top 10 is not just a checklist — it's a map of where real attackers are looking. Every item on this list represents a class of vulnerabilities that has caused major breaches in production systems you've used."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: None

---

## 📋 Table of Contents
1. [What is OWASP?](#-what-is-owasp)
2. [A01 — Broken Access Control](#-a01--broken-access-control)
3. [A02 — Cryptographic Failures](#-a02--cryptographic-failures)
4. [A03 — Injection](#-a03--injection)
5. [A04 — Insecure Design](#-a04--insecure-design)
6. [A05 — Security Misconfiguration](#-a05--security-misconfiguration)
7. [A06 — Vulnerable Components](#-a06--vulnerable-components)
8. [A07 — Authentication Failures](#-a07--authentication-failures)
9. [A08 — Data Integrity Failures](#-a08--data-integrity-failures)
10. [A09 — Logging Failures](#-a09--logging-failures)
11. [A10 — SSRF](#-a10--server-side-request-forgery-ssrf)
12. [Quick Reference Cheat Sheet](#-quick-reference-cheat-sheet)
13. [Mini Challenge](#-mini-challenge)
14. [Interview Q&A](#-interview-qa)

---

## 🌐 What is OWASP?

The **Open Web Application Security Project (OWASP)** is a non-profit that publishes the world's most widely used web security standard. The **OWASP Top 10** (updated 2021) lists the most critical security risks, ranked by prevalence and impact in real-world breaches.

This is mandatory reading for any engineer building web applications. Interviewers at security-conscious companies will ask about these directly.

---

## 🔐 A01 — Broken Access Control

**Rank**: #1 (moved up from #5 in 2017)
**Real-world example**: Facebook's 2021 data breach exposed 533M records partly due to broken access controls on their contact importer API.

### The Vulnerability

```
User A should only see their own data.
User B should only see their own data.

Broken code:
  GET /api/orders/{orderId}
  → returns order without checking if current user owns it
  → User A can fetch User B's orders by incrementing orderId
```

### Types

```
IDOR (Insecure Direct Object Reference):
  /api/users/123/profile  → accessing another user's profile by changing 123 → 456

Privilege Escalation:
  Regular user accesses /admin endpoints because the app only checks if user is logged in,
  not if they have admin role

Missing Function-Level Access Control:
  Hidden admin UI routes (e.g., /api/admin/deleteUser) accessible if URL is known
```

### Fix

```java
// WRONG — no ownership check
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId) {
    return orderRepository.findById(orderId).orElseThrow();
}

// CORRECT — verify the authenticated user owns this resource
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId,
                      @AuthenticationPrincipal UserDetails currentUser) {
    Order order = orderRepository.findById(orderId).orElseThrow();

    // ALWAYS verify ownership
    if (!order.getUserId().equals(currentUser.getUserId())) {
        throw new AccessDeniedException("You don't have permission to view this order");
    }
    return order;
}

// Better: query with ownership baked in (database enforces it)
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId,
                      @AuthenticationPrincipal UserDetails currentUser) {
    return orderRepository.findByIdAndUserId(orderId, currentUser.getUserId())
            .orElseThrow(() -> new ResourceNotFoundException("Order not found"));
}
```

---

## 🔏 A02 — Cryptographic Failures

**Rank**: #2 (was "Sensitive Data Exposure" in 2017)
**Real-world example**: LinkedIn 2012 — 117M password hashes exposed, cracked quickly because they used unsalted SHA-1.

### The Vulnerability

```
Data exposed in transit:
  - HTTP instead of HTTPS
  - TLS 1.0/1.1 (deprecated, broken)

Data at rest without protection:
  - Passwords stored in plaintext or with MD5/SHA1 (not password hashes!)
  - PII (SSN, credit cards) stored unencrypted in database
  - Sensitive data in logs

Weak crypto:
  - ECB mode for AES (reveals patterns in data)
  - MD5/SHA1 for integrity checks (collision vulnerabilities)
```

### Fix

```java
// WRONG — storing password as SHA-256 hash (still crackable)
String hash = DigestUtils.sha256Hex(password);
userRepository.save(user.withPasswordHash(hash));

// CORRECT — use bcrypt (or Argon2 / scrypt)
// BCrypt automatically handles salting and is intentionally slow (cost factor)
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);  // cost factor 12 = ~250ms per hash
}

// Usage:
String encodedPassword = passwordEncoder.encode(rawPassword);
boolean matches = passwordEncoder.matches(rawPassword, storedEncodedPassword);

// For encryption at rest (not passwords — use for reversible data like credit cards):
// Use AES-256-GCM (authenticated encryption)
@Service
public class EncryptionService {
    @Value("${encryption.key}")  // 256-bit key from secret manager
    private String base64Key;

    public String encrypt(String plaintext) {
        byte[] key = Base64.getDecoder().decode(base64Key);
        SecretKeySpec secretKey = new SecretKeySpec(key, "AES");
        // Use GCM mode — provides both confidentiality AND integrity
        Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
        byte[] iv = new byte[12];  // 96-bit IV, random each time
        new SecureRandom().nextBytes(iv);
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, new GCMParameterSpec(128, iv));
        byte[] encrypted = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
        // Prepend IV to ciphertext (needed for decryption, not a secret)
        return Base64.getEncoder().encodeToString(concatenate(iv, encrypted));
    }
}
```

---

## 💉 A03 — Injection

**Rank**: #3 (was #1 for over a decade)
**Real-world example**: Sony Pictures 2011 — SQL injection exposed 1M+ user records including passwords.

### SQL Injection

```java
// WRONG — string concatenation in query
String query = "SELECT * FROM users WHERE email = '" + userInput + "'";
// Attacker input: "' OR '1'='1" → returns ALL users
// Attacker input: "'; DROP TABLE users; --" → drops the users table

// CORRECT — parameterized queries (prepared statements)
// Spring Data JPA:
@Query("SELECT u FROM User u WHERE u.email = :email")
Optional<User> findByEmail(@Param("email") String email);

// JDBC:
PreparedStatement stmt = conn.prepareStatement("SELECT * FROM users WHERE email = ?");
stmt.setString(1, userInput);  // parameter binding, not concatenation
```

### NoSQL Injection (MongoDB)

```java
// WRONG — using user input directly in MongoDB query
Document query = new Document("username", userInput)
        .append("$where", "this.password == '" + password + "'");

// Attacker: password = "' || '1'=='1" → bypasses authentication

// CORRECT — use typed parameters, never $where with user input
Query query = new Query(Criteria.where("username").is(username));
User user = mongoTemplate.findOne(query, User.class);
// Verify password separately using BCrypt
passwordEncoder.matches(rawPassword, user.getPasswordHash());
```

### Command Injection

```java
// WRONG — executing user input as shell command
Runtime.getRuntime().exec("ping " + userInput);
// Attacker: "8.8.8.8; rm -rf /" → deletes files!

// CORRECT — use safe APIs, never shell out with user input
InetAddress address = InetAddress.getByName(hostname);  // Java's network API
boolean reachable = address.isReachable(3000);
// If you must exec: whitelist allowed values, never concat user input into command
```

---

## 🏛️ A04 — Insecure Design

**Rank**: #4 (new in 2021)

This is a broader category about architectural security flaws — not code bugs.

### Examples

```
Missing rate limiting on password reset:
  → Attacker can send 10,000 password reset emails to a user (DoS on their inbox)
  → Or brute-force OTP codes with unlimited attempts

No account lockout:
  → Login endpoint allows unlimited password attempts
  → Attacker brute-forces passwords

Password reset via security questions:
  → "What's your mother's maiden name?" — publicly available from social media
  → Design flaw: security questions are inherently insecure

Uploading files without validation:
  → User uploads "image.jpg" that is actually "malware.php"
  → Server executes it as PHP if stored in web root
```

### Fix (Design-Level)

```java
// Rate limiting on sensitive endpoints
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .addFilterBefore(rateLimitFilter, UsernamePasswordAuthenticationFilter.class)
        ...
}

// Account lockout after N failed attempts
@Service
public class LoginService {
    private static final int MAX_ATTEMPTS = 5;
    private static final Duration LOCKOUT_DURATION = Duration.ofMinutes(15);

    public boolean authenticate(String email, String password) {
        String lockKey = "login_attempts:" + email;
        int attempts = Integer.parseInt(
                Optional.ofNullable(redis.opsForValue().get(lockKey)).orElse("0"));

        if (attempts >= MAX_ATTEMPTS) {
            throw new AccountLockedException("Account locked for 15 minutes");
        }

        User user = userRepository.findByEmail(email).orElse(null);
        if (user == null || !passwordEncoder.matches(password, user.getPasswordHash())) {
            redis.opsForValue().increment(lockKey);
            redis.expire(lockKey, LOCKOUT_DURATION);
            throw new BadCredentialsException("Invalid credentials");
        }

        redis.delete(lockKey);  // reset on success
        return true;
    }
}
```

---

## ⚙️ A05 — Security Misconfiguration

**Rank**: #5
**Real-world example**: Capital One 2019 — SSRF + misconfigured AWS metadata service → 106M customer records exposed.

### Common Misconfigurations

```
Default credentials:
  - admin/admin on Jenkins, Grafana, Kibana, database admin consoles
  - Never deploy with default credentials

Exposed stack traces:
  - Spring Boot showing full exception with stack trace in production
  - Reveals internal structure, library versions, internal paths

Open cloud storage:
  - AWS S3 bucket set to public read
  - Azure Blob Storage without access keys
  - Exposed internal APIs to public internet

Open ports / services:
  - MongoDB / Redis / Elasticsearch bound to 0.0.0.0 (all interfaces)
  - Should be bound to localhost or VPC internal interface only
```

### Fix

```java
// Spring Boot: hide error details in production
// application-prod.yaml:
server:
  error:
    include-stacktrace: never      # never show stack trace
    include-message: never         # don't expose exception messages
    include-binding-errors: never

// Implement a clean global error handler
@ControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception e, WebRequest request) {
        log.error("Unhandled exception for request {}: ", request.getDescription(false), e);
        // Return generic message to client — never expose internal details
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("An unexpected error occurred. Ref: " + UUID.randomUUID()));
    }
}
```

---

## 📦 A06 — Vulnerable and Outdated Components

**Rank**: #6
**Real-world example**: Equifax 2017 — unpatched Apache Struts CVE-2017-5638 exploited → 147M SSNs exposed.

### Detection and Prevention

```bash
# Maven: Check for vulnerable dependencies
mvn org.owasp:dependency-check-maven:check

# Gradle equivalent:
./gradlew dependencyCheckAnalyze

# Output: HTML report showing CVEs in your dependencies
# Any CRITICAL or HIGH CVEs must be addressed immediately
```

```yaml
# GitHub Dependabot (add to .github/dependabot.yml)
version: 2
updates:
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"     # automatic PRs for security updates
    open-pull-requests-limit: 10
```

---

## 🔑 A07 — Identification and Authentication Failures

**Rank**: #7 (was #2 in 2017 as "Broken Authentication")

### Common Issues

```
Weak password policy:
  → Allowing "123456" or single-character passwords

No MFA on critical accounts:
  → Admin panel accessible with just username/password

Session fixation:
  → Session ID doesn't change after login
  → Pre-auth session ID reused post-auth → attacker can hijack session

Credential stuffing vulnerability:
  → No rate limiting on login → automated attacks trying leaked credentials
```

### Fix

```java
// Spring Security: force new session after login (prevents session fixation)
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            .sessionFixation().newSession()  // new session ID after login
            .maximumSessions(1)              // prevent concurrent sessions
            .maxSessionsPreventsLogin(false) // invalidate old session, not new one
        )
        // ... other config
    return http.build();
}
```

---

## 📦 A08 — Software and Data Integrity Failures

**Rank**: #8 (new in 2021, includes former "Insecure Deserialization")

### Java Deserialization Vulnerabilities

```java
// WRONG — deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject();  // DANGEROUS if stream is from user input
// Attacker can craft a serialized object that executes code on deserialization (gadget chains)
// CVE-2015-4852 (WebLogic), CVE-2016-4437 (Apache Shiro) both exploited this

// CORRECT — never deserialize untrusted data
// If you must: use a safe format (JSON with Jackson), validate with schema first
// Or implement a ObjectInputFilter to whitelist allowed classes:
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(ObjectInputFilter.Config.createFilter(
    "com.myapp.MyAllowedClass;java.util.*;!*"  // whitelist + deny all others
));
```

### Supply Chain Integrity

```
CI/CD pipeline attacks (SolarWinds 2020 — injected malicious code into build pipeline):
  - Verify artifact checksums (SHA-256) before deploying
  - Use signed container images
  - Pin dependency versions (not "latest")
  - Use a private artifact registry with vetted dependencies
```

---

## 📋 A09 — Security Logging and Monitoring Failures

**Rank**: #9

### What to Log

```java
// Security events that MUST be logged:
// - Login success (with user ID, IP, timestamp)
// - Login failure (with username attempted, IP)
// - Password change / reset
// - Privilege changes (user becomes admin)
// - Data exports / large data access
// - API errors and anomalies
// - Admin actions

@Service
public class SecurityAuditLogger {

    private static final Logger securityLog = LoggerFactory.getLogger("SECURITY_AUDIT");

    public void logLoginAttempt(String email, String ipAddress, boolean success) {
        // Structured logging — easily parseable by SIEM tools
        securityLog.info("LOGIN_ATTEMPT email={} ip={} success={} timestamp={}",
                email, ipAddress, success, Instant.now());
    }

    public void logPrivilegeChange(Long adminId, Long targetUserId, String newRole) {
        securityLog.warn("PRIVILEGE_CHANGE admin={} target={} newRole={}",
                adminId, targetUserId, newRole);
    }
}
```

### What NOT to Log

```java
// NEVER log these — constitutes a security vulnerability itself
log.info("User logged in with password: {}", password);  // plaintext credentials in logs!
log.info("Credit card: {}", creditCardNumber);           // PCI-DSS violation
log.info("JWT token: {}", jwtToken);                     // tokens in logs = token theft
log.info("SSN: {}", socialSecurityNumber);               // PII in logs

// Mask sensitive data before logging:
log.info("Credit card ending in {}", creditCard.substring(creditCard.length() - 4));
log.info("User {} logged in", userId);  // log ID, never the password
```

---

## 🌐 A10 — Server-Side Request Forgery (SSRF)

**Rank**: #10 (new in 2021)
**Real-world example**: Capital One 2019 — SSRF vulnerability allowed attacker to query AWS metadata service (169.254.169.254) → retrieved IAM credentials → accessed S3 buckets.

### The Vulnerability

```
Application feature: "Enter a URL to import your profile picture from"
User input: http://169.254.169.254/latest/meta-data/iam/security-credentials/

The server fetches this URL → returns AWS IAM credentials to the attacker.
Other targets: internal APIs, Kubernetes API server, Redis, Elasticsearch.
```

### Fix

```java
// WRONG — blindly fetching user-provided URLs
@PostMapping("/import-image")
public void importImage(@RequestParam String imageUrl) {
    byte[] image = httpClient.get(imageUrl);  // SSRF!
}

// CORRECT — validate URL before fetching
@PostMapping("/import-image")
public void importImage(@RequestParam String imageUrl) {
    URL url = validateUrl(imageUrl);  // throws if invalid
    byte[] image = httpClient.get(url.toString());
}

private URL validateUrl(String urlString) {
    URL url;
    try {
        url = new URL(urlString);
    } catch (MalformedURLException e) {
        throw new BadRequestException("Invalid URL");
    }

    // Whitelist allowed schemes
    if (!List.of("http", "https").contains(url.getProtocol())) {
        throw new BadRequestException("Only HTTP/HTTPS URLs allowed");
    }

    // Block private IP ranges (SSRF protection)
    try {
        InetAddress address = InetAddress.getByName(url.getHost());
        if (address.isLoopbackAddress() ||    // 127.0.0.1
            address.isLinkLocalAddress() ||   // 169.254.x.x (AWS metadata!)
            address.isSiteLocalAddress()) {   // 10.x.x.x, 192.168.x.x, 172.16-31.x.x
            throw new BadRequestException("URL points to a private/internal address");
        }
    } catch (UnknownHostException e) {
        throw new BadRequestException("Cannot resolve hostname");
    }

    return url;
}
```

---

## 📊 Quick Reference Cheat Sheet

| # | Vulnerability | Quick Fix | Severity |
|---|---|---|---|
| A01 | Broken Access Control | Check ownership on every resource access | 🔴 Critical |
| A02 | Cryptographic Failures | BCrypt for passwords; AES-GCM for data; TLS everywhere | 🔴 Critical |
| A03 | Injection | Parameterized queries; never concat user input | 🔴 Critical |
| A04 | Insecure Design | Rate limiting; account lockout; threat modeling | 🟠 High |
| A05 | Security Misconfiguration | Disable defaults; hide errors; bind to localhost | 🟠 High |
| A06 | Vulnerable Components | OWASP Dependency Check; Dependabot; pin versions | 🟠 High |
| A07 | Auth Failures | MFA; session fixation fix; credential stuffing protection | 🟠 High |
| A08 | Data Integrity | No unsafe deserialization; sign artifacts | 🟡 Medium |
| A09 | Logging Failures | Log security events; never log secrets | 🟡 Medium |
| A10 | SSRF | Validate URLs; block private IP ranges | 🟠 High |

---

## 🧩 Mini Challenge

**Code review**: Your colleague submits this Spring Boot endpoint:

```java
@PostMapping("/profile/picture")
public ResponseEntity<String> uploadPictureFromUrl(@RequestParam String url,
                                                    @RequestParam String userId) {
    String pictureData = restTemplate.getForObject(url, String.class);
    userService.updateProfilePicture(userId, pictureData);
    return ResponseEntity.ok("Updated");
}
```

**Identify ALL security vulnerabilities in this code (there are at least 3).**

<details>
<summary>💡 Click to reveal answer</summary>

**Vulnerability 1: SSRF (A10)**
The `url` parameter is user-controlled and fetched directly by the server. Attackers can point it to `http://169.254.169.254/latest/meta-data/` (AWS metadata), internal services, or Redis. Fix: validate URL scheme, resolve hostname and block private IP ranges.

**Vulnerability 2: Broken Access Control (A01)**  
The `userId` is a request parameter — the caller can set it to any user's ID and update THEIR profile picture. Fix: never trust `userId` from request parameters. Always derive it from the authenticated session/JWT: `@AuthenticationPrincipal UserDetails currentUser`.

**Vulnerability 3: Missing input validation (A04/A05)**
No validation on the `url` format (could be `file://`, `ftp://`, `javascript:`, or an empty string). The content fetched (`pictureData`) is stored without validating it's actually an image — could be JavaScript, executable code, or HTML (stored XSS if rendered elsewhere). Fix: validate URL scheme, validate returned content-type is an image, scan content before storing.

**Bonus — Vulnerability 4: Information disclosure**
If the REST call fails (e.g., private URL returns an error), the exception message may leak internal network topology details. Add exception handling that returns a generic error message.

</details>

---

## 📝 Interview Q&A

**Q: What's the difference between SQL injection and NoSQL injection?**
> A: Both involve injecting malicious input into database queries. SQL injection targets relational databases via string concatenation in SQL queries — fix with parameterized queries. NoSQL injection targets document stores (MongoDB) by injecting query operators (`$where`, `$regex`) via JSON input — fix by using typed query builders and never constructing queries from raw strings.

**Q: How do you prevent CSRF attacks?**
> A: CSRF (Cross-Site Request Forgery) is when a malicious site tricks a user's browser into making authenticated requests to another site. Prevention: (1) CSRF tokens — unique per-session token in forms that the server validates; (2) SameSite=Strict cookie attribute — browser won't send cookies to cross-origin requests; (3) Custom request headers (AJAX) — browsers prevent cross-origin requests from setting custom headers. Spring Security enables CSRF protection by default.

**Q: What's the safest way to store user passwords?**
> A: Use an adaptive hashing algorithm designed for passwords: **Argon2id** (current best practice), **bcrypt** (widely supported), or **scrypt**. Never use MD5, SHA-1, or plain SHA-256 — these are general-purpose hash functions that are fast (enabling brute-force) and lack built-in salting. BCrypt/Argon2 automatically salts and are intentionally slow (making brute-force impractical).

---

## 🔗 What to Read Next

1. **[Security/Authentication_vs_Authorization.md](./Authentication_vs_Authorization.md)** — OAuth2 and OIDC flows with security analysis
2. **[Security/TLS_SSL_HTTPS.md](./TLS_SSL_HTTPS.md)** — How TLS prevents the man-in-the-middle attacks covered in A02
3. **[Security/ZeroTrust_Architecture.md](./ZeroTrust_Architecture.md)** — Modern security model that addresses A01, A04, A05 at the architecture level

---

*[← Back to Security](./README.md) | [Back to Index](../INDEX.md)*
