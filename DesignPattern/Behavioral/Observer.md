# 👀 Observer Pattern: Notify Me When Something Changes! 🎯

> **"Don't call us, we'll call you — when something interesting happens."**

---

## 🎬 The Story

### 📰 Newspaper Subscription

```
WITHOUT OBSERVER:
Every morning you drive to the news office: "Any new paper?" 
"No." Drive home. Next morning: "Any new paper?" "No." 😤
(This is POLLING — wasteful!)

WITH OBSERVER:
Subscribe to newspaper delivery. Publisher prints new edition → 
automatically delivered to ALL subscribers. 📨
You don't check. You get NOTIFIED. ✅
```

---

## 🏗️ The Structure

```
┌────────────────────────┐         ┌───────────────────────┐
│  Subject (Observable)   │         │  Observer (Interface)  │
│ ─────────────────────── │    *    │ ───────────────────── │
│ - observers: List       │◄────────│ + update(data)         │
│ + subscribe(observer)   │         └──────────┬────────────┘
│ + unsubscribe(observer) │                    │ implements
│ + notifyAll()           │           ┌────────┼────────┐
│   → for each observer   │           │        │        │
│     observer.update()   │       EmailAlert  SMS    Dashboard
└────────────────────────┘
```

---

## 💻 Java Implementation

```java
// 👀 Observer interface
public interface EventListener {
    void update(String eventType, Object data);
}

// 📡 Subject (Publisher)
public class EventManager {
    private final Map<String, List<EventListener>> listeners = new HashMap<>();
    
    public EventManager(String... eventTypes) {
        for (String type : eventTypes) {
            listeners.put(type, new ArrayList<>());
        }
    }
    
    public void subscribe(String eventType, EventListener listener) {
        listeners.get(eventType).add(listener);
    }
    
    public void unsubscribe(String eventType, EventListener listener) {
        listeners.get(eventType).remove(listener);
    }
    
    public void notify(String eventType, Object data) {
        for (EventListener listener : listeners.get(eventType)) {
            listener.update(eventType, data);
        }
    }
}

// 📝 Concrete Subject
public class Store {
    private final EventManager events;
    
    public Store() {
        this.events = new EventManager("newProduct", "sale");
    }
    
    public void addProduct(String product) {
        System.out.println("🏪 New product added: " + product);
        events.notify("newProduct", product);
    }
    
    public void startSale(int discount) {
        System.out.println("🏪 Sale started: " + discount + "% off!");
        events.notify("sale", discount);
    }
    
    public EventManager getEvents() { return events; }
}

// 👂 Concrete Observers
public class EmailAlertListener implements EventListener {
    private final String email;
    public EmailAlertListener(String email) { this.email = email; }
    
    @Override
    public void update(String eventType, Object data) {
        System.out.printf("📧 Email to %s: [%s] → %s%n", email, eventType, data);
    }
}

public class SMSListener implements EventListener {
    private final String phone;
    public SMSListener(String phone) { this.phone = phone; }
    
    @Override
    public void update(String eventType, Object data) {
        System.out.printf("📱 SMS to %s: [%s] → %s%n", phone, eventType, data);
    }
}

// 🎮 Usage
Store store = new Store();
store.getEvents().subscribe("newProduct", new EmailAlertListener("user@email.com"));
store.getEvents().subscribe("sale", new SMSListener("+1-555-0123"));
store.getEvents().subscribe("sale", new EmailAlertListener("deals@email.com"));

store.addProduct("iPhone 16"); // → Email sent to user@email.com
store.startSale(30);           // → SMS to phone + Email to deals@email.com
```

---

## 🌍 Real-World Examples

```java
// ☕ Java's PropertyChangeListener
bean.addPropertyChangeListener("name", evt -> {
    System.out.println("Name changed: " + evt.getOldValue() + " → " + evt.getNewValue());
});

// 🌱 Spring Application Events
@EventListener
public void handleUserCreated(UserCreatedEvent event) {
    sendWelcomeEmail(event.getUser()); // Automatically notified!
}

// ⚛️ React/Angular — state changes notify UI components
// 📱 Android LiveData — observers update UI automatically
// 📈 Kafka/RabbitMQ — pub/sub at system level (distributed Observer)
```

---

## ⚡ When to Use

```
✅ One-to-many dependency (one change → many reactions)
✅ Decoupled notification (publisher doesn't know specific subscribers)
✅ Event-driven architectures
✅ UI frameworks (model changes → view updates)

❌ Simple notification to ONE known listener — direct call is fine
❌ Order of notification matters — Observer doesn't guarantee order
❌ Circular dependencies between observers → infinite loops! 💀
```

---

## 🎯 Interview Key

**Observer vs Pub/Sub:**
- Observer: subject KNOWS its observers (direct reference)
- Pub/Sub: message broker in between (fully decoupled, no direct reference)

---

## 🏆 Achievement: Progress: 14/23 patterns ██████████████░

---

*← [Strategy](./Strategy.md) | [Command →](./Command.md)*
