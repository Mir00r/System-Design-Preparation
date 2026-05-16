# 🚶 Visitor Pattern: Add Operations Without Modifying Classes! 🎯

> **"The health inspector visits every restaurant. They examine everything, change nothing."**

---

## 🎬 The Story

```
🏥 HOSPITAL ANALOGY:
━━━━━━━━━━━━━━━━━━━
Patients: Doctor, Nurse, Admin (different types)
Visitor 1: 🏥 InsuranceCalculator → calculates insurance for each type differently
Visitor 2: 📋 TaxCalculator → calculates tax for each type differently  
Visitor 3: 🎉 BonusCalculator → calculates bonus for each type differently

WITHOUT Visitor: Add calculateInsurance(), calculateTax(), calculateBonus() 
                 to EVERY employee class! 💀 (Violates SRP + OCP)

WITH Visitor: Each calculation is a SEPARATE visitor class.
              Employee classes just "accept" visitors.
              Add new calculation? New visitor class! 
              ZERO changes to employee classes! ✅
```

---

## 💻 Java Implementation

```java
// 🧩 Element interface (accepts visitors)
public interface Shape {
    void accept(ShapeVisitor visitor); // Double dispatch!
}

// Concrete Elements
public class Circle implements Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    public double getRadius() { return radius; }
    
    @Override
    public void accept(ShapeVisitor visitor) {
        visitor.visitCircle(this); // "Hey visitor, I'm a Circle!"
    }
}

public class Rectangle implements Shape {
    private final double width, height;
    public Rectangle(double w, double h) { this.width = w; this.height = h; }
    public double getWidth() { return width; }
    public double getHeight() { return height; }
    
    @Override
    public void accept(ShapeVisitor visitor) {
        visitor.visitRectangle(this); // "Hey visitor, I'm a Rectangle!"
    }
}

// 🚶 Visitor interface
public interface ShapeVisitor {
    void visitCircle(Circle circle);
    void visitRectangle(Rectangle rectangle);
}

// Concrete Visitors — each is a NEW operation!
public class AreaCalculator implements ShapeVisitor {
    private double totalArea = 0;
    
    @Override
    public void visitCircle(Circle c) {
        double area = Math.PI * c.getRadius() * c.getRadius();
        totalArea += area;
        System.out.printf("⭕ Circle area: %.2f%n", area);
    }
    
    @Override
    public void visitRectangle(Rectangle r) {
        double area = r.getWidth() * r.getHeight();
        totalArea += area;
        System.out.printf("🟦 Rectangle area: %.2f%n", area);
    }
    
    public double getTotalArea() { return totalArea; }
}

public class DrawingExporter implements ShapeVisitor {
    private final StringBuilder svg = new StringBuilder();
    
    @Override
    public void visitCircle(Circle c) {
        svg.append(String.format("<circle r=\"%f\"/>%n", c.getRadius()));
    }
    
    @Override
    public void visitRectangle(Rectangle r) {
        svg.append(String.format("<rect w=\"%f\" h=\"%f\"/>%n", r.getWidth(), r.getHeight()));
    }
    
    public String getSVG() { return svg.toString(); }
}

// 🎮 Usage
List<Shape> shapes = List.of(new Circle(5), new Rectangle(3, 4), new Circle(2));

AreaCalculator areaCalc = new AreaCalculator();
shapes.forEach(s -> s.accept(areaCalc));
System.out.println("Total area: " + areaCalc.getTotalArea());

DrawingExporter exporter = new DrawingExporter();
shapes.forEach(s -> s.accept(exporter));
System.out.println("SVG:\n" + exporter.getSVG());

// 🎉 Adding "PerimeterCalculator"? New visitor class! ZERO changes to shapes!
```

---

## 🌍 Real-World Examples

```java
// ☕ Java Compiler — AST visitors (process different node types)
// 🌱 Spring — BeanDefinitionVisitor
// 📊 File system traversal — visitor processes different file types differently
// 🧮 Math expression evaluators — visitor calculates/prints/simplifies
// 📋 Report generators — visit data structures, export to PDF/CSV/JSON
```

---

## ⚡ When to Use

```
✅ Need to add operations to a class hierarchy WITHOUT modifying it
✅ Many unrelated operations on an object structure
✅ Object structure rarely changes, but operations change often
✅ Clean separation of algorithms from data structures

❌ Object structure changes frequently (adding new element type = modify ALL visitors!)
❌ Few operations — simpler to just add methods
❌ Breaks encapsulation (visitor needs access to element internals)
```

---

## 🎯 Key: The "Double Dispatch" trick

```
In Java, method called depends on RECEIVER type (single dispatch).
Visitor achieves DOUBLE dispatch:
1. element.accept(visitor) → dispatches on element type
2. visitor.visitCircle(this) → dispatches on visitor type
Result: correct method for BOTH element AND visitor types!
```

---

## 🏆 Achievement: Progress: 22/23 patterns ██████████████████████░

---

*← [Memento](./Memento.md) | [Interpreter →](./Interpreter.md)*
