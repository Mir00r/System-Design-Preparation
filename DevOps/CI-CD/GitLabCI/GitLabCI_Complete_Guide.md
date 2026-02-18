# **GitLab CI/CD - Complete Guide** 🦊🔄

**Master GitLab's Integrated CI/CD Platform with Auto DevOps and Kubernetes Native Features**

---

## **Table of Contents** 📑
1. [GitLab CI/CD Overview](#1-gitlab-cicd-overview)
2. [GitLab Architecture](#2-gitlab-architecture)
3. [`.gitlab-ci.yml` Syntax](#3-gitlab-ciyml-syntax)
4. [Jobs, Stages & Pipelines](#4-jobs-stages--pipelines)
5. [GitLab Runners](#5-gitlab-runners)
6. [Advanced Pipeline Features](#6-advanced-pipeline-features)
7. [Docker & Kubernetes Integration](#7-docker--kubernetes-integration)
8. [Auto DevOps](#8-auto-devops)
9. [Real-World Pipeline Examples](#9-real-world-pipeline-examples)
10. [Best Practices](#10-best-practices)
11. [Troubleshooting](#11-troubleshooting)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. GitLab CI/CD Overview** 🎯

### **What is GitLab CI/CD?**

**GitLab CI/CD** is a built-in continuous integration and delivery tool integrated directly into GitLab. It's part of the complete DevOps platform, providing version control, CI/CD, security scanning, and deployment capabilities in one application.

### **Key Features:**

```
✅ Integrated with GitLab (no separate tool)
✅ Pipeline as Code (.gitlab-ci.yml)
✅ Built-in Container Registry
✅ Kubernetes native
✅ Auto DevOps (zero-config CI/CD)
✅ Security scanning (SAST, DAST, dependency)
✅ Merge request pipelines
✅ Review apps
✅ Environments & deployments
✅ Artifacts & caching
```

### **Why GitLab CI/CD?**

| Feature | GitLab CI/CD | Competitors |
|---------|--------------|-------------|
| **Integration** | Built-in, seamless | External tools |
| **Container Registry** | Included | Separate service |
| **Kubernetes** | Native support | Plugins needed |
| **Auto DevOps** | Zero-config pipelines | Manual setup |
| **Security** | Built-in SAST/DAST | External tools |
| **Cost** | Free tier generous | Limited free tiers |
| **Deployment** | SaaS or Self-hosted | Mostly SaaS |

### **GitLab vs Competitors:**

```mermaid
graph TB
    A[Complete DevOps Platform] --> B[GitLab CI/CD]
    
    B --> C[Source Control]
    B --> D[CI/CD Pipelines]
    B --> E[Container Registry]
    B --> F[Security Scanners]
    B --> G[K8s Integration]
    B --> H[Monitoring]
    
    style B fill:#FC6D26
```

---

## **2. GitLab Architecture** 🏗️

### **Components:**

```mermaid
graph TB
    Dev[Developer] -->|Push Code| GitLab[GitLab Server]
    GitLab -->|Trigger| Pipeline[Pipeline]
    
    Pipeline -->|Assign Jobs| Runner1[Shared Runner]
    Pipeline -->|Assign Jobs| Runner2[Specific Runner]
    Pipeline -->|Assign Jobs| Runner3[Group Runner]
    
    Runner1 -->|Docker Executor| Container1[Build Container]
    Runner2 -->|Shell Executor| Server1[Build Server]
    Runner3 -->|Kubernetes Executor| Pod1[K8s Pod]
    
    Container1 --> Registry[Container Registry]
    Server1 --> Artifacts[Artifact Storage]
    Pod1 --> K8s[Kubernetes Cluster]
    
    style GitLab fill:#FC6D26
```

### **Core Components:**

#### **1. GitLab Server**
```
Responsibilities:
  - Host repositories
  - Manage pipelines
  - Coordinate runners
  - Store artifacts
  - Container registry
  - Security scanning
  
Deployment:
  - GitLab.com (SaaS)
  - Self-hosted (Community/Enterprise)
```

#### **2. GitLab Runner**
```
Purpose: Execute CI/CD jobs

Types:
  - Shared Runners (for all projects)
  - Specific Runners (dedicated to project)
  - Group Runners (for group projects)
  
Executors:
  - Shell
  - Docker
  - Docker Machine
  - Kubernetes
  - SSH
  - VirtualBox
```

#### **3. `.gitlab-ci.yml`**
```
Purpose: Define pipeline

Location: Repository root
Format: YAML
Contents:
  - Jobs
  - Stages
  - Scripts
  - Rules
  - Variables
```

#### **4. Container Registry**
```
Purpose: Store Docker images

Features:
  - Built-in
  - Per-project
  - Automatic cleanup
  - Access control
  
URL: registry.gitlab.com/username/project
```

### **Pipeline Execution Flow:**

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GitLab as GitLab Server
    participant Runner as GitLab Runner
    participant Docker as Docker Registry
    participant K8s as Kubernetes
    
    Dev->>GitLab: Push code
    GitLab->>GitLab: Parse .gitlab-ci.yml
    GitLab->>GitLab: Create pipeline
    GitLab->>Runner: Assign job
    Runner->>Runner: Pull image
    Runner->>Runner: Execute script
    Runner->>GitLab: Upload artifacts
    Runner->>Docker: Push image
    GitLab->>K8s: Deploy application
    GitLab->>Dev: Notify status
```

---

## **3. `.gitlab-ci.yml` Syntax** 📝

### **Basic Structure:**

```yaml
# .gitlab-ci.yml

# Global variables
variables:
  DOCKER_DRIVER: overlay2
  
# Stages (execution order)
stages:
  - build
  - test
  - deploy
  
# Global before_script
before_script:
  - echo "Starting job"
  
# Job definition
build-job:
  stage: build
  script:
    - echo "Building application"
    - npm install
    - npm run build
    
test-job:
  stage: test
  script:
    - npm test
    
deploy-job:
  stage: deploy
  script:
    - ./deploy.sh
  only:
    - main
```

### **Complete Example:**

```yaml
# Global settings
image: node:18-alpine

variables:
  NPM_CONFIG_CACHE: "$CI_PROJECT_DIR/.npm"
  
# Stages
stages:
  - build
  - test
  - security
  - deploy
  
# Global cache
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/
    
# Global before_script
before_script:
  - node --version
  - npm --version
  
# Jobs
build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 week
  only:
    - branches
    - tags
    
lint:
  stage: test
  script:
    - npm run lint
  allow_failure: false
  
unit-test:
  stage: test
  script:
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
      junit: junit.xml
      
integration-test:
  stage: test
  script:
    - npm run test:integration
  services:
    - name: postgres:14
      alias: postgres
  variables:
    POSTGRES_DB: testdb
    POSTGRES_USER: testuser
    POSTGRES_PASSWORD: testpass
    
security-scan:
  stage: security
  script:
    - npm audit
  allow_failure: true
  
sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto .
  allow_failure: true
  
build-docker:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE:latest
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - main
    - tags
    
deploy-staging:
  stage: deploy
  script:
    - kubectl config use-context staging
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/myapp
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - main
    
deploy-production:
  stage: deploy
  script:
    - kubectl config use-context production
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - kubectl rollout status deployment/myapp
  environment:
    name: production
    url: https://example.com
    on_stop: stop-production
  when: manual
  only:
    - tags
    
stop-production:
  stage: deploy
  script:
    - kubectl delete deployment myapp
  environment:
    name: production
    action: stop
  when: manual
```

### **Job Keywords:**

```yaml
job-name:
  # Execution settings
  stage: test                    # Stage assignment
  image: node:18                 # Docker image
  services:                      # Additional containers
    - postgres:14
  
  # Scripts
  before_script:                 # Before main script
    - echo "Setup"
  script:                        # Main commands (required)
    - npm test
  after_script:                  # After main script (always runs)
    - cleanup.sh
    
  # Artifacts
  artifacts:
    paths:                       # Files to keep
      - dist/
    exclude:                     # Files to exclude
      - dist/test/
    expire_in: 1 week           # Auto cleanup
    reports:                     # Special reports
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
        
  # Cache
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - node_modules/
    policy: pull-push           # pull, push, pull-push
    
  # When to run
  only:                          # Include conditions
    - main
    - /^release-.*$/            # Regex
  except:                        # Exclude conditions
    - branches
  rules:                         # Advanced conditions
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    - when: manual
    
  # Execution control
  allow_failure: false           # Job can fail without failing pipeline
  when: on_success               # on_success, on_failure, always, manual, delayed
  timeout: 1h                    # Max job duration
  retry:                         # Retry failed jobs
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
      
  # Dependencies
  dependencies:                  # Artifacts from specific jobs
    - build-job
  needs:                         # Run before stage completes
    - build-job
    
  # Environment
  environment:
    name: production
    url: https://example.com
    on_stop: stop-job           # Cleanup job
    auto_stop_in: 1 day         # Auto cleanup
    
  # Tags (runner selection)
  tags:
    - docker
    - linux
    
  # Variables
  variables:
    DEPLOY_ENV: "production"
    
  # Coverage parsing
  coverage: '/Coverage: \d+\.\d+%/'
  
  # Parallel execution
  parallel: 3                    # Run job 3 times
```

---

## **4. Jobs, Stages & Pipelines** 📋

### **Stages:**

```yaml
# Define execution order
stages:
  - .pre              # Always first (GitLab 12.5+)
  - build
  - test
  - security
  - deploy
  - .post             # Always last (GitLab 12.5+)
  
# Jobs in same stage run in parallel
# Stages run sequentially
```

### **Job Dependencies:**

```yaml
# Sequential dependencies (traditional)
stages:
  - build
  - test
  - deploy
  
build:
  stage: build
  script: make build
  
test:
  stage: test
  script: make test
  # Waits for all 'build' stage jobs
  
# Directed Acyclic Graph (DAG) - faster
build:
  stage: build
  script: make build
  
test:
  stage: test
  script: make test
  needs: [build]      # Only waits for 'build'job
  
lint:
  stage: test
  script: make lint
  needs: []           # Runs immediately
```

### **Parallel Jobs:**

```yaml
# Method 1: parallel keyword
test:
  stage: test
  script: npm test
  parallel: 5         # Creates 5 instances
  
# Access instance number
test:
  stage: test
  script:
    - echo "Running test ${CI_NODE_INDEX} of ${CI_NODE_TOTAL}"
    - npm test -- --shard=${CI_NODE_INDEX}/${CI_NODE_TOTAL}
  parallel: 5
  
# Method 2: Matrix strategy
test:
  stage: test
  script: npm test
  parallel:
    matrix:
      - NODE_VERSION: ['14', '16', '18']
        OS: ['linux', 'windows']
  # Creates 6 jobs (3 x 2)
```

### **Hidden Jobs (Templates):**

```yaml
# Hidden job (starts with .)
.deploy-template:
  script:
    - kubectl apply -f k8s/
  only:
    - main
    
# Extend template
deploy-staging:
  extends: .deploy-template
  environment:
    name: staging
    
deploy-production:
  extends: .deploy-template
  environment:
    name: production
  when: manual
```

### **Anchors & Aliases (YAML):**

```yaml
# Define anchor
.test-template: &test-definition
  stage: test
  image: node:18
  before_script:
    - npm ci
    
# Use alias
unit-test:
  <<: *test-definition
  script:
    - npm run test:unit
    
integration-test:
  <<: *test-definition
  script:
    - npm run test:integration
```

---

## **5. GitLab Runners** 🏃

### **Installing Runner (Docker):**

```bash
# Pull GitLab Runner image
docker pull gitlab/gitlab-runner:latest

# Start runner
docker run -d \
  --name gitlab-runner \
  --restart always \
  -v /srv/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```

### **Registering Runner:**

```bash
# Get registration token from:
# GitLab → Settings → CI/CD → Runners

# Register runner
docker exec -it gitlab-runner gitlab-runner register

# Follow prompts:
# GitLab URL: https://gitlab.com
# Registration token: <your-token>
# Description: my-docker-runner
# Tags: docker,linux
# Executor: docker
# Default image: alpine:latest
```

### **Runner Configuration:**

```toml
# /etc/gitlab-runner/config.toml

concurrent = 4              # Max parallel jobs
check_interval = 0          # Job check frequency

[[runners]]
  name = "my-docker-runner"
  url = "https://gitlab.com"
  token = "RUNNER_TOKEN"
  executor = "docker"
  
  [runners.docker]
    image = "alpine:latest"
    privileged = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "us-east-1"
```

### **Executor Types:**

```yaml
# Shell Executor
build:
  tags:
    - shell
  script:
    - mvn clean package
    
# Docker Executor
build:
  image: maven:3.8-openjdk-17
  tags:
    - docker
  script:
    - mvn clean package
    
# Kubernetes Executor
build:
  image: maven:3.8-openjdk-17
  tags:
    - kubernetes
  script:
    - mvn clean package
    
# Docker Machine Executor (auto-scaling)
# Configured in config.toml
[[runners]]
  executor = "docker+machine"
  [runners.machine]
    IdleCount = 2
    IdleTime = 1800
    MaxBuilds = 10
    MachineName = "auto-scale-%s"
    MachineDriver = "amazonec2"
```

### **Runner Selectionby Tags:**

```yaml
# .gitlab-ci.yml

build:
  tags:
    - docker
    - linux
  script:
    - make build
    
test-windows:
  tags:
    - windows
    - docker
  script:
    - run-tests.bat
    
deploy:
  tags:
    - deploy
    - production
  script:
    - ./deploy.sh
```

---

## **6. Advanced Pipeline Features** ⚡

### **Rules (Advanced Conditions):**

```yaml
# Replace only/except with rules
deploy:
  stage: deploy
  script: ./deploy.sh
  rules:
    # Run on main branch
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: always
    # Run on tags
    - if: '$CI_COMMIT_TAG'
      when: manual
    # Run on merge requests
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
    # Run if files changed
    - changes:
        - src/**/*
      when: always
    # Default
    - when: never
```

```yaml
# Complex rules
deploy:
  rules:
    # AND condition
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
    # OR with multiple ifs
    - if: '$CI_COMMIT_BRANCH == "develop"'
    - if: '$CI_COMMIT_TAG'
    # File changes with branch
    - if: '$CI_COMMIT_BRANCH == "main"'
      changes:
        - Dockerfile
        - k8s/**/*
```

### **Merge Request Pipelines:**

```yaml
# Run only for merge requests
test-mr:
  script: npm test
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    
# Different behavior for MR vs branch
deploy:
  script: ./deploy.sh $ENVIRONMENT
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      variables:
        ENVIRONMENT: "review"
    - if: '$CI_COMMIT_BRANCH == "main"'
      variables:
        ENVIRONMENT: "production"
```

### **Review Apps:**

```yaml
# Temporary environment for merge requests
review:
  stage: deploy
  script:
    - kubectl create namespace review-$CI_MERGE_REQUEST_IID || true
    - helm upgrade --install review-$CI_MERGE_REQUEST_IID ./chart --namespace review-$CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    url: https://review-$CI_MERGE_REQUEST_IID.example.com
    on_stop: stop-review
    auto_stop_in: 1 day
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    
stop-review:
  stage: deploy
  script:
    - helm uninstall review-$CI_MERGE_REQUEST_IID --namespace review-$CI_MERGE_REQUEST_IID
    - kubectl delete namespace review-$CI_MERGE_REQUEST_IID
  environment:
    name: review/$CI_MERGE_REQUEST_IID
    action: stop
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      when: manual
```

### **Dynamic Child Pipelines:**

```yaml
# Parent pipeline
generate-config:
  stage: build
  script:
    - python generate-pipeline.py > child-pipeline.yml
  artifacts:
    paths:
      - child-pipeline.yml
      
trigger-child:
  stage: deploy
  trigger:
    include:
      - artifact: child-pipeline.yml
        job: generate-config
```

### **Multi-Project Pipelines:**

```yaml
# Trigger pipeline in another project
trigger-downstream:
  stage: deploy
  trigger:
    project: company/downstream-project
    branch: main
    strategy: depend        # Wait for completion
    
# Pass variables
trigger-with-vars:
  trigger:
    project: company/other-project
    forward:
      yaml_variables: true
      pipeline_variables: true
```

### **Include External Config:**

```yaml
# Include from same repo
include:
  - local: '/templates/build.yml'
  
# Include from another project
include:
  - project: 'company/ci-templates'
    file: '/templates/deploy.yml'
    ref: main
    
# Include from URL
include:
  - remote: 'https://example.com/ci-template.yml'
  
# Include template
include:
  - template: Auto-DevOps.gitlab-ci.yml
  
# Multiple includes
include:
  - local: '/templates/build.yml'
  - local: '/templates/test.yml'
  - project: 'company/ci-templates'
    file:
      - '/templates/deploy.yml'
      - '/templates/security.yml'
```

---

## **7. Docker & Kubernetes Integration** 🐳

### **Docker-in-Docker (dind):**

```yaml
build-docker:
  image: docker:latest
  services:
    - docker:dind
  variables:
    DOCKER_DRIVER: overlay2
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

### **GitLab Container Registry:**

```yaml
variables:
  # GitLab provides these automatically
  CI_REGISTRY: registry.gitlab.com
  CI_REGISTRY_IMAGE: registry.gitlab.com/username/project
  CI_REGISTRY_USER: gitlab-ci-token
  CI_REGISTRY_PASSWORD: $CI_JOB_TOKEN
  
docker-build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  script:
    # Login (automatic with CI_JOB_TOKEN)
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    
    # Build
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG .
    - docker build -t $CI_REGISTRY_IMAGE:latest .
    
    # Push
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG
    - docker push $CI_REGISTRY_IMAGE:latest
  only:
    - tags
```

### **Kubernetes Deployment:**

```yaml
deploy-k8s:
  image: bitnami/kubectl:latest
  stage: deploy
  script:
    # Update deployment
    - kubectl set image deployment/myapp \
        myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
        --namespace=production
    
    # Wait for rollout
    - kubectl rollout status deployment/myapp -n production
    
    # Verify
    - kubectl get pods -n production
  environment:
    name: production
    url: https://myapp.com
    kubernetes:
      namespace: production
  only:
    - main
```

### **Helm Deployment:**

```yaml
deploy-helm:
  image: alpine/helm:latest
  stage: deploy
  script:
    # Add repo
    - helm repo add myrepo https://charts.example.com
    - helm repo update
    
    # Deploy
    - helm upgrade --install myapp myrepo/myapp \
        --namespace production \
        --set image.tag=$CI_COMMIT_SHA \
        --set image.repository=$CI_REGISTRY_IMAGE \
        --wait \
        --timeout 5m
  environment:
    name: production
    url: https://myapp.com
```

### **GitLab Agent for Kubernetes:**

```yaml
# .gitlab/agents/production/config.yaml
ci_access:
  projects:
    - id: company/myapp
      
# Use in pipeline
deploy:
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context company/myapp:production
    - kubectl apply -f k8s/
    - kubectl rollout status deployment/myapp
```

---

## **8. Auto DevOps** 🤖

### **What is Auto DevOps?**

Auto DevOps provides zero-configuration CI/CD pipelines with built-in best practices.

### **Enable Auto DevOps:**

```
Project → Settings → CI/CD → Auto DevOps
☑️ Default to Auto DevOps pipeline
```

### **Auto DevOps Features:**

```yaml
Automatic:
✅ Build (detect language)
✅ Test
✅ Code Quality (Code Climate)
✅ SAST (Static Security)
✅ Dependency Scanning
✅ Container Scanning
✅ DAST (Dynamic Security)
✅ License Compliance
✅ Review Apps
✅ Staging Deployment
✅ Production Deployment
✅ Monitoring
```

### **Customize Auto DevOps:**

```yaml
# .gitlab-ci.yml
include:
  - template: Auto-DevOps.gitlab-ci.yml
  
# Override specific jobs
test:
  script:
    - custom-test-command
    
# Disable specific jobs
code_quality:
  rules:
    - when: never
```

### **Auto DevOps Variables:**

```yaml
# Project → Settings → CI/CD → Variables

# Kubernetes
KUBE_NAMESPACE: production
KUBE_INGRESS_BASE_DOMAIN: example.com

# Database
POSTGRES_ENABLED: "true"
POSTGRES_USER: myapp
POSTGRES_DB: myapp_production

# Application
REPLICAS: 3
ADDITIONAL_HOSTS: app.example.com
HELM_UPGRADE_EXTRA_ARGS: --set resources.limits.memory=2Gi

# Security
SAST_EXCLUDED_PATHS: spec,test,tests
CODE_QUALITY_DISABLED: "false"
```

---

## **9. Real-World Pipeline Examples** 📋

### **Example 1: Node.js  Microservice**

```yaml
image: node:18-alpine

variables:
  NPM_CONFIG_CACHE: "$CI_PROJECT_DIR/.npm"
  DOCKER_DRIVER: overlay2
  
stages:
  - build
  - test
  - security
  - docker
  - deploy
  
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
    - .npm/
    
# Build
build:
  stage: build
  script:
    - npm ci
    - npm run build
  artifacts:
    paths:
      - dist/
    expire_in: 1 day
    
# Tests
unit-test:
  stage: test
  script:
    - npm run test:unit -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml
        
lint:
  stage: test
  script:
    - npm run lint
    
# Security
dependency-scan:
  stage: security
  script:
    - npm audit --audit-level=moderate
  allow_failure: true
  
sast:
  stage: security
  image: returntocorp/semgrep
  script:
    - semgrep --config=auto .
  allow_failure: true
  
# Docker
docker-build:
  stage: docker
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA
  only:
    - main
    - tags
    
# Deploy
deploy-staging:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n staging
    - kubectl rollout status deployment/myapp -n staging
  environment:
    name: staging
    url: https://staging.myapp.com
  only:
    - main
    
deploy-production:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl set image deployment/myapp myapp=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA -n production
    - kubectl rollout status deployment/myapp -n production
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - tags
```

### **Example 2: Python Django Application**

```yaml
image: python:3.11

variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"
  POSTGRES_DB: testdb
  POSTGRES_USER: testuser
  POSTGRES_PASSWORD: testpass
  DATABASE_URL: "postgresql://testuser:testpass@postgres:5432/testdb"
  
stages:
  - test
  - build
  - deploy
  
cache:
  paths:
    - .cache/pip
    - venv/
    
before_script:
  - python -V
  - pip install virtualenv
  - virtualenv venv
  - source venv/bin/activate
  - pip install -r requirements.txt
  
lint:
  stage: test
  script:
    - flake8 .
    - black --check .
    - mypy .
    
test:
  stage: test
  services:
    - postgres:14
  script:
    - python manage.py migrate
    - pytest --cov=. --cov-report=xml --junitxml=junit.xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
        
build:
  stage: build
  image: docker:latest
  services:
    - docker:dind
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  only:
    - main
    
deploy:
  stage: deploy
  script:
    - kubectl set image deployment/django-app django-app=$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  environment:
    name: production
  only:
    - main
```

---

## **10. Best Practices** ⭐

### **Pipeline Optimization:**

```yaml
# ✅ Use needs for DAG
build:
  stage: build
  script: npm run build
  
test:
  stage: test
  script: npm test
  needs: [build]       # Don't wait for entire stage
  
# ✅ Cache dependencies
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - node_modules/
  policy: pull-push
  
# ✅ Use artifacts selectively
build:
  artifacts:
    paths:
      - dist/
    expire_in: 1 day   # Auto cleanup
    
# ✅ Fail fast
stages:
  - lint             # Fast
  - test             # Medium
  - build            # Slower
  - deploy           # Slowest
```

### **Security:**

```yaml
# ✅ Use protected variables
# Project → Settings → CI/CD → Variables
# ☑️ Protect variable
# ☑️ Mask variable

# ✅ Enable security scanning
include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  
# ✅ Use signed commits
# git config --global commit.gpgsign true

# ✅ Protect branches
# Settings → Repository → Protected Branches
```

### **Resource Management:**

```yaml
# ✅ Set appropriate timeouts
default:
  timeout: 1h
  
long-job:
  timeout: 3h
  
# ✅ Use retry for flaky tests
test:
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure
      
# ✅ Use allow_failure wisely
security-scan:
  allow_failure: true  # Don't block pipeline
```

---

## **11. Troubleshooting** 🔧

### **Common Issues:**

```yaml
# Issue: Job stuck in pending
# Solutions:
# 1. Check runner availability
#    Settings → CI/CD → Runners
# 2. Check runner tags
#    job: tags: [docker]
#    runner: tags: docker
# 3. Check concurrent limit in runner config

# Issue: Cache not working
cache:
  key: ${CI_COMMIT_REF_SLUG}  # Make sure key is consistent
  paths:
    - node_modules/
  policy: pull-push             # Not just pull or push

# Issue: Artifacts not found
build:
  artifacts:
    paths:
      - dist/               # Check path exists
    untracked: false       # Include git-ignored files if needed
    
test:
  dependencies:
    - build                # Explicitly list dependencies

# Issue: Docker login fails
before_script:
  - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  # Not: docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: GitLab CI/CD vs Jenkins?**
```
GitLab CI/CD:
✅ Integrated (all-in-one platform)
✅ Built-in container registry
✅ Auto DevOps (zero config)
✅ Kubernetes native
✅ SaaS or self-hosted
✅ YAML configuration
✅ Generous free tier

Jenkins:
✅ More plugins (1800+)
✅ Highly customizable
✅ Free and open source
✅ Mature ecosystem
✅ Groovy pipelines
❌ Separate tool (not integrated)
❌ Complex setup

Choose GitLab if: Integrated platform, containerized workflows
Choose Jenkins if: Need maximum flexibility, existing investment
```

### **Q2: Explain GitLab pipeline structure.**
```
Components:
1. Stages: Execution order (build → test → deploy)
2. Jobs: Work units within stages
3. Runners: Execute jobs
4. Artifacts: Files passed between jobs
5. Cache: Dependencies storage

Flow:
Code Push → .gitlab-ci.yml parsed → Pipeline created →
Jobs assigned to runners → Runners execute → Results reported

Example:
stages:
  - build
  - test
  
build-job:
  stage: build
  script: npm run build
  
test-job:
  stage: test
  script: npm test
```

### **Commands:**

```bash
# Validate .gitlab-ci.yml
# UI → CI/CD → Pipelines → CI Lint

# Trigger pipeline via API
curl -X POST \
  --form token=TRIGGER_TOKEN \
  --form ref=main \
  https://gitlab.com/api/v4/projects/PROJECT_ID/trigger/pipeline

# Cancel pipeline
curl -X POST \
  --header "PRIVATE-TOKEN: $ACCESS_TOKEN" \
  https://gitlab.com/api/v4/projects/PROJECT_ID/pipelines/PIPELINE_ID/cancel
```

---

## **Next Steps** 📚

- **[CI/CD Fundamentals](../CI_CD_Fundamentals.md)** - Core concepts
- **[Jenkins](../Jenkins/)** - Alternative CI/CD platform
- **[GitHub Actions](../GitHubActions/)** - Cloud-native CI/CD

---

**🦊 Master GitLab CI/CD, Master Integrated DevOps!**

*GitLab CI/CD offers the most integrated DevOps experience - from source code to production, all in one platform.*
