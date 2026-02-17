# **Linux Commands Quick Reference Cheatsheet for DevOps** ⚡📋

A comprehensive quick reference guide covering all essential Linux commands for DevOps engineers. Perfect for quick lookups, interviews, and daily work.

---

## **Table of Contents** 📑
1. [File System Operations](#1-file-system-operations)
2. [Text Processing](#2-text-processing)
3. [System Monitoring](#3-system-monitoring)
4. [Network Commands](#4-network-commands)
5. [Process Management](#5-process-management)
6. [User & Permissions](#6-user--permissions)
7. [Package Management](#7-package-management)
8. [Quick Tips & Tricks](#8-quick-tips--tricks)
9. [One-Liner Solutions](#9-one-liner-solutions)
10. [Keyboard Shortcuts](#10-keyboard-shortcuts)

---

## **1. File System Operations** 📁

### **Navigation**
```bash
pwd                          # Print working directory
cd /path                     # Change directory
cd ~                         # Go to home
cd -                         # Go to previous directory
cd ..                        # Parent directory
ls -lah                      # List all files with details
```

### **File Operations**
```bash
touch file.txt               # Create file
cat file.txt                 # Display file
head -n 20 file              # First 20 lines
tail -n 20 file              # Last 20 lines
tail -f file.log             # Follow file (real-time)
less file.txt                # Page through file
cp src dest                  # Copy file
cp -r dir1 dir2              # Copy directory
mv old new                   # Move/rename
rm file                      # Delete file
rm -rf dir                   # Delete directory (force)
```

### **Finding Files**
```bash
find . -name "*.log"         # Find by name
find . -type f -size +100M   # Files > 100MB
find . -mtime -7             # Modified last 7 days
find . -name "*.tmp" -delete # Find and delete
locate filename              # Fast search (uses database)
which command                # Find command location
whereis python               # Find binary, source, man pages
```

### **Compression**
```bash
tar -czf archive.tar.gz dir/ # Create tar.gz
tar -xzf archive.tar.gz      # Extract tar.gz
tar -tzf archive.tar.gz      # List contents
gzip file.txt                # Compress file
gunzip file.txt.gz           # Decompress
zip archive.zip files        # Create zip
unzip archive.zip            # Extract zip
```

### **Permissions**
```bash
chmod 755 file               # rwxr-xr-x
chmod +x script.sh           # Make executable
chown user:group file        # Change owner
chgrp group file             # Change group
umask 022                    # Default permissions
```

### **Disk Usage**
```bash
df -h                        # Disk space
du -sh directory/            # Directory size
du -sh *                     # Size of each item
df -i                        # Inode usage
```

---

## **2. Text Processing** 📝

### **Viewing**
```bash
cat file                     # Display entire file
head -20 file                # First 20 lines
tail -20 file                # Last 20 lines
less file                    # Page through
more file                    # Page through (basic)
```

### **grep - Search**
```bash
grep "pattern" file          # Search for pattern
grep -i "error" log          # Case-insensitive
grep -v "debug" log          # Invert match (exclude)
grep -r "TODO" .             # Recursive search
grep -n "error" file         # Show line numbers
grep -c "error" file         # Count matches
grep -A 3 "error" log        # 3 lines after match
grep -B 2 "error" log        # 2 lines before match
grep -E "err|warn" log       # Extended regex
```

### **sed - Stream Editor**
```bash
sed 's/old/new/' file        # Replace first occurrence
sed 's/old/new/g' file       # Replace all
sed -i 's/old/new/g' file    # In-place edit
sed -n '5p' file             # Print line 5
sed '5d' file                # Delete line 5
sed '/pattern/d' file        # Delete matching lines
sed -n '/start/,/end/p' file # Print between patterns
```

### **awk - Text Processing**
```bash
awk '{print $1}' file        # Print first column
awk -F: '{print $1}' file    # Field separator :
awk '/pattern/ {print}' file # Print matching lines
awk '{sum+=$1} END {print sum}' file  # Sum column
awk 'NR>1 {print}' file      # Skip header
awk '{print NR, $0}' file    # Add line numbers
```

### **cut, sort, uniq**
```bash
cut -d: -f1 /etc/passwd      # Extract field
sort file                    # Sort alphabetically
sort -n file                 # Numeric sort
sort -r file                 # Reverse sort
sort -k2 file                # Sort by column 2
uniq file                    # Remove consecutive duplicates
sort file | uniq             # Remove all duplicates
sort file | uniq -c          # Count unique lines
```

### **Other Text Tools**
```bash
wc -l file                   # Count lines
wc -w file                   # Count words
wc -c file                   # Count bytes
tr 'a-z' 'A-Z' < file        # Lowercase to uppercase
tr -d '0-9' < file           # Delete digits
paste file1 file2            # Merge files side-by-side
```

---

## **3. System Monitoring** 📊

### **Process Monitoring**
```bash
top                          # Interactive process viewer
htop                         # Enhanced top
ps aux                       # All processes
ps -ef                       # All processes (Unix style)
ps aux --sort=-%cpu          # Sort by CPU
ps aux --sort=-%mem          # Sort by memory
pgrep nginx                  # Find process by name
pidstat -p PID 1             # Process statistics
pstree                       # Process tree
```

### **Memory**
```bash
free -h                      # Memory usage
vmstat 1                     # Virtual memory stats
cat /proc/meminfo            # Detailed memory info
smem                         # Memory by process
```

### **CPU**
```bash
uptime                       # Load average
mpstat -P ALL                # CPU statistics
lscpu                        # CPU information
nproc                        # Number of processors
```

### **Disk I/O**
```bash
iostat -x 1                  # I/O statistics
iotop                        # I/O by process
df -h                        # Disk space
du -sh *                     # Directory sizes
```

### **System Info**
```bash
uname -a                     # System information
hostname                     # Hostname
cat /etc/os-release          # OS information
lsb_release -a               # Distribution info
uptime                       # System uptime
last                         # Login history
who                          # Who is logged in
w                            # Who and what they're doing
```

### **Logs**
```bash
journalctl -f                # Follow systemd journal
journalctl -u nginx          # Service logs
journalctl --since "1 hour ago"
tail -f /var/log/syslog      # Follow log file
dmesg | tail                 # Kernel messages
```

---

## **4. Network Commands** 🌐

### **Configuration**
```bash
ip addr                      # Show IP addresses
ip route                     # Show routes
ip link                      # Show interfaces
ifconfig                     # Interface config (legacy)
hostname -I                  # Show IP address
```

### **Connectivity**
```bash
ping host                    # Test connectivity
ping -c 4 host               # Send 4 packets
traceroute host              # Trace route
mtr host                     # Combined ping+trace
telnet host port             # Test port connectivity
nc -zv host port             # Test port (netcat)
```

### **DNS**
```bash
dig example.com              # DNS lookup
dig +short example.com       # Short output
host example.com             # Simple lookup
nslookup example.com         # DNS query
getent hosts example.com     # System resolver lookup
```

### **Connections & Ports**
```bash
ss -tulpn                    # Listening ports
ss -tan                      # TCP connections
netstat -tulpn               # Listening ports (legacy)
lsof -i :80                  # What's using port 80
fuser -v 80/tcp              # Process using port
nmap -p 1-1000 host          # Port scan
```

### **HTTP Tools**
```bash
curl https://example.com     # HTTP request
curl -I https://example.com  # Headers only
curl -X POST -d "data" url   # POST request
wget https://example.com/file  # Download file
wget -c url                  # Resume download
```

### **SSH & Transfer**
```bash
ssh user@host                # SSH connection
ssh -p 2222 user@host        # Custom port
scp file user@host:/path     # Secure copy
rsync -avz src/ dest/        # Sync files
rsync -avz --delete src/ dst/  # Mirror
```

### **Traffic Analysis**
```bash
tcpdump -i eth0              # Capture packets
tcpdump port 80              # Capture port 80
iftop                        # Bandwidth monitor
nethogs                      # Bandwidth by process
vnstat                       # Network statistics
```

---

## **5. Process Management** ⚙️

### **Control**
```bash
command &                    # Run in background
Ctrl+Z                       # Suspend process
bg                           # Resume in background
fg                           # Bring to foreground
jobs                         # List background jobs
nohup command &              # Run immune to hangups
disown                       # Detach from shell
```

### **Kill Processes**
```bash
kill PID                     # Graceful termination
kill -9 PID                  # Force kill
kill -15 PID                 # SIGTERM (default)
pkill nginx                  # Kill by name
killall nginx                # Kill all instances
pkill -u username            # Kill user processes
```

### **Priority**
```bash
nice -n 10 command           # Set priority
renice -n 5 -p PID           # Change priority
ionice -c 2 -n 7 command     # I/O priority
```

### **Service Management**
```bash
systemctl start service      # Start service
systemctl stop service       # Stop service
systemctl restart service    # Restart service
systemctl status service     # Service status
systemctl enable service     # Enable on boot
systemctl disable service    # Disable on boot
systemctl list-units         # List all units
systemctl --failed           # Failed services
```

### **Scheduling**
```bash
crontab -e                   # Edit cron jobs
crontab -l                   # List cron jobs
at 2:00 PM                   # One-time task
atq                          # List scheduled tasks
atrm job_number              # Remove scheduled task
```

---

## **6. User & Permissions** 👥

### **User Management**
```bash
whoami                       # Current user
id                           # User ID and groups
sudo command                 # Run as root
sudo -u user command         # Run as user
su - user                    # Switch user
```

### **User Administration**
```bash
sudo useradd -m username     # Create user
sudo passwd username         # Set password
sudo usermod -aG group user  # Add to group
sudo userdel -r username     # Delete user
sudo chage -l username       # Password aging info
```

### **Groups**
```bash
groups username              # Show user's groups
sudo groupadd groupname      # Create group
sudo gpasswd -a user group   # Add user to group
sudo gpasswd -d user group   # Remove from group
```

### **Permissions**
```bash
chmod 755 file               # rwxr-xr-x
chmod u+x file               # Add execute for user
chmod g-w file               # Remove write for group
chmod -R 755 dir/            # Recursive
chown user:group file        # Change owner
chown -R user:group dir/     # Recursive
umask                        # View default mask
umask 022                    # Set default mask
```

### **Special Permissions**
```bash
chmod u+s file               # Set SUID
chmod g+s dir                # Set SGID
chmod +t dir                 # Set sticky bit
chmod 4755 file              # SUID + 755
chmod 2755 dir               # SGID + 755
chmod 1777 dir               # Sticky + 777
```

### **ACLs**
```bash
getfacl file                 # View ACL
setfacl -m u:user:rwx file   # Set ACL
setfacl -x u:user file       # Remove ACL
setfacl -b file              # Remove all ACLs
```

### **Sudo**
```bash
sudo visudo                  # Edit sudoers file
sudo -l                      # List privileges
sudo -k                      # Invalidate timestamp
sudo !!                      # Run last command as sudo
```

---

## **7. Package Management** 📦

### **APT (Debian/Ubuntu)**
```bash
sudo apt update              # Update package index
sudo apt upgrade             # Upgrade packages
sudo apt install package     # Install package
sudo apt remove package      # Remove package
sudo apt purge package       # Remove with config
sudo apt autoremove          # Remove unused packages
apt search keyword           # Search packages
apt show package             # Show package info
apt list --installed         # List installed
dpkg -i package.deb          # Install .deb file
dpkg -l                      # List installed packages
```

### **YUM/DNF (RHEL/CentOS/Fedora)**
```bash
sudo yum update              # Update packages
sudo yum install package     # Install package
sudo yum remove package      # Remove package
yum search keyword           # Search packages
yum info package             # Show package info
yum list installed           # List installed
rpm -ivh package.rpm         # Install .rpm file
rpm -qa                      # List installed packages
sudo dnf install package     # DNF (Fedora/RHEL 8+)
```

### **Snap**
```bash
snap find package            # Search
sudo snap install package    # Install
sudo snap remove package     # Remove
snap list                    # List installed
sudo snap refresh            # Update all
```

### **Common Tasks**
```bash
# Update system
sudo apt update && sudo apt upgrade -y        # Debian/Ubuntu
sudo yum update -y                            # RHEL/CentOS

# Search and install
apt search nginx | grep nginx                 # Search
sudo apt install nginx                        # Install

# List installed packages
dpkg -l | grep nginx                          # Debian/Ubuntu
rpm -qa | grep nginx                          # RHEL/CentOS

# Hold package from update
sudo apt-mark hold nginx                      # Debian/Ubuntu
sudo yum versionlock nginx                    # RHEL/CentOS
```

---

## **8. Quick Tips & Tricks** 💡

### **Command History**
```bash
history                      # Show command history
!!                           # Repeat last command
!number                      # Execute command number
!string                      # Execute last command starting with string
Ctrl+R                       # Reverse search history
history | grep keyword       # Search history
```

### **Redirection & Pipes**
```bash
command > file               # Redirect output (overwrite)
command >> file              # Redirect output (append)
command 2> file              # Redirect errors
command &> file              # Redirect all output
command1 | command2          # Pipe output
command < file               # Input from file
cat file1 file2 > merged     # Concatenate files
```

### **Variables**
```bash
VAR="value"                  # Set variable
echo $VAR                    # Use variable
export VAR="value"           # Export to environment
echo $PATH                   # View PATH
echo $HOME                   # Home directory
echo $USER                   # Current user
```

### **Aliases**
```bash
alias ll='ls -lah'           # Create alias
alias ..='cd ..'             # Quick navigation
unalias ll                   # Remove alias
# Add to ~/.bashrc for persistence
```

### **File Descriptors**
```bash
1>                           # Stdout
2>                           # Stderr
&>                           # Both stdout and stderr
2>&1                         # Redirect stderr to stdout
```

---

## **9. One-Liner Solutions** 🚀

### **System Administration**

**Find and kill process:**
```bash
kill $(pgrep -f "process-name")
pkill -f "process-name"
```

**Find large files:**
```bash
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null
du -ah / | sort -rh | head -20
```

**Top 10 memory-consuming processes:**
```bash
ps aux --sort=-%mem | head -11
```

**Count files in directory:**
```bash
find . -type f | wc -l
ls -1 | wc -l
```

**Disk usage by directory:**
```bash
du -h --max-depth=1 / | sort -rh | head -10
```

**Find and delete old files:**
```bash
find /var/log -name "*.log" -mtime +30 -delete
```

### **Log Analysis**

**Top 10 IP addresses:**
```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

**Count HTTP status codes:**
```bash
awk '{print $9}' access.log | sort | uniq -c | sort -rn
```

**Find errors in last hour:**
```bash
grep "$(date -d '1 hour ago' '+%Y-%m-%d %H')" /var/log/app.log | grep ERROR
```

**Real-time log monitoring:**
```bash
tail -f /var/log/app.log | grep --line-buffered ERROR
```

### **Network**

**Find process on port:**
```bash
lsof -i :8080
ss -tlnp | grep :8080
```

**Test port connectivity:**
```bash
nc -zv host 80
telnet host 80
curl -I http://host:80
```

**Count established connections:**
```bash
ss -tan state established | wc -l
netstat -an | grep ESTABLISHED | wc -l
```

**Find listening ports:**
```bash
ss -tlnp
netstat -tulpn
lsof -i -P -n | grep LISTEN
```

### **Text Processing**

**Remove duplicate lines (preserve order):**
```bash
awk '!seen[$0]++' file.txt
```

**Replace text in multiple files:**
```bash
find . -name "*.txt" -exec sed -i 's/old/new/g' {} \;
```

**Count unique values:**
```bash
awk '{print $2}' file | sort -u | wc -l
```

**Extract column from CSV:**
```bash
awk -F',' '{print $3}' data.csv
cut -d',' -f3 data.csv
```

### **Process Management**

**Kill all processes by name:**
```bash
pkill -9 firefox
killall -9 firefox
```

**Find zombie processes:**
```bash
ps aux | awk '$8=="Z"'
```

**Top CPU consumer:**
```bash
ps aux --sort=-%cpu | head -2 | tail -1
```

### **File Operations**

**Batch rename files:**
```bash
for f in *.txt; do mv "$f" "${f%.txt}.bak"; done
```

**Find and replace in files:**
```bash
grep -rl "old_text" . | xargs sed -i 's/old_text/new_text/g'
```

**Create directory structure:**
```bash
mkdir -p project/{src,bin,lib,docs}
```

**Compare directories:**
```bash
diff -qr dir1/ dir2/
```

---

## **10. Keyboard Shortcuts** ⌨️

### **Bash Shortcuts**

**Navigation:**
```
Ctrl+A          # Beginning of line
Ctrl+E          # End of line
Ctrl+U          # Delete to beginning
Ctrl+K          # Delete to end
Ctrl+W          # Delete word backward
Alt+F           # Forward one word
Alt+B           # Backward one word
```

**Control:**
```
Ctrl+C          # Kill current process
Ctrl+Z          # Suspend process
Ctrl+D          # Logout / EOF
Ctrl+L          # Clear screen (like 'clear')
Ctrl+R          # Reverse search history
Ctrl+S          # Stop output
Ctrl+Q          # Resume output
```

**History:**
```
!!              # Repeat last command
!$              # Last argument of previous command
!^              # First argument of previous command
!*              # All arguments of previous command
```

### **Terminal Multiplexer (tmux)**

```
Ctrl+B D        # Detach session
Ctrl+B C        # Create window
Ctrl+B N        # Next window
Ctrl+B P        # Previous window
Ctrl+B %        # Split vertically
Ctrl+B "        # Split horizontally
Ctrl+B arrows   # Navigate panes
```

### **Vim Quick Reference**

```
i               # Insert mode
Esc             # Normal mode
:w              # Save
:q              # Quit
:wq             # Save and quit
:q!             # Quit without saving
dd              # Delete line
yy              # Copy line
p               # Paste
u               # Undo
Ctrl+R          # Redo
/pattern        # Search
n               # Next match
:%s/old/new/g   # Replace all
```

---

## **Emergency Commands** 🚨

### **System Recovery**

**Kill runaway process:**
```bash
Ctrl+C                       # Interrupt
Ctrl+Z, then kill %1         # Suspend and kill
pkill -9 process-name        # Force kill by name
```

**System too slow:**
```bash
top                          # Find culprit
sudo renice -n 19 -p PID     # Lower priority
pkill -STOP process          # Suspend process
```

**Disk full:**
```bash
df -h                        # Check disk space
du -sh /* | sort -rh | head  # Find large directories
find / -size +100M 2>/dev/null  # Find large files
journalctl --vacuum-time=3d  # Clean old logs
```

**Can't remember command:**
```bash
man -k keyword               # Search man pages
apropos keyword              # Search descriptions
which command                # Find command location
type command                 # Command type/location
```

**Permission denied:**
```bash
sudo !!                      # Repeat with sudo
sudo -i                      # Become root
chmod +x file                # Make executable
```

---

## **Common Patterns** 🔄

### **Check if Command Exists**
```bash
if command -v docker &> /dev/null; then
    echo "Docker is installed"
fi

which docker &> /dev/null && echo "Found" || echo "Not found"
```

### **Loop Through Files**
```bash
for file in *.log; do
    echo "Processing $file"
    gzip "$file"
done
```

### **Check Exit Status**
```bash
if command; then
    echo "Success"
else
    echo "Failed"
    exit 1
fi
```

### **Conditional Execution**bash
```bash
command1 && command2         # Run command2 if command1 succeeds
command1 || command2         # Run command2 if command1 fails
command1 ; command2          # Run both regardless
```

### **Progress Bar**
```bash
while true; do echo -n "."; sleep 1; done &
pid=$!
long_running_command
kill $pid
```

---

## **References** 📚

**Detailed Guides:**
- 📂 [File System Commands](File_System_Commands.md)
- 📝 [Text Processing Commands](Text_Processing_Commands.md)
- 📊 [System Monitoring Commands](System_Monitoring_Commands.md)
- 🌐 [Network Commands](Network_Commands.md)
- ⚙️ [Process Management](Process_Management.md)
- 👥 [User & Permissions Management](User_Permissions_Management.md)
- 📦 [Package Management](Package_Management.md)

**Online Resources:**
- `man command` - Manual pages
- `command --help` - Quick help
- `info command` - Info pages
- [Linux Documentation Project](https://tldp.org)
- [ExplainShell](https://explainshell.com)

---

## **Quick Tips for DevOps** 💼

**Always Backup:**
```bash
cp important.conf important.conf.bak
tar -czf backup-$(date +%Y%m%d).tar.gz /var/www
```

**Use Version Control:**
```bash
git init /etc
git add .
git commit -m "Baseline config"
```

**Document Changes:**
```bash
# Add comments in scripts
# Keep change log
# Use meaningful commit messages
```

**Test in Staging:**
```bash
# Never test in production
# Use identical staging environment
# Automate testing
```

**Automate Everything:**
```bash
# Cron jobs for maintenance
# Scripts for repetitive tasks
# CI/CD pipelines
```

---

## **Summary** ✅

This cheatsheet covers essential Linux commands for DevOps:

- **File System**: Navigation, manipulation, permissions
- **Text Processing**: grep, sed, awk, sorting
- **Monitoring**: Processes, memory, disk, network
- **Network**: Configuration, testing, debugging
- **Processes**: Control, signals, scheduling
- **Users**: Management, permissions, security
- **Packages**: Installation, updates, repositories

**Keep Learning:**
- Practice daily
- Read man pages
- Explore advanced options
- Write scripts to automate
- Share knowledge with team

**Remember:** The best way to learn is by doing. Use these commands regularly, and they'll become second nature!

---

**Happy Linux-ing!** 🐧🚀
