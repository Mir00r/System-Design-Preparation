# 🧱 Foundations: Computer Science Fundamentals for System Design

> *"System design interviews test whether you can apply CS fundamentals at scale. You don't need to implement TCP from scratch, but you must understand WHY things work the way they do — because that understanding drives every architectural decision."*

**⏱️ Estimated Time**: 5 minutes | **🎯 Difficulty**: 🟢 Easy

---

## 📋 What's Inside

```
FOUNDATIONS/
├── Networking/          — TCP, UDP, HTTP, DNS, TLS
│   └── README.md        How data travels across the internet
├── OperatingSystems/    — Processes, threads, memory, I/O
│   └── README.md        How computers manage resources
└── HowInternetWorks/   — DNS resolution, CDN, packet routing
    └── README.md        End-to-end request lifecycle
```

---

## 🎯 Why Foundations Matter for System Design

| Foundation | System Design Application |
|---|---|
| TCP vs UDP | Choosing protocol for real-time vs reliable delivery |
| HTTP/1 vs HTTP/2 vs HTTP/3 | API performance, multiplexing, head-of-line blocking |
| DNS | Load balancing, failover, CDN routing |
| TLS handshake | Latency budget (1-2 RTT for HTTPS), mTLS between services |
| Processes vs Threads | Concurrency model choice (Node.js vs Java) |
| Virtual memory | Understanding OOM kills, container memory limits |
| I/O models | Blocking vs non-blocking, epoll, event loops |
| Context switching | Why too many threads hurts performance |

---

## 🔗 Start Here

1. **[Networking/README.md](./Networking/README.md)** — TCP/UDP, HTTP, DNS, TLS
2. **[OperatingSystems/README.md](./OperatingSystems/README.md)** — Processes, threads, memory
3. **[HowInternetWorks/README.md](./HowInternetWorks/README.md)** — Request lifecycle

---

*[Back to Index](../INDEX.md)*
