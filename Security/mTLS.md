# 🔐 mTLS: Mutual TLS for Service-to-Service Authentication

> *"In a zero-trust microservice mesh, every service must prove its identity to every other service on every request. Regular TLS only authenticates the server — mTLS authenticates BOTH sides, eliminating an entire class of impersonation attacks inside your network."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [TLS/SSL/HTTPS](./TLS_SSL_HTTPS.md), [Zero Trust Architecture](./ZeroTrust_Architecture.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [How mTLS Works](#-how-mtls-works)
3. [mTLS vs Regular TLS](#-mtls-vs-regular-tls)
4. [Certificate Management](#-certificate-management)
5. [Implementation](#-implementation)
6. [Service Mesh Integration](#-service-mesh-integration)
7. [Common Pitfalls](#-common-pitfalls)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
Traditional microservice communication (inside the network):

  [Order Service] ──HTTP (plain)──▶ [Payment Service]

  Problems:
    1. No encryption: anyone on the network can sniff payment data
    2. No authentication: any service can call payment-service (impersonation)
    3. No integrity: requests can be modified in transit (MITM)
    4. "Trust the network" assumption is BROKEN in cloud/k8s environments
       (shared infrastructure, lateral movement after breach)

  Attack scenario:
    Attacker compromises a low-privilege service (logging-service)
    → Makes HTTP call to payment-service pretending to be order-service
    → Payment service has no way to verify the caller's identity
    → Unauthorized payment processed!

With mTLS:
  [Order Service] ◄══mTLS══▶ [Payment Service]
     cert: order-svc.pem       cert: payment-svc.pem
     
  Both sides verify each other's certificate:
    ✅ Payment verifies Order's cert → "yes, this is really order-service"
    ✅ Order verifies Payment's cert → "yes, this is really payment-service"
    ✅ All traffic encrypted (even inside the cluster)
    ❌ Compromised logging-service has wrong cert → connection REJECTED
```

---

## 🏗️ How mTLS Works

```
REGULAR TLS (one-way authentication):
  Client ────────────────────────────────────▶ Server
    1. Client says "hello"
    2. Server sends its certificate (server proves identity)
    3. Client verifies server cert against trusted CA
    4. Encrypted channel established
    5. Client is ANONYMOUS (server doesn't know who's calling)

mTLS (mutual authentication — both sides prove identity):
  Client (Service A) ◄──────────────────────▶ Server (Service B)
    1. Client says "hello"
    2. Server sends its certificate
    3. Client verifies server cert ✅
    4. Server requests client's certificate    ← EXTRA STEP
    5. Client sends its certificate            ← EXTRA STEP
    6. Server verifies client cert ✅           ← EXTRA STEP
    7. Encrypted channel established
    8. BOTH sides know exactly who the other is

  Certificate contents (X.509):
    Subject: CN=order-service, O=mycompany
    Issuer: CN=internal-ca.mycompany.com
    Valid: 2024-01-01 to 2024-01-02 (short-lived!)
    Public Key: RSA 2048 / ECDSA P-256
    SAN (Subject Alternative Names): order-service.default.svc.cluster.local
```

---

## ⚖️ mTLS vs Regular TLS

| Aspect | Regular TLS | mTLS |
|---|---|---|
| **Server authenticated** | ✅ Yes | ✅ Yes |
| **Client authenticated** | ❌ No (anonymous) | ✅ Yes (certificate) |
| **Use case** | Browser → web server | Service → service |
| **Who has certs** | Only server | Both client AND server |
| **Identity** | Server only proves "I am api.example.com" | Both prove "I am order-service" / "I am payment-service" |
| **Authorization** | Separate (JWT, API keys) | Can derive from cert (CN/SAN = service identity) |
| **Complexity** | 🟢 Simple (server cert only) | 🟡 Medium (cert distribution, rotation, CA management) |
| **Performance** | 1 cert verification | 2 cert verifications (slightly more CPU on handshake) |

---

## 🏢 Certificate Management

```
CERTIFICATE AUTHORITY (CA) HIERARCHY:

  [Root CA] (offline, air-gapped, 10-year validity)
       │
       ├──▶ [Intermediate CA: production] (3-year validity)
       │         │
       │         ├──▶ order-service cert (24-hour validity)
       │         ├──▶ payment-service cert (24-hour validity)
       │         └──▶ user-service cert (24-hour validity)
       │
       └──▶ [Intermediate CA: staging] (3-year validity)
                  │
                  └──▶ staging service certs...

SHORT-LIVED CERTIFICATES (best practice):
  - Certificate lifetime: 1-24 hours (not years!)
  - Auto-rotated before expiry (no manual intervention)
  - If cert is stolen, it expires before attacker can use it
  - No need for CRL/OCSP (revocation is "wait for expiry")

CERTIFICATE ISSUANCE FLOW (automated):
  1. Service starts → generates keypair
  2. Service sends CSR (Certificate Signing Request) to CA
  3. CA verifies identity (K8s: check ServiceAccount, pod identity)
  4. CA issues short-lived cert (24h)
  5. Service uses cert for mTLS connections
  6. Before expiry → service automatically requests new cert (rotation)

Tools:
  - Istio Citadel / cert-manager: automatic cert issuance in Kubernetes
  - HashiCorp Vault PKI: enterprise certificate management
  - SPIFFE/SPIRE: universal identity framework for workloads
```

---

## 💻 Implementation

### Spring Boot mTLS Configuration

```java
// application.yml — Server side (accepting mTLS connections)
server:
  ssl:
    enabled: true
    key-store: classpath:server-keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    # mTLS: require client certificate
    client-auth: need  # "need" = mandatory, "want" = optional
    trust-store: classpath:truststore.p12
    trust-store-password: ${SSL_TRUSTSTORE_PASSWORD}

// application.yml — Client side (calling another service with mTLS)
spring:
  ssl:
    bundle:
      jks:
        payment-service:
          key:
            alias: client-cert
          keystore:
            location: classpath:client-keystore.p12
            password: ${CLIENT_KEYSTORE_PASSWORD}
          truststore:
            location: classpath:truststore.p12
            password: ${TRUSTSTORE_PASSWORD}
```

```java
// RestTemplate configured with mTLS
@Configuration
public class MtlsRestTemplateConfig {

    @Bean
    public RestTemplate mtlsRestTemplate() throws Exception {
        SSLContext sslContext = SSLContextBuilder.create()
            .loadKeyMaterial(
                new ClassPathResource("client-keystore.p12").getURL(),
                keystorePassword.toCharArray(),
                keyPassword.toCharArray()
            )
            .loadTrustMaterial(
                new ClassPathResource("truststore.p12").getURL(),
                truststorePassword.toCharArray()
            )
            .build();

        HttpClient httpClient = HttpClients.custom()
            .setSSLContext(sslContext)
            .build();

        return new RestTemplateBuilder()
            .requestFactory(() -> new HttpComponentsClientHttpRequestFactory(httpClient))
            .build();
    }
}

// Extract client identity from certificate in a filter
@Component
public class MtlsIdentityFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
            HttpServletResponse response, FilterChain chain) {
        X509Certificate[] certs = (X509Certificate[]) 
            request.getAttribute("javax.servlet.request.X509Certificate");
        
        if (certs != null && certs.length > 0) {
            String clientIdentity = certs[0].getSubjectX500Principal().getName();
            // CN=order-service,O=mycompany
            // Use this for authorization decisions
            request.setAttribute("caller-service", extractCN(clientIdentity));
        }
        chain.doFilter(request, response);
    }
}
```

---

## 🌐 Service Mesh Integration

```
ISTIO mTLS (zero application code changes):

  Without service mesh:
    Each service must implement mTLS in application code (complex!)

  With Istio sidecar proxy:
    [Order Service] → [Envoy Proxy] ══mTLS══▶ [Envoy Proxy] → [Payment Service]
                      (sidecar)                (sidecar)
    
    - Envoy handles all mTLS negotiation
    - Application code sees plain HTTP (localhost)
    - Certificates auto-issued and rotated by Istio Citadel
    - Zero code changes in your Java/Node/Go services!

  Istio PeerAuthentication policy:
    apiVersion: security.istio.io/v1beta1
    kind: PeerAuthentication
    metadata:
      name: default
      namespace: production
    spec:
      mtls:
        mode: STRICT  # all traffic must be mTLS (no plaintext allowed)

  Istio AuthorizationPolicy (who can call whom):
    apiVersion: security.istio.io/v1beta1
    kind: AuthorizationPolicy
    metadata:
      name: payment-service-policy
    spec:
      selector:
        matchLabels:
          app: payment-service
      rules:
      - from:
        - source:
            principals: ["cluster.local/ns/default/sa/order-service"]
        # Only order-service can call payment-service
        # All other services → 403 Forbidden
```

---

## ⚠️ Common Pitfalls

1. **Long-lived certificates** — Using certificates valid for 1-5 years negates security benefits. If a cert is compromised, the attacker has years of access. Use short-lived certs (hours) with automated rotation. If you can't do short-lived certs, implement proper CRL/OCSP checking.

2. **Trusting any valid certificate** — A valid certificate only means "this cert was signed by our CA." You must ALSO check the identity in the cert (CN/SAN) matches the expected caller. Without authorization rules, any internal service with a valid cert can call any other service.

3. **Storing private keys insecurely** — Private keys in environment variables, config files, or container images are easily extracted. Use: K8s secrets (encrypted at rest), HashiCorp Vault, cloud KMS, or tmpfs mounts that never touch disk.

4. **Certificate rotation downtime** — If the old cert expires before the new one is loaded, connections fail. Implement graceful rotation: load new cert while old is still valid, drain connections, then retire old cert. Service meshes handle this automatically.

5. **Skipping mTLS for "internal" traffic** — The assumption that traffic inside a VPC/cluster is safe is the #1 reason lateral movement succeeds after a breach. Apply mTLS universally — even between pods on the same node.

---

## 🧩 Mini Challenge

**Your company has 200 microservices. You want to enforce that only `order-service` and `admin-service` can call `payment-service`. How would you implement this authorization on top of mTLS?**

<details>
<summary>💡 Click to reveal answer</summary>

**Layer 1: mTLS (authentication — "who are you?")**
- All services have certificates issued by the internal CA
- Certificate CN/SAN contains service identity (e.g., `order-service.default.svc.cluster.local`)
- mTLS ensures the caller IS who they claim to be

**Layer 2: Authorization policy (access control — "are you allowed?")**

Option A: Service mesh policy (Istio/Linkerd):
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-allow-list
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
  - from:
    - source:
        principals:
          - "cluster.local/ns/default/sa/order-service"
          - "cluster.local/ns/default/sa/admin-service"
  # Implicit deny: any other service → 403
```

Option B: Application-level (without service mesh):
```java
@Component
public class ServiceAuthFilter extends OncePerRequestFilter {
    private static final Set<String> ALLOWED_CALLERS = Set.of(
        "order-service", "admin-service"
    );
    
    @Override
    protected void doFilterInternal(...) {
        String callerCN = extractCNFromClientCert(request);
        if (!ALLOWED_CALLERS.contains(callerCN)) {
            response.sendError(403, "Service not authorized");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

**Best practice**: Use service mesh policies (enforced at the proxy layer, outside application code) + back it up with application-level checks for defense in depth.

</details>

---

## 📝 Interview Q&A

**Q: Why use mTLS instead of API keys for service-to-service auth?**
> A: mTLS provides three guarantees that API keys don't: (1) **Encryption** — all traffic is encrypted in transit (API keys over plain HTTP can be sniffed). (2) **Mutual authentication** — both sides prove identity cryptographically (API keys only authenticate the client, and can be stolen/shared). (3) **Non-repudiation** — the TLS handshake proves the caller possesses the private key at this moment (API keys are static secrets that can be leaked months ago). Additionally, short-lived certificates (hours) are far safer than long-lived API keys — even if stolen, they expire quickly.

**Q: How does certificate rotation work without downtime?**
> A: Graceful rotation follows this sequence: (1) Generate new certificate while old is still valid (overlap period). (2) Load new cert into the service's keystore (hot reload without restart). (3) New connections use the new cert; existing connections continue on the old cert until they naturally close. (4) After all old connections drain (or timeout), the old cert is removed. Service meshes like Istio automate this entirely — Citadel issues new certs 10 minutes before expiry and hot-swaps them in the Envoy sidecar with zero application awareness.

---

## 🔗 What to Read Next

1. **[Security/TLS_SSL_HTTPS.md](./TLS_SSL_HTTPS.md)** — Foundation: understand TLS before adding the "mutual" part
2. **[Security/ZeroTrust_Architecture.md](./ZeroTrust_Architecture.md)** — mTLS is a core building block of zero-trust networks
3. **[Security/SecurityInMicroservices.md](./SecurityInMicroservices.md)** — Comprehensive security patterns for microservice architectures

---

*[← Secrets Management](./Secrets_Management.md) | [Back to Security](../INDEX.md) | [Next: API Security →](./API_Security_Best_Practices.md)*
