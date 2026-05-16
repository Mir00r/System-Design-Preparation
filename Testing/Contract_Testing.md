# 📜 Contract Testing: Pact & Consumer-Driven Contracts

> *"In microservices, Service A calls Service B's API. If B changes its response format, A breaks in production. Contract tests catch this BEFORE deployment by verifying both sides agree on the API contract."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Integration Testing](./Integration_Testing.md), Microservices basics

---

## 🤔 The Problem

```
WITHOUT CONTRACT TESTS:

  [Order Service]                        [Payment Service]
  Expects: { "status": "SUCCESS" }       Returns: { "result": "SUCCESS" }
                                         (field renamed in v2!)
  
  Tests pass individually ✅             Tests pass individually ✅
  Both deployed to production...
  Order Service calls Payment Service → FIELD NOT FOUND → 💥 500 ERROR
  
  Integration tests didn't catch it because:
  - Each service tests in isolation (mocked dependencies)
  - E2E tests are too slow / not run before every deploy

WITH CONTRACT TESTS:

  Consumer (Order Service) defines: "I expect field 'status' with value 'SUCCESS'"
  Provider (Payment Service) verifies: "Do I return 'status' field? YES ✅ / NO ❌"
  
  If Payment renames to 'result' → contract test FAILS before deploy → fix it!
```

---

## 🔄 Consumer-Driven Contracts (CDC)

```
FLOW:
  1. CONSUMER writes a contract: "When I call POST /payments with {...},
     I expect response {status: 'SUCCESS', transactionId: 'string'}"
  
  2. Contract is shared (via Pact Broker or file)
  
  3. PROVIDER runs the contract against its real implementation:
     "When I receive POST /payments with {...},
     do I actually return {status: 'SUCCESS', transactionId: 'string'}?"
  
  4. If provider's actual response matches contract → ✅ compatible
     If not → ❌ provider knows it would break the consumer

  ┌────────────────┐         ┌─────────────────┐
  │ Consumer       │         │ Pact Broker     │
  │ (Order Service)│──write──▶│ (stores contracts)│
  │                │         │                 │
  └────────────────┘         └───────┬─────────┘
                                     │ verify
                                     ▼
                             ┌─────────────────┐
                             │ Provider        │
                             │ (Payment Service)│
                             └─────────────────┘

WHY CONSUMER-DRIVEN:
  - Consumer knows what it ACTUALLY uses (not the full API)
  - Provider can evolve freely as long as consumers' contracts are met
  - Adding a field? Fine (no consumer depends on it yet)
  - Removing a field? Contract fails for consumer that uses it
```

---

## 💻 Pact Implementation (Java)

```java
// CONSUMER SIDE (Order Service)
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "payment-service", port = "8080")
class PaymentClientContractTest {
    
    @Pact(consumer = "order-service")
    public V4Pact createPact(PactDslWithProvider builder) {
        return builder
            .given("a valid customer exists")
            .uponReceiving("a request to charge payment")
            .path("/api/payments")
            .method("POST")
            .headers("Content-Type", "application/json")
            .body(new PactDslJsonBody()
                .stringValue("customerId", "cust-123")
                .decimalType("amount", 99.99)
                .stringValue("currency", "USD"))
            .willRespondWith()
            .status(200)
            .headers(Map.of("Content-Type", "application/json"))
            .body(new PactDslJsonBody()
                .stringType("transactionId", "txn-abc")
                .stringValue("status", "SUCCESS")
                .decimalType("chargedAmount", 99.99))
            .toPact(V4Pact.class);
    }
    
    @Test
    @PactTestFor(pactMethod = "createPact")
    void shouldChargePayment(MockServer mockServer) {
        PaymentClient client = new PaymentClient(mockServer.getUrl());
        
        PaymentResponse response = client.charge("cust-123", 99.99, "USD");
        
        assertThat(response.getStatus()).isEqualTo("SUCCESS");
        assertThat(response.getTransactionId()).isNotBlank();
    }
}

// PROVIDER SIDE (Payment Service)
@Provider("payment-service")
@PactBroker(url = "https://pact-broker.company.com")
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class PaymentProviderContractTest {
    
    @TestTarget
    public final Target target = new SpringBootHttpTarget();
    
    @State("a valid customer exists")
    void setupValidCustomer() {
        // Set up test data for this state
        customerRepository.save(new Customer("cust-123", "John Doe"));
    }
}
```

---

## ⚖️ Contract Testing vs Integration Testing

| | Contract Testing | Integration Testing |
|---|---|---|
| Tests | API shape/contract | Full behavior |
| Speed | Fast (mock one side) | Slower (real dependencies) |
| Scope | Interface agreement | Functional correctness |
| Catches | Breaking API changes | Logic bugs, data issues |
| When to run | Every PR, both sides | Before deployment |

---

## ⚠️ Common Pitfalls

1. **Contracts that are too specific** — Don't assert exact values unless they're fixed. Use type matchers (`stringType`, `numberType`) for dynamic values. `assertThat(id).isNotNull()` not `assertThat(id).isEqualTo("txn-123")`.

2. **Provider tests without proper state setup** — The `@State("a valid customer exists")` must create real test data. If the provider test can't set up the state, the contract verification is meaningless.

3. **Not using a Pact Broker** — Sharing contract files via Git is fragile. Use Pact Broker to store contracts, track verification results, and enable "can-i-deploy" checks in CI/CD.

---

## 📝 Interview Q&A

**Q: How do contract tests fit into a microservices CI/CD pipeline?**
> A: (1) **Consumer PR**: generates contract (Pact file), publishes to Pact Broker. (2) **Provider PR**: pulls latest contracts from broker, verifies against its implementation. If verification fails → PR blocked. (3) **Before deploy**: `can-i-deploy` check queries Pact Broker: "Are all consumers' contracts verified by the provider version I'm deploying?" Only deploy if YES. This ensures no service deploys a breaking change. Each service deploys independently BUT knows it won't break others.

---

## 🔗 What to Read Next

1. **[Testing/E2E_Testing.md](./E2E_Testing.md)** — End-to-end testing strategies
2. **[APIs/API_Design_Guidelines.md](../APIs/API_Design_Guidelines.md)** — Designing stable API contracts
3. **[Microservices/DesignEmployeeService.md](../Microservices/DesignEmployeeService.md)** — Microservice communication

---

*[← Integration Testing](./Integration_Testing.md) | [Back to Index](../INDEX.md) | [Next: E2E Testing →](./E2E_Testing.md)*
