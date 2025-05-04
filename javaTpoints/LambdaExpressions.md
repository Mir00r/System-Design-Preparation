# 🚀 **The Ultimate Guide to Java Lambda Expressions: Interview Mastery**

## **🔍 What Are Lambda Expressions?**
Lambda expressions (λ) are **anonymous functions** introduced in Java 8 that enable:
✔ **Concise** functional programming syntax  
✔ **Behavior parameterization** (passing code as data)  
✔ **Simpler** collection operations

🔹 **Analogy**: Like giving a **recipe** (behavior) instead of a **cooked dish** (full class).

---

## **🎯 Why Use Lambdas?**

### **✅ Advantages**
✔ **Reduced boilerplate** (fewer lines of code)  
✔ **Readable** collection operations  
✔ **Enables functional programming** in Java  
✔ **Better APIs** (Streams, CompletableFuture)

### **❌ Disadvantages**
✔ **Debugging challenges** (stack traces less clear)  
✔ **Limited to functional interfaces**  
✔ **Performance overhead** (small, but exists)

🔹 **When NOT to Use?**
- Complex logic needing multiple statements
- Where method references would be clearer

---

## **📜 Lambda Syntax**
```java
(parameters) -> { body }
```
### **Examples**
```java
// 1. No parameters
() -> System.out.println("Hello")

// 2. Single parameter (parentheses optional)
x -> x * 2

// 3. Multiple parameters
(a, b) -> a + b

// 4. Multi-line body
(name) -> {
    String greeting = "Hello " + name;
    System.out.println(greeting);
}
```

---

## **⚙️ Functional Interfaces**
Lambdas work with **single abstract method (SAM)** interfaces:

| Interface       | Method          | Common Use              |
|-----------------|-----------------|-------------------------|
| `Predicate<T>`  | `test(T t)`     | Filtering               |
| `Function<T,R>` | `apply(T t)`    | Transformations         |
| `Consumer<T>`   | `accept(T t)`   | Side-effects            |
| `Supplier<T>`   | `get()`         | Lazy generation         |

🔹 **Example:**
```java
Predicate<String> isLong = s -> s.length() > 10;
Function<String, Integer> toLength = String::length;
Consumer<String> printer = System.out::println;
Supplier<Double> random = Math::random;
```

---

## **🚀 Real-World Use Cases**

### **1. Collections Processing**
```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

// Filter
names.stream()
     .filter(name -> name.startsWith("A"))
     .forEach(System.out::println);

// Map
List<Integer> lengths = names.stream()
                            .map(String::length)
                            .collect(Collectors.toList());
```

### **2. Event Handlers**
```java
button.addActionListener(
    event -> System.out.println("Button clicked!")
);
```

### **3. Threading**
```java
new Thread(() -> {
    System.out.println("Running in thread");
}).start();
```

---

## **📊 Method References**
Shortcut for lambdas calling existing methods:

| Type                  | Syntax                  | Equivalent Lambda         |
|-----------------------|-------------------------|--------------------------|
| Static method         | `Class::method`         | `x -> Class.method(x)`   |
| Instance method       | `object::method`        | `x -> object.method(x)`  |
| Arbitrary object      | `Class::instanceMethod` | `(obj, x) -> obj.method(x)` |
| Constructor           | `Class::new`            | `x -> new Class(x)`      |

🔹 **Example:**
```java
List<String> names = Arrays.asList("a", "b", "c");
names.forEach(System.out::println);  // Equivalent to x -> System.out.println(x)
```

---

## **⚡ Performance Considerations**
- **Small overhead** vs. anonymous classes (~2.5x faster)
- **Non-capturing lambdas** (no external variables) are fastest
- **JIT optimizes** frequently used lambdas

🔹 **Benchmark Tip:**
```java
// Warm up JIT first before measuring!
IntStream.range(0, 1000000)
         .map(x -> x * 2)  // Test this
         .sum();
```

---

## **🏆 Best Practices**
✅ **Keep lambdas short** (1-3 lines ideal)  
✅ **Use method references** where possible  
✅ **Avoid side-effects** in stream operations  
✅ **Type explicitly** when unclear
```java
Function<String, Integer> parser = (String s) -> Integer.parseInt(s);
```

