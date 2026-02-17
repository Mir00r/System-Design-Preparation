# **Linux Commands for DevOps** 🐧🚀

Comprehensive Linux command tutorials and guides specifically designed for DevOps engineers. This collection covers everything from basic file operations to advanced system administration.

---

## **📚 Complete Guide Collection**

### **Essential Commands**
Each guide is structured with detailed explanations, practical examples, real-world DevOps scenarios, industry best practices, and interview preparation content.

| Guide | Description | Topics Covered |
|-------|-------------|----------------|
| **[File System Commands](File_System_Commands.md)** | File and directory operations | Navigation, CRUD operations, permissions, finding files, compression, disk usage |
| **[Text Processing Commands](Text_Processing_Commands.md)** | Log analysis and data manipulation | grep, sed, awk, cut, sort, uniq, regular expressions |
| **[System Monitoring Commands](System_Monitoring_Commands.md)** | Performance tracking and diagnostics | Processes, memory, CPU, disk I/O, logs, debugging tools |
| **[Network Commands](Network_Commands.md)** | Networking and connectivity | Configuration, testing, DNS, HTTP, SSH, traffic analysis, troubleshooting |
| **[Process Management](Process_Management.md)** | Process control and scheduling | Start/stop processes, priorities, services, cron, resource limits |
| **[User & Permissions Management](User_Permissions_Management.md)** | Security and access control | Users, groups, permissions, ACLs, sudo, SSH keys |
| **[Package Management](Package_Management.md)** | Software installation and updates | APT, YUM/DNF, Snap, Flatpak, repositories, dependencies |
| **[Quick Reference Cheatsheet](Linux_Commands_Cheatsheet.md)** | Fast lookup reference | All commands, one-liners, shortcuts, emergency commands |

---

## **🎯 Learning Path**

### **Beginners** (Start Here)
1. 📂 [File System Commands](File_System_Commands.md) - Master basic navigation and file operations
2. 📝 [Text Processing Commands](Text_Processing_Commands.md) - Learn to work with files and logs
3. 📋 [Quick Reference Cheatsheet](Linux_Commands_Cheatsheet.md) - Handy reference for daily use

### **Intermediate**
4. 📊 [System Monitoring Commands](System_Monitoring_Commands.md) - Monitor system performance
5. ⚙️ [Process Management](Process_Management.md) - Control running processes
6. 📦 [Package Management](Package_Management.md) - Install and manage software

### **Advanced**
7. 🌐 [Network Commands](Network_Commands.md) - Network troubleshooting and configuration
8. 👥 [User & Permissions Management](User_Permissions_Management.md) - Security and access control

---

## **🔥 Quick Start**

### **Most Used Commands (Top 20)**

```bash
# File Operations
ls -lah                      # List files
cd /path                     # Change directory
cat file.txt                 # View file
tail -f app.log              # Follow log file

# System Monitoring
top                          # Process monitor
htop                         # Enhanced process monitor
free -h                      # Memory usage
df -h                        # Disk space

# Network
ip addr                      # Show IP addresses
ping host                    # Test connectivity
ss -tulpn                    # Listening ports
curl https://api.example.com # HTTP request

# Process Control
ps aux                       # List processes
kill PID                     # Stop process
systemctl status nginx       # Service status

# Text Processing
grep "error" log.txt         # Search file
awk '{print $1}' file        # Extract column
sed 's/old/new/g' file       # Replace text

# Permissions
chmod 755 file               # Set permissions
chown user:group file        # Change owner
```

---

## **💡 Use Cases by Role**

### **DevOps Engineer**
- **Infrastructure Management**: [System Monitoring](System_Monitoring_Commands.md), [Process Management](Process_Management.md)
- **Deployment**: [Package Management](Package_Management.md), [User & Permissions](User_Permissions_Management.md)
- **Troubleshooting**: [Network Commands](Network_Commands.md), [Text Processing](Text_Processing_Commands.md)

### **SRE (Site Reliability Engineer)**
- **Performance Tuning**: [System Monitoring](System_Monitoring_Commands.md)
- **Log Analysis**: [Text Processing](Text_Processing_Commands.md)
- **Incident Response**: [Quick Reference Cheatsheet](Linux_Commands_Cheatsheet.md)

### **System Administrator**
- **User Management**: [User & Permissions](User_Permissions_Management.md)
- **Software Management**: [Package Management](Package_Management.md)
- **System Maintenance**: [File System Commands](File_System_Commands.md)

### **Security Engineer**
- **Access Control**: [User & Permissions](User_Permissions_Management.md)
- **Network Security**: [Network Commands](Network_Commands.md)
- **Audit & Compliance**: [System Monitoring](System_Monitoring_Commands.md)

---

## **🛠️ Practical Scenarios**

### **Common DevOps Tasks**

**1. Troubleshoot High CPU Usage**
```bash
# Identify process
top -b -n 1 | head -20
ps aux --sort=-%cpu | head -5

# Analyze
strace -p PID
lsof -p PID

# Fix
renice -n 10 -p PID
systemctl restart service
```
📖 See: [System Monitoring](System_Monitoring_Commands.md), [Process Management](Process_Management.md)

**2. Analyze Application Logs**
```bash
# Find errors
grep -i error /var/log/app.log

# Count by type
awk '/ERROR/ {count++} END {print count}' app.log

# Top 10 error IPs
grep ERROR app.log | awk '{print $1}' | sort | uniq -c | sort -rn | head -10
```
📖 See: [Text Processing](Text_Processing_Commands.md)

