# 💾 Memento Pattern: Save and Restore State! 🎯

> **"Ctrl+Z exists because of this pattern. Save state, restore when needed."**

---

## 🎬 The Story

```
🎮 GAME SAVE SYSTEM:
━━━━━━━━━━━━━━━━━━━
Playing Level 5 with 95% health, 3 power-ups, score 5000...
💾 SAVE GAME (create Memento!)

Oh no! Fell into lava! Health = 0! 💀

↩️ LOAD SAVE (restore Memento!)
Back to 95% health, 3 power-ups, score 5000! 🎉

Memento = a SNAPSHOT of object's state at a point in time.
```

---

## 💻 Java Implementation

```java
// 💾 Memento — stores state snapshot (immutable!)
public class EditorMemento {
    private final String content;
    private final int cursorPosition;
    private final String font;
    
    public EditorMemento(String content, int cursorPosition, String font) {
        this.content = content;
        this.cursorPosition = cursorPosition;
        this.font = font;
    }
    
    // Package-private getters — only Originator should access!
    String getContent() { return content; }
    int getCursorPosition() { return cursorPosition; }
    String getFont() { return font; }
}

// 📝 Originator — creates and restores from mementos
public class TextEditor {
    private String content = "";
    private int cursorPosition = 0;
    private String font = "Arial";
    
    public void type(String text) {
        content += text;
        cursorPosition = content.length();
    }
    
    public void setFont(String font) { this.font = font; }
    
    // 💾 Create snapshot
    public EditorMemento save() {
        return new EditorMemento(content, cursorPosition, font);
    }
    
    // ↩️ Restore from snapshot
    public void restore(EditorMemento memento) {
        this.content = memento.getContent();
        this.cursorPosition = memento.getCursorPosition();
        this.font = memento.getFont();
    }
    
    @Override
    public String toString() {
        return String.format("Editor[content='%s', cursor=%d, font=%s]", 
                            content, cursorPosition, font);
    }
}

// 📚 Caretaker — manages memento history (doesn't peek inside!)
public class History {
    private final Deque<EditorMemento> states = new ArrayDeque<>();
    
    public void push(EditorMemento memento) { states.push(memento); }
    
    public EditorMemento pop() {
        if (states.isEmpty()) throw new IllegalStateException("Nothing to undo!");
        return states.pop();
    }
    
    public boolean canUndo() { return !states.isEmpty(); }
}

// 🎮 Usage
TextEditor editor = new TextEditor();
History history = new History();

editor.type("Hello");
history.push(editor.save());  // 💾 Save state 1

editor.type(" World");
history.push(editor.save());  // 💾 Save state 2

editor.type("!!!");
System.out.println(editor);   // Editor[content='Hello World!!!', ...]

editor.restore(history.pop()); // ↩️ Undo!
System.out.println(editor);    // Editor[content='Hello World', ...]

editor.restore(history.pop()); // ↩️ Undo!
System.out.println(editor);    // Editor[content='Hello', ...]
```

---

## 🌍 Real-World Examples

```java
// ☕ java.io.Serializable — save/restore entire object graph!
// 🗄️ Database transactions — savepoints = mementos!
// 🎮 Game saves — serialize game state
// 🌐 Browser history — each page = a memento
// 📝 Git commits — each commit = project state memento!
// 🌱 Spring's @Transactional with savepoints
```

---

## ⚡ When to Use

```
✅ Need undo/redo functionality
✅ Need to create snapshots/checkpoints
✅ Want to restore previous state without exposing internals
✅ Transaction rollback scenarios

❌ Mementos are too large (memory-heavy objects)
❌ State is simple — just store a copy of the field
❌ Frequent snapshots of large objects → memory issues
```

---

## 🎯 Key: Memento vs Command (for Undo)

- **Memento**: Stores full state snapshot → restore to that point
- **Command**: Stores operations → undo by running inverse operation
- Memento = Photo album (exact state). Command = recipe + anti-recipe (do/undo steps)

---

## 🏆 Achievement: Progress: 21/23 patterns █████████████████████░

---

*← [Mediator](./Mediator.md) | [Visitor →](./Visitor.md)*
