# 🔒 TLS, SSL & HTTPS
## How Encryption Protects Data in Transit — From Handshake to HTTP/2

> *"Every time you see the padlock in your browser, a 250ms cryptographic ceremony just happened between your device and a server halfway around the world — involving certificate chains, Diffie-Hellman key exchange, and symmetric encryption. Understanding it is essential for any engineer building secure systems."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [OWASP Top 10](./OWASP_Top10.md)

---

## 📋 Table of Contents
1. [SSL vs TLS — History](#-ssl-vs-tls--history)
2. [How TLS Works — The Handshake](#-how-tls-works--the-handshake)
3. [TLS 1.2 vs TLS 1.3](#-tls-12-vs-tls-13)
4. [X.509 Certificates](#-x509-certificates)
5. [Mutual TLS (mTLS)](#-mutual-tls-mtls)
6. [HTTPS Best Practices](#-https-best-practices)
7. [Spring Boot HTTPS Configuration](#-spring-boot-https-configuration)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

Without TLS, every byte you send over the internet is **plaintext**:

```
You type: POST /login  { "password": "MySecret123" }

What travels through:
  Your home router
  → Your ISP's network
  → Multiple internet exchange points
  → The server's datacenter network

At every hop, anyone with network access can:
  - READ your data (eavesdropping)
  - MODIFY your data (man-in-the-middle)
  - INJECT malicious content (SSL stripping, content injection)
```

TLS (formerly SSL) solves this with a combination of:
1. **Authentication** — verify you're talking to the real server, not an impostor
2. **Confidentiality** — encrypt data so eavesdroppers see only noise
3. **Integrity** — ensure data wasn't tampered with in transit

---

## 📜 SSL vs TLS — History

| Version | Year | Status |
|---|---|---|
| SSL 1.0 | 1994 | Never released (serious flaws) |
| SSL 2.0 | 1995 | Deprecated 2011 — multiple vulnerabilities (DROWN attack) |
| SSL 3.0 | 1996 | Deprecated 2015 — POODLE attack |
| TLS 1.0 | 1999 | Deprecated 2020 — vulnerable to BEAST, POODLE |
| TLS 1.1 | 2006 | Deprecated 2020 — better but still flawed |
| **TLS 1.2** | **2008** | **Still in use — secure if configured correctly** |
| **TLS 1.3** | **2018** | **Current standard — faster, simpler, more secure** |

> "SSL" is colloquially still used to mean TLS. When engineers say "SSL certificate," they mean a TLS certificate. The underlying protocol is TLS 1.2 or 1.3.

---

## 🤝 How TLS Works — The Handshake

### TLS 1.2 Handshake (Simplified)

```
Client                                          Server
  │                                               │
  │── ClientHello ─────────────────────────────→  │
  │   {TLS versions supported, cipher suites,     │
  │    random bytes (client_random)}              │
  │                                               │
  │  ←───────────────────────────── ServerHello ──│
  │   {TLS version chosen, cipher suite chosen,   │
  │    random bytes (server_random)}              │
  │                                               │
  │  ←──────────────────────── Certificate ───────│
  │   {Server's X.509 certificate}                │
  │   (contains server's public key)              │
  │                                               │
  │  Client:                                      │
  │  1. Verify certificate chain (→ trusted CA?)  │
  │  2. Verify server hostname matches cert       │
  │  3. Generate pre_master_secret                │
  │  4. Encrypt pre_master_secret with server's   │
  │     public key                                │
  │                                               │
  │── ClientKeyExchange ───────────────────────→  │
  │   {encrypted pre_master_secret}               │
  │                                               │
  │  Both sides now derive:                       │
  │  master_secret = PRF(pre_master_secret,       │
  │                      client_random,           │
  │                      server_random)           │
  │  session_keys = derive from master_secret     │
  │                                               │
  │── ChangeCipherSpec ────────────────────────→  │
  │── Finished (encrypted) ───────────────────→  │
  │  ←──────────────────── ChangeCipherSpec ──────│
  │  ←──────────────────── Finished (encrypted) ──│
  │                                               │
  │═══════════ Encrypted application data ════════│

Total extra round trips: 2 (1.5 RTT before data flows)
```

### TLS 1.3 Handshake — Faster

```
Client                                          Server
  │                                               │
  │── ClientHello ─────────────────────────────→  │
  │   {TLS 1.3, supported groups, key_share}      │
  │   (client sends its key share immediately)    │
  │                                               │
  │  ←───────────────────────────── ServerHello ──│
  │  ←─────────────────────────── Certificate ────│
  │  ←──────────────────────── CertificateVerify ─│
  │  ←────────────────────────────── Finished ────│
  │                                               │
  │── Finished ────────────────────────────────→  │
  │                                               │
  │═══════════ Encrypted application data ════════│

Total extra round trips: 1 (1 RTT before data flows, vs 2 in TLS 1.2)
With session resumption: 0-RTT (data sent with first message!)
```

### Key Insight: Why TLS 1.3 is Better

```
TLS 1.2:
  - Supports many cipher suites (some weak: RC4, DES, export ciphers)
  - 2 round trips before data flows
  - RSA key exchange: server's private key can decrypt past sessions if compromised
    (no Forward Secrecy)

TLS 1.3:
  - Only 5 cipher suites, all strong (AES-GCM, ChaCha20-Poly1305)
  - 1 round trip before data flows (0-RTT for resumed sessions)
  - Only Diffie-Hellman key exchange: ephemeral keys (Forward Secrecy guaranteed)
  - If server's private key is stolen later, past sessions cannot be decrypted
```

---

## 📄 X.509 Certificates

A certificate is a signed document that binds a **public key** to an **identity** (domain name).

```
Certificate contents:
  Subject:      CN=*.example.com, O=Example Inc, C=US
  Issuer:       CN=DigiCert TLS RSA SHA256 2020 CA1
  Valid from:   2025-01-01
  Valid until:  2026-01-01
  Public key:   RSA 2048-bit (the server's public key)
  Signature:    DigiCert's digital signature (proves DigiCert vouches for this cert)
  SANs:         *.example.com, example.com  (Subject Alternative Names)
```

### Certificate Chain of Trust

```
Root CA (self-signed, in your OS/browser trust store)
  └── Intermediate CA (signed by Root CA)
        └── Server Certificate (signed by Intermediate CA)
              └── Bound to: *.example.com

Browser verification:
  1. Server sends its certificate + intermediate CA cert
  2. Browser: "Is the intermediate CA signed by a trusted Root CA?" → YES
  3. Browser: "Is the server cert signed by this intermediate CA?" → YES
  4. Browser: "Does the cert's CN/SAN match the hostname?" → YES
  5. Browser: "Is the cert still valid (not expired, not revoked)?" → YES
  6. ✅ Secure connection established
```

### Certificate Types

| Type | Description | Cost | Validation |
|---|---|---|---|
| **DV (Domain Validated)** | Proves domain ownership | Free (Let's Encrypt) | Automated DNS/HTTP check |
| **OV (Org Validated)** | Proves org identity too | $50-200/yr | Manual CA verification |
| **EV (Extended Validation)** | Green bar, full org name | $200-500/yr | Extensive CA vetting |
| **Wildcard** | Covers `*.example.com` | Higher | DV or OV |
| **Multi-SAN** | One cert, multiple domains | Higher | Per-domain validation |

---

## 🤝 Mutual TLS (mTLS)

Standard TLS: only the SERVER proves its identity (client is anonymous).
mTLS: **BOTH** client and server prove their identity with certificates.

```
Standard TLS (HTTPS):
  Client: "Who are you?" → Server presents certificate → Client verifies
  Server: (trusts client based on session token/password after connection)

mTLS:
  Client: "Who are you?" → Server presents certificate → Client verifies ✅
  Server: "Who are YOU?" → Client presents certificate → Server verifies ✅

Use cases:
  - Service-to-service auth in microservices (no passwords needed)
  - Zero Trust networks (Istio/Envoy sidecar proxies use mTLS automatically)
  - Client API authentication (client cert replaces API key)
  - IoT device identity (each device has a unique cert)
```

### mTLS Architecture with Istio

```
Without mTLS (all microservices must manage their own auth):
  Service A ──→ Service B: "I'm Service A, here's my API key: abc123"
  Problem: key management, key rotation, key leakage

With Istio mTLS (Envoy sidecar proxies handle all auth transparently):
  Service A → [Envoy Proxy A] ══mTLS══ [Envoy Proxy B] → Service B
  Services communicate on localhost; proxies handle TLS cert rotation automatically
  Service A has NO API key — its identity IS its certificate
  Istio's control plane issues and rotates certs automatically every 24 hours
```

---

## ✅ HTTPS Best Practices

### 1. HTTP Strict Transport Security (HSTS)

```
Problem: User types "example.com" (no https://) → browser tries HTTP first
         Man-in-the-middle can intercept this initial HTTP request (SSL stripping)

Fix: HSTS header tells browsers to ALWAYS use HTTPS for your domain
     Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

     After first HTTPS visit, browser remembers for 1 year
     preload: submit to browser preload list (included even before first visit)
```

### 2. Certificate Transparency Logs

```
Problem: A rogue CA could issue a cert for "bank.com" to an attacker
         Standard chain validation wouldn't catch this

Fix: Certificate Transparency (CT)
  All CAs must log every issued certificate to public CT logs
  Browsers require Signed Certificate Timestamps (SCTs) proving the cert is logged
  Anyone can monitor CT logs for unauthorized certificates for their domain
```

### 3. OCSP Stapling

```
Problem: Browser must check if a cert is revoked (via OCSP or CRL)
         Old approach: browser queries OCSP server on every connection → slow + privacy leak

OCSP Stapling: Server periodically fetches its own OCSP response,
               "staples" it to the TLS handshake
               Browser gets revocation status without a separate OCSP query
               (faster + more private)
```

---

## ⚙️ Spring Boot HTTPS Configuration

### Enable HTTPS in Spring Boot

```yaml
# application.yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}  # from env var, never hardcoded
    key-store-type: PKCS12
    key-alias: myapp
    protocol: TLS
    enabled-protocols:
      - TLSv1.3
      - TLSv1.2       # keep for compatibility, TLS 1.3 preferred
    ciphers:
      - TLS_AES_128_GCM_SHA256         # TLS 1.3 suite
      - TLS_AES_256_GCM_SHA384         # TLS 1.3 suite
      - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256  # TLS 1.2 suite (with Forward Secrecy)
```

### Force HTTPS Redirect (HTTP → HTTPS)

```java
@Configuration
public class HttpsRedirectConfig {

    // Listen on port 8080 for HTTP, redirect to HTTPS 8443
    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");  // forces HTTPS
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        tomcat.addAdditionalTomcatConnectors(createHttpConnector());
        return tomcat;
    }

    private Connector createHttpConnector() {
        Connector connector = new Connector(TomcatServletWebServerFactory.DEFAULT_PROTOCOL);
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        connector.setRedirectPort(8443);
        return connector;
    }
}
```

### Add Security Headers

```java
@Configuration
public class SecurityHeadersConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.headers(headers -> headers
            .httpStrictTransportSecurity(hsts -> hsts
                .maxAgeInSeconds(31536000)   // 1 year
                .includeSubDomains(true)
                .preload(true)
            )
            .frameOptions(frame -> frame.deny())           // prevent clickjacking (X-Frame-Options: DENY)
            .contentTypeOptions(Customizer.withDefaults()) // X-Content-Type-Options: nosniff
            .xssProtection(Customizer.withDefaults())      // X-XSS-Protection
        );
        return http.build();
    }
}
```

### Let's Encrypt with Certbot (Production)

```bash
# Free TLS certificate, auto-renewed every 90 days
certbot certonly --standalone -d example.com -d www.example.com

# Outputs:
#   /etc/letsencrypt/live/example.com/fullchain.pem  (cert + chain)
#   /etc/letsencrypt/live/example.com/privkey.pem    (private key)

# Convert to PKCS12 for Java:
openssl pkcs12 -export \
  -in /etc/letsencrypt/live/example.com/fullchain.pem \
  -inkey /etc/letsencrypt/live/example.com/privkey.pem \
  -out keystore.p12 \
  -name myapp \
  -passout pass:${KEYSTORE_PASSWORD}

# Auto-renewal cron (every 12 hours, renews if expiring within 30 days):
0 0,12 * * * certbot renew --quiet
```

---

## ⚠️ Common Pitfalls

1. **Using self-signed certificates in production** — Browsers reject them, requiring users to click through security warnings. Use Let's Encrypt (free, auto-renewed) for any internet-facing service.

2. **Disabling TLS verification in code** — `trustAllCerts()` or `hostnameVerifier.verify() → return true` in HTTP clients completely defeats TLS security. Often added "temporarily" to fix dev environment issues and shipped to production.

3. **Not pinning certificates in mobile apps** — Mobile apps that trust all CAs can be intercepted by corporate proxies or Burp Suite during security testing (and by attackers). Certificate pinning embeds the expected certificate/public key hash in the app.

4. **Weak cipher suites** — Allowing RC4, DES, 3DES, or NULL ciphers for "backward compatibility" opens the connection to BEAST, SWEET32, and other attacks. Explicitly configure an allowlist of strong ciphers.

5. **Forgetting to renew certificates** — Let's Encrypt certs expire in 90 days. A forgotten renewal takes down HTTPS for your entire site. Always set up auto-renewal with monitoring alerts for cert expiry.

---

## 🧩 Mini Challenge

**Scenario**: Your microservice `order-service` calls `payment-service` over HTTPS. The call fails with a certificate error. A developer suggests: "Just set `ssl.verify=false` to fix it." What are the security implications, and what's the correct fix?

<details>
<summary>💡 Click to reveal answer</summary>

**Security implications of `ssl.verify=false`**:

Disabling TLS verification means the client **no longer checks**:
- Whether the server's certificate is valid
- Whether the certificate was issued by a trusted CA
- Whether the hostname in the cert matches the server you're connecting to

An attacker on the same network can now perform a **man-in-the-middle attack**: intercept the connection, present any certificate (even a self-signed one for a fake domain), and read/modify all traffic. This would expose payment data, PII, and auth tokens.

**Correct fix** (depends on the root cause):

1. **Dev/test environment** — `payment-service` likely uses a self-signed cert. Add the self-signed cert to a custom trust store:
```java
KeyStore trustStore = KeyStore.getInstance("PKCS12");
trustStore.load(new FileInputStream("dev-truststore.p12"), password);
SSLContext sslContext = SSLContexts.custom().loadTrustMaterial(trustStore, null).build();
```

2. **Production** — Certificate is expired or hostname mismatch: renew the cert and ensure the SAN includes the correct internal hostname.

3. **Service mesh** — If using Kubernetes with Istio, all internal traffic is mTLS managed automatically. Don't configure TLS manually; let the sidecar handle it.

</details>

---

## 📝 Interview Q&A

**Q: What is Forward Secrecy and why does it matter?**
> A: Forward Secrecy (also called Perfect Forward Secrecy, PFS) means that compromising the server's long-term private key today cannot decrypt past recorded sessions. TLS 1.3 achieves this by using ephemeral Diffie-Hellman key exchange — a unique session key is generated for each connection and discarded after the session ends. Even if attackers record encrypted traffic now and steal the server's private key later, they cannot decrypt past sessions.

**Q: What happens when a TLS certificate expires?**
> A: The browser shows a certificate error and blocks access (unless the user explicitly bypasses the warning — which most users won't do). For API clients, requests fail with SSL handshake errors. This causes complete outage of HTTPS endpoints. Prevention: monitor cert expiry (e.g., Prometheus `ssl_certificate_expiry` metric), set up automated renewal (Let's Encrypt + certbot), and alert 30 days before expiry.

**Q: How does HTTPS protect against man-in-the-middle attacks?**
> A: The certificate's digital signature (from a trusted CA) binds the server's public key to its domain name. If an attacker intercepts and substitutes their own public key, the certificate chain check fails — the forged cert won't have a valid signature from a trusted CA. Additionally, HSTS prevents protocol downgrade (SSLstripping) attacks by ensuring browsers only use HTTPS for the domain.

**Q: What's the difference between TLS termination at the load balancer vs end-to-end TLS?**
> A: **TLS termination at LB**: the load balancer decrypts TLS and forwards plain HTTP to backend services. Traffic inside the datacenter is unencrypted. Simpler, better performance (offloads crypto from app servers), but internal traffic is not encrypted. **End-to-end TLS**: LB re-encrypts and forwards HTTPS to backend. More secure (encrypted even inside datacenter), but requires cert management on all backend services. For sensitive data (payment, health), end-to-end is required. **mTLS** (Istio) provides end-to-end without per-service cert management.

---

## 🔗 What to Read Next

1. **[Security/ZeroTrust_Architecture.md](./ZeroTrust_Architecture.md)** — mTLS is the transport layer of Zero Trust; see the bigger picture
2. **[BuildingBlocks/APIGateway.md](../BuildingBlocks/APIGateway.md)** — TLS termination and certificate management at the API Gateway
3. **[Security/OWASP_Top10.md](./OWASP_Top10.md)** — A02 (Cryptographic Failures) covers what goes wrong when TLS is misconfigured

---

*[← Authentication vs Authorization](./Authentication_vs_Authorization.md) | [Back to Security](./README.md)*
