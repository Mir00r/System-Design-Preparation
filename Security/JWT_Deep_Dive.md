# 🔐 JWT Deep Dive
## From Base64 to Production-Grade Token Security

> *"JWTs are everywhere — and so are JWT vulnerabilities. Every year, security researchers find applications accepting tokens signed with 'none' as the algorithm, or using symmetric secrets hardcoded in source code. Understanding JWT internals is the difference between using them correctly and creating a critical authentication bypass."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: None

---

## 📋 Table of Contents
1. [What is JWT?](#-what-is-jwt)
2. [JWT Structure Deep Dive](#-jwt-structure-deep-dive)
3. [Signing Algorithms](#-signing-algorithms)
4. [Common Vulnerabilities](#-common-vulnerabilities)
5. [Token Lifecycle Management](#-token-lifecycle-management)
6. [Refresh Token Pattern](#-refresh-token-pattern)
7. [Code Examples](#-code-examples)
8. [JWT vs Sessions](#-jwt-vs-sessions)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

Before JWT, stateful sessions:
```
1. User logs in → server creates session in memory/DB
2. Server returns session_id cookie
3. Every request: server looks up session_id in DB to verify identity
4. Problem: DB lookup on EVERY request; hard to scale horizontally
```

JWT's solution: **stateless authentication**
```
1. User logs in → server creates a signed JWT with user claims
2. Client stores JWT (localStorage, cookie)
3. Every request: client sends JWT → server VERIFIES signature (no DB lookup)
4. If valid → trust the claims inside the token
```

---

## 🧬 JWT Structure Deep Dive

A JWT has 3 parts separated by dots:

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMjM0NTYiLCJuYW1lIjoiQWxpY2UiLCJyb2xlIjoiYWRtaW4iLCJpYXQiOjE3MTYwMDAwMDAsImV4cCI6MTcxNjAwMzYwMH0
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

**Part 1 — Header** (base64url decoded):
```json
{
  "alg": "RS256",    // signing algorithm — NEVER blindly trust this!
  "typ": "JWT"
}
```

**Part 2 — Payload** (base64url decoded):
```json
{
  "sub":   "123456",          // Subject: user ID (who the token is about)
  "name":  "Alice",
  "role":  "admin",
  "iat":   1716000000,        // Issued At: unix timestamp
  "exp":   1716003600,        // Expiration: 1 hour after iat
  "iss":   "auth.myapp.com",  // Issuer
  "aud":   "api.myapp.com"    // Audience (intended recipient)
}
```

**Part 3 — Signature**:
```
RSASHA256(
  base64url(header) + "." + base64url(payload),
  privateKey
)
```

> ⚠️ **Critical**: The payload is NOT encrypted — it's just base64 encoded. Anyone can decode and read it. NEVER put passwords, credit cards, or other secrets in a JWT payload.

---

## 🔑 Signing Algorithms

### Symmetric (HMAC) — HS256, HS384, HS512

```
Same secret key used to SIGN and VERIFY

sign:   signature = HMAC-SHA256(header.payload, secretKey)
verify: HMAC-SHA256(header.payload, secretKey) == signature ?

Pros:  Fast, simple
Cons:  Anyone who can verify can also CREATE tokens (same key)
       If your auth server and API servers share the secret, any API server
       that is compromised can issue new JWTs as any user

Use when: Single service, or all services are fully trusted
```

### Asymmetric (RSA/EC) — RS256, RS512, ES256

```
Private key SIGNS, Public key VERIFIES

sign:   signature = RSA_SIGN(header.payload, privateKey)    ← auth server only
verify: RSA_VERIFY(header.payload, signature, publicKey)    ← any service

Pros:  Auth server keeps private key secret; public key can be distributed
       A compromised API server cannot forge new tokens
Cons:  ~10x slower than HMAC; larger tokens

Use when: Microservices architecture where multiple services verify tokens
          Public key can be published as JWKS (JSON Web Key Set) endpoint
```

### Algorithm Comparison

| Algorithm | Type | Key Size | Speed | Use Case |
|---|---|---|---|---|
| HS256 | Symmetric | 256-bit secret | Fast | Monolith, single-service |
| HS512 | Symmetric | 512-bit secret | Fast | Same but stronger |
| RS256 | Asymmetric (RSA) | 2048-bit keypair | Slow | Microservices, OAuth2 |
| ES256 | Asymmetric (ECDSA) | 256-bit keypair | Medium | Mobile (smaller token) |
| EdDSA | Asymmetric (Ed25519) | 256-bit keypair | Fast | Modern, recommended |

---

## 🚨 Common Vulnerabilities

### Vulnerability 1: Algorithm Confusion (alg:none Attack)

```
The server uses the "alg" field in the header to determine verification.

Attack:
  1. Attacker has a valid JWT with role: "user"
  2. Attacker changes payload: role: "admin"
  3. Attacker sets header: { "alg": "none" }
  4. Attacker removes the signature
  5. Server uses alg from header → uses "none" → no verification → accepts!

Fix: NEVER use the algorithm specified in the JWT header.
     ALWAYS hardcode the expected algorithm on the server side:
     
     // WRONG:
     Jwts.parser().parseClaimsJws(token)  // trusts header's alg

     // CORRECT:
     Jwts.parserBuilder()
         .requireAlgorithm("RS256")        // hardcoded expected algorithm
         .setSigningKey(publicKey)
         .build()
         .parseClaimsJws(token);
```

### Vulnerability 2: RS256 → HS256 Confusion

```
Attack scenario:
  Server uses RS256. Public key is available (at /jwks endpoint).
  
  1. Server's RS256 public key is: "-----BEGIN PUBLIC KEY-----\n..."
  2. Attacker creates a new JWT: header = { "alg": "HS256" }
  3. Attacker SIGNS it with the server's PUBLIC key as the HMAC secret
  4. Server reads header → uses HS256 mode → verifies HMAC with... its PUBLIC key
  5. Since attacker used the same key to sign, verification passes!

Fix: Same as above — hardcode the expected algorithm. Never accept HS256 if you're an RS256 service.
```

### Vulnerability 3: Weak Secrets for HS256

```
Attack: JWT secret is "secret", "password123", or the app name.
        Attacker brute-forces the HMAC secret offline using jwt_tool or hashcat.
        Once cracked → can forge any token.

Fix: Use cryptographically random secrets of at least 256 bits:
     secret = SecureRandom.getInstance("SHA1PRNG").generateSeed(32)  // 256 bits
     Store in environment variable / secret manager, never in code.
```

### Vulnerability 4: Missing Expiry Check / Infinite Tokens

```
Attack: If tokens never expire or have very long TTL (1 year),
        a stolen token gives permanent access.

Fix:
  - Access tokens: short TTL = 15 minutes to 1 hour
  - Validate exp claim on every request
  - Implement token revocation via refresh token rotation
```

---

## 🔄 Token Lifecycle Management

```
  User Login
      │
      ▼
  Auth Server
  ├── Issues Access Token  (TTL: 15 min)  → stored in memory/cookie
  └── Issues Refresh Token (TTL: 30 days) → stored in HttpOnly cookie

  Every API Request:
      Client → sends Access Token in Authorization: Bearer {token}
      Server → validates signature + exp claim → grants access
                    (no DB lookup needed)

  Access Token Expires:
      Client → sends Refresh Token to /auth/refresh
      Auth Server → validates Refresh Token (checks DB for revocation)
      Auth Server → issues new Access Token + new Refresh Token (rotation)
      Old Refresh Token → immediately invalidated (revocation)

  Logout:
      Client → sends Refresh Token to /auth/logout
      Auth Server → adds Refresh Token to revocation list (Redis)
      Client → deletes both tokens from storage
```

### Token Revocation — The Stateless Problem

```
Problem: JWTs are stateless — server can't "invalidate" a token before expiry.
         If a user's Access Token is stolen, it's valid for up to 15 minutes.

Solutions:

Option 1: Short TTL + accept the risk
  - Access tokens expire in 15 min → maximum exposure window
  - Simple, truly stateless
  - Acceptable for most applications

Option 2: Token revocation blacklist (Redis)
  - Maintain a Redis SET of revoked token IDs (jti claim)
  - On every request: check if jti is in blacklist
  - TTL on blacklist entry = token's remaining lifetime (no storage leak)
  - Downside: one Redis lookup per request (but Redis is fast: < 1ms)

Option 3: Refresh Token Rotation (best practice)
  - Short-lived access tokens (15 min) — limited blast radius
  - Long-lived refresh tokens (30 days) — stored in DB, revocable
  - On logout/compromise: revoke refresh token in DB → user must re-login after 15 min max
```

---

## 💻 Code Examples

### Spring Boot JWT Implementation

```java
// JWT Service — Sign and Verify
@Service
public class JwtService {

    @Value("${jwt.private-key}")
    private RSAPrivateKey privateKey;

    @Value("${jwt.public-key}")
    private RSAPublicKey publicKey;

    private static final Duration ACCESS_TOKEN_TTL = Duration.ofMinutes(15);
    private static final Duration REFRESH_TOKEN_TTL = Duration.ofDays(30);

    public String generateAccessToken(User user) {
        Instant now = Instant.now();
        return Jwts.builder()
                .setSubject(String.valueOf(user.getId()))
                .claim("email", user.getEmail())
                .claim("roles", user.getRoles())
                .setIssuedAt(Date.from(now))
                .setExpiration(Date.from(now.plus(ACCESS_TOKEN_TTL)))
                .setIssuer("auth.myapp.com")
                .setAudience("api.myapp.com")
                .setId(UUID.randomUUID().toString())  // jti for revocation
                .signWith(privateKey, SignatureAlgorithm.RS256)  // hardcoded RS256
                .compact();
    }

    public Claims validateAndParseClaims(String token) {
        try {
            return Jwts.parserBuilder()
                    .setSigningKey(publicKey)
                    .requireIssuer("auth.myapp.com")
                    .requireAudience("api.myapp.com")
                    // Algorithm is hardcoded by the key type (RSA public key = RS256 only)
                    .build()
                    .parseClaimsJws(token)
                    .getBody();
        } catch (ExpiredJwtException e) {
            throw new AuthException("Token expired");
        } catch (JwtException e) {
            throw new AuthException("Invalid token: " + e.getMessage());
        }
    }
}

// Security Filter — Validate on every request
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Autowired private JwtService jwtService;
    @Autowired private RedisTemplate<String, String> redis;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        String token = extractToken(request);
        if (token == null) {
            chain.doFilter(request, response);
            return;
        }

        Claims claims = jwtService.validateAndParseClaims(token);

        // Optional: check revocation blacklist
        String jti = claims.getId();
        if (redis.hasKey("revoked_tokens:" + jti)) {
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            return;
        }

        // Set authentication context
        UsernamePasswordAuthenticationToken auth =
                new UsernamePasswordAuthenticationToken(
                        claims.getSubject(),
                        null,
                        parseAuthorities(claims));
        SecurityContextHolder.getContext().setAuthentication(auth);

        chain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

### Refresh Token Rotation

```java
@RestController
@RequestMapping("/v1/auth")
public class AuthController {

    @Autowired private JwtService jwtService;
    @Autowired private RefreshTokenRepository refreshTokenRepo;

    @PostMapping("/refresh")
    public ResponseEntity<TokenResponse> refresh(
            @CookieValue("refresh_token") String refreshToken) {

        // 1. Look up refresh token in DB (validates it was issued by us)
        RefreshToken stored = refreshTokenRepo.findByToken(refreshToken)
                .orElseThrow(() -> new AuthException("Invalid refresh token"));

        // 2. Check not expired
        if (stored.getExpiresAt().isBefore(Instant.now())) {
            throw new AuthException("Refresh token expired, please login again");
        }

        // 3. Rotate: invalidate old refresh token, issue new one
        refreshTokenRepo.delete(stored);  // revoke old token

        User user = stored.getUser();
        String newAccessToken = jwtService.generateAccessToken(user);
        String newRefreshToken = UUID.randomUUID().toString();

        refreshTokenRepo.save(RefreshToken.builder()
                .token(newRefreshToken)
                .user(user)
                .expiresAt(Instant.now().plus(Duration.ofDays(30)))
                .build());

        // 4. Return new access token in body, new refresh token in HttpOnly cookie
        ResponseCookie cookie = ResponseCookie.from("refresh_token", newRefreshToken)
                .httpOnly(true)    // not accessible by JavaScript (XSS protection)
                .secure(true)      // HTTPS only
                .sameSite("Strict")
                .maxAge(Duration.ofDays(30))
                .path("/v1/auth")  // only sent to auth endpoints
                .build();

        return ResponseEntity.ok()
                .header(HttpHeaders.SET_COOKIE, cookie.toString())
                .body(new TokenResponse(newAccessToken));
    }
}
```

---

## ⚖️ JWT vs Sessions

| Aspect | JWT (Stateless) | Server Sessions (Stateful) |
|---|---|---|
| **Storage** | Client-side (localStorage / cookie) | Server-side (Redis / DB) |
| **Scalability** | No shared state between servers | All servers need access to session store |
| **Revocation** | Hard (need blacklist or short TTL) | Instant (delete from session store) |
| **Performance** | Signature verify (no DB) | Redis lookup per request |
| **Token size** | 300-500 bytes | ~32 bytes (session ID only) |
| **Best for** | Microservices, APIs, mobile | Traditional web apps, high-security |
| **CSRF risk** | Lower if in Authorization header | Higher (cookies sent automatically) |

---

## ⚠️ Common Pitfalls

1. **Storing sensitive data in payload** — JWT payloads are base64-encoded, NOT encrypted. Anyone can decode them. Never put passwords, PII, or financial data in a JWT.

2. **Not validating the `aud` (audience) claim** — A JWT issued for `api.myapp.com` should not be accepted by `admin.myapp.com`. Always validate the audience to prevent token reuse across services.

3. **Long-lived access tokens** — 24-hour access tokens mean a stolen token gives 24-hour access. Keep access tokens short (15 min) and use refresh tokens for longevity.

4. **Storing JWTs in localStorage** — JavaScript-accessible storage is vulnerable to XSS attacks. Prefer HttpOnly cookies for refresh tokens. Access tokens can be in memory (they're short-lived anyway).

5. **Trusting the algorithm from the token header** — Always hardcode the expected algorithm server-side. Never trust the `alg` field from an untrusted client token.

---

## 🧩 Mini Challenge

**Security scenario**: You're reviewing a PR and you see this code:

```java
Jwt token = Jwts.parser()
    .setSigningKey(secretKey.getBytes())
    .parseClaimsJws(jwt);
String alg = token.getHeader().getAlgorithm();  // read from token
```

And separately:
```java
String secret = "myapp-jwt-secret";  // hardcoded
```

**What are the 2 security issues?**

<details>
<parameter name="summary">💡 Click to reveal answer</summary>

**Issue 1: Hardcoded, weak secret key**
The secret `"myapp-jwt-secret"` is only 17 characters (~136 bits). An attacker with a valid token can run an offline brute-force attack using tools like `hashcat` or `jwt_tool`. A 17-char alphanumeric secret can be cracked in minutes on modern hardware.

**Fix**: Generate a cryptographically random 256-bit secret and load it from environment variables:
```java
@Value("${jwt.secret}")  // loaded from env: JWT_SECRET=<256-bit hex string>
private String secret;
// Generate: openssl rand -hex 32
```

**Issue 2: Using alg from the token header (implicit)**
The old `Jwts.parser().setSigningKey(...)` API in older JJWT versions does not enforce the algorithm — it uses whatever the token header claims. This enables the `alg:none` attack and the RSA-to-HMAC confusion attack.

**Fix**: Use the newer `parserBuilder()` API which ties algorithm to the key type:
```java
Jwts.parserBuilder()
    .setSigningKey(Keys.hmacShaKeyFor(secretBytes))  // forces HS256/384/512
    .build()
    .parseClaimsJws(jwt);
```

</details>

---

## 📝 Interview Q&A

**Q: What's the difference between authentication and authorization?**
> A: Authentication = "who are you?" (verifying identity — JWT validates you are user #123456). Authorization = "what can you do?" (permissions — the `roles: ["admin"]` claim in the JWT determines what endpoints you can access).

**Q: How do you handle token revocation in a stateless JWT system?**
> A: Three strategies: (1) Short TTL (15 min) — accept small risk window; (2) Redis blacklist — store revoked `jti` claims in Redis with TTL equal to token's remaining lifetime; (3) Refresh token rotation — keep access tokens short-lived, revoke long-lived refresh tokens in the DB when logout/compromise is detected.

**Q: When would you choose sessions over JWTs?**
> A: Sessions are better when: (1) you need instant revocation (financial apps, healthcare), (2) you have a monolith with shared session store, (3) token payload would be too large with JWT (many permissions/claims), (4) you need strict audit trails. JWTs are better for stateless microservices, mobile APIs, and multi-domain auth.

**Q: What claims should every production JWT include?**
> A: `sub` (user ID), `iat` (issued at), `exp` (expiration), `iss` (issuer), `aud` (audience), `jti` (JWT ID for revocation tracking). Optionally: `nbf` (not before — useful for scheduled tokens).

---

## 🔗 What to Read Next

1. **[Security/Authentication_vs_Authorization.md](./Authentication_vs_Authorization.md)** — Expand on OAuth2, OIDC, and modern auth flows
2. **[Security/TLS_SSL_HTTPS.md](./TLS_SSL_HTTPS.md)** — How the transport layer protects JWT tokens in transit
3. **[Security/OWASP_Top10.md](./OWASP_Top10.md)** — Broader security vulnerabilities including those that can expose JWT secrets

---

*[← Back to Security](./README.md) | [Back to Index](../INDEX.md)*
