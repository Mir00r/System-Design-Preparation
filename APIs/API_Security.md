# 🔐 API Security: Protecting Your APIs from Attacks 🚀

> **"An API without security is like a bank vault with a screen door — technically a vault, but not doing its job."**

**⏱️ Estimated Time**: 35 minutes  
**🎯 Difficulty**: 🟡 Intermediate  
**🔗 Prerequisites**: [RESTful APIs](./RESTful.md) | [Authentication vs Authorization](../Security/Authentication_vs_Authorization.md) | [JWT Deep Dive](../Security/JWT_Deep_Dive.md)

---

## 📋 Table of Contents

1. [Why API Security Fails](#-why-api-security-fails)
2. [The OWASP API Security Top 10](#-the-owasp-api-security-top-10)
3. [Authentication Patterns](#-authentication-patterns)
4. [Authorization — Fine-Grained Access Control](#-authorization)
5. [Transport Security (HTTPS/TLS)](#-transport-security)
6. [Input Validation & Injection Prevention](#-input-validation)
7. [Rate Limiting & Abuse Prevention](#-rate-limiting--abuse-prevention)
8. [API Keys — Best Practices](#-api-keys)
9. [Security Headers](#-security-headers)
10. [Spring Boot Security Implementation](#-spring-boot-security-implementation)
11. [Common Pitfalls](#-common-pitfalls)
12. [Interview Q&A](#-interview-qa)
13. [Mini Challenge](#-mini-challenge)
14. [What to Read Next](#-what-to-read-next)

---

## 💀 Why API Security Fails

It's Friday afternoon. A developer pushes a "quick fix" to production. The change accidentally exposes a debug endpoint `/api/v1/admin/users` without authentication. By Monday morning, 2 million user records have been scraped by a bot that discovered the endpoint through Google indexing.

This is not hypothetical. This exact scenario has happened at Venmo, Facebook, and dozens of other companies. APIs are the most attacked surface in modern software because:

```
WHY APIS ARE ATTACK TARGETS:
─────────────────────────────────────────────────────────────────────
1. APIs expose business logic directly (not just data)
2. APIs are machine-readable — bots can enumerate ALL endpoints
3. Mobile apps embed API keys in source code (easily extracted)
4. APIs often skip authentication on "internal" endpoints
5. Developer convenience vs security is always a tension

THE DAMAGE SCALE (Real incidents):
  Peloton API (2021): 4M user profiles exposed, no auth required
  Bumble API (2021): 100M+ user profiles scraped via IDOR
  Twitter API (2022): 5.4M accounts scraped via phone number lookup
  Australia Post (2022): Customer data leaked via misconfigured API
```

```
BEFORE (Insecure API):          AFTER (Secure API):
  GET /users/12345                GET /users/12345
  → Returns ANY user's data       → Returns ONLY the calling user's data
                                   (ownership check enforced)
  POST /transfer                  POST /transfer
  → No idempotency key            → Idempotency key required
  → Can replay to double-charge   → Replay rejected with 409 Conflict
```

---

## 🎯 The OWASP API Security Top 10

OWASP (Open Web Application Security Project) publishes the authoritative list of API vulnerabilities. Every API developer must know these:

```
OWASP API SECURITY TOP 10 (2023):
────────────────────────────────────────────────────────────────────────────────
#1  BOLA — Broken Object Level Authorization       ← Most common (40% of attacks)
#2  Broken Authentication                          ← Stolen/weak tokens
#3  Broken Object Property Level Authorization    ← Mass assignment attacks
#4  Unrestricted Resource Consumption              ← No rate limiting
#5  Broken Function Level Authorization            ← Admin endpoints exposed
#6  Unrestricted Access to Sensitive Business Flows ← Business logic bypass
#7  SSRF — Server-Side Request Forgery             ← Making server fetch URLs
#8  Security Misconfiguration                      ← Debug mode in prod
#9  Improper Inventory Management                  ← Forgotten old API versions
#10 Unsafe Consumption of APIs                     ← Trusting 3rd-party APIs
```

### #1 BOLA — The Most Common Vulnerability

```java
// ❌ VULNERABLE — BOLA (Broken Object Level Authorization):
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId) {
    return orderService.findById(orderId); // Returns ANY user's order!
    // Attacker changes orderId from 1001 to 1002 → sees other user's order
}

// ✅ SECURE — Object-level ownership check:
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId, 
                      @AuthenticationPrincipal UserDetails user) {
    Order order = orderService.findById(orderId);
    if (!order.getUserId().equals(user.getUserId())) {
        throw new AccessDeniedException("Order does not belong to user");
    }
    return order;
}
```

### #3 Broken Object Property Level Authorization (Mass Assignment)

```java
// ❌ VULNERABLE — Mass assignment allows role escalation:
@PutMapping("/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody User userUpdate) {
    User existing = userRepo.findById(id);
    // Direct bind copies ALL fields including 'role', 'isAdmin', 'creditBalance'!
    BeanUtils.copyProperties(userUpdate, existing);
    return userRepo.save(existing);
}

// ✅ SECURE — Use a dedicated DTO with only allowed fields:
record UpdateUserRequest(String firstName, String lastName, String email) {}

@PutMapping("/users/{id}")
public User updateUser(@PathVariable Long id, 
                       @RequestBody UpdateUserRequest req,
                       @AuthenticationPrincipal UserDetails currentUser) {
    // Only firstName/lastName/email can be updated — role is never touched
    return userService.updateProfile(id, req, currentUser);
}
```

---

## 🔑 Authentication Patterns

```
AUTHENTICATION METHOD    BEST FOR             AVOID WHEN
─────────────────────────────────────────────────────────────────────
API Keys                 Server-to-server     User-facing mobile apps
                         Simple integrations  (keys end up in app binaries)

JWT Bearer Tokens        Stateless APIs       Long sessions (no revocation)
                         Microservices        Without proper expiry (<1h)

OAuth 2.0 + OIDC         User-delegated auth  Internal machine-to-machine
                         Third-party access   (too complex for simple uses)

Session Cookies          Browser SPAs         Cross-origin mobile apps
                         Web applications     (CSRF risk without SameSite)

mTLS (Mutual TLS)        Service-to-service   Public-facing consumer APIs
                         Zero-trust networks  (client cert management cost)
```

### JWT Best Practices

```java
// ✅ JWT token validation with all security checks:
@Component
public class JwtValidator {
    
    public Claims validateToken(String token) {
        return Jwts.parserBuilder()
            .setSigningKey(getSecretKey())           // Verify signature
            .requireIssuer("https://auth.myapp.com") // Validate issuer
            .requireAudience("api.myapp.com")        // Validate audience
            .build()
            .parseClaimsJws(token)
            .getBody();
        // Will throw JwtException if:
        // - Signature invalid
        // - Token expired (exp claim)
        // - Issuer/audience mismatch
        // - Token not yet valid (nbf claim)
    }
    
    // ✅ Use RS256 (asymmetric) not HS256 (symmetric) for distributed systems:
    // HS256: same secret key signs AND verifies → all services need the secret
    // RS256: private key signs, public key verifies → safe to distribute public key
    private Key getSecretKey() {
        // Load RSA private key from secrets vault (never hardcode!)
        return rsaKeyLoader.loadPrivateKey();
    }
}
```

---

## 🛡️ Authorization

### Role-Based vs Scope-Based Access Control

```
RBAC (Role-Based):                    SCOPES (OAuth/API):
  User has ROLE_ADMIN                   Token has scopes: [read:orders, write:orders]
  ROLE_ADMIN can access /admin/*        Token can only read/write orders
  
  Problem: All admins get ALL admin     Benefit: Token has minimum necessary
           permissions                           permissions (principle of least privilege)
```

```java
// Spring Security: method-level authorization
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    @GetMapping                           // Any authenticated user
    @PreAuthorize("isAuthenticated()")
    public List<Order> listMyOrders(...) { ... }
    
    @DeleteMapping("/{id}")               // Only ADMIN role
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(...) { ... }
    
    @PostMapping("/bulk-process")         // Specific scope (OAuth)
    @PreAuthorize("hasAuthority('SCOPE_orders:process')")
    public BulkResult processBulk(...) { ... }
    
    @GetMapping("/{id}/payment")          // Owner OR admin
    @PreAuthorize("@orderSecurity.isOwner(#id, authentication) or hasRole('ADMIN')")
    public PaymentInfo getPayment(@PathVariable Long id) { ... }
}
```

---

## 🔒 Transport Security

```
RULE: ALL API traffic MUST use TLS 1.2+ (ideally TLS 1.3).
      HTTP (unencrypted) is unacceptable for any production API.

TLS 1.3 IMPROVEMENTS OVER TLS 1.2:
  ✅ 0-RTT resumption (faster reconnections)
  ✅ Forward secrecy always on (compromise key ≠ decrypt past traffic)
  ✅ Removed weak cipher suites (RC4, MD5, SHA-1)
  ✅ 1-RTT handshake (vs 2-RTT in TLS 1.2)
```

```yaml
# Spring Boot: enforce HTTPS, HSTS, and modern TLS
server:
  ssl:
    enabled: true
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2
    ciphers: TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256

# application.properties  
security.require-ssl=true
```

```java
// Redirect HTTP → HTTPS
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http.requiresChannel(channel ->
        channel.anyRequest().requiresSecure() // Force HTTPS
    );
    return http.build();
}
```

---

## 🧹 Input Validation

```java
// ❌ SQL Injection vulnerable:
@GetMapping("/search")
public List<Product> search(@RequestParam String query) {
    return jdbcTemplate.query(
        "SELECT * FROM products WHERE name = '" + query + "'",
        // Attacker sends: ' OR '1'='1  → dumps entire table!
        ...
    );
}

// ✅ Parameterized query (prevents SQL injection):
@GetMapping("/search")
public List<Product> search(@RequestParam @Size(max = 100) String query) {
    return jdbcTemplate.query(
        "SELECT * FROM products WHERE name = ?",
        new Object[]{query}, // Query parameter, never concatenated
        rowMapper
    );
}

// ✅ Bean Validation on all DTOs:
record CreateOrderRequest(
    @NotNull @Positive Long productId,
    @NotNull @Min(1) @Max(1000) Integer quantity,
    @NotNull @Email String customerEmail,
    @Pattern(regexp = "^[A-Z]{2}\\d{5}$") String promoCode  // Regex whitelist
) {}

@PostMapping("/orders")
public Order createOrder(@Valid @RequestBody CreateOrderRequest req) {
    // @Valid triggers Bean Validation — BadRequest if any constraint violated
    ...
}
```

---

## 🚦 Rate Limiting & Abuse Prevention

Rate limiting is covered in depth in [API Rate Limiting](./API_Rate_Limiting.md). For quick reference:

```java
// Spring Boot with Bucket4j (token bucket rate limiter):
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String key = extractKey(request); // IP or API key
        Bucket bucket = buckets.computeIfAbsent(key, k ->
            Bucket.builder()
                .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
                .build()
        );
        
        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.setHeader("X-RateLimit-Limit", "100");
            response.setHeader("X-RateLimit-Remaining", "0");
            response.getWriter().write("{\"error\":\"Too Many Requests\"}");
        }
    }
}
```

---

## 🗝️ API Keys

```
API KEY SECURITY RULES:
─────────────────────────────────────────────────────────────────────
✅ Store only the HASH of the key (bcrypt or SHA-256), never plain text
✅ Scope API keys to minimum permissions (read-only vs read-write)
✅ Support key rotation without downtime (accept old + new for 24h grace)
✅ Log every API key usage with timestamp, endpoint, IP
✅ Send keys over TLS only, never in URLs (use Authorization header)
✅ Allow customers to revoke keys instantly
✅ Set expiry dates on keys (90-day rotation is common)

❌ Never embed keys in mobile app source code
❌ Never log keys in application logs
❌ Never send keys as query parameters (?api_key=... is in access logs!)
```

```java
// ✅ Correct API key transmission:
// Authorization: ApiKey sk_live_abc123xyz
// NOT: GET /orders?api_key=sk_live_abc123xyz  (ends up in server logs!)

@Bean
public ApiKeyAuthFilter apiKeyFilter() {
    return new ApiKeyAuthFilter("Authorization", key -> {
        String hash = hashKey(key.replace("ApiKey ", ""));
        return apiKeyRepo.findByKeyHash(hash)  // Compare hashes only
            .map(apiKey -> {
                if (apiKey.isExpired() || apiKey.isRevoked()) return null;
                return buildAuthentication(apiKey);
            })
            .orElse(null);
    });
}
```

---

## 🪖 Security Headers

Every API response should include these headers:

```java
// Spring Security — add security headers globally:
@Bean
public SecurityFilterChain security(HttpSecurity http) throws Exception {
    http.headers(headers -> headers
        .contentTypeOptions(Customizer.withDefaults())    // X-Content-Type-Options: nosniff
        .frameOptions(FrameOptionsConfig::deny)           // X-Frame-Options: DENY
        .httpStrictTransportSecurity(hsts -> hsts         // HSTS: max-age=31536000
            .maxAgeInSeconds(31536000)
            .includeSubDomains(true)
        )
        .contentSecurityPolicy(csp -> csp
            .policyDirectives("default-src 'self'")
        )
    )
    .headers(h -> h.addHeaderWriter(
        new StaticHeadersWriter("X-API-Version", "v2") // Custom version header
    ));
    return http.build();
}
```

```
RESPONSE HEADER          VALUE                        PURPOSE
────────────────────────────────────────────────────────────────────────────────
X-Content-Type-Options   nosniff                     Prevent MIME sniffing attacks
X-Frame-Options          DENY                        Prevent clickjacking
Strict-Transport-Security max-age=31536000           Force HTTPS for 1 year
Content-Security-Policy  default-src 'self'          Restrict resource sources
Cache-Control            no-store                    Prevent caching sensitive data
X-Request-ID             [UUID]                      Trace requests across services
```

---

## 💻 Spring Boot Security Implementation

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity // Enables @PreAuthorize, @PostAuthorize
public class ApiSecurityConfig {

    @Bean
    public SecurityFilterChain apiChain(HttpSecurity http) throws Exception {
        http
            // Stateless JWT APIs don't need CSRF (cookie-less)
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            
            // CORS configuration
            .cors(cors -> cors.configurationSource(corsConfigSource()))
            
            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()     // Public auth endpoints
                .requestMatchers("/actuator/health").permitAll()    // Health check
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN") // Admin only
                .anyRequest().authenticated()                       // All others: must auth
            )
            
            // JWT filter before standard auth
            .addFilterBefore(jwtAuthFilter(), UsernamePasswordAuthenticationFilter.class)
            
            // Custom error responses (not HTML redirect for APIs)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(UNAUTHORIZED))
                .accessDeniedHandler((req, res, e) -> {
                    res.setStatus(403);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"error\":\"Access Denied\"}");
                })
            );
        
        return http.build();
    }
}
```

---

## 🌍 Real-World Analogy

Think of API security like **airport security**:

| Concept | Airport | API Security |
|---------|---------|--------------|
| Authentication | Passport check | JWT / API Key verification |
| Authorization | Boarding pass | Permission scopes |
| Rate Limiting | Boarding gate capacity | Request throttling |
| Input Validation | Baggage X-ray scan | Request body validation |
| TLS/HTTPS | Sealed cargo hold | Encrypted transport |
| BOLA | Sitting in someone else's seat | Accessing another user's data |
| Security Headers | "No Photography" signs | Response header policies |

---

## ⚠️ Common Pitfalls

### 💣 Pitfall 1: Returning Too Much Data (Over-Fetching)

```java
// ❌ Returns ALL user fields including hashed passwords, tokens:
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepo.findById(id); // Exposes: passwordHash, sessionTokens, ...
}

// ✅ Return only what the caller needs:
record UserResponse(Long id, String name, String email, String avatarUrl) {}

@GetMapping("/users/{id}")
public UserResponse getUser(@PathVariable Long id) {
    User user = userRepo.findById(id);
    return new UserResponse(user.getId(), user.getName(), 
                            user.getEmail(), user.getAvatarUrl());
}
```

### 💣 Pitfall 2: Verbose Error Messages

```java
// ❌ Reveals system internals:
// {"error": "Column 'user_id' doesn't exist in table 'orders_v2'"}
// {"error": "Connection to 10.0.0.4:5432 refused"}

// ✅ Safe error response — helpful to the caller, opaque about internals:
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleError(Exception ex) {
    log.error("Unhandled error [requestId={}]", requestId, ex); // Log full detail
    return ResponseEntity.status(500).body(
        new ErrorResponse("INTERNAL_ERROR", "Something went wrong", requestId)
        // requestId lets engineers correlate to logs without leaking details
    );
}
```

### 💣 Pitfall 3: Trusting Client-Supplied IDs for Authorization

```java
// ❌ Using client-supplied userId from JWT claims WITHOUT verifying ownership:
@PostMapping("/transfer")
public Transfer createTransfer(@RequestBody TransferRequest req) {
    // req.sourceAccountId comes from request body — never trust this!
    Account source = accountRepo.findById(req.sourceAccountId);
    // Attacker sets sourceAccountId to someone else's account → steals money!
}

// ✅ Always derive resource ownership from the authenticated identity:
@PostMapping("/transfer")
public Transfer createTransfer(@RequestBody TransferRequest req,
                               @AuthenticationPrincipal UserDetails user) {
    // Get accounts that BELONG to the authenticated user — not from request:
    Account source = accountRepo.findByIdAndOwnerId(
        req.sourceAccountId, user.getUserId() // Ownership enforced in query
    );
    if (source == null) throw new AccessDeniedException("Account not found");
    ...
}
```

---

## 💡 Interview Q&A

**Q1: What is the difference between authentication and authorization in API context?**
```
Authentication: "Who are you?" — verify identity (JWT signature valid, API key exists)
Authorization:  "What can you do?" — check permissions (can this user access THIS resource?)

Key insight: Authentication happens ONCE per request. Authorization happens per resource.
A token can be valid (authenticated) but still denied (not authorized for that specific object).
BOLA is an authorization failure — the token is valid but it shouldn't access THAT record.
```

**Q2: Why is BOLA (Broken Object Level Authorization) the #1 API vulnerability?**
```
Because developers naturally write: findById(id) — which fetches ANY record with that ID.
The fix requires an extra line: findByIdAndOwner(id, currentUser.getId()).
This is easy to forget, especially in large teams, and extremely easy to exploit:
just change /orders/1001 to /orders/1002 in the URL bar.

Prevention: Never use a "find by ID" query alone — always add "AND owner = ?" or
check ownership after fetch. Code review automation (SpotBugs rules) can catch this.
```

**Q3: Should you use HS256 or RS256 for JWT signing? Why?**
```
RS256 (RSA asymmetric) is preferred for distributed systems:
  - Private key signs tokens (only the auth server needs it — never shared)
  - Public key verifies tokens (safe to distribute to all microservices)
  - Compromise of a microservice doesn't let it FORGE tokens

HS256 (HMAC symmetric) is only appropriate for:
  - Single-service architectures (same service signs and verifies)
  - Performance-critical paths where RSA overhead matters

Interviewer follow-up: What about key rotation?
  RS256: Publish new public key, old tokens still verify with old public key during rotation
  HS256: Rotating the secret invalidates all existing tokens immediately
```

**Q4: How do you prevent replay attacks on an API?**
```
Multiple layers:
1. Short JWT expiry (15 min): stolen token is useless after expiry
2. Idempotency keys for mutations: server stores processed keys (24h)
   POST /payments with X-Idempotency-Key: uuid → duplicate rejected with 409
3. Nonce in sensitive operations: one-time-use value in payment/transfer requests
4. TLS: prevents capture of tokens in transit (makes replay harder)
5. Token binding (emerging): binds token to the TLS connection
```

---

## 🎲 Mini Challenge

> 🎲 **CHALLENGE** (5 minutes):
> You're reviewing this API endpoint. Find ALL security vulnerabilities:
>
> ```java
> @GetMapping("/admin/users")
> public List<User> getAllUsers(@RequestParam String role) {
>     String query = "SELECT * FROM users WHERE role = '" + role + "'";
>     return jdbcTemplate.query(query, userRowMapper);
> }
> ```

<details>
<summary>💡 Click to reveal answer (4 vulnerabilities)</summary>

**Vulnerability 1 — Missing Authentication**: No `@PreAuthorize` or auth check. Anyone can call `/admin/users`.

**Vulnerability 2 — Missing Authorization**: Even if authenticated, the endpoint doesn't check that the caller is actually an admin.

**Vulnerability 3 — SQL Injection**: `role` is directly concatenated into the query. An attacker can send `role = ' OR '1'='1` to dump the entire table.

**Vulnerability 4 — Excessive Data Exposure**: Returns `List<User>` which likely contains password hashes, tokens, and other sensitive fields.

**Fixed version:**
```java
@GetMapping("/admin/users")
@PreAuthorize("hasRole('ADMIN')")
public List<UserSummary> getAllUsers(@RequestParam @Pattern(regexp = "^[A-Z_]+$") String role) {
    return userRepo.findByRole(role).stream()  // Parameterized query via JPA
        .map(UserSummary::from)                // DTO — no sensitive fields
        .toList();
}
```

</details>

---

## 🏢 Industry Applications

| Company | API Security Practice | Impact |
|---------|----------------------|--------|
| Stripe | All API keys are scoped (test vs live, read vs write) | Limits blast radius of key theft |
| GitHub | Personal access tokens with fine-grained repo permissions | BOLA prevention at token level |
| Twilio | All requests signed with HMAC (webhook verification) | Prevents forged webhook calls |
| Netflix | mTLS between all microservices, JWT for user-facing APIs | Defense in depth |
| Google | OAuth scopes align to individual API method permissions | Principle of least privilege |

---

## 🔗 What to Read Next

| Article | Why You Need It |
|---------|----------------|
| [API Rate Limiting](./API_Rate_Limiting.md) | Abuse prevention — the complement to auth |
| [OAuth2 Deep Dive](../Security/OAuth2.md) | The full delegated authorization framework |
| [JWT Deep Dive](../Security/JWT_Deep_Dive.md) | Token internals, signing algorithms, rotation |
| [OWASP Top 10](../Security/OWASP_Top10.md) | Broader web security vulnerabilities |

---

*Previous: [← WebRTC](./WebRTC.md) | Next: [API Rate Limiting →](./API_Rate_Limiting.md)*
