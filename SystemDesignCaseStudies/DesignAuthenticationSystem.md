# 🔐 Design an Authentication System

> *"Every app needs to answer one fundamental question: 'Who are you?' Authentication is the gateway to everything — and getting it wrong means either locking out legitimate users or letting attackers waltz in. From JWT tokens to OAuth flows to MFA, understanding auth architecture is non-negotiable for any system design interview."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [Security Basics](../Security/), [REST APIs](../APIs/RESTful.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Authentication vs Authorization](#-authentication-vs-authorization)
3. [Auth Mechanisms](#-auth-mechanisms)
4. [JWT Deep Dive](#-jwt-deep-dive)
5. [OAuth 2.0 & OpenID Connect](#-oauth-20--openid-connect)
6. [Session Management](#-session-management)
7. [Multi-Factor Authentication](#-multi-factor-authentication)
8. [High-Level Architecture](#-high-level-architecture)
9. [Java Implementation](#-java-implementation)
10. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • User registration (email/password, social login)
  • Login / Logout
  • Password reset (secure email flow!)
  • Multi-factor authentication (TOTP, SMS, push)
  • OAuth 2.0 (login with Google/GitHub/Facebook)
  • API key management (service-to-service auth)
  • Session management (view active sessions, revoke!)
  
NON-FUNCTIONAL:
  • Security: resist brute force, credential stuffing, XSS, CSRF
  • Latency: auth check < 5ms (token validation!)
  • Availability: 99.99% (auth down = EVERYTHING down!)
  • Scalability: millions of concurrent sessions
  • Compliance: GDPR, SOC2 (audit logging!)

SECURITY REQUIREMENTS:
  • Passwords: bcrypt/Argon2 hashed (NEVER plaintext!)
  • Tokens: short-lived access + long-lived refresh
  • Rate limiting: prevent brute force (5 attempts → lockout)
  • HTTPS only (never transmit credentials over HTTP!)
```

---

## 🔑 Authentication vs Authorization

```
AUTHENTICATION (AuthN): "WHO are you?"
  → Verify identity (username + password, token, certificate)
  → Result: "This is User #42"

AUTHORIZATION (AuthZ): "WHAT can you do?"
  → Check permissions after identity is confirmed!
  → Result: "User #42 can read posts but cannot delete users"

FLOW:
  ┌──────┐     ┌──────────────┐     ┌──────────────┐
  │ User │────►│ Authentication│────►│ Authorization │
  │      │     │ "Who are you?"│     │ "Can you do X?"│
  └──────┘     └──────────────┘     └──────────────┘
                     │                      │
               Identity confirmed!    Permission checked!
               JWT issued!            RBAC/ABAC evaluated!

COMMON PATTERNS:
  ┌───────────────────────────────────────────────────────────────┐
  │  AuthN (Identity)         │  AuthZ (Permissions)              │
  ├───────────────────────────────────────────────────────────────┤
  │  Username/Password        │  RBAC (Role-Based)                │
  │  OAuth 2.0 tokens         │  ABAC (Attribute-Based)           │
  │  API Keys                 │  ACL (Access Control Lists)       │
  │  Certificates (mTLS)      │  Policy engines (OPA, Casbin)     │
  │  Biometrics (fingerprint) │  Scope-based (OAuth scopes)       │
  └───────────────────────────────────────────────────────────────┘
```

---

## 🛡️ Auth Mechanisms

```
┌────────────────────────────────────────────────────────────────────────┐
│  Mechanism           │  How It Works           │  Use Case             │
├────────────────────────────────────────────────────────────────────────┤
│  Session Cookie      │  Server stores session  │  Traditional web apps │
│                      │  Client gets cookie ID  │  (server-rendered!)   │
├────────────────────────────────────────────────────────────────────────┤
│  JWT (Bearer Token)  │  Stateless! Token has   │  APIs, SPAs, mobile   │
│                      │  claims, signed by server│  microservices!       │
├────────────────────────────────────────────────────────────────────────┤
│  API Key             │  Long-lived secret key  │  Server-to-server     │
│                      │  in header/query param  │  (Stripe, AWS)        │
├────────────────────────────────────────────────────────────────────────┤
│  OAuth 2.0           │  Delegated auth via     │  "Login with Google"  │
│                      │  authorization server   │  Third-party access   │
├────────────────────────────────────────────────────────────────────────┤
│  mTLS (mutual TLS)   │  Both sides present     │  Service mesh         │
│                      │  certificates!          │  (zero-trust infra)   │
├────────────────────────────────────────────────────────────────────────┤
│  SAML                │  XML-based SSO          │  Enterprise SSO       │
│                      │  (legacy but common!)   │  (Okta, ADFS)         │
└────────────────────────────────────────────────────────────────────────┘

WHEN TO USE WHICH:
  Web app (server-rendered) → Session cookies!
  SPA + API → JWT (short-lived access token!)
  Mobile app → OAuth 2.0 + PKCE (no client secret!)
  Microservices → mTLS or JWT propagation!
  Third-party integrations → API keys (rotatable!)
  Enterprise SSO → SAML or OIDC!
```

---

## 🎫 JWT Deep Dive

```
JWT (JSON Web Token): Self-contained, signed token!

STRUCTURE (3 parts, base64-encoded, dot-separated):
  xxxxx.yyyyy.zzzzz
  header.payload.signature
  
  HEADER: {"alg": "RS256", "typ": "JWT"}
  PAYLOAD: {
    "sub": "user-42",        ← Subject (who)
    "iat": 1700000000,       ← Issued At
    "exp": 1700003600,       ← Expires (1 hour!)
    "roles": ["admin"],      ← Custom claims
    "email": "user@example.com"
  }
  SIGNATURE: RS256(base64(header) + "." + base64(payload), privateKey)

WHY JWT?
  ✅ STATELESS! No server-side session store needed!
  ✅ Any service can verify (just needs public key!)
  ✅ Contains claims (roles, permissions) — no DB lookup!
  ✅ Works across services (pass in header!)
  
  ❌ Cannot be revoked instantly! (valid until expiry!)
  ❌ Large token size (1-2 KB vs 36-byte session ID!)
  ❌ Payload is readable (base64, not encrypted!)
  
TOKEN STRATEGY (access + refresh):
  ┌─────────────────────────────────────────────────────────┐
  │  Token Type     │  Lifetime  │  Storage        │ Purpose │
  ├─────────────────────────────────────────────────────────┤
  │  Access Token   │  15 min    │  Memory (JS)    │ API auth │
  │  Refresh Token  │  7 days    │  HttpOnly cookie│ Get new  │
  │                 │            │  (secure!)      │ access!  │
  └─────────────────────────────────────────────────────────┘
  
  Flow:
  1. Login → receive access token (15 min) + refresh token (7 days)
  2. API calls: Authorization: Bearer <access_token>
  3. Access expires → POST /auth/refresh (with refresh token cookie)
  4. Get new access token! (silent refresh, no re-login!)
  5. Refresh expires → user must login again.

REVOCATION STRATEGIES (since JWTs are stateless!):
  • Short expiry (15 min) — limits damage window!
  • Blocklist: store revoked token IDs in Redis (check on each request)
  • Token versioning: user.tokenVersion++ → all old tokens invalid!
  • Refresh token rotation: each use issues NEW refresh token!
```

---

## 🔗 OAuth 2.0 & OpenID Connect

```
OAuth 2.0: AUTHORIZATION framework (delegated access!)
OpenID Connect (OIDC): AUTHENTICATION layer on top of OAuth!

"Login with Google" flow (Authorization Code + PKCE):

  ┌──────┐        ┌──────────┐        ┌───────────────┐
  │ User │        │ Your App │        │ Google (IdP)  │
  └──┬───┘        └────┬─────┘        └──────┬────────┘
     │                  │                      │
     │ 1. Click "Login with Google"            │
     │─────────────────►│                      │
     │                  │ 2. Redirect to Google │
     │                  │  /authorize?          │
     │                  │  client_id=XYZ&       │
     │                  │  redirect_uri=...&    │
     │                  │  scope=openid email&  │
     │                  │  code_challenge=...   │
     │◄─────────────────│─────────────────────►│
     │                  │                      │
     │ 3. User logs in at Google               │
     │ 4. User consents to sharing email       │
     │──────────────────────────────────────────►│
     │                  │                      │
     │ 5. Google redirects back with CODE      │
     │◄────────────────────────────────────────│
     │─────────────────►│                      │
     │                  │ 6. Exchange code for tokens │
     │                  │  POST /token          │
     │                  │  code=ABC&            │
     │                  │  code_verifier=...    │
     │                  │─────────────────────►│
     │                  │                      │
     │                  │ 7. Receive:          │
     │                  │  access_token (API access) │
     │                  │  id_token (user identity!) │
     │                  │  refresh_token       │
     │                  │◄─────────────────────│
     │                  │                      │
     │ 8. Your app creates local session       │
     │◄─────────────────│                      │

PKCE (Proof Key for Code Exchange):
  Prevents authorization code interception!
  Used for: mobile apps, SPAs (public clients!)
  
  code_verifier: random string
  code_challenge: SHA256(code_verifier) ← sent in step 2
  code_verifier: sent in step 6 (Google verifies it matches!)
```

---

## 🗂️ Session Management

```
STATEFUL SESSIONS (Traditional):
  Client: Cookie: session_id=abc123
  Server: Redis: abc123 → {userId: 42, roles: ["admin"], exp: ...}
  
  Advantages:
  • Instant revocation (delete from Redis → logged out!)
  • Small cookie (36-byte UUID!)
  • Server controls all state
  
  Disadvantages:
  • Requires shared session store (Redis!)
  • Every request → Redis lookup (1ms overhead)
  • Harder in microservices (all services need Redis access!)

STATELESS TOKENS (JWT):
  Client: Authorization: Bearer eyJhbG...
  Server: Verify signature → trust claims in token!
  
  Advantages:
  • No session store needed!
  • Any service can verify (just needs public key!)
  • Works across domains easily!
  
  Disadvantages:
  • Can't revoke instantly
  • Larger payload sent with every request
  • Must handle refresh carefully

HYBRID (Best Practice!):
  • JWT for service-to-service (stateless, fast!)
  • Session + Redis for user-facing (revocable!)
  
  Or: JWT with short expiry + Redis blocklist for revoked tokens!
  
SESSION SECURITY:
  • HttpOnly flag: JavaScript cannot read cookie! (XSS protection)
  • Secure flag: only sent over HTTPS!
  • SameSite=Strict: prevents CSRF!
  • Rotation: new session ID after login (fixation attack prevention!)
  • Idle timeout: 30 min of inactivity → session expires!
  • Absolute timeout: 8 hours max → must re-authenticate!
```

---

## 📱 Multi-Factor Authentication

```
SOMETHING YOU KNOW + SOMETHING YOU HAVE + SOMETHING YOU ARE

  ┌───────────────────────────────────────────────────────────────┐
  │  Factor              │  Example              │  Strength       │
  ├───────────────────────────────────────────────────────────────┤
  │  Knowledge (know)    │  Password, PIN        │  Weakest alone  │
  │  Possession (have)   │  Phone, hardware key  │  Strong!        │
  │  Inherence (are)     │  Fingerprint, face    │  Strongest!     │
  └───────────────────────────────────────────────────────────────┘

MFA METHODS (ranked by security):
  1. Hardware keys (YubiKey, FIDO2/WebAuthn) — PHISHING PROOF! 🏆
  2. Authenticator app (TOTP: Google Auth, Authy) — good!
  3. Push notification (Duo, MS Authenticator) — good!
  4. SMS code — WEAK! (SIM swap attacks!) but better than nothing.

TOTP (Time-based One-Time Password):
  1. Setup: server generates secret key, shows QR code
  2. App scans QR → stores secret key
  3. On login: app computes TOTP = HMAC-SHA1(secret, time/30)
  4. User enters 6-digit code
  5. Server computes same TOTP, compares!
  
  Why it works: both sides have same secret + same clock!
  30-second window (slight clock skew tolerance!)

WEBAUTHN (passwordless future!):
  • Browser-native API for hardware key auth!
  • Public-key crypto (private key NEVER leaves device!)
  • Phishing-proof (bound to origin domain!)
  • Supports: fingerprint, face ID, hardware keys!
  
  Flow:
  1. Register: browser generates key pair, sends public key to server
  2. Login: server sends challenge → browser signs with private key
  3. Server verifies signature → authenticated!
  
  No password to steal! No phishing possible! The future of auth! 🚀
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTHENTICATION SERVICE                             │
│                                                                       │
│  ┌─────────┐    ┌──────────────────────────────────────────────┐    │
│  │  Client  │    │  API Gateway / Auth Middleware               │    │
│  │  (SPA)   │───►│  • Validate JWT on every request!           │    │
│  └─────────┘    │  • Attach user context to request           │    │
│                  │  • Rate limit auth endpoints!                │    │
│                  └────────────────┬─────────────────────────────┘    │
│                                   │                                   │
│                  ┌────────────────▼─────────────────────────────┐    │
│                  │  Auth Service                                 │    │
│                  │  • /register → create user                   │    │
│                  │  • /login → verify creds, issue tokens       │    │
│                  │  • /refresh → new access token               │    │
│                  │  • /logout → revoke refresh token            │    │
│                  │  • /mfa/setup → generate TOTP secret         │    │
│                  │  • /mfa/verify → validate OTP code           │    │
│                  │  • /oauth/callback → handle OAuth flow       │    │
│                  └───┬──────────┬──────────────┬────────────────┘    │
│                      │          │              │                      │
│              ┌───────▼──┐ ┌────▼──────┐ ┌────▼──────────┐          │
│              │  User DB  │ │  Redis    │ │  Identity     │          │
│              │  (Postgres)│ │  Sessions │ │  Providers    │          │
│              │  • users   │ │  + Revoked│ │  (Google,     │          │
│              │  • passwords│ │  tokens  │ │   GitHub)     │          │
│              │  • MFA keys│ │           │ │               │          │
│              └───────────┘ └───────────┘ └───────────────┘          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘

TOKEN VALIDATION FLOW (per request):
  Request arrives with: Authorization: Bearer <jwt>
  
  1. Decode JWT header (get key ID!)
  2. Verify signature (public key from JWKS endpoint!)
  3. Check expiry (exp claim > now?)
  4. Check blocklist (revoked? → Redis SET lookup!)
  5. Extract claims (userId, roles, permissions)
  6. Attach to request context → forward to service!
  
  Total: ~1ms (no DB call if not blocklisted!)
```

---

## 💻 Java Implementation

### Spring Security + JWT

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // Stateless → no CSRF!
            .sessionManagement(s -> s.sessionCreationPolicy(STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter(), UsernamePasswordAuthenticationFilter.class)
            .build();
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Cost factor 12!
    }
}

@Service
public class AuthService {
    
    @Autowired private UserRepository userRepo;
    @Autowired private PasswordEncoder passwordEncoder;
    @Autowired private JwtTokenProvider tokenProvider;
    @Autowired private RedisTemplate<String, String> redis;
    
    public AuthResponse login(LoginRequest request) {
        User user = userRepo.findByEmail(request.getEmail())
            .orElseThrow(() -> new AuthException("Invalid credentials"));
        
        // Verify password (bcrypt compare!)
        if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
            recordFailedAttempt(user.getId());
            throw new AuthException("Invalid credentials");
        }
        
        // Check if MFA required
        if (user.isMfaEnabled()) {
            return AuthResponse.mfaRequired(user.getId());
        }
        
        return issueTokens(user);
    }
    
    public AuthResponse refresh(String refreshToken) {
        // Validate refresh token
        Claims claims = tokenProvider.validateToken(refreshToken);
        
        // Check if revoked!
        if (redis.hasKey("revoked:" + claims.getId())) {
            throw new AuthException("Token revoked!");
        }
        
        User user = userRepo.findById(claims.getSubject()).orElseThrow();
        
        // Refresh token rotation: invalidate old, issue new!
        redis.opsForValue().set("revoked:" + claims.getId(), "1",
            Duration.ofDays(7)); // Keep until original would expire
        
        return issueTokens(user);
    }
    
    private AuthResponse issueTokens(User user) {
        String accessToken = tokenProvider.createAccessToken(
            user.getId(), user.getRoles(), Duration.ofMinutes(15));
        
        String refreshToken = tokenProvider.createRefreshToken(
            user.getId(), Duration.ofDays(7));
        
        return new AuthResponse(accessToken, refreshToken);
    }
    
    /**
     * Brute force protection: lock account after 5 failures.
     */
    private void recordFailedAttempt(String userId) {
        String key = "login_attempts:" + userId;
        Long attempts = redis.opsForValue().increment(key);
        redis.expire(key, Duration.ofMinutes(15));
        
        if (attempts != null && attempts >= 5) {
            // Lock account for 15 minutes!
            redis.opsForValue().set("locked:" + userId, "1", 
                Duration.ofMinutes(15));
            throw new AuthException("Account locked. Try again in 15 minutes.");
        }
    }
}
```

### JWT Token Provider

```java
@Component
public class JwtTokenProvider {
    
    private final RSAPrivateKey privateKey;
    private final RSAPublicKey publicKey;
    
    public String createAccessToken(String userId, List<String> roles, 
                                     Duration expiry) {
        return Jwts.builder()
            .setId(UUID.randomUUID().toString()) // jti for revocation!
            .setSubject(userId)
            .claim("roles", roles)
            .setIssuedAt(new Date())
            .setExpiration(Date.from(Instant.now().plus(expiry)))
            .signWith(privateKey, SignatureAlgorithm.RS256)
            .compact();
    }
    
    public Claims validateToken(String token) {
        try {
            return Jwts.parserBuilder()
                .setSigningKey(publicKey) // Verify with public key!
                .build()
                .parseClaimsJws(token)
                .getBody();
        } catch (ExpiredJwtException e) {
            throw new AuthException("Token expired");
        } catch (JwtException e) {
            throw new AuthException("Invalid token");
        }
    }
}
```

---

## ❓ Interview Q&A

**Q1: JWT vs Session — when to use which?**
> JWT for: stateless APIs, microservices (any service can verify without shared state), mobile apps (tokens easy to store/send), cross-domain scenarios. Sessions for: traditional web apps needing instant revocation, when you need to track active sessions, when token size matters (36-byte session ID vs 1KB JWT). Hybrid approach: JWT between services (fast, no shared state) + session cookie for user-facing (revocable, smaller). Most production systems use both!

**Q2: How do you handle JWT revocation?**
> Since JWTs are stateless, you can't "delete" them. Strategies: (1) Short expiry (15 min) — limits damage window, combined with refresh token rotation, (2) Token blocklist in Redis — store revoked token IDs (jti claim), check on each request (adds ~1ms but enables instant revocation), (3) User version field — increment user.tokenVersion on logout/password change, reject tokens with old version, (4) Refresh token rotation — each use of refresh token issues a new one and invalidates the old (detects theft!).

**Q3: How do you prevent brute force attacks on login?**
> Defense in depth: (1) Rate limiting per IP (100 requests/min), (2) Account lockout after 5 failed attempts (15-min cooldown), (3) Progressive delays (1s, 2s, 4s, 8s between attempts), (4) CAPTCHA after 3 failures, (5) Credential stuffing detection (many different accounts from same IP), (6) Geo-blocking (alert/block login from unusual country), (7) bcrypt/Argon2 password hashing (intentionally slow — 100ms per check, makes bulk attacks infeasible).

**Q4: Explain the OAuth 2.0 Authorization Code flow with PKCE.**
> Used for SPAs and mobile apps (public clients). (1) App generates random code_verifier, computes code_challenge = SHA256(verifier), (2) Redirects to auth server with code_challenge, (3) User authenticates and consents, (4) Auth server redirects back with authorization code, (5) App exchanges code + code_verifier for tokens, (6) Auth server verifies SHA256(verifier) == stored challenge. PKCE prevents: attacker intercepting the redirect (they have the code but NOT the verifier, so they can't exchange it!). Without PKCE: stolen auth code → stolen tokens!

---

## 🔗 Related Topics
- [API Gateway](../BuildingBlocks/APIGateway.md) — Auth enforcement point
- [Rate Limiting](../BuildingBlocks/RateLimiting.md) — Brute force protection
- [Security](../Security/) — Broader security topics
- [Microservices Auth](../Microservices/) — Service-to-service auth

---

*"Authentication is like the lock on your front door. It doesn't matter how beautiful your house is inside — if the lock is broken, none of it is safe. And unlike a physical lock, your auth system gets attacked millions of times a day, automatically, by bots from every country on Earth." — Security Engineer* 🔐