---

## **💡 Interview Q&A**

### **Q1: Lambdas vs Anonymous Classes?**
✅ **Lambdas**: Concise, no `this` ambiguity, better performance  
✅ **Anonymous Classes**: Stateful, multiple methods

### **Q2: Can lambdas throw exceptions?**
✅ Only if functional interface declares it:
```java
Function<String, Integer> risky = s -> {
    if (s.isEmpty()) throw new IllegalArgumentException();
    return s.length();
};
```

### **Q3: How are lambdas compiled?**
✅ As **invokedynamic** bytecode + synthetic methods

### **Q4: Can you replace all loops with lambdas?**
✅ Not always - sometimes loops are clearer for complex logic

---

# 🚀 **Top 20 Java Lambda Expression Coding Interview Questions (+ Solutions)**

## **🔍 Basic Level**

### **1. Sort a list of strings alphabetically using lambda**
```java
List<String> names = Arrays.asList("John", "Alice", "Bob", "Diana");
names.sort((s1, s2) -> s1.compareTo(s2));
// or method reference: names.sort(String::compareTo);
```

### **2. Filter even numbers from a list**
```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6);
List<Integer> evens = numbers.stream()
                            .filter(n -> n % 2 == 0)
                            .collect(Collectors.toList());
```

### **3. Convert list of strings to uppercase**
```java
List<String> words = Arrays.asList("hello", "world");
List<String> upper = words.stream()
                         .map(String::toUpperCase)
                         .collect(Collectors.toList());
```

### **4. Sum all numbers in a list**
```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
int sum = nums.stream().reduce(0, (a, b) -> a + b);
// or: nums.stream().mapToInt(Integer::intValue).sum();
```

### **5. Find max number in a list**
```java
List<Integer> nums = Arrays.asList(5, 3, 8, 2);
Optional<Integer> max = nums.stream().max(Integer::compare);
```

---

## **⚡ Intermediate Level**

### **6. Group employees by department**
```java
List<Employee> employees = //...;
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));
```

### **7. Check if all numbers are positive**
```java
List<Integer> nums = Arrays.asList(1, 2, 3, -4);
boolean allPositive = nums.stream().allMatch(n -> n > 0);
```

### **8. Convert list to map (id -> name)**
```java
List<Employee> employees = //...;
Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toMap(Employee::getId, Employee::getName));
```

### **9. Find first even number**
```java
List<Integer> nums = Arrays.asList(1, 3, 5, 7, 8, 9);
Optional<Integer> firstEven = nums.stream()
                                .filter(n -> n % 2 == 0)
                                .findFirst();
```

### **10. Remove duplicates from list**
```java
List<Integer> nums = Arrays.asList(1, 2, 2, 3, 4, 4);
List<Integer> distinct = nums.stream()
                            .distinct()
                            .collect(Collectors.toList());
```

---

## **🚀 Advanced Level**

### **11. FlatMap example (merge lists)**
```java
List<List<Integer>> listOfLists = Arrays.asList(
    Arrays.asList(1, 2),
    Arrays.asList(3, 4)
);
List<Integer> flat = listOfLists.stream()
                              .flatMap(List::stream)
                              .collect(Collectors.toList());
```

### **12. Custom comparator with lambda**
```java
List<Employee> employees = //...;
employees.sort((e1, e2) -> e1.getAge() - e2.getAge());
// or: employees.sort(Comparator.comparingInt(Employee::getAge));
```

### **13. Parallel stream sum**
```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
int sum = nums.parallelStream().reduce(0, Integer::sum);
```

### **14. Chain multiple predicates**
```java
Predicate<String> startsWithA = s -> s.startsWith("A");
Predicate<String> length5 = s -> s.length() == 5;
List<String> result = names.stream()
                         .filter(startsWithA.and(length5))
                         .collect(Collectors.toList());
```

### **15. Exception handling in lambda**
```java
List<String> numbers = Arrays.asList("1", "2", "abc");
numbers.forEach(s -> {
    try {
        System.out.println(Integer.parseInt(s));
    } catch (NumberFormatException e) {
        System.out.println("Invalid: " + s);
    }
});
```

---

## **💡 Functional Interfaces Deep Dive**

