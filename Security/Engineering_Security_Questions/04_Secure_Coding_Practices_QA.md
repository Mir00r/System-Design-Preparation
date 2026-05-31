# 💻 Secure Coding Practices Q&A — Writing Code That Can't Be Exploited

> *"Every vulnerability is a coding decision. SQL injection? Someone concatenated strings instead of using parameters. XSS? Someone forgot to encode output. The attacks are sophisticated, but the PREVENTION is almost always simple — if you know the patterns. This chapter teaches you the coding habits that make vulnerabilities impossible to introduce."*

**⏱️ Estimated Time**: 55 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Web Security Q&A](./03_Web_Security_QA.md)

---

## 📋 Table of Contents
1. [The Secure Coding Mindset](#-the-secure-coding-mindset)
2. [Input Validation — Trust No One](#-input-validation--trust-no-one)
3. [Output Encoding — Context Matters](#-output-encoding--context-matters)
4. [Error Handling — Don't Leak Secrets](#-error-handling--dont-leak-secrets)
5. [Logging — What, How, and What NOT To Log](#-logging--what-how-and-what-not-to-log)
6. [Dependency Security](#-dependency-security)
7. [Secure Configuration](#-secure-configuration)
8. [Race Conditions & TOCTOU](#-race-conditions--toctou)
9. [Memory & Resource Safety](#-memory--resource-safety)
10. [Gamification: Code Review Gauntlet](#-gamification-code-review-gauntlet)
11. [Interview Q&A — 20 Must-Know Questions](#-interview-qa--20-must-know-questions)
12. [What's Next](#-whats-next)

---

## 🧠 The Secure Coding Mindset

```
╔══════════════════════════════════════════════════════════════════╗
║  THE 7 COMMANDMENTS OF SECURE CODING                             ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  1️⃣  ALL input is hostile until validated                        ║
║  2️⃣  Encode output for its context (HTML ≠ JS ≠ SQL ≠ URL)      ║
║  3️⃣  Fail CLOSED — deny by default, allow explicitly            ║
║  4️⃣  Never expose internals in errors (stack traces, SQL errors) ║
║  5️⃣  Authenticate THEN authorize EVERY request                  ║
║  6️⃣  Don't roll your own crypto or auth                         ║
║  7️⃣  Keep secrets out of code, logs, and error messages         ║
║                                                                  ║
║  If you follow these 7 rules, you eliminate 90% of              ║
║  common vulnerabilities without needing to know every attack.    ║
╚══════════════════════════════════════════════════════════════════╝
```

### The Trust Boundary Model

```
         UNTRUSTED ZONE             │  TRUSTED ZONE
  (internet, users, third parties)  │  (your validated, internal data)
                                    │
   ┌──────────┐                     │  ┌──────────────────┐
   │ User     │──── HTTP Request ───┼──▶ VALIDATION GATE  │
   │ Input    │                     │  │ (trust boundary)  │
   └──────────┘                     │  └────────┬─────────┘
                                    │           │
   ┌──────────┐                     │           ▼
   │ Third-   │──── API Response ───┼──▶ VALIDATION GATE
   │ Party API│                     │           │
   └──────────┘                     │           ▼
                                    │  ┌──────────────────┐
   ┌──────────┐                     │  │  Business Logic  │
   │ Database │──── Query Result ───┼──▶  (trusted zone)  │
   │ (if shared)│                   │  └──────────────────┘
   └──────────┘                     │
                                    │
  🚨 RULE: Everything crossing the trust boundary MUST be validated.
  This includes data from YOUR OWN database if it could have been
  tainted by user input that wasn't validated before storage.
```

---

## 🚧 Input Validation — Trust No One

### The Three Lines of Defense

```
LINE 1 — SYNTACTIC VALIDATION (format):
  "Does this LOOK like valid data?"
  → Email format, phone number pattern, date format, UUID structure
  → Reject before any processing

LINE 2 — SEMANTIC VALIDATION (meaning):
  "Does this MAKE SENSE in context?"
  → Age between 0-150, price > 0, date in the future for appointments
  → Business logic constraints

LINE 3 — BUSINESS RULE VALIDATION (authorization):
  "Is this user ALLOWED to do this?"
  → Transfer amount within daily limit, user owns the account
  → Access control checks
```

### Allowlist vs Denylist — Why Allowlist ALWAYS Wins

```java
// ❌ DENYLIST APPROACH (always has bypasses):
public boolean isSafe(String input) {
    String[] blacklist = {"<script>", "DROP TABLE", "' OR", "../"};
    for (String bad : blacklist) {
        if (input.contains(bad)) return false;
    }
    return true; // 🚨 What about <SCRIPT>, <scr\nipt>, %3Cscript%3E, etc.?
}
// Attacker bypasses:
//   <SCRIPT>alert(1)</SCRIPT>           ← case variation
//   <scr<script>ipt>alert(1)</script>  ← nested tags
//   &#x3C;script&#x3E;alert(1)         ← HTML entities
//   %3Cscript%3E                        ← URL encoding

// ✅ ALLOWLIST APPROACH (define what IS valid):
public boolean isValidUsername(String input) {
    // Only allow: letters, numbers, underscores, 3-20 chars
    return input != null && input.matches("^[a-zA-Z0-9_]{3,20}$");
    // Everything else is rejected — no bypass possible!
}

public boolean isValidEmail(String input) {
    // Use a well-tested library, not regex
    try {
        InternetAddress addr = new InternetAddress(input);
        addr.validate();
        return true;
    } catch (AddressException e) {
        return false;
    }
}
```

### Java Validation Patterns

```java
// PATTERN 1: Bean Validation (Jakarta Validation)
public class TransferRequest {
    @NotNull
    @Pattern(regexp = "^[A-Z]{2}[0-9]{2}[A-Z0-9]{11,30}$", message = "Invalid IBAN")
    private String recipientIban;
    
    @NotNull
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    @DecimalMax(value = "50000", message = "Amount exceeds daily limit")
    private BigDecimal amount;
    
    @NotBlank
    @Size(max = 140, message = "Reference too long")
    @Pattern(regexp = "^[a-zA-Z0-9 .,'-]+$", message = "Invalid characters")
    private String reference;
}

@PostMapping("/transfer")
public ResponseEntity<?> transfer(@Valid @RequestBody TransferRequest req) {
    // If we reach here, syntactic validation passed ✅
    // Now do semantic + business validation:
    if (req.getAmount().compareTo(dailyLimitService.getRemainingLimit(currentUser)) > 0) {
        throw new BusinessValidationException("Daily limit exceeded");
    }
    // Process transfer...
}

// PATTERN 2: Sanitize what can't be rejected (rich text)
public String sanitizeHtml(String unsafeHtml) {
    // Use OWASP Java HTML Sanitizer — allows safe HTML, strips dangerous elements
    PolicyFactory policy = new HtmlPolicyBuilder()
        .allowElements("p", "b", "i", "em", "strong", "a", "ul", "li")
        .allowAttributes("href").onElements("a")
        .allowUrlProtocols("https")  // No javascript: URLs
        .toFactory();
    return policy.sanitize(unsafeHtml);
}

// PATTERN 3: Validate numeric IDs (prevent negative/overflow)
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    if (id == null || id <= 0) {
        throw new InvalidParameterException("Invalid user ID");
    }
    // Also: verify current user has access to this user's data (AuthZ)
}
```

### 🎮 Input Validation Puzzle

> **Which inputs would you reject? How would you validate them?**

| Input Field | User Provides | Accept/Reject? | Why? |
|------------|--------------|----------------|------|
| Username | `admin<script>` | <details><summary>?</summary>❌ Reject — contains HTML special chars. Allowlist: `[a-zA-Z0-9_]`</details> |
| Age | `-5` | <details><summary>?</summary>❌ Reject — semantically invalid. Range: 0-150</details> |
| Email | `user@example.com` | <details><summary>?</summary>✅ Accept — valid format. Verify with confirmation email.</details> |
| File path | `../../etc/passwd` | <details><summary>?</summary>❌ Reject — path traversal. Never use user input in file paths.</details> |
| Price | `0.001` | <details><summary>?</summary>Depends — validate against business rules (min order amount)</details> |
| Phone | `+1-555-0123` | <details><summary>?</summary>✅ Accept — but validate format with library (libphonenumber)</details> |

---

## 🎨 Output Encoding — Context Matters

> 🎮 **The cardinal rule**: Data that was safe in a database becomes DANGEROUS when placed into a different context (HTML, JavaScript, URL, CSS). You must encode for the DESTINATION context.

### Different Contexts Need Different Encoding

```
THE SAME DATA in different contexts needs DIFFERENT encoding:

  Data: O'Brien <script>alert(1)</script>
  
  In HTML body:
    <p>O&#x27;Brien &lt;script&gt;alert(1)&lt;/script&gt;</p>
    → HTML entity encoding
  
  In HTML attribute:
    <input value="O&#x27;Brien &lt;script&gt;alert(1)&lt;/script&gt;">
    → HTML attribute encoding (also quote the attribute!)
  
  In JavaScript string:
    var name = "O\x27Brien \x3cscript\x3ealert(1)\x3c/script\x3e";
    → JavaScript hex encoding
  
  In URL parameter:
    ?name=O%27Brien+%3Cscript%3Ealert%281%29%3C%2Fscript%3E
    → URL encoding (percent encoding)
  
  In CSS:
    .user::after { content: "O\27Brien"; }
    → CSS escape encoding

  ⚠️ Using HTML encoding inside a JavaScript string DOES NOT WORK!
     Each context has its own special characters and escape rules.
```

### Java Output Encoding

```java
// Use OWASP Java Encoder library for context-specific encoding
import org.owasp.encoder.Encode;

// HTML body context
String safe = Encode.forHtml(userInput);
// <p>${safe}</p>

// HTML attribute context
String safe = Encode.forHtmlAttribute(userInput);
// <input value="${safe}">

// JavaScript context
String safe = Encode.forJavaScript(userInput);
// <script>var x = '${safe}';</script>

// URL parameter context
String safe = Encode.forUriComponent(userInput);
// <a href="/search?q=${safe}">Search</a>

// CSS context
String safe = Encode.forCssString(userInput);
// <div style="background: url('${safe}')"></div>

// ⚠️ COMMON MISTAKE: Encoding once for wrong context
String htmlEncoded = Encode.forHtml(input);
// ❌ Then using it in JavaScript:
// <script>var x = '${htmlEncoded}';</script>  ← STILL VULNERABLE!
// ✅ Must use JS encoding for JS context:
// <script>var x = '${Encode.forJavaScript(input)}';</script>
```

---

## ⚠️ Error Handling — Don't Leak Secrets

### What Errors Reveal to Attackers

```
❌ BAD ERROR RESPONSES (information goldmine for attackers):

  500: org.postgresql.util.PSQLException: ERROR: relation "users" does not exist
       at org.postgresql.core.v3.QueryExecutorImpl.receiveErrorResponse(...)
       at com.company.UserRepository.findByEmail(UserRepository.java:45)
  
  What attacker learns:
    → Database: PostgreSQL
    → Table name: "users"
    → Stack trace: package structure, framework versions, file names
    → Endpoint behavior: can map the entire codebase

  500: java.io.FileNotFoundException: /var/app/config/database.properties
  
  What attacker learns:
    → Server OS: Linux
    → App path: /var/app/
    → Config file location → target for path traversal

✅ GOOD ERROR RESPONSES (helpful to user, useless to attacker):

  {
    "error": "An unexpected error occurred",
    "requestId": "req_7f3a4b2c",
    "timestamp": "2024-01-15T10:30:00Z"
  }
  
  → Generic message externally
  → requestId for internal correlation (support can look up details)
  → Full stack trace ONLY in internal logs (with the requestId)
```

### Java — Global Error Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    // ✅ Catch-all: never expose internals
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnexpected(Exception ex, HttpServletRequest req) {
        String requestId = UUID.randomUUID().toString();
        
        // Log FULL details internally (for debugging)
        log.error("Unexpected error [requestId={}] [path={}]", requestId, req.getRequestURI(), ex);
        
        // Return MINIMAL info externally (for security)
        return ResponseEntity.status(500).body(new ErrorResponse(
            "An unexpected error occurred. Reference: " + requestId,
            requestId
        ));
    }

    // ✅ Validation errors: tell user WHAT's wrong, not HOW the system works
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "Invalid value"
            ));
        return ResponseEntity.badRequest().body(new ErrorResponse("Validation failed", errors));
    }

    // ✅ Auth errors: don't differentiate between "user not found" and "wrong password"
    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuth(AuthenticationException ex) {
        // ❌ DON'T: "User not found" vs "Invalid password" (user enumeration!)
        // ✅ DO: Generic message for both cases
        return ResponseEntity.status(401).body(new ErrorResponse("Invalid credentials"));
    }

    // ✅ Access denied: don't reveal what resources exist
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        // Consider returning 404 instead of 403 to hide resource existence
        return ResponseEntity.status(404).body(new ErrorResponse("Resource not found"));
    }
}
```

---

## 📝 Logging — What, How, and What NOT To Log

### The Logging Security Matrix

```
╔══════════════════════════════════════════════════════════════════╗
║                    LOGGING SECURITY RULES                         ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ✅ DO LOG (security-relevant events):                           ║
║    • Authentication attempts (success AND failure)               ║
║    • Authorization failures (403s)                               ║
║    • Input validation failures (potential attacks)               ║
║    • Privilege changes (role assignments)                        ║
║    • Data access patterns (who accessed what)                    ║
║    • System events (startup, shutdown, config changes)           ║
║    • Rate limit hits (potential brute force/scraping)            ║
║                                                                  ║
║  ❌ NEVER LOG (sensitive data):                                  ║
║    • Passwords (plain or hashed)                                 ║
║    • Session tokens / JWTs / API keys                           ║
║    • Credit card numbers (PCI violation)                         ║
║    • Social Security Numbers / personal IDs                      ║
║    • Private keys or certificates                                ║
║    • Full request bodies containing PII                          ║
║    • Database connection strings with passwords                  ║
║                                                                  ║
║  ⚠️ LOG WITH CARE (mask/truncate):                              ║
║    • Email addresses → "u***@example.com"                        ║
║    • Phone numbers → "***-***-5678"                              ║
║    • IP addresses → depends on jurisdiction (GDPR considers PII) ║
║    • User IDs → OK for internal correlation                      ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### Secure Logging Implementation

```java
// ❌ DANGEROUS LOGGING:
log.info("User login: email={}, password={}", email, password); // 🚨 PASSWORD IN LOGS!
log.info("Payment: card={}", creditCardNumber);                 // 🚨 PCI VIOLATION!
log.debug("JWT: {}", jwtToken);                                // 🚨 SESSION IN LOGS!
log.error("DB error: {}", exception.getMessage());             // 🚨 May contain SQL/data

// ✅ SECURE LOGGING:
log.info("Login attempt: email={}, result={}, ip={}", 
    maskEmail(email), success ? "SUCCESS" : "FAILURE", clientIp);
log.info("Payment processed: last4={}, amount={}, userId={}", 
    last4Digits, amount, userId);
log.warn("Auth failure: userId={}, resource={}, action={}", 
    userId, resourceId, action);

// Masking utility
public static String maskEmail(String email) {
    if (email == null) return "null";
    int atIdx = email.indexOf('@');
    if (atIdx <= 1) return "***" + email.substring(atIdx);
    return email.charAt(0) + "***" + email.substring(atIdx);
}

// Log injection prevention (attacker puts \n in input to forge log entries)
public static String sanitizeForLog(String input) {
    if (input == null) return "null";
    return input.replaceAll("[\\r\\n\\t]", "_")  // Remove newlines
                .substring(0, Math.min(input.length(), 200)); // Limit length
}
```

---

## 📦 Dependency Security

> 🎮 **Analogy**: You're building a house with bricks from 100 different suppliers. If ONE supplier's bricks contain asbestos, your entire house is contaminated. Open-source dependencies are those bricks.

### The Supply Chain Problem

```
YOUR APP'S DEPENDENCY TREE:

  your-app (your code: audited ✅)
    ├── spring-boot-starter-web (well-maintained ✅)
    │   ├── spring-web
    │   ├── jackson-databind
    │   │   └── jackson-core
    │   └── tomcat-embed-core
    ├── log4j-core ← 🚨 LOG4SHELL (CVE-2021-44228)
    ├── commons-collections ← 🚨 Deserialization gadgets
    ├── some-utility-lib (unmaintained, 3 years old ⚠️)
    │   └── vulnerable-transitive-dep ← 🚨 You didn't even know this existed!
    └── ...200 more dependencies

  REALITY:
    → Average Java app has 200-500 transitive dependencies
    → You wrote 10% of the code running in production
    → The other 90% was written by strangers on the internet
    → Log4Shell affected virtually every Java application on Earth
```

### Defense Strategy

```java
// 1. AUDIT: Know what you depend on
// Run: mvn dependency:tree or gradle dependencies
// Use: OWASP Dependency-Check, Snyk, GitHub Dependabot

// 2. PIN VERSIONS: Prevent unexpected updates
// pom.xml — use exact versions, not ranges
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.3</version>  <!-- Exact version, not [2.15,) -->
</dependency>

// 3. LOCK FILE: Ensure reproducible builds
// Maven: use maven-dependency-plugin:resolve-plugins
// Gradle: use gradle.lockfile

// 4. AUTOMATE SCANNING: CI/CD pipeline check
// GitHub Actions example:
// - name: Check for vulnerable dependencies
//   run: mvn org.owasp:dependency-check-maven:check -DfailBuildOnCVSS=7

// 5. MINIMIZE: Every dependency is a risk — remove unused ones
// Question: Do you REALLY need a library for left-pad?
```

---

## 🔒 Secure Configuration

### Configuration Security Checklist

```
╔══════════════════════════════════════════════════════════════════╗
║  PRODUCTION CONFIGURATION SECURITY                               ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  ❌ DISABLE IN PRODUCTION:                                       ║
║    • Debug mode / stack traces in responses                      ║
║    • H2 console / database admin panels                          ║
║    • Swagger/OpenAPI docs (or restrict to internal network)      ║
║    • Spring Boot Actuator (or restrict endpoints)                ║
║    • Verbose error messages                                      ║
║    • CORS wildcard (*)                                           ║
║    • Default credentials (admin/admin)                           ║
║                                                                  ║
║  ✅ ENABLE IN PRODUCTION:                                        ║
║    • HTTPS only (redirect HTTP → HTTPS)                          ║
║    • Security headers (HSTS, CSP, X-Frame-Options)              ║
║    • Rate limiting                                               ║
║    • Access logging / audit trail                                ║
║    • Secrets from external source (Vault, env vars)              ║
║    • Minimum log level: WARN (no DEBUG in prod!)                 ║
║    • Connection pool limits                                      ║
║    • Request size limits                                         ║
╚══════════════════════════════════════════════════════════════════╝
```

```yaml
# application-prod.yml — Secure Spring Boot configuration
server:
  error:
    include-stacktrace: never       # ✅ No stack traces
    include-message: never          # ✅ No exception messages
  servlet:
    session:
      cookie:
        secure: true                # ✅ HTTPS only
        http-only: true             # ✅ No JS access
        same-site: lax             # ✅ CSRF protection

spring:
  h2.console.enabled: false         # ✅ No DB console
  devtools.restart.enabled: false   # ✅ No dev tools
  jpa:
    show-sql: false                 # ✅ No SQL in logs
    open-in-view: false             # ✅ No lazy-loading surprises

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus  # ✅ Minimal actuator
  endpoint:
    health:
      show-details: never          # ✅ No internal details

logging:
  level:
    root: WARN                     # ✅ No debug noise
    com.company: INFO
    org.springframework.security: WARN
```

---

## ⏱️ Race Conditions & TOCTOU

> 🎮 **Analogy**: You check your bank balance (it shows $100). You initiate a $100 transfer. But between checking and transferring, another process also initiated a $100 transfer. Now you've spent $200 from a $100 account. The check and the action happened at different times — that's TOCTOU.

### Time-of-Check to Time-of-Use (TOCTOU)

```java
// ❌ RACE CONDITION: Balance checked, then deducted separately
@PostMapping("/transfer")
public ResponseEntity<?> transfer(@RequestBody TransferRequest req) {
    Account account = accountRepo.findById(req.getFromAccount());
    
    // TIME-OF-CHECK: balance is sufficient
    if (account.getBalance().compareTo(req.getAmount()) >= 0) {
        // ⚠️ WINDOW OF VULNERABILITY: Another thread could deduct here!
        
        // TIME-OF-USE: deduct the amount
        account.setBalance(account.getBalance().subtract(req.getAmount()));
        accountRepo.save(account);
        // If two requests hit simultaneously, both pass the check, both deduct
        // Result: negative balance (double-spending)
    }
}

// ✅ FIX 1: Pessimistic locking (database-level lock)
@Transactional
public void transfer(TransferRequest req) {
    // SELECT ... FOR UPDATE — locks the row until transaction commits
    Account account = accountRepo.findByIdForUpdate(req.getFromAccount());
    if (account.getBalance().compareTo(req.getAmount()) >= 0) {
        account.setBalance(account.getBalance().subtract(req.getAmount()));
    } else {
        throw new InsufficientFundsException();
    }
}

// ✅ FIX 2: Optimistic locking (version check)
@Entity
public class Account {
    @Version
    private Long version; // JPA checks this on save — fails if concurrent modification
}

// ✅ FIX 3: Atomic operation (single SQL statement — no race possible)
@Modifying
@Query("UPDATE Account a SET a.balance = a.balance - :amount WHERE a.id = :id AND a.balance >= :amount")
int debitIfSufficient(@Param("id") Long id, @Param("amount") BigDecimal amount);
// Returns 0 if balance insufficient or concurrent modification — no race condition!
```

### Common Race Condition Scenarios

| Scenario | Exploit | Fix |
|----------|---------|-----|
| Coupon redemption | Apply same coupon twice in parallel | Unique constraint + idempotency key |
| Limited stock purchase | Buy last item twice simultaneously | `UPDATE stock SET qty = qty - 1 WHERE qty > 0` |
| Vote/like buttons | Send 100 concurrent like requests | Rate limit + unique constraint per user |
| Account creation | Register same email in two parallel requests | Unique constraint on email |
| File upload | Check file doesn't exist, then write | Atomic create-if-not-exists operation |

---

## 🧠 Memory & Resource Safety

### Resource Exhaustion Attacks

```java
// ❌ VULNERABLE: Unbounded resource consumption
@PostMapping("/upload")
public void upload(@RequestParam MultipartFile file) {
    byte[] content = file.getBytes(); // 🚨 What if file is 10GB? → OOM
}

@GetMapping("/search")
public List<Product> search(@RequestParam String q) {
    return repo.findByNameContaining(q); // 🚨 Could return 10M rows → OOM
}

@PostMapping("/parse")
public void parseXml(@RequestBody String xml) {
    DocumentBuilder db = DocumentBuilderFactory.newInstance().newDocumentBuilder();
    db.parse(new InputSource(new StringReader(xml))); // 🚨 XXE + Billion Laughs
}

// ✅ SAFE: Bounded resource consumption
@PostMapping("/upload")
public void upload(@RequestParam MultipartFile file) {
    if (file.getSize() > 10_000_000) { // 10MB max
        throw new FileTooLargeException();
    }
    // Process in streaming fashion for large files
}

@GetMapping("/search")
public Page<Product> search(@RequestParam String q,
                            @RequestParam(defaultValue = "20") @Max(100) int size) {
    return repo.findByNameContaining(q, PageRequest.of(0, size));
}

@PostMapping("/parse")
public void parseXml(@RequestBody String xml) {
    DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
    // Disable XXE
    dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
    dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
    dbf.setExpandEntityReferences(false);
    // Set limits
    dbf.setAttribute("http://www.oracle.com/xml/jaxp/properties/maxElementDepth", "100");
}
```

### ReDoS — Regular Expression Denial of Service

```java
// ❌ EVIL REGEX: Catastrophic backtracking
Pattern emailPattern = Pattern.compile("^([a-zA-Z0-9]+)*@example\\.com$");
// Input: "aaaaaaaaaaaaaaaaaaaaaaaaaaaa!" → Takes MINUTES to evaluate!

// Why? The (a+)* causes exponential backtracking:
// Engine tries every possible way to split "aaa...a" between the groups
// Each additional character DOUBLES the time

// ✅ SAFE REGEX: No nested quantifiers
Pattern emailPattern = Pattern.compile("^[a-zA-Z0-9]+@example\\.com$");
// No catastrophic backtracking possible

// ✅ EVEN BETTER: Set a timeout
Matcher matcher = pattern.matcher(input);
// In Java 9+: you can't directly set regex timeout
// Alternative: Run in a thread with timeout
ExecutorService exec = Executors.newSingleThreadExecutor();
Future<Boolean> future = exec.submit(() -> pattern.matcher(input).matches());
try {
    boolean result = future.get(100, TimeUnit.MILLISECONDS); // 100ms max
} catch (TimeoutException e) {
    future.cancel(true);
    throw new ValidationException("Input too complex");
}
```

---

## 🎮 Gamification: Code Review Gauntlet

### 🏰 Level 1: Find the Vulnerability (Score /10)

Review each code snippet. Identify the vulnerability and propose a fix.

**Challenge 1:**
```java
@GetMapping("/download")
public ResponseEntity<Resource> download(@RequestParam String filename) {
    Path file = Paths.get("/uploads/" + filename);
    Resource resource = new FileSystemResource(file);
    return ResponseEntity.ok().body(resource);
}
```
<details>
<summary>🔍 Vulnerability & Fix</summary>

**Vulnerability**: Path Traversal — `filename=../../etc/passwd` reads system files.

**Fix**:
```java
Path basePath = Paths.get("/uploads/").toAbsolutePath().normalize();
Path file = basePath.resolve(filename).toAbsolutePath().normalize();
if (!file.startsWith(basePath)) {
    throw new SecurityException("Path traversal detected");
}
```
</details>

**Challenge 2:**
```java
@PostMapping("/register")
public ResponseEntity<?> register(@RequestBody UserDTO user) {
    if (userRepo.existsByEmail(user.getEmail())) {
        return ResponseEntity.badRequest().body("Email already registered");
    }
    // ... create user
}
```
<details>
<summary>🔍 Vulnerability & Fix</summary>

**Vulnerability**: User Enumeration — Attacker can determine which emails are registered by checking the response. Useful for targeted phishing or credential stuffing.

**Fix**: Always return the same response regardless of whether the email exists:
```java
return ResponseEntity.ok().body("If this email is registered, you'll receive a confirmation.");
// Then handle duplicate internally (unique constraint in DB)
```
</details>

**Challenge 3:**
```java
public void processOrder(HttpServletRequest request) {
    String userId = request.getParameter("userId");
    String role = request.getParameter("role");
    if ("admin".equals(role)) {
        // Grant admin privileges for this operation
        orderService.processAsAdmin(userId);
    }
}
```
<details>
<summary>🔍 Vulnerability & Fix</summary>

**Vulnerability**: Client-controlled authorization — User can set `role=admin` in the request to escalate privileges.

**Fix**: NEVER trust client-provided role/permission information. Derive from server-side session/token:
```java
Authentication auth = SecurityContextHolder.getContext().getAuthentication();
if (auth.getAuthorities().contains(new SimpleGrantedAuthority("ROLE_ADMIN"))) {
    orderService.processAsAdmin(auth.getName());
}
```
</details>

**Challenge 4:**
```java
@PostMapping("/feedback")
public String submitFeedback(@RequestParam String message) {
    String response = "<div class='feedback'>Thank you! You said: " + message + "</div>";
    return response;
}
```
<details>
<summary>🔍 Vulnerability & Fix</summary>

**Vulnerability**: Reflected XSS — User input directly placed in HTML without encoding.

**Fix**: Use output encoding or template engine with auto-escaping:
```java
String safe = Encode.forHtml(message);
String response = "<div class='feedback'>Thank you! You said: " + safe + "</div>";
```
Or better: use Thymeleaf with `th:text` which auto-escapes.
</details>

---

## 🎯 Interview Q&A — 20 Must-Know Questions

**Q1: What is input validation? Allowlist vs denylist?**

> **A**: Input validation ensures data matches expected format/constraints before processing. Allowlist (whitelist) defines what IS valid — reject everything else. Denylist (blacklist) defines what ISN'T valid — accept everything else. Always prefer allowlist because denylists have infinite bypasses (encoding tricks, case variations, Unicode). Example: for a username field, allow `[a-zA-Z0-9_]{3,20}` and reject everything else.

**Q2: How do you prevent information leakage through error messages?**

> **A**: (1) Use a global exception handler that returns generic messages externally, (2) Include a request ID for correlation with internal logs, (3) Log full details (stack traces) only internally, (4) Never differentiate between "user not found" and "wrong password", (5) Consider returning 404 instead of 403 to hide resource existence, (6) Disable stack traces in production (`server.error.include-stacktrace: never`).

**Q3: What should and shouldn't be logged from a security perspective?**

> **A**: LOG: auth events, authz failures, validation failures, privilege changes, data access, rate limit hits. NEVER LOG: passwords, tokens, API keys, credit cards, SSNs, full request bodies with PII. MASK: emails, phone numbers, IPs (GDPR). Also prevent log injection: sanitize user input before including in log messages (remove newlines that could forge entries).

**Q4: What is a TOCTOU vulnerability? Give a real example.**

> **A**: Time-of-Check to Time-of-Use: a race condition where the state changes between when you verify a condition and when you act on it. Example: check balance >= $100, then deduct $100 — if two requests hit simultaneously, both pass the check, resulting in -$100 balance. Fix: atomic operations (single SQL update with WHERE clause), pessimistic locking (SELECT FOR UPDATE), or optimistic locking (@Version annotation).

**Q5: How do you handle secrets in application configuration?**

> **A**: Never in source code, never in config files committed to git, never in environment variables visible in process listings. Use: (1) Secrets manager (Vault, AWS Secrets Manager) with runtime injection, (2) Kubernetes secrets (encrypted at rest), (3) Rotate regularly, (4) Scope secrets per service/environment, (5) Audit access to secrets, (6) Revoke immediately on suspected exposure.

**Q6: What is supply chain security? How did Log4Shell demonstrate the risk?**

> **A**: Supply chain security addresses risks from third-party dependencies, build tools, and deployment pipelines. Log4Shell (CVE-2021-44228): a single vulnerable dependency (log4j-core) was present in virtually every Java application. Attacker could achieve remote code execution by logging a crafted string. Defense: dependency scanning, SBOM, minimal dependencies, pinned versions, automated vulnerability alerts, and incident response plan for zero-day disclosures.

**Q7: Explain the concept of "defense in depth" in code.**

> **A**: Multiple validation layers so if one fails, others catch it. Example for a payment endpoint: (1) WAF blocks known attack patterns, (2) Rate limiter prevents brute force, (3) Input validation rejects malformed data, (4) Business logic validates amount/limits, (5) Parameterized queries prevent injection even if validation missed something, (6) Least-privilege DB user limits damage, (7) Audit log enables detection after the fact.

**Q8: How do you prevent ReDoS attacks?**

> **A**: ReDoS exploits regexes with catastrophic backtracking (exponential time on crafted input). Prevention: (1) Avoid nested quantifiers like `(a+)*`, (2) Use possessive quantifiers `a++` or atomic groups, (3) Set regex evaluation timeout, (4) Validate input length before regex evaluation, (5) Use linear-time regex engines (RE2, Rust's regex) for user-controlled patterns, (6) Test regexes with ReDoS analysis tools before deployment.

**Q9: What are the security risks of deserialization in Java?**

> **A**: Java deserialization (`ObjectInputStream`) can execute arbitrary code if the attacker controls the serialized data and gadget classes exist in classpath. Gadget chains (commons-collections, spring, etc.) allow RCE. Prevention: (1) Never deserialize untrusted data, (2) Use JSON (Jackson/Gson) instead of Java serialization, (3) If unavoidable: use allowlists (`ObjectInputFilter` in Java 9+), (4) Remove unnecessary gadget libraries, (5) Use `serialVersionUID` and custom `readObject()` methods.

**Q10: How do you prevent mass assignment vulnerabilities?**

> **A**: Mass assignment: attacker sends extra fields that get bound to internal object properties. Example: user sends `{"name":"Alice","role":"ADMIN"}` and the framework auto-binds `role`. Prevention: (1) Use DTOs with only the expected fields (not entities directly), (2) Use `@JsonIgnore` on sensitive fields, (3) Explicitly map allowed fields from DTO to entity, (4) Jackson's `@JsonProperty(access = READ_ONLY)` for fields that should never be set by client.

**Q11: What is the principle of "fail secure"? Give a code example.**

> **A**: Fail secure means when something goes wrong, default to DENYING access rather than granting it.
```java
// ❌ Fail OPEN (dangerous):
boolean isAllowed = false;
try { isAllowed = authService.checkPermission(user, resource); }
catch (Exception e) { isAllowed = true; } // If auth service is down, allow everything!

// ✅ Fail CLOSED (secure):
boolean isAllowed = false;
try { isAllowed = authService.checkPermission(user, resource); }
catch (Exception e) { isAllowed = false; log.error("Auth check failed", e); }
// If auth service is down, deny everything
```

**Q12: How would you implement idempotency to prevent duplicate processing?**

> **A**: Use an idempotency key (unique identifier per request): (1) Client sends `Idempotency-Key: uuid` header, (2) Server checks if this key was already processed (in DB/Redis), (3) If already processed: return cached response, (4) If new: process, store result with key, return response. This prevents duplicate charges from retries, race conditions on double-clicks, and network timeout replays. Stripe uses this pattern for all payment APIs.

**Q13: What is output encoding and why is it different from input validation?**

> **A**: Input validation restricts what comes IN (reject bad data). Output encoding transforms what goes OUT (make data safe for its destination context). Both are needed because: validation might miss something, data might be used in an unexpected context later, or legacy data in the DB was never validated. Output encoding is context-specific: HTML entities for HTML, percent-encoding for URLs, hex escapes for JavaScript.

**Q14: How do you secure sensitive data in application memory?**

> **A**: (1) Use `char[]` instead of `String` for passwords (strings are immutable, stay in memory until GC), (2) Zero-out sensitive arrays after use: `Arrays.fill(password, '\0')`, (3) Minimize time sensitive data exists in memory, (4) Don't log/print sensitive values even in debug mode, (5) Use `SecureString` patterns where available, (6) Consider memory-encrypted containers for HSM integration. Note: Java's GC makes this imperfect — true security requires hardware-backed solutions.

**Q15: What secure coding practices should be enforced in CI/CD?**

> **A**: (1) SAST (Static Analysis): SonarQube, Semgrep — catch code-level vulnerabilities, (2) SCA (Dependency Scanning): Snyk, OWASP Dependency-Check — known vulnerable libraries, (3) Secrets detection: git-secrets, TruffleHog — credentials in code, (4) Container scanning: Trivy — vulnerable base images, (5) DAST (Dynamic Analysis): OWASP ZAP — runtime vulnerability scanning, (6) Unit tests for security controls (auth, validation), (7) Mandatory code review for security-sensitive changes.

**Q16: What is the "confused deputy" problem in access control?**

> **A**: A confused deputy is a program with elevated privileges that's tricked into acting on behalf of an attacker. Example: A file server (deputy) has filesystem access. User requests `/files?name=../../etc/shadow` — server uses its OWN privileges to read the file. Fix: Always validate that the ORIGINAL requester has permission, not just that the server can technically perform the operation. This is the root cause of path traversal and SSRF.

**Q17: How do you protect against XML External Entity (XXE) attacks?**

> **A**: XXE exploits XML parsers that process external entity references: `<!ENTITY xxe SYSTEM "file:///etc/passwd">`. Attacker can read local files, SSRF, or DoS (billion laughs). Fix: Disable external entities in the parser:
```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
// Or use JSON instead of XML where possible.
```

**Q18: What is Content-Type confusion and how do you prevent it?**

> **A**: MIME sniffing: browser ignores Content-Type header and guesses content type from file contents. Attacker uploads HTML file with `.jpg` extension → browser renders it as HTML → XSS. Prevention: (1) `X-Content-Type-Options: nosniff` header (forces browser to trust Content-Type), (2) Always set correct Content-Type, (3) Validate uploaded file content matches declared type, (4) Serve user uploads from a separate domain.

**Q19: What are insecure direct object references (IDOR)?**

> **A**: IDOR occurs when an application exposes internal object identifiers (database IDs, filenames) and doesn't verify the requesting user has permission to access them. Example: `/api/invoices/12345` — changing ID to `12346` shows another user's invoice. Fix: (1) Always verify ownership/permission on every request, (2) Use UUIDs instead of sequential IDs (harder to guess, though not a security control by itself), (3) Access control logic: `WHERE id = :id AND user_id = :currentUserId`.

**Q20: How would you conduct a secure code review? What do you look for?**

> **A**: Checklist: (1) Input validation on all external data, (2) Parameterized queries (no string concat in SQL), (3) Output encoding for HTML/JS contexts, (4) Authentication/authorization on every endpoint, (5) Sensitive data handling (no passwords in logs, proper encryption), (6) Error handling (no internal details exposed), (7) Race conditions in state-changing operations, (8) Dependency versions (known vulnerabilities), (9) Hardcoded secrets, (10) Resource limits (file sizes, query results). Use SAST tools to automate, but manual review catches logic flaws tools miss.

---

## 📊 Secure Coding Quick Reference

```
╔══════════════════════════════════════════════════════════════════╗
║  SECURE CODING — THE 1-MINUTE CHEAT SHEET                       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  Input:     Validate with allowlist, reject unknown              ║
║  Output:    Encode for context (HTML/JS/URL/SQL)                 ║
║  Database:  Parameterized queries ONLY                           ║
║  Errors:    Generic to user, detailed in internal logs           ║
║  Logging:   Never log secrets, always log security events        ║
║  Auth:      Check on EVERY request, fail closed                  ║
║  Secrets:   External source, rotate, audit, scope                ║
║  Config:    Disable debug/unnecessary features in production     ║
║  Deps:      Scan, pin versions, minimize, SBOM                   ║
║  Race:      Atomic operations or pessimistic locking             ║
║  Resources: Bound everything (file size, query results, regex)   ║
║  Crypto:    Use libraries (AES-GCM, bcrypt), never roll your own ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Topic | Link | Why Read It? |
|-------|------|-------------|
| Java Security Deep Dive | [05_Java_Security_Deep_Dive_QA.md](./05_Java_Security_Deep_Dive_QA.md) | Java-specific security patterns with Spring |
| Security Scenarios | [06_Security_Scenarios_And_Puzzles.md](./06_Security_Scenarios_And_Puzzles.md) | Apply secure coding in realistic scenarios |
| OWASP Top 10 | [../OWASP_Top10.md](../OWASP_Top10.md) | See these vulnerabilities in the wild |

---

*[← Web Security Q&A](./03_Web_Security_QA.md) | [Back to Security Index](../README.md) | [Next: Java Security Deep Dive →](./05_Java_Security_Deep_Dive_QA.md)*
