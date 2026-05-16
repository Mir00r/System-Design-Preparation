# 🚦 State Pattern: Behavior Changes with State! 🎯

> **"A vending machine behaves DIFFERENTLY when it has coins vs when it's empty. Same machine, different behavior."**

---

## 🎬 The Story

```
🚦 TRAFFIC LIGHT:
━━━━━━━━━━━━━━━━
🔴 RED state     → Cars STOP, pedestrians walk
🟡 YELLOW state  → Cars slow down, prepare to stop
🟢 GREEN state   → Cars GO, pedestrians wait

Same traffic light. DIFFERENT behavior based on CURRENT STATE.
And it transitions automatically: Red → Green → Yellow → Red → ...
```

---

## 🤔 The Problem

```java
// ❌ BAD: Giant if-else state machine
public class VendingMachine {
    enum State { NO_COIN, HAS_COIN, DISPENSING, SOLD_OUT }
    State state = State.NO_COIN;
    
    public void insertCoin() {
        if (state == State.NO_COIN) { state = State.HAS_COIN; }
        else if (state == State.HAS_COIN) { System.out.println("Already have coin"); }
        else if (state == State.DISPENSING) { System.out.println("Wait!"); }
        else if (state == State.SOLD_OUT) { System.out.println("Sold out!"); }
        // 💀 Adding new state = modify EVERY method!
    }
    
    public void pressButton() { /* Another giant if-else */ }
    public void dispense() { /* Yet another if-else */ }
}
```

---

## 💻 Java Implementation (The Clean Way)

```java
// 🚦 State interface
public interface VendingState {
    void insertCoin(VendingMachine machine);
    void pressButton(VendingMachine machine);
    void dispense(VendingMachine machine);
}

// 🔴 Concrete States
public class NoCoinState implements VendingState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("💰 Coin inserted!");
        machine.setState(new HasCoinState()); // Transition!
    }
    @Override
    public void pressButton(VendingMachine machine) {
        System.out.println("⚠️ Insert coin first!");
    }
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("⚠️ Pay first!");
    }
}

public class HasCoinState implements VendingState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("⚠️ Already have a coin!");
    }
    @Override
    public void pressButton(VendingMachine machine) {
        System.out.println("🔘 Button pressed! Dispensing...");
        machine.setState(new DispensingState()); // Transition!
    }
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("⚠️ Press button first!");
    }
}

public class DispensingState implements VendingState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("⚠️ Please wait...");
    }
    @Override
    public void pressButton(VendingMachine machine) {
        System.out.println("⚠️ Already dispensing!");
    }
    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("🎁 Here's your item!");
        machine.decrementItems();
        machine.setState(machine.getItems() > 0 ? new NoCoinState() : new SoldOutState());
    }
}

// 🎰 Context
public class VendingMachine {
    private VendingState state;
    private int items;
    
    public VendingMachine(int items) {
        this.items = items;
        this.state = items > 0 ? new NoCoinState() : new SoldOutState();
    }
    
    public void setState(VendingState state) { this.state = state; }
    public void insertCoin()  { state.insertCoin(this); }
    public void pressButton() { state.pressButton(this); state.dispense(this); }
    public void decrementItems() { items--; }
    public int getItems() { return items; }
}

// 🎮 Usage
VendingMachine machine = new VendingMachine(3);
machine.insertCoin();   // 💰 Coin inserted!
machine.pressButton();  // 🔘 Dispensing... 🎁 Here's your item!
machine.pressButton();  // ⚠️ Insert coin first!
```

---

## 🌍 Real-World Examples

```java
// 🌱 Spring State Machine — explicit state machine implementation
// 📱 Android Activity Lifecycle (CREATED → STARTED → RESUMED → ...)
// 🛒 Order Status (PENDING → PAID → SHIPPED → DELIVERED)
// 🎮 Game characters (IDLE → RUNNING → JUMPING → ATTACKING)
// 🔌 TCP Connection (LISTEN → SYN_SENT → ESTABLISHED → CLOSE_WAIT)
```

---

## 🎯 Key Interview Point

**State vs Strategy:**
```
STATE: Object changes behavior AUTOMATICALLY based on internal state
→ Transitions happen INSIDE the state classes
→ Client doesn't choose the state

STRATEGY: Client EXPLICITLY chooses the algorithm
→ No automatic transitions
→ Client sets the strategy

State = Traffic light (changes by itself)
Strategy = Music player (YOU pick shuffle/repeat/sequential)
```

---

## 🏆 Achievement: Progress: 18/23 patterns ██████████████████░

---

*← [Iterator](./Iterator.md) | [Chain of Responsibility →](./Chain_of_Responsibility.md)*
