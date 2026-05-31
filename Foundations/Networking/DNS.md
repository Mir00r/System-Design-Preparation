# 🌍 Domain Name System (DNS): The Internet's Phone Book

> *"Every time you type google.com, your computer asks 'Hey, what's Google's phone number?' and DNS responds '142.250.80.46.' This happens billions of times per second globally. When Cloudflare's DNS went down in 2022, half the internet became unreachable — not because servers died, but because nobody could find them."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [OSI Model](./OSI_Model.md), [IP Addresses](./IP_Addresses.md)

---

## 📋 Table of Contents
1. [What is DNS?](#-what-is-dns)
2. [DNS Resolution — Step by Step](#-dns-resolution--step-by-step)
3. [DNS Record Types](#-dns-record-types)
4. [DNS Caching — Speed Boost](#-dns-caching--speed-boost)
5. [DNS Architecture — The Hierarchy](#-dns-architecture--the-hierarchy)
6. [DNS in System Design](#-dns-in-system-design)
7. [DNS Security (DNSSEC)](#-dns-security-dnssec)
8. [Real-World DNS at Scale](#-real-world-dns-at-scale)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is DNS?

```
╔══════════════════════════════════════════════════════════════════╗
║  DNS = Domain Name System                                      ║
║                                                                ║
║  Translates human-readable domain names (google.com)           ║
║  into machine-readable IP addresses (142.250.80.46)            ║
║                                                                ║
║  Without DNS, you'd need to memorize IP addresses              ║
║  for every website. Imagine typing 142.250.80.46               ║
║  instead of google.com! 😱                                    ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Phone Book Analogy

```
Before DNS (1980s):                  With DNS (today):
┌──────────────────────┐            ┌──────────────────────┐
│  hosts.txt file:     │            │  DNS Server:         │
│  172.16.0.1 = MIT    │            │  google.com → IP     │
│  192.0.0.1 = BBN    │            │  netflix.com → IP    │
│  ...                 │            │  twitter.com → IP    │
│  (manually updated!) │            │  (automatic!)        │
└──────────────────────┘            └──────────────────────┘

In 1983, there were only ~300 computers on the internet.
Today there are 5+ BILLION devices. Imagine maintaining 
one hosts.txt file for all of them! 📚

DNS was invented in 1983 by Paul Mockapetris to solve 
this exact scalability problem.
```

---

## 🔄 DNS Resolution — Step by Step

```
What happens when you type "www.google.com" in your browser:

┌──────────┐  ①  ┌──────────────┐  ②  ┌──────────────┐
│  Browser │────►│ Local DNS    │────►│ Root DNS     │
│          │     │ Resolver     │     │ (.)          │
└──────────┘     │ (ISP/8.8.8.8)│     └──────┬───────┘
                 └──────┬───────┘            │
                        │                    │ ③ "Ask .com servers"
                        │                    ▼
                        │         ┌──────────────────┐
                        │         │ TLD DNS (.com)   │
                        │         └──────┬───────────┘
                        │                │
                        │                │ ④ "Ask Google's servers"
                        │                ▼
                        │         ┌──────────────────┐
                        │  ⑤      │ Authoritative DNS│
                        │◄────────│ (ns1.google.com) │
                        │  IP!    └──────────────────┘
                        │
                 ┌──────▼───────┐  ⑥  ┌──────────┐
                 │ Returns IP   │────►│  Browser  │
                 │ 142.250.80.46│     │ connects! │
                 └──────────────┘     └──────────┘

Step-by-step:
① Browser asks local resolver: "What's www.google.com's IP?"
② Resolver asks Root: "Who handles .com?"
③ Root says: "Ask the .com TLD server at 192.5.6.30"
④ Resolver asks .com TLD: "Who handles google.com?"
⑤ TLD says: "Ask Google's authoritative server at ns1.google.com"
⑥ Resolver asks ns1.google.com: "What's www.google.com?"
⑦ Authoritative answers: "142.250.80.46" with TTL=300s
⑧ Resolver caches the answer and returns it to browser!

Total time: 20-120ms (first time), <1ms (cached) ⚡
```

---

## 📝 DNS Record Types

```
┌──────────┬──────────────────────────────────────────────────────┐
│  Type    │  Purpose & Example                                   │
├──────────┼──────────────────────────────────────────────────────┤
│  A       │  Maps domain → IPv4 address                          │
│          │  google.com → 142.250.80.46                          │
├──────────┼──────────────────────────────────────────────────────┤
│  AAAA    │  Maps domain → IPv6 address                          │
│          │  google.com → 2607:f8b0:4004:800::200e               │
├──────────┼──────────────────────────────────────────────────────┤
│  CNAME   │  Alias (one domain → another domain)                 │
│          │  www.google.com → google.com                          │
├──────────┼──────────────────────────────────────────────────────┤
│  MX      │  Mail server for the domain                          │
│          │  google.com → smtp.google.com (priority 10)          │
├──────────┼──────────────────────────────────────────────────────┤
│  NS      │  Authoritative nameserver for the domain             │
│          │  google.com → ns1.google.com                          │
├──────────┼──────────────────────────────────────────────────────┤
│  TXT     │  Arbitrary text (SPF, DKIM, verification)            │
│          │  google.com → "v=spf1 include:_spf.google.com"       │
├──────────┼──────────────────────────────────────────────────────┤
│  SRV     │  Service location (host + port)                      │
│          │  _http._tcp.example.com → server1:8080               │
├──────────┼──────────────────────────────────────────────────────┤
│  PTR     │  Reverse lookup (IP → domain)                        │
│          │  46.80.250.142.in-addr.arpa → google.com             │
└──────────┴──────────────────────────────────────────────────────┘
```

### 🎮 Record Type Quick Quiz

> **Q**: You want `blog.company.com` to point to the same server as `company.com`. Which record type?
> **A**: CNAME! `blog.company.com CNAME company.com`

> **Q**: Your email isn't working. Which DNS record should you check?
> **A**: MX record! Tells mail servers where to deliver email.

---

## ⚡ DNS Caching — Speed Boost

```
DNS without caching:                DNS with caching:
  Every request → full resolution     First request → full resolution
  50-100ms each time ❌               Subsequent → <1ms from cache ✅

CACHING LAYERS:
┌─────────────────────────────────────────────────────────────────┐
│  Layer              │  Cache Duration  │  Controlled By          │
├─────────────────────┼──────────────────┼─────────────────────────┤
│  Browser cache      │  Seconds-Minutes │  HTTP headers           │
│  OS cache           │  Minutes         │  OS DNS settings        │
│  Router cache       │  Minutes-Hours   │  Router firmware        │
│  ISP resolver cache │  TTL value       │  Domain owner (TTL)     │
└─────────────────────┴──────────────────┴─────────────────────────┘

TTL (Time To Live): How long a DNS record can be cached
  - Low TTL (30-60s):  Fast changes, more DNS queries (costlier)
  - High TTL (3600s):  Fewer queries, but changes take 1 hour!
  
🎯 SYSTEM DESIGN TIP:
  For failover: Use LOW TTL (30-60s) so DNS changes propagate fast
  For stable sites: Use HIGH TTL (1-24 hours) to reduce latency
```

---

## 🏛️ DNS Architecture — The Hierarchy

```
                         ROOT SERVERS (.)
                         13 logical servers (A-M)
                         1000+ physical machines worldwide
                              │
           ┌──────────────────┼──────────────────────┐
           │                  │                      │
        .com TLD          .org TLD              .io TLD
        (VeriSign)        (PIR)                (Afilias)
           │                  │                      │
     ┌─────┼─────┐      ┌────┼────┐            ┌────┼────┐
     │     │     │      │    │    │            │    │    │
  google amazon fb   wikipedia  ...          github  ...
   .com   .com  .com    .org                    .io
     │
     ├── ns1.google.com (Authoritative)
     ├── ns2.google.com (Authoritative)
     ├── ns3.google.com (Authoritative)
     └── ns4.google.com (Authoritative)

FUN FACTS:
  - There are only 13 root server ADDRESSES (a.root-servers.net to m)
  - But 1,700+ physical root servers worldwide (Anycast!)
  - Root servers handle ~620 BILLION queries per day
  - If all 13 went down, the internet would still work...
    for a while (thanks to caching!)
```

---

## 🏗️ DNS in System Design

### Load Balancing with DNS (Round-Robin)

```
Query: api.company.com → ?

DNS Response (rotates each query):
  Query 1: api.company.com → 10.0.1.1 (Server A)
  Query 2: api.company.com → 10.0.1.2 (Server B)
  Query 3: api.company.com → 10.0.1.3 (Server C)
  Query 4: api.company.com → 10.0.1.1 (Server A again)
  
Pros: Simple, no load balancer hardware needed
Cons: Can't detect unhealthy servers, uneven distribution
      (clients cache one IP and keep hitting same server)
```

### GeoDNS (Route to Nearest Server)

```
User in Tokyo:  api.company.com → 10.0.1.1 (Tokyo DC)
User in NYC:    api.company.com → 10.0.2.1 (Virginia DC)
User in London: api.company.com → 10.0.3.1 (Ireland DC)

HOW: DNS server checks source IP geolocation and returns 
     the nearest data center's IP address.

USED BY: Netflix, Google, CDNs (CloudFlare, Akamai)
```

### DNS Failover Pattern

```
Normal operation:
  api.company.com → 10.0.1.1 (Primary, us-east-1)
  Health check: ✅ Primary healthy

After primary failure:
  Health check: ❌ Primary unhealthy!
  DNS automatically updates:
  api.company.com → 10.0.2.1 (Backup, us-west-2)
  
⚠️ CAVEAT: DNS TTL determines how fast clients see the change!
  TTL=3600 → Users might hit dead server for up to 1 HOUR
  TTL=60   → Most users switch within 1-2 minutes
```

---

## 🔒 DNS Security (DNSSEC)

```
THE PROBLEM — DNS Cache Poisoning:
  
  Hacker intercepts DNS response:
  ┌────────┐     ┌──────────┐     ┌───────────┐
  │  User  │────►│ Resolver │────►│ REAL DNS  │
  └────────┘     └─────┬────┘     └───────────┘
                       │
                  ┌────▼────┐
                  │ HACKER  │ ← Sends fake response FASTER!
                  │ "bank.com│    "bank.com = EVIL_IP"
                  │ = evil!" │
                  └─────────┘
  
  Result: User visits fake bank website, enters credentials! 😱

DNSSEC Solution:
  - Every DNS record is SIGNED with cryptographic key
  - Resolver VERIFIES signature before accepting
  - If signature invalid → REJECT (prevents poisoning)
  
  bank.com A 142.250.80.46  RRSIG=abc123...  (signed by bank.com's key)
  └── Resolver checks: Does abc123 match? YES → Trust. NO → Reject.
```

---

## 🏢 Real-World DNS at Scale

### AWS Route 53
```
Features:
  - Health checks (auto-remove unhealthy endpoints)
  - Latency-based routing (route to fastest endpoint)
  - Geolocation routing (route by user country)
  - Weighted routing (send 90% to v2, 10% to v3 for canary)
  - Failover routing (primary/secondary with health checks)
  
Netflix uses Route 53 for:
  - Regional routing (US users → US servers)
  - Failover between AWS regions
  - Blue-green deployments (instant traffic switching)
```

### Cloudflare DNS (1.1.1.1)
```
- Fastest public DNS resolver (~11ms average)
- Anycast (same IP, 300+ locations worldwide)
- Privacy-focused (doesn't sell query data)
- Handles 1+ TRILLION queries per day
- When it goes down... major internet disruption!
```

---

## ⚠️ Common Pitfalls

| Pitfall | Impact | Fix |
|---------|--------|-----|
| 🔴 High TTL for dynamic services | Failover takes hours | Set TTL to 30-60s for services needing fast failover |
| 🔴 DNS as single point of failure | Entire app unreachable | Use multiple DNS providers (Route53 + Cloudflare) |
| 🔴 Not monitoring DNS | Silent failures go unnoticed | Monitor resolution time and NXDOMAIN rates |
| 🟡 CNAME at zone apex | RFC violation, breaks things | Use ALIAS/ANAME records (provider-specific) |
| 🟡 Ignoring DNS propagation | Changes not instant | Understand TTL + cache behavior |

---

## 🎮 Mini Challenge

### 🧩 DNS Detective

Your website `myapp.com` is unreachable. Users see "DNS_PROBE_FINISHED_NXDOMAIN". Debug it:

```bash
# Step 1: Check if DNS resolves at all
$ nslookup myapp.com
# → Server: 8.8.8.8, Answer: NXDOMAIN

# Step 2: Check authoritative nameservers
$ dig NS myapp.com
# → What do you look for?

# Step 3: Check if nameservers respond
$ dig @ns1.myapp.com myapp.com A
# → What should you see?
```

**Questions:**
1. What does NXDOMAIN mean?
2. If `dig NS` returns nothing, what's the problem?
3. Your domain expired yesterday. Which DNS records disappear first?

<details>
<summary>🔑 Answers</summary>

1. NXDOMAIN = "Non-Existent Domain" — No DNS record exists for this domain
2. If NS records are missing, your domain's nameservers aren't configured (registrar issue or expired domain)
3. The NS records at the TLD (.com) level disappear, making your domain unreachable even if your nameservers still have records
</details>

---

## ❓ Interview Q&A

**Q1: What is DNS and why is it important?**
> DNS translates human-readable domain names to IP addresses. It's the first step of every internet request. Without DNS, users would need to memorize IP addresses. It's critical infrastructure — if DNS fails, applications become unreachable even if servers are healthy.

**Q2: Explain DNS resolution step by step.**
> Browser checks cache → OS cache → Router cache → ISP resolver. If no cache hit: Resolver queries Root DNS (which .com TLD to ask), then TLD DNS (which authoritative server to ask), then authoritative DNS (gets actual IP). Response is cached at each layer based on TTL.

**Q3: How would you use DNS for high availability?**
> Multiple strategies: (1) DNS failover with health checks (Route53), (2) GeoDNS for routing to nearest healthy DC, (3) Low TTL for fast failover propagation, (4) Multiple DNS providers for redundancy, (5) Round-robin for simple load distribution.

**Q4: What's the difference between A, CNAME, and ALIAS records?**
> A record: maps domain directly to IP. CNAME: maps domain to another domain (alias, can't be used at zone apex). ALIAS: provider-specific record that works like CNAME but at zone apex (resolves at DNS server, returns A record to client).

**Q5: What is DNS TTL and how does it affect system design?**
> TTL = how long DNS responses are cached. Low TTL (30-60s): fast failover but more DNS queries (cost/latency). High TTL (hours): fewer queries but slow propagation of changes. For systems needing fast failover, use low TTL. For stable services, high TTL reduces latency.

---

## 🔗 Related Topics
- [IP Addresses](./IP_Addresses.md) — What DNS resolves TO
- [Load Balancing](../../BuildingBlocks/LoadBalancing.md) — Often works with DNS
- [CDN](../../BuildingBlocks/CDN.md) — Uses DNS for geographic routing
- [Availability](../../KeyConcepts/Availability.md) — DNS failures impact availability

---

*"DNS is the most critical single-point-of-failure on the internet that nobody thinks about... until it breaks." — Every SRE after a DNS outage* 🌍
