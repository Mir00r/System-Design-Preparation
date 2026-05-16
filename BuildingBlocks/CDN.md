# 🌐 CDN: Delivering Content at the Speed of Light

> **"The internet is global. Your server is not. CDN bridges that gap."**

---

## 🎯 What You'll Learn
- Why geographic distance kills performance and how CDN solves it
- CDN architecture — PoPs, edge nodes, and origin servers
- Push vs Pull CDN strategies with real trade-offs
- Cache invalidation — the hardest problem in CDN
- How Netflix, Cloudflare, and Akamai use CDNs at scale

**⏱️ Estimated Time**: 18 minutes | **🎯 Difficulty**: 🟡 Medium  
**🔗 Prerequisites**: [Caching](./Caching.md) | [Load Balancing](./LoadBalancing.md)  
**🔗 Related Topics**: [Caching Strategies](./CachingStrategies.md) | [Blob Storage](./Blob_Storage.md) | [System Design: Netflix](../SystemDesignCaseStudies/DesignNetflix.md)

---

## 📋 Table of Contents
1. [The Problem: Geography Is the Enemy](#-the-problem-geography-is-the-enemy)
2. [What Is a CDN?](#-what-is-a-cdn)
3. [How CDN Works — Step by Step](#-how-cdn-works--step-by-step)
4. [CDN Architecture: PoPs and Edge Nodes](#-cdn-architecture-pops-and-edge-nodes)
5. [Push vs Pull CDN](#-push-vs-pull-cdn)
6. [Cache Invalidation](#-cache-invalidation)
7. [What Can (and Can't) Be Cached on CDN](#-what-can-and-cant-be-cached-on-cdn)
8. [CDN and Security](#-cdn-and-security)
9. [Tools & Providers](#-tools--providers)
10. [Industry Examples](#-industry-examples)
11. [Common Pitfalls](#-common-pitfalls)
12. [Mini Challenge](#-mini-challenge)
13. [Interview Q&A](#-interview-qa)
14. [What to Read Next](#-what-to-read-next)

---

## 🤔 The Problem: Geography Is the Enemy

Your server is in **Virginia, USA**. A user opens your app from **Sydney, Australia**.

```
WITHOUT CDN:

User in Sydney ──────────────────────────────► Server in Virginia
                                               ◄──────────────────
         Round trip: ~350ms (light-speed minimum)
         
But with network hops, routing, and processing:
         Real world: 500ms - 1200ms latency

For a typical web page (50+ requests): PAINFUL.
For video streaming: IMPOSSIBLE.
```

Now multiply that by **500 million users worldwide**. Your single Virginia server is not just slow — it's completely overwhelmed.

**The fix?** Bring the content closer to the user.

```
WITH CDN:

User in Sydney ──► Edge Node in Sydney ──► Content served locally
                   (cache hit)
                   
         Round trip: ~10-20ms
         
                   (cache miss — first time only)
Edge Node ──────────────────────────────────► Origin Server Virginia
           ◄────────────────────────────────
           Content fetched & cached for next user
```

> 💡 **Key Insight**: CDN doesn't make your servers faster — it makes the data **geographically closer** to the user. The speed of light is the physics ceiling we're working around.

---

## 💡 What Is a CDN?

A **Content Delivery Network (CDN)** is a geographically distributed network of servers (called **edge nodes** or **PoPs**) that cache and serve content to users from the nearest location.

```
CDN GLOBAL ARCHITECTURE:

         [ORIGIN SERVER — Virginia]
                    │
         ┌──────────┼──────────┐
         ▼          ▼          ▼
  [PoP: London] [PoP: Tokyo] [PoP: Sydney]
    Edge Nodes   Edge Nodes   Edge Nodes
         │            │            │
    EU Users    Asia Users    AU Users
    
Each PoP = cluster of servers in a data center
Each edge node = caches popular content
```

---

## 🌍 Real-World Analogy

Think of CDN like a **chain of convenience stores** vs **one central warehouse**:

```
WITHOUT CDN (Central Warehouse):
Every customer drives to the central warehouse in another city.
Fast if you're nearby. Unbearable if you're far.

WITH CDN (Convenience Stores):
Convenience store opened in every neighborhood.
Popular items stocked locally.
Only rare/new items need to come from the warehouse.
95% of purchases happen locally. Instant.
```

The "warehouse" = your origin server.  
The "convenience stores" = CDN edge nodes.  
The "popular items" = static files, images, videos, HTML.

---

## 🏗️ How CDN Works — Step by Step

### Cache Miss (First Request)

```
STEP 1: User requests image.png from example.com

STEP 2: DNS resolves example.com → CDN's IP (not origin!)
        DNS routing picks the NEAREST CDN PoP

STEP 3: Request hits CDN edge node in Sydney

STEP 4: Edge node checks cache → MISS (not cached yet)

STEP 5: Edge node fetches from origin server (Virginia)

STEP 6: Origin sends image.png + cache headers:
        Cache-Control: public, max-age=86400 (24 hours)

STEP 7: Edge node CACHES the file + serves it to user

STEP 8: All future users in Australia get it from Sydney edge
        (cache HIT — no round trip to Virginia)
```

### Cache Hit (Subsequent Requests)

```
STEP 1: Second user requests same image.png

STEP 2: DNS → Sydney edge node (same as before)

STEP 3: Edge node checks cache → HIT! ✅

STEP 4: Serve directly from edge node
        Total latency: ~10ms
        Origin server: not even consulted
```

---

## 🗺️ CDN Architecture: PoPs and Edge Nodes

### Points of Presence (PoPs)
A **PoP** is a physical data center location where CDN servers are deployed.

```
CLOUDFLARE (as of 2026):
310+ PoPs in 100+ countries

AKAMAI:
340,000+ servers in 4,100+ PoPs

AWS CLOUDFRONT:
450+ PoPs (including 13 regional edge caches)
```

### Multi-Tier CDN Architecture

Large CDNs have multiple tiers:

```
TIER 1: Origin Shield
  Origin Server ← acts as a single entry point for CDN

TIER 2: Regional Edge Cache
  Larger cache, fewer locations (~13 globally)
  If local PoP misses → check regional cache before going to origin

TIER 3: Edge PoP (local)
  Closest to user, smallest cache
  Most requests served here

REQUEST FLOW:
User → Edge PoP
  → (miss) → Regional Edge Cache
    → (miss) → Origin Shield
      → (always hits) → Origin Server
```

This tiering dramatically reduces origin load.

---

## ⬆️⬇️ Push vs Pull CDN

Two fundamental strategies for getting content onto CDN edge nodes:

### Pull CDN (Lazy / On-Demand)

Content is pulled from origin **on first request** from that edge node.

```
PULL CDN FLOW:

First request from Sydney → Edge PoP
  → Cache MISS → Pull from origin → Cache it
  
Second request from Sydney → Edge PoP
  → Cache HIT → Serve immediately
  
Cache TTL expires → Stale content removed
Next request → Pull fresh content again
```

**✅ Pros**:
- Simple to set up — just point CDN to your domain
- No need to push content manually
- Only popular content gets cached (storage efficient)

**❌ Cons**:
- First user in each region experiences origin latency (cold cache)
- If many PoPs miss simultaneously → origin thundering herd problem

**Best For**: Dynamic content sites, websites where not all content is accessed globally

---

### Push CDN (Eager / Pre-upload)

You proactively upload content to CDN edge nodes.

```
PUSH CDN FLOW:

You upload new_video.mp4 to CDN
CDN distributes it to ALL edge nodes globally
  
ANY user from ANY location → Instant cache HIT
No cold-start problem
No origin requests at all
```

**✅ Pros**:
- Zero cold-start — content is always cached everywhere
- No origin thundering herd
- Perfect for content that you know will be globally popular

**❌ Cons**:
- Must manage uploads (more operational work)
- Storage cost — even unpopular content is cached everywhere
- Need to manually purge when content changes

**Best For**: Large static files (software downloads, videos, datasets), content with known global demand

### Push vs Pull Comparison

| Aspect | Pull CDN | Push CDN |
|---|---|---|
| Setup | Simple | More complex |
| First-request latency | High (cache miss) | Zero (always cached) |
| Storage efficiency | High (cache only popular) | Low (cache everything) |
| Origin load | Low (after warmup) | Zero |
| Best for | Dynamic sites, partial global | Global video, software downloads |
| Examples | Most websites | Netflix, game downloads, OS updates |

---

## 🔄 Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

When you update content on origin, CDN edge nodes may still have the old version. How do you fix this?

### Strategy 1: TTL Expiry (Natural)
```
Cache-Control: public, max-age=3600   ← TTL = 1 hour
CDN serves old content for up to 1 hour, then re-fetches
```
Simple but slow invalidation. OK for content that rarely changes.

### Strategy 2: Cache Purge (Immediate)
```
API call to CDN: "Purge /images/logo.png from all edge nodes"
CDN immediately removes it from all caches
Next request → pulls fresh content from origin
```
Immediate but costs money per purge (most CDNs charge).

### Strategy 3: Versioned URLs (Best Practice) ✅
```
Instead of: /images/logo.png
Use:        /images/logo.v3.png  or  /images/logo.png?v=3

New version → new URL → CDN treats it as brand new content
Old URL still cached (old clients use old version)
No purge needed!
```
This is the **gold standard**. Zero invalidation cost, perfect cache efficiency.

### Strategy 4: Immutable Content
```
Cache-Control: public, max-age=31536000, immutable
# 1 year TTL, never changes
```
Use for hashed filenames (Webpack: `main.abc123.js`). The hash changes = new URL.

---

## 📦 What Can (and Can't) Be Cached on CDN

### ✅ Cache on CDN
```
Static assets:     JavaScript, CSS, fonts, images, icons
Media:             Videos, audio, PDFs, documents  
HTML (sometimes):  Marketing pages, blog posts
API responses:      GET /api/products (product catalog, rarely changes)
Software:          App downloads, OS updates, game patches
```

### ❌ Do NOT Cache on CDN
```
User-specific data:    Shopping cart, account info, order history
Authentication:        Login, tokens, session data
Real-time data:        Live stock prices, chat messages
Write operations:      POST/PUT/DELETE requests (CDN passes through)
Personalized content:  Recommendations, user feeds
```

---

## 🔐 CDN and Security

Modern CDNs are not just about performance — they're a security layer:

```
CDN SECURITY FEATURES:

1. DDoS Protection
   Absorb volumetric attacks (100+ Gbps) at the edge
   Cloudflare, Akamai have "infinite" scrubbing capacity

2. Web Application Firewall (WAF)
   Block SQL injection, XSS, OWASP Top 10 at the edge
   Before requests ever reach your origin

3. SSL/TLS Termination
   CDN handles HTTPS certificates (auto-renewed with Let's Encrypt)
   Origin ↔ CDN connection can use HTTP internally (or HTTPS with internal cert)

4. Bot Protection
   Challenge suspicious traffic (CAPTCHA, JS challenges)
   Block scrapers, credential stuffing

5. IP Reputation
   Block known malicious IPs globally
```

> 💡 **Key Insight**: Cloudflare's "Magic Transit" and "Spectrum" products are proof that CDN is now a full security platform, not just a cache.

---

## 🛠️ Tools & Providers

| Provider | Best For | Key Differentiator |
|---|---|---|
| **AWS CloudFront** | AWS ecosystem | Deep S3/ALB/Lambda@Edge integration |
| **Cloudflare** | Security + performance | Workers (edge computing), strongest DDoS protection |
| **Akamai** | Enterprise | Largest PoP network, enterprise SLAs |
| **Fastly** | Real-time, dynamic | Instant purge (<150ms), VCL-based customization |
| **Google Cloud CDN** | GCP ecosystem | Tight Google Cloud integration |
| **Azure CDN** | Azure ecosystem | Tight Azure integration |
| **BunnyCDN** | Cost-optimized | Cheapest pricing for startups/side projects |

---

## 🏢 Industry Examples

```
NETFLIX:
- Built their own CDN: "Netflix Open Connect" (NOC)
- Partners with ISPs to place servers INSIDE their networks
- 15,000+ servers in 1,000+ ISP locations worldwide
- 95%+ of Netflix traffic served from ISP-embedded servers
- Why? To avoid paying ISP transit costs at their scale

CLOUDFLARE:
- 1 trillion+ requests per day served
- 310+ cities — they're in more places than any cloud provider
- "Cloudflare Workers" runs code at the edge (in 310+ cities)

FACEBOOK/META:
- Uses Akamai + own CDN for global media delivery
- Profile pictures, videos, attachments all CDN-served
- "Haystack" is their internal media storage + CDN system

STEAM (VALVE):
- Delivers 40+ TB/s during major game launches
- Uses Akamai + own caching infrastructure
- Without CDN, a single game launch would take days per user
```

---

## ⚠️ Common Pitfalls

### 1. Not Using CDN for APIs
```
❌ BAD: Only using CDN for images, not for cacheable API responses
✅ GOOD: Cache GET /api/products (changes every 5 mins) on CDN
         5 minutes cache × millions of users = massive origin savings
```

### 2. Forgetting Cache Headers
```
❌ BAD: No Cache-Control header → CDN doesn't know what to cache
✅ GOOD: Set explicit Cache-Control headers on every response
         Cache-Control: public, max-age=3600      ← 1 hour
         Cache-Control: no-store                  ← never cache (user data)
         Cache-Control: no-cache                  ← cache but revalidate
```

### 3. Caching User-Specific Content
```
❌ BAD: CDN caches GET /user/profile → User A sees User B's data!
✅ GOOD: Use Vary: Authorization or Cache-Control: private
         Or don't put user-specific endpoints behind CDN
```

### 4. Long TTLs Without Versioned URLs
```
❌ BAD: logo.png cached for 1 year, you update it → old logo shows for 1 year
✅ GOOD: logo.v2.png (new URL) → instant new content worldwide
```

### 5. Origin Not Handling CDN Miss Load
```
❌ BAD: CDN cache expires simultaneously on many nodes → stampede to origin
✅ GOOD: Cache-Control: stale-while-revalidate=60 
         "Serve stale content for 60s while background-refresh"
         Origin Shield (single entry point to origin)
```

---

## 🧩 Mini Challenge

```
🎲 SCENARIO (3 minutes):

You're building a news website. Your tech choices:
  - Articles: HTML pages (rarely change, maybe every hour)
  - Breaking news ticker: Updates every 30 seconds
  - User comments: Real-time, unique per user session
  - Site logo/CSS/JS: Changes every deployment

QUESTION: 
Design the CDN caching strategy for each content type.
What TTL? Push or Pull? Version URLs?
```

<details>
<summary>💡 Click to reveal answer</summary>

| Content | Strategy | Reasoning |
|---|---|---|
| **Articles (HTML)** | Pull CDN, TTL=1hr, `Cache-Control: public, max-age=3600` | Cacheable, changes hourly, Pull is fine since not ALL articles are globally popular |
| **Breaking news ticker** | CDN **bypass** or TTL=30s | Updates every 30s → CDN overhead not worth it. Use WebSocket or polling directly to origin |
| **User comments** | **No CDN** — `Cache-Control: private, no-store` | User-specific, real-time. MUST NOT cache on CDN |
| **Logo/CSS/JS** | Push CDN with **versioned URLs** + `Cache-Control: immutable, max-age=31536000` | Changes only on deploy. Hash the filenames (`main.abc123.js`). 1-year TTL since URL changes = new content |

**Key insight**: Different content types need completely different CDN strategies within the same site.
</details>

---

## 📝 Interview Q&A

**Q1: What is a CDN and why do we use it?**
> A CDN is a distributed network of edge servers geographically close to users. We use it to reduce latency (data travels shorter distance), reduce origin server load (edge serves cached content), and improve availability (edge servers provide redundancy).

**Q2: What is the difference between Push and Pull CDN?**
> Pull: content fetched from origin on first cache miss, then cached. Simple but cold-start latency. Pull: you pre-upload content to CDN. Zero cold-start but more operational overhead and storage cost. Pull is most common; push for predictably popular global content (software downloads, video).

**Q3: How do you handle cache invalidation in a CDN?**
> Best approach: versioned/hashed URLs. When content changes, the URL changes, so CDN treats it as new content — no invalidation needed. For content with fixed URLs, use CDN purge API (immediate but may cost money) or rely on TTL expiry for acceptable staleness.

**Q4: What is an Origin Shield?**
> An Origin Shield is an additional CDN tier that acts as a single point through which all CDN PoPs contact your origin server. Instead of 300 edge nodes making separate requests to your origin on a cache miss, they all go through the Shield. Dramatically reduces origin load.

**Q5: Can CDN help with API responses, not just static files?**
> Yes. GET requests with stable, non-user-specific responses can be cached at CDN. Example: `/api/product-catalog` (changes every 5 mins). Set `Cache-Control: public, max-age=300`. CDN serves millions of users from cache, reducing DB queries massively.

---

## 🔗 What to Read Next

| Topic | Why You Need It |
|---|---|
| [Caching Strategies](./CachingStrategies.md) | CDN is one caching layer — understand all caching layers (in-memory, DB, CDN) together |
| [Blob Storage](./Blob_Storage.md) | CDN typically sits in front of object storage (S3). Know the full stack |
| [System Design: Netflix](../SystemDesignCaseStudies/DesignNetflix.md) | CDN is the #1 topic when designing Netflix — apply everything you learned here |

---

*[← Back to Building Blocks](../BuildingBlocks/) | [← Index](../INDEX.md)*
