# **GitHub Actions - Complete Guide** 🚀⚡

**Master Cloud-Native CI/CD with GitHub's Integrated Automation Platform**

---

## **Table of Contents** 📑
1. [GitHub Actions Overview](#1-github-actions-overview)
2. [GitHub Actions Architecture](#2-github-actions-architecture)
3. [Workflow Syntax](#3-workflow-syntax)
4. [Events & Triggers](#4-events--triggers)
5. [Jobs, Steps & Actions](#5-jobs-steps--actions)
6. [Runners](#6-runners)
7. [Advanced Workflows](#7-advanced-workflows)
8. [Actions Marketplace](#8-actions-marketplace)
9. [Real-World Workflow Examples](#9-real-world-workflow-examples)
10. [Best Practices](#10-best-practices)
11. [Troubleshooting](#11-troubleshooting)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. GitHub Actions Overview** 🎯

### **What is GitHub Actions?**

**GitHub Actions** is GitHub's built-in CI/CD and automation platform. It enables you to automate workflows directly in your GitHub repository using event-driven triggers.

### **Key Features:**

```
✅ Native GitHub integration
✅ Event-driven (push, PR, issues, schedule)
✅ Reusable workflows
✅ Actions Marketplace (12,000+ actions)
✅ Matrix builds
✅ GitHub-hosted runners (Linux, Windows, macOS)
✅ Self-hosted runners
✅ Secrets management
✅ Environments & approvals
✅ Free tier (2,000 min/month for private repos)
```

### **Why GitHub Actions?**

| Feature | GitHub Actions | Competitors |
|---------|----------------|-------------|
| **Integration** | Native GitHub | External tools |
| **Setup** | Zero install | Separate service |
| **Actions** | 12,000+ reusable | Limited plugins |
| **Runners** | Cloud (3 OS) + Self-hosted | Varies |
| **Triggers** | 40+ event types | Fewer events |
| **Cost** | Generous free tier | Varies |
| **Matrix** | Built-in | Manual setup |

### **GitHub Actions vs Competitors:**

```mermaid
graph TB
    A[GitHub Repository] --> B[GitHub Actions]
    
    B --> C[Push Events]
    B --> D[Pull Requests]
    B --> E[Issues]
    B --> F[Schedules]
    B --> G[Manual]
    B --> H[External Events]
    
    B --> I[GitHub-hosted Runners]
    B --> J[Self-hosted Runners]
    
    style B fill:#2088FF
```

---

## **2. GitHub Actions Architecture** 🏗️

### **Components:**

```mermaid
graph TB
    Event[Event Trigger] -->|Activates| Workflow[Workflow]
    
    Workflow -->|Contains| Job1[Job 1]
    Workflow -->|Contains| Job2[Job 2]
    Workflow -->|Contains| Job3[Job 3]
    
    Job1 -->|Runs on| Runner1[GitHub Runner Linux]
    Job2 -->|Runs on| Runner2[GitHub Runner Windows]
    Job3 -->|Runs on| Runner3[Self-hosted Runner]
    
    Job1 --> Step1[Step: Checkout]
    Job1 --> Step2[Step: Setup Node]
    Job1 --> Step3[Step: Run Tests]
    
    Step2 -->|Uses| Action[Action from Marketplace]
    
    style Workflow fill:#2088FF
```

### **Core Components:**

#### **1. Workflows**
```
Location: .github/workflows/*.yml
Purpose: Define automation
Trigger: Events (push, PR, schedule, etc.)
Contains: One or more jobs

Example: .github/workflows/ci.yml
```

#### **2. Events**
```
Types:
  - push (code pushed)
  - pull_request (PR opened/updated)
  - schedule (cron)
  - workflow_dispatch (manual)
  - release (new release)
  - issues (issue created/updated)
  - 40+ more events

Purpose: Trigger workflows automatically
```

#### **3. Jobs**
```
Purpose: Group related steps
Runs on: Specific runner
Dependencies: Run in parallel or sequential
Environment: Optional environment (staging, prod)

Default: Parallel execution
Sequential: use 'needs'
```

#### **4. Steps**
```
Purpose: Individual tasks
Types:
  - Run commands (run:)
  - Use actions (uses:)
  
Execution: Sequential within a job
```

#### **5. Actions**
```
Purpose: Reusable automation units
Types:
  - JavaScript actions
  - Docker actions
  - Composite actions
  
Source:
  - GitHub Marketplace
  - Your repository
  - Public repositories
```

#### **6. Runners**
```
GitHub-hosted:
  - ubuntu-latest (Ubuntu 22.04)
  - windows-latest (Windows Server 2022)
  - macos-latest (macOS 12)
  - Free tier: 2,000 min/month (private repos)
  
Self-hosted:
  - Your own infrastructure
  - Custom OS/tools
  - Unlimited minutes
  - Cost: infrastructure only
```

### **Workflow Execution Flow:**

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant GitHub as GitHub Server
    participant Runner as GitHub Runner
    participant Action as GitHub Action
    
    Dev->>GitHub: Push code
    GitHub->>GitHub: Detect event (push)
    GitHub->>GitHub: Parse workflow YAML
    GitHub->>GitHub: Create workflow run
    GitHub->>Runner: Assign job
    Runner->>Runner: Setup environment
    Runner->>Action: Execute action
    Action->>Runner: Return result
    Runner->>GitHub: Upload artifacts
    GitHub->>Dev: Notify status
```

---

## **3. Workflow Syntax** 📝

### **Basic Structure:**

```yaml
# .github/workflows/ci.yml

name: CI                       # Workflow name

on: [push, pull_request]      # Events

jobs:                         # Jobs collection
  build:                      # Job ID
    runs-on: ubuntu-latest    # Runner
    
    steps:                    # Steps collection
      - name: Checkout code   # Step name
        uses: actions/checkout@v4
        
      - name: Run tests
        run: npm test
```

### **Complete Example:**

```yaml
name: CI/CD Pipeline

# Events
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'       # Weekly
  workflow_dispatch:          # Manual trigger

# Environment variables
env:
  NODE_VERSION: '18'
  REGISTRY: ghcr.io

# Jobs
jobs:
  # Job 1: Build and Test
  test:
    name: Test on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Run linter
        run: npm run lint
        
      - name: Run tests
        run: npm test -- --coverage
        
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json
          
  # Job 2: Build Docker image
  docker:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
      
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:latest
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            
  # Job 3: Deploy
  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: docker
    environment:
      name: production
      url: https://myapp.com
      
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }}
          kubectl rollout status deployment/myapp
```

### **Workflow Keywords:**

```yaml
name: Workflow Name             # Display name

on:                             # Event triggers
  push:
    branches: [main]
    paths:                      # Only if these files change
      - 'src/**'
    paths-ignore:               # Ignore these files
      - '**.md'
  pull_request:
    types: [opened, synchronize]
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:            # Manual trigger
    inputs:                     # Manual inputs
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'

env:                            # Global environment variables
  NODE_VERSION: '18'

defaults:                       # Global defaults
  run:
    shell: bash
    working-directory: ./src

jobs:
  job-name:
    name: Display Name
    runs-on: ubuntu-latest      # Runner
    timeout-minutes: 60         # Max duration
    
    strategy:                   # Matrix builds
      matrix:
        os: [ubuntu, windows]
        node: [16, 18]
      fail-fast: true
      max-parallel: 2
      
    env:                        # Job environment variables
      JOB_VAR: value
      
    if: github.ref == 'refs/heads/main'  # Conditional execution
    
    needs: [previous-job]       # Job dependencies
    
    permissions:                # Token permissions
      contents: read
      packages: write
      
    environment:                # Deployment environment
      name: production
      url: https://example.com
      
    outputs:                    # Job outputs
      output-name: ${{ steps.step-id.outputs.output-name }}
      
    steps:
      - name: Step Name
        id: step-id
        uses: actions/checkout@v4      # Use action
        with:                          # Action inputs
          ref: main
          
      - name: Run command
        run: |                         # Shell commands
          npm install
          npm test
        shell: bash
        working-directory: ./app
        env:                           # Step environment
          STEP_VAR: value
          
      - name: Conditional step
        if: success()                  # Run if previous steps succeeded
        run: echo "Success!"
        
      - name: Continue on error
        run: npm audit
        continue-on-error: true
        
    services:                   # Service containers
      postgres:
        image: postgres:14
        env:
          POSTGRES_PASSWORD: password
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
```

---

## **4. Events & Triggers** ⚡

### **Common Events:**

```yaml
# Push event
on:
  push:
    branches:
      - main
      - 'releases/**'
    tags:
      - v1.*
    paths:
      - 'src/**'
      - '**.js'
      
# Pull request
on:
  pull_request:
    types: [opened, synchronize, reopened]
    branches:
      - main
      
# Multiple events
on: [push, pull_request, workflow_dispatch]

# Schedule (cron)
on:
  schedule:
    - cron: '0 0 * * *'         # Daily at midnight
    - cron: '0 */6 * * *'       # Every 6 hours
    
# Manual trigger with inputs
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment'
        required: true
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version'
        required: false
        default: 'latest'
        
# Release
on:
  release:
    types: [published, created]
    
# Issues
on:
  issues:
    types: [opened, labeled]
    
# Repository dispatch (external API)
on:
  repository_dispatch:
    types: [deploy]
    
# Workflow call (reusable)
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
```

### **Event Filters:**

```yaml
on:
  push:
    # Branch filters
    branches:
      - main
      - 'releases/**'
      - '!experiment-*'
      
    # Tag filters
    tags:
      - v1.*
      - '!v1.0.0-beta'
      
    # Path filters
    paths:
      - 'src/**'
      - '**.js'
    paths-ignore:
      - '**.md'
      - 'docs/**'
```

---

## **5. Jobs, Steps & Actions** 📋

### **Job Dependencies:**

```yaml
jobs:
  # Run in parallel (default)
  lint:
    runs-on: ubuntu-latest
    steps:
      - run: npm run lint
      
  test:
    runs-on: ubuntu-latest
    steps:
      - run: npm test
      
  # Sequential with needs
  build:
    runs-on: ubuntu-latest
    needs: [lint, test]              # Runs after lint AND test
    steps:
      - run: npm run build
      
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    steps:
      - run: ./deploy.sh staging
      
  deploy-production:
    runs-on: ubuntu-latest
    needs: [build, deploy-staging]
    if: github.ref == 'refs/heads/main'
    steps:
      - run: ./deploy.sh production
```

### **Matrix Strategy:**

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [16, 18, 20]
        # Creates 9 jobs (3 OS × 3 Node versions)
        
      fail-fast: false             # Continue all jobs if one fails
      max-parallel: 3              # Limit concurrent jobs
      
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
      
# Matrix with include/exclude
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu, windows]
        node: [16, 18, 20]
        include:
          # Add extra job
          - os: macos
            node: 18
        exclude:
          # Remove combination
          - os: windows
            node: 20
```

### **Using Actions:**

```yaml
steps:
  # Public action (GitHub Marketplace)
  - name: Checkout code
    uses: actions/checkout@v4
    with:
      fetch-depth: 0              # Action inputs
      
  # Action from different repo
  - uses: docker/build-push-action@v5
    with:
      context: .
      push: true
      tags: myapp:latest
      
  # Local action (same repo)
  - uses: ./.github/actions/my-action
    with:
      param: value
      
  # Docker action
  - uses: docker://alpine:latest
    with:
      entrypoint: /bin/sh
      args: -c "echo hello"
```

### **Step Outputs:**

```yaml
jobs:
  job1:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.get-version.outputs.version }}
      
    steps:
      - name: Get version
        id: get-version
        run: |
          VERSION=$(cat package.json | jq -r .version)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          
  job2:
    runs-on: ubuntu-latest
    needs: job1
    steps:
      - name: Use version
        run: echo "Version is ${{ needs.job1.outputs.version }}"
```

---

## **6. Runners** 🏃

### **GitHub-Hosted Runners:**

```yaml
jobs:
  # Ubuntu (most common)
  ubuntu-job:
    runs-on: ubuntu-latest    # Ubuntu 22.04
    # Alternatives: ubuntu-20.04, ubuntu-22.04
    
  # Windows
  windows-job:
    runs-on: windows-latest   # Windows Server 2022
    # Alternatives: windows-2019, windows-2022
    
  # macOS
  macos-job:
    runs-on: macos-latest     # macOS 12 (Monterey)
    # Alternatives: macos-11, macos-12, macos-13
```

### **Pre-installed Software:**

```
Ubuntu runners include:
  - Git, Docker, kubectl, helm
  - Node.js (multiple versions via nvm)
  - Python, Ruby, Go, Java
  - AWS CLI, Azure CLI, gcloud
  - Build tools (make, gcc, etc.)
  
Full list:
https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2204-Readme.md
```

### **Self-Hosted Runners:**

```bash
# Add self-hosted runner
# Repository → Settings → Actions → Runners → New self-hosted runner

# Download runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.311.0.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.311.0/actions-runner-linux-x64-2.311.0.tar.gz
tar xzf ./actions-runner-linux-x64-2.311.0.tar.gz

# Configure
./config.sh --url https://github.com/myorg/myrepo --token TOKEN

# Run
./run.sh

# Install as service
sudo ./svc.sh install
sudo ./svc.sh start
```

```yaml
# Use self-hosted runner
jobs:
  build:
    runs-on: self-hosted
    # Or with labels
    runs-on: [self-hosted, linux, x64, gpu]
    
    steps:
      - uses: actions/checkout@v4
      - run: make build
```

### **Runner Groups (Enterprise):**

```yaml
jobs:
  deploy:
    runs-on: 
      group: production-runners
      labels: [linux, docker]
```

---

## **7. Advanced Workflows** ⚡

### **Reusable Workflows:**

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      version:
        required: false
        type: string
        default: 'latest'
    secrets:
      deploy-key:
        required: true
    outputs:
      url:
        description: "Deployment URL"
        value: ${{ jobs.deploy.outputs.url }}
        
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    outputs:
      url: ${{ steps.deploy.outputs.url }}
      
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        id: deploy
        run: |
          ./deploy.sh ${{ inputs.environment }} ${{ inputs.version }}
          echo "url=https://${{ inputs.environment }}.example.com" >> $GITHUB_OUTPUT
```

```yaml
# .github/workflows/main.yml (caller)
name: Main

on: [push]

jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      version: '1.0.0'
    secrets:
      deploy-key: ${{ secrets.DEPLOY_KEY }}
      
  deploy-production:
    needs: deploy-staging
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: production
    secrets:
      deploy-key: ${{ secrets.DEPLOY_KEY }}
```

### **Composite Actions:**

```yaml
# .github/actions/setup-app/action.yml
name: 'Setup Application'
description: 'Setup Node.js and install dependencies'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '18'

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      shell: bash
      
    - name: Cache build
      uses: actions/cache@v3
      with:
        path: dist
        key: build-${{ github.sha }}
```

```yaml
# Use composite action
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-app
        with:
          node-version: '20'
      - run: npm run build
```

### **Matrix with Dynamic Values:**

```yaml
jobs:
  generate-matrix:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          MATRIX='{"include":[{"os":"ubuntu-latest","node":"16"},{"os":"ubuntu-latest","node":"18"}]}'
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT
          
  test:
    needs: generate-matrix
    runs-on: ${{ matrix.os }}
    strategy:
      matrix: ${{ fromJSON(needs.generate-matrix.outputs.matrix) }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

### **Approval Workflows:**

```yaml
# Create environment with protection rules
# Repository → Settings → Environments → New environment
# ☑️ Required reviewers

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production      # Requires approval
    steps:
      - name: Deploy
        run: ./deploy.sh
```

---

## **8. Actions Marketplace** 🏪

### **Popular Actions:**

```yaml
# Checkout code
- uses: actions/checkout@v4
  with:
    fetch-depth: 0              # Full git history
    submodules: true            # Include submodules

# Setup languages
- uses: actions/setup-node@v4
  with:
    node-version: '18'
    cache: 'npm'
    
- uses: actions/setup-python@v4
  with:
    python-version: '3.11'
    cache: 'pip'
    
- uses: actions/setup-java@v3
  with:
    distribution: 'temurin'
    java-version: '17'
    
# Caching
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      ${{ runner.os }}-node-
      
# Upload artifacts
- uses: actions/upload-artifact@v3
  with:
    name: dist
    path: dist/
    retention-days: 7
    
# Download artifacts
- uses: actions/download-artifact@v3
  with:
    name: dist
    path: dist/
    
# Docker
- uses: docker/setup-buildx-action@v3
- uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}
    
- uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ghcr.io/${{ github.repository }}:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    
# Kubernetes
- uses: azure/setup-kubectl@v3
- uses: azure/k8s-set-context@v3
  with:
    method: kubeconfig
    kubeconfig: ${{ secrets.KUBE_CONFIG }}
    
# AWS
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/MyRole
    aws-region: us-east-1
    
# Code coverage
- uses: codecov/codecov-action@v3
  with:
    files: ./coverage/coverage-final.json
    
# GitHub releases
- uses: softprops/action-gh-release@v1
  with:
    files: |
      dist/*
      README.md
```

### **Creating Custom Actions:**

```yaml
# .github/actions/my-action/action.yml
name: 'My Custom Action'
description: 'Does something useful'

inputs:
  input-name:
    description: 'Input description'
    required: true
    default: 'default-value'
    
outputs:
  output-name:
    description: 'Output description'
    value: ${{ steps.step-id.outputs.output-name }}
    
runs:
  using: 'composite'
  steps:
    - name: Do something
      id: step-id
      run: |
        echo "Doing something with ${{ inputs.input-name }}"
        echo "output-name=result" >> $GITHUB_OUTPUT
      shell: bash
```

---

## **9. Real-World Workflow Examples** 📋

### **Example 1: Node.js CI/CD**

```yaml
name: Node.js CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  NODE_VERSION: '18'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        node: [16, 18, 20]
        
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
          cache: 'npm'
          
      - name: Install dependencies
        run: npm ci
        
      - name: Lint
        run: npm run lint
        
      - name: Test
        run: npm test -- --coverage
        
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        if: matrix.node == '18'
        
  build:
    runs-on: ubuntu-latest
    needs: test
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    
    permissions:
      contents: read
      packages: write
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
          
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/${{ env.IMAGE_NAME }}:latest
            ghcr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          
  deploy:
    runs-on: ubuntu-latest
    needs: build
    environment: production
    
    steps:
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ env.IMAGE_NAME }}:${{ github.sha }}
          kubectl rollout status deployment/myapp
```

### **Example 2: Python Django**

```yaml
name: Django CI/CD

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:14
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
          
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379
          
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
      REDIS_URL: redis://localhost:6379
      
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
          
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt
          
      - name: Lint
        run: |
          flake8 .
          black --check .
          mypy .
          
      - name: Run migrations
        run: python manage.py migrate
        
      - name: Run tests
        run: |
          pytest --cov=. --cov-report=xml --junitxml=junit.xml
          
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        
  deploy:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Heroku
        uses: akhileshns/heroku-deploy@v3.12.14
        with:
          heroku_api_key: ${{ secrets.HEROKU_API_KEY }}
          heroku_app_name: my-django-app
          heroku_email: ${{ secrets.HEROKU_EMAIL }}
```

---

## **10. Best Practices** ⭐

### **Security:**

```yaml
# ✅ Use GITHUB_TOKEN (auto-generated)
- uses: actions/checkout@v4
  with:
    token: ${{ secrets.GITHUB_TOKEN }}

# ✅ Pin actions to SHA (not tag)
- uses: actions/checkout@f43a0e5ff2bd294095638e18286ca9a3d1956744  # v4.0.0

# ✅ Limit permissions
permissions:
  contents: read
  packages: write
  
# ✅ Use environments for secrets
jobs:
  deploy:
    environment: production     # Secrets scoped to environment
    
# ❌ Don't expose secrets in logs
- run: echo "Secret is ${{ secrets.MY_SECRET }}"  # WRONG!

# ✅ Use encrypted secrets
# Repository → Settings → Secrets → Actions
```

### **Performance:**

```yaml
# ✅ Use caching
- uses: actions/cache@v3
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
    
# ✅ Use artifacts efficiently
- uses: actions/upload-artifact@v3
  with:
    name: dist
    path: dist/
    retention-days: 1          # Don't keep forever
    
# ✅ Fail fast
strategy:
  fail-fast: true
  matrix:
    os: [ubuntu, windows, macos]
    
# ✅ Use concurrency to cancel old runs
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```

### **Code Organization:**

```yaml
# ✅ Use reusable workflows
- uses: ./.github/workflows/reusable-test.yml

# ✅ Use composite actions
- uses: ./.github/actions/setup-app

# ✅ Use matrix for multiple configurations
strategy:
  matrix:
    node: [16, 18, 20]
    os: [ubuntu, windows, macos]
```

---

## **11. Troubleshooting** 🔧

### **Common Issues:**

```yaml
# Issue: Workflow not triggering
# Solution: Check event filters
on:
  push:
    branches:
      - main              # Must match exact branch name
      
# Issue: Job skipped
# Solution: Check 'if' conditions
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'  # Must match exactly

# Issue: Action not found
# Solution: Check action exists and version is correct
- uses: actions/checkout@v4    # ✅ Correct
- uses: actions/checkout@v5    # ❌ Doesn't exist yet

# Issue: Permission denied
# Solution: Add required permissions
permissions:
  contents: write
  packages: write
  
# Issue: Artifact not found
# Solution: Ensure job dependencies
jobs:
  build:
    steps:
      - uses: actions/upload-artifact@v3
        with:
          name: dist
          path: dist/
          
  deploy:
    needs: build              # Must depend on build
    steps:
      - uses: actions/download-artifact@v3
        with:
          name: dist
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: GitHub Actions vs Jenkins?**
```
GitHub Actions:
✅ Native GitHub integration
✅ Zero setup (cloud-hosted)
✅ Event-driven (40+ events)
✅ 12,000+ marketplace actions
✅ Matrix builds built-in
✅ Free tier (2,000 min/month)
✅ Multi-OS (Linux, Windows, macOS)

Jenkins:
✅ More flexible/customizable
✅ Self-hosted (full control)
✅ 1,800+ plugins
✅ No runtime limits
❌ Requires setup/maintenance
❌ Primarily CI (not Git-integrated)

Choose GitHub Actions if: Using GitHub, want zero setup
Choose Jenkins if: Need maximum flexibility, self-hosted required
```

### **Q2: Explain GitHub Actions workflow structure.**
```
Components:
1. Workflow: .github/workflows/*.yml
2. Events: Triggers (push, PR, schedule)
3. Jobs: Parallel units of work
4. Steps: Sequential tasks
5. Actions: Reusable automation

Example:
on: push                  # Event
jobs:
  build:                  # Job
    runs-on: ubuntu-latest
    steps:                # Steps
      - uses: actions/checkout@v4    # Action
      - run: npm test               # Command
```

### **Q3: What are GitHub Actions secrets?**
```
Encrypted environment variables stored in GitHub.

Types:
- Repository secrets (one repo)
- Organization secrets (all repos)
- Environment secrets (specific environment)

Usage:
${{ secrets.MY_SECRET }}

Best practices:
✅ Use for credentials, tokens, keys
✅ Scope to environment when possible
✅ Rotate regularly
✅ Never echo secrets in logs
❌ Don't commit secrets in code
```

### **Commands:**

```bash
# Trigger workflow via API
curl -X POST \
  -H "Authorization: token $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github.v3+json" \
  https://api.github.com/repos/owner/repo/actions/workflows/ci.yml/dispatches \
  -d '{"ref":"main"}'

# List workflow runs
gh run list --workflow=ci.yml

# Cancel workflow
gh run cancel <run-id>

# View workflow
gh run view <run-id>

# Download artifact
gh run download <run-id> -n artifact-name
```

---

## **Next Steps** 📚

- **[CI/CD Fundamentals](../CI_CD_Fundamentals.md)** - Core CI/CD concepts
- **[Jenkins](../Jenkins/)** - Traditional CI/CD platform
- **[GitLab CI](../GitLabCI/)** - Integrated DevOps platform

---

**🚀 Master GitHub Actions, Master Cloud-Native CI/CD!**

*GitHub Actions makes automation accessible to everyone - from first commit to production deployment, all within GitHub.*
