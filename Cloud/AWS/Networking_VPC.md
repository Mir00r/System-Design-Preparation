# 🌐 AWS Networking: VPC, Subnets, Security Groups & Load Balancers

> *"A VPC is your private network in the cloud. You control who can talk to whom, what's public, what's private, and how traffic flows. Getting networking wrong means either security breaches or services that can't communicate."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [AWS Core Services](./Core_Services.md)

---

## 📋 Table of Contents
1. [VPC Fundamentals](#-vpc-fundamentals)
2. [Subnets & Routing](#-subnets--routing)
3. [Security Groups & NACLs](#-security)
4. [Load Balancers](#-load-balancers)
5. [Common Architectures](#-common-architectures)
6. [Interview Q&A](#-interview-qa)

---

## 🏗️ VPC Fundamentals

```
VPC (Virtual Private Cloud): Your isolated network in AWS
  - IP range: you define (CIDR block, e.g., 10.0.0.0/16 = 65,536 IPs)
  - Spans all AZs in a region
  - You control: subnets, routing, internet access, peering

  ┌──────────────────────────────────────────────────────────────┐
  │ AWS Region (us-east-1)                                       │
  │                                                              │
  │  ┌────────────────────────────────────────────────────────┐  │
  │  │ VPC: 10.0.0.0/16                                      │  │
  │  │                                                        │  │
  │  │  AZ-A (us-east-1a)    │   AZ-B (us-east-1b)           │  │
  │  │  ┌──────────────────┐ │   ┌──────────────────┐        │  │
  │  │  │ Public Subnet    │ │   │ Public Subnet    │        │  │
  │  │  │ 10.0.1.0/24      │ │   │ 10.0.2.0/24      │        │  │
  │  │  │ [ALB] [NAT GW]   │ │   │ [ALB]            │        │  │
  │  │  └──────────────────┘ │   └──────────────────┘        │  │
  │  │  ┌──────────────────┐ │   ┌──────────────────┐        │  │
  │  │  │ Private Subnet   │ │   │ Private Subnet   │        │  │
  │  │  │ 10.0.3.0/24      │ │   │ 10.0.4.0/24      │        │  │
  │  │  │ [EC2] [ECS]      │ │   │ [EC2] [ECS]      │        │  │
  │  │  └──────────────────┘ │   └──────────────────┘        │  │
  │  │  ┌──────────────────┐ │   ┌──────────────────┐        │  │
  │  │  │ Data Subnet      │ │   │ Data Subnet      │        │  │
  │  │  │ 10.0.5.0/24      │ │   │ 10.0.6.0/24      │        │  │
  │  │  │ [RDS] [ElastiC]  │ │   │ [RDS Standby]    │        │  │
  │  │  └──────────────────┘ │   └──────────────────┘        │  │
  │  └────────────────────────────────────────────────────────┘  │
  └──────────────────────────────────────────────────────────────┘

KEY COMPONENTS:
  Internet Gateway (IGW): connects VPC to the internet
  NAT Gateway: lets private instances access internet (outbound only)
  Route Tables: control where traffic goes
  Subnets: subdivisions of VPC (public or private)
```

---

## 🔀 Subnets & Routing

```
PUBLIC SUBNET:
  - Route table has: 0.0.0.0/0 → Internet Gateway
  - Resources get public IPs
  - Directly accessible from internet
  - Use for: load balancers, bastion hosts, NAT gateways
  
PRIVATE SUBNET:
  - Route table has: 0.0.0.0/0 → NAT Gateway (outbound only)
  - No public IPs, not directly reachable from internet
  - Use for: application servers, containers
  
DATA SUBNET:
  - No internet route at all (most restrictive)
  - Only accessible from within VPC
  - Use for: databases, caches, sensitive data stores

ROUTING EXAMPLE:
  Public subnet route table:
    10.0.0.0/16 → local (within VPC)
    0.0.0.0/0   → igw-abc123 (internet gateway)
    
  Private subnet route table:
    10.0.0.0/16 → local (within VPC)
    0.0.0.0/0   → nat-xyz789 (NAT gateway in public subnet)
    
  Data subnet route table:
    10.0.0.0/16 → local (within VPC only — no internet!)
```

---

## 🔒 Security

```
SECURITY GROUPS (instance-level firewall):
  - Stateful: if you allow inbound, outbound reply is automatic
  - ALLOW rules only (no deny rules)
  - Applied to: EC2, RDS, ECS tasks, Lambda (in VPC)
  
  Example - Web server SG:
    Inbound: TCP 443 from 0.0.0.0/0 (HTTPS from anywhere)
    Inbound: TCP 80 from 0.0.0.0/0 (HTTP from anywhere)
    Outbound: All traffic (default)
  
  Example - App server SG:
    Inbound: TCP 8080 from sg-webserver (only from web tier!)
    Outbound: All traffic
    
  Example - Database SG:
    Inbound: TCP 5432 from sg-appserver (only from app tier!)
    Outbound: None needed

NETWORK ACLs (subnet-level firewall):
  - Stateless: must explicitly allow both inbound AND outbound
  - Allow AND deny rules
  - Rules evaluated in order (lowest number first)
  - Applied to: entire subnet
  - Use for: blocking specific IPs, additional defense layer

SECURITY GROUP vs NACL:
  ┌──────────────────┬──────────────────┬──────────────────┐
  │                  │ Security Group   │ Network ACL      │
  ├──────────────────┼──────────────────┼──────────────────┤
  │ Level            │ Instance         │ Subnet           │
  │ Stateful         │ Yes              │ No               │
  │ Rules            │ Allow only       │ Allow + Deny     │
  │ Evaluation       │ All rules        │ In order         │
  │ Default          │ Deny all inbound │ Allow all        │
  └──────────────────┴──────────────────┴──────────────────┘
```

---

## ⚖️ Load Balancers

```
ALB (Application Load Balancer) — Layer 7 (HTTP/HTTPS):
  - Path-based routing: /api/* → backend, /images/* → CDN
  - Host-based routing: api.example.com → service A
  - WebSocket support
  - Integration with WAF (web application firewall)
  - Best for: HTTP microservices, APIs, web apps

NLB (Network Load Balancer) — Layer 4 (TCP/UDP):
  - Ultra-low latency (<100 microseconds)
  - Millions of requests per second
  - Static IP / Elastic IP support
  - Best for: gaming, IoT, extreme performance, non-HTTP protocols

GWLB (Gateway Load Balancer) — Layer 3:
  - Routes traffic through third-party appliances (firewalls, IDS)
  - Transparent to source/destination
  - Best for: security appliances in traffic path

ALB EXAMPLE:
  ┌─────────────────────────────────────────────────────────┐
  │  ALB (public subnet)                                    │
  │  ┌───────────────────────────────────────────┐          │
  │  │ Listener: HTTPS:443                       │          │
  │  │                                           │          │
  │  │ Rule 1: /api/*    → Target Group: backend │          │
  │  │ Rule 2: /ws/*     → Target Group: websocket│         │
  │  │ Rule 3: default   → Target Group: frontend│          │
  │  └───────────────────────────────────────────┘          │
  │                                                         │
  │  Target Groups:                                         │
  │    backend:   [EC2-1:8080, EC2-2:8080] (health: /health)│
  │    websocket: [EC2-3:9090]                              │
  │    frontend:  [EC2-4:3000, EC2-5:3000]                  │
  └─────────────────────────────────────────────────────────┘
```

---

## 🏛️ Common Architectures

```
THREE-TIER VPC:
  Internet → IGW → ALB (public) → App (private) → DB (data)
  
  Traffic flow:
  [User] → [CloudFront] → [ALB in public subnet]
         → [ECS tasks in private subnet]
         → [RDS in data subnet]
  
  Outbound from private subnet:
  [ECS] → [NAT Gateway in public subnet] → [Internet]
  (for pulling Docker images, calling external APIs)

VPC PEERING / TRANSIT GATEWAY:
  Connect multiple VPCs (microservices in different VPCs)
  
  [VPC-A: Orders] ←peer→ [VPC-B: Payments]
                     ↕
            [Transit Gateway]
                     ↕
  [VPC-C: Products] ←→ [VPC-D: Users]
  
  Transit Gateway: hub-and-spoke (better than mesh of peering connections)

VPC ENDPOINTS (PrivateLink):
  Access AWS services WITHOUT going through internet
  [EC2 in private subnet] → [VPC Endpoint] → [S3, DynamoDB, SQS]
  No NAT Gateway needed! (saves cost + lower latency)
```

---

## ⚠️ Common Pitfalls

1. **Database in public subnet** — NEVER put RDS, ElastiCache, or any database in a public subnet. Always use private/data subnets with security groups restricting access to only application servers.

2. **Overly permissive security groups** — `0.0.0.0/0` on port 22 (SSH) is a security risk. Use bastion hosts or AWS Systems Manager Session Manager for access. Restrict source to specific security groups, not CIDR ranges.

3. **Single NAT Gateway** — One NAT Gateway in one AZ = single point of failure. If that AZ fails, private instances in other AZs lose internet access. Deploy one NAT Gateway per AZ for high availability.

---

## 📝 Interview Q&A

**Q: How would you design a secure VPC for a production microservices application?**
> A: (1) **3-tier subnet architecture**: public (ALB, NAT GW), private (app servers/containers), data (databases). (2) **Multi-AZ**: deploy across at least 2 AZs for HA. (3) **Security Groups**: layer-specific — ALB SG allows 443 from internet, app SG allows 8080 from ALB SG only, DB SG allows 5432 from app SG only. (4) **NAT Gateway per AZ** for outbound internet. (5) **VPC Endpoints** for S3, DynamoDB, ECR (no internet needed). (6) **Flow Logs** enabled for audit and troubleshooting. (7) **No SSH** — use Systems Manager Session Manager. (8) **CIDR planning**: /16 VPC with /24 subnets leaves room for growth.

---

## 🔗 What to Read Next

1. **[Cloud/AWS/AWS_Well_Architected.md](./AWS_Well_Architected.md)** — Best practices framework
2. **[Security/TLS_SSL_HTTPS.md](../../Security/TLS_SSL_HTTPS.md)** — Securing communications
3. **[BuildingBlocks/LoadBalancing.md](../../BuildingBlocks/LoadBalancing.md)** — Load balancing algorithms

---

*[← Databases](./Databases_RDS_DynamoDB.md) | [Back to Index](../../INDEX.md) | [Next: Well-Architected →](./AWS_Well_Architected.md)*
