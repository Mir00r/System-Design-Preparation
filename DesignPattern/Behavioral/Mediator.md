# 📡 Mediator Pattern: Central Hub for Communication! 🎯

> **"Pilots don't talk to each other. They ALL talk to the control tower."**

---

## 🎬 The Story

```
WITHOUT MEDIATOR (Everyone talks to everyone 💀):
Pilot A ↔ Pilot B ↔ Pilot C ↔ Pilot D
   ↕         ↕         ↕
Pilot E ↔ Pilot F ↔ Pilot G
n(n-1)/2 connections = CHAOS! ✈️💥✈️

WITH MEDIATOR (Control Tower 🏢):
Pilot A ─┐
Pilot B ─┤
Pilot C ─┼── 🏢 CONTROL TOWER ── coordinates all
Pilot D ─┤
Pilot E ─┘
n connections = ORDER! ✅
```

---

## 💻 Java Implementation

```java
// 📡 Mediator interface
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

// 🏢 Concrete Mediator
public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();
    
    @Override
    public void addUser(User user) { users.add(user); }
    
    @Override
    public void sendMessage(String message, User sender) {
        for (User user : users) {
            if (user != sender) { // Don't send to yourself!
                user.receive(message, sender.getName());
            }
        }
    }
}

// 👤 Colleague (Component)
public class User {
    private final String name;
    private final ChatMediator mediator;
    
    public User(String name, ChatMediator mediator) {
        this.name = name;
        this.mediator = mediator;
        mediator.addUser(this);
    }
    
    public void send(String message) {
        System.out.println(name + " sends: " + message);
        mediator.sendMessage(message, this); // Goes through mediator!
    }
    
    public void receive(String message, String from) {
        System.out.println(name + " received from " + from + ": " + message);
    }
    
    public String getName() { return name; }
}

// 🎮 Usage
ChatMediator chatRoom = new ChatRoom();
User alice = new User("Alice", chatRoom);
User bob = new User("Bob", chatRoom);
User charlie = new User("Charlie", chatRoom);

alice.send("Hi everyone!"); // Bob and Charlie receive it
// Alice doesn't know about Bob or Charlie — only the mediator does!
```

---

## 🌍 Real-World Examples

```java
// 🌱 Spring's ApplicationContext — mediates between beans!
// 📱 Android's Activity — mediates between Fragments
// ✈️ Air Traffic Control — mediates between aircraft
// 💬 Message Brokers (Kafka, RabbitMQ) = distributed mediators
// 🖥️ MediatR (.NET) / similar in Spring — CQRS mediator pattern

// ☕ Java's java.util.Timer — mediates between TimerTasks
// 🌐 Event Bus libraries (Guava EventBus, GreenRobot EventBus)
```

---

## ⚡ When to Use

```
✅ Many objects communicate in complex ways → centralize communication
✅ Objects are tightly coupled → reduce dependencies
✅ Want to reuse components without their dependencies on others
✅ Behavior should be customizable by changing the mediator

❌ Few components with simple communication → overkill
❌ Mediator becomes God Object → anti-pattern! Keep it thin
```

---

## 🎯 Key Interview Point

**Mediator vs Observer:**
- **Observer**: One-to-many broadcast (one publishes, many listen)
- **Mediator**: Many-to-many coordination (all go through central hub)
- Observer = YouTube subscriber. Mediator = Air traffic control.

---

## 🏆 Achievement: Progress: 20/23 patterns ████████████████████░

---

*← [Chain of Responsibility](./Chain_of_Responsibility.md) | [Memento →](./Memento.md)*
