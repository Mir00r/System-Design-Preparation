# 🌉 Bridge Pattern: Separate What from How! 🎯

> **"Decouple an abstraction from its implementation so they can vary independently."**

---

## 🎬 The Story

### 📱 Remote Controls & Devices

```
WITHOUT BRIDGE (Class Explosion! 💥):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BasicRemote + TV      = BasicTVRemote
BasicRemote + Radio   = BasicRadioRemote  
AdvancedRemote + TV   = AdvancedTVRemote
AdvancedRemote + Radio = AdvancedRadioRemote
// 2 remotes × 2 devices = 4 classes 😰
// 3 remotes × 5 devices = 15 classes! 💀

WITH BRIDGE (Elegant! ✅):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Remote (abstraction) ──bridge──▶ Device (implementation)
  ├── BasicRemote                    ├── TV
  └── AdvancedRemote                 ├── Radio
                                     └── Speaker
// 2 remotes + 3 devices = 5 classes! 🎉
// Vary INDEPENDENTLY!
```

---

## 💡 The Solution

```
BRIDGE separates two dimensions of variation:

DIMENSION 1 (Abstraction): WHAT to do → Remote control features
DIMENSION 2 (Implementation): HOW to do it → Device-specific behavior

These two can change independently without affecting each other!
```

---

## 🏗️ The Structure

```
┌────────────────────┐         ┌─────────────────────────┐
│   Abstraction       │ bridge  │   Implementation (intf)  │
│ ────────────────── │────────▶│ ─────────────────────── │
│ - impl: Implementor│         │ + operationImpl()        │
│ + operation()       │         └───────────┬─────────────┘
│   → impl.opImpl()  │                     │ implements
└────────┬───────────┘              ┌───────┴───────┐
         │ extends                  │               │
┌────────▼───────────┐    ┌────────▼──────┐ ┌──────▼────────┐
│ RefinedAbstraction  │    │ ConcreteImplA │ │ ConcreteImplB │
│ + extraOperation()  │    └───────────────┘ └───────────────┘
└────────────────────┘
```

---

## 💻 Java Implementation

```java
// 🖥️ Implementation interface (HOW — the device)
public interface Device {
    void turnOn();
    void turnOff();
    void setVolume(int volume);
    int getVolume();
    void setChannel(int channel);
}

// Concrete implementations
public class TV implements Device {
    private int volume = 50;
    private int channel = 1;
    
    public void turnOn()  { System.out.println("📺 TV is ON"); }
    public void turnOff() { System.out.println("📺 TV is OFF"); }
    public void setVolume(int v) { this.volume = v; System.out.println("📺 Volume: " + v); }
    public int getVolume() { return volume; }
    public void setChannel(int ch) { this.channel = ch; System.out.println("📺 Channel: " + ch); }
}

public class Radio implements Device {
    private int volume = 30;
    private int channel = 1;
    
    public void turnOn()  { System.out.println("📻 Radio is ON"); }
    public void turnOff() { System.out.println("📻 Radio is OFF"); }
    public void setVolume(int v) { this.volume = v; System.out.println("📻 Volume: " + v); }
    public int getVolume() { return volume; }
    public void setChannel(int ch) { this.channel = ch; System.out.println("📻 FM: " + (87.5 + ch * 0.5)); }
}

// 🎮 Abstraction (WHAT — the remote)
public abstract class Remote {
    protected Device device; // ← THE BRIDGE!
    
    public Remote(Device device) { this.device = device; }
    
    public void togglePower() { device.turnOn(); }
    public void volumeUp()    { device.setVolume(device.getVolume() + 10); }
    public void volumeDown()  { device.setVolume(device.getVolume() - 10); }
    public void channelUp()   { device.setChannel(1); }
}

// Refined Abstraction
public class AdvancedRemote extends Remote {
    public AdvancedRemote(Device device) { super(device); }
    
    public void mute() { device.setVolume(0); System.out.println("🔇 Muted!"); }
    public void partyMode() { device.setVolume(100); System.out.println("🎉 PARTY!"); }
}

// 🎮 Usage — mix ANY remote with ANY device!
Remote tvRemote = new AdvancedRemote(new TV());
tvRemote.togglePower();   // 📺 TV is ON
((AdvancedRemote) tvRemote).partyMode(); // 📺 Volume: 100, 🎉 PARTY!

Remote radioRemote = new Remote(new Radio()) {};
radioRemote.volumeUp();   // 📻 Volume: 40
```

---

## ⚡ When to Use

```
✅ Two independent dimensions of variation (shape × color, platform × feature)
✅ Avoid permanent binding between abstraction and implementation
✅ Both sides should be extensible independently
✅ Prevent class explosion from cartesian product of variants

❌ Only ONE dimension varies → simpler patterns suffice
❌ Makes code more complex for simple hierarchies
```

---

## 🎯 Key Interview Point

**Bridge vs Adapter:**
- **Adapter** = retrofit (after design) — make existing things work together
- **Bridge** = up-front design — prevent coupling before it happens

---

## 🏆 Achievement Unlocked!

```
Progress: 7/23 patterns ███████░░ 
```

---

*← [Adapter](./Adapter.md) | [Composite →](./Composite.md)*
