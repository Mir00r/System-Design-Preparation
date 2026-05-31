# 🌐 REST vs RPC: Choosing Your API Communication Style

> *"Google uses gRPC internally for ALL inter-service communication (10 BILLION calls per second). Meanwhile, every public-facing Google API is REST. Why? Because REST is for humans (developers consuming APIs), and RPC is for machines (services talking to services). Know your audience."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [RESTful APIs](../APIs/RESTful.md), [gRPC](../APIs/gRPC.md), [HTTP](../APIs/HTTP.md)

---

## 📋 Table of Contents
1. [REST vs RPC: The Core Difference](#-rest-vs-rpc-the-core-difference)
2. [REST Deep Dive](#-rest-deep-dive)
3. [RPC Deep Dive](#-rpc-deep-dive)
4. [gRPC: Modern RPC](#-grpc-modern-rpc)
5. [Performance Comparison](#-performance-comparison)
6. [When to Use Each](#-when-to-use-each)
7. [Java Examples](#-java-examples)
8. [Common Interview Trap](#-common-interview-trap)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 REST vs RPC: The Core Difference

```
╔══════════════════════════════════════════════════════════════════╗
║  REST = Resource-oriented: "What THING do you want to act on?" ║
║  RPC  = Action-oriented: "What FUNCTION do you want to call?"  ║
║                                                                ║
║  REST: GET /users/123         (noun-centric)                   ║
║  RPC:  getUser(123)           (verb-centric)                   ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Analogy

```
REST (like a library):
  📚 "I want the BOOK with ID 978-0134685991" (resource!)
  Operations: Get it, Update it, Delete it, List all books
  The BOOK is the center of the universe.
  
RPC (like a remote control):
  🎮 "Execute: turnOnTV()" | "Execute: setVolume(50)"
  Operations: Whatever functions are defined
  The ACTION is the center of the universe.
```

---

## 🌐 REST Deep Dive

```
REST (REpresentational State Transfer):
  Everything is a RESOURCE with a URL.
  Operations = HTTP methods (GET, POST, PUT, DELETE)
  
  ┌────────────────────────────────────────────────────┐
  │  GET    /users         → List users                │
  │  GET    /users/123     → Get user 123              │
  │  POST   /users         → Create user               │
  │  PUT    /users/123     → Update user 123           │
  │  DELETE /users/123     → Delete user 123           │
  │  GET    /users/123/orders → User 123's orders      │
  └────────────────────────────────────────────────────┘
  
  Properties:
  • Stateless (each request is self-contained)
  • Cacheable (GET responses can be cached)
  • Uniform interface (same HTTP verbs everywhere)
  • Resource-based (URLs identify things, not actions)

AWKWARD IN REST (doesn't fit resource model):
  "Send a notification"     → POST /notifications (ok-ish)
  "Transfer money"          → POST /transfers (noun-ified verb)
  "Calculate shipping cost" → GET /shipping-quotes?... (weird)
  "Restart server"          → POST /servers/123/restart (RPC-like!)
```

---

## 📞 RPC Deep Dive

```
RPC (Remote Procedure Call):
  Call a function on a remote server AS IF it were local.
  
  ┌────────────────────────────────────────────────────┐
  │  userService.getUser(123)                          │
  │  userService.createUser(name, email)               │
  │  paymentService.chargeCard(userId, amount)         │
  │  notificationService.sendEmail(to, subject, body)  │
  │  serverService.restart(serverId)                   │
  └────────────────────────────────────────────────────┘
  
  Properties:
  • Action-oriented (verb-centric)
  • Strongly typed (defined interfaces/contracts)
  • Can be sync or async
  • Feels like calling a local function
  
  Implementations:
  • gRPC (Google) — Protocol Buffers, HTTP/2
  • Apache Thrift (Facebook) — Multi-language
  • JSON-RPC — Simple, JSON over HTTP
  • XML-RPC / SOAP — Legacy
```

### RPC Under the Hood

```
  Client code:           Network:                Server code:
  
  User u = userService   [serialize]              
    .getUser(123) ───────► {id: 123} ────────────► getUser(123)
                                                        │
  u = User{...} ◄──────── {name:"Alice"} ◄──────── return user
                          [deserialize]
                          
  The NETWORK is hidden! Client doesn't know/care about:
  • Protocol (HTTP? TCP? UDP?)
  • Serialization (JSON? Protobuf? MessagePack?)
  • Server location (localhost? another continent?)
```

---

## 🚀 gRPC: Modern RPC

```
gRPC = Google's Remote Procedure Call framework
  • Uses Protocol Buffers (protobuf) for serialization
  • Runs over HTTP/2 (multiplexing, streaming)
  • Code generation (client + server stubs from .proto file)
  • Supports 4 communication patterns

PROTOCOL BUFFER (.proto file):
  service UserService {
    rpc GetUser (UserRequest) returns (User);
    rpc ListUsers (ListRequest) returns (stream User);
    rpc CreateUser (User) returns (User);
  }
  
  message User {
    int32 id = 1;
    string name = 2;
    string email = 3;
  }

4 PATTERNS:
  1. Unary: Request → Response (like REST)
  2. Server streaming: Request → stream of responses
  3. Client streaming: stream of requests → Response
  4. Bidirectional streaming: stream ↔ stream

SIZE COMPARISON (same data):
  JSON: {"id":123,"name":"Alice","email":"a@b.com"} = 47 bytes
  Protobuf: [binary] = 19 bytes (60% smaller!)
```

---

## 📊 Performance Comparison

```
┌────────────────────┬─────────────────┬─────────────────────────┐
│  Metric            │  REST/JSON      │  gRPC/Protobuf          │
├────────────────────┼─────────────────┼─────────────────────────┤
│  Payload size      │  Large (text)   │  ~60% smaller (binary)  │
│  Serialization     │  Slow (JSON)    │  Fast (protobuf)        │
│  Latency           │  Higher         │  ~2-10x faster          │
│  HTTP version      │  HTTP/1.1       │  HTTP/2 (multiplexed)   │
│  Streaming         │  Workarounds    │  Native bidirectional   │
│  Type safety       │  None/Optional  │  Strong (generated)     │
│  Browser support   │  ✅ Native      │  ❌ Needs grpc-web      │
│  Debugging         │  Easy (curl)    │  Harder (binary)        │
│  Caching           │  ✅ HTTP cache  │  ❌ Not built-in        │
│  Discovery         │  ✅ Swagger/OAS │  ✅ Protobuf reflection │
│  Learning curve    │  Low            │  Medium                 │
└────────────────────┴─────────────────┴─────────────────────────┘
```

---

## 🎯 When to Use Each

```
USE REST WHEN:
  ✅ Public-facing APIs (third-party developers)
  ✅ Web/browser clients (native HTTP)
  ✅ CRUD operations (natural fit for resources)
  ✅ Need caching (HTTP caching is well-understood)
  ✅ Simple integrations (curl-friendly)
  ✅ Loose coupling between teams/organizations

USE RPC (gRPC) WHEN:
  ✅ Internal microservice communication
  ✅ Performance-critical paths (low latency)
  ✅ Streaming data (real-time feeds)
  ✅ Polyglot services (code generation for any language)
  ✅ Strong type safety needed (contract-first)
  ✅ High throughput (thousands of calls/second between services)

THE PATTERN AT BIG COMPANIES:
  External API: REST (developer-friendly, cacheable)
  Internal communication: gRPC (fast, typed, efficient)
  
  ┌──────────┐  REST  ┌───────────┐  gRPC  ┌───────────┐
  │  Mobile  │───────►│  API      │────────►│  Internal │
  │  App     │        │  Gateway  │         │  Services │
  └──────────┘        └───────────┘         └───────────┘
```

---

## 💻 Java Examples

### REST: Spring Boot Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<UserDto> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(UserDto.from(user));
    }
    
    @PostMapping
    public ResponseEntity<UserDto> createUser(@RequestBody CreateUserRequest req) {
        User user = userService.create(req);
        return ResponseEntity.created(URI.create("/api/users/" + user.getId()))
            .body(UserDto.from(user));
    }
}
```

### gRPC: Service Definition + Implementation

```protobuf
// user_service.proto
syntax = "proto3";
package com.example.user;

service UserService {
    rpc GetUser (GetUserRequest) returns (UserResponse);
    rpc CreateUser (CreateUserRequest) returns (UserResponse);
    rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
    int64 id = 1;
}

message UserResponse {
    int64 id = 1;
    string name = 2;
    string email = 3;
}
```

```java
// gRPC Server Implementation
@GrpcService
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {
    
    @Autowired private UserRepository userRepository;
    
    @Override
    public void getUser(GetUserRequest request, 
                        StreamObserver<UserResponse> responseObserver) {
        User user = userRepository.findById(request.getId())
            .orElseThrow(() -> Status.NOT_FOUND
                .withDescription("User not found")
                .asRuntimeException());
        
        responseObserver.onNext(toProto(user));
        responseObserver.onCompleted();
    }
    
    @Override
    public void listUsers(ListUsersRequest request,
                          StreamObserver<UserResponse> responseObserver) {
        // Server streaming: send users one by one
        userRepository.findAll().forEach(user -> {
            responseObserver.onNext(toProto(user));
        });
        responseObserver.onCompleted();
    }
}

// gRPC Client (calling the service)
@Service
public class UserGrpcClient {
    
    private final UserServiceGrpc.UserServiceBlockingStub stub;
    
    public UserGrpcClient(@GrpcChannel("user-service") Channel channel) {
        this.stub = UserServiceGrpc.newBlockingStub(channel);
    }
    
    public UserResponse getUser(long id) {
        // Feels like a local function call!
        return stub.getUser(GetUserRequest.newBuilder().setId(id).build());
    }
}
```

---

## ⚠️ Common Interview Trap

```
TRAP: "REST and RPC are completely different things"

REALITY: REST is often implemented OVER HTTP (which is a request-response
protocol). gRPC also uses HTTP (HTTP/2). The difference is PHILOSOPHY:

  REST thinks in RESOURCES:   "The user with ID 123"
  RPC thinks in FUNCTIONS:    "Get me user 123"
  
  REST URL: GET /users/123
  RPC call: userService.getUser(123)
  
  Both accomplish the same thing! The difference is:
  - REST constrains you (uniform interface, stateless, cacheable)
  - RPC gives you freedom (any function signature, any pattern)
  
  REST's constraints are FEATURES for public APIs.
  RPC's freedom is EFFICIENT for internal communication.
```

---

## 🎮 Mini Challenge

### 🧩 Design: API Strategy for an E-commerce Platform

You have 20 internal microservices and public APIs for mobile apps + third-party sellers. Decide REST vs RPC for each:
1. Mobile app ↔ API Gateway
2. API Gateway ↔ Product Service
3. Product Service ↔ Inventory Service (real-time stock check)
4. Order Service → Notification Service (send confirmation email)
5. Third-party seller API (upload products)

<details>
<summary>🔑 Answer</summary>

1. **Mobile ↔ API Gateway**: REST (browser-compatible, cacheable, developer-friendly)
2. **Gateway ↔ Product Service**: gRPC (internal, low latency, type-safe)
3. **Product ↔ Inventory (real-time)**: gRPC (performance-critical, high-frequency)
4. **Order → Notification**: Neither! Use async message queue (Kafka). Not every communication needs request-response!
5. **Third-party seller API**: REST (external developers, documentation with Swagger, standard HTTP)

Key insight: Communication style should match the relationship (public vs internal) and requirements (sync vs async, latency, coupling).
</details>

---

## ❓ Interview Q&A

**Q1: What's the fundamental difference between REST and RPC?**
> REST is resource-oriented — you model your API around nouns (users, orders, products) and use standard HTTP verbs. RPC is action-oriented — you model your API around functions/procedures (getUser, createOrder, calculateShipping). REST constrains you with a uniform interface; RPC gives you freedom to define any operation.

**Q2: Why do companies like Google and Netflix use gRPC internally but REST externally?**
> gRPC is faster (protobuf binary serialization, HTTP/2 multiplexing) and type-safe (generated code from .proto), ideal for internal service-to-service communication. REST is more accessible for external developers (curl-friendly, JSON readable, browser native, cacheable via HTTP headers). Different audiences have different needs.

**Q3: What are the advantages of gRPC over REST?**
> (1) 2-10x faster due to protobuf binary serialization (smaller payloads), (2) HTTP/2 multiplexing (many calls on one connection), (3) Native bidirectional streaming, (4) Strong typing with code generation, (5) Built-in deadline propagation. Disadvantages: harder to debug (binary), no browser support without grpc-web, no HTTP caching.

**Q4: When does REST struggle as an API style?**
> For operations that don't map to CRUD on resources: "restart a server," "calculate shipping cost," "send a notification." You end up awkwardly noun-ifying verbs (POST /restarts, POST /calculations). Also struggles with: real-time streaming, high-performance internal calls, and strongly-typed contracts.

**Q5: Can you use both REST and RPC in the same system?**
> Absolutely — this is the recommended pattern at scale. API Gateway exposes REST to clients (public-facing, cacheable, well-documented). Behind the gateway, services communicate via gRPC (fast, typed, efficient). The gateway translates between REST and gRPC. This gives you the best of both worlds.

---

## 🔗 Related Topics
- [RESTful APIs](../APIs/RESTful.md) — REST principles in depth
- [gRPC](../APIs/gRPC.md) — gRPC deep dive
- [API Gateway](../BuildingBlocks/APIGateway.md) — REST→gRPC translation point
- [GraphQL](../APIs/GraphQL.md) — Another alternative (query language)

---

*"REST is an architectural style. RPC is a paradigm. GraphQL is a query language. They're not competitors — they're tools for different problems." — API Design Wisdom* 🌐

---

*Previous: [← Concurrency vs Parallelism](./Concurrency_vs_Parallelism.md) | Next: [Batch vs Stream Processing →](./Batch_vs_Stream_Processing.md)*
