# **Java `sealed` Keyword: Ultimate Guide for Interview Preparation** 🚀

The **`sealed`** keyword, introduced in **Java 15** (JEP 360) and finalized in **Java 17**, is a powerful feature that allows developers to **restrict inheritance** in a controlled manner. It enhances **type safety**, **domain modeling**, and **pattern matching** in modern Java applications.

This guide covers everything you need to know for interviews and real-world usage:

✅ **What is the `sealed` keyword?**  
✅ **Why & When to Use `sealed`?**  
✅ **Industry Best Practices**  
✅ **Advantages & Disadvantages**  
✅ **How Big Companies Use `sealed`**  
✅ **Code Examples with Explanations**  
✅ **Diagrams & Visualizations**  
✅ **Interview Q&A** 🎯

---

## **1. What is the `sealed` Keyword in Java?** 🔍

A `sealed` class or interface **restricts which other classes or interfaces can extend or implement it**. This provides **fine-grained control** over inheritance hierarchies.

### **Key Components of `sealed`**
| Keyword | Purpose | Example |
|---------|---------|---------|
| `sealed` | Declares a class/interface as sealed. | `sealed class Shape permits Circle, Square` |
| `permits` | Specifies allowed subclasses. | `permits Circle, Square` |
| `non-sealed` | Allows unrestricted inheritance. | `non-sealed class Circle extends Shape` |
| `final` | Prevents further inheritance. | `final class Square extends Shape` |

---

## **2. Why & When to Use `sealed`?** 🤔

### **✅ When to Use `sealed`**
✔ **Restricted Inheritance** (e.g., domain models like `Shape → Circle, Square`)  
✔ **Enhanced Pattern Matching** (Better `instanceof` and `switch` handling)  
✔ **Security & Maintainability** (Prevents unauthorized subclassing)

### **❌ When NOT to Use `sealed`**
✖ **Open-Ended Hierarchies** (If future extensibility is needed)  
✖ **Overly Complex Models** (If inheritance rules are too restrictive)

---

## **3. Industry Best Practices** 🏆

🔹 **Use `sealed` for well-defined domain models** (e.g., payment methods, shapes).  
🔹 **Combine with `record` for immutable data** (Java 16+).  
🔹 **Prefer `sealed` interfaces for API contracts**.  
🔹 **Document permitted subclasses clearly**.

### **How Big Companies Use `sealed`**
- **Spring 6+** → Uses `sealed` for `HttpMessageReader` hierarchy.
- **Kotlin Interop** → Kotlin’s `sealed class` works seamlessly with Java.
- **FinTech** → Used in payment processing models (`CreditCard`, `PayPal`).

---

## **4. Advantages & Disadvantages** ⚖️

| **Advantages** | **Disadvantages** |
|---------------|------------------|
| ✅ **Type Safety** (Prevents rogue subclasses) | ❌ **Limited Extensibility** |
| ✅ **Better Pattern Matching** | ❌ **Boilerplate** (Must list all subclasses) |
| ✅ **Clear Domain Modeling** | ❌ **Java 17+ Only** |

---

## **5. Code Examples with Explanations** 💻

### **1. Basic `sealed` Class Example**
```java
// Define a sealed hierarchy
sealed class Shape permits Circle, Square, Rectangle {
    abstract double area();
}

// Allowed subclasses
final class Circle extends Shape {
    private final double radius;
    
    Circle(double radius) { this.radius = radius; }
    
    @Override double area() { return Math.PI * radius * radius; }
}

final class Square extends Shape {
    private final double side;
    
    Square(double side) { this.side = side; }
    
    @Override double area() { return side * side; }
}

non-sealed class Rectangle extends Shape {  // Can be extended further
    private final double length, width;
    
    Rectangle(double l, double w) { length = l; width = w; }
    
    @Override double area() { return length * width; }
}
```
📌 **Explanation:**
- `Shape` is `sealed` and only permits `Circle`, `Square`, and `Rectangle`.
- `Circle` and `Square` are `final` (no further inheritance).
- `Rectangle` is `non-sealed` (can be extended).

### **2. `sealed` Interface + Pattern Matching (Java 17+)**
```java
sealed interface PaymentMethod permits CreditCard, PayPal {
    void processPayment(double amount);
}

record CreditCard(String number) implements PaymentMethod {
    @Override public void processPayment(double amount) {
        System.out.println("Processing credit card: " + number);
    }
}

record PayPal(String email) implements PaymentMethod {
    @Override public void processPayment(double amount) {
        System.out.println("Processing PayPal: " + email);
    }
}

public class Main {
    public static void main(String[] args) {
        PaymentMethod pm = new CreditCard("1234-5678-9012");
        
        // Enhanced switch (Java 17+)
        switch (pm) {
            case CreditCard cc -> System.out.println("Credit card: " + cc.number());
            case PayPal pp -> System.out.println("PayPal: " + pp.email());
            // No default needed (exhaustive)
        }
    }
}
```
📌 **Explanation:**
- `PaymentMethod` is a `sealed` interface with two permitted implementations (`CreditCard`, `PayPal`).
- **Pattern matching** ensures **exhaustive** `switch` handling.

---

## **6. Interview Q&A** 🎯

### **Q1: Why was `sealed` introduced in Java?**
**A:** To **control inheritance** and **improve pattern matching** by restricting subclassing to known types.

### **Q2: Can a `sealed` class be extended outside its module?**
**A:** ❌ No, unless the subclass is in the same module or explicitly permitted.

### **Q3: What happens if a subclass is not listed in `permits`?**
**A:** ❌ **Compilation error** – all permitted subclasses must be declared.

### **Q4: How does `sealed` help with `instanceof` checks?**
**A:** The compiler knows **all possible subtypes**, enabling **exhaustive checks** in `switch` statements.

### **Q5: Can a `sealed` interface have `default` methods?**
**A:** ✅ Yes, `default` methods work normally.

---

## **7. Diagrams & Visualizations** 📊

### **`sealed` Class Hierarchy**
```
        ┌─────────────┐
        │   Shape     │ (sealed)
        └─────────────┘
           ／   |   ＼
          /    |     \
┌─────────┐ ┌───────┐ ┌──────────┐
│ Circle  │ │ Square │ │ Rectangle│ (non-sealed)
└─────────┘ └───────┘ └──────────┘
                                 ＼
                                   ┌────────────┐
                                   │  CustomRect│ (extends Rectangle)
                                   └────────────┘
```

### **`sealed` vs. `final` vs. `non-sealed`**
| **Keyword** | **Inheritance Allowed?** | **Use Case** |
|------------|-------------------------|-------------|
| `sealed`   | Only permitted classes  | Controlled hierarchies |
| `final`    | ❌ No                   | Immutable types |
| `non-sealed` | ✅ Yes (unrestricted) | Open extension points |

---

## **8. Conclusion & Recommendations** �

✔ **Use `sealed` for domain modeling** (e.g., payment methods, AST nodes).  
✔ **Combine with `record` and pattern matching** for **immutable data + exhaustive checks**.  
✔ **Avoid overusing** – prefer `final` or `non-sealed` when appropriate.

🚀 **Recommended Technologies:**
- **Java 17+** (Full `sealed` support)
- **Spring 6** (Uses `sealed` in HTTP message handling)
- **Kotlin** (Interoperable `sealed class`)

---

### **Final Thoughts**
The `sealed` keyword is a **game-changer** for **type-safe hierarchies** in Java. Mastering it will **boost your interview performance** and **improve code quality** in real-world applications! 🚀

---

🔗 **Happy Learning!** 🎉  
📢 **Share this guide with your peers!** 💬
