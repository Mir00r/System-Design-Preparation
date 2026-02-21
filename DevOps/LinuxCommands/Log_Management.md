# **Linux Log Management Commands** 📝

**Complete guide for system logs, log rotation, and log analysis**

---

## **Table of Contents** 📑
1. [Understanding Linux Logs](#1-understanding-linux-logs)
2. [System Log Files](#2-system-log-files)
3. [journalctl - Systemd Logs](#3-journalctl---systemd-logs)
4. [Log Rotation](#4-log-rotation)
5. [rsyslog Configuration](#5-rsyslog-configuration)
6. [Log Analysis Tools](#6-log-analysis-tools)
7. [Centralized Logging](#7-centralized-logging)
8. [Application Logging](#8-application-logging)
9. [DevOps Use Cases](#9-devops-use-cases)
10. [Troubleshooting](#10-troubleshooting)
11. [Best Practices](#11-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Understanding Linux Logs** 📊

### **Log Locations**

```bash
# Main log directories
/var/log/                    # Primary log directory
/var/log/syslog             # System log (Debian/Ubuntu)
/var/log/messages           # System log (RHEL/CentOS)
/var/log/auth.log           # Authentication logs (Debian/Ubuntu)
/var/log/secure             # Authentication logs (RHEL/CentOS)
/var/log/kern.log           # Kernel logs
/var/log/dmesg              # Boot messages
/var/log/boot.log           # System boot log

# Application logs
/var/log/apache2/           # Apache web server
/var/log/nginx/             # Nginx web server
/var/log/mysql/             # MySQL database
/var/log/postgresql/        # PostgreSQL database

# Systemd journal (binary format)
/var/log/journal/           # Systemd journal storage
```

### **Log Levels**

```
Syslog Severity Levels:
0 - Emergency (emerg)    - System unusable
1 - Alert (alert)        - Action required immediately
2 - Critical (crit)      - Critical conditions
3 - Error (err)          - Error conditions
4 - Warning (warning)    - Warning conditions
5 - Notice (notice)      - Normal but significant
6 - Informational (info) - Informational messages
7 - Debug (debug)        - Debug-level messages
```

---

## **2. System Log Files** 📁

### **View System Logs**

```bash
# View system log (Ubuntu/Debian)
sudo tail -f /var/log/syslog
sudo less /var/log/syslog
sudo cat /var/log/syslog | grep error

# View system log (RHEL/CentOS)
sudo tail -f /var/log/messages
sudo less /var/log/messages

# View authentication logs
sudo tail -f /var/log/auth.log      # Debian/Ubuntu
sudo tail -f /var/log/secure        # RHEL/CentOS

# View kernel logs
sudo dmesg                           # Kernel ring buffer
sudo dmesg -T                        # Human-readable timestamps
sudo dmesg -l err,warn              # Only errors and warnings
sudo tail -f /var/log/kern.log

# View boot logs
sudo journalctl -b                   # Current boot
sudo less /var/log/boot.log
```

### **Common Log Files**

```bash
# Authentication and security
/var/log/auth.log                    # SSH, sudo, user login (Debian/Ubuntu)
/var/log/secure                      # SSH, sudo, user login (RHEL/CentOS)
sudo grep "Failed password" /var/log/auth.log
sudo grep "sudo" /var/log/auth.log

# System events
/var/log/syslog                      # General system messages
/var/log/messages                    # General system messages (RHEL)

# Kernel
/var/log/kern.log                    # Kernel messages
/var/log/dmesg                       # Device driver messages

# Cron jobs
/var/log/cron                        # Cron job execution
/var/log/cron.log

# Mail server
/var/log/mail.log                    # Mail server logs
/var/log/mail.err                    # Mail errors

# Package management
/var/log/dpkg.log                    # Package installation (Debian/Ubuntu)
/var/log/apt/history.log            # APT history
/var/log/yum.log                     # YUM (RHEL/CentOS)
```

### **Web Server Logs**

```bash
# Apache
/var/log/apache2/access.log         # Access logs
/var/log/apache2/error.log          # Error logs
sudo tail -f /var/log/apache2/access.log
sudo tail -f /var/log/apache2/error.log

# Nginx
/var/log/nginx/access.log           # Access logs
/var/log/nginx/error.log            # Error logs
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log

# Analyze access log
sudo cat /var/log/nginx/access.log | awk '{print $1}' | sort | uniq -c | sort -rn
# Shows IP addresses with request counts
```

---

## **3. journalctl - Systemd Logs** 🔍

### **Basic journalctl Usage**

```bash
# View all logs
sudo journalctl

# Follow logs (like tail -f)
sudo journalctl -f

# View from current boot
sudo journalctl -b
sudo journalctl -b 0                # Current boot
sudo journalctl -b -1               # Previous boot
sudo journalctl -b -2               # Two boots ago

# List boots
sudo journalctl --list-boots

# View logs since specific time
sudo journalctl --since "2024-01-01"
sudo journalctl --since "1 hour ago"
sudo journalctl --since "10 minutes ago"
sudo journalctl --since today
sudo journalctl --since yesterday

# View logs until specific time
sudo journalctl --until "2024-01-31"
sudo journalctl --since "1 hour ago" --until "30 minutes ago"

# Combine time filters
sudo journalctl --since "2024-01-01" --until "2024-01-31"
```

### **Filter journalctl**

```bash
# By priority (log level)
sudo journalctl -p err              # Only errors
sudo journalctl -p warning          # Warning and above
sudo journalctl -p 0..3             # Emergency to Error

# By unit (service)
sudo journalctl -u nginx
sudo journalctl -u ssh
sudo journalctl -u docker
sudo journalctl -u nginx -f         # Follow Nginx logs

# Multiple units
sudo journalctl -u nginx -u mysql

# By process ID
sudo journalctl _PID=1234

# By user ID
sudo journalctl _UID=1000

# By executable
sudo journalctl /usr/bin/dockerd

# By kernel messages
sudo journalctl -k                  # Kernel messages only
sudo journalctl -k -b               # Kernel messages from current boot
```

### **journalctl Output Options**

```bash
# Reverse order (newest first)
sudo journalctl -r

# Last N lines
sudo journalctl -n 50               # Last 50 lines
sudo journalctl -n 100 -f           # Last 100 lines, then follow

# Output formats
sudo journalctl -o json             # JSON format
sudo journalctl -o json-pretty      # Pretty JSON
sudo journalctl -o verbose          # Verbose
sudo journalctl -o short-iso        # ISO timestamp
sudo journalctl -o cat              # Only message (no metadata)

# No pager (for scripts)
sudo journalctl --no-pager

# Export logs
sudo journalctl --since today -o json > logs.json
sudo journalctl -u nginx > nginx.log
```

### **journalctl Maintenance**

```bash
# Check journal disk usage
sudo journalctl --disk-usage

# Vacuum journal (cleanup)
sudo journalctl --vacuum-size=100M   # Keep only 100MB
sudo journalctl --vacuum-time=7d    # Keep last 7 days
sudo journalctl --vacuum-files=5    # Keep only 5 journal files

# Verify journal integrity
sudo journalctl --verify

# Rotate journals immediately
sudo journalctl --rotate
```

### **Persistent journald Configuration**

```bash
# Edit journald config
sudo nano /etc/systemd/journald.conf

# Common settings:
[Journal]
Storage=persistent          # Store logs on disk
SystemMaxUse=500M          # Max disk usage
SystemKeepFree=1G          # Keep 1GB free
MaxRetentionSec=7d         # Keep logs for 7 days
MaxFileSec=1month          # Rotate monthly

# Apply changes
sudo systemctl restart systemd-journald
```

---

## **4. Log Rotation** 🔄

### **logrotate Configuration**

```bash
# Main configuration
/etc/logrotate.conf                  # Global config
/etc/logrotate.d/                    # Service-specific configs

# View logrotate config
cat /etc/logrotate.conf
ls /etc/logrotate.d/

# Test logrotate (dry run)
sudo logrotate -d /etc/logrotate.conf
sudo logrotate -d /etc/logrotate.d/nginx

# Force rotation
sudo logrotate -f /etc/logrotate.conf
sudo logrotate -f /etc/logrotate.d/nginx
```

### **Create Custom logrotate Config**

```bash
# Create config for custom application
sudo nano /etc/logrotate.d/myapp

# Example config:
/var/log/myapp/*.log {
    daily                           # Rotate daily
    rotate 7                        # Keep 7 days
    compress                        # Compress old logs
    delaycompress                   # Delay compression for 1 cycle
    missingok                       # Don't error if log missing
    notifempty                      # Don't rotate if empty
    create 0640 myapp myapp         # Create new log with permissions
    sharedscripts                   # Run scripts once for all logs
    postrotate
        systemctl reload myapp > /dev/null 2>&1 || true
    endscript
}
```

### **logrotate Options**

```bash
# Rotation frequency
daily                               # Daily rotation
weekly                              # Weekly rotation
monthly                             # Monthly rotation
yearly                              # Yearly rotation
size 100M                           # Rotate when file reaches 100MB

# Retention
rotate 7                            # Keep 7 rotated logs
maxage 30                           # Remove logs older than 30 days

# Compression
compress                            # Compress rotated logs
nocompress                          # Don't compress
delaycompress                       # Compress on next rotation
compresscmd /bin/gzip              # Compression command
compressext .gz                     # Extension for compressed files

# File handling
create 0644 user group              # Create new log with permissions
nocreate                            # Don't create new log
copytruncate                        # Copy and truncate original
notifempty                          # Don't rotate empty files
missingok                           # Don't error if missing
ifempty                             # Rotate even if empty

# Scripts
prerotate                           # Run before rotation
    command
endscript

postrotate                          # Run after rotation
    command
endscript
```

### **Common logrotate Configurations**

```bash
# Nginx
/var/log/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 $(cat /var/run/nginx.pid)
    endscript
}

# Apache
/var/log/apache2/*.log {
    daily
    rotate 14
    compress
    delaycompress
    notifempty
    create 0640 root adm
    sharedscripts
    postrotate
        /etc/init.d/apache2 reload > /dev/null
    endscript
}

# MySQL
/var/log/mysql/*.log {
    daily
    rotate 7
    missingok
    create 640 mysql adm
    compress
    postrotate
        test -x /usr/bin/mysqladmin && \
        /usr/bin/mysqladmin --defaults-file=/etc/mysql/debian.cnf flush-error-log \
        2>/dev/null || true
    endscript
}
```

---

## **5. rsyslog Configuration** 📡

### **rsyslog Basics**

```bash
# Main config file
/etc/rsyslog.conf

# Additional configs
/etc/rsyslog.d/

# Check rsyslog status
sudo systemctl status rsyslog

# Restart rsyslog
sudo systemctl restart rsyslog

# Test configuration
sudo rsyslogd -N1
```

### **rsyslog Rules**

```bash
# Edit rsyslog config
sudo nano /etc/rsyslog.conf

# Rule format:
# facility.priority    action

# Examples:
*.info;mail.none;authpriv.none;cron.none    /var/log/messages
authpriv.*                                   /var/log/secure
mail.*                                       /var/log/maillog
cron.*                                       /var/log/cron
*.emerg                                      :omusrmsg:*
kern.*                                       /var/log/kern.log
```

### **Custom rsyslog Rules**

```bash
# Create custom rule
sudo nano /etc/rsyslog.d/50-myapp.conf

# Example: Send application logs to separate file
:programname, isequal, "myapp" /var/log/myapp.log
& stop

# Send specific priority to file
*.warn    /var/log/warnings.log

# Send to remote server
*.* @remote-server:514              # UDP
*.* @@remote-server:514             # TCP

# Filter by message content
:msg, contains, "error" /var/log/errors.log

# Restart rsyslog
sudo systemctl restart rsyslog
```

### **Remote Logging**

```bash
# Server (receiver) configuration
sudo nano /etc/rsyslog.conf

# Enable TCP reception (port 514)
module(load="imtcp")
input(type="imtcp" port="514")

# Or UDP
module(load="imudp")
input(type="imudp" port="514")

# Client (sender) configuration
sudo nano /etc/rsyslog.d/remote.conf

# Send all logs to remote server
*.* @remote-server:514              # UDP
*.* @@remote-server:514             # TCP

# Send specific logs
authpriv.* @@remote-server:514
```

---

## **6. Log Analysis Tools** 🔎

### **grep for Log Analysis**

```bash
# Search for errors
sudo grep -i error /var/log/syslog
sudo grep -i "error\|fail\|critical" /var/log/syslog

# Count occurrences
sudo grep -c "error" /var/log/syslog

# Search with context
sudo grep -A 5 -B 5 "error" /var/log/syslog  # 5 lines before/after

# Search multiple files
sudo grep "error" /var/log/*.log

# Recursive search
sudo grep -r "error" /var/log/

# Search with line numbers
sudo grep -n "error" /var/log/syslog

# Exclude pattern
sudo grep -v "info" /var/log/syslog

# Case-insensitive
sudo grep -i "ERROR" /var/log/syslog
```

### **awk for Log Analysis**

```bash
# Extract specific fields from Apache/Nginx logs
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
# Shows top IP addresses

# Filter by status code
sudo awk '$9 == 404 {print $7}' /var/log/nginx/access.log
# Shows all 404 URLs

# Count by hour
sudo awk '{print $4}' /var/log/nginx/access.log | cut -c 14-15 | sort | uniq -c
# Request count by hour

# Sum response sizes
sudo awk '{sum += $10} END {print sum/1024/1024 " MB"}' /var/log/nginx/access.log
```

### **sed for Log Processing**

```bash
# Replace text in logs
sed 's/ERROR/WARNING/g' /var/log/app.log

# Extract date range
sed -n '/2024-01-01/,/2024-01-31/p' /var/log/syslog

# Remove blank lines
sed '/^$/d' /var/log/app.log

# Print specific lines
sed -n '100,200p' /var/log/syslog
```

### **Specialized Tools**

```bash
# multitail - Monitor multiple logs
sudo multitail /var/log/syslog /var/log/auth.log

# lnav - Log file navigator
sudo lnav /var/log/syslog
sudo lnav /var/log/*.log

# ccze - Colorized log viewer
sudo tail -f /var/log/syslog | ccze -A

# goaccess - Web log analyzer
sudo goaccess /var/log/nginx/access.log -o report.html
```

---

## **7. Centralized Logging** 🌐

### **ELK Stack (Elasticsearch, Logstash, Kibana)**

```bash
# Install Elasticsearch
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.0.0-amd64.deb
sudo dpkg -i elasticsearch-8.0.0-amd64.deb
sudo systemctl start elasticsearch

# Install Logstash
wget https://artifacts.elastic.co/downloads/logstash/logstash-8.0.0-amd64.deb
sudo dpkg -i logstash-8.0.0-amd64.deb

# Logstash config example
sudo nano /etc/logstash/conf.d/syslog.conf

input {
  file {
    path => "/var/log/syslog"
    type => "syslog"
  }
}

filter {
  grok {
    match => { "message" => "%{SYSLOGBASE}" }
  }
}

output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "syslog-%{+YYYY.MM.dd}"
  }
}

# Install Kibana
wget https://artifacts.elastic.co/downloads/kibana/kibana-8.0.0-amd64.deb
sudo dpkg -i kibana-8.0.0-amd64.deb
sudo systemctl start kibana
```

### **Fluentd/Fluent Bit**

```bash
# Install Fluent Bit
curl https://raw.githubusercontent.com/fluent/fluent-bit/master/install.sh | sh

# Configure Fluent Bit
sudo nano /etc/fluent-bit/fluent-bit.conf

[INPUT]
    Name        tail
    Path        /var/log/nginx/access.log
    Tag         nginx.access

[OUTPUT]
    Name        es
    Match       *
    Host        elasticsearch
    Port        9200
    Index       nginx-logs
```

---

## **8. Application Logging** 📱

### **Application Log Locations**

```bash
# Docker containers
docker logs container_name
docker logs -f container_name
docker logs --since 1h container_name

# Kubernetes
kubectl logs pod_name
kubectl logs -f pod_name
kubectl logs pod_name -c container_name

# Python application (example)
/var/log/app/application.log

# Node.js PM2
~/.pm2/logs/app-out.log
~/.pm2/logs/app-error.log
pm2 logs
pm2 logs app_name

# Java applications
/var/log/tomcat/catalina.out
```

### **Custom Application Logging**

```bash
# Python logging to syslog
import logging
import logging.handlers

logger = logging.getLogger('myapp')
handler = logging.handlers.SysLogHandler(address='/dev/log')
logger.addHandler(handler)
logger.warning('This goes to syslog')

# Node.js logging to syslog
const winston = require('winston');
require('winston-syslog').Syslog;

const logger = winston.createLogger({
  transports: [
    new winston.transports.Syslog()
  ]
});

logger.info('This goes to syslog');
```

---

## **9. DevOps Use Cases** 🚀

### **Log Monitoring Script**

```bash
#!/bin/bash
# monitor-errors.sh - Alert on error patterns

LOGFILE="/var/log/syslog"
ERROR_PATTERN="error|fail|critical"
THRESHOLD=10

ERROR_COUNT=$(sudo grep -Ei "$ERROR_PATTERN" "$LOGFILE" | tail -100 | wc -l)

if [ $ERROR_COUNT -gt $THRESHOLD ]; then
    echo "WARNING: $ERROR_COUNT errors found in last 100 lines"
    sudo grep -Ei "$ERROR_PATTERN" "$LOGFILE" | tail -20
    # Send alert
fi
```

### **Log Analysis for Security**

```bash
#!/bin/bash
# detect-ssh-attacks.sh

LOGFILE="/var/log/auth.log"

echo "=== Failed SSH Login Attempts ===="
sudo grep "Failed password" "$LOGFILE" | \
    awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -10

echo -e "\n=== Successful SSH Logins ==="
sudo grep "Accepted" "$LOGFILE" | \
    awk '{print $1, $2, $3, $(NF-3), $(NF-1)}' | tail -20
```

### **Log Aggregation Script**

```bash
#!/bin/bash
# aggregate-logs.sh

SERVERS=("web1" "web2" "web3")
LOG_DIR="./collected-logs/$(date +%Y%m%d)"

mkdir -p "$LOG_DIR"

for server in "${SERVERS[@]}"; do
    echo "Collecting logs from $server..."
    ssh $server "sudo tar -czf /tmp/logs.tar.gz /var/log/nginx/" 2>/dev/null
    scp ${server}:/tmp/logs.tar.gz "$LOG_DIR/${server}-logs.tar.gz"
    ssh $server "sudo rm /tmp/logs.tar.gz"
done

echo "Logs collected in $LOG_DIR"
```

---

## **10. Troubleshooting** 🔧

### **Log Issues**

```bash
# Logs not being written
sudo systemctl status rsyslog
sudo systemctl restart rsyslog
sudo journalctl -xe                  # Check systemd errors

# Log disk space issues
df -h /var/log
du -sh /var/log/*
# Clean old logs
sudo find /var/log -name "*.gz" -mtime +30 -delete

# Corrupted journal
sudo journalctl --verify
sudo rm -rf /var/log/journal/*
sudo systemctl restart systemd-journald

# Permission denied
sudo chmod 644 /var/log/syslog
sudo chown syslog:adm /var/log/syslog
```

---

## **11. Best Practices** ⭐

### **Do's**

```
✅ Centralize logs from all servers
✅ Set up log rotation
✅ Monitor log disk usage
✅ Regular log analysis
✅ Alert on critical errors
✅ Retain logs for compliance (30-90 days minimum)
✅ Compress old logs
✅ Use structured logging (JSON)
✅ Include timestamps and context
✅ Separate application logs from system logs
```

### **Don'ts**

```
❌ Don't log sensitive data (passwords, keys)
❌ Don't fill disk with logs
❌ Don't ignore log warnings
❌ Don't disable logs for troubleshooting later
❌ Don't use only local logging in production
❌ Don't forget to rotate logs
❌ Don't log too verbosely in production
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: Where are system logs located?**
```
Main locations:
/var/log/syslog        # System log (Debian/Ubuntu)
/var/log/messages      # System log (RHEL/CentOS)
/var/log/auth.log      # Authentication (Debian/Ubuntu)
/var/log/secure        # Authentication (RHEL/CentOS)
/var/log/kern.log      # Kernel messages

Systemd journal:
/var/log/journal/      # Binary logs
journalctl             # Command to view

Application logs:
/var/log/nginx/
/var/log/apache2/
/var/log/mysql/
```

### **Q2: How to view logs in real-time?**
```
Traditional logs:
tail -f /var/log/syslog
tail -f /var/log/nginx/access.log

Systemd logs:
journalctl -f
journalctl -u nginx -f

Multiple logs:
multitail /var/log/syslog /var/log/auth.log
```

### **Q3: What is log rotation? Why needed?**
```
Log Rotation:
- Automatically manage log file sizes
- Rename, compress, and delete old logs
- Prevent disk full issues

Implementation:
- logrotate (system logs)
- Application-specific rotation

Configuration:
/etc/logrotate.conf
/etc/logrotate.d/

Example:
/var/log/nginx/*.log {
    daily
    rotate 7
    compress
    delaycompress
}

Why needed:
- Prevent disk space issues
- Maintain performance
- Easier log management
- Compliance requirements
```

### **Q4: journalctl vs traditional logs?**
```
journalctl (Systemd):
✅ Binary format (faster queries)
✅ Structured data
✅ Built-in filtering
✅ Automatic rotation
❌ Need journalctl to read

Traditional logs:
✅ Plain text (any tool can read)
✅ Universal compatibility
✅ Simple grep/awk/sed
❌ Manual rotation needed
❌ Less structured

Modern systems use both.
Systemd services → journalctl
Applications → /var/log/
```

### **Q5: Common log analysis commands?**
```
View logs:
journalctl                     # Systemd logs
tail -f /var/log/syslog       # Follow log
less /var/log/syslog          # Page through

Search:
grep -i error /var/log/syslog
journalctl -p err

Filter by service:
journalctl -u nginx
journalctl -u nginx -f

Time range:
journalctl --since "1 hour ago"
journalctl --since today

Count errors:
grep -c error /var/log/syslog

Top IPs (nginx):
awk '{print $1}' /var/log/nginx/access.log | \
sort | uniq -c | sort -rn | head -10
```

---

**📝 Master Log Management for Effective System Monitoring!**

*Logs are the foundation of troubleshooting and monitoring - essential for DevOps operations.*
