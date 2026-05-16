# ⛓️ Chain of Responsibility: Pass Until Someone Handles It! 🎯

> **"Customer complaint? Cashier → Manager → Owner. Someone will handle it!"**

---

## 🎬 The Story

```
🏢 TECH SUPPORT ESCALATION:
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Customer: "My app crashed!"

Level 1 (Bot):     "Have you tried restarting?" → Can't handle? PASS! →
Level 2 (Junior):  "Let me check your logs..." → Can't handle? PASS! →
Level 3 (Senior):  "I see the issue, fixing..." → ✅ HANDLED!

Each handler tries. If it can't handle → passes to the NEXT in chain.
Request travels the chain until someone handles it (or falls off the end).
```

---

## 💻 Java Implementation

```java
// ⛓️ Abstract Handler
public abstract class SupportHandler {
    private SupportHandler next; // Link to next in chain
    
    public SupportHandler setNext(SupportHandler next) {
        this.next = next;
        return next; // For fluent chaining!
    }
    
    public final void handle(Ticket ticket) {
        if (canHandle(ticket)) {
            process(ticket);
        } else if (next != null) {
            next.handle(ticket); // Pass to next!
        } else {
            System.out.println("❌ No handler for: " + ticket.getIssue());
        }
    }
    
    protected abstract boolean canHandle(Ticket ticket);
    protected abstract void process(Ticket ticket);
}

// Concrete Handlers
public class BotHandler extends SupportHandler {
    @Override protected boolean canHandle(Ticket ticket) {
        return ticket.getSeverity() == Severity.LOW;
    }
    @Override protected void process(Ticket ticket) {
        System.out.println("🤖 Bot: Auto-resolved '" + ticket.getIssue() + "'");
    }
}

public class JuniorDevHandler extends SupportHandler {
    @Override protected boolean canHandle(Ticket ticket) {
        return ticket.getSeverity() == Severity.MEDIUM;
    }
    @Override protected void process(Ticket ticket) {
        System.out.println("👨‍💻 Junior: Fixed '" + ticket.getIssue() + "'");
    }
}

public class SeniorDevHandler extends SupportHandler {
    @Override protected boolean canHandle(Ticket ticket) {
        return ticket.getSeverity() == Severity.HIGH || 
               ticket.getSeverity() == Severity.CRITICAL;
    }
    @Override protected void process(Ticket ticket) {
        System.out.println("👩‍🔬 Senior: Resolved critical '" + ticket.getIssue() + "'");
    }
}

// 🎮 Usage — build the chain
SupportHandler chain = new BotHandler();
chain.setNext(new JuniorDevHandler())
     .setNext(new SeniorDevHandler());

chain.handle(new Ticket("Password reset", Severity.LOW));       // 🤖 Bot handles
chain.handle(new Ticket("Slow query", Severity.MEDIUM));        // 👨‍💻 Junior handles
chain.handle(new Ticket("Data corruption", Severity.CRITICAL)); // 👩‍🔬 Senior handles
```

---

## 🌍 Real-World Examples

```java
// 🌐 Servlet Filter Chain — THE classic Chain of Responsibility!
// Auth filter → Logging filter → CORS filter → Servlet

// 🌱 Spring Security Filter Chain
// UsernamePasswordFilter → JWTFilter → ExceptionFilter → Controller

// ☕ java.util.logging — Handler chain (ConsoleHandler → FileHandler)
// 🌐 Middleware in Express.js/Koa — each middleware calls next()
// 📧 Exception handling — try/catch chain up the call stack!
```

---

## ⚡ When to Use

```
✅ Multiple handlers for a request, but which one handles isn't known in advance
✅ Request should be handled by one handler in a sequence
✅ Want to decouple sender from receiver
✅ Handlers can be assembled/changed dynamically

❌ Only ONE handler needed — just call it directly
❌ EVERY handler must process the request — use Decorator instead
```

---

## 🏆 Achievement: Progress: 19/23 patterns ███████████████████░

---

*← [State](./State.md) | [Mediator →](./Mediator.md)*
