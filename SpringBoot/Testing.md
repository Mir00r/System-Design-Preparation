# 🧪 Spring Boot Testing Strategies: Test Like a Pro 🎯

---

> **"Code without tests is broken by design."** — Jacob Kaplan-Moss
>
> **"Testing shows the presence of bugs, not their absence. But it gives you CONFIDENCE to deploy at 5 PM on a Friday."** — Every Senior Dev

---

## 🎯 Why Testing Matters for Interviews

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  INTERVIEW REALITY:                                                     │
│  "How do you ensure code quality?"                                      │
│  "What's your testing strategy?"                                        │
│  "Write a test for this service" (live coding!)                         │
│                                                                         │
│  Companies that ask testing questions:                                  │
│  Google, Amazon, Netflix, Uber, Spotify, ALL FAANG                     │
│                                                                         │
│  WHY: Testing reveals your understanding of:                            │
│  ✅ Design (testable code = well-designed code)                         │
│  ✅ Edge cases (do you think about failures?)                           │
│  ✅ Production readiness (will this break at 3 AM?)                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Table of Contents

1. [The Testing Pyramid](#-the-testing-pyramid)
2. [Unit Testing with Mockito](#-unit-testing-with-mockito)
3. [Integration Testing](#-integration-testing)
4. [Testing REST APIs (MockMvc)](#-testing-rest-apis)
5. [Testing with Testcontainers](#-testing-with-testcontainers)
6. [Testing @Transactional Behavior](#-testing-transactional-behavior)
7. [Testing Spring Security](#-testing-spring-security)
8. [TDD in Spring Boot](#-tdd-in-spring-boot)
9. [Common Testing Anti-Patterns](#-common-testing-anti-patterns)
10. [Interview Q&A](#-interview-qa)

---

## 🔺 The Testing Pyramid

```
                    ▲
                   / \
                  / E2E \          Few, Slow, Expensive
                 / Tests \         (Selenium, Cypress)
                /─────────\
               / Integration \     Some, Medium Speed
              /    Tests      \    (DB, API, Message Queue)
             /─────────────────\
            /    Unit Tests      \  Many, Fast, Cheap
           /─────────────────────\  (Mockito, JUnit)

  GOLDEN RATIO:
  70% Unit Tests (fast, isolated, cheap)
  20% Integration Tests (verify components work together)
  10% E2E Tests (full system validation)
```

### Spring Boot Testing Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <!-- Includes: JUnit 5, Mockito, AssertJ, Hamcrest, JsonPath -->
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

---

## 🎯 Unit Testing with Mockito

### The Philosophy: Test ONE Thing in Isolation

```java
// ✅ GOOD UNIT TEST: Tests OrderService logic, mocks everything else
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private OrderRepository orderRepository;
    
    @Mock
    private PaymentService paymentService;
    
    @Mock
    private NotificationService notificationService;
    
    @InjectMocks
    private OrderService orderService;  // The class under test

    @Test
    @DisplayName("Should create order when payment succeeds")
    void shouldCreateOrderWhenPaymentSucceeds() {
        // GIVEN (Arrange)
        OrderRequest request = new OrderRequest("user-1", List.of(
            new ItemRequest("product-1", 2, BigDecimal.valueOf(29.99))
        ));
        when(paymentService.charge(any())).thenReturn(PaymentResult.success());
        when(orderRepository.save(any())).thenAnswer(inv -> {
            Order order = inv.getArgument(0);
            order.setId(1L);
            return order;
        });
        
        // WHEN (Act)
        Order result = orderService.createOrder(request);
        
        // THEN (Assert)
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        verify(paymentService).charge(any());
        verify(notificationService).sendOrderConfirmation(any());
        verify(orderRepository).save(any());
    }
    
    @Test
    @DisplayName("Should throw exception when payment fails")
    void shouldThrowExceptionWhenPaymentFails() {
        // GIVEN
        OrderRequest request = new OrderRequest("user-1", List.of(
            new ItemRequest("product-1", 1, BigDecimal.TEN)
        ));
        when(paymentService.charge(any())).thenReturn(PaymentResult.failed("Insufficient funds"));
        
        // WHEN & THEN
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(PaymentFailedException.class)
            .hasMessageContaining("Insufficient funds");
        
        // Verify order was NOT saved
        verify(orderRepository, never()).save(any());
    }
}
```

### Mockito Cheat Sheet

| Method | Purpose | Example |
|--------|---------|---------|
| `when().thenReturn()` | Stub a return value | `when(repo.findById(1L)).thenReturn(Optional.of(user))` |
| `when().thenThrow()` | Stub an exception | `when(service.call()).thenThrow(new RuntimeException())` |
| `verify()` | Verify method was called | `verify(repo).save(any())` |
| `verify(times(n))` | Verify call count | `verify(repo, times(2)).save(any())` |
| `verify(never())` | Verify NOT called | `verify(repo, never()).delete(any())` |
| `@Captor` | Capture arguments | `verify(repo).save(captor.capture())` |
| `doNothing().when()` | Stub void methods | `doNothing().when(service).delete(any())` |

---

## 🔗 Integration Testing

### @SpringBootTest — Full Application Context

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
class OrderIntegrationTest {

    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Test
    @DisplayName("Full order creation flow - end to end")
    void shouldCreateOrderEndToEnd() {
        // GIVEN
        OrderRequest request = new OrderRequest("user-1", 
            List.of(new ItemRequest("prod-1", 2, BigDecimal.valueOf(19.99))));
        
        // WHEN
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", request, Order.class);
        
        // THEN
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getStatus()).isEqualTo(OrderStatus.CONFIRMED);
        
        // Verify persisted in DB
        Optional<Order> saved = orderRepository.findById(response.getBody().getId());
        assertThat(saved).isPresent();
    }
}
```

### Slice Tests — Test One Layer at a Time

| Annotation | What It Tests | What It Loads |
|-----------|--------------|---------------|
| `@WebMvcTest` | Controllers only | MVC infrastructure, no service/repo |
| `@DataJpaTest` | Repository layer | JPA + embedded DB, no web layer |
| `@WebFluxTest` | WebFlux controllers | Reactive web layer |
| `@JsonTest` | JSON serialization | Jackson ObjectMapper |
| `@RestClientTest` | REST client | RestTemplate/WebClient mock server |

---

## 🌐 Testing REST APIs

### @WebMvcTest — Test Controllers Without Full App

```java
@WebMvcTest(UserController.class)  // Only loads this controller!
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;
    
    @MockBean  // Creates mock and adds to Spring context
    private UserService userService;
    
    @Test
    @DisplayName("GET /users/{id} returns user when found")
    void shouldReturnUserWhenFound() throws Exception {
        // GIVEN
        User user = new User(1L, "John", "john@example.com");
        when(userService.findById(1L)).thenReturn(Optional.of(user));
        
        // WHEN & THEN
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }
    
    @Test
    @DisplayName("GET /users/{id} returns 404 when not found")
    void shouldReturn404WhenNotFound() throws Exception {
        when(userService.findById(99L)).thenReturn(Optional.empty());
        
        mockMvc.perform(get("/api/users/99"))
            .andExpect(status().isNotFound());
    }
    
    @Test
    @DisplayName("POST /users validates request body")
    void shouldValidateRequestBody() throws Exception {
        // Empty name should fail validation
        String invalidJson = """
            {"name": "", "email": "invalid-email"}
            """;
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidJson))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isNotEmpty());
    }
}
```

---

## 🐳 Testing with Testcontainers

### Real Database Testing (Not H2!)

```java
@SpringBootTest
@Testcontainers
@ActiveProfiles("integration")
class UserRepositoryIntegrationTest {

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
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @DisplayName("Custom query works with real PostgreSQL")
    void shouldFindUsersByEmail() {
        // GIVEN
        userRepository.save(new User("Alice", "alice@company.com"));
        userRepository.save(new User("Bob", "bob@company.com"));
        
        // WHEN
        List<User> result = userRepository.findByEmailContaining("@company.com");
        
        // THEN
        assertThat(result).hasSize(2);
    }
}
```

### Why Testcontainers Over H2?

```
❌ H2 Problems:
  - Different SQL dialect (H2 ≠ PostgreSQL)
  - Missing features (JSONB, full-text search, CTEs)
  - Tests pass on H2, FAIL in production PostgreSQL!
  
✅ Testcontainers Benefits:
  - Real database in Docker container
  - Same dialect as production
  - Catches real SQL issues
  - Also works with: Redis, Kafka, MongoDB, Elasticsearch
```

---

## 🔄 Testing @Transactional Behavior

### Verify Rollback on Exception

```java
@SpringBootTest
@Transactional  // Each test runs in a transaction, rolled back after!
class TransferServiceTest {

    @Autowired
    private TransferService transferService;
    
    @Autowired
    private AccountRepository accountRepository;
    
    @Test
    @DisplayName("Transfer rolls back when recipient account invalid")
    void shouldRollbackOnFailure() {
        // GIVEN
        Account sender = accountRepository.save(new Account("Alice", BigDecimal.valueOf(1000)));
        
        // WHEN & THEN
        assertThatThrownBy(() -> 
            transferService.transfer(sender.getId(), 999L, BigDecimal.valueOf(500)))
            .isInstanceOf(AccountNotFoundException.class);
        
        // Verify sender's balance unchanged (transaction rolled back!)
        Account refreshed = accountRepository.findById(sender.getId()).get();
        assertThat(refreshed.getBalance()).isEqualByComparingTo(BigDecimal.valueOf(1000));
    }
}
```

---

## 🔒 Testing Spring Security

```java
@WebMvcTest(AdminController.class)
class AdminControllerSecurityTest {

    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private AdminService adminService;
    
    @Test
    @DisplayName("Unauthenticated user gets 401")
    void shouldReturn401WhenNotAuthenticated() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
            .andExpect(status().isUnauthorized());
    }
    
    @Test
    @WithMockUser(roles = "USER")
    @DisplayName("Regular user gets 403 for admin endpoint")
    void shouldReturn403ForNonAdmin() throws Exception {
        mockMvc.perform(get("/api/admin/dashboard"))
            .andExpect(status().isForbidden());
    }
    
    @Test
    @WithMockUser(roles = "ADMIN")
    @DisplayName("Admin user can access admin endpoint")
    void shouldReturn200ForAdmin() throws Exception {
        when(adminService.getDashboard()).thenReturn(new Dashboard());
        
        mockMvc.perform(get("/api/admin/dashboard"))
            .andExpect(status().isOk());
    }
}
```

---

## 🔴🟢 TDD in Spring Boot

### The Red-Green-Refactor Cycle

```
1. 🔴 RED:    Write a failing test (define expected behavior)
2. 🟢 GREEN:  Write minimum code to make test pass
3. 🔄 REFACTOR: Clean up without changing behavior

EXAMPLE: Building a discount calculator

// Step 1: 🔴 Write failing test
@Test
void shouldApply10PercentDiscountForOrdersOver100() {
    DiscountService service = new DiscountService();
    BigDecimal discount = service.calculateDiscount(BigDecimal.valueOf(200));
    assertThat(discount).isEqualByComparingTo(BigDecimal.valueOf(20)); // 10% of 200
}
// Test FAILS: DiscountService doesn't exist yet!

// Step 2: 🟢 Make it pass (simplest implementation)
public class DiscountService {
    public BigDecimal calculateDiscount(BigDecimal orderTotal) {
        if (orderTotal.compareTo(BigDecimal.valueOf(100)) > 0) {
            return orderTotal.multiply(BigDecimal.valueOf(0.10));
        }
        return BigDecimal.ZERO;
    }
}
// Test PASSES! ✅

// Step 3: 🔄 Refactor (extract constants, improve naming)
```

---

## 🚫 Common Testing Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| **Testing Implementation** | Tests break when refactoring | Test behavior, not how |
| **No Assertions** | Test always passes | Every test needs assertThat() |
| **Giant Test Methods** | Hard to understand failures | One concept per test |
| **Shared Mutable State** | Tests influence each other | @BeforeEach reset, @Transactional |
| **Sleeping in Tests** | Flaky, slow | Use Awaitility for async |
| **Testing Frameworks** | Testing Spring, not your code | Trust the framework, test YOUR logic |
| **@SpringBootTest Everywhere** | Slow test suite | Use slice tests (@WebMvcTest) |

### Speed Comparison

```
@SpringBootTest:       ~3-5 seconds per test class (loads EVERYTHING)
@WebMvcTest:           ~0.5-1 second (loads web layer only)
@DataJpaTest:          ~1-2 seconds (loads JPA only)
Plain Mockito (@Mock): ~0.01 seconds (no Spring context!) ⚡
```

---

## 🎓 Interview Q&A

### Q1: "What's the difference between @Mock and @MockBean?"

| | `@Mock` (Mockito) | `@MockBean` (Spring) |
|---|---|---|
| **Context** | No Spring context | Adds mock to Spring ApplicationContext |
| **Speed** | Ultra fast | Requires context startup |
| **Use With** | `@ExtendWith(MockitoExtension.class)` | `@SpringBootTest` or `@WebMvcTest` |
| **When** | Unit tests | Integration tests needing Spring wiring |

### Q2: "How do you test a @Transactional method?"

**Answer**: "I write an integration test with `@SpringBootTest`, perform the operation, then verify the database state. For rollback testing, I intentionally trigger an exception and verify that no partial data was persisted. Using `@Transactional` on the test itself ensures each test gets a clean slate."

### Q3: "What's your testing strategy for a new microservice?"

**Answer Framework:**
```
1. Unit tests (70%): Service layer logic with Mockito
2. Slice tests (15%): @WebMvcTest for controllers, @DataJpaTest for repos
3. Integration tests (10%): Testcontainers for real DB/Kafka
4. Contract tests (5%): Spring Cloud Contract for API compatibility
```

---

## 🎲 Boss Battle: Write the Test! 📝

> **Scenario**: Given this service method, write a complete test:
> ```java
> @Service
> public class TransferService {
>     public void transfer(Long fromId, Long toId, BigDecimal amount) {
>         Account from = accountRepo.findById(fromId).orElseThrow();
>         Account to = accountRepo.findById(toId).orElseThrow();
>         if (from.getBalance().compareTo(amount) < 0) {
>             throw new InsufficientFundsException();
>         }
>         from.debit(amount);
>         to.credit(amount);
>         accountRepo.save(from);
>         accountRepo.save(to);
>     }
> }
> ```
>
> <details>
> <summary>🔓 Click to reveal answer</summary>
>
> ```java
> @ExtendWith(MockitoExtension.class)
> class TransferServiceTest {
>     @Mock AccountRepository accountRepo;
>     @InjectMocks TransferService transferService;
>     
>     @Test
>     void shouldTransferMoneyBetweenAccounts() {
>         Account from = new Account(1L, BigDecimal.valueOf(1000));
>         Account to = new Account(2L, BigDecimal.valueOf(500));
>         when(accountRepo.findById(1L)).thenReturn(Optional.of(from));
>         when(accountRepo.findById(2L)).thenReturn(Optional.of(to));
>         
>         transferService.transfer(1L, 2L, BigDecimal.valueOf(200));
>         
>         assertThat(from.getBalance()).isEqualByComparingTo("800");
>         assertThat(to.getBalance()).isEqualByComparingTo("700");
>         verify(accountRepo, times(2)).save(any());
>     }
>     
>     @Test
>     void shouldThrowWhenInsufficientFunds() {
>         Account from = new Account(1L, BigDecimal.valueOf(100));
>         when(accountRepo.findById(1L)).thenReturn(Optional.of(from));
>         
>         assertThatThrownBy(() -> 
>             transferService.transfer(1L, 2L, BigDecimal.valueOf(500)))
>             .isInstanceOf(InsufficientFundsException.class);
>         verify(accountRepo, never()).save(any());
>     }
> }
> ```
> </details>

---

*Remember: Tests are not just about catching bugs — they're DOCUMENTATION of how your code should behave!* 🧪✨
