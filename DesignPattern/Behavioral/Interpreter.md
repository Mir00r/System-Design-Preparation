# 🔤 Interpreter Pattern: Build a Mini Language! 🎯

> **"SQL, regex, math expressions — they all need interpreters to understand them."**

---

## 🎬 The Story

```
🧮 MATH EXPRESSION PARSER:
━━━━━━━━━━━━━━━━━━━━━━━━━
Input: "3 + 5 * 2"

How does a computer understand this?
Break it into a TREE of expressions:

        [+]
       /   \
     [3]   [*]
           /  \
         [5]  [2]

Each node "interprets" itself:
- NumberExpr(3).interpret() → 3
- MultiplyExpr(5,2).interpret() → 10
- AddExpr(3,10).interpret() → 13 ✅
```

---

## 💻 Java Implementation

```java
// 🔤 Abstract Expression
public interface Expression {
    int interpret(Map<String, Integer> context); // Context holds variables
}

// Terminal Expressions (leaf nodes)
public class NumberExpression implements Expression {
    private final int number;
    public NumberExpression(int number) { this.number = number; }
    
    @Override
    public int interpret(Map<String, Integer> context) { return number; }
}

public class VariableExpression implements Expression {
    private final String name;
    public VariableExpression(String name) { this.name = name; }
    
    @Override
    public int interpret(Map<String, Integer> context) {
        if (!context.containsKey(name)) 
            throw new IllegalArgumentException("Unknown variable: " + name);
        return context.get(name);
    }
}

// Non-Terminal Expressions (composite nodes)
public class AddExpression implements Expression {
    private final Expression left, right;
    public AddExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int interpret(Map<String, Integer> context) {
        return left.interpret(context) + right.interpret(context);
    }
}

public class MultiplyExpression implements Expression {
    private final Expression left, right;
    public MultiplyExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }
    
    @Override
    public int interpret(Map<String, Integer> context) {
        return left.interpret(context) * right.interpret(context);
    }
}

// 🎮 Usage — Build expression tree for "x + 5 * 2"
Expression expr = new AddExpression(
    new VariableExpression("x"),
    new MultiplyExpression(
        new NumberExpression(5),
        new NumberExpression(2)
    )
);

Map<String, Integer> context = Map.of("x", 3);
System.out.println(expr.interpret(context)); // 3 + 10 = 13 ✅

// Change x without rebuilding the expression tree!
context = Map.of("x", 100);
System.out.println(expr.interpret(context)); // 100 + 10 = 110 ✅
```

---

## 🌍 Real-World Examples

```java
// ☕ java.util.regex.Pattern — interprets regex grammar
// 🌱 Spring Expression Language (SpEL) — #{expression}
// 📊 SQL parsers — SELECT/WHERE/JOIN are expression nodes
// 🧮 Calculator apps — math expression trees
// 📋 Rule engines (Drools) — interpret business rules
// 🌐 Template engines (Thymeleaf, FreeMarker) — interpret template syntax
```

---

## ⚡ When to Use

```
✅ Simple grammar/language that can be represented as a tree
✅ Efficiency is NOT critical (tree walking is slow for complex grammars)
✅ Grammar doesn't change often
✅ DSLs (Domain-Specific Languages) with limited scope

❌ Complex grammars → use parser generators (ANTLR, JavaCC) instead!
❌ Performance-critical parsing → compile to bytecode instead
❌ Grammar changes frequently → maintenance nightmare
```

---

## 🎯 Key Interview Point

**Interpreter vs Visitor:**
- **Interpreter**: Each node interprets ITSELF (behavior inside the node)
- **Visitor**: External visitor interprets nodes (behavior outside the node)
- Same tree, different "who does the work" philosophy!

**In Practice**: Interpreter is rare in interviews. Know it exists for DSLs, but Strategy/Command/Observer are 100x more common.

---

## 🏆 Achievement: 🎊 ALL 23 PATTERNS COMPLETE! Progress: 23/23 ███████████████████████

```
🏅 GRAND MASTER OF DESIGN PATTERNS!
You've mastered ALL Gang of Four patterns:
✅ 5 Creational  | ✅ 7 Structural  | ✅ 11 Behavioral
```

---

*← [Visitor](./Visitor.md) | [🏠 Back to Main](../README.md)*
