# **Kubernetes Tutorials Collection** ☸️

Comprehensive Kubernetes tutorials and guides for container orchestration mastery. This collection covers everything from fundamentals to production-ready deployments.

---

## **📚 Complete Tutorial Collection**

### **Core Guides**

| Guide | Description | Topics Covered |
|-------|-------------|----------------|
| **[Kubernetes Fundamentals](Kubernetes_Fundamentals.md)** | Architecture and core concepts | Control Plane, Worker Nodes, etcd, Container Runtime (CRI), API Objects, Cluster Setup |
| **[Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md)** | Complete kubectl command reference | kubectl basics, CRUD operations, debugging, logs, monitoring, troubleshooting |
| **[Pods & Deployments](Kubernetes_Pods_Deployments.md)** | Workload resources | Pods, ReplicaSets, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs, HPA |
| **[Services & Networking](Kubernetes_Services_Networking.md)** | Networking and service discovery | Services (ClusterIP/NodePort/LoadBalancer), Ingress, Network Policies, DNS |
| **[Legacy Interview Guide](Kubernetes_Legacy.md)** | Original comprehensive interview guide | Preserved original content with Java examples |

---

## **🎯 Learning Paths**

### **Path 1: Beginner to Professional** (8 weeks)

**Week 1-2: Foundations**
1. [Kubernetes Fundamentals](Kubernetes_Fundamentals.md)
   - Understand architecture (Control Plane, Nodes)
   - Learn about Pods and containers
   - Set up local cluster (minikube or kind)
   - Practice with kubectl basics

**Week 3-4: Core Resources**
2. [Pods & Deployments](Kubernetes_Pods_Deployments.md)
   - Create and manage Pods
   - Work with Deployments and ReplicaSets
   - Implement health checks (liveness/readiness)
   - Practice rolling updates and rollbacks

**Week 5-6: Networking**
3. [Services & Networking](Kubernetes_Services_Networking.md)
   - Understand Service types
   - Configure Ingress for HTTP routing
   - Implement Network Policies
   - Master DNS and service discovery

**Week 7-8: Production Ready**
4. Review [Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md)
   - Master kubectl commands
   - Learn debugging techniques
   - Practice troubleshooting scenarios
   - Build complete applications

**Practice Project**: Deploy a full microservices application with frontend, backend API, database, Ingress, and monitoring.

---

### **Path 2: Interview Preparation** (2-3 weeks)

**Week 1: Core Concepts**
- Read all Section 12 (Interview Cheat Sheet) from each guide
- Review [Kubernetes Fundamentals](Kubernetes_Fundamentals.md) - Architecture deep dive
- Study [Pods & Deployments](Kubernetes_Pods_Deployments.md) - Workload resources
- Review [Legacy Interview Guide](Kubernetes_Legacy.md) - 810 lines of interview Q&A

**Week 2: Hands-on Practice**
- Complete all examples in [Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md)
- Practice troubleshooting scenarios
- Deploy sample applications
- Understand failure scenarios and recovery

**Week 3: Advanced Topics**
- Master [Services & Networking](Kubernetes_Services_Networking.md)
- Understand Network Policies and security
- Practice scaling and performance tuning
- Review real-world scenarios

**Interview Topics Checklist**:
- ✅ Kubernetes architecture (Control Plane components)
- ✅ Pod lifecycle and phases
- ✅ Deployments vs StatefulSets
- ✅ Service types and when to use each
- ✅ Ingress vs LoadBalancer
- ✅ ConfigMaps and Secrets
- ✅ Resource requests and limits
- ✅ Rolling updates and rollbacks
- ✅ DNS and service discovery
- ✅ Troubleshooting failing pods

---

### **Path 3: DevOps Professional** (Self-Paced)

**Foundation** → **Workloads** → **Networking** → **Production**

1. **Understand the Platform**
   - [Kubernetes Fundamentals](Kubernetes_Fundamentals.md)
   - Architecture, components, cluster setup

2. **Master Resource Management**
   - [Pods & Deployments](Kubernetes_Pods_Deployments.md)
   - Workload types, scaling, updates

