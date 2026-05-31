# 🚀 Design a Code Deployment System (CI/CD)

> *"'Deploy on Friday at 5 PM' used to be a developer's worst nightmare. Now, companies like Netflix deploy thousands of times per day with ZERO downtime. The secret? A robust deployment pipeline that builds, tests, and rolls out code safely — with automatic rollback if anything goes wrong. Designing this system teaches you about distributed systems, queue-based processing, progressive rollouts, and the art of making deployments boring."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Microservices](../Microservices/), [Message Queues](../BuildingBlocks/MessageQueues.md), [Load Balancing](../BuildingBlocks/LoadBalancing.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [High-Level Architecture](#-high-level-architecture)
3. [Build Pipeline](#-build-pipeline)
4. [Deployment Strategies](#-deployment-strategies)
5. [Rollback Mechanisms](#-rollback-mechanisms)
6. [Artifact Management](#-artifact-management)
7. [Health Monitoring & Auto-Rollback](#-health-monitoring--auto-rollback)
8. [Java Implementation](#-java-implementation)
9. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Git push triggers automated build + test + deploy pipeline
  • Support multiple environments (dev → staging → prod)
  • Multiple deployment strategies (rolling, blue-green, canary)
  • Automatic rollback on failure (health check based!)
  • Parallel builds for multiple services
  • Approval gates (human approval before prod!)
  • Environment-specific configuration management
  • Deployment history and audit trail
  
NON-FUNCTIONAL:
  • Zero-downtime deployments (users NEVER see errors!)
  • Build time: < 10 minutes (fast feedback!)
  • Deploy time: < 5 minutes per service
  • Rollback time: < 1 minute (instant!)
  • Concurrent deploys: 100+ services simultaneously
  • Reliability: failed deploy = auto-rollback (not stuck!)

SCALE:
  • 500+ microservices
  • 1000+ deployments per day
  • 100+ concurrent builds
  • Multi-region deployment (US, EU, APAC)
```

---

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      CODE DEPLOYMENT SYSTEM                                │
│                                                                            │
│  ┌─────────┐    ┌────────────────────────────────────────────────────┐   │
│  │   Git   │───►│  Webhook Handler (trigger on push/merge!)          │   │
│  │  (GitHub)│    └────────────────────┬───────────────────────────────┘   │
│  └─────────┘                          │                                    │
│                              ┌────────▼────────────┐                      │
│                              │  Pipeline Orchestrator│                      │
│                              │  (state machine!)    │                      │
│                              └────┬─────┬─────┬────┘                      │
│                                   │     │     │                            │
│          ┌────────────────────────▼┐    │    ┌▼────────────────────────┐  │
│          │  BUILD STAGE            │    │    │  DEPLOY STAGE           │  │
│          │  • Clone repo           │    │    │  • Select strategy      │  │
│          │  • Install deps         │    │    │  • Progressive rollout  │  │
│          │  • Compile              │    │    │  • Health check         │  │
│          │  • Unit tests           │    │    │  • Auto-rollback!       │  │
│          │  • Build Docker image   │    │    │                         │  │
│          │  • Push to registry     │    │    │  Targets:               │  │
│          └─────────────────────────┘    │    │  • Kubernetes clusters  │  │
│                                          │    │  • Cloud VMs            │  │
│                              ┌───────────▼┐   │  • Serverless           │  │
│                              │ TEST STAGE  │   └─────────────────────────┘  │
│                              │ • Integration│                               │
│                              │ • E2E tests  │                               │
│                              │ • Security   │                               │
│                              │   scan       │                               │
│                              │ • Performance│                               │
│                              └─────────────┘                               │
│                                                                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  │
│  │   Artifact   │  │   Config     │  │   Metrics    │  │  Approval  │  │
│  │   Registry   │  │   Store      │  │   Collector  │  │  Service   │  │
│  │   (Docker)   │  │   (Vault)    │  │   (Prometheus)│  │  (Slack!)  │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘

PIPELINE STATE MACHINE:
  TRIGGERED → BUILDING → TESTING → AWAITING_APPROVAL → DEPLOYING → 
  → VERIFYING → COMPLETED
  
  At any point: failure → ROLLING_BACK → ROLLED_BACK
```

---

## 🔨 Build Pipeline

```
BUILD STAGE (deterministic, reproducible!):

  ┌─────────────────────────────────────────────────────────────────┐
  │  1. SOURCE: Clone git repo at specific commit SHA               │
  │     git clone --depth 1 --branch $BRANCH $REPO                 │
  │                                                                  │
  │  2. DEPENDENCIES: Install (cached for speed!)                    │
  │     Cache key: hash(pom.xml) or hash(package-lock.json)         │
  │     HIT: restore from cache (save 2-3 min!)                     │
  │     MISS: fresh install                                          │
  │                                                                  │
  │  3. BUILD: Compile source code                                   │
  │     Maven: mvn clean package -DskipTests                        │
  │     Gradle: ./gradlew assemble                                   │
  │                                                                  │
  │  4. TEST: Unit tests (fast! < 2 min)                            │
  │     Maven: mvn test                                              │
  │     Report: JUnit XML → parsed for pass/fail!                   │
  │                                                                  │
  │  5. SCAN: Security + quality                                     │
  │     SonarQube: code quality gate                                 │
  │     Snyk/Trivy: dependency vulnerabilities                       │
  │     FAIL if critical vulnerabilities found!                      │
  │                                                                  │
  │  6. PACKAGE: Build Docker image                                  │
  │     docker build -t service:$GIT_SHA .                          │
  │     Tag: git SHA (immutable, traceable!)                         │
  │                                                                  │
  │  7. PUBLISH: Push to container registry                          │
  │     docker push registry.company.com/service:$GIT_SHA           │
  └─────────────────────────────────────────────────────────────────┘

BUILD OPTIMIZATION:
  • Multi-stage Docker builds (smaller images!)
  • Build cache layers (only rebuild changed layers!)
  • Parallel test execution (split across N workers!)
  • Incremental compilation (only recompile changed modules!)
  • Remote build cache (share cache across developers!)
```

---

## 🎯 Deployment Strategies

```
STRATEGY 1: ROLLING UPDATE (Kubernetes default!)
  ┌─────────────────────────────────────────────────────────────────┐
  │  Replace instances one-by-one:                                   │
  │                                                                  │
  │  Start:  [v1] [v1] [v1] [v1]  (4 replicas, all v1)            │
  │  Step 1: [v2] [v1] [v1] [v1]  (1 replaced, health check!)      │
  │  Step 2: [v2] [v2] [v1] [v1]  (2 replaced)                     │
  │  Step 3: [v2] [v2] [v2] [v1]  (3 replaced)                     │
  │  Done:   [v2] [v2] [v2] [v2]  (all v2!)                        │
  │                                                                  │
  │  ✅ Zero downtime (always some instances serving!)               │
  │  ✅ Low resource overhead (max 1 extra instance)                 │
  │  ❌ Mix of versions during rollout (compatibility!)              │
  │  ❌ Slow rollback (must roll forward or restart all!)            │
  └─────────────────────────────────────────────────────────────────┘

STRATEGY 2: BLUE-GREEN DEPLOYMENT
  ┌─────────────────────────────────────────────────────────────────┐
  │  Two identical environments, only one receives traffic!          │
  │                                                                  │
  │  ┌─────────┐       ┌──────────────────────────┐                │
  │  │   Load  │──────►│  BLUE (v1) ← LIVE!       │                │
  │  │Balancer │       └──────────────────────────┘                │
  │  └─────────┘       ┌──────────────────────────┐                │
  │                     │  GREEN (v2) ← idle, testing! │            │
  │                     └──────────────────────────┘                │
  │                                                                  │
  │  After validation: switch LB to GREEN!                           │
  │  Rollback: switch LB back to BLUE! (instant! < 1 second!)      │
  │                                                                  │
  │  ✅ Instant rollback (just switch traffic!)                      │
  │  ✅ Full testing before going live                               │
  │  ❌ 2× infrastructure cost (both environments running!)         │
  │  ❌ Database migrations must be backward-compatible!             │
  └─────────────────────────────────────────────────────────────────┘

STRATEGY 3: CANARY DEPLOYMENT (safest for production!)
  ┌─────────────────────────────────────────────────────────────────┐
  │  Gradually shift traffic to new version, monitoring metrics!     │
  │                                                                  │
  │  Phase 1: 5% traffic → v2 (canary!)                             │
  │           Monitor: error rate, latency, CPU...                   │
  │           If OK after 10 min → proceed!                          │
  │           If errors spike → ROLLBACK immediately!                │
  │                                                                  │
  │  Phase 2: 25% traffic → v2                                      │
  │           Monitor for 10 min...                                  │
  │                                                                  │
  │  Phase 3: 50% traffic → v2                                      │
  │           Monitor for 10 min...                                  │
  │                                                                  │
  │  Phase 4: 100% traffic → v2 (DONE! ✅)                          │
  │                                                                  │
  │  ✅ Minimal blast radius (5% affected if bug!)                   │
  │  ✅ Automatic rollback on metric degradation!                    │
  │  ✅ Real production traffic validates the release!               │
  │  ❌ Slower rollout (30-60 min for full deployment)               │
  │  ❌ Complex traffic splitting infrastructure needed              │
  └─────────────────────────────────────────────────────────────────┘
```

---

## ⏪ Rollback Mechanisms

```
AUTOMATIC ROLLBACK TRIGGERS:
  • Error rate > 1% (5xx responses!)
  • P99 latency > 2× baseline
  • Health check failures > 3 consecutive
  • CPU/Memory spike > 90%
  • Custom business metric degradation (e.g., order rate drops!)

ROLLBACK APPROACHES:
  ┌───────────────────────────────────────────────────────────────────┐
  │  Approach          │  Speed   │  How                              │
  ├───────────────────────────────────────────────────────────────────┤
  │  Traffic switch    │  < 1s    │  LB points back to old version   │
  │  (blue-green)     │          │  (old version still running!)     │
  ├───────────────────────────────────────────────────────────────────┤
  │  Canary abort     │  < 10s   │  Route canary traffic back to v1  │
  │                   │          │  Kill canary instances!            │
  ├───────────────────────────────────────────────────────────────────┤
  │  Kubernetes rollback│ 30-60s  │  kubectl rollout undo             │
  │  (rolling)        │          │  Redeploys previous ReplicaSet!   │
  ├───────────────────────────────────────────────────────────────────┤
  │  Redeploy previous│  3-5 min │  Deploy known-good image again    │
  │  version          │          │  (artifact still in registry!)    │
  └───────────────────────────────────────────────────────────────────┘

DATABASE ROLLBACK (the HARD part!):
  Code rollback is easy. Database migrations are NOT easily reversible!
  
  RULE: All migrations must be BACKWARD-COMPATIBLE!
  
  Adding a column? → Old code ignores it. ✅
  Removing a column? → Deploy code that stops using it FIRST!
                       Then remove column in NEXT deploy! (2-step!)
  Renaming? → Add new column → migrate data → deploy code → drop old!
  
  NEVER: drop a column in the same deploy that removes code using it!
  (Rollback would bring back code that needs the dropped column! 💀)
```

---

## 📦 Artifact Management

```
IMMUTABLE ARTIFACTS:
  Build ONCE → deploy EVERYWHERE (same binary in dev, staging, prod!)
  
  ┌────────────────────────────────────────────────────────────────┐
  │  Artifact: myservice:a3f8b2c1 (tagged with git SHA!)           │
  │  Built from: commit a3f8b2c1                                    │
  │  Tested: unit ✅, integration ✅, security ✅                   │
  │  Promoted: dev ✅ → staging ✅ → prod (pending approval!)       │
  └────────────────────────────────────────────────────────────────┘
  
  Same image in all environments! Config differs, not the binary!
  
  Config injection (environment-specific):
  • Environment variables (12-factor app!)
  • Kubernetes ConfigMaps / Secrets
  • Vault (HashiCorp) for sensitive config!

ARTIFACT REGISTRY:
  • Store: Docker images, JAR files, Helm charts
  • Retention: keep last 30 days + tagged releases forever!
  • Vulnerability scanning: scan on push! (Trivy/Snyk)
  • Promotion: "promote staging → prod" = tag image, not rebuild!
```

---

## 📊 Health Monitoring & Auto-Rollback

```
DEPLOYMENT VERIFICATION:

  After deploying new version:
  
  ┌─────────────────────────────────────────────────────────────────┐
  │  MONITORING WINDOW (10 minutes!)                                 │
  │                                                                  │
  │  Metrics watched:                                                │
  │  • Error rate (5xx): baseline=0.1%, alert if > 1%              │
  │  • Latency P99: baseline=200ms, alert if > 500ms              │
  │  • Request rate: baseline=1000 rps, alert if drops > 20%       │
  │  • Health check: /health returns 200?                           │
  │  • CPU/Memory: alert if > 90%                                   │
  │  • Custom: order conversion rate, payment success rate          │
  │                                                                  │
  │  Decision engine:                                                │
  │  IF any metric breaches threshold for > 2 minutes:              │
  │    → AUTOMATIC ROLLBACK! 🚨                                     │
  │    → Alert on-call engineer!                                     │
  │    → Pipeline marked FAILED!                                     │
  │                                                                  │
  │  IF all metrics healthy after 10 min:                           │
  │    → Deployment CONFIRMED! ✅                                    │
  │    → Proceed to next canary phase (or complete!)                │
  └─────────────────────────────────────────────────────────────────┘

PROGRESSIVE DELIVERY (Argo Rollouts / Flagger):
  spec:
    strategy:
      canary:
        steps:
        - setWeight: 5     # 5% traffic to canary
        - pause: {duration: 5m}
        - analysis:         # Check metrics!
            templates: [error-rate, latency]
        - setWeight: 25
        - pause: {duration: 5m}
        - analysis: ...
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100   # Full rollout!
```

---

## 💻 Java Implementation

### Pipeline Orchestrator

```java
@Service
public class PipelineOrchestrator {
    
    @Autowired private BuildService buildService;
    @Autowired private TestService testService;
    @Autowired private DeployService deployService;
    @Autowired private MetricsService metricsService;
    @Autowired private NotificationService notificationService;
    
    /**
     * Execute deployment pipeline (state machine!).
     */
    public PipelineResult executePipeline(PipelineConfig config) {
        Pipeline pipeline = Pipeline.create(config);
        
        try {
            // Stage 1: BUILD
            pipeline.transition(PipelineState.BUILDING);
            BuildResult build = buildService.build(config);
            if (!build.isSuccess()) {
                return fail(pipeline, "Build failed: " + build.getError());
            }
            
            // Stage 2: TEST
            pipeline.transition(PipelineState.TESTING);
            TestResult tests = testService.runAll(build.getArtifactId());
            if (!tests.allPassed()) {
                return fail(pipeline, "Tests failed: " + tests.getFailures());
            }
            
            // Stage 3: APPROVAL (for production!)
            if (config.getEnvironment() == Environment.PRODUCTION) {
                pipeline.transition(PipelineState.AWAITING_APPROVAL);
                boolean approved = waitForApproval(pipeline, Duration.ofHours(4));
                if (!approved) {
                    return fail(pipeline, "Approval timeout or rejected");
                }
            }
            
            // Stage 4: DEPLOY
            pipeline.transition(PipelineState.DEPLOYING);
            DeployResult deploy = deployService.deploy(
                build.getArtifactId(), config);
            
            // Stage 5: VERIFY
            pipeline.transition(PipelineState.VERIFYING);
            boolean healthy = metricsService.verifyHealth(
                config.getService(), Duration.ofMinutes(10));
            
            if (!healthy) {
                // AUTO-ROLLBACK!
                pipeline.transition(PipelineState.ROLLING_BACK);
                deployService.rollback(config);
                return fail(pipeline, "Health check failed — rolled back!");
            }
            
            pipeline.transition(PipelineState.COMPLETED);
            notificationService.notifySuccess(pipeline);
            return PipelineResult.success(pipeline);
            
        } catch (Exception e) {
            pipeline.transition(PipelineState.FAILED);
            notificationService.notifyFailure(pipeline, e);
            return PipelineResult.failed(pipeline, e.getMessage());
        }
    }
}
```

### Canary Deployment Service

```java
@Service
public class CanaryDeployService {
    
    @Autowired private KubernetesClient k8s;
    @Autowired private MetricsService metrics;
    @Autowired private TrafficRouter trafficRouter;
    
    /**
     * Progressive canary deployment with auto-rollback!
     */
    public DeployResult canaryDeploy(String service, String newImage, 
                                      CanaryConfig config) {
        // Deploy canary replicas (small set running new version!)
        String canaryDeployment = deployCanaryReplicas(service, newImage);
        
        for (CanaryPhase phase : config.getPhases()) {
            // Shift traffic to canary
            trafficRouter.setWeight(service, "canary", phase.getWeightPercent());
            log.info("Canary phase: {}% traffic to new version", 
                phase.getWeightPercent());
            
            // Monitor for configured duration
            MetricSnapshot before = metrics.snapshot(service);
            
            boolean healthy = metrics.monitorHealth(service, phase.getDuration(),
                MetricThresholds.builder()
                    .maxErrorRate(0.01)        // Max 1% errors
                    .maxLatencyP99Ms(500)      // Max 500ms P99
                    .minRequestRate(before.getRequestRate() * 0.8) // No crash
                    .build());
            
            if (!healthy) {
                // ROLLBACK! Route all traffic back to stable!
                log.error("Canary health check FAILED at {}%! Rolling back!", 
                    phase.getWeightPercent());
                trafficRouter.setWeight(service, "canary", 0);
                deleteCanaryReplicas(canaryDeployment);
                return DeployResult.rolledBack("Metrics degraded at " 
                    + phase.getWeightPercent() + "% canary");
            }
            
            log.info("Phase {}% healthy! Proceeding...", phase.getWeightPercent());
        }
        
        // All phases passed! Promote canary to stable!
        promoteCanaryToStable(service, newImage);
        deleteCanaryReplicas(canaryDeployment);
        
        return DeployResult.success("Canary promoted to 100%!");
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you achieve zero-downtime deployments?**
> Three mechanisms: (1) Rolling updates: replace instances one at a time, always maintaining minimum healthy instances (K8s: maxUnavailable=0, maxSurge=1), (2) Readiness probes: new instance only receives traffic AFTER it's fully warmed up (loaded caches, DB connections established), (3) Connection draining: old instance stops receiving NEW requests but finishes existing ones (graceful shutdown period: 30-60s). Together: users always hit a healthy instance, never see 503/502 errors during deployment.

**Q2: How would you design a deployment system for 500 microservices?**
> (1) Pipeline-per-service: each service has its own pipeline config (build, test, deploy steps), (2) Shared pipeline library: common steps abstracted into reusable templates, (3) Parallel execution: independent services deploy in parallel (dependency graph determines ordering), (4) Gitops: desired state in Git, controller converges actual state to match (ArgoCD), (5) Service mesh integration: traffic shifting for canary at mesh level (Istio), (6) Centralized dashboard: all 500 services' deploy status visible, aggregate health metrics.

**Q3: How do you handle database migrations during deployment?**
> "Expand and contract" pattern: (1) EXPAND: add new schema (column, table) while keeping old working, (2) MIGRATE: backfill data + update code to use new schema, (3) CONTRACT: remove old schema (in a LATER deploy!). Never combine breaking schema change + code change in one deploy! This ensures rollback is always safe — old code works with expanded schema, new code works with expanded schema. Tools: Flyway/Liquibase with versioned migrations, always forward-only (never "down" migrations in production!).

**Q4: Canary vs Blue-Green — when to use each?**
> Canary: when you want to test with REAL production traffic at LOW risk (5% traffic → catch bugs that only appear at scale!). Best for: services with high traffic (statistically significant sample at 5%), gradual confidence building, ML model deployments. Blue-Green: when you need INSTANT switch + INSTANT rollback, and you can validate in staging. Best for: critical services where any production error is unacceptable, scheduled maintenance windows, major version upgrades. In practice: use canary for regular deploys, blue-green for major releases and database-heavy changes.

---

## 🔗 Related Topics
- [Load Balancing](../BuildingBlocks/LoadBalancing.md) — Traffic splitting for canary
- [Circuit Breaker](../BuildingBlocks/CircuitBreaker.md) — Failure detection
- [Observability](../Observability/) — Monitoring deployments
- [Microservices](../Microservices/) — Service-level deployments

---

*"The goal of a deployment system is to make deploys so safe and fast that they become boring. If your team celebrates a successful deployment, your deployment system needs work. Deployments should be as routine and unremarkable as sending an email." — Charity Majors, Honeycomb* 🚀
