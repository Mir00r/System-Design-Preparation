# 🎰 Design a Vending Machine (OOP + State Machine)

> *"The humble vending machine is the interviewer's favorite OOP question disguised as system design. It seems simple — put money in, get snack out — but it's actually a perfect exercise in state machine design, concurrency handling, and clean object-oriented principles. If you can design a vending machine well, you understand encapsulation, single responsibility, strategy pattern, and state pattern — all in one compact problem."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: [Design Patterns](../DesignPattern/), [Clean Architecture](../Architectures/Clean.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [State Machine Design](#-state-machine-design)
3. [Class Diagram](#-class-diagram)
4. [Java Implementation](#-java-implementation)
5. [Concurrency & Edge Cases](#-concurrency--edge-cases)
6. [Scaling to Connected Machines](#-scaling-to-connected-machines)
7. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Accept coins and bills (1¢, 5¢, 10¢, 25¢, $1, $5)
  • Accept card payments (credit/debit!)
  • Display available products with prices
  • Dispense selected product after sufficient payment
  • Return change (optimal coin combination!)
  • Cancel transaction and refund inserted money
  • Admin: restock products, collect money, view sales report
  • Handle "sold out" gracefully
  
NON-FUNCTIONAL:
  • Thread-safe (concurrent button presses!)
  • Reliable (never eat money without dispensing!)
  • Maintainable (easy to add new payment methods!)
  • Testable (each state independently testable!)

CONSTRAINTS:
  • Single machine = single user at a time
  • Finite inventory slots (e.g., 10 slots × 15 items each)
  • Limited change available (may need to refuse large bills!)
```

---

## 🔄 State Machine Design

```
STATE MACHINE (the HEART of the design!):

  ┌──────────────────────────────────────────────────────────────┐
  │                                                               │
  │  ┌──────────┐  insertMoney()   ┌─────────────┐              │
  │  │   IDLE   │─────────────────►│  HAS_MONEY  │◄─┐           │
  │  │          │◄───────────┐     │             │  │           │
  │  └──────────┘   cancel() │     └──────┬──────┘  │ more     │
  │       │                   │            │          │ money    │
  │       │                   │   select() │          │          │
  │       │                   │            ▼          │          │
  │       │                   │     ┌──────────────┐  │          │
  │       │                   └─────│  DISPENSING  │──┘          │
  │       │                  refund │             │              │
  │       │                         └──────┬──────┘              │
  │       │                                │ dispense()          │
  │       │                                ▼                      │
  │       │                   ┌────────────────────┐             │
  │       │◄──────────────────│  CHANGE_RETURNED   │             │
  │       │   returnToIdle()  │                    │             │
  │       │                   └────────────────────┘             │
  └──────────────────────────────────────────────────────────────┘
  
  STATES:
  • IDLE: waiting for user, displaying "INSERT MONEY"
  • HAS_MONEY: user has inserted money, waiting for selection
  • DISPENSING: product selected, checking payment, dispensing
  • CHANGE_RETURNED: dispensed product + change, resetting

  TRANSITIONS (events):
  • insertMoney(amount): IDLE → HAS_MONEY, or HAS_MONEY → HAS_MONEY
  • selectProduct(slot): HAS_MONEY → DISPENSING (if enough money!)
  • cancel(): HAS_MONEY → IDLE (refund all money!)
  • dispenseComplete(): DISPENSING → CHANGE_RETURNED
  • reset(): CHANGE_RETURNED → IDLE

  INVALID TRANSITIONS (rejected!):
  • selectProduct in IDLE state (no money inserted!)
  • insertMoney in DISPENSING state (busy!)
  • selectProduct for sold-out item!
```

---

## 📐 Class Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         VendingMachine                                │
│  ─────────────────────────────────────────────────                   │
│  - currentState: VendingState                                        │
│  - inventory: Inventory                                              │
│  - paymentProcessor: PaymentProcessor                                │
│  - display: Display                                                  │
│  ─────────────────────────────────────────────────                   │
│  + insertMoney(coin: Coin): void                                     │
│  + selectProduct(slot: int): Product                                 │
│  + cancel(): List<Coin>                                              │
│  + getDisplay(): String                                              │
│  ─────────────────────────────────────────────────                   │
│  - setState(state: VendingState): void                               │
└─────────────────┬───────────────────────────────────────────────────┘
                  │ has-a
    ┌─────────────┼─────────────────┐
    ▼             ▼                 ▼
┌──────────┐ ┌──────────┐  ┌───────────────┐
│Inventory │ │ Payment  │  │ VendingState  │ (interface!)
│──────────│ │Processor │  │───────────────│
│- slots[] │ │──────────│  │+ insertMoney()│
│+ getItem()│ │- balance │  │+ selectProduct│
│+ restock()│ │+ add()   │  │+ cancel()     │
│+ isEmpty()│ │+ refund() │  │+ dispense()   │
└──────────┘ │+ enough()?│  └───────┬───────┘
             └──────────┘          │ implementations
                         ┌─────────┼─────────────┐
                         ▼         ▼             ▼
                  ┌──────────┐ ┌──────────┐ ┌──────────┐
                  │IdleState │ │HasMoney  │ │Dispensing│
                  │          │ │State     │ │State     │
                  └──────────┘ └──────────┘ └──────────┘

DESIGN PATTERNS USED:
  • State Pattern: each state is a class with its own behavior!
  • Strategy Pattern: PaymentProcessor is swappable (cash, card!)
  • Observer Pattern: notify display on state change!
  • Singleton: one VendingMachine instance per physical machine!
```

---

## 💻 Java Implementation

### Core Classes

```java
/**
 * State interface — each state handles events differently!
 */
public interface VendingState {
    void insertMoney(VendingMachine machine, Money money);
    void selectProduct(VendingMachine machine, int slot);
    void cancel(VendingMachine machine);
    void dispense(VendingMachine machine);
}

/**
 * Product in the vending machine.
 */
@Data
@AllArgsConstructor
public class Product {
    private String name;
    private BigDecimal price;
    private int slot;
}

/**
 * Money representation (coins and bills!).
 */
public enum Money {
    PENNY(new BigDecimal("0.01")),
    NICKEL(new BigDecimal("0.05")),
    DIME(new BigDecimal("0.10")),
    QUARTER(new BigDecimal("0.25")),
    DOLLAR(new BigDecimal("1.00")),
    FIVE_DOLLAR(new BigDecimal("5.00"));
    
    @Getter
    private final BigDecimal value;
    Money(BigDecimal value) { this.value = value; }
}
```

### Vending Machine (Context)

```java
/**
 * Main vending machine — delegates to current state!
 * Thread-safe via synchronized methods.
 */
public class VendingMachine {
    
    private VendingState currentState;
    private final Inventory inventory;
    private BigDecimal currentBalance = BigDecimal.ZERO;
    private Product selectedProduct;
    private final List<Money> insertedMoney = new ArrayList<>();
    
    public VendingMachine(int numberOfSlots) {
        this.inventory = new Inventory(numberOfSlots);
        this.currentState = new IdleState(); // Start idle!
    }
    
    // --- Delegated to current state (State Pattern!) ---
    
    public synchronized void insertMoney(Money money) {
        currentState.insertMoney(this, money);
    }
    
    public synchronized void selectProduct(int slot) {
        currentState.selectProduct(this, slot);
    }
    
    public synchronized List<Money> cancel() {
        currentState.cancel(this);
        return returnChange(currentBalance);
    }
    
    // --- Internal state management ---
    
    void setState(VendingState state) {
        this.currentState = state;
        System.out.println("State → " + state.getClass().getSimpleName());
    }
    
    void addBalance(Money money) {
        this.currentBalance = currentBalance.add(money.getValue());
        this.insertedMoney.add(money);
    }
    
    void resetBalance() {
        this.currentBalance = BigDecimal.ZERO;
        this.insertedMoney.clear();
    }
    
    boolean hasSufficientBalance(Product product) {
        return currentBalance.compareTo(product.getPrice()) >= 0;
    }
    
    /**
     * Calculate optimal change (greedy algorithm!).
     */
    List<Money> returnChange(BigDecimal amount) {
        List<Money> change = new ArrayList<>();
        Money[] denominations = {Money.FIVE_DOLLAR, Money.DOLLAR, 
            Money.QUARTER, Money.DIME, Money.NICKEL, Money.PENNY};
        
        BigDecimal remaining = amount;
        for (Money coin : denominations) {
            while (remaining.compareTo(coin.getValue()) >= 0) {
                change.add(coin);
                remaining = remaining.subtract(coin.getValue());
            }
        }
        return change;
    }
    
    // Getters
    public BigDecimal getCurrentBalance() { return currentBalance; }
    public Inventory getInventory() { return inventory; }
    public Product getSelectedProduct() { return selectedProduct; }
    public void setSelectedProduct(Product p) { this.selectedProduct = p; }
}
```

### State Implementations

```java
/**
 * IDLE: waiting for money. Only insertMoney() is valid!
 */
public class IdleState implements VendingState {
    
    @Override
    public void insertMoney(VendingMachine machine, Money money) {
        machine.addBalance(money);
        System.out.println("Inserted: " + money + 
            " | Balance: $" + machine.getCurrentBalance());
        machine.setState(new HasMoneyState()); // Transition!
    }
    
    @Override
    public void selectProduct(VendingMachine machine, int slot) {
        System.out.println("Please insert money first!");
        // Stay in IDLE state — no transition!
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Nothing to cancel!");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Invalid operation in IDLE state!");
    }
}

/**
 * HAS_MONEY: user has inserted money. Can select product or cancel!
 */
public class HasMoneyState implements VendingState {
    
    @Override
    public void insertMoney(VendingMachine machine, Money money) {
        machine.addBalance(money);
        System.out.println("Balance: $" + machine.getCurrentBalance());
        // Stay in HAS_MONEY (can keep adding!)
    }
    
    @Override
    public void selectProduct(VendingMachine machine, int slot) {
        Product product = machine.getInventory().getProduct(slot);
        
        if (product == null) {
            System.out.println("Invalid slot!");
            return;
        }
        if (machine.getInventory().isEmpty(slot)) {
            System.out.println("SOLD OUT! Choose another.");
            return;
        }
        if (!machine.hasSufficientBalance(product)) {
            BigDecimal needed = product.getPrice()
                .subtract(machine.getCurrentBalance());
            System.out.println("Insert $" + needed + " more!");
            return;
        }
        
        // Sufficient funds! Move to dispensing!
        machine.setSelectedProduct(product);
        machine.setState(new DispensingState());
        machine.getCurrentState().dispense(machine); // Auto-dispense!
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cancelling! Returning $" + 
            machine.getCurrentBalance());
        machine.resetBalance();
        machine.setState(new IdleState());
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Select a product first!");
    }
}

/**
 * DISPENSING: product selected, dispensing + returning change!
 */
public class DispensingState implements VendingState {
    
    @Override
    public void insertMoney(VendingMachine machine, Money money) {
        System.out.println("Please wait, dispensing...");
    }
    
    @Override
    public void selectProduct(VendingMachine machine, int slot) {
        System.out.println("Already dispensing, please wait!");
    }
    
    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cannot cancel during dispensing!");
    }
    
    @Override
    public void dispense(VendingMachine machine) {
        Product product = machine.getSelectedProduct();
        
        // Dispense product!
        machine.getInventory().removeItem(product.getSlot());
        System.out.println("🎉 Dispensed: " + product.getName());
        
        // Calculate and return change!
        BigDecimal change = machine.getCurrentBalance()
            .subtract(product.getPrice());
        if (change.compareTo(BigDecimal.ZERO) > 0) {
            List<Money> changeCoins = machine.returnChange(change);
            System.out.println("Change: $" + change + " → " + changeCoins);
        }
        
        // Reset and return to IDLE!
        machine.resetBalance();
        machine.setSelectedProduct(null);
        machine.setState(new IdleState());
    }
}
```

### Inventory Management

```java
/**
 * Manages product slots and quantities.
 */
public class Inventory {
    
    private final Map<Integer, Product> products = new ConcurrentHashMap<>();
    private final Map<Integer, Integer> quantities = new ConcurrentHashMap<>();
    private final int maxSlots;
    
    public Inventory(int maxSlots) {
        this.maxSlots = maxSlots;
    }
    
    public void stock(int slot, Product product, int quantity) {
        if (slot < 0 || slot >= maxSlots) 
            throw new IllegalArgumentException("Invalid slot: " + slot);
        products.put(slot, product);
        quantities.put(slot, quantity);
    }
    
    public Product getProduct(int slot) {
        return products.get(slot);
    }
    
    public boolean isEmpty(int slot) {
        return quantities.getOrDefault(slot, 0) <= 0;
    }
    
    public void removeItem(int slot) {
        quantities.computeIfPresent(slot, (k, v) -> v > 0 ? v - 1 : 0);
    }
    
    public Map<Integer, Product> getAvailableProducts() {
        return products.entrySet().stream()
            .filter(e -> !isEmpty(e.getKey()))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    }
}
```

---

## 🔒 Concurrency & Edge Cases

```
EDGE CASES (what interviewers LOVE to ask!):

  1. EXACT CHANGE ONLY:
     Machine has no coins for change → display "EXACT CHANGE ONLY!"
     Solution: track available change coins, reject if can't make change!
  
  2. COIN JAM:
     Dispenser mechanism fails → DON'T deduct money!
     Solution: only deduct balance AFTER physical dispense confirmed!
     (sensor detects product fell → then deduct!)
  
  3. POWER FAILURE:
     Mid-transaction, power goes out!
     Solution: persist state to non-volatile memory.
     On reboot: check if money was inserted → refund!
  
  4. OVERPAYMENT WITH NO CHANGE:
     $5 bill for $1.25 item, machine has no change!
     Solution: REJECT the bill BEFORE accepting it!
     Pre-check: can I make $3.75 change? No → reject $5 bill!
  
  5. CONCURRENT ACCESS:
     Two buttons pressed simultaneously!
     Solution: synchronized methods (only one thread at a time!)
     Physical machine: single-threaded by nature (one coin slot!)

CONCURRENCY IN CONNECTED MACHINES:
  If machines share inventory (cloud-connected!):
  • Use optimistic locking: check stock → dispense → confirm
  • If stock changed between check and dispense → refund!
```

---

## 📡 Scaling to Connected Machines

```
CONNECTED VENDING MACHINES (IoT!):

  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │  Machine 1  │     │  Machine 2  │     │  Machine N  │
  │  (Office)   │     │  (Mall)     │     │  (Airport)  │
  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              │ MQTT / HTTP
                              ▼
                    ┌───────────────────┐
                    │  Cloud Backend    │
                    │  • Inventory mgmt │
                    │  • Sales analytics│
                    │  • Restock alerts │
                    │  • Remote pricing │
                    │  • Payment proc.  │
                    └───────────────────┘

  Benefits:
  • Real-time inventory → auto-restock orders!
  • Dynamic pricing (surge pricing at events!)
  • Remote diagnostics (machine offline alert!)
  • Sales analytics (which products sell where!)
  • Cashless payments (mobile wallet, NFC!)
```

---

## ❓ Interview Q&A

**Q1: Why use the State pattern instead of if-else chains?**
> Without State pattern: every method has `if (state == IDLE) ... else if (state == HAS_MONEY) ... else if ...` — this becomes unmaintainable with 4+ states and 5+ transitions! With State pattern: each state is its own class handling only its valid transitions. Adding a new state (e.g., MAINTENANCE_MODE) = add ONE new class, no modification to existing code! This follows Open/Closed Principle: open for extension, closed for modification. Testing is also cleaner: test each state class independently!

**Q2: How would you handle the change-making problem optimally?**
> This is a classic greedy/DP problem! Greedy works for standard US denominations (25, 10, 5, 1 — always give largest coin first). But for arbitrary denominations (e.g., 1, 3, 4): greedy fails! (Amount 6: greedy = 4+1+1, optimal = 3+3). Solution: use DP if denominations are non-standard. In practice: vending machines use standard coins → greedy is fine. Also: track available coins in the machine! If you have no quarters, fall back to dimes!

**Q3: How do you ensure atomicity — never eat money without dispensing?**
> The key insight: separate "accept money" from "commit transaction". (1) Money is held in escrow (inserted but not deposited yet), (2) Only AFTER product physically dispensed (sensor confirms!) → move money from escrow to vault, (3) If dispense fails → return all escrowed money! This is like a 2-phase commit: prepare (accept money) → commit (dispense + deposit) OR rollback (return money). In code: wrap dispense in try-catch, catch block always refunds!

**Q4: How would you design this for a distributed system (1000 machines)?**
> Each machine operates independently (local state machine) — no network dependency for basic operation! But connects to cloud for: (1) Payment processing (Stripe/Square API for card payments), (2) Inventory sync (report stock levels every 5 min), (3) Configuration (price updates pushed via MQTT), (4) Analytics (sales events streamed to Kafka!). Key design: machine works OFFLINE if network fails (accept cash, use cached prices, queue events for later sync). This is an edge computing pattern: compute at the edge, sync to cloud async!

---

## 🏆 Mini Challenge

```
🎮 CHALLENGE: Extend the vending machine!

  Add a LOYALTY PROGRAM:
  • Users scan a QR code (loyalty card!)
  • Every 10th purchase = FREE item!
  • Track purchase history per user
  • Display "3 more purchases until free item!" 
  
  Questions to think about:
  1. Where does loyalty state live? (machine or cloud?)
  2. What if user uses multiple machines?
  3. How to prevent abuse (scanning same code twice)?
  4. What design pattern would you add?
     (Hint: Decorator pattern for discounts!)
```

---

## 🔗 Related Topics
- [Design Patterns](../DesignPattern/) — State, Strategy, Observer
- [Design Parking Garage](./DesignParkingGarage.md) — Similar OOP problem
- [SOLID Principles](../Principles/) — Applied here!

---

*"The vending machine problem isn't about vending machines. It's about proving you can decompose a real-world system into clean objects with clear responsibilities, handle edge cases gracefully, and design state transitions that are impossible to corrupt. Master this, and you'll nail any OOP interview question they throw at you."* 🎰
