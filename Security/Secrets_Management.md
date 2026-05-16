# 🔑 Secrets Management
## Vault, AWS Secrets Manager, and Never Storing Credentials in Code

> *"The most common cause of data breaches isn't sophisticated zero-day exploits — it's credentials committed to git repositories. In 2023, GitHub found over 10 million secrets exposed in public repos. The solution isn't telling developers to 'be careful.' It's making safe practices the default."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Zero Trust Architecture](./ZeroTrust_Architecture.md)

---

## 📋 Table of Contents
1. [The Secrets Problem](#-the-secrets-problem)
2. [What Counts as a Secret?](#-what-counts-as-a-secret)
3. [Anti-Patterns to Avoid](#-anti-patterns-to-avoid)
4. [HashiCorp Vault](#-hashicorp-vault)
5. [AWS Secrets Manager](#-aws-secrets-manager)
6. [Kubernetes Secrets (and their limits)](#-kubernetes-secrets-and-their-limits)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [Secret Rotation](#-secret-rotation)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Secrets Problem

```
The classic secret leak scenario:

1. Developer adds DB password to application.properties for "quick testing"
   spring.datasource.password=MyPr0dPassword!

2. Commits it to git: git commit -m "fix: connect to DB"

3. Pushes to GitHub

4. Realizes the mistake, deletes the file... but git history never forgets

5. Attacker running a github-secrets-scanner finds it within minutes

6. Attacker now has production DB credentials — game over

This happened to:
  - Uber (2016): AWS credentials in private GitHub repo → 57M user records
  - Toyota (2023): GitHub leaked API key exposed 300,000 customer records for 5 years
  - Twitch (2021): Source code + credentials leaked → payment info exposed
```

---

## 📋 What Counts as a Secret?

```
Secrets (must NEVER be in code, config files, or environment variables):
  ✅ Database passwords
  ✅ API keys (Stripe, Twilio, SendGrid, etc.)
  ✅ OAuth2 client secrets
  ✅ JWT signing keys / private keys
  ✅ TLS private keys and certificates
  ✅ SSH private keys
  ✅ Encryption keys (AES, RSA private)
  ✅ Service account credentials

NOT secrets (fine in code/config):
  ℹ️ Public keys / certificates
  ℹ️ Non-sensitive configuration (port numbers, feature flags)
  ℹ️ Database hostnames/usernames (only the password is a secret)
  ℹ️ OAuth2 client IDs (but NOT client secrets)
```

---

## 🚫 Anti-Patterns to Avoid

### 1. Secrets in Source Code (Worst)

```java
// NEVER DO THIS
private static final String API_KEY = "sk_live_abc123def456";
DataSource ds = new DataSource("jdbc:mysql://prod.db", "admin", "Sup3rSecr3t!");
```

### 2. Secrets in application.properties/yaml

```yaml
# NEVER commit this
spring:
  datasource:
    password: MyProductionPassword123
aws:
  access-key: AKIAIOSFODNN7EXAMPLE
  secret-key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

### 3. Secrets in Environment Variables (Better, but insufficient alone)

```bash
# Better than code, but:
# - Visible via /proc/PID/environ
# - Visible in process listings
# - Inherited by child processes
# - Often logged accidentally
export DB_PASSWORD="secret"
java -jar app.jar  # DB_PASSWORD is in process environment
```

### 4. Secrets in CI/CD Logs

```yaml
# BAD: this logs the secret
- name: Deploy
  run: echo "Connecting to $DB_PASSWORD"  # prints secret in logs!
  
# Also BAD: curl with auth header visible in CI logs
- run: curl -H "Authorization: Bearer $TOKEN" https://api.example.com
```

---

## 🏛️ HashiCorp Vault

Vault is the gold-standard open-source secrets management solution.

```
Vault architecture:

  [Application] ─auth─▶ [Vault] ─stores─▶ [Backend Storage]
                                            (Consul, S3, PostgreSQL)
       ↕
  [Vault Agent]  ← renders secrets to files/env vars
  (sidecar/init)    without app knowing Vault exists

Key features:
  Dynamic secrets:   Vault generates a NEW database credential for each request
                     Credential auto-expires in 1 hour — no long-lived passwords
  Secret rotation:   Rotate credentials without app downtime
  Audit logging:     Every secret access is logged — who accessed what, when
  Leasing:           Secrets have TTL, automatically revoked when expired
  Multiple auth:     Kubernetes SA, AWS IAM, LDAP, AppRole, TLS certs
```

### Vault Dynamic Secrets — The Key Insight

```
Static secrets (traditional):
  All service instances share ONE database password.
  Password leaked → all instances compromised.
  Rotation requires updating all instances simultaneously (risky).

Dynamic secrets (Vault):
  Each app instance requests a temporary credential from Vault:
    order-service-pod-1: receives postgres user "v-order-abc123" valid 1h
    order-service-pod-2: receives postgres user "v-order-def456" valid 1h
    order-service-pod-3: receives postgres user "v-order-ghi789" valid 1h
  
  Leak of one credential:
    - Only affects one pod
    - Expires within 1 hour automatically
    - Can be revoked immediately
  
  "Rotation": credentials naturally expire; no coordinated rollout needed.
```

### Vault Configuration Example

```bash
# Enable PostgreSQL dynamic secrets
vault secrets enable database

# Configure the DB connection (Vault connects with a superuser)
vault write database/config/orders-db \
    plugin_name=postgresql-database-plugin \
    allowed_roles="order-service-role" \
    connection_url="postgresql://{{username}}:{{password}}@postgres:5432/orders" \
    username="vault_superuser" \
    password="vault_superpassword"

# Create a role: Vault will CREATE this SQL role on demand
vault write database/roles/order-service-role \
    db_name=orders-db \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
                         GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"

# App requests a credential:
vault read database/creds/order-service-role
# Key               Value
# lease_id          database/creds/order-service-role/abc123
# lease_duration    1h
# password          A1a-2Xy3Bz...  (generated, unique)
# username          v-order-service-abc123
```

---

## ☁️ AWS Secrets Manager

AWS-native solution, tightly integrated with AWS services.

```
Comparison: Vault vs AWS Secrets Manager

Feature                  │ HashiCorp Vault     │ AWS Secrets Manager
─────────────────────────┼─────────────────────┼────────────────────
Dynamic secrets          │ ✅ Yes               │ ⚠️ Partial (rotation only)
Multi-cloud              │ ✅ Yes               │ ❌ AWS-only
Open source              │ ✅ Yes               │ ❌ Proprietary
Kubernetes integration   │ ✅ Native (CSI)      │ ✅ AWS Secrets CSI driver
Auto-rotation            │ ✅ Yes               │ ✅ Yes (Lambda-based)
Cost                     │ Infrastructure cost  │ $0.40/secret/month + API calls
Audit logging            │ ✅ Built-in          │ ✅ CloudTrail
Fine-grained access      │ ✅ Vault policies     │ ✅ IAM policies
Setup complexity         │ 🔴 High              │ 🟢 Low (managed service)

Choose AWS Secrets Manager if: you're AWS-only and want low operational overhead
Choose Vault if: multi-cloud, need dynamic secrets, or want more control
```

```java
// Fetching from AWS Secrets Manager in Spring Boot
@Configuration
public class SecretsConfig {

    @Bean
    public DataSource dataSource() {
        // AWS SDK fetches from Secrets Manager (cached by SDK)
        SecretsManagerClient client = SecretsManagerClient.builder()
                .region(Region.US_EAST_1)
                .build();

        GetSecretValueResponse response = client.getSecretValue(
                GetSecretValueRequest.builder()
                        .secretId("prod/order-service/db")
                        .build()
        );

        // Secret stored as JSON: {"username":"admin","password":"..."}
        ObjectMapper mapper = new ObjectMapper();
        JsonNode secret = mapper.readTree(response.secretString());

        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://orders.prod.db:5432/orders");
        config.setUsername(secret.get("username").asText());
        config.setPassword(secret.get("password").asText());
        return new HikariDataSource(config);
    }
}
```

---

## ☸️ Kubernetes Secrets (and their limits)

```
Kubernetes Secret (basic):

apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: TXlQcm9kUGFzc3dvcmQ=  # base64 encoded (NOT encrypted!)

Problems with plain Kubernetes Secrets:
  ❌ Stored as base64 in etcd — base64 is encoding, NOT encryption
  ❌ Any pod in the namespace can mount the secret
  ❌ Visible to anyone with kubectl get secret access
  ❌ No automatic rotation
  ❌ No audit trail of who accessed what

Better options:
  ✅ Sealed Secrets (Bitnami): encrypt secrets so only the cluster can decrypt
  ✅ External Secrets Operator: sync from Vault/AWS Secrets Manager into k8s secrets
  ✅ Vault Agent Injector: inject secrets directly as files into pods (never in k8s secrets)
  ✅ CSI Secret Store Driver: mount secrets as volume from external providers
```

### External Secrets Operator (Best Practice)

```yaml
# External Secrets Operator: sync secret from AWS Secrets Manager to k8s
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  refreshInterval: 1h      # re-sync from source every hour
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  target:
    name: db-credentials    # creates/updates this k8s secret
  data:
    - secretKey: password   # k8s secret key name
      remoteRef:
        key: prod/order-service/db   # AWS Secrets Manager path
        property: password
```

---

## ⚙️ Spring Boot Integration

### Option 1: Spring Cloud Vault

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yaml (loaded before application.yaml)
spring:
  cloud:
    vault:
      host: vault.internal.company.com
      port: 8200
      scheme: https
      authentication: KUBERNETES           # use k8s service account token
      kubernetes:
        role: order-service
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
      kv:
        enabled: true
        default-context: order-service     # reads secrets/order-service/*
        backend: secret
```

```yaml
# Vault stores:  secret/order-service/database.password = "MySecretPass"
# Spring Boot maps this to property: database.password
# Used as: @Value("${database.password}") String dbPassword;
```

### Option 2: Environment Variable Injection via Vault Agent

```yaml
# Vault Agent injects secrets as environment variables before app starts
# pod annotation triggers injection:
vault.hashicorp.com/agent-inject: "true"
vault.hashicorp.com/agent-inject-secret-db: "database/creds/order-service-role"
vault.hashicorp.com/agent-inject-template-db: |
  {{- with secret "database/creds/order-service-role" -}}
  export DB_PASSWORD="{{ .Data.password }}"
  export DB_USERNAME="{{ .Data.username }}"
  {{- end }}
vault.hashicorp.com/role: "order-service"
```

---

## 🔄 Secret Rotation

```
Rotation strategies:

1. MANUAL ROTATION (worst, but common)
   - Someone updates the secret in Vault/Secrets Manager manually
   - Requires coordinating app restart
   - Often done annually (or never)

2. AUTOMATIC ROTATION WITHOUT DOWNTIME
   - Critical for zero-downtime production systems

   Algorithm for zero-downtime rotation:
   
   Phase 1: ADD new credential alongside old
     DB user "app_v2" created with same permissions as "app_v1"
     Both credentials are valid simultaneously
   
   Phase 2: DEPLOY new version using new credential
     Rolling deployment: some pods use v1, some use v2
     Both work against the database simultaneously
   
   Phase 3: REMOVE old credential
     All pods now use v2
     Revoke "app_v1"
   
   AWS Secrets Manager does this automatically with the Lambda rotation function.
   Vault dynamic secrets make it unnecessary (each credential is temporary anyway).

3. BREAK-GLASS PROCEDURES
   Emergency: credential compromised
   → Immediately revoke ALL credentials for that secret path
   → All service instances will fail to authenticate
   → This is intentional — better than leaving compromised creds active
   → Recovery: on-call engineer re-issues credentials via emergency procedure
```

---

## ⚠️ Common Pitfalls

1. **Using base64 "encoding" and calling it encryption** — `echo -n "password" | base64` is trivially reversible. Kubernetes Secrets are base64-encoded by default, not encrypted. Enable [etcd encryption at rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/) and use external secret stores for real protection.

2. **Overly permissive Vault policies** — `path "*" { capabilities = ["read"] }` lets any authenticated service read any secret. Use the principle of least privilege: `path "secret/data/order-service/*"` allows order-service to read only its own secrets.

3. **Long-lived dynamic secret leases** — Vault dynamic secrets have a TTL. Setting `max_ttl=87600h` (10 years) defeats the purpose. Keep it short: 1h for DB creds, 8h for API tokens.

4. **Not auditing secret access** — Vault has a built-in audit device. Not enabling it means you can't answer "who accessed the payment service DB credentials last Tuesday?" Enable audit logging and ship to your SIEM.

5. **Secrets in Docker image layers** — `COPY config.yaml /app/` where config.yaml contains a secret bakes it into the image. Even if you delete it in a later layer, `docker history` reveals it. Secrets must be injected at runtime, never baked into images.

---

## 🧩 Mini Challenge

**A developer says**: "We use environment variables for all secrets in our Docker containers. That's secure, right? The env vars are only in the container's environment, not in code."

**What are the security risks, and what's the better approach?**

<details>
<summary>💡 Click to reveal answer</summary>

**Risks with environment variables for secrets**:

1. **Container inspection**: `docker inspect <container>` lists all environment variables including secrets — accessible to anyone with Docker daemon access on the host.

2. **Process listing**: On Linux, `/proc/<PID>/environ` exposes the process's environment to anyone with read access to that proc file.

3. **Child process inheritance**: All child processes spawned by the app inherit environment variables — including frameworks, subprocesses, and eval() calls.

4. **Accidental logging**: Spring Boot's Actuator `/env` endpoint (if misconfigured) exposes all environment variables. Libraries and frameworks sometimes log env vars during startup or error handling.

5. **Docker Compose files**: `docker-compose.yml` with `environment: DB_PASSWORD=secret` gets committed to git. Sharing compose files = sharing secrets.

6. **12-factor apps**: Environment variables are fine for _non-sensitive_ config. But secrets need a higher standard.

**Better approach**:

```
1. Use a secrets manager (Vault, AWS Secrets Manager)
2. Inject secrets as files (not env vars) mounted at runtime:
   - Vault Agent Sidecar injects to /vault/secrets/db-credentials
   - App reads file, not env var
   - File permissions: 0400 (readable only by app process)
   - Secret never in process environment

3. If env vars are unavoidable: use a secret-aware orchestrator
   - Kubernetes Secrets mounted as files (not env vars)
   - External Secrets Operator syncs from Vault to k8s secrets
   - Ensure /proc/<PID>/environ is not accessible (Linux namespace isolation)

4. Scan for exposed secrets: use tools like:
   - truffleHog, gitleaks (scan git history)
   - detect-secrets (pre-commit hook)
   - GitHub secret scanning (automatic on push)
```

</details>

---

## 📝 Interview Q&A

**Q: Where should secrets be stored in a cloud-native application?**
> A: Secrets should be stored in a dedicated secrets management system — HashiCorp Vault for multi-cloud/complex environments, AWS Secrets Manager for AWS-native applications, or Azure Key Vault / GCP Secret Manager for their respective clouds. Never in: source code, application config files committed to git, Docker images, or plain environment variables. At runtime, secrets should be injected as files (not env vars) by a sidecar agent or CSI driver, with short TTLs and automatic rotation. Every access to a secret should be logged for audit purposes.

**Q: What are dynamic secrets in HashiCorp Vault and why are they better than static credentials?**
> A: Dynamic secrets are short-lived credentials generated on-demand by Vault, rather than long-lived static passwords. When order-service needs to connect to PostgreSQL, it requests a credential from Vault, which creates a new PostgreSQL user (e.g., `v-order-abc123`) with a 1-hour TTL. The credential automatically expires and is revoked after the TTL. Benefits: no long-lived passwords to manage or rotate; if a credential leaks, it expires quickly; each service instance gets unique credentials so a single leak doesn't compromise all instances; no coordination required to rotate credentials.

**Q: How do you avoid secrets in Kubernetes while still giving pods access to them?**
> A: Three approaches: (1) **External Secrets Operator**: sync secrets from Vault/AWS Secrets Manager into Kubernetes Secrets automatically, with automatic refresh. (2) **Vault Agent Sidecar**: an init container fetches secrets from Vault and writes them as files to a shared volume; the main container reads files, never storing secrets in k8s. (3) **CSI Secret Store Driver**: mounts secrets directly as a volume from external providers without creating k8s Secret objects. All three avoid secrets being stored in etcd (the k8s control plane), which even with encryption-at-rest is a higher-value attack target than a proper secrets manager.

---

## 🔗 What to Read Next

1. **[Security/ZeroTrust_Architecture.md](./ZeroTrust_Architecture.md)** — Secrets management is the credential layer of Zero Trust
2. **[Security/OWASP_Top10.md](./OWASP_Top10.md)** — A02 (Cryptographic Failures) and A07 (Identification Failures) cover what goes wrong
3. **[DevOps/](../DevOps/)** — CI/CD secret injection patterns (GitHub Actions secrets, sealed secrets in GitOps)

---

*[← Zero Trust Architecture](./ZeroTrust_Architecture.md) | [Back to Security](./README.md)*
