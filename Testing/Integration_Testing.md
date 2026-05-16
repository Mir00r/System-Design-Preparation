# 🔗 Integration Testing: Testcontainers, Spring Boot & Real Dependencies

> *"Unit tests verify your logic is correct. Integration tests verify your code actually works with the database, message broker, and external services it depends on. Without integration tests, you're just testing that your mocks behave as expected."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Unit Testing](./Unit_Testing.md), Spring Boot basics

---

## 📋 Table of Contents
1. [What to Integration Test](#-what-to-test)
2. [Spring Boot Testing](#-spring-boot-testing)
3. [Testcontainers](#-testcontainers)
4. [Repository Tests](#-repository-tests)
5. [API Tests](#-api-tests)
6. [Common Pitfalls](#-common-pitfalls)
7. [Interview Q&A](#-interview-qa)

---

## 🎯 What to Test

```
INTEGRATION TESTS VERIFY BOUNDARIES:
  ✅ Your SQL query actually returns correct data from a real DB
  ✅ Your REST endpoint handles serialization/deserialization correctly
  ✅ Your Kafka consumer processes messages with correct offsets
  ✅ Your Redis caching actually caches and evicts correctly
  ✅ Multiple components wired together produce correct outcome
  
  ❌ Business logic (unit test that)
  ❌ Every permutation of inputs (unit test that)
  ❌ Full user journeys (E2E test that)

TEST SCOPE COMPARISON:
  Unit:        OrderService.calculateDiscount() → mock repo
  Integration: POST /orders → real controller → real service → real DB → response
  E2E:         Browser → login → add to cart → checkout → payment → confirmation
```

---

## 🍃 Spring Boot Testing

```java
// SLICE TESTS (test one layer at a time — faster than full context)

@WebMvcTest(OrderController.class)  // Only loads web layer
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService;  // mock the service layer
    
    @Test
    void shouldReturnOrder() throws Exception {
        when(orderService.findById(1L)).thenReturn(new OrderDTO(1L, "COMPLETED", 99.99));
        
        mockMvc.perform(get("/api/orders/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.status").value("COMPLETED"))
            .andExpect(jsonPath("$.total").value(99.99));
    }
    
    @Test
    void shouldReturn404ForMissingOrder() throws Exception {
        when(orderService.findById(999L)).thenThrow(new OrderNotFoundException(999L));
        
        mockMvc.perform(get("/api/orders/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("Order 999 not found"));
    }
}

@DataJpaTest  // Only loads JPA layer (repositories + in-memory DB)
class OrderRepositoryTest {
    @Autowired OrderRepository repository;
    @Autowired TestEntityManager em;
    
    @Test
    void shouldFindOrdersByCustomer() {
        em.persist(new Order("customer-1", OrderStatus.COMPLETED, 50.0));
        em.persist(new Order("customer-1", OrderStatus.PENDING, 30.0));
        em.persist(new Order("customer-2", OrderStatus.COMPLETED, 70.0));
        
        List<Order> orders = repository.findByCustomerId("customer-1");
        
        assertThat(orders).hasSize(2);
        assertThat(orders).allMatch(o -> o.getCustomerId().equals("customer-1"));
    }
}

// FULL INTEGRATION TEST (loads entire Spring context)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {
    @Autowired TestRestTemplate restTemplate;
    
    @Test
    void shouldCreateAndRetrieveOrder() {
        // Create
        CreateOrderRequest request = new CreateOrderRequest("cust-1", List.of(item));
        ResponseEntity<OrderDTO> createResponse = restTemplate.postForEntity(
            "/api/orders", request, OrderDTO.class);
        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        
        // Retrieve
        Long orderId = createResponse.getBody().getId();
        ResponseEntity<OrderDTO> getResponse = restTemplate.getForEntity(
            "/api/orders/" + orderId, OrderDTO.class);
        assertThat(getResponse.getBody().getStatus()).isEqualTo("PENDING");
    }
}
```

---

## 🐳 Testcontainers

```java
// Real PostgreSQL in Docker for tests (not H2!)
@SpringBootTest
@Testcontainers
class OrderRepositoryIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired OrderRepository repository;
    
    @Test
    void shouldExecuteNativeQueryOnRealPostgres() {
        // This tests REAL PostgreSQL behavior (not H2 approximation)
        // PostgreSQL-specific features: JSONB, arrays, window functions
        Order order = repository.save(new Order("cust-1", OrderStatus.PENDING, 99.99));
        
        List<Order> result = repository.findRecentOrdersWithTotalAbove(50.0);
        assertThat(result).contains(order);
    }
}

// Kafka Testcontainer
@Testcontainers
class OrderEventConsumerTest {
    
    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));
    
    @Test
    void shouldProcessOrderCreatedEvent() {
        // Produce message to real Kafka
        KafkaProducer<String, String> producer = createProducer(kafka.getBootstrapServers());
        producer.send(new ProducerRecord<>("orders", orderCreatedJson));
        
        // Verify consumer processed it
        await().atMost(10, SECONDS).untilAsserted(() ->
            assertThat(orderRepository.findById(orderId)).isPresent());
    }
}

// Redis Testcontainer
@Container
static GenericContainer<?> redis = new GenericContainer<>("redis:7")
    .withExposedPorts(6379);
```

---

## ⚠️ Common Pitfalls

1. **Using H2 instead of real database** — H2 behaves differently from PostgreSQL/MySQL for: JSONB columns, window functions, locking, sequences, case sensitivity. Use Testcontainers with the SAME database as production.

2. **Shared test state** — Tests that depend on data from previous tests are brittle and order-dependent. Use `@Transactional` (auto-rollback) or `@Sql` annotations to reset state. Each test should set up its own data.

3. **Testing too much in integration tests** — Don't test every business rule with a full Spring Boot context. It's slow. Test business rules at unit level. Integration tests verify the wiring works.

---

## 📝 Interview Q&A

**Q: When would you use @WebMvcTest vs @SpringBootTest?**
> A: **@WebMvcTest** loads ONLY the web layer (controllers, filters, advice). Service/repository beans are mocked with @MockBean. Use for: testing request/response mapping, validation, error handling, security filters. Runs fast (~2s). **@SpringBootTest** loads the ENTIRE application context (all beans). Use for: full integration tests that exercise the whole stack (controller → service → repository → DB). Runs slower (~5-10s). Strategy: use @WebMvcTest for controller logic, @DataJpaTest for repository queries, and @SpringBootTest sparingly for critical end-to-end verification within the service.

---

## 🔗 What to Read Next

1. **[Testing/Contract_Testing.md](./Contract_Testing.md)** — Testing APIs between services
2. **[Testing/Performance_Testing.md](./Performance_Testing.md)** — Load testing
3. **[Testing/Unit_Testing.md](./Unit_Testing.md)** — Unit testing patterns

---

*[← Unit Testing](./Unit_Testing.md) | [Back to Index](../INDEX.md) | [Next: Contract Testing →](./Contract_Testing.md)*
