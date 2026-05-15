# **Tutorial 13: DevSecOps Concepts** 🔒

**Master Security in DevOps Before Security Tools**

---

## **📋 Table of Contents**

1. [The Security Breach](#1-the-security-breach)
2. [What is DevSecOps?](#2-what-is-devsecops)
3. [Shift-Left Security](#3-shift-left-security)
4. [Security in CI/CD Pipeline](#4-security-in-cicd-pipeline)
5. [Secrets Management](#5-secrets-management)
6. [Container Security](#6-container-security)
7. [Vulnerability Management](#7-vulnerability-management)
8. [Interview Q&A](#8-interview-qa)
9. [Challenges](#9-challenges)

---

## **1. The Security Breach**

```
Friday 4:00 PM - Security Alert

Security Team: "We detected API keys in your GitHub repo!"
You: "What? Where?"
Security: "Committed 3 months ago in application.yml"
You: "Oh no... is it in public repo?"
Security: "Yes. 50,000 downloads."

Immediate Actions:
  ❌ API keys compromised (AWS, Stripe, Database)
  ❌ Attackers already using them (since 3 months)
  ❌ $50,000 in unauthorized AWS charges
  ❌ Customer data potentially accessed
  ❌ Company reputation damaged

Post-Incident:
  - All keys rotated
  - Security audit ($100K)
  - Customer notifications
  - Regulatory fines ($500K)
  
Total Cost: $650K + reputation damage

Root Cause: No security in development process
```

**Without DevSecOps:**
- Security as afterthought
- Vulnerabilities in production
- Secrets in code
- No automated scanning

---

## **2. What is DevSecOps?**

### **Traditional Security vs DevSecOps**

```
Traditional (Security Afterward):
  Dev → Code → Test → Security Review → Prod
                         ↑
                    Bottleneck (weeks)
                    Found issues late
                    Expensive to fix

DevSecOps (Security Throughout):
  Dev (Sec) → Build (Sec) → Test (Sec) → Deploy (Sec)
       ↓          ↓            ↓            ↓
    IDE       SAST/SCA    Pen Testing   Runtime
   Security   Scans       DAST Scans    Protection
```

### **Core Principles**

```
1. Shift-Left
   Catch security issues early (IDE, commit time)
   
2. Automation
   Security scans in CI/CD pipeline
   
3. Continuous Monitoring
   Runtime security, threat detection
   
4. Shared Responsibility
   Security is everyone's job, not just security team
```

---

## **3. Shift-Left Security**

### **Security at Every Stage**

```
Developer's Machine:
  ├─ IDE plugins (detect secrets, vulnerable code)
  ├─ Pre-commit hooks (block secrets)
  └─ Local scanning

Commit Time:
  ├─ Secret scanning
  ├─ Code quality checks
  └─ Dependency checks

Build Time:
  ├─ SAST (Static Analysis)
  ├─ SCA (Software Composition Analysis)
  └─ Container scanning

Test Time:
  ├─ DAST (Dynamic Analysis)
  ├─ Penetration testing
  └─ Security test suites

Deployment:
  ├─ Infrastructure scanning
  ├─ Configuration validation
  └─ Compliance checks

Runtime:
  ├─ WAF (Web Application Firewall)
  ├─ Runtime protection
  └─ Threat monitoring
```

### **Pre-Commit Hooks**

```bash
# .git/hooks/pre-commit

#!/bin/bash

# Check for secrets
echo "Checking for secrets..."
git diff --cached | grep -E "(api_key|password|secret|token)" && {
    echo "❌ Potential secret detected!"
    echo "Please remove secrets before committing"
    exit 1
}

# Check for AWS keys
git diff --cached | grep -E "AKIA[0-9A-Z]{16}" && {
    echo "❌ AWS Access Key detected!"
    exit 1
}

echo "✅ Pre-commit checks passed"
exit 0
```

---

## **4. Security in CI/CD Pipeline**

### **Pipeline Security Gates**

```yaml
# .github/workflows/security.yml
name: Security Pipeline

on: [push, pull_request]

jobs:
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: TruffleHog Secret Scan
        run: |
          docker run --rm -v "$PWD:/repo" \
            trufflesecurity/trufflehog:latest \
            filesystem /repo
  
  sast-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: SonarQube Scan
        run: |
          mvn sonar:sonar \
            -Dsonar.projectKey=payment-service \
            -Dsonar.host.url=$SONAR_URL \
            -Dsonar.login=$SONAR_TOKEN
      - name: Fail on Critical Issues
        run: |
          # Fail if critical vulnerabilities found
          RATING=$(curl "$SONAR_URL/api/qualitygates/project_status?projectKey=payment-service")
          echo $RATING | grep "ERROR" && exit 1
  
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: OWASP Dependency Check
        run: |
          mvn dependency-check:check
      - name: Upload Report
        uses: actions/upload-artifact@v3
        with:
          name: dependency-check-report
          path: target/dependency-check-report.html
  
  container-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Build Image
        run: docker build -t payment-service:${{ github.sha }} .
      - name: Trivy Scan
        run: |
          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --severity HIGH,CRITICAL \
            --exit-code 1 \
            payment-service:${{ github.sha }}
      - name: Push if Scan Passes
        if: success()
        run: docker push payment-service:${{ github.sha }}
```

### **Security Scan Types**

```
SAST (Static Application Security Testing):
  - Analyzes source code without running it
  - Finds: SQL injection, XSS, hardcoded secrets
  - Tools: SonarQube, Checkmarx, Fortify
  - Speed: Fast (minutes)
  
SCA (Software Composition Analysis):
  - Scans dependencies for known vulnerabilities
  - Checks: CVE databases, license compliance
  - Tools: OWASP Dependency Check, Snyk
  - Speed: Fast (minutes)
  
DAST (Dynamic Application Security Testing):
  - Tests running application
  - Finds: Runtime vulnerabilities, config issues
  - Tools: OWASP ZAP, Burp Suite
  - Speed: Slow (hours)
```

---

## **5. Secrets Management**

### **Never Hardcode Secrets**

```java
// ❌ BAD: Hardcoded secret
@RestController
public class PaymentController {
    
    private static final String API_KEY = "sk_live_abc123xyz"; // NEVER!
    
    public PaymentResponse process(PaymentRequest req) {
        stripe.setApiKey(API_KEY);
        return stripe.charge(req);
    }
}

// ✅ GOOD: Environment variable
@RestController
public class PaymentController {
    
    @Value("${stripe.api.key}")
    private String apiKey;  // From environment
    
    public PaymentResponse process(PaymentRequest req) {
        stripe.setApiKey(apiKey);
        return stripe.charge(req);
    }
}

// ✅ BETTER: Secret management service
@RestController
public class PaymentController {
    
    @Autowired
    private VaultTemplate vaultTemplate;
    
    public PaymentResponse process(PaymentRequest req) {
        // Fetch secret at runtime
        String apiKey = vaultTemplate.read(
            "secret/stripe/api-key"
        ).getData().get("value");
        
        stripe.setApiKey(apiKey);
        return stripe.charge(req);
    }
}
```

### **Secret Rotation**

```
Manual Rotation (Old Way):
  1. Generate new API key
  2. Update every place it's used
  3. Deploy all services
  4. Test everything
  Time: Hours to days
  Risk: Downtime

Automated Rotation (DevSecOps):
  1. Vault generates new API key
  2. Updates secret in Vault
  3. Applications auto-refresh
  Time: Minutes
  Risk: Minimal
  
Example Configuration:
  aws secretsmanager rotate-secret \
    --secret-id stripe-api-key \
    --rotation-lambda-arn arn:aws:lambda:... \
    --rotation-rules AutomaticallyAfterDays=30
```

---

## **6. Container Security**

### **Secure Dockerfile Practices**

```dockerfile
# ❌ BAD Dockerfile
FROM ubuntu:latest  # Don't use 'latest'
RUN apt-get update && apt-get install -y *  # Too many packages
USER root  # Running as root
COPY . /app  # Copies everything including secrets

# ✅ GOOD Dockerfile
FROM eclipse-temurin:17-jre-alpine  # Specific, minimal base
                                     # JRE only (not JDK)

# Create non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Install only needed packages
RUN apk add --no-cache curl

# Copy only necessary files
COPY --chown=appuser:appgroup target/app.jar /app/app.jar

# Switch to non-root user
USER appuser

WORKDIR /app

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

# Run application
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### **Container Image Scanning**

```bash
# Scan with Trivy
trivy image payment-service:1.0.0

Results:
  ┌───────────────┬──────────────────┬──────────┬───────────────────┐
  │    Library    │  Vulnerability   │ Severity │  Installed Version │
  ├───────────────┼──────────────────┼──────────┼───────────────────┤
  │ spring-core   │ CVE-2022-22965   │ CRITICAL │     5.3.18         │
  │ log4j-core    │ CVE-2021-44228   │ CRITICAL │     2.14.1         │
  │ jackson-bind  │ CVE-2022-12345   │   HIGH   │     2.12.3         │
  └───────────────┴──────────────────┴──────────┴───────────────────┘

Action: Update dependencies
  spring-core: 5.3.18 → 5.3.20
  log4j-core: 2.14.1 → 2.17.1
  jackson-bind: 2.12.3 → 2.13.3
```

---

## **7. Vulnerability Management**

### **CVE (Common Vulnerabilities and Exposures)**

```
CVE-2021-44228 (Log4Shell)
  Severity: CRITICAL (10.0/10)
  Impact: Remote Code Execution
  Affected: log4j-core 2.0-2.14.1
  Fix: Upgrade to 2.17.1+

Response Process:
  1. Detection (Alert from scanner)
  2. Assessment (Is our app affected?)
  3. Prioritization (Based on severity + exposure)
  4. Remediation (Patch, upgrade, mitigate)
  5. Verification (Rescan, test)
  6. Monitoring (Ensure no regression)
```

### **Patch Management**

```
Priority Matrix:

┌─────────────┬──────────────┬──────────────────┐
│  Severity   │  Exposure    │  Response Time   │
├─────────────┼──────────────┼──────────────────┤
│  CRITICAL   │  Internet    │  24 hours        │
│  CRITICAL   │  Internal    │  1 week          │
│  HIGH       │  Internet    │  1 week          │
│  HIGH       │  Internal    │  2 weeks         │
│  MEDIUM     │  Any         │  1 month         │
│  LOW        │  Any         │  Next cycle      │
└─────────────┴──────────────┴──────────────────┘

Example: Log4Shell
  Severity: CRITICAL
  Exposure: Internet-facing
  Response: Patched within 12 hours
  Process: Emergency deploy on weekend
```

---

## **8. Interview Q&A**

### **Q1: What is the difference between SAST and DAST?**

**✅ Good Answer:**
"SAST analyzes source code without executing it, finding vulnerabilities like SQL injection or hardcoded secrets during the build phase. It's fast but can have false positives. DAST tests the running application, finding runtime issues like authentication bypasses or server misconfigurations. It's slower but finds real exploitable vulnerabilities. In practice, I use both: SAST in CI/CD for every commit, and DAST before production deployment. They're complementary—SAST catches code-level issues early, while DAST validates actual security posture."

---

### **Q2: How do you handle secrets in a containerized application?**

**✅ Good Answer:**
"I never bake secrets into container images or store them in environment variables hardcoded in deployment configs. Instead, I use secret management tools like HashiCorp Vault or AWS Secrets Manager. In Kubernetes, I create Secrets and mount them as volumes or environment variables at runtime. The application fetches secrets on startup using IAM-based authentication, not stored credentials. I also implement secret rotation, so compromised secrets are automatically replaced. This ensures secrets are never in source control, container images, or logs."

**Real Example:**
"At my last job, we migrated from hardcoded database passwords to Vault. The application fetched credentials on startup and refreshed them every hour. When we had a potential credential exposure, we rotated the secret in Vault, and all pods automatically picked up the new credential within an hour—no deployment needed."

---

## **9. Challenges**

### **Challenge: Secure CI/CD Pipeline**

**Task:** Design security checks for Java Spring Boot app

<details>
<summary>💡 Solution</summary>

```yaml
name: Secure CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # 1. Secret Scanning
  secret-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: GitGuardian Scan
        run: |
          docker run --rm -v $PWD:/repo gitguardian/ggshield \
            secret scan repo /repo
  
  # 2. Code Quality & Security (SAST)
  code-security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          java-version: '17'
      - name: SonarQube Scan
        run: |
          mvn clean verify sonar:sonar \
            -Dsonar.qualitygate.wait=true
  
  # 3. Dependency Vulnerability Scan (SCA)
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: OWASP Dependency Check
        run: |
          mvn dependency-check:check \
            -DfailBuildOnCVSS=7  # Fail on HIGH+
  
  # 4. Build & Container Scan
  build-and-scan:
    needs: [secret-scan, code-security, dependency-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Build JAR
        run: mvn clean package -DskipTests
      - name: Build Container
        run: |
          docker build -t payment-service:${{ github.sha }} .
      - name: Trivy Container Scan
        run: |
          docker run --rm \
            -v /var/run/docker.sock:/var/run/docker.sock \
            aquasec/trivy:latest image \
            --severity CRITICAL,HIGH \
            --exit-code 1 \
            payment-service:${{ github.sha }}
  
  # 5. Deploy to Staging
  deploy-staging:
    needs: build-and-scan
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/payment-service \
            app=payment-service:${{ github.sha }}
  
  # 6. DAST Scan (on staging)
  dast-scan:
    needs: deploy-staging
    runs-on: ubuntu-latest
    steps:
      - name: OWASP ZAP Scan
        run: |
          docker run --rm \
            owasp/zap2docker-stable zap-baseline.py \
            -t https://staging.example.com \
            -r zap-report.html
      - name: Upload Report
        uses: actions/upload-artifact@v3
        with:
          name: zap-report
          path: zap-report.html
  
  # 7. Deploy to Production
  deploy-production:
    needs: dast-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.example.com
    steps:
      - name: Deploy with Approval
        run: |
          kubectl set image deployment/payment-service \
            app=payment-service:${{ github.sha }} \
            --namespace=production
```

**Security Gates:**
1. ✅ Secret scanning (block secrets)
2. ✅ SAST (code vulnerabilities)
3. ✅ SCA (dependency vulnerabilities)
4. ✅ Container scanning (image vulnerabilities)
5. ✅ DAST (runtime vulnerabilities)
6. ✅ Manual approval before production

**XP: +80** 🏆

</details>

---

**Achievement Unlocked**: 🏆 **Security Champion** (+600 XP)

**Next**: [14: Cloud Native Concepts →](14_Cloud_Native_Concepts.md)

**Total XP**: +80 from challenges, +600 achievement = **+680 XP** 🚀
