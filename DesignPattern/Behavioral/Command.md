# 📜 Command Pattern: Encapsulate Requests as Objects! 🎯

> **"A waiter doesn't cook your food. They write your order on a ticket and pass it to the kitchen."**

---

## 🎬 The Story

```
🍽️ RESTAURANT ORDER SYSTEM:
━━━━━━━━━━━━━━━━━━━━━━━━━━━
Customer says: "I want a burger"
Waiter WRITES it on a ticket (Command object!)
Ticket goes to kitchen (Invoker executes Command)
Chef reads ticket, cooks (Receiver does the work)

Benefits of the ticket:
✅ Can queue orders (first come, first served)
✅ Can cancel/undo orders (tear up ticket)
✅ Can log all orders (for billing/audit)
✅ Can retry failed orders (remake if dropped)
✅ Waiter doesn't need to know HOW to cook!
```

---

## 💻 Java Implementation

```java
// 📜 Command interface
public interface Command {
    void execute();
    void undo(); // Optional but powerful!
}

// 🎯 Receiver (does the actual work)
public class TextEditor {
    private StringBuilder content = new StringBuilder();
    
    public void insertText(String text, int position) {
        content.insert(position, text);
        System.out.println("📝 Inserted '" + text + "' → " + content);
    }
    
    public void deleteText(int position, int length) {
        content.delete(position, position + length);
        System.out.println("🗑️ Deleted " + length + " chars → " + content);
    }
    
    public String getContent() { return content.toString(); }
}

// 📜 Concrete Commands
public class InsertCommand implements Command {
    private final TextEditor editor;
    private final String text;
    private final int position;
    
    public InsertCommand(TextEditor editor, String text, int position) {
        this.editor = editor;
        this.text = text;
        this.position = position;
    }
    
    @Override public void execute() { editor.insertText(text, position); }
    @Override public void undo()    { editor.deleteText(position, text.length()); }
}

public class DeleteCommand implements Command {
    private final TextEditor editor;
    private final int position;
    private final int length;
    private String deletedText; // Save for undo!
    
    public DeleteCommand(TextEditor editor, int position, int length) {
        this.editor = editor;
        this.position = position;
        this.length = length;
    }
    
    @Override
    public void execute() {
        deletedText = editor.getContent().substring(position, position + length);
        editor.deleteText(position, length);
    }
    
    @Override
    public void undo() { editor.insertText(deletedText, position); }
}

// 🎮 Invoker — manages command execution and history
public class CommandHistory {
    private final Deque<Command> history = new ArrayDeque<>();
    private final Deque<Command> redoStack = new ArrayDeque<>();
    
    public void executeCommand(Command cmd) {
        cmd.execute();
        history.push(cmd);
        redoStack.clear(); // New action clears redo
    }
    
    public void undo() {
        if (!history.isEmpty()) {
            Command cmd = history.pop();
            cmd.undo();
            redoStack.push(cmd);
            System.out.println("↩️ Undo!");
        }
    }
    
    public void redo() {
        if (!redoStack.isEmpty()) {
            Command cmd = redoStack.pop();
            cmd.execute();
            history.push(cmd);
            System.out.println("↪️ Redo!");
        }
    }
}

// 🎮 Usage
TextEditor editor = new TextEditor();
CommandHistory history = new CommandHistory();

history.executeCommand(new InsertCommand(editor, "Hello ", 0));  // "Hello "
history.executeCommand(new InsertCommand(editor, "World!", 6));  // "Hello World!"
history.undo();  // "Hello " ← World removed!
history.redo();  // "Hello World!" ← World back!
```

---

## 🌍 Real-World Examples

```java
// ☕ Java's Runnable IS a Command!
Runnable cmd = () -> System.out.println("Execute!");
new Thread(cmd).start(); // Encapsulated action!

// 🌱 Spring's @Scheduled tasks, Spring Batch Steps
// 📱 Android onClick listeners = Command objects
// 🎮 Game: action queue (move, attack, spell = command objects)
// 💰 Banking: Transaction objects (execute/rollback)
// 📋 Task queues: RabbitMQ messages = serialized commands
```

---

## ⚡ When to Use

```
✅ Need undo/redo functionality
✅ Queue and schedule operations  
✅ Log and audit all operations
✅ Decouple invoker from receiver (sender from handler)
✅ Parameterize objects with operations (callbacks)

❌ Simple method calls that don't need history/undo
❌ Command adds complexity — don't use for trivial operations
```

---

## 🎯 Key Interview Point

**Command vs Strategy:**
- **Strategy**: Swap HOW something is done (different algorithms for same goal)
- **Command**: Wrap WHAT to do as an object (any operation becomes data)
- Strategy = many ways to sort. Command = any action as an object (undo, queue, log)

---

## 🏆 Achievement: Progress: 15/23 patterns ███████████████░

---

*← [Observer](./Observer.md) | [Template Method →](./Template_Method.md)*
