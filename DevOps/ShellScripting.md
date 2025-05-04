# **Essential Linux Commands & Shell Scripting Guide for Automation** 🐧🛠️

To become proficient in automating system tasks and deployments with shell scripting, you need to master these core Linux commands and scripting concepts:

---

## **1. Fundamental Linux Commands for Scripting**
These commands form the building blocks of shell scripts:

### **File Operations**
```bash
ls -alh              # List files with details
cp -r src dest       # Recursive copy
mv old new           # Move/rename
rm -rf dir           # Force remove directory
chmod +x script.sh   # Make executable
chown user:file      # Change ownership
find / -name "*.log" # Search files
```

### **Text Processing**
```bash
grep "error" log.txt      # Search text
awk '{print $1}' file     # Extract columns
sed 's/foo/bar/g' file    # Find & replace
cat file.txt              # Display file
head/tail -n 5 file       # First/last 5 lines
sort | uniq -c            # Count unique lines
```

### **System Monitoring**
```bash
top/htop                # Process monitoring
df -h                   # Disk space
free -m                 # Memory usage
ps aux | grep nginx     # Find processes
netstat -tuln           # Open ports
journalctl -xe          # System logs
```

### **Networking**
```bash
curl -s http://example.com # HTTP requests
wget url                # Download files
ssh user@host           # Remote login
scp file user@host:path # Secure copy
ping google.com         # Network test
ifconfig/ip a           # IP addresses
```

---

## **2. Essential Shell Scripting Concepts**
### **Basic Script Structure**
```bash
#!/bin/bash             # Shebang (required)
# Comment              # Script description
VAR="value"            # Variable declaration
echo "Hello $VAR"      # Print variable
exit 0                 # Exit status
```

### **Variables & Input**
```bash
name="Alice"
read -p "Enter name: " name  # User input
echo "Hello $name"           # String interpolation
```

### **Conditionals**
```bash
if [ $age -gt 18 ]; then
  echo "Adult"
elif [ $age -gt 12 ]; then
  echo "Teen"
else
  echo "Child"
fi
```

### **Loops**
```bash
# For loop
for i in {1..5}; do
  echo "Iteration $i"
done

# While loop
while [ $counter -lt 5 ]; do
  echo $counter
  ((counter++))
done
```

### **Functions**
```bash
greet() {
  echo "Hello, $1"
}
greet "Alice"  # Call function
```

### **Exit Codes & Error Handling**
```bash
if ! command; then
  echo "Command failed" >&2
  exit 1
fi
```

---

## **3. Practical Automation Scripts**
### **1. System Backup Script**
```bash
#!/bin/bash
# Backup files to /backup with date
BACKUP_DIR="/backup"
SOURCE_DIR="/var/www"
DATE=$(date +%Y-%m-%d)

mkdir -p $BACKUP_DIR
tar -czf "$BACKUP_DIR/backup-$DATE.tar.gz" $SOURCE_DIR

if [ $? -eq 0 ]; then
  echo "Backup successful"
else
  echo "Backup failed!" >&2
  exit 1
fi
```

### **2. Log Cleanup Script**
```bash
#!/bin/bash
# Delete logs older than 30 days
LOG_DIR="/var/log"
DAYS=30

find $LOG_DIR -name "*.log" -type f -mtime +$DAYS -delete
echo "$(date): Cleaned logs" >> /var/log/cleanup.log
```

### **3. Deployment Script**
```bash
#!/bin/bash
# Pull latest code and restart service
PROJECT_DIR="/opt/myapp"
SERVICE="myapp.service"

cd $PROJECT_DIR
git pull origin main
systemctl restart $SERVICE
```

### **4. Disk Monitoring Script**
```bash
#!/bin/bash
# Alert if disk usage > 90%
THRESHOLD=90
USAGE=$(df / | awk 'NR==2 {print $5}' | tr -d '%')

if [ $USAGE -gt $THRESHOLD ]; then
  echo "Disk usage is $USAGE%" | mail -s "Disk Alert" admin@example.com
fi
```

---

## **4. Must-Know Advanced Commands**
### **Process Management**
```bash
nohup ./script.sh &     # Run in background
disown -h %1            # Detach process
kill -9 PID             # Force kill
pgrep -f "nginx"        # Find PID
```

### **Parallel Execution**
```bash
parallel -j 4 ./process.sh ::: {1..10}  # Run 10 jobs in parallel
```

### **JSON Processing (jq)**
```bash
curl -s api.example.com | jq '.data[].name'
```

### **SSH Automation**
```bash
sshpass -p "password" ssh user@host "command"  # Non-interactive SSH
```

---

## **5. Best Practices for Reliable Scripts**
1. **Use `set -e`** - Exit on error
   ```bash
   #!/bin/bash
   set -euo pipefail  # Exit on error/unset vars/pipefail
   ```
2. **Log everything**
   ```bash
   exec > >(tee -a /var/log/myscript.log) 2>&1
   ```
3. **Validate inputs**
   ```bash
   if [ -z "$1" ]; then
     echo "Usage: $0 <filename>" >&2
     exit 1
   fi
   ```
4. **Use configuration files**
   ```bash
   source /etc/myscript.conf  # Load variables
   ```

---

## **Learning Path Recommendation** 📚
1. **Beginner**: Master basic commands (`grep`, `awk`, `find`)
2. **Intermediate**: Learn flow control (`if`, `for`, `while`)
3. **Advanced**: Study signal handling (`trap`), parallelism
4. **Expert**: Explore `expect`, `ansible`, and infrastructure-as-code

---

## **Key Tools for Professional Automation**
| Tool | Purpose |
|------|---------|
| **Cron** | Schedule scripts |
| **Ansible** | Configuration management |
| **Docker** | Containerized deployments |
| **Jenkins** | CI/CD pipelines |
| **Terraform** | Infrastructure provisioning |

Start with simple scripts and gradually incorporate these tools into your automation workflow! 🚀
