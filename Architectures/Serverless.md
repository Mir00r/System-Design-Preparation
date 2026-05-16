# ⚡ Serverless Architecture: Functions as a Service (FaaS)

> *"Serverless doesn't mean no servers — it means you don't THINK about servers. You write a function, deploy it, and the cloud handles scaling from zero to millions of invocations automatically."*

**⏱️ Estimated Time**: 22 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Cloud Services](../Cloud/README.md), [Microservices](../Microservices/)

---

## 🤔 What Is Serverless?

```
TRADITIONAL SERVER:
  You provision EC2 → install OS → deploy app → configure scaling
  You pay 24/7 even at 3 AM with 0 users
  You manage patches, monitoring, capacity planning
  
SERVERLESS:
  You write a function → deploy → done
  Cloud runs it on demand, scales automatically
  You pay only for actual execution time (millisecond billing)
  
  ┌──────────────────────────────────────────────────────────────┐
  │ SERVERLESS SPECTRUM                                          │
  │                                                              │
  │  FaaS           BaaS              Managed Services           │
  │  (Functions)    (Backend-as-a-)   (Fully managed)            │
  │                                                              │
  │  AWS Lambda     Firebase          DynamoDB                   │
  │  Azure Funcs    Auth0             S3                         │
  │  GCP Cloud Fn   Stripe            SQS                       │
  │                 Twilio            API Gateway                │
  └──────────────────────────────────────────────────────────────┘
  
  KEY PROPERTIES:
  ✅ No server management
  ✅ Auto-scaling (0 → ∞)
  ✅ Pay-per-use (no idle cost)
  ✅ Event-driven
  ❌ Cold starts (latency on first invocation)
  ❌ Execution time limits (15 min max on Lambda)
  ❌ Stateless (no local state between invocations)
```

---

## 🏗️ How It Works

```
EVENT-DRIVEN EXECUTION:

  ┌───────────┐    trigger    ┌─────────────────┐    invoke    ┌──────────┐
  │ Event     │ ────────────→ │ Cloud Platform   │ ───────────→│ Function │
  │ Source    │               │ (Lambda runtime) │             │ (your    │
  └───────────┘               └─────────────────┘             │  code)   │
                                                              └──────────┘
  Event Sources:
  • HTTP request (API Gateway)
  • Message queue (SQS, Kafka)
  • File upload (S3 event)
  • Database change (DynamoDB Stream)
  • Scheduled (CloudWatch cron)
  • IoT device signal

COLD START vs WARM START:
  
  COLD START (first invocation or after idle):
  ┌──────────┬────────────┬──────────┬──────────────┐
  │ Download │ Start      │ Init     │ Execute      │
  │ code     │ container  │ runtime  │ function     │
  │ (100ms)  │ (200ms)    │ (varies) │ (your code)  │
  └──────────┴────────────┴──────────┴──────────────┘
  Total: 200ms-2s extra latency
  
  WARM START (container already running):
  ┌──────────────┐
  │ Execute      │  ← Just runs your code, ~1ms overhead
  │ function     │
  └──────────────┘
```

---

## 💻 AWS Lambda Example (Java)

```java
// Lambda Function Handler
public class OrderProcessor implements RequestHandler<SQSEvent, String> {
    
    private final OrderService orderService;
    
    // Constructor runs ONCE per cold start (reuse connections!)
    public OrderProcessor() {
        this.orderService = new OrderService(
            DynamoDbClient.create(),
            SnsClient.create()
        );
    }
    
    @Override
    public String handleRequest(SQSEvent event, Context context) {
        for (SQSEvent.SQSMessage message : event.getRecords()) {
            OrderRequest order = parseOrder(message.getBody());
            
            // Process order
            orderService.processOrder(order);
            
            context.getLogger().log("Processed order: " + order.getId());
        }
        return "Processed " + event.getRecords().size() + " orders";
    }
}

// Spring Cloud Function (framework-agnostic serverless)
@SpringBootApplication
public class OrderFunctionApp {
    
    @Bean
    public Function<OrderRequest, OrderResponse> processOrder(OrderService service) {
        return request -> {
            Order order = service.create(request);
            return new OrderResponse(order.getId(), "CREATED");
        };
    }
}
```

