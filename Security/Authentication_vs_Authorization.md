# 🛡️ Authentication vs Authorization
## OAuth2, OIDC, RBAC, ABAC — The Complete Guide

> *"Authentication is knowing who you are. Authorization is knowing what you're allowed to do. Confusing the two is one of the most common sources of security vulnerabilities in production systems."*

**⏱️ Estimated Time**: 50 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [JWT Deep Dive](./JWT_Deep_Dive.md)

---

## 📋 Table of Contents
1. [AuthN vs AuthZ — The Core Distinction](#-authn-vs-authz--the-core-distinction)
2. [OAuth2 — Delegated Authorization](#-oauth2--delegated-authorization)
3. [OAuth2 Grant Types](#-oauth2-grant-types)
4. [OpenID Connect (OIDC)](#-openid-connect-oidc)
5. [Authorization Models: RBAC vs ABAC](#-authorization-models-rbac-vs-abac)
6. [Spring Security Implementation](#-spring-security-implementation)
7. [Common Pitfalls](#-common-pitfalls)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## 🤔 AuthN vs AuthZ — The Core Distinction

```
AUTHENTICATION (AuthN) — "Who are you?"
  Answers: "I am Alice, user ID 12345, and I can prove it."
  Mechanism: Password check, biometrics, OTP, hardware key, JWT validation
  Result: An identity (claims: sub, email, name)

AUTHORIZATION (AuthZ) — "What can you do?"
  Answers: "Alice is allowed to read orders but not delete users."
  Mechanism: Role checks, permission checks, policy evaluation
  Result: Allowed or Denied

Timeline of a request:
  1. Client sends request with credentials (Bearer token)
  2. AUTH FILTER: Validate token → confirm identity (AuthN)
  3. SECURITY FILTER: Check permissions → confirm access (AuthZ)
  4. Handler executes (or 401 Unauthorized / 403 Forbidden returned)

HTTP Status Codes:
  401 Unauthorized → AuthN failed (not logged in, bad token)
  403 Forbidden    → AuthZ failed (logged in, but no permission)
```

---

## 🔑 OAuth2 — Delegated Authorization

OAuth2 solves a specific problem: **"How do I let App B access my data on Service A without giving App B my password?"**

### Real-World Example

```
"Log in with Google" on a third-party app:

Without OAuth2:
  → You give the app your Google password
  → App can now read your email, send emails, delete everything
  → If app is breached → your Google account is compromised forever

With OAuth2:
  → You authorize Google to give the app an ACCESS TOKEN
  → Access token has limited scope: "read your name and email only"
  → App never sees your password
  → You can revoke the token at any time from Google's security settings
  → Access token expires (short-lived)
```

### OAuth2 Roles

```
Resource Owner:   The USER who owns the data (you)
Client:           The app requesting access (third-party app)
Authorization Server: Issues tokens after user consent (Google, Okta, Keycloak)
Resource Server:  The API that has the data (Google's API, your backend)
```

### OAuth2 Token Types

```
Access Token:
  - Short-lived (15 min - 1 hour)
  - Sent with every API request: Authorization: Bearer {access_token}
  - Opaque string OR JWT (depends on AS)

Refresh Token:
  - Long-lived (days to months)
  - Used to get a new access token without re-login
  - Stored securely server-side (HttpOnly cookie)
  - Rotated on each use (security best practice)

Authorization Code:
  - One-time use, very short-lived (60 seconds)
  - Exchanged for access + refresh tokens
  - Never exposed to browser (PKCE prevents interception)
```

---

## 🔄 OAuth2 Grant Types

### 1. Authorization Code + PKCE (Recommended for all browser/mobile apps)

```
PKCE = Proof Key for Code Exchange (prevents authorization code interception)

Flow:
  1. Client generates code_verifier (random string) + code_challenge = SHA256(code_verifier)
  2. Client → Browser: GET /authorize?client_id=...&code_challenge=...&response_type=code
  3. User → Auth Server: Login + Consent
  4. Auth Server → Browser: Redirect to callback?code=AUTHCODE123 (one-time code)
  5. Client → Auth Server: POST /token { code: AUTHCODE123, code_verifier: ... }
     (code_verifier proves this is the same client that started the flow)
  6. Auth Server → Client: { access_token: ..., refresh_token: ..., expires_in: 3600 }
  7. Client → Resource Server: GET /api/data  Authorization: Bearer {access_token}

Why PKCE?
  Without PKCE: attacker intercepts the authorization code in step 4 (URL in logs/history)
  With PKCE: intercepted code is useless — attacker doesn't have the code_verifier

Use for: Single-page apps (SPA), mobile apps, web apps with user login
```

### 2. Client Credentials (Machine-to-Machine)

```
No user involved — service authenticates as itself

Flow:
  1. Service A → Auth Server: POST /token
     { grant_type: client_credentials, client_id: "svc-a", client_secret: "secret" }
  2. Auth Server → Service A: { access_token: ..., expires_in: 3600 }
  3. Service A → Service B: GET /api/internal  Authorization: Bearer {access_token}
  4. Service B validates token → processes request

Use for: Microservice-to-microservice auth, batch jobs, CI/CD pipelines
Never use: when a real user is involved
```

### 3. Device Authorization Grant (TV / IoT)

```
For devices without keyboards or limited input (Smart TVs, CLI tools)

Flow:
  1. Device → Auth Server: POST /device/authorize { client_id: ... }
  2. Auth Server → Device: { device_code, user_code: "ABCD-1234", verification_uri: "https://..." }
  3. Device displays: "Go to https://example.com/device and enter ABCD-1234"
  4. User → Browser → Auth Server: enters ABCD-1234, logs in, grants consent
  5. Device: polls POST /token { device_code: ..., grant_type: urn:...device_code } every 5s
  6. Auth Server → Device: { access_token: ... } (once user completes step 4)

Use for: Smart TV apps (Netflix, YouTube), CLI tools (GitHub CLI, AWS CLI)
```

---

## 🪪 OpenID Connect (OIDC)

OAuth2 is about **authorization** — it gives you a token to access resources.
OIDC is about **authentication** — it adds a layer on top of OAuth2 to verify identity.

```
OAuth2 alone:   "Here's an access token. Use it to get data."
OIDC adds:      "Here's an ID token. It tells you WHO the user is."

ID Token (JWT):
{
  "sub":   "1234567890",        // unique user identifier
  "name":  "Alice Smith",
  "email": "alice@example.com",
  "iss":   "https://accounts.google.com",
  "aud":   "your_app_client_id",
  "iat":   1716000000,
  "exp":   1716003600,
  "nonce": "abc123"             // prevents replay attacks
}
```

### OIDC vs OAuth2

| | OAuth2 | OIDC |
|---|---|---|
| Purpose | Authorization (access) | Authentication (identity) |
| Token returned | Access Token (opaque or JWT) | ID Token (always JWT) + Access Token |
| "Who is the user?" | No standard way | `sub` claim in ID Token |
| Standard user info | No | `/userinfo` endpoint |
| SSO support | Partial | Built-in |
| Use case | Third-party API access | Login ("Sign in with Google") |

---

## 🎭 Authorization Models: RBAC vs ABAC

### RBAC — Role-Based Access Control

```
Users are assigned ROLES. Roles have PERMISSIONS.

Roles:   ADMIN, MANAGER, USER, GUEST
Permissions:
  ADMIN:   read, write, delete, manage_users
  MANAGER: read, write
  USER:    read, write_own
  GUEST:   read

Decision: Does Alice's role set contain the required permission?

Pros:  Simple to understand, easy to audit, scales well
Cons:  Coarse-grained — you can't say "Alice can read orders but only from Region A"
       Role explosion: as permissions get complex, you create many roles
```

```java
// Spring Security RBAC
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/users/{userId}")
public void deleteUser(@PathVariable Long userId) { ... }

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
@PostMapping("/orders")
public Order createOrder(@RequestBody OrderRequest req) { ... }

// Method-level security on service
@PreAuthorize("hasRole('USER') and #userId == authentication.principal.id")
public UserProfile getProfile(Long userId) { ... }
```

### ABAC — Attribute-Based Access Control

```
Decisions based on ATTRIBUTES of: the user, the resource, and the environment.

Policy example:
  "Allow access if:
   user.department == resource.department
   AND user.clearanceLevel >= resource.sensitivityLevel
   AND time.hour BETWEEN 9 AND 17
   AND request.ipAddress IN allowed_networks"

Pros:  Fine-grained, flexible, no role explosion
Cons:  Complex to implement and audit; policy management is hard
Used by: AWS IAM policies, financial systems, healthcare (HIPAA compliance)
```

```java
// Spring Security ABAC using SpEL expressions
@PreAuthorize("@permissionEvaluator.canAccess(authentication, #orderId, 'ORDER', 'READ')")
@GetMapping("/orders/{orderId}")
public Order getOrder(@PathVariable Long orderId) { ... }

// Custom permission evaluator
@Component("permissionEvaluator")
public class OrderPermissionEvaluator {

    @Autowired private OrderRepository orderRepository;

    public boolean canAccess(Authentication auth, Long orderId, String type, String action) {
        UserPrincipal user = (UserPrincipal) auth.getPrincipal();

        if ("ORDER".equals(type) && "READ".equals(action)) {
            Order order = orderRepository.findById(orderId).orElse(null);
            if (order == null) return false;

            // Admins can see all orders
            if (user.hasRole("ADMIN")) return true;

            // Managers can see orders from their region
            if (user.hasRole("MANAGER"))
                return order.getRegion().equals(user.getRegion());

            // Users can only see their own orders
            return order.getUserId().equals(user.getId());
        }
        return false;
    }
}
```

### RBAC vs ABAC Quick Comparison

| Aspect | RBAC | ABAC |
|---|---|---|
| Complexity | Low | High |
| Flexibility | Medium | Very High |
| Performance | Fast (role lookup) | Slower (policy evaluation) |
| Auditability | Easy | Complex |
| Best for | Most applications | Enterprise, compliance, multi-tenancy |
| Example | "Admin can delete users" | "User can read document if they're in the same department and clearance level ≥ 3" |

---

## 🔧 Spring Security Implementation

### Full OAuth2 Resource Server Configuration

```java
@Configuration
@EnableMethodSecurity  // enables @PreAuthorize, @PostAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())           // disabled for stateless APIs
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // no server sessions
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()     // public endpoints
                .requestMatchers("/v1/auth/**").permitAll()          // login/register
                .requestMatchers("/v1/admin/**").hasRole("ADMIN")    // role check
                .anyRequest().authenticated()                        // everything else needs auth
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter =
                new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");   // custom claim name
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");        // Spring's prefix convention

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return converter;
    }
}
```

### Keycloak Integration (Popular Auth Server)

```yaml
# application.yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://keycloak.mycompany.com/realms/myapp
          # Spring auto-fetches JWKS from: {issuer-uri}/.well-known/openid-configuration
```

```java
// JWT contains: { "realm_access": { "roles": ["admin", "user"] } }
// Custom converter to extract Keycloak-specific role structure

@Component
public class KeycloakRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {
    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        Map<String, Object> realmAccess =
                jwt.getClaimAsMap("realm_access");
        if (realmAccess == null) return Collections.emptyList();

        @SuppressWarnings("unchecked")
        List<String> roles = (List<String>) realmAccess.get("roles");
        if (roles == null) return Collections.emptyList();

        return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                .collect(Collectors.toList());
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Using Implicit Flow in SPAs** — The OAuth2 Implicit flow (response_type=token) returns the access token directly in the URL fragment, exposing it to browser history and referrer headers. Always use Authorization Code + PKCE for SPAs (PKCE prevents interception without needing a client secret).

2. **Hardcoding client secrets in frontend code** — Public clients (browsers, mobile apps) cannot safely store a client secret (it's visible in source code). Use PKCE which doesn't require a client secret.

3. **Not validating the `state` parameter** — The `state` parameter in OAuth2 prevents CSRF attacks on the authorization flow. Always generate a random `state` value, store it in session, and verify it matches the redirect callback.

4. **Confusing 401 and 403** — Return 401 when the user is not authenticated (no/invalid token). Return 403 when the user IS authenticated but lacks permission. Returning 404 instead of 403 (to hide resource existence) is sometimes appropriate for security but can confuse debugging.

5. **Missing scope validation** — An access token with `scope: read` should not be accepted by an endpoint that writes data. Always validate that the token's scopes include the required scope for the operation.

---

## 🧩 Mini Challenge

**Design question**: Your company is building a multi-tenant SaaS platform. Tenant A's users should never access Tenant B's data, even if they somehow get a valid JWT. What authorization strategy do you use?

<details>
<summary>💡 Click to reveal answer</summary>

**This is a classic multi-tenancy authorization problem — ABAC solves it.**

**JWT includes tenant claim**:
```json
{ "sub": "user_123", "tenant_id": "tenant_A", "roles": ["admin"] }
```

**Database rows include tenant_id**:
```sql
CREATE TABLE orders (
    order_id   BIGINT PRIMARY KEY,
    tenant_id  VARCHAR(50) NOT NULL,  -- every row scoped to a tenant
    user_id    BIGINT,
    ...
);
```

**Authorization strategy — two layers**:

**Layer 1: Query-level tenant isolation** (always enforce in repository)
```java
// Never query without tenant scope
public List<Order> findByOrderId(Long orderId, String tenantId) {
    return jdbcTemplate.query(
        "SELECT * FROM orders WHERE order_id = ? AND tenant_id = ?",
        orderId, tenantId);  // tenant_id always in WHERE clause
}
```

**Layer 2: Permission evaluator validates tenant match**
```java
public boolean canAccess(Authentication auth, Long orderId) {
    String tokenTenantId = jwt.getClaim("tenant_id");
    Order order = orderRepo.findById(orderId);  // this query includes tenant scoping
    return order != null && order.getTenantId().equals(tokenTenantId);
}
```

**Why both layers**: Layer 1 ensures the database never returns cross-tenant data even if authorization code has a bug. Defense in depth — each layer is a safety net for the other.

</details>

---

## 📝 Interview Q&A

**Q: What's the difference between OAuth2 and OIDC?**
> A: OAuth2 is an authorization framework — it gives an app an access token to act on behalf of a user. It doesn't specify WHO the user is. OIDC (OpenID Connect) is an authentication layer built on top of OAuth2 — it adds an ID Token (always a JWT) that contains the user's identity. Use OAuth2 for API access delegation; use OIDC for "login with Google"-style authentication.

**Q: How does the Authorization Code + PKCE flow protect against token theft?**
> A: The client generates a `code_verifier` (random secret), hashes it to `code_challenge`, and sends the hash to the authorization server. The auth server stores the hash. When the authorization code is exchanged for tokens, the client sends the original `code_verifier`. The auth server verifies `SHA256(code_verifier) == stored code_challenge`. An attacker who intercepts the authorization code can't use it without the `code_verifier`, which was never transmitted.

**Q: When would you use RBAC vs ABAC?**
> A: RBAC works well for most applications — simple role-based permissions are easy to implement, audit, and reason about. Choose ABAC when you need fine-grained control based on resource attributes, environmental context, or cross-cutting policies (e.g., "read your department's documents only during business hours from office IPs"). ABAC is common in healthcare, finance, and multi-tenant SaaS.

**Q: How do you implement SSO (Single Sign-On) across multiple services?**
> A: Use OIDC with a centralized Identity Provider (Keycloak, Okta, Auth0). All services are configured as OIDC Resource Servers pointing to the same IDP. User logs in once to the IDP → gets an ID Token + Access Token → presents Access Token to any service → each service validates the token against the IDP's public keys (JWKS endpoint) without redirecting to login. Token expiry + refresh is handled centrally.

---

## 🔗 What to Read Next

1. **[Security/TLS_SSL_HTTPS.md](./TLS_SSL_HTTPS.md)** — How OAuth2 tokens are protected in transit
2. **[Security/ZeroTrust_Architecture.md](./ZeroTrust_Architecture.md)** — Modern auth model: "never trust, always verify"
3. **[BuildingBlocks/APIGateway.md](../BuildingBlocks/APIGateway.md)** — How API Gateways enforce OAuth2 token validation centrally

---

*[← JWT Deep Dive](./JWT_Deep_Dive.md) | [Back to Security](./README.md) | [Next: TLS/SSL/HTTPS →](./TLS_SSL_HTTPS.md)*
