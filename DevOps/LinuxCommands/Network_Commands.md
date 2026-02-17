# **Linux Network Commands – Complete Guide for DevOps** 🌐🚀

Master essential networking commands for troubleshooting connectivity, analyzing traffic, managing services, and ensuring secure communication - critical skills for DevOps engineers managing distributed systems and cloud infrastructure.

---

## **Table of Contents** 📑
1. [Network Fundamentals](#1-network-fundamentals)
2. [Network Configuration](#2-network-configuration)
3. [Connectivity Testing](#3-connectivity-testing)
4. [Network Diagnostics](#4-network-diagnostics)
5. [Port and Socket Management](#5-port-and-socket-management)
6. [DNS and Name Resolution](#6-dns-and-name-resolution)
7. [HTTP and API Testing](#7-http-and-api-testing)
8. [Secure Communication](#8-secure-communication)
9. [Traffic Analysis](#9-traffic-analysis)
10. [Practical DevOps Scenarios](#10-practical-devops-scenarios)
11. [Industry Best Practices](#11-industry-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Network Fundamentals** 🎯

### **OSI Model Quick Reference**
```
Layer 7: Application  (HTTP, FTP, SSH, DNS)
Layer 4: Transport    (TCP, UDP)
Layer 3: Network      (IP, ICMP)
Layer 2: Data Link    (Ethernet, MAC)
Layer 1: Physical     (Cables, Signals)
```

**Common Ports:**
```
22    SSH
80    HTTP
443   HTTPS
3306  MySQL
5432  PostgreSQL
6379  Redis
9200  Elasticsearch
8080  Application servers
```

---

## **2. Network Configuration** ⚙️

### **ifconfig - Network Interface Configuration (Legacy)**
```bash
ifconfig                    # Show all interfaces
ifconfig eth0               # Specific interface
ifconfig eth0 up            # Enable interface
ifconfig eth0 down          # Disable interface
ifconfig eth0 192.168.1.100 netmask 255.255.255.0  # Set IP
```

**Note**: `ifconfig` is deprecated. Use `ip` command instead.

### **ip - Modern Network Configuration**
```bash
# Show interfaces
ip addr                     # All IP addresses
ip addr show eth0           # Specific interface
ip link                     # Link layer information
ip -s link                  # Statistics

# Configure interface
ip addr add 192.168.1.100/24 dev eth0      # Add IP
ip addr del 192.168.1.100/24 dev eth0      # Remove IP
ip link set eth0 up                         # Enable
ip link set eth0 down                       # Disable

# Routing
ip route                    # Show routing table
ip route show               # Same as above
ip route add default via 192.168.1.1        # Add default gateway
ip route add 10.0.0.0/8 via 192.168.1.254  # Add route
ip route del 10.0.0.0/8                    # Delete route

# Neighbor (ARP) table
ip neigh                    # Show ARP cache
ip neigh show dev eth0      # Specific interface
```

**Practical Examples:**
```bash
# Show all IPv4 addresses
ip -4 addr

# Show all IPv6 addresses
ip -6 addr

# Brief output
ip -br addr

# Show only up interfaces
ip link show up

# Flush all IPs from interface
ip addr flush dev eth0
```

### **nmcli - NetworkManager CLI**
```bash
nmcli device                # Show devices
nmcli connection            # Show connections
nmcli connection show       # Detailed connections
nmcli device status         # Device status
nmcli connection up eth0    # Activate connection
nmcli connection down eth0  # Deactivate
nmcli device wifi list      # List WiFi networks

# Get IP address
nmcli -g IP4.ADDRESS device show eth0
```

### **route - Routing Table (Legacy)**
```bash
route -n                    # Show routing table
route add default gw 192.168.1.1  # Add default gateway
route del default gw 192.168.1.1  # Delete gateway
```

**Use `ip route` instead!**

---

## **3. Connectivity Testing** 🔌

### **ping - ICMP Echo Request**
```bash
ping google.com             # Ping host
ping -c 4 8.8.8.8          # Send 4 packets
ping -i 2 example.com       # 2-second interval
ping -s 1000 host           # Packet size 1000 bytes
ping -W 1 host              # 1-second timeout
ping -I eth0 host           # Use specific interface
ping -4 host                # Force IPv4
ping -6 host                # Force IPv6
```

**Continuous Monitoring:**
```bash
ping -c 100 -i 0.2 host > ping-results.txt
```

### **ping6 - IPv6 Ping**
```bash
ping6 google.com
ping6 2001:4860:4860::8888
```

### **traceroute - Trace Network Path**
```bash
traceroute google.com       # Trace route to host
traceroute -n google.com    # No DNS resolution (faster)
traceroute -I google.com    # Use ICMP (requires root)
traceroute -T -p 80 host    # TCP SYN on port 80
traceroute -m 15 host       # Max 15 hops
traceroute -q 1 host        # 1 query per hop (faster)
```

Install traceroute:
```bash
sudo apt install traceroute
```

### **tracepath - Like traceroute (No Root)**
```bash
tracepath google.com
tracepath -n google.com     # No DNS
```

### **mtr - Network Diagnostic Tool**
```bash
mtr google.com              # Interactive traceroute + ping
mtr -r -c 10 google.com     # Report mode, 10 cycles
mtr -n google.com           # No DNS resolution
mtr --tcp -P 443 host       # TCP on port 443
```

Install mtr:
```bash
sudo apt install mtr
```

---

## **4. Network Diagnostics** 🔍

### **netstat - Network Statistics (Legacy)**
```bash
netstat -a                  # All connections
netstat -l                  # Listening ports
netstat -t                  # TCP connections
netstat -u                  # UDP connections
netstat -n                  # Numeric (no DNS)
netstat -p                  # Show PID/Program
netstat -tulpn              # Common combo
netstat -s                  # Statistics by protocol
netstat -r                  # Routing table
netstat -i                  # Interface statistics
```

**Note**: `netstat` is deprecated. Use `ss` instead!

### **ss - Socket Statistics (Modern)**
```bash
ss -a                       # All sockets
ss -l                       # Listening sockets
ss -t                       # TCP connections
ss -u                       # UDP connections
ss -n                       # Numeric (no DNS)
ss -p                       # Show processes
ss -tulpn                   # Common combo
ss -s                       # Summary statistics
ss -o                       # Show timer info
ss -e                       # Extended info
ss -m                       # Memory info
```

**Filtering:**
```bash
ss -t state established     # Established TCP
ss -t state listening       # Listening TCP
ss dst 192.168.1.100        # Destination IP
ss sport = :22              # Source port 22
ss dport = :80              # Destination port 80
ss src 192.168.1.0/24       # Source subnet
ss -t '( dport = :ssh or sport = :ssh )' # SSH connections
```

**Practical Examples:**
```bash
# Count connections by state
ss -tan | awk '{print $1}' | sort | uniq -c

# Show processes using port 80
ss -tlnp | grep :80

# Count established connections
ss -t state established | wc -l

# Show sockets with timers
ss -to

# Find connections to specific IP
ss dst 8.8.8.8
```

### **nmap - Network Scanner**
```bash
nmap 192.168.1.100          # Scan host
nmap 192.168.1.0/24         # Scan subnet
nmap -p 22,80,443 host      # Specific ports
nmap -p 1-1000 host         # Port range
nmap -p- host               # All 65535 ports
nmap -sV host               # Service version detection
nmap -O host                # OS detection
nmap -A host                # Aggressive scan (OS, version, scripts)
nmap -sn 192.168.1.0/24     # Ping scan (no port scan)
nmap --top-ports 100 host   # Top 100 common ports
```

**Advanced Scans:**
```bash
nmap -sS host               # SYN scan (stealth)
nmap -sT host               # TCP connect scan
nmap -sU host               # UDP scan
nmap -Pn host               # Skip ping (assume host up)
nmap -T4 host               # Timing template (0-5, faster)
nmap --script vuln host     # Vulnerability scan
```

Install nmap:
```bash
sudo apt install nmap
```

⚠️ **Warning**: Only scan systems you own or have permission to test!

### **nc (netcat) - Network Swiss Army Knife**
```bash
# Check if port is open
nc -zv google.com 80        # TCP port scan
nc -zvu host 161            # UDP port scan

# Listen on port
nc -l 8080                  # Listen on port 8080
nc -lk 8080                 # Keep listening (multiple connections)

# Connect to port
nc example.com 80           # Connect to HTTP
echo "GET / HTTP/1.0" | nc google.com 80

# Transfer file
# Receiver:
nc -l 9999 > received.txt
# Sender:
nc receiver_ip 9999 < file.txt

# Simple chat
# Host A:
nc -l 9999
# Host B:
nc host_a_ip 9999

# Port forwarding
nc -l 8080 | nc remote_host 80
```

### **tcpdump - Packet Analyzer**
```bash
sudo tcpdump                # Capture all traffic
sudo tcpdump -i eth0        # Specific interface
sudo tcpdump -i any         # All interfaces
sudo tcpdump -n             # No DNS resolution
sudo tcpdump -c 100         # Capture 100 packets
sudo tcpdump -w capture.pcap  # Write to file
sudo tcpdump -r capture.pcap  # Read from file

# Filters
sudo tcpdump host 192.168.1.100           # Specific host
sudo tcpdump src 192.168.1.100            # Source
sudo tcpdump dst 8.8.8.8                  # Destination
sudo tcpdump port 80                      # Port
sudo tcpdump tcp                          # TCP only
sudo tcpdump udp                          # UDP only
sudo tcpdump icmp                         # ICMP only
sudo tcpdump net 192.168.1.0/24           # Network
sudo tcpdump portrange 1-1024             # Port range

# Advanced filters
sudo tcpdump 'tcp[tcpflags] & (tcp-syn) != 0'  # SYN packets
sudo tcpdump 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'  # HTTP with data

# Combinations
sudo tcpdump -i eth0 -nn 'tcp and port 443 and host 1.2.3.4'
```

**Practical Examples:**
```bash
# Capture HTTP traffic
sudo tcpdump -i any -s 0 -A 'tcp port 80'

# Capture specific host conversation
sudo tcpdump -i eth0 host 192.168.1.100 -w host-traffic.pcap

# Monitor DNS queries
sudo tcpdump -i any -s 0 port 53
```

---

## **5. Port and Socket Management** 🔌

### **lsof - List Open Files (Network)**
```bash
lsof -i                     # All network connections
lsof -i TCP                 # TCP only
lsof -i UDP                 # UDP only
lsof -i :80                 # Port 80
lsof -i TCP:22              # TCP port 22
lsof -i @192.168.1.100      # Specific IP
lsof -i -sTCP:LISTEN        # Listening TCP
lsof -i -sTCP:ESTABLISHED   # Established connections
lsof -p 1234                # Specific process
lsof -u username            # Specific user
```

**Practical Examples:**
```bash
# Find what's using port 8080
lsof -i :8080

# Find all network files for nginx
lsof -c nginx -a -i

# Find deleted network sockets
lsof -i | grep deleted

# Count connections per process
lsof -i -n | awk '{print $1}' | sort | uniq -c | sort -rn
```

### **fuser - Identify Process Using Files/Sockets**
```bash
fuser -v 80/tcp             # What's using port 80
fuser -k 8080/tcp           # Kill process on port 8080
fuser -n tcp 443            # Check port 443
```

---

## **6. DNS and Name Resolution** 🏷️

### **host - DNS Lookup**
```bash
host google.com             # Basic lookup
host -t A google.com        # A record
host -t MX google.com       # MX record
host -t NS google.com       # NS record
host -t TXT google.com      # TXT record
host -a google.com          # All records
host 8.8.8.8                # Reverse lookup
```

### **nslookup - DNS Query**
```bash
nslookup google.com         # Basic lookup
nslookup google.com 8.8.8.8 # Use specific DNS server

# Interactive mode
nslookup
> set type=MX
> google.com
> exit
```

### **dig - DNS Investigation**
```bash
dig google.com              # Basic query
dig google.com A            # A record
dig google.com MX           # MX record
dig google.com NS           # NS record
dig google.com TXT          # TXT record
dig @8.8.8.8 google.com     # Use specific DNS
dig +short google.com       # Short output
dig +trace google.com       # Trace DNS path
dig -x 8.8.8.8              # Reverse lookup
dig +noall +answer google.com  # Only answer section
```

**Advanced dig:**
```bash
# Check DNSSEC
dig +dnssec google.com

# Measure query time
dig google.com +stats

# Multiple queries
dig google.com A google.com MX

# Check all record types
dig google.com ANY
```

### **/etc/hosts - Local DNS**
```bash
# View
cat /etc/hosts

# Edit (requires root)
sudo nano /etc/hosts

# Format:
# IP_ADDRESS    HOSTNAME    ALIAS
# 127.0.0.1     localhost
# 192.168.1.100 myserver.local myserver
```

### **/etc/resolv.conf - DNS Servers**
```bash
# View DNS servers
cat /etc/resolv.conf

# Sample content:
# nameserver 8.8.8.8
# nameserver 8.8.4.4
# search example.com
```

### **getent - Get Entries from Administrative Database**
```bash
getent hosts google.com     # Hostname lookup (respects /etc/hosts)
getent ahosts google.com    # All host addresses
getent services ssh         # Service lookup
```

---

## **7. HTTP and API Testing** 📡

### **curl - Transfer Data**
```bash
# GET request
curl https://api.example.com/users

# POST request
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}'

# PUT request
curl -X PUT https://api.example.com/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name":"Jane"}'

# DELETE request
curl -X DELETE https://api.example.com/users/1

# Headers
curl -H "Authorization: Bearer TOKEN" https://api.example.com/data
curl -H "Accept: application/json" https://api.example.com

# Save output
curl -o output.html https://example.com
curl -O https://example.com/file.zip  # Use remote filename

# Follow redirects
curl -L https://example.com

# Include headers in output
curl -i https://example.com

# Show only response headers
curl -I https://example.com
curl --head https://example.com

# Verbose output
curl -v https://example.com

# Timing
curl -w "\nTime: %{time_total}s\n" https://example.com

# Multiple URLs
curl https://example1.com https://example2.com

# Send cookies
curl -b "session=abc123" https://example.com

# Save cookies
curl -c cookies.txt https://example.com

# Upload file
curl -F "file=@/path/to/file" https://example.com/upload

# Basic authentication
curl -u username:password https://api.example.com
```

**Advanced curl:**
```bash
# Custom request method
curl -X PATCH https://api.example.com/users/1

# Send form data
curl -d "name=John&email=john@example.com" https://example.com/form

# Limit rate
curl --limit-rate 1M https://example.com/large-file.zip

# Resume download
curl -C - -O https://example.com/large-file.zip

# Proxy
curl -x proxy:port https://example.com

# Ignore SSL errors (development only!)
curl -k https://self-signed.example.com

# Only show HTTP status code
curl -o /dev/null -s -w "%{http_code}\n" https://example.com

# Detailed timing
curl -w "@curl-format.txt" -o /dev/null -s https://example.com
# curl-format.txt:
#   time_namelookup:  %{time_namelookup}
#   time_connect:  %{time_connect}
#   time_starttransfer:  %{time_starttransfer}
#   time_total:  %{time_total}
```

### **wget - Network Downloader**
```bash
wget https://example.com/file.zip           # Download file
wget -O custom-name.zip https://example.com/file.zip  # Custom filename
wget -c https://example.com/file.zip        # Resume download
wget -b https://example.com/large-file.zip  # Background download
wget -r https://example.com                 # Recursive download
wget -m https://example.com                 # Mirror website
wget --limit-rate=200k https://example.com/file.zip  # Limit speed
wget -q https://example.com/file.zip        # Quiet mode
wget -i urls.txt                            # Download from URL list
wget --user=username --password=pass https://example.com/file.zip
```

### **httpie - User-Friendly HTTP Client**
```bash
http GET https://api.example.com/users
http POST https://api.example.com/users name=John email=john@example.com
http PUT https://api.example.com/users/1 name=Jane
http DELETE https://api.example.com/users/1

# Custom headers
http GET https://api.example.com/data Authorization:"Bearer TOKEN"

# Download
http --download https://example.com/file.zip

# Form submission
http --form POST https://example.com/upload file@/path/to/file

# JSON by default
http POST https://api.example.com/users name=John age:=30  # age as number
```

Install httpie:
```bash
sudo apt install httpie
# or
pip install httpie
```

---

## **8. Secure Communication** 🔐

### **ssh - Secure Shell**
```bash
ssh user@host               # Connect to host
ssh -p 2222 user@host       # Custom port
ssh -i ~/.ssh/id_rsa user@host  # Specific key
ssh -v user@host            # Verbose (debug)
ssh -L 8080:localhost:80 user@host  # Local port forwarding
ssh -R 8080:localhost:80 user@host  # Remote port forwarding
ssh -D 1080 user@host       # SOCKS proxy
ssh -N -L 3306:db-server:3306 user@jump-host  # Tunnel without shell
ssh -J jump-host user@target-host  # Jump host (proxy)
```

**SSH Config (~/.ssh/config):**
```
Host myserver
    HostName example.com
    User myuser
    Port 2222
    IdentityFile ~/.ssh/mykey
    ForwardAgent yes

Host * !proxy
    ProxyJump proxy
```

### **scp - Secure Copy**
```bash
scp file.txt user@host:/path/          # Copy to remote
scp user@host:/path/file.txt .         # Copy from remote
scp -r directory/ user@host:/path/     # Copy directory
scp -P 2222 file.txt user@host:/path/  # Custom port
scp -i key.pem file.txt user@host:/    # Specific key
scp -C file.txt user@host:/            # Compress during transfer
```

**Between remote hosts:**
```bash
scp user1@host1:/file user2@host2:/destination
```

### **sftp - Secure FTP**
```bash
sftp user@host
# Interactive commands:
# put local.txt        # Upload
# get remote.txt       # Download
# ls                   # List remote
# lls                  # List local
# cd /path             # Change remote dir
# lcd /path            # Change local dir
# mkdir newdir         # Create remote dir
# rm file.txt          # Delete remote file
# bye                  # Exit
```

### **rsync - Efficient File Transfer**
```bash
rsync -av source/ dest/              # Archive mode, verbose
rsync -avz source/ user@host:dest/   # With compression
rsync -avz --delete source/ dest/    # Delete in dest
rsync -avz -e "ssh -p 2222" source/ user@host:dest/  # Custom SSH port
rsync -avz --progress source/ dest/  # Show progress
rsync -avz --exclude='*.log' source/ dest/  # Exclude files
rsync -avz --dry-run source/ dest/   # Test run
```

**Practical Examples:**
```bash
# Backup directory
rsync -avz --delete /var/www/ backup@server:/backups/www/

# Sync only specific files
rsync -avz --include='*.conf' --exclude='*' source/ dest/

# Bandwidth limit
rsync -avz --bwlimit=1000 source/ dest/  # 1000 KB/s

# Resume interrupted transfer
rsync -avz --partial source/ dest/
```

### **openssl - SSL/TLS Testing**
```bash
# Test SSL connection
openssl s_client -connect example.com:443

# Check certificate
openssl s_client -connect example.com:443 -showcerts

# Check certificate expiry
openssl s_client -connect example.com:443 </dev/null 2>/dev/null | openssl x509 -noout -dates

# Test specific TLS version
openssl s_client -connect example.com:443 -tls1_2

# Check supported ciphers
openssl s_client -connect example.com:443 -cipher 'ALL'

# Verify certificate
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -text
```

---

## **9. Traffic Analysis** 📈

### **iftop - Network Bandwidth Monitor**
```bash
sudo iftop                  # Monitor bandwidth
sudo iftop -i eth0          # Specific interface
sudo iftop -n               # No DNS resolution
sudo iftop -P               # Show ports
sudo iftop -F 192.168.1.0/24  # Filter subnet
```

Install iftop:
```bash
sudo apt install iftop
```

### **nethogs - Bandwidth by Process**
```bash
sudo nethogs                # Monitor bandwidth per process
sudo nethogs eth0           # Specific interface
```

Install nethogs:
```bash
sudo apt install nethogs
```

### **vnstat - Network Statistics**
```bash
vnstat                      # Summary
vnstat -l                   # Live traffic
vnstat -h                   # Hourly statistics
vnstat -d                   # Daily statistics
vnstat -m                   # Monthly statistics
vnstat -t                   # Top 10 days
vnstat -i eth0              # Specific interface
```

Install and setup:
```bash
sudo apt install vnstat
sudo systemctl enable vnstat
sudo systemctl start vnstat
```

### **bmon - Bandwidth Monitor**
```bash
bmon                        # Monitor interfaces
bmon -p eth0                # Specific interface
```

Install bmon:
```bash
sudo apt install bmon
```

---

## **10. Practical DevOps Scenarios** 🛠️

### **Scenario 1: Service Not Accessible**
```bash
# 1. Check if service is listening
ss -tlnp | grep :80
lsof -i :80

# 2. Test locally
curl localhost:80

# 3. Check firewall
sudo iptables -L -n
sudo ufw status

# 4. Test connectivity
ping server-ip
telnet server-ip 80
nc -zv server-ip 80

# 5. Trace route
traceroute server-ip
mtr server-ip

# 6. Check DNS
dig example.com
nslookup example.com

# 7. Check from different location
curl -I https://example.com
```

### **Scenario 2: High Network Traffic**
```bash
# 1. Monitor bandwidth
sudo iftop -i eth0
sudo nethogs eth0

# 2. Find top connections
ss -tan state established | awk '{print $4}' | cut -d':' -f1 | sort | uniq -c | sort -rn | head

# 3. Analyze traffic
sudo tcpdump -i eth0 -nn -c 1000 | awk '{print $3}' | sort | uniq -c | sort -rn | head

# 4. Check specific process
lsof -i -n | grep -i established | awk '{print $1}' | sort | uniq -c | sort -rn

# 5. Historical data
vnstat -d
sar -n DEV 1 10
```

### **Scenario 3: Slow Network Performance**
```bash
# 1. Test latency
ping -c 100 server > ping-results.txt
awk '/time=/ {sum+=$7; count++} END {print "Avg:", sum/count}' ping-results.txt

# 2. Test bandwidth
iperf3 -c server-ip
wget --output-document=/dev/null http://speedtest/file

# 3. Check packet loss
mtr --report --report-cycles 100 server-ip

# 4. Analyze connections
ss -ti

# 5. Check MTU
ip link show eth0
ping -M do -s 1472 server  # Test MTU
```

### **Scenario 4: Connection Refused**
```bash
# 1. Verify port is open
nmap -p 80 server-ip

# 2. Check if service is running
ssh user@server "systemctl status nginx"

# 3. Verify firewall
ssh user@server "sudo iptables -L -n | grep 80"

# 4. Test local connectivity on server
ssh user@server "curl localhost:80"

# 5. Check SELinux/AppArmor
ssh user@server "getenforce"
ssh user@server "sudo aa-status"
```

### **Scenario 5: DNS Issues**
```bash
# 1. Test DNS resolution
dig example.com
host example.com

# 2. Check DNS servers
cat /etc/resolv.conf

# 3. Test different DNS
dig @8.8.8.8 example.com
dig @1.1.1.1 example.com

# 4. Trace DNS path
dig +trace example.com

# 5. Clear DNS cache
sudo systemd-resolve --flush-caches
# or
sudo /etc/init.d/nscd restart

# 6. Test with getent (respects /etc/hosts)
getent hosts example.com
```

### **Scenario 6: SSL/TLS Certificate Issues**
```bash
# 1. Check certificate
openssl s_client -connect example.com:443 -servername example.com

# 2. View certificate details
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -text

# 3. Check expiry
echo | openssl s_client -connect example.com:443 2>/dev/null | openssl x509 -noout -dates

# 4. Verify certificate chain
openssl s_client -connect example.com:443 -showcerts

# 5. Test specific TLS version
openssl s_client -connect example.com:443 -tls1_2
```

---

## **11. Industry Best Practices** 🏆

### **Network Monitoring Strategy**

**1. Continuous Monitoring:**
```bash
# Network health check script
#!/bin/bash
check_connectivity() {
    if ping -c 1 -W 1 8.8.8.8 > /dev/null 2>&1; then
        echo "Internet: OK"
    else
        echo "Internet: FAIL"
    fi
}

check_dns() {
    if host google.com > /dev/null 2>&1; then
        echo "DNS: OK"
    else
        echo "DNS: FAIL"
    fi
}

check_port() {
    local port=$1
    if ss -tlnp | grep -q ":$port "; then
        echo "Port $port: LISTENING"
    else
        echo "Port $port: NOT LISTENING"
    fi
}

check_connectivity
check_dns
check_port 80
check_port 443
```

**2. Firewall Configuration:**
```bash
# Check firewall rules
sudo iptables -L -n -v

# Allow specific port
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.168.1.0/24 to any port 22

# Enable firewall
sudo ufw enable
```

**3. Network Performance Baseline:**
```bash
# Create baseline
{
    echo "=== Network Baseline $(date) ==="
    echo "=== Interfaces ==="
    ip -br addr
    echo "=== Routing ==="
    ip route
    echo "=== Listening Ports ==="
    ss -tlnp
    echo "=== Bandwidth ==="
    vnstat -h
} > network-baseline-$(date +%Y%m%d).txt
```

**4. Security Scanning:**
```bash
# Port scan your own servers
nmap -sV -O your-server.com

# Check for open ports
ss -tlnp

# Monitor failed connections
journalctl -u sshd | grep "Failed"
```

**5. Bandwidth Management:**
```bash
# Install traffic control tools
sudo apt install iproute2

# Limit bandwidth (example)
sudo tc qdisc add dev eth0 root tbf rate 1mbit burst 32kbit latency 400ms

# Remove limit
sudo tc qdisc del dev eth0 root
```

### **Troubleshooting Checklist**

```
□ Physical Layer: Cable connected? Link light on?
□ Network Layer: IP configured? Can ping gateway?
□ DNS: Can resolve hostnames?
□ Routing: Correct routing table?
□ Firewall: Ports open?
□ Service: Running and listening?
□ Application: Logs showing errors?
```

---

## **12. Interview Cheat Sheet** 📝

### **Quick Commands**

```bash
# Configuration
ip addr                      # Show IP addresses
ip route                     # Show routes
ss -tulpn                    # Listening ports

# Connectivity
ping host                    # Test reachability
traceroute host              # Trace path
mtr host                     # Combined ping+trace

# DNS
dig example.com              # DNS lookup
host example.com             # Simple lookup
nslookup example.com         # DNS query

# Diagnostics
ss -tan                      # TCP connections
lsof -i :80                  # What's on port 80
tcpdump -i eth0              # Capture packets

# HTTP
curl -I https://example.com  # HTTP headers
wget https://example.com/file  # Download file

# Secure
ssh user@host                # Secure shell
scp file user@host:/path     # Secure copy
rsync -avz src/ dest/        # Sync files

# Monitoring
iftop                        # Bandwidth monitor
nethogs                      # Per-process bandwidth
vnstat                       # Network statistics
```

### **Common Interview Questions**

**Q1: How to check if a port is open?**
```bash
telnet host port
nc -zv host port
nmap -p port host
ss -tlnp | grep :port
```

**Q2: How to find which process is using a port?**
```bash
lsof -i :8080
ss -tlnp | grep :8080
fuser -v 8080/tcp
```

**Q3: Difference between TCP and UDP?**
- **TCP**: Connection-oriented, reliable, ordered, slower (HTTP, SSH, FTP)
- **UDP**: Connectionless, unreliable, faster (DNS, DHCP, streaming)

**Q4: How to test DNS resolution?**
```bash
dig example.com
host example.com
nslookup example.com
getent hosts example.com  # Respects /etc/hosts
```

**Q5: How to capture network traffic?**
```bash
sudo tcpdump -i any -w capture.pcap
sudo tcpdump port 80 -A
```

**Q6: How to check network connectivity?**
```bash
ping 8.8.8.8              # Internet
ping gateway              # Local network
ping localhost            # Loopback
traceroute example.com    # Path to destination
```

**Q7: What's the difference between netstat and ss?**
- **ss**: Modern, faster, more features
- **netstat**: Legacy, deprecated
- **ss** reads directly from kernel, **netstat** reads from /proc

**Q8: How to check routing table?**
```bash
ip route
route -n                  # Legacy
netstat -rn              # Legacy
```

---

## **Summary** ✅

Network commands are essential for DevOps engineers:

1. **Configuration**: `ip`, `ifconfig`, `nmcli`
2. **Connectivity**: `ping`, `traceroute`, `mtr`
3. **Diagnostics**: `ss`, `netstat`, `nmap`, `tcpdump`
4. **DNS**: `dig`, `host`, `nslookup`
5. **HTTP**: `curl`, `wget`
6. **Security**: `ssh`, `scp`, `rsync`, `openssl`
7. **Monitoring**: `iftop`, `nethogs`, `vnstat`

**Key Principles:**
- Use modern tools (ip, ss) over legacy (ifconfig, netstat)
- Always check multiple layers (physical, network, application)
- Monitor bandwidth and establish baselines
- Secure all communications
- Document network configurations

---

**Next Topics**: [Process Management](Process_Management.md) | [User & Permissions](User_Permissions_Management.md)
