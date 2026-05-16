# 🔬 Unit Testing: JUnit 5, Mockito & Best Practices

> *"A unit test verifies that a single unit of behavior works correctly in isolation. If your unit test needs a database, a web server, or an external service — it's not a unit test."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Testing Pyramid](./Testing_Pyramid.md), Java basics

---

## 📋 Table of Contents
1. [Unit Testing Principles](#-principles)
2. [JUnit 5 Essentials](#-junit-5)
3. [Mockito (Mocking Dependencies)](#-mockito)
4. [Test Patterns](#-test-patterns)
5. [Common Pitfalls](#-common-pitfalls)
6. [Interview Q&A](#-interview-qa)

---

## 🎯 Principles

```
FIRST Principles of Unit Tests:
  F — Fast:       < 10ms per test (run thousands in seconds)
  I — Isolated:   no shared state, no ordering dependency
  R — Repeatable: same result every time (no randomness, no network)
  S — Self-validating: pass or fail (no manual inspection)
  T — Timely:     written WITH the code (not weeks later)

WHAT TO UNIT TEST:
  ✅ Business logic (calculations, rules, state machines)
  ✅ Data transformations (mapping, formatting, parsing)
  ✅ Validation (input validation, domain invariants)
  ✅ Edge cases (null, empty, boundary values, overflow)
  ✅ Error handling (exceptions thrown correctly)
  
  ❌ Trivial getters/setters (no logic = no value testing)
  ❌ Framework configuration (Spring wiring, annotations)
  ❌ Third-party library behavior (they have their own tests)
```

---

## 🧪 JUnit 5

```java
import org.junit.jupiter.api.*;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.*;
import static org.assertj.core.api.Assertions.*;

class OrderServiceTest {
    
    private OrderService orderService;
    private PaymentGateway mockPayment; // will be mocked
    
    @BeforeEach
    void setUp() {
        mockPayment = Mockito.mock(PaymentGateway.class);
        orderService = new OrderService(mockPayment);
    }
    
    @Test
    @DisplayName("should calculate 20% discount for orders over $100")
    void shouldApplyDiscountForLargeOrders() {
        // Arrange
        Order order = new Order(List.of(
            new LineItem("Widget", 60.00),
            new LineItem("Gadget", 50.00)
        )); // total = $110
        
        // Act
        double total = orderService.calculateTotal(order);
        
        // Assert
        assertThat(total).isEqualTo(88.00); // 110 * 0.8 = 88
    }
    
    @Test
    @DisplayName("should throw exception for empty order")
    void shouldRejectEmptyOrder() {
        Order emptyOrder = new Order(List.of());
        
        assertThatThrownBy(() -> orderService.calculateTotal(emptyOrder))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("Order must have at least one item");
    }
    
    @ParameterizedTest
    @CsvSource({
        "50.0,  0,  50.0",    // no discount under $100
        "100.0, 0,  100.0",   // exactly $100 = no discount
        "100.01, 20, 80.008", // just over $100 = 20% off
        "200.0, 20, 160.0"   // $200 = 20% off
    })
    @DisplayName("should apply correct discount based on total")
    void shouldApplyDiscountRules(double subtotal, int expectedDiscountPercent, double expectedTotal) {
        Order order = OrderFixture.withSubtotal(subtotal);
        
        double total = orderService.calculateTotal(order);
        
        assertThat(total).isCloseTo(expectedTotal, within(0.01));
    }
    
    @Nested
    @DisplayName("Payment Processing")
    class PaymentProcessingTests {
        
        @Test
        @DisplayName("should mark order as PAID when payment succeeds")
        void shouldMarkOrderPaidOnSuccess() {
            when(mockPayment.charge(any())).thenReturn(PaymentResult.success("txn_123"));
            
            Order order = OrderFixture.readyForPayment();
            orderService.processPayment(order);
            
            assertThat(order.getStatus()).isEqualTo(OrderStatus.PAID);
            assertThat(order.getTransactionId()).isEqualTo("txn_123");
        }
        
        @Test
        @DisplayName("should mark order as PAYMENT_FAILED when payment declines")
        void shouldHandlePaymentDecline() {
            when(mockPayment.charge(any())).thenReturn(PaymentResult.declined("insufficient funds"));
            
            Order order = OrderFixture.readyForPayment();
            orderService.processPayment(order);
            
            assertThat(order.getStatus()).isEqualTo(OrderStatus.PAYMENT_FAILED);
        }
    }
}
```

---

## 🎭 Mockito

```java
import static org.mockito.Mockito.*;

// CREATING MOCKS
UserRepository mockRepo = mock(UserRepository.class);   // manual
@Mock UserRepository mockRepo;                          // annotation
@InjectMocks UserService userService;                   // auto-injects mocks

// STUBBING (define behavior)
when(mockRepo.findById(1L)).thenReturn(Optional.of(user));
when(mockRepo.findById(999L)).thenReturn(Optional.empty());
when(mockRepo.save(any(User.class))).thenAnswer(inv -> inv.getArgument(0));

// VERIFICATION (assert interactions)
verify(mockRepo).save(any(User.class));           // was called once
verify(mockRepo, times(2)).findById(anyLong());   // called twice
verify(mockRepo, never()).delete(any());          // never called
verifyNoMoreInteractions(mockRepo);              // nothing else called

// ARGUMENT CAPTORS (capture what was passed)
ArgumentCaptor<User> captor = ArgumentCaptor.forClass(User.class);
verify(mockRepo).save(captor.capture());
User savedUser = captor.getValue();
assertThat(savedUser.getName()).isEqualTo("John");

// SPIES (partial mocking — real object, override specific methods)
List<String> realList = spy(new ArrayList<>());
realList.add("hello");  // real add
when(realList.size()).thenReturn(100);  // overridden
```

---

## 🏗️ Test Patterns

```
PATTERN 1: ARRANGE-ACT-ASSERT (AAA)
  @Test
  void shouldDoSomething() {
      // Arrange — set up test data and mocks
      User user = new User("john@example.com");
      when(repo.existsByEmail("john@example.com")).thenReturn(false);
      
      // Act — execute the thing being tested
      Result result = service.register(user);
      
      // Assert — verify the outcome
      assertThat(result.isSuccess()).isTrue();
      verify(repo).save(user);
  }

PATTERN 2: BUILDER/FIXTURE for test data
  // Instead of: new User("John", "Doe", "john@test.com", "pwd", Role.USER, true, ...)
  // Use:
  User user = UserFixture.aUser().withEmail("john@test.com").build();
  Order order = OrderFixture.completed().withTotal(99.99).build();

PATTERN 3: ONE ASSERTION PER CONCEPT
  // BAD — multiple unrelated assertions
  assertThat(user.getName()).isEqualTo("John");
  assertThat(user.getEmail()).isEqualTo("john@test.com");
  assertThat(user.getRole()).isEqualTo(Role.USER);
  
  // OK — multiple related assertions (one concept: "user was created correctly")
  assertThat(user)
      .extracting("name", "email", "role")
      .containsExactly("John", "john@test.com", Role.USER);
```

---

## ⚠️ Common Pitfalls

1. **Testing implementation, not behavior** — Don't test that `repository.save()` was called with specific arguments. Test that after registering a user, you can find them. Implementation changes shouldn't break tests.

2. **Mocking everything** — If you mock everything, you're testing that your mocks work, not your code. Only mock external boundaries (database, APIs, messaging). Keep domain logic with real objects.

3. **Fragile tests from over-specification** — `verify(mock, times(1)).method(eq("exact string"))` breaks if implementation adds a retry or changes the string slightly. Use `any()` matchers when exact values don't matter for the test.

4. **No test naming convention** — `test1()`, `testOrder()` tell you nothing. Use `shouldApplyDiscountWhenOrderExceeds100` or `givenInvalidEmail_whenRegister_thenThrowException`.

---

## 📝 Interview Q&A

**Q: What's the difference between a mock and a stub?**
> A: **Stub**: provides canned answers to calls (you define return values). Used for indirect inputs — controlling what the test subject sees. Example: `when(repo.findById(1)).thenReturn(user)`. **Mock**: verifies interactions (you assert something was called). Used for indirect outputs — verifying the test subject did something. Example: `verify(emailService).send(any())`. In practice with Mockito, the same object serves both roles — you stub behavior AND verify calls on the same mock. The distinction matters conceptually: stubs setup the test, mocks verify the outcome.

**Q: When would you NOT write unit tests?**
> A: Skip unit tests for: (1) Trivial code with no logic (getters/setters, data classes). (2) Thin wrappers around framework calls (just delegates to Spring). (3) Integration-heavy code where the value is in the integration (repository query correctness — test with real DB). (4) Rapidly prototyping/exploring (write tests when design stabilizes). Always write unit tests for: business rules, calculations, state machines, validation logic, anything that could break silently.

---

## 🔗 What to Read Next

1. **[Testing/Integration_Testing.md](./Integration_Testing.md)** — Testing with real databases
2. **[Testing/TDD_BDD.md](./TDD_BDD.md)** — Writing tests first
3. **[Testing/Contract_Testing.md](./Contract_Testing.md)** — API contract verification

---

*[← Testing Pyramid](./Testing_Pyramid.md) | [Back to Index](../INDEX.md) | [Next: Integration Testing →](./Integration_Testing.md)*