---

## 📊 Serverless vs Containers vs VMs

| Dimension | Serverless (Lambda) | Containers (ECS/K8s) | VMs (EC2) |
|---|---|---|---|
| Scaling | Automatic (0→∞) | Auto (with config) | Manual/Auto-scaling |
| Cold start | 100ms-2s | Seconds | Minutes |
| Max execution | 15 min | Unlimited | Unlimited |
| State | Stateless | Stateful possible | Fully stateful |
| Cost at scale | Can be expensive | Moderate | Cheapest at high util |
| Cost at low traffic | Near zero | Min containers running | Always paying |
| Ops overhead | None | Medium | High |
| Vendor lock-in | High | Medium | Low |

---

## 🎯 When to Use Serverless

```
✅ GREAT FIT:
  • Event processing (file upload → thumbnail → store)
  • API backends with variable traffic (startup MVPs)
  • Scheduled jobs (daily reports, cleanup tasks)
  • Webhooks and integrations
  • Chat bots, Alexa skills
  • Data transformation pipelines (ETL)

❌ POOR FIT:
  • Long-running processes (>15 min)
  • Latency-sensitive (real-time gaming, HFT)
  • High-throughput steady-state (cheaper on containers)
  • Stateful applications (WebSocket servers)
  • Complex orchestration (better with containers + service mesh)
  • Large binaries / ML inference (slow cold starts)
```

---

## ⚠️ Common Pitfalls

1. **Cold start ignorance** — Java/Spring Boot Lambda cold starts can be 3-10 seconds. Mitigations: use GraalVM native image, Provisioned Concurrency, keep functions warm with scheduled pings, or use lighter runtimes (Node.js/Python for latency-critical paths).

2. **Distributed monolith** — 200 Lambda functions tightly coupled via synchronous calls = worse than a monolith. Use async events (SQS, SNS, EventBridge) between functions. Each function should be independently deployable.

3. **Not considering cost at scale** — Lambda is cheap at low traffic but at 100M+ invocations/month, containers are often 3-5x cheaper. Model your cost at expected scale before committing.

4. **Testing difficulty** — Hard to test locally (different environment). Use LocalStack, SAM Local, or Testcontainers. Write business logic in plain classes, test without Lambda-specific code.

---

## 📝 Interview Q&A

**Q: How would you design an image processing pipeline using serverless?**
> A: (1) User uploads image to **S3**. (2) S3 event triggers **Lambda A** (validate format, check size). (3) Lambda A puts message on **SQS** with processing instructions. (4) **Lambda B** (triggered by SQS) generates thumbnails (small, medium, large). (5) Thumbnails stored back in S3. (6) Lambda B publishes **SNS notification** "processing complete." (7) **Lambda C** (triggered by SNS) updates DynamoDB with image metadata + thumbnail URLs. (8) User polls or receives WebSocket push that processing is done. Benefits: scales to 0 (no cost when idle), handles bursts (1000 uploads/sec), each step retries independently.

**Q: How do you handle the cold start problem?**
> A: (1) **Provisioned Concurrency**: pre-warm N instances (costs more but guaranteed latency). (2) **Smaller packages**: reduce deployment artifact size (fewer dependencies = faster init). (3) **Runtime choice**: Python/Node.js cold starts are 100-200ms vs Java 1-3s. (4) **GraalVM native image**: compiles Java ahead-of-time → 100ms cold starts. (5) **Architecture**: put latency-sensitive paths on always-running containers, use Lambda for async/background work where cold starts don't matter.

---

## 🔗 What to Read Next

1. **[Cloud/AWS/Compute_EC2_Lambda.md](../Cloud/AWS/Compute_EC2_Lambda.md)** — Lambda deep dive
2. **[Architectures/Event_Driven.md](./Event_Driven.md)** — Event-driven patterns
3. **[Architectures/Microkernel.md](./Microkernel.md)** — Plugin architecture

---

*[← SOA](./SOA.md) | [Back to Index](../INDEX.md)*
