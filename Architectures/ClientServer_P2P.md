# 🏛️ Client-Server & Peer-to-Peer Architecture

> *"Every time you open Netflix, your browser (client) talks to Netflix's servers. Every time you download via BitTorrent, your computer talks to OTHER computers (peers) — no central server! These two architectural paradigms shape how 99% of internet systems are built. Understanding when to use each (and when to HYBRID them) is fundamental to system design."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [Networking Basics](../../Foundations/Networking/DNS.md)

---

## 📋 Table of Contents
1. [Client-Server Architecture](#-client-server-architecture)
2. [Peer-to-Peer Architecture](#-peer-to-peer-architecture)
3. [Comparison](#-comparison)
4. [Hybrid Approaches](#-hybrid-approaches)
5. [Real-World Examples](#-real-world-examples)
6. [Java Implementation](#-java-implementation)
7. [Interview Q&A](#-interview-qa)

---

## 🖥️ Client-Server Architecture

```
THE MOST COMMON ARCHITECTURE ON THE INTERNET!

  ┌────────────┐         ┌────────────┐
  │  Client A  │────────►│            │
  └────────────┘         │            │
  ┌────────────┐         │   SERVER   │ (centralized!)
  │  Client B  │────────►│            │
  └────────────┘         │  Processes │
  ┌────────────┐         │  requests, │
  │  Client C  │────────►│  stores    │
  └────────────┘         │  data      │
                         └────────────┘

  Client: Makes requests, presents UI
  Server: Processes requests, manages data, enforces rules

CHARACTERISTICS:
  ✅ Centralized control (server manages everything)
  ✅ Easy to update (deploy on server → all clients get new version)
  ✅ Security: server validates all actions
  ✅ Single source of truth (data consistency!)
  ❌ Single point of failure (server down = everything down!)
  ❌ Scalability bottleneck (all traffic → one place)
  ❌ Server cost: YOU pay for bandwidth & compute

TIERS:
  1-Tier: Client + server on same machine (desktop app)
  2-Tier: Client → Server/DB (traditional web)
  3-Tier: Client → App Server → Database (modern standard!)
  N-Tier: Client → LB → App → Cache → DB → ... (distributed!)

  Modern web:
  ┌──────┐    ┌─────┐    ┌─────┐    ┌────┐
  │ React│───►│ API │───►│Cache│───►│ DB │
  │ App  │    │(REST)│    │(Redis)│  │(PG)│
  └──────┘    └─────┘    └─────┘    └────┘
  client       server      server    server
```

---

## 🌐 Peer-to-Peer Architecture

```
NO CENTRAL SERVER! Every node is both client AND server!

  ┌────────────┐         ┌────────────┐
  │   Peer A   │◄───────►│   Peer B   │
  └──────┬─────┘         └─────┬──────┘
         │                      │
         │    ┌────────────┐    │
         └───►│   Peer C   │◄──┘
              └──────┬─────┘
                     │
              ┌──────▼─────┐
              │   Peer D   │
              └────────────┘

  Each peer: stores data, forwards requests, serves content!
  No single point of failure! If Peer A dies → others continue!

CHARACTERISTICS:
  ✅ No single point of failure (decentralized!)
  ✅ Scales naturally (more users = more capacity!)
  ✅ Low cost (users provide bandwidth & storage)
  ✅ Censorship resistant (no central authority to shut down)
  ❌ Complex: discovery, coordination, consistency
  ❌ Security: hard to verify content (malicious peers!)
  ❌ Performance: inconsistent (peer has slow internet?)
  ❌ Data availability depends on peers being online!

P2P TOPOLOGIES:
  ┌────────────────────────────────────────────────────────────┐
  │  Type            │  How Peers Find Each Other              │
  ├────────────────────────────────────────────────────────────┤
  │  Unstructured    │  Random connections, flood queries      │
  │  (Gnutella)      │  Simple but wasteful!                   │
  ├────────────────────────────────────────────────────────────┤
  │  Structured      │  DHT (Distributed Hash Table)           │
  │  (Chord, Kademlia)│  Key → responsible peer. Efficient!    │
  ├────────────────────────────────────────────────────────────┤
  │  Hybrid          │  Central tracker + P2P data transfer    │
  │  (BitTorrent)    │  Best of both worlds!                   │
  └────────────────────────────────────────────────────────────┘

DISTRIBUTED HASH TABLE (DHT):
  How to find data without a central server?
  
  Each peer "owns" a range of keys (like consistent hashing!)
  Key "file_ABC" → hash → responsible peer is Node 42!
  
  Lookup: O(log N) hops through the network!
  Node 1 → "Who has key X?" → Node 15 → Node 38 → Node 42 → "I have it!"
```

---

## ⚖️ Comparison

```
┌────────────────────────────────────────────────────────────────────┐
│  Aspect           │  Client-Server       │  Peer-to-Peer          │
├────────────────────────────────────────────────────────────────────┤
│  Central control  │  YES (server owner)  │  NO (decentralized!)   │
│  Scalability      │  Limited by server   │  Natural (more peers!) │
│  Cost             │  Server owner pays   │  Distributed among peers│
│  Data consistency │  Easy (single source)│  Hard (distributed!)   │
│  Security         │  Easier (centralized)│  Harder (untrusted!)   │
│  Reliability      │  SPOF risk!          │  No SPOF!              │
│  Performance      │  Predictable         │  Variable (peer quality)│
│  Updates          │  Easy (server-side)  │  Hard (every peer!)    │
│  Discovery        │  Known server address│  Complex (DHT, gossip) │
│  Latency          │  Depends on server   │  Potentially lower     │
│                   │  distance            │  (nearby peers!)       │
│  Use cases        │  Web apps, APIs,     │  File sharing, crypto, │
│                   │  SaaS, streaming     │  video calls, CDN      │
└────────────────────────────────────────────────────────────────────┘

WHEN TO USE WHICH:

  Client-Server:
  • You need centralized control (business logic, rules)
  • Data consistency is critical (banking, e-commerce)
  • You want easy deployment and updates
  • Security/authentication is important
  → 95% of web applications!

  Peer-to-Peer:
  • You need massive scalability without server cost
  • Decentralization is a requirement (blockchain!)
  • Content distribution (large files to many users)
  • Real-time communication (video calls, gaming)
  → BitTorrent, WebRTC, cryptocurrency, IPFS
```

---

## 🔄 Hybrid Approaches

```
MOST MODERN SYSTEMS ARE HYBRID!

  BITTORRENT (Hybrid P2P):
  ┌────────────┐
  │  Tracker   │ ← Centralized! (knows which peers have which files)
  │  Server    │
  └─────┬──────┘
        │ "Who has file X?"
        │ "Peers A, B, C have chunks!"
        ▼
  ┌────────────┐     ┌────────────┐
  │   Peer A   │◄───►│   Peer B   │ ← Data transfer is P2P!
  └────────────┘     └────────────┘

  DISCORD / ZOOM (Hybrid Real-time):
  • Signaling: Client → Server (WebSocket)
  • Media: Peer → SFU Server → Peer (hybrid!)
  • Small calls: P2P (WebRTC direct!)
  • Large calls: Server-mediated (SFU)
  
  SPOTIFY (Hybrid Streaming):
  • Metadata: Client-Server (search, playlists)
  • Audio delivery: Was P2P (peers cache + share songs)
  • Now mostly CDN-based (cheaper than P2P overhead!)
  
  BLOCKCHAIN (Hybrid):
  • Consensus: P2P (all nodes validate!)
  • Light clients: Client-Server (query full nodes)
  • Infrastructure: Full nodes are "servers" for light clients

CDN AS P2P-LIKE DISTRIBUTION:
  CDN is essentially "server → many edge nodes → clients"
  Like P2P but with controlled, trusted nodes!
  
  ┌────────────┐
  │   Origin   │ (your server)
  └──────┬─────┘
    ┌────┼────┬────────┐
    ▼    ▼    ▼        ▼
  [Edge][Edge][Edge] [Edge] (CDN nodes = like "super peers"!)
    │     │     │      │
  Users  Users Users  Users
```

---

## 🌍 Real-World Examples

```
CLIENT-SERVER EXAMPLES:
  ┌───────────────────────────────────────────────────────────────┐
  │  System        │  Client              │  Server               │
  ├───────────────────────────────────────────────────────────────┤
  │  Gmail         │  Browser/App         │  Google servers        │
  │  Netflix       │  Smart TV/Browser    │  Netflix CDN + API     │
  │  Uber          │  Mobile app          │  Uber backend          │
  │  Slack         │  Desktop/Mobile      │  Slack infra           │
  │  Instagram     │  Mobile app          │  Meta servers          │
  └───────────────────────────────────────────────────────────────┘

PEER-TO-PEER EXAMPLES:
  ┌───────────────────────────────────────────────────────────────┐
  │  System        │  What Peers Do                               │
  ├───────────────────────────────────────────────────────────────┤
  │  BitTorrent    │  Share file chunks with each other           │
  │  Bitcoin       │  Validate transactions, maintain ledger      │
  │  IPFS          │  Store & serve content (decentralized web!)  │
  │  WebRTC calls  │  Stream audio/video directly to each other   │
  │  Skype (old)   │  Relay calls through supernodes              │
  └───────────────────────────────────────────────────────────────┘

HYBRID EXAMPLES:
  ┌───────────────────────────────────────────────────────────────┐
  │  System        │  Server Part         │  P2P Part             │
  ├───────────────────────────────────────────────────────────────┤
  │  Zoom          │  Signaling, auth     │  WebRTC small calls   │
  │  BitTorrent    │  Tracker             │  File chunk transfer  │
  │  Blockchain    │  Explorer, APIs      │  Consensus, relay     │
  │  Gaming        │  Matchmaking, auth   │  Game state sync      │
  └───────────────────────────────────────────────────────────────┘
```

---

## 💻 Java Implementation

### Simple Client-Server (REST)

```java
// SERVER (Spring Boot)
@RestController
@RequestMapping("/api/messages")
public class MessageController {
    
    @Autowired private MessageService messageService;
    
    @PostMapping
    public ResponseEntity<Message> send(@RequestBody MessageRequest req) {
        Message saved = messageService.save(req);
        return ResponseEntity.ok(saved);
    }
    
    @GetMapping("/{userId}")
    public List<Message> getMessages(@PathVariable String userId) {
        return messageService.getForUser(userId);
    }
}

// CLIENT (RestTemplate)
public class MessageClient {
    private final RestTemplate rest = new RestTemplate();
    private final String serverUrl = "http://api.example.com";
    
    public Message sendMessage(String to, String content) {
        return rest.postForObject(
            serverUrl + "/api/messages",
            new MessageRequest(to, content),
            Message.class);
    }
}
```

### Simple P2P Node

```java
/**
 * Simplified P2P node that can both serve and request data.
 * Each node maintains a list of known peers.
 */
public class P2PNode {
    
    private final String nodeId;
    private final int port;
    private final Set<PeerAddress> knownPeers = ConcurrentHashMap.newKeySet();
    private final Map<String, byte[]> localStore = new ConcurrentHashMap<>();
    
    /**
     * Store data locally AND announce to peers.
     */
    public void put(String key, byte[] data) {
        localStore.put(key, data);
        // Announce to peers: "I have key X!"
        for (PeerAddress peer : knownPeers) {
            announceAsync(peer, key);
        }
    }
    
    /**
     * Get data: check locally first, then ask peers!
     */
    public byte[] get(String key) {
        // 1. Check local store
        byte[] local = localStore.get(key);
        if (local != null) return local;
        
        // 2. Ask peers (fan-out query!)
        for (PeerAddress peer : knownPeers) {
            byte[] result = queryPeer(peer, key);
            if (result != null) {
                localStore.put(key, result); // Cache locally!
                return result;
            }
        }
        
        return null; // Not found in network!
    }
    
    /**
     * Discovery: learn about new peers through gossip.
     */
    public void gossip() {
        // Pick random peer, exchange known peer lists
        PeerAddress randomPeer = pickRandom(knownPeers);
        Set<PeerAddress> theirPeers = exchangePeerList(randomPeer);
        knownPeers.addAll(theirPeers); // Learn new peers!
    }
    
    /**
     * Handle incoming request from another peer.
     */
    public byte[] handleRequest(String key) {
        return localStore.get(key); // Serve if we have it!
    }
}
```

---

## ❓ Interview Q&A

**Q1: Why is client-server dominant for web applications?**
> Three reasons: (1) Centralized security — server validates all actions, clients can't be trusted (users can modify client code!), (2) Data consistency — single source of truth, no conflict resolution needed, (3) Ease of updates — deploy on server once, all users get new version. P2P adds complexity (discovery, security, consistency) that most web apps don't need. The trade-off (SPOF, scalability) is solved with redundancy + horizontal scaling.

**Q2: When would you choose P2P over client-server?**
> When: (1) Content distribution to millions (BitTorrent: users provide bandwidth, not you!), (2) Real-time communication (WebRTC: direct connection = lowest possible latency), (3) Censorship resistance (blockchain: no authority can shut down the network), (4) Massive scale without server cost (CDN-like distribution using users' devices). Key insight: P2P is about distributing cost and removing single points of control, not about performance.

**Q3: How does WebRTC work as a P2P protocol in the browser?**
> WebRTC allows browser-to-browser direct connections, but still needs servers for: (1) Signaling — exchanging connection info (SDP offers/answers via WebSocket server), (2) NAT traversal — STUN/TURN servers help peers behind firewalls discover their public address. After signaling: direct P2P connection for audio/video/data. If both peers are behind symmetric NAT: TURN relay server is needed (not truly P2P but looks like it to the app).

**Q4: What are the security challenges of P2P systems?**
> (1) Sybil attacks: one attacker creates 1000 fake peers to control the network, (2) Poisoning: malicious peer serves corrupted data (solution: content-addressable hashing — verify hash matches!), (3) Eclipse attacks: isolate a node by controlling all its connections, (4) No central auth: who do you trust? (solution: reputation systems, cryptographic verification), (5) Privacy: peers know each other's IPs (solution: onion routing like Tor).

**Q5: How does BitTorrent achieve better download speeds than client-server?**
> A file is split into chunks (e.g., 256KB pieces). Instead of one server sending the entire file: 100 peers each send different chunks simultaneously! You download from many peers in parallel → aggregate bandwidth = sum of all peers' upload speeds! Also: "tit-for-tat" incentivizes sharing (upload more → get faster downloads). Rare chunks are prioritized (rarest-first strategy) to ensure all chunks are available somewhere in the swarm.

---

## 🔗 Related Topics
- [Load Balancing](../../BuildingBlocks/LoadBalancing.md) — Scaling client-server
- [CDN](../../BuildingBlocks/CDN.md) — Distributed content delivery
- [WebRTC](../../APIs/WebRTC.md) — P2P in the browser
- [Gossip Protocol](../../Microservices/GossipProtocol.md) — P2P coordination
- [Service Discovery](../../BuildingBlocks/ServiceDiscovery.md) — Finding services

---

*"The internet started as a peer-to-peer network (ARPANET — all nodes equal). Then we centralized everything into client-server (easier to build, monetize, control). Now we're partially decentralizing again (blockchain, WebRTC, edge computing). The pendulum swings." — Internet History* 🏛️