3. **Network Your Applications**
   - [Services & Networking](Kubernetes_Services_Networking.md)
   - Service mesh, security, policies

4. **Production Operations**
   - [Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md)
   - Monitoring, logging, troubleshooting

---

## **🔥 Quick Start**

### **5-Minute Setup**

```bash
# Install kubectl
brew install kubectl  # macOS
# or
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Install minikube
brew install minikube  # macOS

# Start local cluster
minikube start --cpus=4 --memory=8192

# Verify
kubectl cluster-info
kubectl get nodes
```

### **First Deployment**

```bash
# Create deployment
kubectl create deployment nginx --image=nginx:1.21 --replicas=3

# Expose as service
kubectl expose deployment nginx --port=80 --type=NodePort

# Access application
minikube service nginx

# Check status
kubectl get pods,svc,deploy

# Clean up
kubectl delete deployment nginx
kubectl delete service nginx
```

---

## **💡 Common Use Cases**

### **For Application Developers**

**Deploy containerized application**:
```bash
# Create deployment
kubectl create deployment myapp --image=myapp:v1.0 --replicas=3

# Expose service
kubectl expose deployment myapp --port=80 --target-port=8080

# Scale application
kubectl scale deployment myapp --replicas=5

# Update to new version
kubectl set image deployment/myapp myapp=myapp:v2.0

# Check rollout
kubectl rollout status deployment/myapp
```
📖 See: [Pods & Deployments](Kubernetes_Pods_Deployments.md#5-deployments)

---

### **For DevOps Engineers**

**Set up complete microservices stack**:
```bash
# Deploy frontend
kubectl apply -f frontend-deployment.yaml
kubectl expose deployment frontend --type=LoadBalancer --port=80

# Deploy backend API
kubectl apply -f api-deployment.yaml
kubectl expose deployment api --port=8080

# Deploy database (StatefulSet)
kubectl apply -f postgres-statefulset.yaml

# Configure Ingress
kubectl apply -f ingress.yaml

# Monitor
kubectl get all
kubectl top pods
```
📖 See: [Services & Networking](Kubernetes_Services_Networking.md#9-devops-use-cases)

---

### **For SREs**

**Troubleshoot production issues**:
```bash
# Check cluster health
kubectl get nodes
kubectl get pods --all-namespaces

# Find failing pods
kubectl get pods --field-selector=status.phase!=Running

# Debug specific pod
kubectl describe pod <pod-name>
kubectl logs <pod-name> -f
kubectl logs <pod-name> --previous

# Check resource usage
kubectl top nodes
kubectl top pods

# Check events
kubectl get events --sort-by='.lastTimestamp'
```
📖 See: [Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md#10-troubleshooting-commands)

---

## **📊 Topic Index**

### **Architecture & Concepts**
- [Kubernetes Architecture](Kubernetes_Fundamentals.md#2-kubernetes-architecture)
- [Control Plane Components](Kubernetes_Fundamentals.md#3-control-plane-components)
- [Worker Node Components](Kubernetes_Fundamentals.md#4-worker-node-components)
- [Container Runtime (CRI)](Kubernetes_Fundamentals.md#5-container-runtime-interface-cri)
- [etcd Database](Kubernetes_Fundamentals.md#6-etcd---the-cluster-database)

### **Workload Resources**
- [Pods](Kubernetes_Pods_Deployments.md#1-understanding-pods)
- [Pod Lifecycle](Kubernetes_Pods_Deployments.md#2-pod-lifecycle--phases)
- [Deployments](Kubernetes_Pods_Deployments.md#5-deployments)
- [StatefulSets](Kubernetes_Pods_Deployments.md#6-statefulsets)
- [DaemonSets](Kubernetes_Pods_Deployments.md#7-daemonsets)
- [Jobs & CronJobs](Kubernetes_Pods_Deployments.md#8-jobs--cronjobs)

### **Networking**
- [Services](Kubernetes_Services_Networking.md#2-services)
- [Service Types](Kubernetes_Services_Networking.md#3-service-types)
- [Ingress](Kubernetes_Services_Networking.md#4-ingress)
- [Network Policies](Kubernetes_Services_Networking.md#5-network-policies)
- [DNS & Service Discovery](Kubernetes_Services_Networking.md#6-dns-in-kubernetes)

### **Operations**
- [kubectl Commands](Kubernetes_Commands_Cheatsheet.md)
- [Cluster Setup](Kubernetes_Fundamentals.md#8-cluster-setup-options)
- [Troubleshooting](Kubernetes_Commands_Cheatsheet.md#10-troubleshooting-commands)
- [Resource Monitoring](Kubernetes_Commands_Cheatsheet.md#9-resource-monitoring)

---

## **🛠️ Practical Scenarios**

### **Scenario 1: Pod is CrashLooping**

```bash
# 1. Check pod status
kubectl get pods
kubectl describe pod <pod-name>

# 2. Check logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous

# 3. Common causes:
# - Application error → Fix app code
# - Missing config → Check ConfigMaps/Secrets
# - Resource limits → Increase limits
# - Failed health check → Adjust probe settings

# 4. Debug interactively
kubectl exec -it <pod-name> -- /bin/bash
```
📖 See: [Troubleshooting](Kubernetes_Pods_Deployments.md#10-troubleshooting)

---

### **Scenario 2: Service Not Accessible**

```bash
# 1. Check service and endpoints
kubectl get svc <service-name>
kubectl get endpoints <service-name>

# 2. If endpoints empty, check pod labels
kubectl get pods --show-labels
kubectl get pods -l app=myapp

# 3. Test from within cluster
kubectl run curl --image=curlimages/curl -it --rm -- curl http://<service-name>

# 4. Check network policies
kubectl get networkpolicies
```
📖 See: [Services Troubleshooting](Kubernetes_Services_Networking.md#10-troubleshooting)

---

### **Scenario 3: Deploy New Version**

```bash
# 1. Update image
kubectl set image deployment/myapp myapp=myapp:v2.0

# 2. Watch rollout
kubectl rollout status deployment/myapp

# 3. If issues, rollback
kubectl rollout undo deployment/myapp

# 4. Check history
kubectl rollout history deployment/myapp
```
📖 See: [Deployments](Kubernetes_Pods_Deployments.md#5-deployments)

---

## **📖 Features of Each Guide**

Every guide includes:

✅ **12-Section Structure** - Comprehensive, consistent organization
✅ **Table of Contents** - Quick navigation with emojis
✅ **Mermaid Diagrams** - Visual architecture representations
✅ **Code Examples** - Real YAML manifests with explanations
✅ **Command Examples** - Practical kubectl commands
✅ **DevOps Use Cases** - Real-world production scenarios
✅ **Troubleshooting** - Common issues and solutions
✅ **Best Practices** - Industry-standard approaches (Do's and Don'ts)
✅ **Interview Cheat Sheet** - Most asked questions with detailed answers
✅ **Cross-References** - Links to related guides

---

## **🎓 Certification Preparation**

These guides help prepare for:

- **CKA** (Certified Kubernetes Administrator)
  - Focus: [Fundamentals](Kubernetes_Fundamentals.md), [Commands](Kubernetes_Commands_Cheatsheet.md), [Pods & Deployments](Kubernetes_Pods_Deployments.md)
  
- **CKAD** (Certified Kubernetes Application Developer)
  - Focus: [Pods & Deployments](Kubernetes_Pods_Deployments.md), [Services & Networking](Kubernetes_Services_Networking.md), [Commands](Kubernetes_Commands_Cheatsheet.md)
  
- **CKS** (Certified Kubernetes Security Specialist)
  - Focus: [Network Policies](Kubernetes_Services_Networking.md#5-network-policies), Security best practices

---

## **🔧 Additional Resources**

### **Related DevOps Topics**
- 🐳 [Docker](../Docker/) - Container fundamentals
- 🔄 [CI/CD](../CI-CD/) - Continuous integration and deployment
- 🐧 [Linux Commands](../LinuxCommands/) - Essential Linux skills
- 🔧 [Git](../Git/) - Version control

### **Official Documentation**
- [Kubernetes Docs](https://kubernetes.io/docs/)
- [kubectl Reference](https://kubernetes.io/docs/reference/kubectl/)
- [API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)

### **Interactive Learning**
- [Play with Kubernetes](https://labs.play-with-k8s.com/)
- [Katacoda Kubernetes](https://www.katacoda.com/courses/kubernetes)
- [KillerCoda](https://killercoda.com/kubernetes)

### **Tools**
- [k9s](https://k9scli.io/) - Terminal UI for Kubernetes
- [Lens](https://k8slens.dev/) - Kubernetes IDE
- [Helm](https://helm.sh/) - Package manager for Kubernetes
- [kubectx/kubens](https://github.com/ahmetb/kubectx) - Context and namespace switching

---

## **💻 Hands-On Labs**

### **Lab 1: Deploy a Complete Application**
1. Create a Deployment for a web application
2. Expose it via a Service
3. Configure an Ingress for external access
4. Implement health checks
5. Scale the application
6. Perform a rolling update

### **Lab 2: StatefulSet with Persistent Storage**
1. Deploy a database using StatefulSet
2. Configure persistent volumes
3. Test data persistence across pod restarts
4. Scale the StatefulSet
5. Perform backup and restore

### **Lab 3: Microservices with Service Mesh**
1. Deploy multiple microservices
2. Configure inter-service communication
3. Implement network policies
4. Set up Ingress routing
5. Monitor service health

---

## **🚀 Pro Tips**

**For Daily Use**:
```bash
# Useful aliases
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deployments'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'
alias kx='kubectl exec -it'

# Enable kubectl completion
source <(kubectl completion bash)  # bash
source <(kubectl completion zsh)   # zsh
```

**For Faster Debugging**:
```bash
# Quick pod shell
alias ksh='kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash'

# Watch all resources
watch kubectl get all

# Get all events
kubectl get events --sort-by='.lastTimestamp' -A
```

**For Production**:
```
✅ Always set resource requests and limits
✅ Use namespaces for environment isolation
✅ Implement proper health checks
✅ Enable monitoring and logging
✅ Use Network Policies for security
✅ Version your manifests in Git
✅ Test in staging before production
✅ Have rollback plan ready
✅ Document your architecture
✅ Regular cluster backups (etcd)
```

---

## **📞 Quick Navigation**

| Need | Go To |
|------|-------|
| **Understand K8s architecture** | [Fundamentals](Kubernetes_Fundamentals.md#2-kubernetes-architecture) |
| **Create a Pod** | [Pods & Deployments](Kubernetes_Pods_Deployments.md#1-understanding-pods) |
| **Deploy an application** | [Pods & Deployments](Kubernetes_Pods_Deployments.md#5-deployments) |
| **Expose an application** | [Services & Networking](Kubernetes_Services_Networking.md#2-services) |
| **Configure Ingress** | [Services & Networking](Kubernetes_Services_Networking.md#4-ingress) |
| **Debug failing pod** | [Commands](Kubernetes_Commands_Cheatsheet.md#10-troubleshooting-commands) |
| **Scale deployment** | [Commands](Kubernetes_Commands_Cheatsheet.md#5-deployment--scaling) |
| **View logs** | [Commands](Kubernetes_Commands_Cheatsheet.md#8-logs--debugging) |
| **Interview prep** | All Section 12 + [Legacy Guide](Kubernetes_Legacy.md) |
| **Quick reference** | [Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md) |

---

## **📝 What's Next?**

After mastering these guides, explore:
1. **Helm** - Package manager for Kubernetes
2. **Operators** - Kubernetes application management
3. **Service Mesh** - Istio, Linkerd for advanced networking
4. **GitOps** - ArgoCD, Flux for declarative deployments
5. **Observability** - Prometheus, Grafana, ELK stack
6. **Security** - OPA, Falco, admission controllers

---

**Start Learning**: Pick a guide based on your experience level. Each guide is comprehensive and standalone but references others when relevant.

**Pro Tip**: Keep the [Commands Cheatsheet](Kubernetes_Commands_Cheatsheet.md) bookmarked for quick kubectl command lookups!

---

**Happy Learning! ☸️✨**

*Kubernetes is the de facto standard for container orchestration - mastering it opens endless opportunities!*
