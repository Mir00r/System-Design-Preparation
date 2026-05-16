# 🏰 Security in Microservices: Defending a Distributed Attack Surface

> *"A monolith has one door to guard. A microservice architecture has 200 doors, 50 windows, and a ventilation shaft — each one a potential entry point. Security must be built into the fabric of the system, not bolted onto the perimeter."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [mTLS](./mTLS.md), [OAuth2](./OAuth2.md), [JWT Deep Dive](./JWT_Deep_Dive.md), [Zero Trust Architecture](./ZeroTrust_Architecture.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Security Challenges in Microservices](#-security-challenges-in-microservices)
3. [Authentication Patterns](#-authentication-patterns)
4. [Authorization Patterns](#-authorization-patterns)
5. [Network Security](#-network-security)
6. [Data Security](#-data-security)
7. [Secrets Management](#-secrets-management)
8. [Security Observability](#-security-observability)
9. [Spring Boot Implementation](#-spring-boot-implementation)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
MONOLITH SECURITY (relatively simple):
  [Internet] ──▶ [Firewall] ──▶ [Single App] ──▶ [Single DB]
  
  - One authentication point
  - Method-level authorization (in-process)
  - Encrypted at the boundary only
  - One set of secrets to manage
  - One deployment to audit

MICROSERVICES SECURITY (exponentially more complex):
  [Internet] ──▶ [API Gateway] ──▶ [Service A] ──▶ [Service B] ──▶ [Service C]
                        │                │               │               │
                        │           [DB-A]          [DB-B]          [DB-C]
                        │
                        └──▶ [Service D] ──▶ [Service E] ──▶ [External API]

  Challenges:
    - 200+ services = 200+ authentication/authorization endpoints
    - Service-to-service calls: who is allowed to call whom?
    - Token propagation: how does Service C know the original user?
    - Network: lateral movement after one service is compromised
    - Secrets: 200 services × 5 secrets each = 1000 secrets to manage
    - Attack surface: each service is independently deployable and exploitable
```

---

## 🔓 Security Challenges in Microservices

| Challenge | Monolith | Microservices |
|---|---|---|
| **Authentication** | One login, session in memory | Token-based, propagated across services |
| **Authorization** | Method annotations, single codebase | Distributed policy enforcement |
| **Network trust** | Internal calls are in-process | Network calls between services (attackable) |
| **Data isolation** | Single DB, single access layer | Multiple DBs, each service owns its data |
| **Secrets** | One config file | Distributed secrets across 200+ services |
| **Audit trail** | Single log | Correlated logs across services (distributed tracing) |
| **Attack blast radius** | Whole app compromised | One service compromised → can it spread? |
| **Dependency security** | One dependency tree | 200 dependency trees to keep patched |

---

## 🔑 Authentication Patterns

### Pattern 1: API Gateway Authentication (Centralized)

```
Most common pattern — gateway handles auth, passes identity downstream:

  [Client] ──Bearer token──▶ [API Gateway]
                                    │
                              1. Validate JWT
                              2. Extract user identity
                              3. Add X-User-Id, X-User-Roles headers
                                    │
                              ┌─────▼──────┐
                              │ Internal    │
                              │ services    │
                              │ trust the   │
                              │ gateway's   │
                              │ headers     │
                              └────────────┘

  Pros: Single point of token validation, simple internal services
  Cons: Gateway is a single point of failure/attack
  Mitigate: mTLS between gateway and services (prevent header spoofing)
```

### Pattern 2: Token Propagation (Distributed)

```
Each service validates the token independently:

  [Client] ──JWT──▶ [API Gateway] ──JWT──▶ [Order Service] ──JWT──▶ [Payment Service]

  Every service:
    1. Receives the JWT
    2. Validates signature (using shared public key / JWKS endpoint)
    3. Checks expiry, audience, issuer
    4. Extracts claims for authorization decisions

  Pros: No single point of trust, services work independently
  Cons: Token validation overhead on every service, token size in every request
```

### Pattern 3: Token Exchange (Service-specific tokens)

```
Gateway exchanges user token for service-specific tokens:

  [Client] ──user JWT──▶ [API Gateway]
                               │
                    Exchange user token for:
                    - order-service token (scopes: read:orders, write:orders)
                    - payment-service token (scopes: charge:payment)
                               │
                     ┌─────────▼──────────┐
                     │ Order Service       │
                     │ receives: token     │
                     │ with ONLY order     │
                     │ scopes (not payment)│
                     └────────────────────┘

  Pros: Principle of least privilege — each service gets minimal permissions
  Cons: Complexity, token exchange adds latency
  Used by: Netflix, Google (internal systems)
```

---

## 🛡️ Authorization Patterns

### Centralized Policy Engine (OPA / Open Policy Agent)

```
┌──────────────────────────────────────────────────────┐
│              AUTHORIZATION ARCHITECTURE               │
│                                                      │
│  [Service A]  "Can user X do action Y on resource Z?" │
│       │                                              │
│       ▼                                              │
│  [OPA Sidecar / Policy Service]                       │
│       │                                              │
│       │  Policy (Rego language):                      │
│       │    allow {                                    │
│       │      input.user.role == "editor"              │
│       │      input.action == "publish"               │
│       │      input.resource.owner == input.user.id   │
│       │    }                                          │
│       │                                              │
│       ▼                                              │
│  Response: { "allow": true/false, "reason": "..." }  │
└──────────────────────────────────────────────────────┘

Benefits:
  - Policies are code (versioned, tested, reviewed)
  - Decoupled from services (update policies without redeploying services)
  - Consistent enforcement across all services
  - Audit trail of all authorization decisions
```

### Distributed Authorization (Per-Service)

```
Each service enforces its own authorization rules:

  Order Service: "Can this user access this order?"
    → Check: order.userId == authenticatedUser.id OR user has ADMIN role

  Payment Service: "Can this service initiate a charge?"
    → Check: caller certificate CN is in allowlist [order-service, admin-service]

  Report Service: "Can this user generate this report?"
    → Check: user.subscription == "enterprise" AND user.department == report.department
```

---

## 🌐 Network Security

```
DEFENSE IN DEPTH (multiple layers):

Layer 1: PERIMETER (API Gateway / WAF)
  - DDoS protection (CloudFlare, AWS Shield)
  - Web Application Firewall (SQL injection, XSS blocking)
  - Rate limiting per client/IP
  - TLS termination

Layer 2: SERVICE MESH (mTLS everywhere)
  - All service-to-service traffic encrypted
  - Mutual authentication (services prove identity)
  - AuthorizationPolicy: which services can talk to which

Layer 3: NETWORK POLICIES (Kubernetes)
  apiVersion: networking.k8s.io/v1
  kind: NetworkPolicy
  metadata:
    name: payment-service-ingress
  spec:
    podSelector:
      matchLabels:
        app: payment-service
    ingress:
    - from:
      - podSelector:
          matchLabels:
            app: order-service
      # ONLY order-service pods can reach payment-service
      # All other pods → connection refused at network level

Layer 4: NAMESPACE ISOLATION
  - Sensitive services (payment, user-data) in isolated namespaces
  - Separate service accounts per service
  - RBAC: services can only access their own secrets/configmaps

BLAST RADIUS CONTAINMENT:
  If order-service is compromised:
    ❌ Cannot reach payment-service DB (network policy blocks)
    ❌ Cannot impersonate admin-service (wrong mTLS cert)
    ❌ Cannot read payment secrets (RBAC denies)
    ✅ Can only do what order-service is authorized to do
    ✅ Alert fires because order-service makes unusual API calls
```

---

## 🗄️ Data Security

```
DATA ISOLATION PRINCIPLE: Each service owns its data exclusively

  ❌ WRONG: Multiple services access the same database
    [Order Service] ──▶ ┌─────────────┐ ◀── [Payment Service]
                        │ Shared DB    │
                        └─────────────┘
    Problem: one compromised service → all data exposed

  ✅ RIGHT: Database-per-service with API access only
    [Order Service] ──▶ [Order DB]
    [Payment Service] ──▶ [Payment DB] (encrypted at rest)
    
    Order needs payment info? → calls Payment Service API (authorized, audited)

ENCRYPTION:
  At rest: AES-256 for databases, S3 server-side encryption
  In transit: TLS 1.3 between all services (mTLS)
  Application-level: encrypt PII fields before storing
    user.email → store as encrypted blob
    Decrypt only in the owning service, never expose raw PII to other services

DATA CLASSIFICATION:
  Public: product catalog, public profiles → minimal protection
  Internal: order details, user preferences → access control + encryption at rest
  Confidential: PII, payment data → encryption at rest + in transit + field-level encryption
  Restricted: auth credentials, encryption keys → HSM/Vault, never in application memory longer than needed
```

---

## 🔐 Secrets Management

```
SECRET LIFECYCLE IN MICROSERVICES:

  ❌ ANTI-PATTERNS:
    - Secrets in environment variables (visible in process list, crash dumps)
    - Secrets in Docker images (anyone with image access sees them)
    - Secrets in Git (even if later removed — still in history)
    - Shared secrets across services (one leak compromises everything)

  ✅ PROPER APPROACH: HashiCorp Vault / AWS Secrets Manager

  [Service starts]
       │
       ▼
  [Authenticate to Vault] (using K8s service account / IAM role)
       │
       ▼
  [Request secrets] (lease: 1 hour, auto-renewable)
       │
       ▼
  [Receive dynamic credentials]
    DB password: randomly generated, unique to this instance
    Expires in 1 hour → Vault generates new one automatically
       │
       ▼
  [Service uses credentials]
    If credentials expire → renew lease or get new ones
    If service crashes → credentials expire automatically (no manual cleanup)

  DYNAMIC SECRETS (gold standard):
    Each service instance gets UNIQUE database credentials
    Credentials are short-lived (1 hour)
    Compromised credential → only affects one instance, expires soon
    Vault audit log shows exactly which service accessed which secret
```

---

## 📡 Security Observability

```
SECURITY MONITORING ACROSS MICROSERVICES:

  What to monitor:
    1. Authentication failures (spike = credential stuffing attack)
    2. Authorization denials (spike = privilege escalation attempt)
    3. Unusual inter-service call patterns (lateral movement)
    4. Data volume anomalies (bulk export = data exfiltration)
    5. Certificate errors (expired cert = misconfiguration or attack)

  Distributed tracing for security:
    Every request gets a trace ID propagated across services:
    
    TraceID: abc-123
    [Gateway] → [Order Service] → [Payment Service] → [Notification Service]
    
    Security team can trace: "Who initiated this payment? Through which path?"
    Detect: "Why is logging-service calling payment-service?" (never happened before → alert!)

  SIEM Integration:
    All services → structured security logs → Kafka → SIEM (Splunk/ELK)
    Correlation rules:
      IF (auth_failures > 50 from same IP in 5 min) → ALERT: brute force
      IF (service A calls service B for first time) → ALERT: new communication pattern
      IF (data export > 10x normal volume) → ALERT: potential exfiltration
```

---

## 💻 Spring Boot Implementation

```java
// 1. JWT validation at service level (token propagation pattern)
@Configuration
@EnableWebSecurity
public class ServiceSecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwkSetUri("https://auth-service/.well-known/jwks.json")
                    .jwtAuthenticationConverter(jwtConverter())
                ))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/internal/**").hasAuthority("SCOPE_internal")
                .anyRequest().authenticated()
            )
            .build();
    }
    
    private JwtAuthenticationConverter jwtConverter() {
        var converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            var scopes = jwt.getClaimAsStringList("scopes");
            return scopes.stream()
                .map(s -> new SimpleGrantedAuthority("SCOPE_" + s))
                .collect(Collectors.toList());
        });
        return converter;
    }
}

// 2. Propagate security context to downstream calls
@Component
public class TokenPropagationInterceptor implements ClientHttpRequestInterceptor {
    
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
            ClientHttpRequestExecution execution) throws IOException {
        
        // Propagate the incoming JWT to downstream services
        var authentication = SecurityContextHolder.getContext().getAuthentication();
        if (authentication instanceof JwtAuthenticationToken jwtAuth) {
            request.getHeaders().setBearerAuth(jwtAuth.getToken().getTokenValue());
        }
        
        // Also propagate trace context for security auditing
        request.getHeaders().set("X-Trace-Id", MDC.get("traceId"));
        
        return execution.execute(request, body);
    }
}

// 3. Inter-service authorization (verify caller identity from mTLS cert)
@Aspect
@Component
public class ServiceAuthorizationAspect {
    
    @Before("@annotation(requireService)")
    public void verifyCallerService(RequireService requireService) {
        HttpServletRequest request = getCurrentRequest();
        String callerService = request.getHeader("X-Caller-Service");
        
        // In mTLS setup, this header is set by the sidecar proxy
        // from the client certificate's CN field (cannot be spoofed)
        if (!Set.of(requireService.allowed()).contains(callerService)) {
            throw new AccessDeniedException(
                "Service " + callerService + " not authorized for this endpoint");
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequireService {
    String[] allowed();
}

// Usage:
@RequireService(allowed = {"order-service", "admin-service"})
@PostMapping("/internal/charge")
public PaymentResult charge(@RequestBody ChargeRequest request) {
    return paymentService.processCharge(request);
}

// 4. Security event logging
@Component
public class SecurityEventLogger {
    private static final Logger securityLog = LoggerFactory.getLogger("SECURITY");
    
    @EventListener
    public void onAuthFailure(AuthenticationFailureBadCredentialsEvent event) {
        securityLog.warn("AUTH_FAILURE user={} ip={} reason={}",
            event.getAuthentication().getName(),
            getClientIp(),
            event.getException().getMessage());
    }
    
    @EventListener  
    public void onAccessDenied(AuthorizationDeniedEvent event) {
        securityLog.warn("ACCESS_DENIED user={} resource={} action={}",
            event.getAuthentication().getName(),
            event.getResource(),
            event.getAction());
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Trusting internal network traffic** — "It's inside the VPC, so it's safe" is the #1 reason lateral movement succeeds. Enforce mTLS + authorization policies for ALL service-to-service communication, even on the same host.

2. **JWT without validation** — Services that decode JWT payload without verifying the signature are vulnerable to token forgery. Always validate: signature (is it signed by our auth server?), expiry (is it still valid?), audience (is it intended for this service?), issuer (did our auth server issue it?).

3. **Shared database access** — Multiple services accessing the same database creates a massive blast radius. If one service is compromised, the attacker accesses ALL data. Enforce database-per-service ownership with API-level access.

4. **Static long-lived secrets** — Database passwords that never change and are the same across all instances. If leaked once, they're valid forever. Use dynamic secrets from Vault (unique per instance, expire in hours).

5. **No security monitoring for internal traffic** — Most APM tools focus on performance, not security. Without security-specific monitoring, compromised services can exfiltrate data for weeks before detection. Implement baseline behavior models and alert on deviations.

---

## 🧩 Mini Challenge

**Service A calls Service B, which calls Service C. The user authenticated at Service A. How does Service C know:**
1. **Who the original user is?**
2. **That the request legitimately came through Service A → B → C (not directly from attacker to C)?**

<details>
<summary>💡 Click to reveal answer</summary>

**Solution: Combination of token propagation + mTLS + call chain verification**

**1. Who is the original user?** — JWT Token Propagation:
```
Service A receives user's JWT → validates → passes JWT to Service B → passes to Service C
Service C validates JWT and extracts user identity from claims

Alternative (more secure): Token exchange at each hop
  A has user-token → exchanges for B-scoped token → B exchanges for C-scoped token
  Each token has narrower scopes (principle of least privilege)
  Token includes "act" claim showing the chain: { "sub": "user123", "act": { "sub": "serviceA" } }
```

**2. Legitimate call chain verification?** — mTLS + call context:
```
Layer 1: mTLS ensures Service C knows the immediate caller is Service B
  (B's certificate proves it's really B, not an attacker impersonating B)

Layer 2: Call chain header (set by service mesh, not application):
  X-Forwarded-Client-Cert: includes the full chain [gateway → A → B]
  Service C can verify: "this request originated at the gateway, went through A, then B"
  
Layer 3: Authorization policy at Service C:
  ALLOW if:
    - mTLS caller == service-B (verified by certificate)
    - JWT is valid and not expired
    - JWT audience includes "service-C"
    - Call chain matches expected pattern (A → B → C)
  DENY otherwise

If attacker calls C directly:
  ❌ No valid mTLS cert for service-B → connection rejected
  ❌ Even with a stolen JWT, network policy blocks direct access to C
  ❌ Call chain header won't show proper A → B → C flow
```

**Industry approach (Google BeyondCorp / Netflix)**: Each service has a unique SPIFFE identity. The service mesh tracks the full call graph. Authorization decisions consider not just "who is the user" and "who is the caller" but "what was the full request path." Anomalous paths trigger alerts.

</details>

---

## 📝 Interview Q&A

**Q: How do you handle authentication in a microservice architecture?**
> A: The most common pattern is API Gateway authentication with token propagation. The gateway validates the user's credentials (OAuth 2.0 / JWT), then propagates the validated token (or a stripped-down internal token) to downstream services. Each service validates the token's signature using a shared JWKS endpoint. For service-to-service calls not initiated by a user (batch jobs, async events), use mTLS with service identity certificates. The key principle: authenticate at the edge, authorize at each service. Services must never trust headers alone without cryptographic verification.

**Q: What's the blast radius of a compromised microservice, and how do you contain it?**
> A: Blast radius = what an attacker can access after compromising one service. Containment strategies: (1) **mTLS + authorization policies** — compromised service can only call services it's explicitly authorized to reach. (2) **Database-per-service** — attacker only accesses that service's data, not the entire system. (3) **Short-lived credentials** — Vault-issued DB passwords expire in 1 hour; attacker's access is time-limited. (4) **Network policies** — K8s NetworkPolicy blocks unauthorized network paths at the kernel level. (5) **Behavioral monitoring** — alert when a service makes unusual API calls (e.g., logging-service calling payment-service for the first time). Combined, these turn a breach from "full system compromise" to "one service's data for a limited time with immediate detection."

---

## 🔗 What to Read Next

1. **[Security/mTLS.md](./mTLS.md)** — Deep dive into mutual TLS for service-to-service authentication
2. **[Security/Secrets_Management.md](./Secrets_Management.md)** — HashiCorp Vault and dynamic secrets
3. **[Observability/Distributed_Tracing.md](../Observability/Distributed_Tracing.md)** — Trace requests across services for security auditing

---

*[← API Security Best Practices](./API_Security_Best_Practices.md) | [Back to Security](../INDEX.md)*
