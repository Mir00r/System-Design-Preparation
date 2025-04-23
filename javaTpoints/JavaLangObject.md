# 💡 Mastering `java.lang.Object` in Java — The Root of All Classes

When preparing for Java interviews, one of the most frequently overlooked yet fundamentally crucial topics is the `java.lang.Object` class. Understanding it in-depth not only helps in interviews but also enhances your understanding of Java’s object-oriented nature.

In this blog, we’ll explore:
- What is `Object` class?
- Its key methods (with in-depth explanations)
- Industry examples and real-world usage
- Best practices
- Java code snippets
- A handy summary table

---

## 🧱 What is `java.lang.Object`?

`Object` is the root class of the Java class hierarchy. Every class in Java directly or indirectly inherits from `Object`. This means every Java object has the methods defined in `Object`, making it a powerful abstraction base.

### 🔗 Why is this important?
- It provides a common protocol for all Java objects.
- Many of its methods are overridden to implement custom behaviors like comparison, hashing, and representation.

---

## ⚙️ Methods in `java.lang.Object`

Let’s deep dive into the major methods provided by the Object class and why they matter.

### 1. `Object()` - The Constructor
This is the default constructor used when no superclass constructor is explicitly called.

```java
public class MyClass extends Object {
    public MyClass() {
        super();  // This is implicitly called
    }
}
```

### 2. `public boolean equals(Object obj)`
Used to compare two objects for equality.

#### 🔍 Default Behavior:
Compares memory references.

#### ✅ Best Practice:
Override in custom classes to compare fields.

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    MyClass myClass = (MyClass) o;
    return this.id == myClass.id;
}
```

### 3. `public int hashCode()`
Returns a hash value for the object, essential for hash-based collections like `HashMap` or `HashSet`.

#### ✅ Best Practice:
Always override `hashCode()` when overriding `equals()` to maintain the general contract:
> Equal objects must have equal hash codes.

```java
@Override
public int hashCode() {
    return Objects.hash(id);
}
```

### 4. `public String toString()`
Returns a string representation of the object.

#### 🔍 Default Output:
`ClassName@HashCodeHex`

#### ✅ Best Practice:
Override to give meaningful information.

```java
@Override
public String toString() {
    return "MyClass{id=" + id + ", name=" + name + "}";
}
```

### 5. `protected Object clone() throws CloneNotSupportedException`
Creates and returns a copy of the object.

#### 🔐 Default Behavior:
Performs a shallow copy.

#### ⚠️ Best Practice:
- Implement `Cloneable` interface.
- Deep copy if needed.

```java
@Override
protected Object clone() throws CloneNotSupportedException {
    return super.clone();
}
```

> 🔄 Many companies prefer copy constructors or serialization-based cloning due to `clone()` being error-prone.

### 6. `protected void finalize() throws Throwable` (Deprecated since Java 9)
Called by the garbage collector before reclaiming memory.

#### ❌ Why Avoid?
- Unpredictable.
- Deprecated due to performance and reliability issues.

---

## 🏢 How Big Companies Use It

- **Amazon / Google**: Custom implementations of `equals()` and `hashCode()` for DTOs (Data Transfer Objects).
- **LinkedIn / Netflix**: Custom `toString()` in logs to improve traceability in distributed systems.
- **Uber / Twitter**: Prefer `Copy Constructors` or libraries like Apache Commons `SerializationUtils.clone()` instead of `clone()`.

---

## 📊 Summary Table

| Method            | Purpose                          | Best Practice                         |
|-------------------|----------------------------------|----------------------------------------|
| `equals()`        | Logical equality                 | Compare relevant fields                |
| `hashCode()`      | Hash-based collections           | Consistent with `equals()`             |
| `toString()`      | Object description               | Return meaningful representation       |
| `clone()`         | Copy of object                   | Use copy constructors or deep copy     |
| `finalize()`      | Cleanup before GC (deprecated)   | Avoid usage                            |

---

## ✅ Interview Tips

- Always mention that `equals()` and `hashCode()` must be overridden together.
- Be ready to explain shallow vs. deep copy with `clone()`.
- Know why `finalize()` is deprecated and suggest alternatives (e.g., `try-with-resources` for cleanup).
- Use examples from real-world applications (e.g., `HashMap` key collisions).

---

## 💻 Code Playground

```java
public class Employee {
    private int id;
    private String name;

    // Constructors, Getters

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee that = (Employee) o;
        return id == that.id && Objects.equals(name, that.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }

    @Override
    public String toString() {
        return "Employee{id=" + id + ", name='" + name + "'}";
    }
}
```

---

## 🎯 Conclusion

The `Object` class is the silent backbone of every Java application. Understanding its methods helps you write better, cleaner, and more robust Java code — and ace those tricky interview questions!

If you're preparing for Java or system design interviews, mastering this topic is an absolute must.
