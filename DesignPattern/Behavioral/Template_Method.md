# 📐 Template Method Pattern: Define Skeleton, Let Kids Fill In! 🎯

> **"The recipe says: prep → cook → plate. But HOW you cook depends on the dish."**

---

## 🎬 The Story

```
🏗️ BUILDING A HOUSE — THE TEMPLATE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1: Lay foundation    ← SAME for all houses
Step 2: Build walls       ← DIFFERENT (brick, wood, glass)
Step 3: Add roof          ← DIFFERENT (flat, gabled, dome)
Step 4: Install utilities ← SAME for all houses

The SKELETON (order of steps) is fixed.
But SPECIFIC STEPS vary by house type.
That's Template Method! 🏠
```

---

## 💻 Java Implementation

```java
// 📐 Abstract class with Template Method
public abstract class DataMiner {
    
    // 🎯 THE TEMPLATE METHOD — defines the skeleton (final = can't override!)
    public final void mine(String path) {
        openFile(path);           // Step 1: Open
        String rawData = extractData(); // Step 2: Extract (varies!)
        String parsed = parseData(rawData); // Step 3: Parse (varies!)
        analyzeData(parsed);      // Step 4: Analyze (same for all)
        generateReport(parsed);   // Step 5: Report (same for all)
        closeFile();              // Step 6: Close
    }
    
    // Steps that VARY — subclasses implement these (abstract = MUST override)
    protected abstract String extractData();
    protected abstract String parseData(String data);
    
    // Steps that are SAME for all — concrete methods
    private void openFile(String path) {
        System.out.println("📂 Opening: " + path);
    }
    
    private void analyzeData(String data) {
        System.out.println("📊 Analyzing " + data.length() + " records...");
    }
    
    private void generateReport(String data) {
        System.out.println("📋 Report generated!");
    }
    
    private void closeFile() {
        System.out.println("📁 File closed.");
    }
    
    // HOOK — optional override (empty default)
    protected void beforeAnalysis() { } // Subclass CAN override, but doesn't have to
}

// 📄 Concrete: CSV Miner
public class CSVDataMiner extends DataMiner {
    @Override
    protected String extractData() {
        System.out.println("📄 Extracting CSV rows...");
        return "csv_data";
    }
    
    @Override
    protected String parseData(String data) {
        System.out.println("🔤 Parsing CSV comma-separated values...");
        return "parsed_" + data;
    }
}

// 📊 Concrete: Excel Miner
public class ExcelDataMiner extends DataMiner {
    @Override
    protected String extractData() {
        System.out.println("📊 Extracting Excel cells via Apache POI...");
        return "excel_data";
    }
    
    @Override
    protected String parseData(String data) {
        System.out.println("📊 Parsing Excel formulas and cell references...");
        return "parsed_" + data;
    }
}

// 🎮 Usage — same flow, different implementations!
DataMiner csvMiner = new CSVDataMiner();
csvMiner.mine("data.csv");   // Uses CSV-specific extract & parse

DataMiner excelMiner = new ExcelDataMiner();
excelMiner.mine("data.xlsx"); // Uses Excel-specific extract & parse
```

---

## 🌍 Real-World Examples

```java
// 🌱 Spring's JdbcTemplate — THE classic Template Method!
// You provide: SQL + RowMapper. Spring handles: connection, statement, cleanup!

// ☕ Java's AbstractList — subclass only implements get() and size()!
// java.io.InputStream — read(byte[], int, int) calls abstract read()
// javax.servlet.http.HttpServlet — service() calls doGet(), doPost()

// 📱 Android Activity Lifecycle:
// onCreate() → onStart() → onResume() → [your code] → onPause() → onStop()
```

---

## ⚡ When to Use

```
✅ Multiple classes share same algorithm STRUCTURE but differ in STEPS
✅ Want to control which parts subclasses can/can't override
✅ Prevent code duplication across similar algorithms
✅ Framework hooks — let users customize specific steps

❌ Subclasses need to change the ORDER of steps → Strategy instead
❌ Only one implementation exists → no need for template
```

---

## 🎯 Key Interview Point

**Template Method vs Strategy:**
- **Template Method**: Uses INHERITANCE. Parent defines skeleton, child overrides steps.
- **Strategy**: Uses COMPOSITION. Context delegates ENTIRE algorithm to strategy object.
- Template Method = "override some steps." Strategy = "replace the whole thing."

---

## 🏆 Achievement: Progress: 16/23 patterns ████████████████░

---

*← [Command](./Command.md) | [Iterator →](./Iterator.md)*