### **16. Custom functional interface**
```java
@FunctionalInterface
interface StringProcessor {
    String process(String input);
}

StringProcessor toUpper = s -> s.toUpperCase();
System.out.println(toUpper.process("hello")); // HELLO
```

### **17. Supplier for lazy initialization**
```java
Supplier<LocalDateTime> timeSupplier = LocalDateTime::now;
System.out.println(timeSupplier.get()));
```

### **18. Consumer with side effects**
```java
Consumer<String> logger = s -> System.out.println("[LOG] " + s);
logger.accept("Application started");
```

### **19. Function composition**
```java
Function<Integer, Integer> doubleIt = x -> x * 2;
Function<Integer, Integer> squareIt = x -> x * x;
Function<Integer, Integer> doubleThenSquare = doubleIt.andThen(squareIt);
System.out.println(doubleThenSquare.apply(3)); // 36 (3*2=6, 6²=36)
```

### **20. BiFunction example**
```java
BiFunction<Integer, Integer, Integer> adder = (a, b) -> a + b;
System.out.println(adder.apply(5, 3)); // 8
```

---

## **🎯 Pro Tips for Interviews**
1. **Method references** are preferred over lambdas when possible
2. **Avoid side-effects** in stream operations
3. **Primitive streams** (IntStream, LongStream) are more efficient
4. **Parallel streams** need thread-safe operations
5. **Practice common patterns** (map-filter-reduce, grouping, partitioning)

**Example Problem:** Given a list of transactions, find the total value of transactions from a specific category:
```java
List<Transaction> transactions = //...;
double total = transactions.stream()
                         .filter(t -> t.getCategory().equals("GROCERIES"))
                         .mapToDouble(Transaction::getAmount)
                         .sum();
``` 

---

## **🎯 Conclusion**
Java lambdas revolutionized how we write:
✔ **Cleaner collection operations**  
✔ **Expressive APIs**  
✔ **Concurrent code**

**🚀 Pro Tip:** Master `Stream`, `Optional`, and `CompletableFuture` APIs to fully leverage lambdas!

#Java #Lambda #FunctionalProgramming #Java8 #CodingInterview

---

# 🚀 **Real-World Lambda Expression Use Cases in Java**

Lambda expressions have revolutionized Java programming by enabling concise functional-style code. Here are practical, real-world applications across different domains:

## **1. Collection Processing (Everyday Use)**
### **Filtering Log Files**
```java
// Find ERROR logs in last 24 hours
List<String> errorLogs = Files.lines(Paths.get("app.log"))
    .filter(line -> line.contains("ERROR"))
    .filter(line -> line.contains(LocalDate.now().toString()))
    .collect(Collectors.toList());
```

### **Data Transformation Pipeline**
```java
// CSV → Object transformation
List<Employee> employees = Files.lines(Paths.get("employees.csv"))
    .skip(1) // Skip header
    .map(line -> {
        String[] parts = line.split(",");
        return new Employee(parts[0], Integer.parseInt(parts[1]), parts[2]);
    })
    .collect(Collectors.toList());
```

## **2. GUI/Web Development**
### **Swing Event Handling**
```java
JButton saveButton = new JButton("Save");
saveButton.addActionListener(e -> {
    try {
        saveToDatabase();
        showSuccessMessage();
    } catch (Exception ex) {
        logger.error("Save failed", ex);
    }
});
```

### **Spring Web Controllers**
```java
@GetMapping("/products")
public ResponseEntity<List<Product>> getFilteredProducts(
    @RequestParam(required = false) Predicate<Product> filter) {
    
    return productService.getAllProducts()
        .stream()
        .filter(filter != null ? filter : p -> true)
        .collect(Collectors.toList());
}
```

## **3. Concurrent Programming**
### **Parallel Data Processing**
```java
// Process images in parallel
List<File> images = getImageFiles();
images.parallelStream()
    .forEach(image -> {
        BufferedImage processed = applyFilters(image);
        saveProcessedImage(processed);
    });
```

