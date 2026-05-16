# 🛡️ Zero Trust Architecture
## Never Trust, Always Verify — The End of the Perimeter Security Model

> *"The castle-and-moat model of security assumes everyone inside the walls is safe. Zero Trust assumes the castle has already been breached. Every knight, every servant, every guest must prove who they are before they touch anything — every single time."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [TLS/SSL/HTTPS](./TLS_SSL_HTTPS.md), [Authentication vs Authorization](./Authentication_vs_Authorization.md)

---

## 📋 Table of Contents
1. [Why Perimeter Security Failed](#-why-perimeter-security-failed)
2. [Zero Trust Principles](#-zero-trust-principles)
3. [Zero Trust Architecture Components](#-zero-trust-architecture-components)
4. [Identity as the New Perimeter](#-identity-as-the-new-perimeter)
5. [mTLS for Service-to-Service Auth](#-mtls-for-service-to-service-auth)
6. [Network Microsegmentation](#-network-microsegmentation)
7. [Continuous Verification in Practice](#-continuous-verification-in-practice)
8. [Zero Trust with Kubernetes & Istio](#-zero-trust-with-kubernetes--istio)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 Why Perimeter Security Failed

The traditional security model: build a firewall, trust everything inside it.

```
Traditional "Castle-and-Moat":

 Outside (untrusted)    │  Inside (trusted)
 ─────────────────────  │  ─────────────────────────────
 Internet               │  Corporate network
 Attackers              │  All employees ← trusted implicitly
                        │  All servers   ← trusted implicitly
                        │  All services  ← trusted implicitly
        [FIREWALL] ─────┘
```

**What actually happens**:

```
1. Target Breach (2013): HVAC vendor had VPN access to the corporate network.
   Once inside, moved laterally to POS systems. 40 million credit cards stolen.

2. SolarWinds (2020): Malicious update to SolarWinds software, deployed by trusted
   software update mechanism inside thousands of corporate networks. 
   All "inside the firewall" = trusted = no additional checks.

3. Colonial Pipeline (2021): Single compromised VPN credential (no MFA).
   Attacker walked in through the front door.

The pattern:
  - Attacker compromises one trusted credential, device, or service
  - Traditional model: now fully trusted everywhere inside the network
  - Result: complete network access, lateral movement, data exfiltration
```

---

## 🏛️ Zero Trust Principles

NIST SP 800-207 defines Zero Trust Architecture on three core tenets:

```
1. VERIFY EXPLICITLY
   Authenticate and authorize every request, every time,
   based on ALL available data:
     - User identity + MFA
     - Device health (patched? managed? known?)
     - Location (normal? anomalous?)
     - Service identity (cert-based, not IP-based)
     - Data classification (what is being accessed?)

2. USE LEAST PRIVILEGE ACCESS
   Users and services get only the permissions they need,
   for only as long as they need them.
     - Time-limited tokens (hours, not months)
     - Just-In-Time (JIT) access for privileged operations
     - No standing permissions for admin access

3. ASSUME BREACH
   Design as if the attacker is already inside.
     - Encrypt all traffic (even internal)
     - Log everything for forensic analysis
     - Segment networks so breach doesn't = full access
     - Detect and respond to lateral movement
```

---

## 🏗️ Zero Trust Architecture Components

```
┌────────────────────────────────────────────────────────────────────┐
│                    ZERO TRUST ARCHITECTURE                        │
│                                                                    │
│  ┌─────────┐     ┌──────────────────┐     ┌─────────────────────┐│
│  │  User   │────▶│  Identity Provider│────▶│  Policy Engine      ││
│  │ + Device│     │  (IdP / OIDC)    │     │  (decision point)   ││
│  └─────────┘     └──────────────────┘     └──────────┬──────────┘│
│                                                       │            │
│       ┌───────────────────────────────────────────────▼──────────┐│
│       │              Policy Enforcement Point (PEP)              ││
│       │         (API Gateway / Service Mesh / Proxy)             ││
│       └───────────────────────────────────────────────┬──────────┘│
│                                                       │            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐      │            │
│  │ Service A  │  │ Service B  │  │ Database   │◀─────┘            │
│  │ (workload) │  │ (workload) │  │ (resource) │                   │
│  │   identity │  │   identity │  │  (access   │                   │
│  │    = cert  │  │    = cert  │  │  policy)   │                   │
│  └────────────┘  └────────────┘  └────────────┘                   │
│                                                                    │
│  ══════ All traffic encrypted (mTLS internally) ══════            │
│  ══════ All access decisions logged ══════════════════            │
└────────────────────────────────────────────────────────────────────┘

Key components:
  Identity Provider (IdP): Okta, Azure AD, Google Workspace — issues tokens
  Policy Engine: Decides "can user X on device Y access resource Z?"
  Policy Enforcement Point (PEP): Enforces the decision (API Gateway, Istio)
  Device Trust: MDM-managed, patched, compliant devices only
```

---

## 👤 Identity as the New Perimeter

In Zero Trust, **identity** replaces network location as the trust anchor.

```
Old model: IP 10.0.0.1 → trusted (internal network)
New model: Certificate CN=payment-service, signed by internal CA → trusted

Three types of identity to manage:

1. HUMAN IDENTITY (users)
   - Federated via OIDC/SAML (Okta, Azure AD)
   - Phishing-resistant MFA required (FIDO2/WebAuthn, not SMS)
   - Short-lived tokens (1h for access, 8h for refresh)
   - Device compliance check on every auth (OS version, disk encryption, EDR installed)

2. WORKLOAD IDENTITY (services/applications)
   - X.509 certificates issued by SPIFFE/SPIRE or Istio CA
   - Certificate identifies the service, not its IP
   - Short-lived certs (24h), automatically rotated
   - No passwords or long-lived API keys for service-to-service

3. DEVICE IDENTITY (machines)
   - Certificates or TPM-backed attestation
   - Device must be enrolled in MDM (Jamf, Intune) to get a cert
   - Device posture check before granting access
```

### SPIFFE/SPIRE — Workload Identity Standard

```
SPIFFE (Secure Production Identity Framework for Everyone):
  Defines a standard for workload identity as SPIFFE Verifiable Identity Document (SVID)
  Format: spiffe://trust-domain/path  (e.g., spiffe://mycompany.com/ns/prod/sa/order-service)

SPIRE (SPIFFE Runtime Environment):
  Issues SVID certificates to workloads automatically
  No human involved in cert issuance — fully automated
  Works across clouds, on-prem, Kubernetes, VMs

How it works:
  1. SPIRE Agent runs on each node, attests the workload (is this really order-service?)
  2. SPIRE Server issues a short-lived X.509 certificate with the SPIFFE ID
  3. Workload uses the cert for mTLS — no passwords anywhere
  4. Cert expires every 1-24 hours; SPIRE rotates automatically
```

---

## 🔐 mTLS for Service-to-Service Auth

In Zero Trust, services never use API keys or passwords to call each other. They use mTLS.

```
API key approach (NOT Zero Trust):
  order-service → payment-service: "I'm order-service. Here's my API key: sk_prod_xyz123"
  Problems:
    - Key can be leaked (in logs, env vars, git history)
    - Key doesn't expire automatically
    - Hard to rotate without downtime
    - Doesn't prove device identity

mTLS approach (Zero Trust):
  order-service → payment-service: TLS handshake with cert proving:
    "My identity is spiffe://company.com/order-service, 
     this cert was signed by the company's internal CA,
     and this cert expires in 6 hours."
  payment-service validates:
    1. Cert is signed by trusted internal CA ✅
    2. SPIFFE ID is on the allowlist ✅
    3. Cert hasn't expired ✅
  No secrets to leak. No keys to rotate manually.
```

---

## 🔒 Network Microsegmentation

Zero Trust requires restricting which services can talk to which — regardless of being "inside."

```
Without segmentation:
  If order-service is compromised, attacker can reach:
  → payment-service (steal card data!)
  → user-service (steal PII!)
  → admin-service (elevate privileges!)
  → database directly (exfiltrate all data!)

With microsegmentation (Kubernetes NetworkPolicy example):

# Only allow payment-service to receive traffic from order-service and audit-service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: order-service      # only from order-service
        - podSelector:
            matchLabels:
              app: audit-service      # and audit-service
      ports:
        - port: 8080
          protocol: TCP

# Default deny: block all other ingress to payment-service
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

---

## ✅ Continuous Verification in Practice

Zero Trust doesn't just authenticate once — it continuously re-evaluates trust.

```
User logs in (authentication):
  ✅ Valid credentials
  ✅ FIDO2 hardware key MFA passed
  ✅ Device is managed (Jamf enrolled, disk encrypted, OS up to date)
  ✅ Login from known location
  → Issued: short-lived token (1 hour)

30 minutes later, user accesses payment data:
  ✅ Token still valid
  ✅ Device still compliant (re-checked)
  ✅ Behavior normal (no anomalies)
  → Access granted

45 minutes later, user attempts to download 10GB of customer data:
  ❌ Anomalous behavior detected (unusual data volume)
  → Access blocked, session terminated, security team alerted

1 hour later, token expires:
  → User must re-authenticate
  → If device was compromised in the interim, new auth catches it
```

### Conditional Access Policies

```
Google BeyondCorp / Azure Conditional Access examples:

Policy: Allow access to financial data ONLY if:
  AND:
    user.role == "finance" OR "finance-admin"
    device.managed == true
    device.os_version >= "Windows 11 22H2"
    device.disk_encrypted == true
    device.endpoint_protection_active == true
    location.country IN ["US", "UK", "DE"]      // geofencing
    time.hour BETWEEN 7 AND 22                   // business hours only
    NOT risk.user_risk_level == "high"           // no anomalous behavior

If any condition fails → deny access OR require step-up MFA
```

---

## ☸️ Zero Trust with Kubernetes & Istio

Istio is the most popular service mesh implementing Zero Trust for Kubernetes workloads.

```
Istio Zero Trust features:
  1. mTLS everywhere (automatic, transparent to app code)
     Every pod gets a sidecar Envoy proxy
     All pod-to-pod traffic is mTLS using SPIFFE/SPIRE certs
     App developer writes HTTP code; Istio handles TLS

  2. Authorization Policies (who can call what)
     Based on SPIFFE IDs, not IPs

  3. Observability (who called what, when, how long)
     All traffic logged + traced automatically

  4. Certificate rotation
     Istio CA issues 24-hour certs, rotated automatically
```

```yaml
# Istio AuthorizationPolicy: Zero Trust service access control
# "Only order-service can call payment-service's /charge endpoint"
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-service-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            # Only order-service's SPIFFE identity can call us
            principals:
              - "cluster.local/ns/production/sa/order-service"
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/v1/charge"]
---
# Enforce mTLS for all traffic in namespace
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # reject any non-mTLS connection
```

---

## ⚠️ Common Pitfalls

1. **"We have a VPN, that's Zero Trust"** — VPN is the opposite of Zero Trust. VPN grants broad network access once connected. Zero Trust grants access to specific resources based on identity + device posture, with no inherent trust from network position.

2. **Implementing Zero Trust at the network layer only** — Network segmentation without workload identity is incomplete. An attacker who compromises a pod can still impersonate it if there's no certificate-based identity verification.

3. **Long-lived credentials** — API keys valid for 1 year, service account tokens that never expire — these are the antithesis of Zero Trust. Short-lived, automatically-rotated credentials are fundamental.

4. **Starting too big** — "We'll implement full Zero Trust in Q1" fails. Zero Trust is a journey. Start with the highest-risk surface: admin access (JIT privileged access), then service-to-service auth in the most sensitive services, then expand.

5. **Ignoring developer experience** — If Zero Trust makes developers' lives miserable (constant re-authentication, broken dev environments), they'll work around it. Transparent tools (Istio handles mTLS without app code changes) maintain security without friction.

---

## 🧩 Mini Challenge

**Scenario**: Your company's security team asks you to implement Zero Trust for the `payments` namespace in Kubernetes. Currently, all pods in the cluster can talk to each other freely (no NetworkPolicies). You have Istio installed.

**What are your first 4 steps, in order?**

<details>
<summary>💡 Click to reveal answer</summary>

**Step 1: Enable STRICT mTLS for the payments namespace**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: payments-strict-mtls
  namespace: payments
spec:
  mtls:
    mode: STRICT
```
This rejects any non-mTLS connection to any service in the namespace. Istio sidecars now mutually authenticate using SPIFFE certs. All traffic is encrypted.

**Step 2: Create a default-deny AuthorizationPolicy**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: payments
spec:
  {}  # empty spec = deny all
```
Now nothing can reach anything in the payments namespace. 

**Step 3: Create explicit allow rules for each required service communication**
```yaml
# Allow order-service to call payment-service
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-order-to-payment
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payment-service
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/orders/sa/order-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/v1/payments"]
```
Build the allowlist incrementally — test each service pair.

**Step 4: Add Kubernetes NetworkPolicy for defense in depth**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: payments
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```
Istio AuthorizationPolicy works at Layer 7; NetworkPolicy adds Layer 3/4 enforcement. Defense in depth: even if the Istio proxy is bypassed, the network blocks the connection.

</details>

---

## 📝 Interview Q&A

**Q: What is Zero Trust and how is it different from traditional perimeter security?**
> A: Traditional perimeter security ("castle-and-moat") trusts everything inside the corporate network by default — once past the firewall, users and services can access resources freely. Zero Trust replaces this with the principle "never trust, always verify": every request must authenticate and authorize regardless of network origin. The key shift is that identity (who you are + device posture) replaces network location (IP address) as the trust anchor. This matters because modern threats operate inside the network — through compromised credentials, insider threats, or lateral movement after initial breach.

**Q: How does mTLS contribute to Zero Trust?**
> A: In Zero Trust, services must prove their identity to each other — not just humans to services. mTLS (mutual TLS) enables this: both the client and server present X.509 certificates during the TLS handshake. Service identities are SPIFFE IDs embedded in certificates, issued by an internal CA (SPIRE or Istio CA), and automatically rotated every 1-24 hours. This eliminates static API keys and passwords from service-to-service communication — there are no secrets to leak or rotate manually.

**Q: What is the relationship between Zero Trust and a service mesh like Istio?**
> A: Istio is one of the most practical implementations of Zero Trust for microservices. It automatically injects Envoy sidecar proxies into every pod, which handle: (1) mTLS for all pod-to-pod communication — transparent to application code, (2) workload identity via SPIFFE certs issued and rotated by Istio's built-in CA, (3) authorization policies based on SPIFFE identities (not IPs) — "only order-service can call payment-service's /charge endpoint", (4) full observability — all traffic is logged and traced. The application developer writes plain HTTP code; the mesh enforces Zero Trust transparently.

---

## 🔗 What to Read Next

1. **[Security/Secrets_Management.md](./Secrets_Management.md)** — How to manage credentials and secrets in a Zero Trust world
2. **[Security/TLS_SSL_HTTPS.md](./TLS_SSL_HTTPS.md)** — Deep-dive into mTLS mechanics and certificate management
3. **[Security/Authentication_vs_Authorization.md](./Authentication_vs_Authorization.md)** — Identity and access management foundations for Zero Trust

---

*[← TLS/SSL/HTTPS](./TLS_SSL_HTTPS.md) | [Back to Security](./README.md)*
