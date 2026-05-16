# 🔍 Service Discovery: How Microservices Find Each Other

> **"In a world where services are constantly born, scaled, and restarted, hardcoded addresses are the enemy. Service Discovery is how services keep in touch."**

---

## 🎯 What You'll Learn
- Why service discovery is essential in dynamic microservices environments
- Client-side vs Server-side discovery patterns
- DNS-based vs registry-based discovery
- Consul, Eureka, and Kubernetes DNS compared
- How to implement service discovery with Spring Boot + Eureka

**⏱️ Estimated Time**: 18 minutes | **🎯 Difficulty**: 🟡 Medium  
**🔗 Prerequisites**: [Load Balancing](./LoadBalancing.md) | [API Gateway](./APIGateway.md)  
**🔗 Related Topics**: [Circuit Breaker](./CircuitBreaker.md) | [Microservices/DistributedSystem](../Microservices/DistributedSystem.md)

---

## 📋 Table of Contents
1. [The Problem: The Address Book Problem](#-the-problem-the-address-book-problem)
2. [What Is Service Discovery?](#-what-is-service-discovery)
3. [Client-Side vs Server-Side Discovery](#-client-side-vs-server-side-discovery)
4. [Service Registry: The Phone Book](#-service-registry-the-phone-book)
5. [Self-Registration vs Third-Party Registration](#-self-registration-vs-third-party-registration)
6. [Tools: Consul, Eureka, Kubernetes DNS](#-tools-consul-eureka-kubernetes-dns)
7. [Code Examples: Spring Boot + Eureka](#-code-examples-spring-boot--eureka)
8. [Health Checks & Deregistration](#-health-checks--deregistration)
9. [Industry Examples](#-industry-examples)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem: The Address Book Problem

In the old days:

```
STATIC DEPLOYMENT (easy):
  UserService is ALWAYS at: 10.0.0.1:8001
  OrderService is ALWAYS at: 10.0.0.2:8002
  
  Hardcode these in config. Done.
```

In the modern cloud era:

```
DYNAMIC CLOUD (nightmare):
  UserService starts with:   3 instances on 10.0.1.5, 10.0.2.7, 10.0.3.2
  Deployment happens:        3 new instances at 10.0.4.1, 10.0.5.3, 10.0.6.8
  Scale event:               10 instances now!
  Server failure:            5 instances gone, 5 new ones at different IPs
  
  Old hardcoded addresses: useless.
  
  How does OrderService find UserService RIGHT NOW?
```

This is the **Address Book Problem** — in dynamic environments, service locations change constantly.

> 💡 **Key Insight**: Container orchestration (Kubernetes, Docker Swarm) starts, stops, and migrates services constantly. IP addresses are **ephemeral**. Service Discovery provides a stable, always-accurate directory.

---

## 💡 What Is Service Discovery?

**Service Discovery** is the mechanism by which services in a distributed system locate each other's network addresses (IP + port) dynamically at runtime.

```
SERVICE DISCOVERY FLOW:

1. Service REGISTERS itself:
   "I'm OrderService, I'm alive at 10.0.1.5:8002"
   
2. Service DISCOVERS others:
   "Where is UserService right now?"
   Registry: "10.0.1.7:8001, 10.0.2.3:8001, 10.0.3.5:8001"
   
3. Service CALLS discovered instance:
   HTTP GET http://10.0.1.7:8001/users/123
   
4. When service SHUTS DOWN:
   Registry notified → address removed from pool
```

---

## 🌍 Real-World Analogy

```
RESTAURANT RESERVATION ANALOGY:

Without Service Discovery:
  You know a restaurant is at "123 Main Street."
  You show up. It moved. You have no idea where.
  
With Service Discovery (like Yelp/Google Maps):
  You search "Italian restaurants near me" (service name)
  Yelp/Maps returns current addresses + hours (registry)
  You go to the current, active location (call the service)
  If restaurant closes, it disappears from results (deregistration)
  
The directory is always up-to-date. You never hardcode addresses.
```

---

## 🏗️ Client-Side vs Server-Side Discovery

Two fundamental patterns. Know both — interviews love this distinction.

### Client-Side Discovery

The **client** queries the service registry, gets a list of instances, and **picks one** using load balancing logic.

```
CLIENT-SIDE DISCOVERY:

1. OrderService asks registry: "Where is UserService?"
2. Registry returns: [10.0.1.5:8001, 10.0.1.7:8001, 10.0.2.3:8001]
3. OrderService picks one (Round Robin, Random, etc.)
4. OrderService calls: http://10.0.1.7:8001/users/123

LOAD BALANCING: Client does it (no LB needed in the network path)
```

```
ARCHITECTURE:
                              ┌─────────────┐
OrderService ──── queries ───►│   Registry  │
                              │  (Eureka /  │
OrderService ◄── addresses ───│   Consul)   │
     │                        └─────────────┘
     │ (picks one, calls directly)
     ▼
UserService instance 2
```

**✅ Pros**: No extra network hop, client controls load balancing algorithm  
**❌ Cons**: Service discovery logic in every client (every language needs an SDK)  
**Used by**: Netflix Eureka (with Ribbon), Spring Cloud LoadBalancer

---

### Server-Side Discovery

The **load balancer** queries the registry and routes for the client.

```
SERVER-SIDE DISCOVERY:

1. OrderService sends request to LOAD BALANCER (not directly to UserService)
2. Load Balancer queries registry: "Where is UserService?"
3. Registry returns instances
4. Load Balancer picks one and forwards request
5. OrderService never knows actual UserService IPs

LOAD BALANCING: Done by the load balancer/API gateway
```

```
ARCHITECTURE:

OrderService ──► API Gateway / LB ──── queries ───► Registry
                       │               ◄─ addresses─┘
                       │ (routes to correct instance)
                       ▼
               UserService instance 1
```

**✅ Pros**: Client is simple (no discovery SDK), works with any language  
**❌ Cons**: Extra network hop through LB, LB is another component to manage  
**Used by**: Kubernetes, AWS ELB + ECS, Consul + HAProxy

---

## 📒 Service Registry: The Phone Book

The **Service Registry** is a database of service instance locations. It's the source of truth.

```
REGISTRY DATA STRUCTURE:

ServiceName: "UserService"
Instances:
  ├── id: "user-1"
  │   host: "10.0.1.5"
  │   port: 8001
  │   health: HEALTHY
  │   metadata: {version: "2.1", zone: "us-east-1a"}
  │   last_heartbeat: 2026-01-17T14:30:00Z
  │
  ├── id: "user-2"
  │   host: "10.0.2.7"
  │   port: 8001
  │   health: HEALTHY
  │   last_heartbeat: 2026-01-17T14:30:01Z
  │
  └── id: "user-3"
      host: "10.0.3.2"
      port: 8001
      health: UNHEALTHY ← removed from routing pool
      last_heartbeat: 2026-01-17T14:25:00Z (stale!)
```

---

## 🤝 Self-Registration vs Third-Party Registration

### Self-Registration
The service itself registers and deregisters.

```java
// Service registers on startup
@SpringBootApplication
@EnableEurekaClient  // Registers on startup, deregisters on shutdown
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

**✅ Pros**: Simple, built into service lifecycle  
**❌ Cons**: Services must have discovery logic, bad behavior if service crashes before deregistering

### Third-Party Registration
An external process (sidecar, orchestrator) manages registration.

```
Kubernetes:
  kubelet starts Pod → notifies kube-dns
  Pod crashes → kubelet notifies kube-dns → removed from DNS
  
  Service is unaware of discovery — it just runs.
  
Consul with Registrator sidecar:
  Registrator watches Docker events
  Container starts → Registrator registers with Consul
  Container stops → Registrator deregisters
```

**✅ Pros**: Services don't need discovery SDK, works with any tech stack  
**❌ Cons**: More infrastructure to manage, registration not under service's control

---

## 🛠️ Tools: Consul, Eureka, Kubernetes DNS

### Netflix Eureka
```
Type: Client-side service registry (Java-centric)
Pattern: Self-registration

Features:
  - REST-based registry API
  - Built-in client-side load balancing (Ribbon integration)
  - Zone-aware routing
  - 30-second heartbeat interval
  - 90-second eviction (3 missed heartbeats)

Used By: Netflix (original creator), many Spring Boot shops
Status: Still widely used but less active development
```

### HashiCorp Consul
```
Type: Service registry + health checking + KV store + service mesh
Pattern: Self-registration or Third-party (via Registrator)

Features:
  - Multi-datacenter support
  - Built-in health checks (HTTP, TCP, script, TTL)
  - Service mesh (Consul Connect) with mTLS
  - DNS and HTTP API interfaces
  - Key-value store for config
  - Watch mechanisms for real-time updates
  - Works with ANY language

DNS Usage:
  userservice.service.consul → returns A records of healthy instances
```

### Kubernetes DNS (CoreDNS)
```
Type: DNS-based service discovery (built into k8s)
Pattern: Third-party (kubelet + CoreDNS)

How it works:
  - Create a Service object in k8s
  - Kubernetes assigns a stable DNS name: userservice.default.svc.cluster.local
  - All Pods can resolve this DNS name
  - DNS returns ClusterIP (virtual IP)
  - kube-proxy forwards to actual Pod IPs

DNS Format: <service-name>.<namespace>.svc.cluster.local

Example:
  GET http://userservice/api/users/123
  → DNS resolves "userservice" → ClusterIP 10.100.0.1
  → kube-proxy → actual Pod at 10.0.1.5:8080
```

### Comparison

| Feature | Eureka | Consul | Kubernetes DNS |
|---|---|---|---|
| Language support | Java-first | Any language | Any language |
| Health checks | Heartbeat only | Rich (HTTP/TCP/script) | Kubelet liveness probes |
| DNS interface | No (REST only) | Yes | Yes (primary) |
| Config store | No | Yes (KV) | ConfigMap (separate) |
| Service mesh | No | Yes (Consul Connect) | Requires Istio |
| Multi-datacenter | Limited | Yes | Multi-cluster (complex) |
| Best for | Spring Boot, Java shops | Multi-language, Consul mesh | Kubernetes workloads |

---

## 💻 Code Examples: Spring Boot + Eureka

### Eureka Server (Registry)

```java
// pom.xml dependency:
// spring-cloud-starter-netflix-eureka-server

@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication {
    public static void main(String[] args) {
        SpringApplication.run(ServiceRegistryApplication.class, args);
    }
}
```

```yaml
# application.yml
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false  # Registry doesn't register itself
    fetch-registry: false
  server:
    enable-self-preservation: false  # Dev mode
```

### Service Registration (Client)

```java
// pom.xml: spring-cloud-starter-netflix-eureka-client

@SpringBootApplication
@EnableEurekaClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

```yaml
# application.yml
spring:
  application:
    name: user-service  # This is the service name in registry

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10   # Heartbeat every 10s
    lease-expiration-duration-in-seconds: 30 # Evict after 30s no heartbeat
```

### Service Discovery (Calling Another Service)

```java
@Configuration
public class WebClientConfig {
    
    @Bean
    @LoadBalanced  // Spring will intercept and resolve service names
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class OrderService {
    
    private final WebClient webClient;
    
    public OrderService(WebClient.Builder webClientBuilder) {
        this.webClient = webClientBuilder
            .baseUrl("http://user-service")  // Service name, not IP!
            .build();
    }
    
    public Mono<User> getUserForOrder(String userId) {
        return webClient.get()
            .uri("/api/users/{id}", userId)
            .retrieve()
            .bodyToMono(User.class);
        // Spring resolves "user-service" → actual IP via Eureka
        // Applies Round Robin load balancing across instances
    }
}
```

### Manual Discovery with DiscoveryClient

```java
@Service
public class ManualDiscoveryService {
    
    @Autowired
    private DiscoveryClient discoveryClient;
    
    public String getServiceUrl(String serviceName) {
        List<ServiceInstance> instances = 
            discoveryClient.getInstances(serviceName);
        
        if (instances.isEmpty()) {
            throw new ServiceNotFoundException(serviceName);
        }
        
        // Simple round-robin (Spring LoadBalancer handles this automatically)
        ServiceInstance instance = instances.get(
            (int)(Math.random() * instances.size())
        );
        
        return instance.getUri().toString();
    }
}
```

---

## 🩺 Health Checks & Deregistration

A stale registry entry (dead service still listed as alive) is dangerous.

```
HEALTH CHECK STRATEGIES:

1. HEARTBEAT (Eureka):
   Service sends: POST /eureka/apps/{appName}/{instanceId}
   Every 30 seconds
   No heartbeat for 90 seconds → Evicted from registry

2. ACTIVE HEALTH CHECK (Consul):
   Registry pings: GET http://service:8080/health
   Every 10 seconds
   2 failures → Mark CRITICAL
   Mark critical → Remove from routing

3. KUBERNETES READINESS PROBE:
   kubelet pings: GET /actuator/health
   Every 10 seconds
   Fails → Pod removed from Service endpoints (not killed, just excluded)
   
   LIVENESS PROBE:
   kubelet pings: GET /health/liveness
   Fails → Pod killed and restarted
```

```java
// Spring Boot Actuator health endpoint (auto-exposed)
// GET /actuator/health → {"status": "UP"}
// GET /actuator/health/readiness → Kubernetes readiness check

// Custom health indicator
@Component
public class DatabaseHealthIndicator extends AbstractHealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try (Connection conn = dataSource.getConnection()) {
            conn.createStatement().executeQuery("SELECT 1");
            builder.up().withDetail("database", "responding");
        } catch (Exception e) {
            builder.down().withException(e);
        }
    }
}
```

---

## 🏢 Industry Examples

```
NETFLIX:
- Created Eureka (open-sourced 2012)
- Used for all internal service discovery
- 100+ microservices registered
- Client-side discovery with Ribbon load balancer
- Eventually moved more toward Kubernetes

UBER:
- Uses Consul for service discovery
- Multi-datacenter aware
- Services register with local Consul agent
- Consul DNS interface used for discovery

AMAZON:
- AWS ECS + Cloud Map for managed discovery
- Route 53 Service Discovery (DNS-based)
- 1000s of internal services discovered dynamically
- "Service Discovery" is a first-class API in AWS

AIRBNB:
- Moved from Eureka to Kubernetes service discovery (CoreDNS)
- Migration: 1000+ services gradually moved to k8s
```

---

## ⚠️ Common Pitfalls

### 1. Stale Registry Entries
```
❌ Problem: Service crashes without deregistering
            Registry still shows it as healthy
            Other services route to dead instance → errors

✅ Fix: Short heartbeat intervals (10-30s)
         Short eviction timeouts (30-90s)
         Active health checks (not just heartbeats)
```

### 2. Registry as Single Point of Failure
```
❌ Problem: Eureka/Consul crashes → all service discovery fails
✅ Fix: High-availability registry cluster (Eureka: multiple peers,
         Consul: Raft cluster with 3-5 nodes)
         Client-side caching of discovered addresses
         Stale cache better than no cache during registry outage
```

### 3. Too Many Registered Services
```
❌ Problem: 1000 services, all registering/heartbeating
            Registry becomes bottleneck
✅ Fix: Consul scales well to 100k+ services
         Use zones/namespaces to partition registry
         Kubernetes CoreDNS handles millions of endpoints
```

### 4. Ignoring the "Deregistration on Crash" Problem
```
❌ Problem: JVM crash, OOM kill → process dies without deregistering
✅ Fix: Short eviction timeout (registry removes dead nodes quickly)
         Use Kubernetes liveness probes (k8s handles deregistration)
         Graceful shutdown hooks + try-finally deregistration
```

---

## 🧩 Mini Challenge

```
🎲 SCENARIO (3 minutes):

You have 50 microservices on Kubernetes.
Your team just heard about Consul and wants to add it.

Current state: Kubernetes with CoreDNS (built-in service discovery)

QUESTION:
Should you add Consul on top of Kubernetes CoreDNS?
When would Consul add value, and when is CoreDNS sufficient?
```

<details>
<summary>💡 Click to reveal answer</summary>

**CoreDNS is sufficient when:**
- All services run in the same Kubernetes cluster
- You don't need multi-datacenter or multi-cluster service discovery
- You don't need a config store (use ConfigMaps instead)
- You don't need service mesh features (use Istio for that)
- Simple DNS-based discovery is acceptable

**Consul adds value when:**
- **Multi-datacenter**: Services across multiple k8s clusters or bare-metal need to discover each other
- **Mixed infrastructure**: Some services run in k8s, some on VMs — need unified discovery
- **Rich health checks**: Need script-based health checks beyond HTTP liveness probes
- **Config store**: Need distributed configuration (Consul KV)
- **Service mesh without Istio**: Consul Connect provides mTLS without Istio complexity

**Recommendation**: For a pure Kubernetes setup, CoreDNS is usually sufficient and simpler. Add Consul when you have multi-datacenter needs or mixed infrastructure.
</details>

---

## 📝 Interview Q&A

**Q1: What problem does service discovery solve?**
> In dynamic cloud environments, service IP addresses change constantly due to scaling, deployments, and failures. Hardcoding IPs is brittle. Service discovery provides a dynamic directory where services register themselves and others can look them up by name, always getting current, healthy addresses.

**Q2: What is the difference between client-side and server-side service discovery?**
> Client-side: The calling service queries the registry directly and performs load balancing itself. Example: Netflix Eureka + Ribbon. Server-side: The load balancer or API gateway queries the registry and routes transparently. The client only knows the LB address. Example: Kubernetes Services + CoreDNS.

**Q3: What happens if the service registry goes down?**
> If registry goes down: New registrations fail, new lookups fail. Mitigations: (1) Clients cache discovered addresses — serve from cache during outage. (2) Run registry in HA cluster (Consul: 3-5 Raft nodes, Eureka: peer-to-peer replication). (3) Self-preservation mode: Eureka stops evicting entries during partitions to avoid mass invalidation.

**Q4: How does Kubernetes handle service discovery?**
> You create a Service object pointing to Pods. Kubernetes assigns a stable ClusterIP and DNS name (`service-name.namespace.svc.cluster.local`). CoreDNS resolves this name. kube-proxy maintains iptables/IPVS rules to forward to actual Pod IPs. When Pods come and go, Kubernetes updates the endpoint set automatically.

**Q5: What is the difference between Consul and Eureka?**
> Eureka: Java-centric, simple heartbeat-based, REST API only, no built-in DNS, originally Netflix. Consul: Language-agnostic, rich health checks (HTTP/TCP/script), DNS + HTTP interfaces, built-in KV store, multi-datacenter, service mesh (Consul Connect). Consul is more feature-rich; Eureka simpler for Java Spring Boot shops.

---

## 🔗 What to Read Next

| Topic | Why You Need It |
|---|---|
| [Circuit Breaker](./CircuitBreaker.md) | Once you discover and call a service, you need Circuit Breaker for when it fails |
| [API Gateway](./APIGateway.md) | API Gateways implement server-side discovery — understand how they use registries |
| [Microservices/DistributedSystem](../Microservices/DistributedSystem.md) | Service discovery is one piece of the larger distributed systems puzzle |

---

*[← Back to Building Blocks](../BuildingBlocks/) | [← Index](../INDEX.md)*
