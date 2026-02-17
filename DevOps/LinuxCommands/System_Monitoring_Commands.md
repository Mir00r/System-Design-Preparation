# **Linux System Monitoring Commands – Complete Guide for DevOps** 📊🚀

Master essential monitoring commands to track system performance, diagnose issues, and ensure optimal resource utilization - critical skills for DevOps engineers managing production infrastructure.

---

## **Table of Contents** 📑
1. [Why System Monitoring Matters](#1-why-system-monitoring-matters)
2. [Process Monitoring](#2-process-monitoring)
3. [Memory Monitoring](#3-memory-monitoring)
4. [CPU Monitoring](#4-cpu-monitoring)
5. [Disk I/O Monitoring](#5-disk-io-monitoring)
6. [System Information](#6-system-information)
7. [Log Monitoring](#7-log-monitoring)
8. [Performance Analysis Tools](#8-performance-analysis-tools)
9. [Real-time Monitoring](#9-real-time-monitoring)
10. [Practical DevOps Scenarios](#10-practical-devops-scenarios)
11. [Industry Best Practices](#11-industry-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Why System Monitoring Matters** 🎯

**Critical for DevOps:**
- **Performance**: Identify bottlenecks and optimize resources
- **Troubleshooting**: Diagnose issues quickly
- **Capacity Planning**: Predict resource needs
- **Security**: Detect anomalies and intrusions
- **SLA Compliance**: Ensure uptime and performance targets

**Key Metrics to Monitor:**
```
CPU Usage      → top, mpstat, sar
Memory Usage   → free, vmstat
Disk I/O       → iostat, iotop
Network        → netstat, ss, iftop
Processes      → ps, pgrep, htop
Logs           → journalctl, tail
```

---

## **2. Process Monitoring** 🔄

### **ps - Process Status**
```bash
ps                          # Current user processes
ps aux                      # All processes (BSD style)
ps -ef                      # All processes (Unix style)
ps -u username              # User's processes
ps -p 1234                  # Specific process by PID
```

**Detailed Output:**
```bash
ps aux | head -1            # Header
ps aux | grep nginx         # Find nginx processes
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%cpu | head -10  # Top CPU consumers
ps -eo pid,ppid,cmd,%mem,%cpu --sort=-%mem | head -10  # Top memory consumers
```

**ps Fields Explained:**
```
USER    # Process owner
PID     # Process ID
%CPU    # CPU usage
%MEM    # Memory usage
VSZ     # Virtual memory size (KB)
RSS     # Resident set size (physical memory, KB)
TTY     # Terminal
STAT    # Process state (R=running, S=sleeping, D=uninterruptible, Z=zombie)
START   # Start time
TIME    # CPU time
COMMAND # Command
```

### **pgrep - Find Processes by Name**
```bash
pgrep nginx                 # Find nginx PIDs
pgrep -u www-data           # Processes by user
pgrep -l java               # Show PIDs and names
pgrep -a postgres           # Full command line
pgrep -c httpd              # Count processes
```

### **pstree - Process Tree**
```bash
pstree                      # Show process tree
pstree -p                   # Show PIDs
pstree username             # User's process tree
pstree -a                   # Show command-line arguments
```

### **top - Dynamic Process Viewer**
```bash
top                         # Interactive process viewer
top -u username             # Filter by user
top -p 1234,5678            # Monitor specific PIDs
top -d 2                    # Update every 2 seconds
top -o %MEM                 # Sort by memory
top -b -n 1                 # Batch mode (one iteration)
```

**top Interactive Commands:**
```
h       # Help
k       # Kill process
r       # Renice (change priority)
M       # Sort by memory
P       # Sort by CPU
T       # Sort by time
u       # Filter by user
1       # Show individual CPUs
q       # Quit
```

**top Output:**
```
top - 10:30:15 up 5 days,  1:23,  2 users,  load average: 0.15, 0.25, 0.30
Tasks: 120 total,   1 running, 119 sleeping,   0 stopped,   0 zombie
%Cpu(s):  5.0 us,  2.0 sy,  0.0 ni, 92.0 id,  1.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7890.5 total,   1234.2 free,   3456.3 used,   3200.0 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   3900.5 avail Mem
```

**Load Average Explained:**
- **1 min, 5 min, 15 min averages**
- Value < # of CPUs: System healthy
- Value == # of CPUs: Fully utilized
- Value > # of CPUs: System overloaded

### **htop - Enhanced Interactive Viewer**
```bash
htop                        # Better than top (needs install)
```

**Features:**
- Color-coded output
- Mouse support
- Horizontal/vertical scrolling
- Process tree view
- Multi-CPU visualization

Install htop:
```bash
# Ubuntu/Debian
sudo apt install htop

# CentOS/RHEL
sudo yum install htop

# macOS
brew install htop
```

**htop Commands:**
```
F1      # Help
F2      # Setup
F3      # Search
F4      # Filter
F5      # Tree view
F6      # Sort by
F9      # Kill
F10     # Quit
Space   # Tag process
```

### **pidstat - Process Statistics**
```bash
pidstat                     # Process statistics
pidstat 1 5                 # Every 1 second, 5 times
pidstat -p 1234             # Specific process
pidstat -u                  # CPU statistics
pidstat -r                  # Memory statistics
pidstat -d                  # Disk I/O
pidstat -t                  # Thread statistics
```

---

## **3. Memory Monitoring** 💾

### **free - Memory Usage**
```bash
free                        # Memory in KB
free -h                     # Human-readable
free -m                     # In MB
free -g                     # In GB
free -s 2                   # Update every 2 seconds
free -t                     # Show total
```

**Output Explained:**
```
              total        used        free      shared  buff/cache   available
Mem:          7890        3456        1234         100        3200        3900
Swap:         2048           0        2048
```

- **total**: Total installed RAM
- **used**: Memory in use by applications
- **free**: Unused memory
- **shared**: Memory shared between processes
- **buff/cache**: Kernel buffers and page cache
- **available**: Memory available for new applications (most important!)

**Important**: Linux uses free memory for caching. Low "free" doesn't mean problem if "available" is sufficient.

### **vmstat - Virtual Memory Statistics**
```bash
vmstat                      # Basic statistics
vmstat 1                    # Update every 1 second
vmstat 1 10                 # Every second, 10 times
vmstat -s                   # Summary
vmstat -d                   # Disk statistics
```

**Output:**
```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 1  0      0 123456  12345 678901    0    0     5    10  500  800  5  2 92  1  0
```

- **r**: Processes waiting for CPU
- **b**: Processes in uninterruptible sleep
- **swpd**: Virtual memory used
- **si/so**: Swap in/out (high values = problem!)
- **bi/bo**: Blocks in/out (disk I/O)
- **us/sy/id/wa**: CPU time (user/system/idle/wait)

### **smem - Memory Reporting**
```bash
smem                        # Memory usage by process
smem -t                     # Show totals
smem -u                     # By user
smem -p                     # Show percentages
smem -s pss                 # Sort by PSS
```

Install smem:
```bash
sudo apt install smem
```

### **Check for Memory Leaks**
```bash
# Monitor process memory over time
watch -n 1 "ps aux | grep java | grep -v grep"

# Track specific process
pidstat -r -p $(pgrep java) 1
```

---

## **4. CPU Monitoring** ⚡

### **uptime - System Uptime and Load**
```bash
uptime
# Output: 10:30:15 up 5 days, 1:23, 2 users, load average: 0.15, 0.25, 0.30
```

### **mpstat - Multi-Processor Statistics**
```bash
mpstat                      # CPU statistics
mpstat -P ALL               # All CPUs
mpstat 1 5                  # Every second, 5 times
mpstat -P 0,1               # Specific CPUs
```

**Output:**
```
%usr   %nice  %sys   %iowait %irq   %soft  %steal %guest %idle
5.21   0.00   2.15   1.05    0.00   0.10   0.00   0.00   91.49
```

- **%usr**: User processes
- **%sys**: System/kernel
- **%iowait**: Waiting for I/O (high = disk bottleneck)
- **%idle**: CPU idle time

### **lscpu - CPU Information**
```bash
lscpu                       # Detailed CPU info
lscpu | grep "^CPU(s)"      # Number of CPUs
```

### **nproc - Number of Processors**
```bash
nproc                       # Available processors
nproc --all                 # Total processors
```

### **stress - CPU Stress Testing**
```bash
stress --cpu 4 --timeout 60 # Stress 4 CPUs for 60 seconds
```

Install stress:
```bash
sudo apt install stress
```

---

## **5. Disk I/O Monitoring** 💿

### **iostat - I/O Statistics**
```bash
iostat                      # Basic I/O stats
iostat -x                   # Extended statistics
iostat -x 1 5               # Every second, 5 times
iostat -d                   # Disk statistics only
iostat -c                   # CPU statistics only
iostat -p sda               # Specific device
```

**Output:**
```
Device  tps    kB_read/s  kB_wrtn/s  kB_read  kB_wrtn
sda     50.23  1024.45    512.30     1000000  500000
```

- **tps**: Transfers per second
- **kB_read/s**: KB read per second
- **kB_wrtn/s**: KB written per second
- **%util**: Device utilization (100% = saturated)

### **iotop - I/O by Process**
```bash
sudo iotop                  # Interactive I/O monitor
sudo iotop -o               # Only show processes doing I/O
sudo iotop -p 1234          # Specific process
sudo iotop -a               # Accumulated I/O
```

Install iotop:
```bash
sudo apt install iotop
```

### **df - Disk Space**
```bash
df -h                       # Human-readable
df -i                       # Inode usage
df -T                       # Filesystem type
df /var                     # Specific mount
df -h --total               # Show total
```

### **du - Directory Usage**
```bash
du -sh /var/log             # Summary of directory
du -h --max-depth=1 /var    # Top-level directories
du -ah /var/log | sort -rh | head -20  # Top 20 largest
```

### **ioping - Disk Latency**
```bash
ioping /dev/sda             # Test disk latency
ioping -c 10 .              # 10 requests to current directory
```

Install ioping:
```bash
sudo apt install ioping
```

---

## **6. System Information** ℹ️

### **uname - System Information**
```bash
uname -a                    # All information
uname -r                    # Kernel version
uname -m                    # Machine hardware
uname -o                    # Operating system
```

### **hostname - System Hostname**
```bash
hostname                    # Show hostname
hostname -f                 # Fully qualified domain name
hostname -i                 # IP address
```

### **who/w - Logged in Users**
```bash
who                         # Who is logged in
w                           # Who and what they're doing
last                        # Login history
lastlog                     # Last login per user
```

### **dmesg - Kernel Ring Buffer**
```bash
dmesg                       # Kernel messages
dmesg | tail                # Recent messages
dmesg | grep -i error       # Error messages
dmesg -T                    # Human-readable timestamps
dmesg -w                    # Follow mode
```

### **lsb_release - Distribution Info**
```bash
lsb_release -a              # Distribution information
cat /etc/os-release         # Alternative
```

### **dmidecode - Hardware Information**
```bash
sudo dmidecode              # All hardware info
sudo dmidecode -t system    # System information
sudo dmidecode -t memory    # Memory information
sudo dmidecode -t processor # CPU information
sudo dmidecode -t bios      # BIOS information
```

---

## **7. Log Monitoring** 📋

### **journalctl - Systemd Journal**
```bash
journalctl                  # All logs
journalctl -f               # Follow (tail -f)
journalctl -u nginx         # Specific service
journalctl -p err           # Error priority and above
journalctl --since "1 hour ago"
journalctl --since "2024-01-01" --until "2024-01-31"
journalctl -k               # Kernel messages
journalctl --disk-usage     # Journal disk usage
journalctl --vacuum-size=100M  # Clean old logs
```

**Priority Levels:**
```
0: emerg
1: alert
2: crit
3: err
4: warning
5: notice
6: info
7: debug
```

### **tail - Follow Logs**
```bash
tail -f /var/log/syslog     # Follow log file
tail -f /var/log/nginx/access.log | grep "404"
tail -F /var/log/app.log    # Follow with retry
tail -n 100 -f /var/log/messages  # Last 100 lines, then follow
```

### **multitail - Multiple Logs**
```bash
multitail /var/log/syslog /var/log/auth.log
```

Install multitail:
```bash
sudo apt install multitail
```

### **logrotate - Log Rotation**
```bash
logrotate -f /etc/logrotate.conf  # Force rotation
logrotate -d /etc/logrotate.conf  # Debug mode
```

---

## **8. Performance Analysis Tools** 🔧

### **sar - System Activity Report**
```bash
sar                         # CPU usage
sar -r                      # Memory usage
sar -b                      # I/O statistics
sar -n DEV                  # Network statistics
sar -u 1 10                 # CPU every second, 10 times
sar -f /var/log/sysstat/sa20  # From specific file
```

Install sar:
```bash
sudo apt install sysstat
sudo systemctl enable sysstat
```

### **strace - System Call Trace**
```bash
strace command              # Trace command execution
strace -p 1234              # Trace running process
strace -c command           # Summary statistics
strace -e open ls           # Trace specific syscall
strace -o output.txt command  # Save to file
```

**Common Use Cases:**
```bash
# Debug why command is slow
strace -T ls

# Find which files a process opens
strace -e open,openat nginx

# Find why process fails
strace -f -e trace=file ./script.sh
```

### **lsof - List Open Files**
```bash
lsof                        # All open files
lsof /var/log/syslog        # Which process has file open
lsof -u username            # Files opened by user
lsof -p 1234                # Files opened by process
lsof -i                     # Network connections
lsof -i :80                 # Port 80
lsof -i TCP                 # TCP connections
lsof +D /var/log            # All files in directory
```

**Practical Examples:**
```bash
# Find process using deleted file (showing disk usage)
lsof | grep deleted

# Find what's using /dev/sda1
lsof /dev/sda1

# Find process listening on port
lsof -i :3000
```

### **perf - Performance Monitoring**
```bash
perf top                    # Real-time performance counter
perf stat command           # Performance statistics
perf record command         # Record performance data
perf report                 # Show recorded data
```

Install perf:
```bash
sudo apt install linux-tools-generic
```

---

## **9. Real-time Monitoring** ⏱️

### **watch - Execute Periodically**
```bash
watch command               # Run every 2 seconds
watch -n 1 "df -h"          # Every 1 second
watch -d "ps aux | grep nginx"  # Highlight differences
watch -t uptime             # Without header
```

**Examples:**
```bash
watch -n 1 "free -h"
watch "du -sh /var/log/*"
watch "netstat -an | grep ESTABLISHED | wc -l"
```

### **glances - All-in-One Monitoring**
```bash
glances                     # Comprehensive monitoring
glances -t 1                # Update every second
glances --export csv --export-csv-file metrics.csv
```

Install glances:
```bash
sudo apt install glances
# or
pip install glances
```

### **nmon - Performance Monitor**
```bash
nmon                        # Interactive monitoring
```

Install nmon:
```bash
sudo apt install nmon
```

---

## **10. Practical DevOps Scenarios** 🛠️

### **Scenario 1: High CPU Usage**
```bash
# Identify top CPU consumers
top -b -n 1 | head -20
ps aux --sort=-%cpu | head -10

# Check per-CPU usage
mpstat -P ALL 1 5

# Find specific application threads
ps -eLf | grep java | sort -k4 -rn | head -10

# Trace process
strace -p $(pgrep -o java) -c
```

### **Scenario 2: Memory Issues**
```bash
# Check memory usage
free -h
vmstat 1 10

# Find memory hogs
ps aux --sort=-%mem | head -10

# Check for swap usage
swapon --show
vmstat -s | grep swap

# Monitor process memory over time
watch -n 1 "ps aux | awk 'NR==1 || /java/' | head -5"

# Find memory leaks
pidstat -r -p $(pgrep java) 1
```

### **Scenario 3: Disk I/O Bottleneck**
```bash
# Check I/O statistics
iostat -x 1 5

# Find processes doing I/O
sudo iotop -o

# Check disk latency
ioping /dev/sda

# Find large files
du -ah /var | sort -rh | head -20

# Check for deleted files still held open
lsof | grep deleted
```

### **Scenario 4: System Unresponsive**
```bash
# Check load average
uptime
cat /proc/loadavg

# Check processes in uninterruptible sleep
ps aux | awk '$8=="D"'

# Check zombie processes
ps aux | awk '$8=="Z"'

# System calls causing hang
sudo strace -p $(pgrep -o nginx)

# Check kernel messages
dmesg | tail -50
```

### **Scenario 5: Application Performance**
```bash
# Monitor application process
top -p $(pgrep -d',' java)

# Full statistics
pidstat -r -u -d -p $(pgrep java) 1

# System calls
strace -c -p $(pgrep java)

# Open files and connections
lsof -p $(pgrep java)

# Thread information
ps -eLf | grep java
```

---

## **11. Industry Best Practices** 🏆

### **Monitoring Strategy**

**1. Establish Baselines:**
```bash
# Collect normal behavior
sar -A > baseline-$(date +%Y%m%d).txt

# Regular snapshots
crontab -e
# */5 * * * * /usr/bin/sar -u -r -b 1 1 >> /var/log/perf/$(date +\%Y\%m\%d).log
```

**2. Set Up Alerting:**
```bash
# CPU threshold check
cpu_usage=$(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1)
if (( $(echo "$cpu_usage > 80" | bc -l) )); then
    echo "High CPU: $cpu_usage%" | mail -s "Alert" admin@example.com
fi

# Memory threshold
mem_usage=$(free | awk '/Mem:/{printf("%.0f"), $3/$2*100}')
if [ $mem_usage -gt 90 ]; then
    echo "High memory: $mem_usage%"
fi

# Disk space threshold
df -H | awk '{ if($5+0 > 85) print $0 }'
```

**3. Regular Health Checks:**
```bash
#!/bin/bash
# system-health.sh
echo "=== System Health Check ==="
echo "Load Average: $(uptime | awk -F'load average:' '{print $2}')"
echo "CPU Usage: $(top -b -n 1 | grep "Cpu(s)" | awk '{print $2}')"
echo "Memory: $(free -h | awk '/Mem:/{print $3 "/" $2}')"
echo "Disk: $(df -h / | awk 'NR==2{print $5}')"
echo "Top Processes:"
ps aux --sort=-%cpu | head -5
```

**4. Performance Tuning:**
```bash
# Check swappiness (lower = less swap)
cat /proc/sys/vm/swappiness
echo "vm.swappiness=10" | sudo tee -a /etc/sysctl.conf

# Check file descriptors limit
ulimit -n
# Increase if needed
echo "* soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "* hard nofile 65536" | sudo tee -a /etc/security/limits.conf
```

**5. Log Management:**
```bash
# Centralize logs
rsyslog or syslog-ng configuration

# Rotate logs
logrotate configuration

# Monitor log growth
du -sh /var/log/*

# Clean old logs
find /var/log -name "*.log" -mtime +30 -delete
```

### **Troubleshooting Workflow**

```
1. Check basic metrics (top, free, df)
2. Review logs (journalctl, /var/log/)
3. Identify suspect processes (ps, pidstat)
4. Deep dive (strace, lsof)
5. Analyze patterns (sar, vmstat)
6. Correlate with application behavior
7. Document findings
```

---

## **12. Interview Cheat Sheet** 📝

### **Quick Commands**

```bash
# Overall system status
top               # Interactive viewer
htop              # Enhanced viewer
uptime            # Load average
free -h           # Memory usage
df -h             # Disk space

# Process monitoring
ps aux            # All processes
pgrep nginx       # Find by name
pidstat -r -u 1   # Process stats

# Performance
iostat -x 1       # I/O statistics
vmstat 1          # Virtual memory
mpstat -P ALL     # CPU statistics
sar -A            # All statistics

# Logs
journalctl -f     # Follow journal
tail -f /var/log/syslog  # Follow log
dmesg | tail      # Kernel messages

# Debugging
strace -p PID     # Trace system calls
lsof -p PID       # Open files
netstat -tulpn    # Network connections
```

### **Common Interview Questions**

**Q1: How to find which process is using most CPU?**
```bash
top -b -n 1 | head -20
ps aux --sort=-%cpu | head -5
```

**Q2: How to check disk I/O bottleneck?**
```bash
iostat -x 1 5
# Check %util column (>80% = bottleneck)
sudo iotop -o
```

**Q3: What does high load average mean?**
- Load average shows average number of processes waiting for CPU
- Compare to CPU count: load > CPUs = overloaded
- Check with: `uptime` and `nproc`

**Q4: How to find memory leaks?**
```bash
# Monitor process memory over time
pidstat -r -p PID 1
watch -n 1 "ps aux | grep process"
# Look for steadily increasing RSS/VSZ
```

**Q5: How to check if disk is full?**
```bash
df -h                    # Check space
df -i                    # Check inodes
du -sh /* | sort -rh     # Find large directories
```

**Q6: What does %iowait indicate?**
- High iowait = CPU waiting for I/O operations
- Indicates disk bottleneck
- Check with: `iostat -x` and look for high %util

**Q7: How to monitor application in real-time?**
```bash
top -p $(pgrep java)
pidstat -r -u -d -p $(pgrep java) 1
watch -n 1 "ps aux | grep java"
```

**Q8: Difference between virtual and resident memory?**
- **VSZ (Virtual)**: Total virtual memory (includes swapped)
- **RSS (Resident)**: Physical memory actually in RAM

---

## **Summary** ✅

System monitoring is crucial for maintaining healthy infrastructure:

1. **Process Monitoring**: `top`, `htop`, `ps`, `pidstat`
2. **Memory**: `free`, `vmstat`, `smem`
3. **CPU**: `uptime`, `mpstat`, `sar`
4. **Disk I/O**: `iostat`, `iotop`, `df`, `du`
5. **Logs**: `journalctl`, `tail`, `dmesg`
6. **Debugging**: `strace`, `lsof`, `perf`

**Key Principles:**
- Establish performance baselines
- Monitor proactively, not reactively
- Use appropriate tools for each metric
- Correlate multiple data sources
- Document and automate monitoring

---

**Next Topics**: [Network Commands](Network_Commands.md) | [Process Management](Process_Management.md)