### **Scheduled Tasks**
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);
scheduler.scheduleAtFixedRate(
    () -> System.out.println("Heartbeat at " + LocalTime.now()),
    0, 1, TimeUnit.SECONDS
);
```

## **4. Data Analysis & Reporting**
### **Sales Report Generation**
```java
Map<String, Double> salesByRegion = transactions.stream()
    .collect(Collectors.groupingBy(
        Transaction::getRegion,
        Collectors.summingDouble(Transaction::getAmount)
    ));
```

### **Anomaly Detection**
```java
double average = sensorReadings.stream()
    .mapToDouble(Double::doubleValue)
    .average()
    .orElse(0.0);

List<Double> anomalies = sensorReadings.stream()
    .filter(value -> Math.abs(value - average) > 2 * standardDeviation)
    .collect(Collectors.toList());
```

## **5. API Development**
### **Dynamic API Response Transformation**
```java
public <T, R> ResponseEntity<R> getResource(
    String id, 
    Function<T, R> transformer) {
    
    T rawData = dataService.getById(id);
    return ResponseEntity.ok(transformer.apply(rawData));
}

// Usage: 
getResource("123", user -> new UserDTO(user.getName(), user.getEmail()));
```

### **Validation Chain**
```java
List<Predicate<FormData>> validations = Arrays.asList(
    data -> !data.getUsername().isEmpty(),
    data -> data.getPassword().length() >= 8,
    data -> data.getEmail().contains("@")
);

boolean isValid = validations.stream()
    .allMatch(predicate -> predicate.test(formData));
```

## **6. File/IO Operations**
### **Directory Cleanup**
```java
Files.list(Paths.get("/tmp"))
    .filter(path -> Files.isRegularFile(path))
    .filter(path -> {
        try {
            return Files.getLastModifiedTime(path).toInstant()
                .isBefore(Instant.now().minus(7, ChronoUnit.DAYS));
        } catch (IOException e) {
            return false;
        }
    })
    .forEach(path -> {
        try {
            Files.delete(path);
        } catch (IOException e) {
            logger.error("Failed to delete " + path, e);
        }
    });
```

## **7. Database Operations**
### **JDBC Batch Processing**
```java
List<Customer> customers = getCustomersToInsert();
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement("INSERT...")) {
    
    customers.forEach(customer -> {
        try {
            ps.setString(1, customer.getName());
            ps.setInt(2, customer.getAge());
            ps.addBatch();
        } catch (SQLException e) {
            logger.error("Batch error", e);
        }
    });
    ps.executeBatch();
}
```

## **8. Testing**
### **Parameterized Tests (JUnit 5)**
```java
@ParameterizedTest
@MethodSource("provideTestData")
void testCalculateTax(int income, int expectedTax) {
    assertEquals(expectedTax, taxCalculator.calculate(income));
}

private static Stream<Arguments> provideTestData() {
    return Stream.of(
        Arguments.of(50_000, 5_000),
        Arguments.of(100_000, 15_000),
        Arguments.of(150_000, 30_000)
    );
}
```

## **9. Functional Utilities**
### **Timed Execution Wrapper**
```java
public static <T> T timed(String operationName, Supplier<T> operation) {
    long start = System.currentTimeMillis();
    T result = operation.get();
    long duration = System.currentTimeMillis() - start;
    logger.info("{} took {} ms", operationName, duration);
    return result;
}

// Usage:
List<User> users = timed("Database query", 
    () -> userRepository.findAllActiveUsers());
```

## **10. Cloud & Serverless**
### **AWS Lambda Handler**
```java
public class OrderProcessor implements RequestHandler<SQSEvent, Void> {
    public Void handleRequest(SQSEvent event, Context context) {
        event.getRecords().stream()
            .map(SQSMessage::getBody)
            .map(this::parseOrder)
            .filter(order -> order.isValid())
            .forEach(order -> processOrder(order));
        return null;
    }
}
```

## **Key Patterns & Best Practices**
1. **Replace anonymous classes** with lambdas for cleaner code
2. **Use method references** where possible (`String::length`)
3. **Keep lambdas short** (extract complex logic to methods)
4. **Avoid side-effects** in stream operations
5. **Consider primitive streams** (IntStream, LongStream) for better performance

These patterns are used by companies like:
- **Uber** for real-time trip analysis
- **Netflix** for content recommendation pipelines
- **Airbnb** for dynamic pricing calculations
- **Banks** for fraud detection systems
