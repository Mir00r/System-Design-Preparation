# **SSH & Remote Management Commands** 🔐

**Complete guide for secure remote access and server management**

---

## **Table of Contents** 📑
1. [SSH Basics](#1-ssh-basics)
2. [SSH Key Management](#2-ssh-key-management)
3. [SSH Configuration](#3-ssh-configuration)
4. [Secure File Transfer](#4-secure-file-transfer)
5. [SSH Tunneling & Port Forwarding](#5-ssh-tunneling--port-forwarding)
6. [Remote Command Execution](#6-remote-command-execution)
7. [SSH Agent & Key Agent Forwarding](#7-ssh-agent--key-agent-forwarding)
8. [Advanced SSH Techniques](#8-advanced-ssh-techniques)
9. [Remote Desktop & VNC](#9-remote-desktop--vnc)
10. [DevOps Use Cases](#10-devops-use-cases)
11. [Troubleshooting](#11-troubleshooting)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. SSH Basics** 🔑

### **What is SSH?**

**SSH (Secure Shell)** is a cryptographic network protocol for secure remote login and command execution over unsecured networks. It provides encrypted communication between client and server.

### **Basic SSH Connection**

```bash
# Connect to remote server
ssh user@hostname
ssh user@192.168.1.100
ssh user@example.com

# Connect on specific port (default: 22)
ssh -p 2222 user@hostname

# Connect with verbose output (debugging)
ssh -v user@hostname          # Verbose
ssh -vv user@hostname         # More verbose
ssh -vvv user@hostname        # Maximum verbosity

# Connect and execute command
ssh user@hostname 'ls -la'
ssh user@hostname 'uptime'

# Interactive login with specific user
ssh -l username hostname
```

### **First Time Connection**

```bash
# First connection shows fingerprint
ssh user@newserver
# The authenticity of host 'newserver (192.168.1.100)' can't be established.
# ECDSA key fingerprint is SHA256:abc123...
# Are you sure you want to continue connecting (yes/no)?

# Server fingerprint added to
~/.ssh/known_hosts

# Verify fingerprint before accepting
ssh-keygen -lf /etc/ssh/ssh_host_ecdsa_key.pub  # On server
```

### **SSH Connection Options**

```bash
# Disable strict host key checking (not recommended for production)
ssh -o StrictHostKeyChecking=no user@host

# Disable password authentication
ssh -o PasswordAuthentication=no user@host

# Use specific identity file
ssh -i ~/.ssh/mykey user@host

# Keep connection alive
ssh -o ServerAliveInterval=60 user@host

# Disable pseudo-terminal allocation
ssh -T user@host

# Force pseudo-terminal
ssh -t user@host

# Compression (useful for slow connections)
ssh -C user@host

# X11 forwarding (GUI applications)
ssh -X user@host
ssh -Y user@host  # Trusted X11 forwarding
```

---

## **2. SSH Key Management** 🔐

### **Generate SSH Keys**

```bash
# Generate RSA key (default)
ssh-keygen

# Generate with specific name and comment
ssh-keygen -t rsa -b 4096 -C "your_email@example.com" -f ~/.ssh/id_rsa_work

# Generate different key types
ssh-keygen -t rsa -b 4096      # RSA 4096-bit (recommended)
ssh-keygen -t ed25519          # Ed25519 (modern, secure, fast)
ssh-keygen -t ecdsa -b 521     # ECDSA
ssh-keygen -t dsa              # DSA (deprecated)

# Generate without passphrase (automation)
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa_automation

# Change key passphrase
ssh-keygen -p -f ~/.ssh/id_rsa
```

### **SSH Key Files**

```bash
# Default key locations
~/.ssh/id_rsa          # Private key (NEVER share)
~/.ssh/id_rsa.pub      # Public key (safe to share)
~/.ssh/known_hosts     # Verified host fingerprints
~/.ssh/authorized_keys # Allowed public keys (on server)
~/.ssh/config          # Client configuration

# Secure permissions
chmod 700 ~/.ssh                    # SSH directory
chmod 600 ~/.ssh/id_rsa            # Private key
chmod 644 ~/.ssh/id_rsa.pub        # Public key
chmod 644 ~/.ssh/authorized_keys   # Authorized keys
chmod 644 ~/.ssh/known_hosts       # Known hosts
chmod 600 ~/.ssh/config            # Config file
```

### **Copy SSH Key to Server**

```bash
# Automatic (recommended)
ssh-copy-id user@hostname
ssh-copy-id -i ~/.ssh/id_rsa.pub user@hostname

# Manual method
cat ~/.ssh/id_rsa.pub | ssh user@hostname 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'

# Or copy-paste manually
cat ~/.ssh/id_rsa.pub
# Copy output, then on server:
mkdir -p ~/.ssh
echo "ssh-rsa AAAAB3...your-key..." >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### **Manage Multiple SSH Keys**

```bash
# Generate keys for different services
ssh-keygen -t rsa -f ~/.ssh/id_rsa_github -C "github"
ssh-keygen -t rsa -f ~/.ssh/id_rsa_work -C "work"
ssh-keygen -t rsa -f ~/.ssh/id_rsa_personal -C "personal"

# Use specific key
ssh -i ~/.ssh/id_rsa_work user@work-server
ssh -i ~/.ssh/id_rsa_github git@github.com

# Or configure in ~/.ssh/config (see next section)
```

---

## **3. SSH Configuration** ⚙️

### **SSH Client Config (~/.ssh/config)**

```bash
# Create/edit SSH config
nano ~/.ssh/config

# Basic host configuration
Host myserver
    HostName 192.168.1.100
    User admin
    Port 22
    IdentityFile ~/.ssh/id_rsa

# Now connect with just:
ssh myserver

# Multiple server configuration
Host work-server
    HostName work.example.com
    User deploy
    Port 2222
    IdentityFile ~/.ssh/id_rsa_work
    
Host personal-server
    HostName personal.example.com
    User john
    IdentityFile ~/.ssh/id_rsa_personal
    
# GitHub configuration
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_github

# Wildcard pattern
Host *.dev.company.com
    User developer
    IdentityFile ~/.ssh/id_rsa_dev
    ForwardAgent yes

# Jump host (bastion/proxy)
Host internal-server
    HostName 10.0.1.50
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jumper
    IdentityFile ~/.ssh/id_rsa_bastion
```

### **Advanced SSH Config Options**

```bash
# Full-featured configuration
Host production-*
    User deploy
    Port 22
    IdentityFile ~/.ssh/id_rsa_prod
    
    # Keep connection alive
    ServerAliveInterval 60
    ServerAliveCountMax 3
    
    # Connection multiplexing (reuse connections)
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
    
    # Compression
    Compression yes
    
    # Faster connection
    TCPKeepAlive yes
    
    # Forward SSH agent
    ForwardAgent yes
    
    # Disable password authentication
    PasswordAuthentication no
    PubkeyAuthentication yes
    
    # X11 forwarding
    ForwardX11 no
    
    # Suppress banner
    LogLevel ERROR

# Default settings for all hosts
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600
    
# Create socket directory
mkdir -p ~/.ssh/sockets
```

### **Server SSH Config (/etc/ssh/sshd_config)**

```bash
# Edit server SSH configuration (requires root)
sudo nano /etc/ssh/sshd_config

# Security hardening
Port 2222                              # Change from default 22
PermitRootLogin no                     # Disable root login
PasswordAuthentication no              # Only key-based auth
PubkeyAuthentication yes               # Enable public key auth
PermitEmptyPasswords no                # No empty passwords
MaxAuthTries 3                         # Limit auth attempts
MaxSessions 2                          # Limit concurrent sessions
LoginGraceTime 60                      # Timeout for authentication

# Restrict users
AllowUsers deploy admin                # Only these users
DenyUsers guest temp                   # Deny these users
AllowGroups sshusers                   # Only this group

# Protocol and encryption
Protocol 2                             # Use SSH protocol 2 only
Ciphers aes256-ctr,aes192-ctr,aes128-ctr
MACs hmac-sha2-512,hmac-sha2-256
KexAlgorithms curve25519-sha256@libssh.org

# Other settings
X11Forwarding no                       # Disable unless needed
PrintMotd yes                          # Message of the day
PrintLastLog yes                       # Show last login
TCPKeepAlive yes                       # Keep connection alive
ClientAliveInterval 300               # Send alive message every 5 min
ClientAliveCountMax 2                 # Disconnect after 2 failed keepalives

# Restart SSH service after changes
sudo systemctl restart sshd           # SystemD
sudo service ssh restart              # SysVinit
```

---

## **4. Secure File Transfer** 📁

### **scp - Secure Copy**

```bash
# Copy file to remote server
scp file.txt user@host:/remote/path/
scp -P 2222 file.txt user@host:/path/  # Custom port

# Copy from remote server
scp user@host:/remote/file.txt ./local/

# Copy directory recursively
scp -r directory/ user@host:/remote/path/

# Copy multiple files
scp file1.txt file2.txt user@host:/path/
scp *.txt user@host:/path/

# Copy with compression
scp -C large-file.tar user@host:/path/

# Copy preserving attributes
scp -p file.txt user@host:/path/

# Copy with bandwidth limit (KB/s)
scp -l 1000 file.txt user@host:/path/

# Verbose output
scp -v file.txt user@host:/path/

# Copy between two remote hosts
scp user1@host1:/file.txt user2@host2:/path/

# Use specific SSH key
scp -i ~/.ssh/mykey file.txt user@host:/path/
```

### **rsync - Advanced Sync**

```bash
# Basic sync
rsync -avz source/ user@host:/destination/

# Common options
rsync -avz source/ dest/
# a = archive (preserve permissions, timestamps, etc.)
# v = verbose
# z = compress during transfer

# Dry run (test without changes)
rsync -avz --dry-run source/ dest/

# Show progress
rsync -avz --progress source/ dest/

# Delete files in destination not in source
rsync -avz --delete source/ dest/

# Exclude patterns
rsync -avz --exclude='*.log' source/ dest/
rsync -avz --exclude-from='exclude.txt' source/ dest/

# Include/exclude specific files
rsync -avz --include='*.conf' --exclude='*' source/ dest/

# Partial transfer (resume interrupted)
rsync -avz --partial source/ dest/

# Bandwidth limit
rsync -avz --bwlimit=1000 source/ dest/

# Backup old files
rsync -avz --backup --backup-dir=/backup source/ dest/

# SSH with custom port
rsync -avz -e "ssh -p 2222" source/ user@host:/dest/

# Sync with specific SSH key
rsync -avz -e "ssh -i ~/.ssh/mykey" source/ user@host:/dest/
```

### **sftp - Interactive File Transfer**

```bash
# Connect to SFTP server
sftp user@hostname
sftp -P 2222 user@hostname  # Custom port

# SFTP commands (interactive mode)
ls                # List remote files
lls               # List local files
pwd               # Remote working directory
lpwd              # Local working directory
cd /path          # Change remote directory
lcd /path         # Change local directory

get file.txt      # Download file
get -r directory/ # Download directory
put file.txt      # Upload file
put -r directory/ # Upload directory

mkdir newdir      # Create remote directory
lmkdir newdir     # Create local directory
rm file.txt       # Delete remote file
rmdir directory   # Delete remote directory

rename old new    # Rename remote file
chmod 644 file    # Change remote permissions

exit              # Quit SFTP
bye               # Quit SFTP

# Batch mode (non-interactive)
sftp -b commands.txt user@host

# Example commands.txt:
cd /remote/path
get file1.txt
put file2.txt
exit
```

---

## **5. SSH Tunneling & Port Forwarding** 🌐

### **Local Port Forwarding**

```bash
# Forward local port to remote server
ssh -L local_port:destination:destination_port user@ssh_server

# Example: Access remote MySQL through SSH tunnel
ssh -L 3306:localhost:3306 user@dbserver
# Now connect to localhost:3306 on your machine

# Access internal service through jump host
ssh -L 8080:internal-server:80 user@bastion
# Access http://localhost:8080 → bastion → internal-server:80

# Multiple port forwards
ssh -L 3306:db:3306 -L 6379:redis:6379 user@server

# Keep tunnel open without shell
ssh -L 8080:remote:80 -N user@server
# -N = No command execution, just forwarding
```

### **Remote Port Forwarding**

```bash
# Forward remote port to local machine
ssh -R remote_port:localhost:local_port user@remote

# Example: Share local web server with remote
ssh -R 8080:localhost:3000 user@remote-server
# remote-server:8080 → your-machine:3000

# Make service accessible to remote network
ssh -R 0.0.0.0:8080:localhost:3000 user@remote
# Requires GatewayPorts yes in sshd_config

# Keep tunnel running
ssh -R 8080:localhost:3000 -N user@remote
```

### **Dynamic Port Forwarding (SOCKS Proxy)**

```bash
# Create SOCKS proxy
ssh -D 1080 user@server

# Configure browser to use localhost:1080 as SOCKS5 proxy
# All browser traffic routes through SSH server

# Use with curl
curl --proxy socks5://localhost:1080 http://example.com

# Background SOCKS proxy
ssh -D 1080 -fN user@server
# -f = background
# -N = no command
```

### **Jump Host / Proxy Jump**

```bash
# Connect through jump host (old method)
ssh -J jump-host final-destination
ssh -J user1@bastion user2@internal-server

# Multiple jump hosts
ssh -J jump1,jump2,jump3 final-destination

# ProxyCommand (alternative)
ssh -o ProxyCommand="ssh -W %h:%p jump-host" final-destination

# In ~/.ssh/config:
Host internal
    HostName 10.0.1.50
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User jumper
```

---

## **6. Remote Command Execution** 💻

### **Execute Single Commands**

```bash
# Run command on remote server
ssh user@host 'command'

# Examples
ssh user@host 'uptime'
ssh user@host 'df -h'
ssh user@host 'ps aux | grep nginx'

# Multiple commands
ssh user@host 'cd /var/log && tail -n 20 syslog'
ssh user@host "command1; command2; command3"

# Store output locally
ssh user@host 'cat /var/log/app.log' > local-app.log

# Pipe local data to remote command
cat local-file.txt | ssh user@host 'cat > remote-file.txt'

# Interactive sudo (requires -t)
ssh -t user@host 'sudo systemctl restart nginx'
```

### **Execute Scripts Remotely**

```bash
# Run local script on remote server
ssh user@host 'bash -s' < local-script.sh

# With arguments
ssh user@host 'bash -s' < script.sh arg1 arg2

# Run script and keep it on server
scp script.sh user@host:/tmp/
ssh user@host 'bash /tmp/script.sh'

# Heredoc remote execution
ssh user@host <<'EOF'
echo "Running commands on $(hostname)"
uptime
df -h
EOF
```

### **Parallel Execution on Multiple Servers**

```bash
# Simple loop
for server in server1 server2 server3; do
    ssh user@$server 'uptime' &
done
wait

# Using xargs
echo -e "server1\nserver2\nserver3" | xargs -I {} -P 3 ssh user@{} 'command'

# Using GNU parallel (install first)
parallel -j 10 ssh user@{} uptime ::: server{1..10}

# pssh (parallel-ssh) - install first
pssh -h hosts.txt -i 'uptime'
# hosts.txt contains server list
```

---

## **7. SSH Agent & Key Agent Forwarding** 🔐

### **SSH Agent**

```bash
# Start SSH agent
eval "$(ssh-agent -s)"
# Agent pid 12345

# Add key to agent
ssh-add ~/.ssh/id_rsa
ssh-add ~/.ssh/id_rsa_work
ssh-add ~/.ssh/id_rsa_github

# List keys in agent
ssh-add -l

# Remove key from agent
ssh-add -d ~/.ssh/id_rsa

# Remove all keys
ssh-add -D

# Lock agent (require passphrase)
ssh-add -x

# Unlock agent
ssh-add -X

# Auto-add keys on macOS ( add to ~/.ssh/config)
Host *
    AddKeysToAgent yes
    UseKeychain yes
```

### **Agent Forwarding**

```bash
# Enable agent forwarding
ssh -A user@host

# In ~/.ssh/config:
Host trusted-server
    ForwardAgent yes

# Use case: Access GitHub from remote server
ssh -A user@server
# On server:
git clone git@github.com:user/repo.git
# Uses your local SSH key without copying it

# Verify agent forwarding
ssh user@host 'echo $SSH_AUTH_SOCK'
```

### **SSH Agent Systemd Service**

```bash
# Auto-start SSH agent (systemd)
mkdir -p ~/.config/systemd/user

cat > ~/.config/systemd/user/ssh-agent.service <<'EOF'
[Unit]
Description=SSH key agent

[Service]
Type=simple
Environment=SSH_AUTH_SOCK=%t/ssh-agent.socket
ExecStart=/usr/bin/ssh-agent -D -a $SSH_AUTH_SOCK

[Install]
WantedBy=default.target
EOF

systemctl --user enable ssh-agent
systemctl --user start ssh-agent

# Add to ~/.bashrc or ~/.zshrc:
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/ssh-agent.socket"
```

---

## **8. Advanced SSH Techniques** 🚀

### **SSH Connection Multiplexing**

```bash
# Share connection between multiple SSH sessions
# Add to ~/.ssh/config:
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h-%p
    ControlPersist 600

mkdir -p ~/.ssh/sockets

# First connection creates master
ssh user@server

# Subsequent connections reuse master (much faster)
ssh user@server
scp file.txt user@server:/path/
```

### **SSH Escape Sequences**

```bash
# Press Enter, then ~? for help

~.    # Terminate connection
~^Z   # Suspend SSH session
~#    # List forwarded connections
~~    # Send ~ character
~?    # Show help

# Example: Terminate frozen session
[Enter]
~.
# Connection closed
```

### **SSH Jump with ProxyJump**

```bash
# Modern jump host configuration
ssh -J bastion internal-server
ssh -J bastion1,bastion2 final-server

# In ~/.ssh/config:
Host internal-*
    ProxyJump bastion

Host bastion
    HostName bastion.company.com
    User jumper
```

### **SSH Certificate Authentication**

```bash
# Generate CA key (on CA server)
ssh-keygen -t rsa -f ca_key -C "SSH CA"

# Sign user key
ssh-keygen -s ca_key -I user_cert -n user1,user2 -V +52w ~/.ssh/id_rsa.pub
# Creates id_rsa-cert.pub (valid for 52 weeks)

# Configure server to trust CA (/etc/ssh/sshd_config)
TrustedUserCAKeys /etc/ssh/ca_key.pub

# User connects with certificate
ssh -i ~/.ssh/id_rsa user@host
```

---

## **9. Remote Desktop & VNC** 🖥️

### **X11 Forwarding (GUI Apps)**

```bash
# Enable X11 forwarding
ssh -X user@host
ssh -Y user@host  # Trusted forwarding

# Run GUI application
ssh -X user@host xclock
ssh -X user@host firefox

# In ~/.ssh/config:
Host gui-server
    ForwardX11 yes
    ForwardX11Trusted yes
```

### **VNC over SSH Tunnel**

```bash
# Remote server has VNC on port 5901
# Create tunnel
ssh -L 5901:localhost:5901 user@server

# Connect VNC viewer to localhost:5901
vncviewer localhost:5901
```

### **Remote Desktop (RDP) through SSH**

```bash
# Forward RDP port
ssh -L 3389:windows-server:3389 user@linux-gateway

# Connect RDP client to localhost:3389
```

---

## **10. DevOps Use Cases** 🚀

### **Automated Deployment**

```bash
#!/bin/bash
# deploy.sh - Automated deployment script

SERVERS=("web1" "web2" "web3")
APP_DIR="/opt/app"

for server in "${SERVERS[@]}"; do
    echo "Deploying to $server..."
    
    # Backup current version
    ssh deploy@$server "tar -czf /backup/app-$(date +%Y%m%d-%H%M%S).tar.gz $APP_DIR"
    
    # Copy new version
    rsync -avz --delete ./app/ deploy@$server:$APP_DIR/
    
    # Restart service
    ssh -t deploy@$server "sudo systemctl restart app"
    
    # Health check
    ssh deploy@$server "curl -f http://localhost:8080/health || exit 1"
    
    echo "$server deployed successfully"
done
```

### **Database Tunnel for Local Development**

```bash
#!/bin/bash
# db-tunnel.sh - Create SSH tunnel to production database

# Production database access through bastion
ssh -L 5432:db-internal:5432 -N -f bastion.company.com

echo "PostgreSQL tunnel established"
echo "Connect to: localhost:5432"

# Usage:
# psql -h localhost -p 5432 -U dbuser -d production_db
```

### **Monitoring Script**

```bash
#!/bin/bash
# monitor-servers.sh

servers=(
    "web1.example.com"
    "web2.example.com"
    "db.example.com"
)

for server in "${servers[@]}"; do
    echo "=== $server ==="
    ssh monitoring@$server <<'SCRIPT'
        echo "Uptime: $(uptime -p)"
        echo "Load: $(cat /proc/loadavg)"
        echo "Memory: $(free -h | grep Mem | awk '{print $3"/"$2}')"
        echo "Disk: $(df -h / | tail -1 | awk '{print $5}')"
        echo ""
SCRIPT
done
```

### **Log Aggregation**

```bash
#!/bin/bash
# collect-logs.sh

SERVERS=("app1" "app2" "app3")
LOG_DIR="/var/log/app"
DEST="./logs/$(date +%Y%m%d)"

mkdir -p "$DEST"

for server in "${SERVERS[@]}"; do
    echo "Collecting logs from $server..."
    rsync -avz --progress ${server}:${LOG_DIR}/app.log "$DEST/${server}-app.log"
done

echo "Logs collected in $DEST"
```

---

## **11. Troubleshooting** 🔧

### **Connection Issues**

```bash
# Test connection
ssh -v user@host              # Verbose
ssh -vvv user@host            # Maximum verbosity

# Common issues and fixes:

# 1. Connection timeout
# Check firewall
sudo ufw status
sudo iptables -L -n

# Test port connectivity
telnet host 22
nc -zv host 22

# 2. Permission denied (publickey)
# Check key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub

# Verify key on server
cat ~/.ssh/authorized_keys

# Test with password
ssh -o PubkeyAuthentication=no user@host

# 3. Host key verification failed
# Remove old key
ssh-keygen -R hostname

# Update known_hosts
ssh-keyscan hostname >> ~/.ssh/known_hosts

# 4. Too many authentication failures
# Limit identity files
ssh -o IdentitiesOnly=yes -i ~/.ssh/specific_key user@host
```

### **Debug SSH Server**

```bash
# Check SSH server status
sudo systemctl status sshd
sudo systemctl status ssh

# View SSH server logs
sudo tail -f /var/log/auth.log        # Debian/Ubuntu
sudo tail -f /var/log/secure          # RHEL/CentOS
sudo journalctl -u sshd -f            # Systemd

# Test SSH configuration
sudo sshd -t

# Restart SSH service
sudo systemctl restart sshd
sudo service ssh restart
```

### **Performance Issues**

```bash
# Disable DNS lookup (server-side)
# Add to /etc/ssh/sshd_config:
UseDNS no

# Enable compression
ssh -C user@host

# Disable encryption (NOT for production!)
ssh -c none user@host

# Use faster cipher
ssh -c aes128-ctr user@host

# Connection multiplexing (see Advanced section)
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: How does SSH authentication work?**
```
Answer:
SSH uses asymmetric encryption (public/private key pairs).

Process:
1. Client requests connection to server
2. Server sends its public key
3. Client verifies server key against known_hosts
4. Client encrypts session key with server's public key
5. Server decrypts with private key
6. Secure encrypted channel established
7. User authentication (password or key-based)

Key-based auth:
- Server has user's public key (authorized_keys)
- Client proves ownership of private key
- More secure than passwords

Files:
~/.ssh/id_rsa (private) - NEVER share
~/.ssh/id_rsa.pub (public) - Safe to share
~/.ssh/authorized_keys (server) - Allowed public keys
```

### **Q2: Difference between ssh-copy-id and manual copy?**
```
ssh-copy-id (recommended):
ssh-copy-id user@host
- Automatic
- Sets correct permissions
- Appends to authorized_keys
- Removes duplicates

Manual:
cat ~/.ssh/id_rsa.pub | ssh user@host 'cat >> ~/.ssh/authorized_keys'
- More control
- Works without ssh-copy-id
- Manual permission fixing needed

Both achieve same result.
```

### **Q3: Explain SSH port forwarding types?**
```
Local Port Forwarding (-L):
ssh -L local_port:destination:dest_port user@ssh_server
Use: Access remote service through SSH tunnel
Example: ssh -L 3306:db:3306 user@bastion
→ Access db:3306 via localhost:3306

Remote Port Forwarding (-R):
ssh -R remote_port:localhost:local_port user@remote
Use: Share local service with remote server
Example: ssh -R 8080:localhost:3000 user@server
→ Share local:3000 as server:8080

Dynamic Port Forwarding (-D):
ssh -D 1080 user@server
Use: SOCKS proxy for all traffic
Example: Browse web through SSH tunnel
```

### **Q4: How to secure SSH server?**
```
/etc/ssh/sshd_config hardening:

Port 2222                          # Change default port
PermitRootLogin no                 # Disable root login
PasswordAuthentication no          # Only key-based
MaxAuthTries 3                     # Limit attempts
AllowUsers deploy admin            # Whitelist users
Protocol 2                         # Use SSH2 only

Additional security:
- Fail2ban (block brute force)
- Strong key algorithms
- Firewall (allow only specific IPs)
- Regular updates
- Monitor auth logs
- Use ssh-audit tool

Commands:
sudo sshd -t                       # Test config
sudo systemctl restart sshd        # Apply changes
sudo fail2ban-client status sshd   # Check fail2ban
```

### **Q5: Common SSH commands for DevOps?**
```
Connection:
ssh user@host                      # Connect
ssh -i key user@host              # Specific key
ssh -J bastion internal           # Jump host

File Transfer:
scp file user@host:/path          # Copy file
rsync -avz src/ user@host:dst/    # Sync directories

Port Forwarding:
ssh -L 3306:db:3306 user@bastion  # Local forward
ssh -D 1080 user@host             # SOCKS proxy

Remote Commands:
ssh user@host 'command'           # Execute command
ssh -t user@host 'sudo cmd'       # Interactive sudo

Key Management:
ssh-keygen -t rsa -b 4096        # Generate key
ssh-copy-id user@host            # Copy key to server
ssh-add ~/.ssh/id_rsa            # Add to agent

Debugging:
ssh -v user@host                  # Verbose
ssh-keygen -R hostname            # Remove host key
tail -f /var/log/auth.log         # View logs
```

---

## **Next Steps** 📚

- **[Network Commands](Network_Commands.md)** - Networking and troubleshooting
- **[Security & Firewall](Security_Firewall_Commands.md)** - iptables, firewalld, security  
- **[System Monitoring](System_Monitoring_Commands.md)** - Performance monitoring
- **[Process Management](Process_Management.md)** - Process control

---

**🔐 Master SSH for Secure Remote Server Management!**

*SSH is the foundation of secure remote administration - essential for every DevOps engineer.*
