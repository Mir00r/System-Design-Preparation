# 🌐 IP Addresses & Subnetting

> *"Every device on the internet has an address — just like every house has a street address. Without IP addresses, packets would have no idea where to go. Understanding IPs, subnetting, and address spaces is fundamental to system design because it determines how you network your services, segment your infrastructure, and handle billions of connected devices."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [OSI Model](OSI_Model.md), [DNS](DNS.md)

---

## 📋 Table of Contents
1. [IPv4 Basics](#-ipv4-basics)
2. [IPv6 Basics](#-ipv6-basics)
3. [Public vs Private IPs](#-public-vs-private-ips)
4. [Subnetting](#-subnetting)
5. [NAT & Port Forwarding](#-nat--port-forwarding)
6. [IP in System Design](#-ip-in-system-design)
7. [Interview Q&A](#-interview-qa)

---

## 🔢 IPv4 Basics

```
IPv4: 32 bits = 4 octets separated by dots
Example: 192.168.1.100

  192  .  168  .  1    .  100
  ┌────┐  ┌────┐  ┌────┐  ┌────┐
  │8bit│  │8bit│  │8bit│  │8bit│  = 32 bits total
  └────┘  └────┘  └────┘  └────┘
  
  Each octet: 0-255 (2^8 = 256 values)
  Total addresses: 2^32 = ~4.3 billion (NOT ENOUGH!)

CLASSES (historical, mostly obsolete now):
  ┌────────────────────────────────────────────────────────────┐
  │  Class │ First Octet │  Network/Host    │ # Networks       │
  ├────────────────────────────────────────────────────────────┤
  │  A     │  1-126      │  N.H.H.H        │ 128 (huge orgs!) │
  │  B     │  128-191    │  N.N.H.H        │ 16,384           │
  │  C     │  192-223    │  N.N.N.H        │ 2,097,152        │
  │  D     │  224-239    │  Multicast       │ N/A              │
  │  E     │  240-255    │  Reserved        │ N/A              │
  └────────────────────────────────────────────────────────────┘
  
  Modern approach: CIDR (Classless Inter-Domain Routing)
  192.168.1.0/24 means "first 24 bits = network, last 8 = host"

SPECIAL ADDRESSES:
  127.0.0.1    → Localhost (loopback, your own machine!)
  0.0.0.0      → "Any" address (listen on all interfaces)
  255.255.255.255 → Broadcast (send to everyone on subnet)
  169.254.x.x  → Link-local (DHCP failed, self-assigned)
```

---

## 🆕 IPv6 Basics

```
IPv6: 128 bits = 8 groups of 4 hex digits
Example: 2001:0db8:85a3:0000:0000:8a2e:0370:7334

  Total addresses: 2^128 = 340 undecillion (enough for every atom!)
  
WHY IPv6?
  • IPv4 exhausted! (only 4.3B addresses, 8B+ devices!)
  • No NAT needed (every device gets a public IP!)
  • Built-in security (IPsec mandatory in spec)
  • Simpler headers (faster routing!)
  • Auto-configuration (devices configure themselves!)

SHORTHAND RULES:
  • Drop leading zeros: 0db8 → db8
  • Replace consecutive zeros with ::  (once only!)
  
  Full:  2001:0db8:0000:0000:0000:0000:0000:0001
  Short: 2001:db8::1

SPECIAL IPv6:
  ::1          → Localhost (like 127.0.0.1)
  fe80::/10    → Link-local (like 169.254.x.x)
  ::ffff:x.x.x.x → IPv4-mapped IPv6 (transition!)

ADOPTION (as of 2024):
  Google: ~45% of traffic is IPv6
  Most clouds: dual-stack (both IPv4 + IPv6)
  Interview tip: mention IPv6 awareness, but most system design
  discussions still use IPv4 notation for simplicity!
```

---

## 🏠 Public vs Private IPs

```
PRIVATE IPs: Used INSIDE your network (not routable on internet!)
PUBLIC IPs: Used on the internet (globally unique!)

PRIVATE IP RANGES (RFC 1918):
  ┌──────────────────────────────────────────────────────────┐
  │  Range                    │  CIDR         │  # Addresses │
  ├──────────────────────────────────────────────────────────┤
  │  10.0.0.0 - 10.255.255.255│  10.0.0.0/8  │  16 million  │
  │  172.16.0.0 - 172.31.255.255│ 172.16.0.0/12│  1 million  │
  │  192.168.0.0 - 192.168.255.255│192.168.0.0/16│  65,536   │
  └──────────────────────────────────────────────────────────┘

HOME NETWORK:
  Internet ──► [Router: 203.0.113.5 (public)]
                    │
              Private network (192.168.1.0/24):
              ├── 192.168.1.1  (router)
              ├── 192.168.1.10 (laptop)
              ├── 192.168.1.11 (phone)
              └── 192.168.1.12 (smart TV)

  All devices share ONE public IP via NAT!

CLOUD VPC (AWS/GCP/Azure):
  VPC: 10.0.0.0/16 (65,536 private IPs!)
    ├── Subnet A: 10.0.1.0/24 (256 hosts, public)
    ├── Subnet B: 10.0.2.0/24 (256 hosts, private)
    └── Subnet C: 10.0.3.0/24 (256 hosts, private)
  
  Public subnet: instances get both private + public IP
  Private subnet: only private IP (no direct internet access!)
  NAT Gateway: allows private subnet → internet (outbound only!)
```

---

## 🧩 Subnetting

```
SUBNETTING: Dividing a network into smaller sub-networks!

WHY SUBNET?
  • Security: isolate sensitive systems (DB ≠ web tier!)
  • Organization: departments, environments (prod/staging)
  • Efficiency: reduce broadcast domain size
  • Routing: control traffic flow between segments

CIDR NOTATION:
  10.0.0.0/24
     └─────┘ └──┘
     network   prefix length (how many bits = network part)
  
  /24 = 24 bits network + 8 bits host = 256 addresses (254 usable)
  /16 = 16 bits network + 16 bits host = 65,536 addresses
  /8  = 8 bits network + 24 bits host = 16,777,216 addresses

QUICK REFERENCE:
  ┌───────────────────────────────────────────────────────────┐
  │  CIDR  │  Subnet Mask      │  Hosts │  Use Case          │
  ├───────────────────────────────────────────────────────────┤
  │  /32   │  255.255.255.255  │  1     │  Single host!      │
  │  /28   │  255.255.255.240  │  14    │  Small service tier │
  │  /24   │  255.255.255.0    │  254   │  Typical subnet    │
  │  /20   │  255.255.240.0    │  4094  │  Large subnet      │
  │  /16   │  255.255.0.0      │  65534 │  VPC range         │
  └───────────────────────────────────────────────────────────┘

EXAMPLE: Design a VPC for a microservices app:
  VPC: 10.0.0.0/16 (entire address space)
  
  ┌─────────────────────────────────────────────────────────┐
  │  Public subnets (internet-facing):                      │
  │    10.0.1.0/24 → Load Balancers (AZ-a)                │
  │    10.0.2.0/24 → Load Balancers (AZ-b)                │
  ├─────────────────────────────────────────────────────────┤
  │  App subnets (private):                                 │
  │    10.0.10.0/24 → App servers (AZ-a)                   │
  │    10.0.11.0/24 → App servers (AZ-b)                   │
  ├─────────────────────────────────────────────────────────┤
  │  Data subnets (most restricted!):                       │
  │    10.0.20.0/24 → Databases (AZ-a)                     │
  │    10.0.21.0/24 → Databases (AZ-b)                     │
  └─────────────────────────────────────────────────────────┘
  
  Security groups:
  • LB: allow 80/443 from internet
  • App: allow traffic ONLY from LB subnet!
  • DB: allow traffic ONLY from App subnet!
  → Defense in depth via network segmentation!
```

---

## 🔀 NAT & Port Forwarding

```
NAT (Network Address Translation):
  Allows many private IPs to share ONE public IP!

HOW NAT WORKS:
  Inside network:     Router (NAT):        Internet:
  192.168.1.10:5000 → 203.0.113.5:40001 → Server
  192.168.1.11:3000 → 203.0.113.5:40002 → Server
  
  Router maintains a translation table:
  ┌──────────────────────────────────────────────────────────┐
  │  Internal          │  External          │  Destination   │
  ├──────────────────────────────────────────────────────────┤
  │  192.168.1.10:5000 │  203.0.113.5:40001 │  Server:80    │
  │  192.168.1.11:3000 │  203.0.113.5:40002 │  Server:443   │
  └──────────────────────────────────────────────────────────┘

NAT TYPES (important for P2P/WebRTC!):
  • Full Cone: any external host can send to mapped port
  • Restricted: only the contacted external IP can reply
  • Port Restricted: only exact IP:port can reply
  • Symmetric: different mapping for each destination! (hardest for P2P!)
  
  WebRTC NAT traversal: STUN (discover public IP) + TURN (relay if NAT is too strict!)

IN CLOUD:
  • NAT Gateway (AWS): allows private subnet → internet (outbound)
  • Elastic IP: static public IP attached to instance
  • Internal LB: routes traffic within private subnets (no public IP!)
```

---

## 🏗️ IP in System Design

```
HOW IP KNOWLEDGE HELPS IN INTERVIEWS:

1. VPC DESIGN:
   "I'd create a VPC with /16 CIDR, separate public and private
    subnets across multiple AZs. Load balancers in public subnet,
    app servers and databases in private subnets."

2. SERVICE COMMUNICATION:
   "Services communicate via private IPs within VPC.
    Service discovery (DNS or Consul) resolves service names to IPs.
    No traffic goes over public internet!"

3. MULTI-REGION:
   "Each region has its own VPC with non-overlapping CIDRs.
    Connected via VPC peering or Transit Gateway.
    Global load balancer routes users to nearest region (Anycast IP!)."

4. SECURITY:
   "Database in private subnet (no public IP!).
    Security groups: only allow app subnet → DB port.
    VPN/bastion host for admin access."

ANYCAST (bonus!):
  One IP address → multiple servers worldwide!
  Traffic routed to NEAREST server (by BGP routing)!
  Used by: CDNs, DNS (8.8.8.8), DDoS protection (Cloudflare)
  
  User in Tokyo → 1.2.3.4 → routes to Tokyo server
  User in London → 1.2.3.4 → routes to London server
  Same IP, different physical server! Magic of BGP routing!
```

---

## ❓ Interview Q&A

**Q1: Why do we use private IPs inside a VPC instead of public IPs?**
> Four reasons: (1) Security — private IPs aren't reachable from internet, reducing attack surface, (2) Cost — public IPs are scarce and often charged (AWS charges for public IPv4!), (3) Scalability — with 65K private IPs per /16 VPC, you have plenty of addresses, (4) Network control — security groups and NACLs control traffic between private subnets. Only load balancers need public IPs; everything else stays private.

**Q2: How would you design the network for a multi-tier application?**
> Three-tier subnet design: (1) Public subnet: load balancers with public IPs, (2) Private app subnet: application servers only accessible from LB, (3) Private data subnet: databases only accessible from app tier. Each tier in multiple AZs for HA. Security groups enforce tier boundaries. NAT gateway for outbound internet (updates, third-party APIs). This is defense-in-depth — even if LB is compromised, attacker can't reach DB directly.

**Q3: What is Anycast and when would you use it?**
> Anycast: same IP address advertised from multiple locations via BGP. Routing automatically sends users to nearest server. Used for: DNS (Google's 8.8.8.8 is anycast across 20+ PoPs), CDN edge nodes, DDoS mitigation (absorb attack traffic across all locations!). Advantage: zero-config failover — if one location goes down, BGP routes traffic elsewhere automatically. Limitation: TCP connections break if routing changes (less of an issue for DNS which is UDP).

**Q4: How does IPv4 exhaustion affect system design?**
> IPv4 exhausted since 2019! Impact: (1) Cloud providers now charge for public IPv4 addresses (AWS: $3.65/month per IP), (2) Forces NAT usage (adds complexity, breaks P2P), (3) Drives IPv6 adoption (every device gets unique address). Design implications: minimize public IPs (use internal LBs, private subnets), consider IPv6 dual-stack, use Elastic IPs sparingly. Container orchestration (K8s) uses overlay networks to avoid IP exhaustion within clusters.

---

## 🔗 Related Topics
- [DNS](DNS.md) — Resolving names to IPs
- [Load Balancing](../../BuildingBlocks/LoadBalancing.md) — Distributing traffic across IPs
- [CDN](../../BuildingBlocks/CDN.md) — Anycast for content delivery
- [WebRTC](../../APIs/WebRTC.md) — NAT traversal challenges

---

*"An IP address is just a number. But understanding how those numbers are organized — subnets, private ranges, NAT, anycast — is what separates someone who 'deploys to the cloud' from someone who ARCHITECTS cloud infrastructure." — Network Engineer* 🌐
