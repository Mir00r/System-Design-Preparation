# **The Principles of Clean Code: DRY, KISS, and YAGNI**

Clean Code principles help developers write readable, maintainable, and efficient code. Three of the most important principles are **DRY (Don’t Repeat Yourself), KISS (Keep It Simple, Stupid), and YAGNI (You Ain’t Gonna Need It)**. These principles improve code quality, reduce bugs, and make code easier to modify.

---

## **1️⃣ DRY – Don’t Repeat Yourself**
### **What is DRY?**
**DRY (Don’t Repeat Yourself)** states that **a piece of knowledge should exist in only one place in a system**. Instead of writing duplicate code, we should abstract common logic into reusable components.

### **Why is DRY important?**
- Reduces code duplication.
- Improves maintainability—changes need to be made in only one place.
- Makes code more readable and concise.
- Reduces the chances of inconsistency and bugs.

### **Example of Code Violation (Not Following DRY)**
```java
public class InvoiceService {
    public double calculateTotal(double price, int quantity) {
        return price * quantity + (price * quantity * 0.15); // 15% tax
    }
}

public class OrderService {
    public double computeAmount(double price, int quantity) {
        return price * quantity + (price * quantity * 0.15); // Duplicate tax logic
    }
}
```
🔴 **Problem:** The tax calculation logic is duplicated in two places. If we need to change the tax rate, we must update it in multiple places.

### **Refactored Code Following DRY**
```java
public class TaxCalculator {
    public static double calculateTax(double amount) {
        return amount * 0.15;
    }
}

public class InvoiceService {
    public double calculateTotal(double price, int quantity) {
        double subtotal = price * quantity;
        return subtotal + TaxCalculator.calculateTax(subtotal);
    }
}

public class OrderService {
    public double computeAmount(double price, int quantity) {
        double subtotal = price * quantity;
        return subtotal + TaxCalculator.calculateTax(subtotal);
    }
}
```
✅ **Solution:** The tax calculation logic is extracted into a separate `TaxCalculator` class. Now, any changes to tax logic are made in one place.

---

## **2️⃣ KISS – Keep It Simple, Stupid**
### **What is KISS?**
**KISS (Keep It Simple, Stupid)** suggests that software should be **as simple as possible** while still being functional. **Complexity should be avoided unless necessary**.

### **Why is KISS important?**
- Easier to read and understand.
- Easier to maintain and debug.
- Reduces unnecessary complexity and overhead.

### **Example of Code Violation (Not Following KISS)**
```java
public class DiscountService {
    public double calculateDiscount(double price, int customerLoyaltyPoints) {
        if (customerLoyaltyPoints > 1000) {
            return price * 0.10; // 10% discount
        } else if (customerLoyaltyPoints > 500) {
            return price * 0.07; // 7% discount
        } else if (customerLoyaltyPoints > 100) {
            return price * 0.05; // 5% discount
        } else {
            return 0;
        }
    }
}
```
🔴 **Problem:** The method has too many conditions, making it hard to read and maintain.

### **Refactored Code Following KISS**
```java
public class DiscountService {
    public double calculateDiscount(double price, int customerLoyaltyPoints) {
        double discountRate = getDiscountRate(customerLoyaltyPoints);
        return price * discountRate;
    }

    private double getDiscountRate(int points) {
        return (points > 1000) ? 0.10 : (points > 500) ? 0.07 : (points > 100) ? 0.05 : 0.0;
    }
}
```
✅ **Solution:** The logic is simplified by extracting the discount rate calculation into a separate method using the ternary operator.

---

## **3️⃣ YAGNI – You Ain’t Gonna Need It**
### **What is YAGNI?**
**YAGNI (You Ain’t Gonna Need It)** advises **against adding unnecessary features or complexity that are not immediately required**.

### **Why is YAGNI important?**
- Avoids unnecessary code.
- Reduces development time.
- Keeps the codebase clean and maintainable.
- Avoids technical debt.

### **Example of Code Violation (Not Following YAGNI)**
```java
public class UserService {
    public void registerUser(String username, String password) {
        if (username.isEmpty() || password.isEmpty()) {
            throw new IllegalArgumentException("Username and password cannot be empty");
        }
        // Registration logic...
    }

    public void registerUser(String username, String password, String phoneNumber, String address, String dateOfBirth) {
        if (username.isEmpty() || password.isEmpty()) {
            throw new IllegalArgumentException("Username and password cannot be empty");
        }
        // Unused parameters phoneNumber, address, dateOfBirth...
    }
}
```
🔴 **Problem:** The second `registerUser` method adds unnecessary parameters (`phoneNumber, address, dateOfBirth`) that are **not needed for user registration**.

### **Refactored Code Following YAGNI**
```java
public class UserService {
    public void registerUser(String username, String password) {
        if (username.isEmpty() || password.isEmpty()) {
            throw new IllegalArgumentException("Username and password cannot be empty");
        }
        // Registration logic...
    }
}
```
✅ **Solution:** Removed unnecessary parameters and kept only the required ones.

---

## **Conclusion:**
| Principle | Meaning | Why It's Important | Example Violation | Solution |
|-----------|---------|--------------------|-------------------|----------|
| **DRY** (Don’t Repeat Yourself) | Avoid duplicate code | Easier maintenance, fewer bugs | Same logic repeated in multiple places | Extract common logic into reusable methods/classes |
| **KISS** (Keep It Simple, Stupid) | Keep code simple and easy to understand | Improves readability and maintainability | Overcomplicated logic with unnecessary conditions | Simplify code with clear and concise logic |
| **YAGNI** (You Ain’t Gonna Need It) | Avoid adding unnecessary features | Prevents bloated and unmaintainable code | Adding extra parameters and features that are not required | Implement only what is needed for current requirements |

### **Final Thought:**
By following **DRY, KISS, and YAGNI**, you can write **clean, maintainable, and efficient** code, which is essential for scalable software development.
