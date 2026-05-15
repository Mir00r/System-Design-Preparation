# **Tutorial 10: Configuration Management** ⚙️

**Master Config Before Spring Cloud Config/Vault**

---

## **📋 Table of Contents**

1. [The Configuration Hell](#1-the-configuration-hell)
2. [What is Configuration Management?](#2-what-is-configuration-management)
3. [Environment-Specific Configuration](#3-environment-specific-configuration)
4. [Secret Management](#4-secret-management)
5. [Configuration as Code](#5-configuration-as-code)
6. [Dynamic Configuration](#6-dynamic-configuration)
7. [Best Practices](#7-best-practices)
8. [Interview Q&A](#8-interview-qa)
9. [Challenges](#9-challenges)

---

## **1. The Configuration Hell**

```
Production Outage - 2 AM

Alert: Database connection failed!
You: *Checks application.yml*

application.yml (hardcoded):
  database.url=jdbc:mysql://prod-db-1:3306/app
  
Problem: prod-db-1 migrated to prod-db-2 last week
Fix: Need to update config and redeploy
Time: 30 minutes downtime

Next Day:
  Same issue in test environment
  Same hardcoded value
  Every environment needs code change
  
Manager: "Why is config in the code?!"
```

**Without Config Management:**
- Hardcoded values
- Secrets in source control
- Deploy needed for config changes
- No environment separation

---

## **2. What is Configuration Management?**

### **The 12-Factor App Principle**

```
III. Config
  Store config in the environment, not in code
  
What is config?
  ✅ Database URLs
  ✅ API keys
  ✅ Feature flags
  ✅ Resource limits
  
  ❌ NOT application code
  ❌ NOT internal constants
```

### **Configuration Hierarchy**

```
1. Defaults (in code)
2. Config files (application.yml)
3. Environment variables
4. Command-line arguments
5. External config server

Higher number = Higher priority
```

---

## **3. Environment-Specific Configuration**

### **Spring Boot Profiles**

```yaml
# application.yml (defaults)
server:
  port: 8080

app:
  name: payment-service

---
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    username: dev_user
    password: dev_password
    
logging:
  level:
    root: DEBUG

---
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/prod_db
    username: ${DB_USERNAME}  # From environment
    password: ${DB_PASSWORD}  # From secret store
    
logging:
  level:
    root: WARN
```

```java
@SpringBootApplication
public class PaymentServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(PaymentServiceApplication.class, args);
    }
}

// Run with:
// java -jar app.jar --spring.profiles.active=prod
```

### **Environment Variables**

```bash
# Development
export DB_HOST=localhost
export DB_PORT=3306
export API_KEY=dev_key_123

# Production (Kubernetes)
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: prod-db.example.com
  DB_PORT: "3306"
  LOG_LEVEL: "WARN"

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: payment
        envFrom:
        - configMapRef:
            name: app-config
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

---

## **4. Secret Management**

### **The Problem**

```
❌ BAD: Secrets in code
application.yml:
  stripe:
    api-key: sk_live_abc123xyz789  # NEVER DO THIS!
    
❌ BAD: Secrets in Git
.env:
  DATABASE_PASSWORD=super_secret_password
  
git add .env  # ← Leaked to version control!
```

### **Solutions**

#### **1. Environment Variables**
```yaml
# Config references env vars
spring:
  datasource:
    password: ${DB_PASSWORD}  # From environment

# Set in deployment
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  password: c3VwZXJfc2VjcmV0  # base64 encoded
```

#### **2. External Secret Store**

```java
@Configuration
public class SecretsConfig {
    
    @Value("${vault.token}")
    private String vaultToken;
    
    @Bean
    public DataSource dataSource() {
        // Fetch secret from Vault at runtime
        String password = vaultClient.read(
            "secret/database/password",
            vaultToken
        );
        
        return DataSourceBuilder.create()
            .url(dbUrl)
            .username(dbUsername)
            .password(password)  // From Vault
            .build();
    }
}
```

### **Secret Rotation**

```
Problem: API key compromised

With Hardcoded Secrets:
  1. Update code
  2. Commit to Git
  3. Build new version
  4. Deploy to production
  Time: 30+ minutes

With Secret Management:
  1. Update secret in Vault
  2. Restart application (reads new secret)
  Time: 2 minutes
  
Or with dynamic secrets:
  1. Update secret in Vault
  2. Application auto-refreshes
  Time: 0 minutes (automatic)
```

---

## **5. Configuration as Code**

### **Declarative Configuration**

```yaml
# config.yaml - Version controlled
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-config
data:
  application.yml: |
    server:
      port: 8080
    
    payment:
      max-amount: 10000
      retry-attempts: 3
      timeout: 30s
    
    integration:
      stripe:
        api-url: https://api.stripe.com
      
  logback.xml: |
    <configuration>
      <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
          <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
      </appender>
    </configuration>
```

**Benefits:**
- ✅ Version controlled
- ✅ Code review for config changes
- ✅ Rollback capability
- ✅ Audit trail

---

## **6. Dynamic Configuration**

### **Feature Flags**

```java
@Service
public class PaymentService {
    
    @Autowired
    private FeatureFlagClient featureFlags;
    
    public PaymentResult processPayment(PaymentRequest request) {
        
        // Dynamic configuration - no deploy needed!
        if (featureFlags.isEnabled("use-stripe-api-v2")) {
            return stripeV2.process(request);
        } else {
            return stripeV1.process(request);
        }
    }
}

// Toggle via dashboard, no deployment!
```

### **Configuration Server**

```
Spring Cloud Config:

Config Server → Git Repository
               ├─ payment-service-dev.yml
               ├─ payment-service-prod.yml
               └─ common.yml

Application startup:
  1. Contact config server
  2. Fetch configuration
  3. Apply settings
  
Runtime refresh (no restart):
  1. Update config in Git
  2. POST /actuator/refresh
  3. Application reloads config
```

---

## **7. Best Practices**

### **Configuration Checklist**

```
✅ DO:
  - Externalize all environment-specific values
  - Use environment variables for secrets
  - Version control configuration files
  - Separate config from code
  - Use meaningful names
  - Document configuration options
  - Validate configuration at startup

❌ DON'T:
  - Hardcode credentials
  - Commit secrets to Git
  - Use production credentials locally
  - Mix code and configuration
  - Deploy to change simple config values
```

### **Configuration Structure**

```yaml
# Good: Clear, hierarchical
database:
  connection:
    host: ${DB_HOST}
    port: ${DB_PORT:3306}  # Default: 3306
    name: ${DB_NAME}
  pool:
    max-size: ${DB_POOL_MAX_SIZE:20}
    min-size: ${DB_POOL_MIN_SIZE:5}
  timeout:
    connection: ${DB_CONN_TIMEOUT:30s}
    query: ${DB_QUERY_TIMEOUT:60s}

# Bad: Flat, unclear
db_host: ${DB_HOST}
db_port: ${DB_PORT}
max_connections: 20
min_connections: 5
```

---

## **8. Interview Q&A**

### **Q1: How do you handle secrets in configuration?**

**✅ Good Answer:**
"I never commit secrets to version control. For development, I use environment variables or local `.env` files that are gitignored. For production, I use a secret management system like HashiCorp Vault or AWS Secrets Manager. The application fetches secrets at runtime using IAM roles or service accounts, not hardcoded credentials. For Kubernetes, I use Secrets with RBAC to control access. I also implement secret rotation to periodically update credentials without downtime."

---

### **Q2: Explain the difference between ConfigMap and Secret in Kubernetes**

**✅ Good Answer:**
"Both store configuration data, but Secrets are for sensitive information like passwords and API keys, while ConfigMaps are for non-sensitive configuration. Secrets are base64-encoded (not encrypted by default, but can be with encryption-at-rest) and have additional security controls like RBAC. ConfigMaps store data as plain text. In practice, I use ConfigMaps for application.yml and Secrets for database passwords, keeping sensitive data separate."

---

## **9. Challenges**

### **Challenge: Multi-Environment Config**

**Task:** Design configuration for dev/staging/prod environments

<details>
<summary>💡 Solution</summary>

```yaml
# Base (application.yml)
spring:
  application:
    name: payment-service

app:
  version: ${APP_VERSION:1.0.0}

---
# Dev (application-dev.yml)
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    username: dev_user
    password: dev_pass  # OK for local dev

logging.level.root: DEBUG

---
# Staging (application-staging.yml)
spring:
  datasource:
    url: jdbc:mysql://staging-db:3306/staging_db
    username: ${DB_USER}
    password: ${DB_PASS}  # From environment

logging.level.root: INFO

---
# Production (application-prod.yml)
spring:
  datasource:
    url: jdbc:mysql://prod-db.cluster:3306/prod_db
    username: ${DB_USER}
    password: ${DB_PASS}  # From Vault/Secrets Manager

logging.level.root: WARN

resilience4j:
  circuitbreaker:
    enabled: true
```

**XP: +50** 🏆

</details>

---

**Achievement Unlocked**: 🏆 **Configuration Master** (+400 XP)

**Next**: [12: Incident Management →](12_Incident_Management.md)

**Total XP**: +50 from challenges, +400 achievement = **+450 XP** 🚀
