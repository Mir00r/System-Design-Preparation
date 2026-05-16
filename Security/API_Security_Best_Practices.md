# 🛡️ API Security Best Practices: Protecting Your APIs from the Top 10 Attack Vectors

> *"APIs are the front door to your data. They handle 83% of internet traffic, yet most breaches exploit basic API vulnerabilities — broken authentication, excessive data exposure, and missing rate limiting. A single misconfigured endpoint can expose millions of user records."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [OAuth2](./OAuth2.md), [JWT Deep Dive](./JWT_Deep_Dive.md), [OWASP Top 10](./OWASP_Top10.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [OWASP API Security Top 10](#-owasp-api-security-top-10)
3. [Authentication & Authorization](#-authentication--authorization)
4. [Input Validation](#-input-validation)
5. [Rate Limiting & Throttling](#-rate-limiting--throttling)
6. [Data Exposure Protection](#-data-exposure-protection)
7. [Transport Security](#-transport-security)
8. [Spring Boot Implementation](#-spring-boot-implementation)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
Common API attack surface:

  [Attacker] ──▶ GET /api/users/123        (IDOR: access other users' data)
  [Attacker] ──▶ POST /api/admin/users      (broken function-level auth)
  [Attacker] ──▶ GET /api/users?page=1...∞  (scrape entire database)
  [Attacker] ──▶ POST /api/login × 1M       (credential stuffing, no rate limit)
  [Attacker] ──▶ POST /api/search?q='; DROP TABLE users;--  (injection)

  Real breaches:
    - Optus (2022): sequential user IDs in API → 10M records exposed
    - Peloton (2021): unauthenticated API → all user profiles accessible
    - Facebook (2019): phone lookup API → 533M records scraped
    - T-Mobile (2023): API abuse → 37M customer records
```

---

## 📊 OWASP API Security Top 10 (2023)

| # | Vulnerability | Description | Mitigation |
|---|---|---|---|
| API1 | **Broken Object Level Auth** | Access other users' objects by changing IDs | Verify ownership on every request |
| API2 | **Broken Authentication** | Weak auth mechanisms, credential stuffing | MFA, token rotation, account lockout |
| API3 | **Broken Object Property Level Auth** | Mass assignment, excessive data exposure | Allowlist response fields, block mass assignment |
| API4 | **Unrestricted Resource Consumption** | No rate limiting, unbounded queries | Rate limit, pagination limits, query cost analysis |
| API5 | **Broken Function Level Auth** | Access admin functions as regular user | RBAC, deny by default |
| API6 | **Unrestricted Access to Sensitive Business Flows** | Automated abuse (scraping, ticket scalping) | CAPTCHA, device fingerprinting, behavioral analysis |
| API7 | **Server-Side Request Forgery** | Trick server into making internal requests | Allowlist URLs, block internal IPs |
| API8 | **Security Misconfiguration** | Debug enabled, permissive CORS, missing headers | Security hardening checklists |
| API9 | **Improper Inventory Management** | Forgotten/undocumented API endpoints | API gateway, version management, discovery |
| API10 | **Unsafe Consumption of APIs** | Trusting third-party API responses | Validate/sanitize all external data |

---

## 🔑 Authentication & Authorization

```
AUTHENTICATION LAYERS (verify identity):

  Layer 1: API Key (simple, machine-to-machine)
    Header: X-API-Key: sk_live_abc123
    Good for: server-to-server, internal services
    ❌ Don't use for: user-facing apps (keys leak in client code)

  Layer 2: OAuth 2.0 + JWT (user authorization)
    Header: Authorization: Bearer eyJhbG...
    Token contains: user_id, scopes, expiry
    Good for: user-facing APIs, delegated access

  Layer 3: mTLS (service identity)
    Certificate-based mutual authentication
    Good for: service-to-service in zero-trust environments

AUTHORIZATION PATTERNS:

  RBAC (Role-Based Access Control):
    User → Role → Permissions
    user_123 → "editor" → [read, write, publish]
    user_456 → "viewer" → [read]

  ABAC (Attribute-Based Access Control):
    Policy: "allow if user.department == resource.department AND time.hour in 9-17"
    More flexible than RBAC, used for fine-grained policies

  Object-Level Auth (CRITICAL — #1 vulnerability):
    ❌ WRONG: GET /api/orders/456 → return order 456 (no ownership check!)
    ✅ RIGHT: GET /api/orders/456 → verify order.user_id == authenticated_user.id
```

---

## 🧹 Input Validation

```
VALIDATE EVERYTHING at the API boundary:

  1. TYPE VALIDATION
     ❌ userId: "abc" (expected integer)
     ❌ email: "not-an-email" (expected email format)
     ❌ amount: -100 (expected positive number)

  2. LENGTH/SIZE LIMITS
     ❌ username: "a" × 10000 (buffer overflow attempt)
     ❌ file_upload: 50GB (resource exhaustion)
     ❌ array_field: [1,2,3,...,1000000] (memory bomb)

  3. INJECTION PREVENTION
     ❌ search: "'; DROP TABLE users;--" (SQL injection)
     ❌ name: "<script>alert('xss')</script>" (XSS)
     ❌ url: "file:///etc/passwd" (SSRF)

  4. BUSINESS LOGIC VALIDATION
     ❌ transfer_amount > account_balance (overdraft)
     ❌ order_quantity: 999999 (inventory check)
     ❌ date_of_birth: "2025-01-01" (future date)

  SANITIZATION STRATEGY:
     Input → Validate (reject if invalid) → Sanitize (encode special chars) → Process
     NEVER trust user input, even from authenticated users
     NEVER trust input from other internal services (defense in depth)
```

---

## 🚦 Rate Limiting & Throttling

```
RATE LIMITING STRATEGIES:

  Fixed Window:
    100 requests per minute per user
    Simple but has burst problem at window boundaries

  Sliding Window:
    100 requests in any rolling 60-second period
    Smoother, no boundary burst

  Token Bucket:
    Bucket holds 100 tokens, refills at 2/second
    Allows controlled bursts (up to 100) then throttles

  Adaptive:
    Normal: 100 req/min
    Suspicious behavior detected: drop to 10 req/min
    Account flagged: block entirely

RATE LIMIT BY:
  - API key / user ID (per-user limits)
  - IP address (catches unauthenticated abuse)
  - Endpoint (expensive endpoints get lower limits)
  - Request cost (complex queries cost more "tokens")

RESPONSE WHEN LIMITED:
  HTTP 429 Too Many Requests
  Headers:
    X-RateLimit-Limit: 100
    X-RateLimit-Remaining: 0
    X-RateLimit-Reset: 1640000000 (Unix timestamp)
    Retry-After: 30 (seconds)
```

---

## 🔒 Data Exposure Protection

```
PRINCIPLE: Return ONLY what the client needs

  ❌ OVER-EXPOSURE (real breach pattern):
    GET /api/users/123
    Response: {
      "id": 123,
      "name": "John",
      "email": "john@example.com",
      "password_hash": "$2b$10$...",     ← NEVER expose!
      "ssn": "123-45-6789",             ← PII leak!
      "internal_notes": "VIP customer",  ← internal data leak!
      "credit_card": "4111..."           ← PCI violation!
    }

  ✅ MINIMAL RESPONSE (allowlist approach):
    GET /api/users/123
    Response: {
      "id": 123,
      "name": "John",
      "email": "john@example.com"
    }

  STRATEGIES:
    1. Response DTOs: never return entity objects directly
    2. Field-level permissions: admin sees more fields than regular user
    3. Pagination: ALWAYS limit results (max 100 per page)
    4. Query depth limiting (GraphQL): prevent nested query attacks
```

---

## 💻 Spring Boot Implementation

```java
// 1. Input validation with Bean Validation
@RestController
@Validated
public class UserController {

    @PostMapping("/api/users")
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        // Bean Validation runs BEFORE this method executes
        return ResponseEntity.ok(userService.create(request));
    }
}

public record CreateUserRequest(
    @NotBlank @Size(min = 2, max = 50) 
    String name,
    
    @NotBlank @Email 
    String email,
    
    @NotBlank @Size(min = 8, max = 128)
    @Pattern(regexp = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d).*$",
             message = "Must contain uppercase, lowercase, and digit")
    String password,
    
    @Min(18) @Max(150) 
    Integer age
) {}

// 2. Object-level authorization (prevent IDOR)
@GetMapping("/api/orders/{orderId}")
@PreAuthorize("@authz.isOrderOwner(#orderId, authentication)")
public OrderResponse getOrder(@PathVariable Long orderId) {
    return orderService.getById(orderId);
}

@Component("authz")
public class AuthorizationService {
    public boolean isOrderOwner(Long orderId, Authentication auth) {
        Long userId = ((UserPrincipal) auth.getPrincipal()).getId();
        return orderRepository.existsByIdAndUserId(orderId, userId);
    }
}

// 3. Response DTO (never expose entity directly)
public record UserResponse(
    Long id,
    String name,
    String email,
    String role
    // password_hash, ssn, internal fields are NOT here
) {
    public static UserResponse from(User entity) {
        return new UserResponse(entity.getId(), entity.getName(), 
                                entity.getEmail(), entity.getRole());
    }
}

// 4. Rate limiting with Bucket4j
@Configuration
public class RateLimitConfig {
    
    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        // 100 requests per minute per user
        var filter = new RateLimitFilter(
            Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1)))
        );
        var registration = new FilterRegistrationBean<>(filter);
        registration.addUrlPatterns("/api/*");
        return registration;
    }
}

// 5. Security headers
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .headers(headers -> headers
                .contentTypeOptions(Customizer.withDefaults())
                .frameOptions(frame -> frame.deny())
                .httpStrictTransportSecurity(hsts -> hsts
                    .maxAgeInSeconds(31536000)
                    .includeSubDomains(true))
                .contentSecurityPolicy(csp -> 
                    csp.policyDirectives("default-src 'self'"))
            )
            .csrf(csrf -> csrf.csrfTokenRepository(
                CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .cors(cors -> cors.configurationSource(corsConfig()))
            .build();
    }
    
    private CorsConfigurationSource corsConfig() {
        var config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://myapp.com")); // explicit allowlist
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        config.setMaxAge(3600L);
        var source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

---

## 📋 API Security Checklist

```
✅ AUTHENTICATION
  □ Use OAuth 2.0 with PKCE for public clients (mobile/SPA)
  □ Short-lived access tokens (15 min) + refresh tokens (7 days)
  □ Revoke refresh tokens on password change / suspicious activity
  □ Implement account lockout after 5 failed attempts

✅ AUTHORIZATION
  □ Verify object ownership on EVERY request (not just authentication)
  □ Implement RBAC/ABAC with deny-by-default
  □ Separate admin APIs onto different endpoints/servers
  □ Log all authorization failures for security monitoring

✅ INPUT
  □ Validate all input (type, length, format, range)
  □ Use parameterized queries (never string concatenation for SQL)
  □ Sanitize output (prevent XSS in responses)
  □ Reject unexpected fields (strict schema validation)

✅ TRANSPORT
  □ TLS 1.2+ everywhere (no HTTP, no TLS 1.0/1.1)
  □ HSTS header with includeSubDomains
  □ Certificate pinning for mobile apps

✅ RATE LIMITING
  □ Per-user and per-IP rate limits
  □ Lower limits for expensive endpoints (search, export)
  □ Implement CAPTCHA for public-facing forms
  □ Return 429 with Retry-After header

✅ MONITORING
  □ Log all API requests (who, what, when, from where)
  □ Alert on: auth failures spike, unusual traffic patterns, data export volume
  □ Implement request tracing (correlation ID across services)
```

---

## ⚠️ Common Pitfalls

1. **Relying on client-side validation only** — Client validation is for UX, not security. Attackers bypass your UI entirely and call APIs directly with curl/Postman. ALL validation must be duplicated server-side.

2. **Sequential/predictable resource IDs** — Using auto-increment IDs (1, 2, 3...) makes IDOR trivial: just increment the ID to access the next user's data. Use UUIDs or at minimum, always verify ownership server-side.

3. **Returning full entity objects** — Serializing database entities directly to JSON exposes internal fields (password hashes, internal flags, related entities). Always use dedicated response DTOs with explicitly allowlisted fields.

4. **Permissive CORS** — `Access-Control-Allow-Origin: *` means any website can make authenticated requests to your API on behalf of logged-in users. Explicitly allowlist your application's origins only.

5. **Logging sensitive data** — Access tokens, passwords, credit cards, and PII in logs are a breach waiting to happen. Implement structured logging with automatic PII redaction for sensitive fields.

---

## 🧩 Mini Challenge

**Your API has this endpoint: `GET /api/documents/{documentId}/download`. A penetration tester found they can download ANY user's documents by changing the ID. Fix the vulnerability.**

<details>
<summary>💡 Click to reveal answer</summary>

**Root cause**: Broken Object-Level Authorization (OWASP API1) — the endpoint checks authentication (is the user logged in?) but NOT authorization (does this user own this document?).

**Fix**:

```java
@GetMapping("/api/documents/{documentId}/download")
public ResponseEntity<Resource> downloadDocument(
        @PathVariable UUID documentId,
        @AuthenticationPrincipal UserPrincipal user) {
    
    // FIX: Verify the authenticated user owns this document
    Document doc = documentRepository.findById(documentId)
        .orElseThrow(() -> new ResponseStatusException(NOT_FOUND));
    
    if (!doc.getOwnerId().equals(user.getId()) 
            && !doc.getSharedWith().contains(user.getId())) {
        // Return 404 (not 403) to prevent information disclosure
        // 403 reveals "document exists but you can't access it"
        throw new ResponseStatusException(NOT_FOUND);
    }
    
    Resource file = storageService.load(doc.getStoragePath());
    return ResponseEntity.ok()
        .contentType(MediaType.APPLICATION_OCTET_STREAM)
        .body(file);
}
```

**Additional safeguards**:
- Use UUIDs instead of sequential IDs (harder to enumerate)
- Add audit logging: log every download with user_id and document_id
- Rate limit downloads: max 100 downloads/hour per user (catches scraping)
- Return 404 (not 403) for unauthorized access (prevents enumeration)

</details>

---

## 📝 Interview Q&A

**Q: What's the difference between authentication and authorization in API security?**
> A: Authentication answers "who are you?" — verifying the caller's identity (valid JWT, API key, or certificate). Authorization answers "what can you do?" — determining if the authenticated identity has permission to perform the specific action on the specific resource. A common mistake is implementing only authentication (verify the token is valid) without authorization (verify this user owns this resource). This leads to IDOR vulnerabilities — the #1 API security issue per OWASP.

**Q: How would you protect a public API from abuse (scraping, DDoS)?**
> A: Defense in depth with multiple layers: (1) **Rate limiting** — per-IP and per-API-key limits at the API gateway (e.g., 100 req/min). (2) **Authentication** — require API keys even for "public" endpoints (allows per-key throttling and revocation). (3) **Cost-based limits** — assign costs to expensive endpoints; search = 5 credits, list = 1 credit. (4) **Behavioral analysis** — detect patterns (sequential scraping, unusual hours, high page counts) and CAPTCHA or block. (5) **Response limiting** — paginate all list endpoints (max 100 items), add cursor-based pagination to prevent full-database dumps. (6) **WAF/CDN** — Cloudflare/AWS WAF for DDoS protection at the edge.

---

## 🔗 What to Read Next

1. **[Security/OAuth2.md](./OAuth2.md)** — OAuth 2.0 flows for secure API authentication
2. **[BuildingBlocks/RateLimiting.md](../BuildingBlocks/RateLimiting.md)** — Deep dive into rate limiting algorithms
3. **[Security/SecurityInMicroservices.md](./SecurityInMicroservices.md)** — Security patterns across distributed services

---

*[← mTLS](./mTLS.md) | [Back to Security](../INDEX.md) | [Next: Security in Microservices →](./SecurityInMicroservices.md)*
