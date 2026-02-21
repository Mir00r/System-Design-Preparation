# **Kubernetes Commands Cheatsheet** ☸️

**Complete kubectl command reference for Kubernetes cluster management**

---

## **Table of Contents** 📑
1. [kubectl Basics](#1-kubectl-basics)
2. [Cluster Information](#2-cluster-information)
3. [Resource Management (CRUD)](#3-resource-management-crud)
4. [Pod Operations](#4-pod-operations)
5. [Deployment & Scaling](#5-deployment--scaling)
6. [Service & Networking](#6-service--networking)
7. [ConfigMaps & Secrets](#7-configmaps--secrets)
8. [Logs & Debugging](#8-logs--debugging)
9. [Resource Monitoring](#9-resource-monitoring)
10. [Troubleshooting Commands](#10-troubleshooting-commands)
11. [Best Practices](#11-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. kubectl Basics** 🔧

### **Installation**

```bash
# Install on macOS
brew install kubectl

# Install on Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client

# Check kubectl version and server version
kubectl version
```

### **Configuration**

```bash
# View kubeconfig
kubectl config view

# Get current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context <context-name>

# Set default namespace
kubectl config set-context --current --namespace=<namespace>

# Add new cluster
kubectl config set-cluster <cluster-name> --server=https://1.2.3.4:6443

# Set credentials
kubectl config set-credentials <user> --token=<bearer-token>
```

### **kubectl Syntax**

```
kubectl [command] [TYPE] [NAME] [flags]

command: Operation (get, create, apply, delete, etc.)
TYPE: Resource type (pod, service, deployment, etc.)
NAME: Resource name (optional)
flags: Additional options (--namespace, -o yaml, etc.)
```

---

## **2. Cluster Information** 📊

### **Cluster Status**

```bash
# Cluster information
kubectl cluster-info

# Get nodes
kubectl get nodes
kubectl get nodes -o wide

# Describe node
kubectl describe node <node-name>

# Node resource usage
kubectl top nodes

# Check component status (deprecated but useful)
kubectl get componentstatuses
kubectl get cs

# API resources
kubectl api-resources

# API versions
kubectl api-versions
```

### **Namespaces**

```bash
# List namespaces
kubectl get namespaces
kubectl get ns

# Create namespace
kubectl create namespace dev
kubectl create ns prod

# Describe namespace
kubectl describe namespace dev

# Delete namespace (deletes all resources in it!)
kubectl delete namespace dev

# Set default namespace for context
kubectl config set-context --current --namespace=dev
```

---

## **3. Resource Management (CRUD)** 📝

### **Create Resources**

```bash
# Create from file
kubectl create -f deployment.yaml
kubectl create -f https://example.com/manifest.yaml

# Create from multiple files
kubectl create -f ./configs/

# Create resource imperatively
kubectl create deployment nginx --image=nginx:1.21
kubectl create service clusterip my-svc --tcp=80:80
kubectl create configmap my-config --from-literal=key1=value1
kubectl create secret generic my-secret --from-literal=password=secret123

# Apply (create or update)
kubectl apply -f deployment.yaml
kubectl apply -f ./configs/
```

### **Get Resources**

```bash
# Get common resources
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get svc  # Shorthand

# Get all resources
kubectl get all
kubectl get all -n kube-system

# Get with wide output
kubectl get pods -o wide
kubectl get nodes -o wide

# Get in YAML/JSON format
kubectl get pod nginx -o yaml
kubectl get deployment app -o json

# Get with custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Get with JSONPath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Watch for changes
kubectl get pods --watch
kubectl get pods -w

# Get from all namespaces
kubectl get pods --all-namespaces
kubectl get pods -A
```

### **Describe Resources**

```bash
# Describe pod
kubectl describe pod nginx

# Describe deployment
kubectl describe deployment app-deployment

# Describe service
kubectl describe service my-service

# Describe node
kubectl describe node worker-1
```

### **Update Resources**

```bash
# Apply changes from file
kubectl apply -f deployment.yaml

# Edit resource in editor
kubectl edit deployment nginx-deployment

# Patch resource
kubectl patch deployment nginx -p '{"spec":{"replicas":5}}'

# Set image
kubectl set image deployment/nginx nginx=nginx:1.22

# Scale deployment
kubectl scale deployment nginx --replicas=3

# Autoscale
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80
```

### **Delete Resources**

```bash
# Delete from file
kubectl delete -f deployment.yaml

# Delete by name
kubectl delete pod nginx
kubectl delete deployment app-deployment
kubectl delete service my-service

# Delete all pods in namespace
kubectl delete pods --all

# Delete by label
kubectl delete pods -l app=nginx

# Force delete (use with caution!)
kubectl delete pod nginx --force --grace-period=0

# Delete namespace and all resources
kubectl delete namespace dev
```

---

## **4. Pod Operations** 🐳

### **Creating Pods**

```bash
# Run pod imperatively
kubectl run nginx --image=nginx:1.21
kubectl run busybox --image=busybox --command -- sleep 3600

# Run pod with specific labels
kubectl run nginx --image=nginx --labels="app=web,tier=frontend"

# Run pod with environment variables
kubectl run nginx --image=nginx --env="ENV=prod" --env="LOG_LEVEL=info"

# Dry run (generate YAML without creating)
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Create from file
kubectl apply -f pod.yaml
```

### **Managing Pods**

```bash
# List pods
kubectl get pods
kubectl get pods -o wide
kubectl get pods --show-labels
kubectl get pods -l app=nginx  # Filter by label

# Describe pod
kubectl describe pod nginx

# Get pod logs
kubectl logs nginx
kubectl logs nginx -f  # Follow logs
kubectl logs nginx --tail=100  # Last 100 lines
kubectl logs nginx --since=1h  # Last hour
kubectl logs nginx -c container-name  # Specific container in multi-container pod

# Execute command in pod
kubectl exec nginx -- ls /usr/share/nginx/html
kubectl exec -it nginx -- /bin/bash  # Interactive shell
kubectl exec -it nginx -c sidecar -- sh  # Specific container

# Port forwarding
kubectl port-forward pod/nginx 8080:80
kubectl port-forward pod/nginx :80  # Random local port

# Copy files to/from pod
kubectl cp /tmp/file.txt nginx:/tmp/
kubectl cp nginx:/var/log/app.log /tmp/app.log

# Delete pod
kubectl delete pod nginx
```

### **Pod Status & Debugging**

```bash
# Get pod status
kubectl get pods

# Pod phases: Pending, Running, Succeeded, Failed, Unknown

# Check pod events
kubectl describe pod nginx | grep -A 10 Events

# Get pod YAML
kubectl get pod nginx -o yaml

# Check specific container status
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[*].state}'

# Restart pod (delete and let controller recreate)
kubectl delete pod nginx
```

---

## **5. Deployment & Scaling** 🚀

### **Creating Deployments**

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21
kubectl create deployment app --image=myapp:v1 --replicas=3

# Create from file
kubectl apply -f deployment.yaml

# Generate deployment YAML
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deployment.yaml

# Create with specific resource limits
kubectl create deployment nginx --image=nginx \
  --port=80 \
  --dry-run=client -o yaml | \
  kubectl set resources --local -f - \
    --requests=cpu=100m,memory=128Mi \
    --limits=cpu=200m,memory=256Mi \
    --dry-run=client -o yaml > deployment.yaml
```

### **Managing Deployments**

```bash
# List deployments
kubectl get deployments
kubectl get deploy  # Shorthand

# Describe deployment
kubectl describe deployment nginx

# Update image
kubectl set image deployment/nginx nginx=nginx:1.22

# Rollout status
kubectl rollout status deployment/nginx

# Rollout history
kubectl rollout history deployment/nginx

# Rollout undo (rollback)
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2

# Pause rollout
kubectl rollout pause deployment/nginx

# Resume rollout
kubectl rollout resume deployment/nginx

# Restart deployment
kubectl rollout restart deployment/nginx
```

### **Scaling**

```bash
# Manual scaling
kubectl scale deployment nginx --replicas=5

# Autoscaling
kubectl autoscale deployment nginx --min=2 --max=10 --cpu-percent=80

# Get HPA
kubectl get hpa

# Delete HPA
kubectl delete hpa nginx
```

### **ReplicaSets**

```bash
# List ReplicaSets
kubectl get replicasets
kubectl get rs

# Describe ReplicaSet
kubectl describe rs nginx-<hash>

# Scale ReplicaSet (normally done via Deployment)
kubectl scale rs nginx-<hash> --replicas=3
```

---

## **6. Service & Networking** 🌐

### **Creating Services**

```bash
# Expose deployment as service
kubectl expose deployment nginx --port=80 --type=ClusterIP
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Create service imperatively
kubectl create service clusterip my-svc --tcp=80:8080
kubectl create service nodeport my-svc --tcp=80:8080 --node-port=30080
kubectl create service loadbalancer my-svc --tcp=80:8080

# Service types:
# - ClusterIP: Internal cluster IP (default)
# - NodePort: Exposes on each node's IP at static port
# - LoadBalancer: Cloud provider load balancer
# - ExternalName: Maps to external DNS name
```

### **Managing Services**

```bash
# List services
kubectl get services
kubectl get svc

# Describe service
kubectl describe service nginx

# Get service endpoints
kubectl get endpoints nginx

# Delete service
kubectl delete service nginx

# Test service from within cluster
kubectl run curl --image=curlimages/curl -it --rm -- sh
# Inside pod: curl http://nginx:80
```

### **Ingress**

```bash
# List ingress
kubectl get ingress
kubectl get ing

# Describe ingress
kubectl describe ingress my-ingress

# Create from file
kubectl apply -f ingress.yaml

# Delete ingress
kubectl delete ingress my-ingress
```

### **Network Policies**

```bash
# Get network policies
kubectl get networkpolicies
kubectl get netpol

# Describe network policy
kubectl describe networkpolicy deny-all

# Apply network policy
kubectl apply -f network-policy.yaml
```

---

## **7. ConfigMaps & Secrets** 🔐

### **ConfigMaps**

```bash
# Create from literal
kubectl create configmap app-config --from-literal=ENV=prod --from-literal=LOG_LEVEL=info

# Create from file
kubectl create configmap app-config --from-file=config.properties

# Create from directory
kubectl create configmap app-config --from-file=./configs/

# Create from env file
kubectl create configmap app-config --from-env-file=.env

# Get ConfigMaps
kubectl get configmaps
kubectl get cm

# Describe ConfigMap
kubectl describe configmap app-config

# Get ConfigMap data
kubectl get configmap app-config -o yaml

# Edit ConfigMap
kubectl edit configmap app-config

# Delete ConfigMap
kubectl delete configmap app-config
```

### **Secrets**

```bash
# Create generic secret
kubectl create secret generic db-password --from-literal=password=secret123

# Create from file
kubectl create secret generic ssh-key --from-file=ssh-privatekey=~/.ssh/id_rsa

# Create TLS secret
kubectl create secret tls my-tls-secret --cert=cert.crt --key=cert.key

# Create Docker registry secret
kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# Get secrets
kubectl get secrets

# Describe secret (doesn't show values)
kubectl describe secret db-password

# Get secret value (base64 encoded)
kubectl get secret db-password -o yaml

# Decode secret
kubectl get secret db-password -o jsonpath='{.data.password}' | base64 --decode

# Delete secret
kubectl delete secret db-password
```

---

## **8. Logs & Debugging** 🔍

### **Logs**

```bash
# View pod logs
kubectl logs pod-name

# Follow logs (tail -f)
kubectl logs pod-name -f

# Last 100 lines
kubectl logs pod-name --tail=100

# Logs from last hour
kubectl logs pod-name --since=1h

# Logs from last 5 minutes
kubectl logs pod-name --since=5m

# Logs from specific container
kubectl logs pod-name -c container-name

# Previous instance logs (after crash)
kubectl logs pod-name --previous

# All pods with label
kubectl logs -l app=nginx --all-containers=true

# Logs from deployment
kubectl logs deployment/nginx
```

### **Debugging**

```bash
# Execute command in pod
kubectl exec pod-name -- ls /app

# Interactive shell
kubectl exec -it pod-name -- /bin/bash
kubectl exec -it pod-name -- sh

# Port forwarding for debugging
kubectl port-forward pod/nginx 8080:80

# Run debug pod
kubectl run debug --image=busybox -it --rm -- sh

# Run debug pod with network access to another pod
kubectl debug -it pod-name --image=busybox --target=container-name

# Attach to running container
kubectl attach pod-name -i

# Copy files for debugging
kubectl cp pod-name:/var/log/app.log ./app.log

# Check events
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning

# Describe for troubleshooting
kubectl describe pod pod-name
kubectl describe deployment deployment-name
```

---

## **9. Resource Monitoring** 📈

### **Resource Usage**

```bash
# Node resource usage
kubectl top nodes

# Pod resource usage
kubectl top pods

# Pod resource usage in specific namespace
kubectl top pods -n kube-system

# Pod resource usage with containers
kubectl top pods --containers

# Sort by CPU
kubectl top pods --sort-by=cpu

# Sort by memory
kubectl top pods --sort-by=memory

# Resource usage for specific pod
kubectl top pod pod-name
```

### **Resource Quotas**

```bash
# Get resource quotas
kubectl get resourcequotas
kubectl get quota

# Describe quota
kubectl describe resourcequota my-quota

# Create quota
kubectl create quota my-quota --hard=pods=10,cpu=4,memory=8Gi
```

### **Limit Ranges**

```bash
# Get limit ranges
kubectl get limitranges
kubectl get limits

# Describe limit range
kubectl describe limitrange my-limits
```

---

## **10. Troubleshooting Commands** 🔧

### **Cluster Troubleshooting**

```bash
# Check cluster health
kubectl cluster-info
kubectl get nodes
kubectl get componentstatuses

# Check system pods
kubectl get pods -n kube-system

# Check API server
kubectl get --raw /healthz
kubectl get --raw /readyz

# Check events
kubectl get events --all-namespaces --sort-by='.lastTimestamp'

# Check resource usage
kubectl top nodes
kubectl top pods --all-namespaces
```

### **Pod Troubleshooting**

```bash
# Pod not starting - check status
kubectl get pods
kubectl describe pod pod-name

# Common issues to check:
# 1. Image pull errors
kubectl describe pod pod-name | grep -A 5 Events

# 2. Resource constraints
kubectl describe node node-name | grep -A 5 Allocated

# 3. Configuration errors
kubectl logs pod-name
kubectl logs pod-name --previous

# 4. Liveness/Readiness probe failures
kubectl describe pod pod-name | grep -A 10 Liveness
kubectl describe pod pod-name | grep -A 10 Readiness

# Check pod YAML
kubectl get pod pod-name -o yaml
```

### **Service Troubleshooting**

```bash
# Service not accessible
kubectl get svc service-name
kubectl describe svc service-name

# Check endpoints
kubectl get endpoints service-name

# Check if pods are labeled correctly
kubectl get pods --show-labels
kubectl get pods -l app=nginx

# Test service from within cluster
kubectl run curl --image=curlimages/curl -i --rm -- curl http://service-name:80
```

### **Network Troubleshooting**

```bash
# DNS troubleshooting
kubectl run busybox --image=busybox:1.28 -it --rm -- nslookup kubernetes.default
kubectl run busybox --image=busybox:1.28 -it --rm -- nslookup service-name

# Network connectivity test
kubectl run curl --image=curlimages/curl -it --rm -- curl http://service-name:80

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy policy-name
```

---

## **11. Best Practices** ⭐

### **Command Best Practices**

```
✅ Use declarative approach (kubectl apply -f)
✅ Version control your manifests
✅ Use namespaces for organization
✅ Label resources consistently
✅ Use --dry-run for validation
✅ Check resource usage regularly (kubectl top)
✅ Monitor events (kubectl get events)
✅ Use resource quotas and limits
✅ Regular backups of etcd
✅ Use kubectl diff before apply

❌ Don't use kubectl run in production
❌ Don't expose secrets in commands (use files)
❌ Don't delete namespaces without verification
❌ Don't skip resource requests/limits
❌ Don't use --force unless absolutely necessary
❌ Don't ignore warnings and events
```

### **Useful Aliases**

```bash
# Add to ~/.bashrc or ~/.zshrc

alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kdel='kubectl delete'
alias kl='kubectl logs'
alias kex='kubectl exec -it'
alias kpf='kubectl port-forward'
alias kaf='kubectl apply -f'
alias kdelf='kubectl delete -f'
alias kgp='kubectl get pods'
alias kgd='kubectl get deployments'
alias kgs='kubectl get services'
alias kgn='kubectl get nodes'

# Enable kubectl autocompletion
source <(kubectl completion bash)  # bash
source <(kubectl completion zsh)   # zsh
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: Most common kubectl commands?**
```bash
# Get resources
kubectl get pods/deployments/services/nodes

# Describe resource
kubectl describe pod <pod-name>

# Logs
kubectl logs <pod-name> -f

# Execute command
kubectl exec -it <pod-name> -- /bin/bash

# Apply configuration
kubectl apply -f <file.yaml>

# Scale
kubectl scale deployment <name> --replicas=3

# Delete
kubectl delete pod/deployment/service <name>
```

### **Q2: How to debug a failing pod?**
```bash
# 1. Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# 2. Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous instance

# 3. Check events
kubectl get events --sort-by='.lastTimestamp'

# 4. Interactive debugging
kubectl exec -it <pod-name> -- /bin/bash

# 5. Check resource constraints
kubectl describe node <node-name>

# Common issues:
- ImagePullBackOff: Wrong image name/tag or auth
- CrashLoopBackOff: Application crashing, check logs
- Pending: Insufficient resources, check node capacity
- Error: Various issues, check describe output
```

### **Q3: How to update a deployment?**
```bash
# Method 1: Update image
kubectl set image deployment/nginx nginx=nginx:1.22

# Method 2: Edit directly
kubectl edit deployment nginx

# Method 3: Apply updated file
kubectl apply -f deployment.yaml

# Check rollout status
kubectl rollout status deployment/nginx

# Rollback if needed
kubectl rollout undo deployment/nginx
```

### **Q4: How to expose a deployment as a service?**
```bash
# ClusterIP (internal only)
kubectl expose deployment nginx --port=80 --type=ClusterIP

# NodePort (external access via node IP)
kubectl expose deployment nginx --port=80 --type=NodePort

# LoadBalancer (cloud load balancer)
kubectl expose deployment nginx --port=80 --type=LoadBalancer

# Verify
kubectl get svc nginx
kubectl describe svc nginx
```

### **Q5: Essential troubleshooting commands?**
```bash
# Cluster health
kubectl cluster-info
kubectl get nodes
kubectl get componentstatuses

# Resource status
kubectl get pods --all-namespaces
kubectl get events --sort-by='.lastTimestamp'

# Detailed information
kubectl describe pod <pod-name>
kubectl logs <pod-name> -f

# Resource usage
kubectl top nodes
kubectl top pods

# Network debugging
kubectl run curl --image=curlimages/curl -it --rm -- sh
kubectl exec -it <pod> -- /bin/bash

# Check configurations
kubectl get cm/secrets
kubectl describe cm/secret <name>
```

---

**Quick Command Reference**:

```bash
# Create resources
kubectl create/apply -f <file>

# Get resources
kubectl get pods/svc/deploy -o wide/yaml/json

# Update resources
kubectl edit/patch/set image

# Delete resources  
kubectl delete pod/svc/deploy <name>

# Debug
kubectl describe/logs/exec/port-forward

# Scale
kubectl scale deployment <name> --replicas=N

# Rollout
kubectl rollout status/history/undo
```

---

**Related Guides**:
- [Kubernetes Fundamentals](Kubernetes_Fundamentals.md)
- [Kubernetes Pods & Deployments](Kubernetes_Pods_Deployments.md)
- [Kubernetes Services & Networking](Kubernetes_Services_Networking.md)
- [Kubernetes Troubleshooting](Kubernetes_RBAC_Security.md)

---

**☸️ Master kubectl Commands for Efficient Kubernetes Management!**

*The kubectl CLI is your primary interface to Kubernetes - mastering it is essential.*