**3. Network Connectivity Issues**
```bash
# Test connectivity
ping -c 4 host
traceroute host

# Check DNS
dig example.com
nslookup example.com

# Verify port
nc -zv host 80
lsof -i :80
```
📖 See: [Network Commands](Network_Commands.md)

**4. Deploy Application**
```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y nodejs npm

# Setup user
sudo useradd -m -s /bin/bash appuser

# Set permissions
sudo chown -R appuser:appuser /opt/app
sudo chmod 755 /opt/app

# Configure service
sudo systemctl enable app
sudo systemctl start app
```
📖 See: [Package Management](Package_Management.md), [User & Permissions](User_Permissions_Management.md)

---

## **📖 Features of Each Guide**

Every guide includes:

✅ **Comprehensive Coverage** - All essential commands explained in detail
✅ **Practical Examples** - Real-world usage scenarios
✅ **DevOps Focused** - Production environment best practices
✅ **Visual Aids** - Tables, diagrams, and formatted output
✅ **Interview Prep** - Common questions and answers
✅ **Best Practices** - Industry-standard approaches
✅ **Troubleshooting** - Common issues and solutions
✅ **Quick Reference** - Summary sections for fast lookup

---

## **🎓 Interview Preparation**

Each guide contains an **Interview Cheat Sheet** section with:
- Most commonly asked questions
- Command syntax examples
- Explanation of concepts
- Practical scenarios
- Best practices

**Top Interview Topics:**
1. File permissions (chmod, chown, umask)
2. Process management (ps, kill, systemctl)
3. Text processing (grep, sed, awk)
4. Network troubleshooting (ping, netstat, tcpdump)
5. System monitoring (top, free, df)

---

## **🔧 Additional Resources**

### **Related Topics**
- 🐳 [Docker](../Docker.md) - Container management
- ☸️ [Kubernetes](../Kubernetes.md) - Container orchestration
- 📜 [Shell Scripting](../ShellScripting.md) - Bash scripting fundamentals

### **Online Tools**
- [ExplainShell](https://explainshell.com) - Explain shell commands
- [ShellCheck](https://www.shellcheck.net) - Shell script analyzer
- [Regex101](https://regex101.com) - Regular expression tester

---

## **💻 Practice Labs**

### **Hands-on Exercises**

**Lab 1: File System Mastery**
- Create complex directory structures
- Practice file permissions
- Work with symbolic links
- Manage disk space

**Lab 2: Log Analysis**
- Parse nginx/apache logs
- Extract and analyze data
- Create reports with awk
- Automate with scripts

**Lab 3: System Monitoring**
- Track CPU and memory usage
- Identify performance bottlenecks
- Monitor disk I/O
- Create monitoring scripts

**Lab 4: Network Troubleshooting**
- Diagnose connectivity issues
- Analyze packet captures
- Configure network interfaces
- Test services and ports

---

## **🚀 Pro Tips**

**For Daily Use:**
1. Use `history` command effectively
2. Create useful aliases in ~/.bashrc
3. Master tab completion
4. Learn keyboard shortcuts (Ctrl+R, Ctrl+A, Ctrl+E)
5. Use `man` pages and `--help`

**For Automation:**
1. Write idempotent scripts
2. Add error handling
3. Use logging and verbosity
4. Test in non-production first
5. Version control your scripts

**For Security:**
1. Always use sudo judiciously
2. Set appropriate file permissions
3. Regularly audit users and permissions
4. Keep systems updated
5. Use SSH keys instead of passwords

---

## **📊 Command Frequency Guide**

**Daily Use (Must Know):**
```
ls, cd, pwd, cat, grep, ps, top, systemctl, curl, ssh
```

**Weekly Use (Very Important):**
```
find, chmod, chown, tail, df, du, netstat, ss, awk, sed
```

**Monthly Use (Important):**
```
tar, rsync, cron, useradd, usermod, iptables, tcpdump
```

---

## **🎯 Certification Prep**

These guides help prepare for:
- **Linux Foundation Certified System Administrator (LFCS)**
- **Red Hat Certified System Administrator (RHCSA)**
- **CompTIA Linux+**
- **AWS Certified SysOps Administrator**
- **Kubernetes Administrator (CKA)**

---

## **📝 Contributing**

Found an error or want to suggest improvements?
- All commands tested on Ubuntu 20.04/22.04 and CentOS 7/8
- Examples verified in production environments
- Best practices based on industry standards

---

## **🙏 Acknowledgments**

Based on:
- Linux man pages and documentation
- Industry best practices from major tech companies
- DevOps community guidelines
- Real-world production experience

---

## **📞 Quick Navigation**

| Need | Go To |
|------|-------|
| **Find a file** | [File System Commands](File_System_Commands.md#6-finding-files) |
| **Analyze logs** | [Text Processing](Text_Processing_Commands.md#3-grep---pattern-matching) |
| **High CPU** | [System Monitoring](System_Monitoring_Commands.md#10-practical-devops-scenarios) |
| **Network down** | [Network Commands](Network_Commands.md#10-practical-devops-scenarios) |
| **Kill process** | [Process Management](Process_Management.md#2-process-control) |
| **Set permissions** | [User & Permissions](User_Permissions_Management.md#4-permission-management) |
| **Install software** | [Package Management](Package_Management.md#2-apt-debianubuntu) |
| **Quick lookup** | [Cheatsheet](Linux_Commands_Cheatsheet.md) |

---

**Start Learning:** Pick a guide based on your current skill level and needs. Each guide is standalone but references others when relevant.

**Pro Tip:** Keep the [Quick Reference Cheatsheet](Linux_Commands_Cheatsheet.md) bookmarked for fast lookups during daily work!

---

Happy Learning! 🐧✨
