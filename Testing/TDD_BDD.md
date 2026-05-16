# 🔄 TDD & BDD: Test-Driven and Behavior-Driven Development

> *"TDD doesn't slow you down — it speeds you up by eliminating debugging time. Write a failing test, make it pass, refactor. Red → Green → Refactor. The tests guide your design toward simplicity."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Unit Testing](./Unit_Testing.md)

---

## 📋 Table of Contents
1. [TDD (Test-Driven Development)](#-tdd)
2. [The Red-Green-Refactor Cycle](#-red-green-refactor)
3. [TDD Example](#-tdd-example)
4. [BDD (Behavior-Driven Development)](#-bdd)
5. [BDD with Cucumber](#-bdd-with-cucumber)
6. [Common Pitfalls](#-common-pitfalls)
7. [Interview Q&A](#-interview-qa)

---

## 🧪 TDD

```
TEST-DRIVEN DEVELOPMENT:
  Write the test FIRST → then write the code to make it pass

TRADITIONAL:
  Write code → Write tests → Hope tests cover everything
  Problem: tests are afterthought, cover happy path only
  
TDD:
  Write test (fails) → Write MINIMUM code to pass → Refactor
  Benefit: 100% of written code is tested, design emerges naturally

THE THREE LAWS OF TDD (Uncle Bob):
  1. You may NOT write production code until you have a failing test
  2. You may NOT write more of a test than is sufficient to fail
  3. You may NOT write more production code than is sufficient to pass

WHY TDD:
  ✅ Forces clear thinking about requirements BEFORE coding
  ✅ Produces testable code by design (loosely coupled)
  ✅ Living documentation (tests describe what code does)
  ✅ Fearless refactoring (change anything, tests catch regressions)
  ✅ No debugging: if test passes, feature works. Period.
  ✅ Faster development (counter-intuitive but proven)
```

---

## 🚦 Red-Green-Refactor

```
  ┌─────────┐         ┌─────────┐         ┌─────────┐
  │  RED    │────────▶│  GREEN  │────────▶│ REFACTOR│
  │ (fail) │         │ (pass)  │         │ (clean) │
  └─────────┘         └─────────┘         └────┬────┘
       ▲                                        │
       └────────────────────────────────────────┘
       
  RED:    Write a test that FAILS (compile error counts as fail)
          - Be specific: what behavior do you want?
          - Run it — see it fail (verify test works!)
  
  GREEN:  Write the SIMPLEST code to make the test pass
          - Don't over-engineer
          - Hardcode if that's simplest (seriously!)
          - No refactoring yet — just make it GREEN
  
  REFACTOR: Clean up the code (remove duplication, improve names)
          - Tests still pass? Good — keep going
          - Tests fail? You broke something — undo and try again
          
  Repeat: next test for next behavior
```

---

## 💻 TDD Example

```java
// REQUIREMENT: FizzBuzz — return "Fizz" for multiples of 3, 
//   "Buzz" for 5, "FizzBuzz" for both, else the number

// STEP 1: RED — simplest failing test
@Test
void shouldReturn1For1() {
    assertThat(fizzBuzz(1)).isEqualTo("1");
}
// FAILS: fizzBuzz() doesn't exist yet!

// STEP 2: GREEN — simplest passing code
String fizzBuzz(int n) { return "1"; }  // hardcoded!
// PASSES ✅ (yes, this is valid TDD!)

// STEP 3: RED — next test forces real implementation
@Test
void shouldReturn2For2() {
    assertThat(fizzBuzz(2)).isEqualTo("2");
}
// FAILS: returns "1" for everything

// STEP 4: GREEN — generalize
String fizzBuzz(int n) { return String.valueOf(n); }
// PASSES ✅

// STEP 5: RED — add Fizz behavior
@Test
void shouldReturnFizzFor3() {
    assertThat(fizzBuzz(3)).isEqualTo("Fizz");
}
// FAILS: returns "3"

// STEP 6: GREEN
String fizzBuzz(int n) {
    if (n % 3 == 0) return "Fizz";
    return String.valueOf(n);
}
// PASSES ✅

// Continue: Buzz (5), FizzBuzz (15), edge cases...
// Each cycle: RED → GREEN → REFACTOR
// Design EMERGES from tests, not from upfront planning
```

---

## 📖 BDD

```
BEHAVIOR-DRIVEN DEVELOPMENT:
  Extension of TDD that uses natural language (Given/When/Then)
  Focuses on BEHAVIOR from user's perspective, not implementation
  Bridges gap between business people and developers

TDD:  "Test that calculateDiscount returns 20 for orders over 100"
BDD:  "Given a customer with a $110 order, When they checkout, Then they get 20% off"

BDD LANGUAGE (Gherkin):
  Feature: describe the feature
  Scenario: specific example of the feature
  Given:    setup / preconditions (arrange)
  When:     action being tested (act)
  Then:     expected outcome (assert)
  And:      additional steps in any section

BENEFITS:
  ✅ Non-technical stakeholders can read/write scenarios
  ✅ Shared understanding of requirements
  ✅ Executable specifications (living documentation)
  ✅ Catches requirement misunderstandings EARLY
```

---

## 🥒 BDD with Cucumber

```gherkin
# features/order_checkout.feature
Feature: Order Checkout
  As a customer
  I want to checkout my cart
  So that I receive my products

  Scenario: Successful checkout with discount
    Given a customer "John" with a cart containing:
      | product | price  | quantity |
      | Widget  | 60.00  | 1        |
      | Gadget  | 50.00  | 1        |
    When John proceeds to checkout
    Then the order total should be $88.00
    And the discount applied should be 20%
    And the order status should be "PENDING"

  Scenario: Checkout fails with empty cart
    Given a customer "Jane" with an empty cart
    When Jane proceeds to checkout
    Then the checkout should fail with "Cart is empty"
```

```java
// Step definitions (Java + Cucumber)
public class CheckoutStepDefinitions {
    private Customer customer;
    private CheckoutResult result;
    
    @Given("a customer {string} with a cart containing:")
    public void customerWithCart(String name, DataTable items) {
        customer = new Customer(name);
        items.asMaps().forEach(row -> 
            customer.addToCart(new CartItem(
                row.get("product"),
                Double.parseDouble(row.get("price")),
                Integer.parseInt(row.get("quantity"))
            )));
    }
    
    @When("{word} proceeds to checkout")
    public void proceedsToCheckout(String customerName) {
        result = checkoutService.checkout(customer);
    }
    
    @Then("the order total should be ${double}")
    public void orderTotalShouldBe(double expectedTotal) {
        assertThat(result.getOrder().getTotal()).isEqualTo(expectedTotal);
    }
    
    @Then("the discount applied should be {int}%")
    public void discountShouldBe(int expectedDiscount) {
        assertThat(result.getOrder().getDiscountPercent()).isEqualTo(expectedDiscount);
    }
}
```

---

## ⚖️ TDD vs BDD

| | TDD | BDD |
|---|---|---|
| Focus | Code correctness | Business behavior |
| Language | Technical (assertions) | Natural (Given/When/Then) |
| Audience | Developers | Developers + Business |
| Granularity | Function/method level | Feature/scenario level |
| Tools | JUnit, Mockito | Cucumber, JBehave |
| When | Always (for unit tests) | Acceptance criteria, features |

---

## ⚠️ Common Pitfalls

1. **Writing tests after the code** — "I'll write tests later" = tests never get written, or they test implementation (not behavior). Discipline: write the test FIRST, even if it feels slow initially.

2. **Too-large test steps** — TDD works with SMALL increments. If you write 5 tests at once then 200 lines of code, you're not doing TDD. One test → simplest code → repeat.

3. **BDD scenarios that are too technical** — `Given the database has a row with id=5` is not BDD. Write from the user's perspective: `Given a customer with an existing order`. Business people should be able to read scenarios.

4. **TDD dogmatism** — Not EVERYTHING needs TDD. Exploratory code, prototypes, UI layout — TDD doesn't always fit. Use TDD for business logic, algorithms, and anything with clear inputs/outputs.

---

## 📝 Interview Q&A

**Q: What are the benefits and challenges of TDD in practice?**
> A: **Benefits**: (1) Better design — testable code is loosely coupled by necessity. (2) Confidence to refactor — change anything, tests catch regressions immediately. (3) Documentation — tests describe what code does. (4) Less debugging — when a test passes, feature works. (5) Fewer bugs in production (IBM study: 40-90% fewer defects). **Challenges**: (1) Learning curve — feels slow initially (2-4 weeks to get comfortable). (2) Discipline required — easy to skip "just this once." (3) Legacy code — hard to retrofit TDD (use characterization tests first). (4) Changing requirements — tests need updating when requirements change. (5) Over-testing — beginners test trivial things. Overall: TDD is an investment that pays off within weeks for complex business logic.

---

## 🔗 What to Read Next

1. **[Testing/Unit_Testing.md](./Unit_Testing.md)** — JUnit 5 and Mockito
2. **[Testing/Integration_Testing.md](./Integration_Testing.md)** — Testing with real dependencies
3. **[Principles/Clean_Code_Principles.md](../Principles/Clean_Code_Principles.md)** — Clean code practices

---

*[← Performance Testing](./Performance_Testing.md) | [Back to Index](../INDEX.md)*
