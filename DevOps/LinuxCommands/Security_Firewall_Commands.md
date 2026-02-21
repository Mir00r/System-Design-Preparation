# **Linux Security & Firewall Commands** 🔒

**Complete guide for system security, firewall management, and hardening**

---

## **Table of Contents** 📑
1. [Firewall Basics](#1-firewall-basics)
2. [iptables - Traditional Firewall](#2-iptables---traditional-firewall)
3. [firewalld - Dynamic Firewall](#3-firewalld---dynamic-firewall)
4. [ufw - Uncomplicated Firewall](#4-ufw---uncomplicated-firewall)
5. [SELinux - Security-Enhanced Linux](#5-selinux---security-enhanced-linux)
6. [AppArmor](#6-apparmor)
7. [Security Scanning & Auditing](#7-security-scanning--auditing)
8. [SSL/TLS Certificates](#8-ssltls-certificates)
9. [DevOps Security Practices](#9-devops-security-practices)
10. [Troubleshooting](#10-troubleshooting)
11. [Best Practices](#11-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Firewall Basics** 🛡️

### **Understanding Firewalls**

```
Firewall Types:

1. Packet Filtering (iptables)
   - Inspects individual packets
   - Based on IP, port, protocol
   
2. Stateful Inspection
   - Tracks connection state
   - More secure than packet filtering
   
3. Application-Level
   - Deep packet inspection
   - Application-aware filtering
```

### **Check Firewall Status**

```bash
# Check which firewall is active
sudo systemctl status firewalld   # firewalld (RHEL 7+)
sudo systemctl status ufw          # ufw (Ubuntu)
sudo iptables -L -n               # iptables (traditional)

# Check if firewall is enabled
sudo firewall-cmd --state          # firewalld
sudo ufw status                    # ufw
```

---

## **2. iptables - Traditional Firewall** 🔧

### **iptables Basics**

```bash
# View rules
sudo iptables -L                   # List all rules
sudo iptables -L -n                # Numeric (no DNS resolution)
sudo iptables -L -v                # Verbose (packet counts)
sudo iptables -L -n -v --line-numbers  # With line numbers

# View specific chain
sudo iptables -L INPUT
sudo iptables -L OUTPUT
sudo iptables -L FORWARD

# View NAT rules
sudo iptables -t nat -L -n -v
```

### **iptables Rules**

```bash
# Allow incoming SSH (port 22)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow incoming HTTP (port 80)
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT

# Allow incoming HTTPS (port 443)
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow from specific IP
sudo iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# Allow from subnet
sudo iptables -A INPUT -s 192.168.1.0/24 -j ACCEPT

# Block specific IP
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# Block specific port
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP

# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT
```

### **Default Policies**

```bash
# Set default policies
sudo iptables -P INPUT DROP        # Drop all incoming by default
sudo iptables -P FORWARD DROP      # Drop all forwarding
sudo iptables -P OUTPUT ACCEPT     # Allow all outgoing

# Basic secure server setup
sudo iptables -F                   # Flush all rules
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
```

### **Delete and Insert Rules**

```bash
# Delete rule by number
sudo iptables -L INPUT --line-numbers
sudo iptables -D INPUT 3           # Delete rule #3

# Delete rule by specification
sudo iptables -D INPUT -p tcp --dport 8080 -j ACCEPT

# Insert rule at specific position
sudo iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT

# Replace rule
sudo iptables -R INPUT 1 -p tcp --dport 22 -j DROP
```

### **Save and Restore iptables**

```bash
# Save rules (Debian/Ubuntu)
sudo iptables-save > /etc/iptables/rules.v4
sudo netfilter-persistent save

# Save rules (RHEL/CentOS)
sudo service iptables save
sudo iptables-save > /etc/sysconfig/iptables

# Restore rules
sudo iptables-restore < /etc/iptables/rules.v4

# Auto-restore on boot (Debian/Ubuntu)
sudo apt install iptables-persistent
```

### **Advanced iptables**

```bash
# Rate limiting (prevent DDoS)
sudo iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute \
  --limit-burst 100 -j ACCEPT

# Block ping (ICMP)
sudo iptables -A INPUT -p icmp --icmp-type echo-request -j DROP

# Allow ping from specific network
sudo iptables -A INPUT -p icmp -s 192.168.1.0/24 -j ACCEPT

# Port forwarding
sudo iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080

# SNAT (Source NAT)
sudo iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to-source 203.0.113.1

# MASQUERADE (dynamic SNAT)
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Log dropped packets
sudo iptables -A INPUT -j LOG --log-prefix "iptables-dropped: "
sudo iptables -A INPUT -j DROP
```

---

## **3. firewalld - Dynamic Firewall** 🔥

### **firewalld Basics**

```bash
# Start/stop/status
sudo systemctl start firewalld
sudo systemctl stop firewalld
sudo systemctl status firewalld
sudo systemctl enable firewalld    # Enable on boot

# Check state
sudo firewall-cmd --state

# Reload firewall
sudo firewall-cmd --reload
sudo firewall-cmd --complete-reload  # More thorough
```

### **Zones**

```bash
# List zones
sudo firewall-cmd --get-zones
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --get-default-zone

# Set default zone
sudo firewall-cmd --set-default-zone=public

# Zone information
sudo firewall-cmd --zone=public --list-all
sudo firewall-cmd --list-all-zones

# Add interface to zone
sudo firewall-cmd --zone=public --add-interface=eth0 --permanent
```

### **Services**

```bash
# List available services
sudo firewall-cmd --get-services

# List enabled services
sudo firewall-cmd --list-services

# Add service
sudo firewall-cmd --add-service=http           # Temporary
sudo firewall-cmd --add-service=http --permanent  # Permanent
sudo firewall-cmd --reload                     # Apply permanent

# Add multiple services
sudo firewall-cmd --add-service={http,https,ssh} --permanent

# Remove service
sudo firewall-cmd --remove-service=http --permanent

# Common services
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-service=https --permanent
sudo firewall-cmd --add-service=mysql --permanent
```

### **Ports**

```bash
# Add port
sudo firewall-cmd --add-port=8080/tcp          # Temporary
sudo firewall-cmd --add-port=8080/tcp --permanent  # Permanent

# Add port range
sudo firewall-cmd --add-port=5000-5100/tcp --permanent

# Remove port
sudo firewall-cmd --remove-port=8080/tcp --permanent

# List ports
sudo firewall-cmd --list-ports
```

### **Rich Rules**

```bash
# Allow from specific IP
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.100" accept' --permanent

# Allow specific IP to port
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="22" protocol="tcp" accept' --permanent

# Block specific IP
sudo firewall-cmd --add-rich-rule='rule family="ipv4" source address="192.168.1.100" reject' --permanent

# Rate limiting
sudo firewall-cmd --add-rich-rule='rule service name="ssh" limit value="10/m" accept' --permanent

# List rich rules
sudo firewall-cmd --list-rich-rules
```

### **Port Forwarding**

```bash
# Forward port 80 to 8080
sudo firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080 --permanent

# Forward to different host
sudo firewall-cmd --add-forward-port=port=80:proto=tcp:toaddr=192.168.1.100:toport=8080 --permanent

# Enable masquerading (required for forwarding)
sudo firewall-cmd --add-masquerade --permanent
```

---

## **4. ufw - Uncomplicated Firewall** 🐧

### **ufw Basics**

```bash
# Enable/disable
sudo ufw enable
sudo ufw disable
sudo ufw status
sudo ufw status verbose
sudo ufw status numbered         # With rule numbers

# Reset to default
sudo ufw reset
```

### **Default Policies**

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw default deny routed
```

### **Allow/Deny Rules**

```bash
# Allow services
sudo ufw allow ssh
sudo ufw allow http
sudo ufw allow https

# Allow specific port
sudo ufw allow 22
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow port range
sudo ufw allow 6000:6007/tcp
sudo ufw allow 6000:6007/udp

# Allow from specific IP
sudo ufw allow from 192.168.1.100
sudo ufw allow from 192.168.1.0/24

# Allow from IP to specific port
sudo ufw allow from 192.168.1.100 to any port 22
sudo ufw allow from 192.168.1.0/24 to any port 3306

# Deny
sudo ufw deny 23                 # Deny telnet
sudo ufw deny from 192.168.1.100
```

### **Delete Rules**

```bash
# Delete by rule number
sudo ufw status numbered
sudo ufw delete 2

# Delete by specification
sudo ufw delete allow 80/tcp
sudo ufw delete allow from 192.168.1.100
```

### **Advanced ufw**

```bash
# Allow outgoing to specific port
sudo ufw allow out 25/tcp        # SMTP outgoing

# Limit (rate limiting)
sudo ufw limit ssh               # Max 6 connections per 30 seconds

# Application profiles
sudo ufw app list
sudo ufw allow 'Nginx Full'
sudo ufw allow 'OpenSSH'

# Logging  
sudo ufw logging on
sudo ufw logging off
sudo ufw logging low
sudo ufw logging medium
sudo ufw logging high

# Insert rule at position
sudo ufw insert 1 allow from 192.168.1.100
```

---

## **5. SELinux - Security-Enhanced Linux** 🔐

### **SELinux Status**

```bash
# Check SELinux status
sestatus
getenforce                       # Current mode

# SELinux modes:
# Enforcing - SELinux policy enforced
# Permissive - SELinux policy not enforced (logs violations)
# Disabled - SELinux disabled

# Set mode temporarily
sudo setenforce 0                # Permissive
sudo setenforce 1                # Enforcing

# Set mode permanently
sudo nano /etc/selinux/config
# SELINUX=enforcing
# SELINUX=permissive
# SELINUX=disabled
```

### **SELinux Contexts**

```bash
# View file context
ls -Z file.txt
ls -Zd /var/www/html

# View process context
ps -eZ
ps -eZ | grep httpd

# View user context
id -Z

# Change file context
sudo chcon -t httpd_sys_content_t /var/www/html/index.html

# Restore default context
sudo restorecon -v /var/www/html/index.html
sudo restorecon -Rv /var/www/html  # Recursive
```

### **SELinux Booleans**

```bash
# List booleans
getsebool -a
getsebool -a | grep httpd

# Get specific boolean
getsebool httpd_can_network_connect

# Set boolean temporarily
sudo setsebool httpd_can_network_connect on

# Set boolean permanently
sudo setsebool -P httpd_can_network_connect on

# Common booleans
sudo setsebool -P httpd_can_network_connect on      # Apache network access
sudo setsebool -P httpd_can_network_connect_db on   # Apache database access
sudo setsebool -P httpd_use_nfs on                  # Apache use NFS
```

### **SELinux Troubleshooting**

```bash
# View SELinux denials
sudo ausearch -m avc -ts recent
sudo ausearch -m avc -ts today

# Use audit2why
sudo grep AVC /var/log/audit/audit.log | audit2why

# Generate allow policy
sudo grep httpd /var/log/audit/audit.log | audit2allow -M mypolicy
sudo semodule -i mypolicy.pp

# Install troubleshooting tools
sudo yum install setroubleshoot setools
```

---

## **6. AppArmor** 🛡️

### **AppArmor Basics**

```bash
# Check status
sudo apparmor_status
sudo aa-status

# AppArmor modes:
# enforce - AppArmor policy enforced
# complain - Policy not enforced (logs violations)
# unconfined - No AppArmor profile

# List profiles
sudo aa-status
```

### **Profile Management**

```bash
# Set profile to complain mode
sudo aa-complain /etc/apparmor.d/usr.bin.firefox

# Set profile to enforce mode
sudo aa-enforce /etc/apparmor.d/usr.bin.firefox

# Disable profile
sudo ln -s /etc/apparmor.d/usr.bin.firefox /etc/apparmor.d/disable/
sudo apparmor_parser -R /etc/apparmor.d/usr.bin.firefox

# Enable profile
sudo rm /etc/apparmor.d/disable/usr.bin.firefox
sudo apparmor_parser -r /etc/apparmor.d/usr.bin.firefox

# Reload all profiles
sudo systemctl reload apparmor
```

### **Create AppArmor Profile**

```bash
# Generate profile
sudo aa-genprof /usr/bin/myapp

# Update profile
sudo aa-logprof

# Edit profile manually
sudo nano /etc/apparmor.d/usr.bin.myapp

# Example profile:
#include <tunables/global>

/usr/bin/myapp {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  /usr/bin/myapp mr,
  /etc/myapp/** r,
  /var/log/myapp/* w,
  /tmp/** rw,
}
```

---

## **7. Security Scanning & Auditing** 🔍

### **Lynis - Security Auditing**

```bash
# Install Lynis
sudo apt install lynis           # Debian/Ubuntu
sudo yum install lynis           # RHEL/CentOS

# Run security audit
sudo lynis audit system

# Specific tests
sudo lynis show options
sudo lynis audit system --tests-from-group malware
sudo lynis audit system --tests-from-group authentication

# View report
sudo cat /var/log/lynis.log
sudo cat /var/log/lynis-report.dat
```

### **rkhunter - Rootkit Hunter**

```bash
# Install
sudo apt install rkhunter

# Update database
sudo rkhunter --update

# Run scan
sudo rkhunter --check
sudo rkhunter --check --skip-keypress  # Non-interactive

# View log
sudo cat /var/log/rkhunter.log
```

### **ClamAV - Antivirus**

```bash
# Install ClamAV
sudo apt install clamav clamav-daemon

# Update virus database
sudo freshclam

# Scan directory
sudo clamscan -r /home
sudo clamscan -r --infected --remove /home  # Remove infected files

# Scan and log
sudo clamscan -r -i -l /var/log/clamav-scan.log /home
```

### **AIDE - File Integrity**

```bash
# Install AIDE
sudo apt install aide

# Initialize database
sudo aideinit
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db

# Run check
sudo aide --check

# Update database
sudo aide --update
sudo mv /var/lib/aide/aide.db.new /var/lib/aide/aide.db
```

---

## **8. SSL/TLS Certificates** 🔑

### **OpenSSL Commands**

```bash
# Generate private key
openssl genrsa -out private.key 2048
openssl genrsa -out private.key 4096  # 4096-bit

# Generate CSR (Certificate Signing Request)
openssl req -new -key private.key -out request.csr

# Generate self-signed certificate
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365

# View certificate
openssl x509 -in cert.pem -text -noout

# Check certificate expiry
openssl x509 -enddate -noout -in cert.pem

# Verify certificate
openssl verify cert.pem

# Test SSL connection
openssl s_client -connect example.com:443
```

### **Let's Encrypt (Certbot)**

```bash
# Install Certbot
sudo apt install certbot
sudo apt install python3-certbot-nginx   # Nginx plugin
sudo apt install python3-certbot-apache  # Apache plugin

# Obtain certificate (Nginx)
sudo certbot --nginx -d example.com -d www.example.com

# Obtain certificate (Apache)
sudo certbot --apache -d example.com

# Obtain certificate (standalone)
sudo certbot certonly --standalone -d example.com

# List certificates
sudo certbot certificates

# Renew certificates
sudo certbot renew
sudo certbot renew --dry-run  # Test renewal

# Auto-renewal (cron)
sudo crontab -e
0 0 * * * /usr/bin/certbot renew --quiet
```

---

## **9. DevOps Security Practices** 🚀

### **Secure SSH**

```bash
# Harden SSH (/etc/ssh/sshd_config)
Port 2222                            # Change default port
PermitRootLogin no                   # Disable root login
PasswordAuthentication no            # Only key-based auth
PubkeyAuthentication yes
MaxAuthTries 3
MaxSessions 2
AllowUsers deploy admin              # Whitelist users
LoginGraceTime 60

# Restart SSH
sudo systemctl restart sshd
```

### **Fail2Ban - Intrusion Prevention**

```bash
# Install Fail2Ban
sudo apt install fail2ban

# Configure
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local

# Example config:
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600

# Start Fail2Ban
sudo systemctl start fail2ban
sudo systemctl enable fail2ban

# Check status
sudo fail2ban-client status
sudo fail2ban-client status sshd

# Unban IP
sudo fail2ban-client set sshd unbanip 192.168.1.100
```

### **Security Hardening Script**

```bash
#!/bin/bash
# security-hardening.sh

# Update system
sudo apt update && sudo apt upgrade -y

# Install security tools
sudo apt install -y fail2ban ufw aide rkhunter

# Configure firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable

# Harden SSH
sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo systemctl restart sshd

# Enable automatic updates
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

echo "Security hardening completed!"
```

---

## **10. Troubleshooting** 🔧

### **Firewall Issues**

```bash
# Test if port is open
telnet hostname 80
nc -zv hostname 80
curl -v telnet://hostname:80

# Check if service is listening
sudo netstat -tuln | grep ':80'
sudo ss -tuln | grep ':80'
sudo lsof -i :80

# Temporarily disable firewall
sudo systemctl stop firewalld
sudo ufw disable

# Check firewall logs
sudo tail -f /var/log/firewalld
sudo grep UFW /var/log/syslog
sudo journalctl -u firewalld -f
```

### **SELinux Issues**

```bash
# Check for denials
sudo ausearch -m avc -ts recent
sudo grep AVC /var/log/audit/audit.log

# Temporarily disable SELinux
sudo setenforce 0

# If it works, create policy
sudo grep httpd /var/log/audit/audit.log | audit2allow -M mypolicy
sudo semodule -i mypolicy.pp
sudo setenforce 1
```

---

## **11. Best Practices** ⭐

### **Do's**

```
✅ Enable firewall on all servers
✅ Use fail2ban for intrusion prevention
✅ Disable root SSH login
✅ Use SSH keys instead of passwords
✅ Keep systems updated
✅ Regular security audits (Lynis)
✅ Monitor logs for suspicious activity
✅ Use strong passwords (or better, no passwords)
✅ Implement least privilege principle
✅ Use HTTPS/TLS for all web services
```

### **Don'ts**

```
❌ Don't expose unnecessary ports
❌ Don't use default ports (22, 3306, etc.)
❌ Don't disable firewall
❌ Don't ignore security updates
❌ Don't run services as root
❌ Don't store passwords in plain text
❌ Don't disable SELinux/AppArmor without good reason
❌ Don't trust user input
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: iptables vs firewalld vs ufw?**
```
iptables:
- Low-level, powerful
- Static rules
- RHEL 6 and earlier
- Manual configuration

firewalld:
- Dynamic (no restart needed)
- Zone-based
- RHEL 7+ default
- Runtime and permanent rules

ufw:
- User-friendly
- Ubuntu default
- Simplified iptables
- Easy for beginners

DevOps choice:
- Modern RHEL → firewalld
- Ubuntu → ufw
- Advanced control → iptables
```

### **Q2: How to allow port 8080 on firewall?**
```
iptables:
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT
sudo iptables-save > /etc/iptables/rules.v4

firewalld:
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload

ufw:
sudo ufw allow 8080/tcp
```

### **Q3: SELinux enforcing vs permissive?**
```
Enforcing:
- SELinux policies enforced
- Violations blocked
- Production mode

Permissive:
- SELinux policies NOT enforced
- Violations logged only
- Troubleshooting mode

Disabled:
- SELinux completely disabled
- Not recommended

Check mode:
getenforce

Set temporarily:
sudo setenforce 0  # Permissive
sudo setenforce 1  # Enforcing

Set permanently:
/etc/selinux/config → SELINUX=enforcing
```

### **Q4: Secure a Linux server checklist?**
```
1. Firewall:
   - Enable ufw/firewalld
   - Allow only required ports
   - Default deny incoming

2. SSH:
   - Change default port
   - Disable root login
   - Key-based auth only
   - Install fail2ban

3. Updates:
   - apt update && apt upgrade
   - Enable auto-updates

4. Users:
   - Remove unnecessary users
   - Strong password policy
   - Use sudo, not root

5. Services:
   - Disable unused services
   - Principle of least privilege
   
6. Monitoring:
   - Enable logging
   - Monitor auth.log
   - Set up alerts

7. Audit:
   - Run Lynis
   - Check for rootkits
   - File integrity (AIDE)
```

### **Q5: Common security commands?**
```
Firewall:
sudo ufw enable
sudo ufw allow 80/tcp
sudo firewall-cmd --add-service=http --permanent

SSH:
ssh-keygen -t rsa -b 4096
ssh-copy-id user@host

Updates:
sudo apt update && sudo apt upgrade
sudo yum update

Auditing:
sudo lynis audit system
sudo rkhunter --check
sudo fail2ban-client status

Logs:
sudo tail -f /var/log/auth.log
sudo ausearch -m avc -ts recent

SELinux:
getenforce
sudo setsebool -P httpd_can_network_connect on
```

---

**🔒 Master Security & Firewalls for Robust System Protection!**

*Security is not optional - it's essential for every production system.*
