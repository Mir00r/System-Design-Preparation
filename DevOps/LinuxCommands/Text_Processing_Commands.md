# **Linux Text Processing Commands – Complete Guide for DevOps** 📝🚀

Master essential text processing commands for log analysis, file manipulation, and data extraction - critical skills for DevOps engineers working with configuration files, logs, and scripts.

---

## **Table of Contents** 📑
1. [Why Text Processing Matters](#1-why-text-processing-matters)
2. [Viewing File Contents](#2-viewing-file-contents)
3. [grep - Pattern Matching](#3-grep---pattern-matching)
4. [sed - Stream Editor](#4-sed---stream-editor)
5. [awk - Text Processing](#5-awk---text-processing)
6. [cut, paste, join](#6-cut-paste-join)
7. [sort, uniq, wc](#7-sort-uniq-wc)
8. [tr - Translate Characters](#8-tr---translate-characters)
9. [Regular Expressions](#9-regular-expressions)
10. [Practical DevOps Scenarios](#10-practical-devops-scenarios)
11. [Industry Best Practices](#11-industry-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Why Text Processing Matters** 🎯

DevOps engineers constantly work with:
- **Log files** - Analyzing application and system logs
- **Configuration files** - Parsing and modifying configs
- **CSV/JSON data** - Processing structured data
- **Script output** - Parsing command results

**Common Use Cases:**
```bash
# Find errors in logs
grep "ERROR" /var/log/app.log

# Extract specific columns from data
awk '{print $1, $3}' access.log

# Replace configuration values
sed -i 's/localhost/prod-server/g' config.ini

# Count unique IP addresses
awk '{print $1}' access.log | sort | uniq -c
```

---

## **2. Viewing File Contents** 👀

### **cat - Concatenate and Display**
```bash
cat file.txt                      # Display entire file
cat file1.txt file2.txt           # Concatenate files
cat -n file.txt                   # Show line numbers
cat -b file.txt                   # Number non-empty lines
cat -s file.txt                   # Squeeze blank lines
cat -A file.txt                   # Show all characters (tabs, line endings)
```

**Practical Examples:**
```bash
cat /etc/passwd                   # View system users
cat /var/log/syslog | tail -100   # Last 100 log lines
cat << EOF > config.yml           # Create file with heredoc
server:
  host: localhost
  port: 8080
EOF
```

### **head - Display First Lines**
```bash
head file.txt                     # First 10 lines (default)
head -n 20 file.txt               # First 20 lines
head -c 100 file.txt              # First 100 bytes
head -n -5 file.txt               # All except last 5 lines
```

**Multi-file:**
```bash
head -n 5 *.log                   # First 5 lines of each .log file
```

### **tail - Display Last Lines**
```bash
tail file.txt                     # Last 10 lines (default)
tail -n 50 file.txt               # Last 50 lines
tail -c 100 file.txt              # Last 100 bytes
tail -n +10 file.txt              # From line 10 to end
tail -f /var/log/app.log          # Follow file (real-time)
tail -F /var/log/app.log          # Follow with retry
tail -f --pid=1234 app.log        # Follow until process dies
```

**Real-time Monitoring:**
```bash
tail -f /var/log/nginx/access.log | grep "404"
tail -f app.log | grep -i error
tail -f -n 100 /var/log/syslog    # Last 100 lines, then follow
```

### **less/more - Page Through Files**
```bash
less file.txt                     # Interactive viewer
less +F /var/log/app.log          # Follow mode (like tail -f)
```

**less Navigation:**
```
Space      # Next page
b          # Previous page
/pattern   # Search forward
?pattern   # Search backward
n          # Next match
N          # Previous match
g          # Go to start
G          # Go to end
q          # Quit
```

---

## **3. grep - Pattern Matching** 🔍

### **Basic grep Usage**
```bash
grep "pattern" file.txt           # Search for pattern
grep -i "error" log.txt           # Case-insensitive
grep -v "debug" log.txt           # Invert match (exclude)
grep -n "function" script.sh      # Show line numbers
grep -c "ERROR" log.txt           # Count matches
grep -l "TODO" *.py               # List files with matches
grep -L "test" *.txt              # List files without matches
grep -w "error" log.txt           # Match whole word only
```

### **Recursive Search**
```bash
grep -r "password" /etc/          # Recursive search
grep -R "TODO" .                  # Recursive (follow symlinks)
grep -r "import" --include="*.py" .  # Search only .py files
grep -r "error" --exclude-dir=node_modules .
```

### **Advanced grep**
```bash
grep -A 3 "ERROR" log.txt         # 3 lines after match
grep -B 2 "ERROR" log.txt         # 2 lines before match
grep -C 5 "ERROR" log.txt         # 5 lines before and after
grep -E "error|warning" log.txt   # Extended regex (multiple patterns)
grep -F "literal.string" file     # Fixed string (no regex)
grep -o "http[s]*://[^[:space:]]*" file  # Only matching part
```

**Combining Patterns:**
```bash
grep -E "^(error|warning|critical)" log.txt
grep "ERROR" log.txt | grep -v "DEBUG"
grep -P "\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}" log.txt  # Perl regex
```

### **Practical Examples**
```bash
# Find all error logs today
grep "$(date +%Y-%m-%d)" /var/log/app.log | grep -i error

# Find failed login attempts
grep "Failed password" /var/log/auth.log

# Search for IP addresses
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" access.log

# Find files modified in last 24 hours with specific content
find . -mtime -1 -type f -exec grep -l "deployment" {} \;

# Count unique error types
grep "ERROR" app.log | awk '{print $4}' | sort | uniq -c
```

---

## **4. sed - Stream Editor** ✏️

### **Basic Substitution**
```bash
sed 's/old/new/' file.txt         # Replace first occurrence per line
sed 's/old/new/g' file.txt        # Replace all occurrences
sed 's/old/new/gi' file.txt       # Case-insensitive
sed 's/old/new/2' file.txt        # Replace 2nd occurrence only
sed -i 's/old/new/g' file.txt     # In-place edit
sed -i.bak 's/old/new/g' file.txt # In-place with backup
```

### **Line-based Operations**
```bash
sed -n '5p' file.txt              # Print line 5
sed -n '1,10p' file.txt           # Print lines 1-10
sed '5d' file.txt                 # Delete line 5
sed '1,5d' file.txt               # Delete lines 1-5
sed '/pattern/d' file.txt         # Delete lines matching pattern
sed '/^$/d' file.txt              # Delete empty lines
sed '/^#/d' file.txt              # Delete comment lines
```

### **Advanced sed**
```bash
sed -n '/start/,/end/p' file.txt  # Print between patterns
sed '5a\New line' file.txt        # Append after line 5
sed '5i\New line' file.txt        # Insert before line 5
sed '5c\Replacement' file.txt     # Change line 5
sed 's/\(.*\)@\(.*\)/\2 <\1@\2>/' emails.txt  # Capture groups
```

**Multiple Commands:**
```bash
sed -e 's/foo/bar/g' -e 's/old/new/g' file.txt
sed '/pattern1/s/old/new/g' file.txt  # Substitute only on matching lines
```

### **Practical Examples**
```bash
# Replace localhost with production server
sed -i 's/localhost:3000/prod.example.com/g' config.js

# Remove comments and empty lines
sed '/^#/d; /^$/d' config.conf

# Extract email addresses
sed -n 's/.*\([a-zA-Z0-9._%+-]*@[a-zA-Z0-9.-]*\.[a-zA-Z]\{2,\}\).*/\1/p' file.txt

# Change log level
sed -i 's/log_level=DEBUG/log_level=INFO/g' app.conf

# Add header to file
sed -i '1i# Configuration File' config.ini

# Replace timestamp format
sed 's/\([0-9]\{4\}\)-\([0-9]\{2\}\)-\([0-9]\{2\}\)/\3\/\2\/\1/g' dates.txt
```

---

## **5. awk - Text Processing** 📊

### **Basic awk Syntax**
```awk
awk 'pattern {action}' file.txt
```

### **Field Processing**
```bash
awk '{print $1}' file.txt         # Print first column
awk '{print $1, $3}' file.txt     # Print columns 1 and 3
awk '{print $NF}' file.txt        # Print last column
awk '{print $0}' file.txt         # Print entire line
awk '{print NR, $0}' file.txt     # Print line number and line
```

**Field Separator:**
```bash
awk -F: '{print $1}' /etc/passwd  # Use : as separator
awk -F',' '{print $2}' data.csv   # CSV processing
awk 'BEGIN{FS=","} {print $1}'    # Set separator in BEGIN
```

### **Patterns and Conditions**
```bash
awk '/pattern/ {print}' file.txt  # Print lines matching pattern
awk '/ERROR/ {print $1, $2}' log.txt
awk '$3 > 100 {print}' data.txt   # Numeric comparison
awk '$1 == "admin" {print}' file.txt
awk 'NR > 1 {print}' file.txt     # Skip header line
awk 'length($0) > 80 {print}' file.txt  # Lines longer than 80 chars
```

### **Built-in Variables**
```bash
awk '{print NR, NF, $0}' file.txt # Line number, field count, line
```

| Variable | Meaning |
|----------|---------|
| **NR** | Current line number |
| **NF** | Number of fields |
| **FS** | Field separator |
| **RS** | Record separator |
| **OFS** | Output field separator |
| **ORS** | Output record separator |
| **FILENAME** | Current filename |

### **Advanced awk**
```bash
# Sum values
awk '{sum += $1} END {print sum}' numbers.txt

# Average
awk '{sum += $1; count++} END {print sum/count}' numbers.txt

# Conditional sum
awk '$2 == "ERROR" {count++} END {print count}' log.txt

# Multiple patterns
awk '/ERROR/ {errors++} /WARNING/ {warnings++} END {print "Errors:", errors, "Warnings:", warnings}' log.txt

# Custom output separator
awk 'BEGIN{OFS=","} {print $1, $2, $3}' file.txt

# Process CSV
awk -F',' '{print $1 "\t" $3}' data.csv
```

### **Practical Examples**
```bash
# Extract IP addresses and count
awk '{print $1}' access.log | sort | uniq -c | sort -rn

# Calculate total disk usage
ls -l | awk '{sum += $5} END {print sum/1024/1024 " MB"}'

# Find users with UID > 1000
awk -F: '$3 > 1000 {print $1}' /etc/passwd

# Memory usage analysis
free -m | awk 'NR==2 {printf "Memory Usage: %.2f%%\n", $3/$2*100}'

# Parse nginx access log
awk '{print $1, $7, $9}' /var/log/nginx/access.log

# Extract and sum response times
awk '{sum += $NF; count++} END {print "Avg:", sum/count "ms"}' response_times.log

# Group and count by status code
awk '{count[$9]++} END {for (code in count) print code, count[code]}' access.log
```

---

## **6. cut, paste, join** ✂️

### **cut - Extract Columns**
```bash
cut -d':' -f1 /etc/passwd         # Extract first field
cut -d',' -f1,3 data.csv          # Fields 1 and 3
cut -d' ' -f2-5 file.txt          # Fields 2 through 5
cut -c1-10 file.txt               # Characters 1-10
cut -c1,5,10 file.txt             # Specific characters
```

**Examples:**
```bash
ps aux | cut -d' ' -f1,11-       # Username and command
echo "192.168.1.100" | cut -d'.' -f1-3  # 192.168.1
```

### **paste - Merge Lines**
```bash
paste file1.txt file2.txt         # Side-by-side
paste -d',' file1.txt file2.txt   # With comma separator
paste -s file.txt                 # Serial (one file, all lines)
```

**Example:**
```bash
paste names.txt ages.txt
# John    25
# Jane    30

paste -d',' names.txt emails.txt > contacts.csv
```

### **join - Merge Based on Common Field**
```bash
join file1.txt file2.txt          # Join on first field
join -t',' file1.csv file2.csv    # CSV join
join -1 2 -2 1 file1 file2        # Join field 2 of file1 with field 1 of file2
```

**Example:**
```bash
# users.txt: 1 John, 2 Jane
# departments.txt: 1 Engineering, 2 Marketing
join -t',' users.txt departments.txt
```

---

## **7. sort, uniq, wc** 📈

### **sort - Sort Lines**
```bash
sort file.txt                     # Alphabetical sort
sort -r file.txt                  # Reverse sort
sort -n numbers.txt               # Numeric sort
sort -k2 file.txt                 # Sort by 2nd column
sort -t',' -k3 data.csv           # Sort CSV by 3rd column
sort -u file.txt                  # Sort and remove duplicates
sort -h sizes.txt                 # Human-readable numbers (1K, 1M)
```

**Advanced Sorting:**
```bash
du -sh * | sort -h                # Sort by size
ls -lh | sort -k5 -h              # Sort by file size
sort -t',' -k2,2n -k3,3 data.csv  # Multi-column sort
```

### **uniq - Remove Duplicates**
```bash
uniq file.txt                     # Remove consecutive duplicates
uniq -c file.txt                  # Count occurrences
uniq -d file.txt                  # Show only duplicates
uniq -u file.txt                  # Show only unique lines
uniq -i file.txt                  # Case-insensitive
```

**Important**: `uniq` only removes consecutive duplicates. Use with `sort`:
```bash
sort file.txt | uniq              # Remove all duplicates
sort file.txt | uniq -c           # Count unique lines
```

**Practical Examples:**
```bash
# Find most common errors
grep "ERROR" app.log | sort | uniq -c | sort -rn | head -10

# Count unique IP addresses
awk '{print $1}' access.log | sort | uniq -c

# Find duplicate lines
sort file.txt | uniq -d
```

### **wc - Word Count**
```bash
wc file.txt                       # Lines, words, bytes
wc -l file.txt                    # Count lines
wc -w file.txt                    # Count words
wc -c file.txt                    # Count bytes
wc -m file.txt                    # Count characters
wc -L file.txt                    # Longest line length
```

**Examples:**
```bash
ls -1 | wc -l                     # Count files
grep "ERROR" log.txt | wc -l      # Count errors
find . -name "*.py" | wc -l       # Count Python files
cat large.txt | wc -c             # File size in bytes
```

---

## **8. tr - Translate Characters** 🔄

### **Basic Translation**
```bash
tr 'a-z' 'A-Z' < file.txt         # Lowercase to uppercase
tr 'A-Z' 'a-z' < file.txt         # Uppercase to lowercase
tr ' ' '_' < file.txt             # Replace spaces with underscores
tr -d '0-9' < file.txt            # Delete digits
tr -s ' ' < file.txt              # Squeeze repeated spaces
```

### **Advanced tr**
```bash
tr -cd '[:alnum:]' < file.txt     # Keep only alphanumeric
tr -d '\r' < dos.txt > unix.txt   # Remove carriage returns (DOS to Unix)
tr -s '\n' ' ' < file.txt         # Replace newlines with spaces
tr '{}' '()' < code.txt           # Replace braces with parentheses
```

**Practical Examples:**
```bash
# Convert CSV to TSV
tr ',' '\t' < data.csv > data.tsv

# Remove all punctuation
tr -d '[:punct:]' < text.txt

# ROT13 encoding
tr 'A-Za-z' 'N-ZA-Mn-za-m' < message.txt

# Count words (using tr to split)
cat file.txt | tr -s ' ' '\n' | sort | uniq -c
```

---

## **9. Regular Expressions** 🎯

### **Basic Regex Patterns**
```
.          # Any single character
^          # Start of line
$          # End of line
*          # Zero or more
+          # One or more (extended regex)
?          # Zero or one (extended regex)
[]         # Character class
[^]        # Negated class
|          # OR (extended regex)
()         # Grouping
\          # Escape special character
```

### **Character Classes**
```
[0-9]      # Any digit
[a-z]      # Lowercase letter
[A-Z]      # Uppercase letter
[a-zA-Z]   # Any letter
[^0-9]     # Not a digit
[:alnum:]  # Alphanumeric
[:alpha:]  # Alphabetic
[:digit:]  # Digits
[:space:]  # Whitespace
```

### **Examples**
```bash
# Match email addresses
grep -E "[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}" file.txt

# Match IP addresses
grep -E "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" log.txt

# Match dates (YYYY-MM-DD)
grep -E "[0-9]{4}-[0-9]{2}-[0-9]{2}" file.txt

# Match URLs
grep -E "https?://[a-zA-Z0-9./?=_-]+" file.txt

# Lines starting with #
grep "^#" file.txt

# Lines ending with ;
grep ";$" code.js

# Empty lines
grep "^$" file.txt
```

---

## **10. Practical DevOps Scenarios** 🛠️

### **Scenario 1: Log Analysis**
```bash
# Find top 10 IP addresses accessing the server
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Count HTTP status codes
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# Find failed requests (4xx, 5xx)
grep -E " (4[0-9]{2}|5[0-9]{2}) " access.log

# Average response time
awk '{sum += $NF; count++} END {print sum/count}' response_times.log

# Extract errors from last hour
grep "$(date -d '1 hour ago' '+%Y-%m-%d %H')" /var/log/app.log | grep ERROR
```

### **Scenario 2: Configuration File Processing**
```bash
# Remove comments and empty lines from config
grep -v "^#" nginx.conf | grep -v "^$"
sed '/^#/d; /^$/d' config.ini

# Replace environment-specific values
sed -i 's/DB_HOST=localhost/DB_HOST=prod-db.example.com/g' .env

# Extract database credentials
grep -E "^(DB_HOST|DB_USER|DB_PASS)" .env

# Merge multiple config files
cat default.conf custom.conf > final.conf
```

### **Scenario 3: Performance Monitoring**
```bash
# Top 10 memory-consuming processes
ps aux | sort -k4 -rn | head -10

# Find large log files
find /var/log -type f -size +100M -exec ls -lh {} \; | awk '{print $9, $5}'

# Count active connections
netstat -an | grep ESTABLISHED | wc -l

# Monitor disk I/O
iostat | awk 'NR>3 {print $1, $4}' # Device and write speed
```

### **Scenario 4: Data Transformation**
```bash
# Convert log format
awk '{print $4, $1, $7, $9}' access.log > simplified.log

# Generate CSV report
echo "IP,Requests,Bandwidth" > report.csv
awk '{ip[$1]++; bw[$1]+=$10} END {for (i in ip) print i","ip[i]","bw[i]}' access.log | sort -t',' -k2 -rn >> report.csv

# Parse JSON log (simple)
grep "error" app.json | sed 's/.*"message":"\([^"]*\)".*/\1/'
```

### **Scenario 5: System Audit**
```bash
# Find recently modified config files
find /etc -name "*.conf" -mtime -7 -ls

# List users with shell access
awk -F: '$7 !~ /nologin|false/ {print $1, $7}' /etc/passwd

# Find SUID files (security audit)
find / -perm -4000 -type f 2>/dev/null

# Check failed SSH logins
grep "Failed password" /var/log/auth.log | awk '{print $11}' | sort | uniq -c | sort -rn
```

---

## **11. Industry Best Practices** 🏆

### **Performance Tips**

**1. Use grep for simple patterns, awk for complex processing:**
```bash
# Fast
grep "ERROR" large.log

# More flexible
awk '/ERROR/ {print $1, $2}' large.log
```

**2. Avoid unnecessary cat (UUOC - Useless Use of Cat):**
```bash
# Slow
cat file.txt | grep "pattern"

# Fast
grep "pattern" file.txt
```

**3. Use -F for fixed strings (faster than regex):**
```bash
grep -F "exact.string" file.txt  # Faster than grep "exact\.string"
```

**4. Process data in pipeline efficiently:**
```bash
# Good: single pass
awk '/ERROR/ {errors++} /WARNING/ {warnings++} END {print errors, warnings}' log.txt

# Bad: multiple passes
grep "ERROR" log.txt | wc -l
grep "WARNING" log.txt | wc -l
```

### **Script Best Practices**

**1. Always quote variables:**
```bash
grep "$pattern" "$file"  # Correct
grep $pattern $file      # Can break with spaces
```

**2. Check files exist:**
```bash
if [ -f "$file" ]; then
    grep "pattern" "$file"
fi
```

**3. Handle errors:**
```bash
if ! grep -q "pattern" file.txt; then
    echo "Pattern not found" >&2
    exit 1
fi
```

**4. Use appropriate tools:**
```bash
# Use cut for simple column extraction
cut -d',' -f2 data.csv

# Use awk for complex processing
awk -F',' '{if($3 > 100) print $1, $2}' data.csv
```

---

## **12. Interview Cheat Sheet** 📝

### **Quick Command Reference**

```bash
# Viewing
cat file | head -20 | tail -10     # Lines 11-20
less +F /var/log/app.log           # Follow like tail -f

# Searching
grep -r "pattern" /path            # Recursive search
grep -i "error" log.txt            # Case-insensitive
grep -v "debug" log.txt            # Exclude pattern
grep -E "err|warn" log.txt         # Multiple patterns

# Processing
sed 's/old/new/g' file             # Replace all
awk '{print $1, $3}' data          # Extract columns
awk -F':' '/root/ {print $1}' /etc/passwd  # Field separator + pattern

# Counting
sort data | uniq -c                # Count unique
wc -l file                         # Count lines
grep -c "pattern" file             # Count matches

# Transformation
tr 'a-z' 'A-Z' < file              # Uppercase
tr -d '\r' < dos > unix            # DOS to Unix
```

### **Common Interview Questions**

**Q1: How to find top 10 most frequent items in a file?**
```bash
sort file.txt | uniq -c | sort -rn | head -10
```

**Q2: How to remove duplicate lines while preserving order?**
```bash
awk '!seen[$0]++' file.txt
```

**Q3: How to extract IP addresses from log file?**
```bash
grep -oE "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b" access.log
```

**Q4: How to count unique values in 2nd column?**
```bash
awk '{print $2}' file | sort -u | wc -l
```

**Q5: How to replace text in multiple files?**
```bash
find . -name "*.conf" -exec sed -i 's/old/new/g' {} \;
```

**Q6: Difference between grep -E and grep -P?**
- **grep -E**: Extended regex (ERE)
- **grep -P**: Perl-compatible regex (PCRE) - more powerful

**Q7: How to print lines between two patterns?**
```bash
sed -n '/START/,/END/p' file.txt
awk '/START/,/END/' file.txt
```

---

## **Summary** ✅

Text processing is essential for DevOps. Master these commands:

1. **grep** - Pattern matching and searching
2. **sed** - Stream editing and substitution
3. **awk** - Advanced text processing and reporting
4. **cut/paste** - Column manipulation
5. **sort/uniq** - Sorting and deduplication
6. **tr** - Character translation
7. **wc** - Counting lines/words/characters

**Key Principles:**
- Use the right tool for the job
- Leverage pipelines for complex operations
- Quote variables and handle errors
- Prefer efficiency (avoid UUOC)
- Practice regular expressions

---

**Next Topics**: [System Monitoring Commands](System_Monitoring_Commands.md) | [Network Commands](Network_Commands.md)
