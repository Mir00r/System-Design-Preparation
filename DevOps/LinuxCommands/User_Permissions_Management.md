# **Linux User & Permissions Management – Complete Guide for DevOps** 👥🔐

Master essential commands for managing users, groups, and permissions - critical skills for DevOps engineers securing systems, managing access control, and implementing the principle of least privilege.

---

## **Table of Contents** 📑
1. [User Management Fundamentals](#1-user-management-fundamentals)
2. [User Commands](#2-user-commands)
3. [Group Management](#3-group-management)
4. [Permission Management](#4-permission-management)
5. [Special Permissions](#5-special-permissions)
6. [Access Control Lists (ACLs)](#6-access-control-lists-acls)
7. [Sudo and Root Access](#7-sudo-and-root-access)
8. [SSH Key Management](#8-ssh-key-management)
9. [Practical DevOps Scenarios](#9-practical-devops-scenarios)
10. [Industry Best Practices](#10-industry-best-practices)
11. [Interview Cheat Sheet](#11-interview-cheat-sheet)

---

## **1. User Management Fundamentals** 🎯

### **User Types**

```
Regular User    UID 1000+      Normal user accounts
System User     UID 1-999      Service accounts (nginx, mysql)
Root User       UID 0          Superuser (unrestricted access)
```

### **Key Files**

**/etc/passwd - User Database:**
```
username:x:UID:GID:comment:home:shell
root:x:0:0:root:/root:/bin/bash
john:x:1000:1000:John Doe:/home/john:/bin/bash
```

**Fields:**
1. Username
2. Password (x = in /etc/shadow)
3. UID (User ID)
4. GID (Primary Group ID)
5. Comment/GECOS
6. Home directory
7. Login shell

**/etc/shadow - Password Database:**
```
username:$encrypted$password:lastchange:min:max:warn:inactive:expire:reserved
```

**Fields:**
1. Username
2. Encrypted password
3. Days since Jan 1, 1970 password was last changed
4. Min days between password changes
5. Max days password is valid
6. Days warning before password expires
7. Days after expiration account is disabled
8. Days since Jan 1, 1970 account is disabled
9. Reserved

**/etc/group - Group Database:**
```
groupname:x:GID:members
sudo:x:27:john,jane
developers:x:1001:alice,bob
```

---

## **2. User Commands** 👤

### **useradd - Add User**

```bash
sudo useradd username                        # Basic user creation
sudo useradd -m username                     # Create with home directory
sudo useradd -m -s /bin/bash username        # Specify shell
sudo useradd -m -G sudo,developers username  # Add to groups
sudo useradd -m -d /custom/home username     # Custom home directory
sudo useradd -m -e 2024-12-31 username       # Set expiry date
sudo useradd -m -c "John Doe" username       # Add comment
sudo useradd -m -u 1500 username             # Specify UID
sudo useradd -r app-user                     # Create system user
```

**Full Example:**
```bash
sudo useradd -m \
  -s /bin/bash \
  -G sudo,developers \
  -c "John Doe" \
  -e 2025-12-31 \
  john
```

### **adduser - Interactive User Creation (Debian/Ubuntu)**

```bash
sudo adduser username                        # Interactive creation
sudo adduser --system app-user               # System user
sudo adduser --disabled-login username       # No login
sudo adduser --home /custom/home username    # Custom home
```

### **usermod - Modify User**

```bash
sudo usermod -aG groupname username          # Add to group (append)
sudo usermod -G group1,group2 username       # Set groups (replace)
sudo usermod -d /new/home username           # Change home directory
sudo usermod -m -d /new/home username        # Move home directory
sudo usermod -s /bin/zsh username            # Change shell
sudo usermod -l newname oldname              # Rename user
sudo usermod -L username                     # Lock account
sudo usermod -U username                     # Unlock account
sudo usermod -e 2024-12-31 username          # Set expiry
sudo usermod -e "" username                  # Remove expiry
```

**Examples:**
```bash
# Add user to docker group
sudo usermod -aG docker username

# Change user's shell
sudo usermod -s /bin/zsh john

# Lock user account
sudo usermod -L baduser

# Move home and change shell
sudo usermod -m -d /opt/appuser -s /bin/bash appuser
```

### **userdel - Delete User**

```bash
sudo userdel username                        # Delete user (keep home)
sudo userdel -r username                     # Delete user and home directory
sudo userdel -f username                     # Force delete
```

**Safe Deletion:**
```bash
# 1. Check running processes
ps -u username

# 2. Kill user processes
sudo pkill -u username

# 3. Backup home directory
sudo tar -czf username-backup.tar.gz /home/username

# 4. Delete user
sudo userdel -r username
```

### **passwd - Password Management**

```bash
passwd                                       # Change own password
sudo passwd username                         # Change user password
sudo passwd -l username                      # Lock password
sudo passwd -u username                      # Unlock password
sudo passwd -d username                      # Delete password (no password)
sudo passwd -e username                      # Expire password (force change)
sudo passwd -S username                      # Password status
sudo passwd -n 7 username                    # Min 7 days between changes
sudo passwd -x 90 username                   # Max 90 days validity
sudo passwd -w 14 username                   # 14 days warning
```

**Password Status:**
```bash
sudo passwd -S john
# john PS 2024-02-18 7 90 14 30 (Password set, last changed, min, max, warn, inactive)
```

### **chage - Change Password Aging**

```bash
chage -l username                            # List aging info
sudo chage -m 7 username                     # Min days between changes
sudo chage -M 90 username                    # Max days valid
sudo chage -W 14 username                    # Warning days
sudo chage -I 30 username                    # Inactive days
sudo chage -E 2024-12-31 username            # Expiry date
sudo chage -d 0 username                     # Force password change on next login
```

**Interactive Mode:**
```bash
sudo chage username
```

### **User Information Commands**

```bash
whoami                                       # Current username
id                                           # Current user ID and groups
id username                                  # User's UID, GID, groups
who                                          # Who is logged in
w                                            # Who and what they're doing
last                                         # Login history
lastlog                                      # Last login per user
finger username                              # User information
getent passwd username                       # Query user database
```

**Examples:**
```bash
# Current user info
id
# Output: uid=1000(john) gid=1000(john) groups=1000(john),27(sudo),999(docker)

# Check if user exists
id username &>/dev/null && echo "Exists" || echo "Does not exist"

# List all users
getent passwd | cut -d: -f1

# List users with UID > 1000 (regular users)
awk -F: '$3 >= 1000 {print $1}' /etc/passwd
```

---

## **3. Group Management** 👥

### **groupadd - Create Group**

```bash
sudo groupadd groupname                      # Create group
sudo groupadd -g 5000 groupname              # Specify GID
sudo groupadd -r app-group                   # System group
```

### **groupmod - Modify Group**

```bash
sudo groupmod -n newname oldname             # Rename group
sudo groupmod -g 5000 groupname              # Change GID
```

### **groupdel - Delete Group**

```bash
sudo groupdel groupname                      # Delete group
```

### **groups - Show User's Groups**

```bash
groups                                       # Current user's groups
groups username                              # User's groups
```

### **gpasswd - Group Administration**

```bash
sudo gpasswd -a username groupname           # Add user to group
sudo gpasswd -d username groupname           # Remove user from group
sudo gpasswd -M user1,user2,user3 groupname  # Set group members
sudo gpasswd -A adminuser groupname          # Set group admin
```

**Examples:**
```bash
# Add multiple users to docker group
for user in alice bob charlie; do
    sudo gpasswd -a $user docker
done

# Remove user from sudo group
sudo gpasswd -d john sudo

# Set group members
sudo gpasswd -M alice,bob,charlie developers
```

### **newgrp - Change Current Group**

```bash
newgrp groupname                             # Switch to group (new shell)
exit                                         # Return to original group
```

---

## **4. Permission Management** 🔐

### **Understanding Permissions**

**Permission String:**
```
-rwxr-xr--
│├─┼─┼─┼─┤
││ │ │ │ └─ Others: read
││ │ │ └─── Group: read, execute
││ │ └───── User: read, write, execute
│└─────────  File type (- = file, d = directory, l = link)
```

**File Types:**
```
-    Regular file
d    Directory
l    Symbolic link
b    Block device
c    Character device
s    Socket
p    Named pipe (FIFO)
```

**Permission Values:**
```
r (read)    = 4
w (write)   = 2
x (execute) = 1

rwx = 7  (4+2+1)
rw- = 6  (4+2)
r-x = 5  (4+1)
r-- = 4  (4)
-wx = 3  (2+1)
-w- = 2  (2)
--x = 1  (1)
--- = 0  (0)
```

**Common Permission Modes:**
```
777  rwxrwxrwx  All permissions for everyone (dangerous!)
755  rwxr-xr-x  Standard for directories/executables
644  rw-r--r--  Standard for files
600  rw-------  Private file
700  rwx------  Private directory
```

### **chmod - Change Permissions**

**Numeric Mode:**
```bash
chmod 755 file.sh                            # rwxr-xr-x
chmod 644 file.txt                           # rw-r--r--
chmod 600 secret.key                         # rw-------
chmod 700 ~                                  # rwx------
chmod -R 755 directory/                      # Recursive
```

**Symbolic Mode:**
```bash
chmod u+x file.sh                            # Add execute for user
chmod g-w file.txt                           # Remove write for group
chmod o+r file.txt                           # Add read for others
chmod a+x file.sh                            # Add execute for all
chmod u=rwx,g=rx,o=r file.sh                 # Set specific permissions
chmod u+x,g+x file.sh                        # Add multiple
```

**Who:**
```
u    User/Owner
g    Group
o    Others
a    All (ugo)
```

**Operations:**
```
+    Add permission
-    Remove permission
=    Set exact permission
```

**Examples:**
```bash
# Make script executable
chmod +x script.sh
chmod u+x script.sh

# Remove write for group and others
chmod go-w file.txt

# Set directory permissions
chmod 755 /var/www/html

# Recursive - directories 755, files 644
find /var/www -type d -exec chmod 755 {} \;
find /var/www -type f -exec chmod 644 {} \;

# Copy permissions from another file
chmod --reference=file1.txt file2.txt
```

### **chown - Change Owner**

```bash
sudo chown user file.txt                     # Change owner
sudo chown user:group file.txt               # Change owner and group
sudo chown :group file.txt                   # Change group only
sudo chown -R user:group directory/          # Recursive
sudo chown --reference=file1 file2           # Match file1's ownership
```

**Examples:**
```bash
# Change web directory ownership
sudo chown -R www-data:www-data /var/www/html

# Change ownership of all files for user
sudo find /home/john -exec chown john:john {} \;

# Change ownership but preserve group
sudo chown john file.txt
```

### **chgrp - Change Group**

```bash
sudo chgrp group file.txt                    # Change group
sudo chgrp -R group directory/               # Recursive
sudo chgrp --reference=file1 file2           # Match file1's group
```

### **umask - Default Permissions**

```bash
umask                                        # Show current umask
umask 022                                    # Set umask
umask -S                                     # Symbolic display
```

**How umask Works:**
```
Default file:      666 (rw-rw-rw-)
umask:            -022 (----w--w-)
Result:            644 (rw-r--r--)

Default directory: 777 (rwxrwxrwx)
umask:            -022 (----w--w-)
Result:            755 (rwxr-xr-x)
```

**Common umask Values:**
```
022    User: rwx, Group: r-x, Others: r-x
027    User: rwx, Group: r-x, Others: ---
002    User: rwx, Group: rwx, Others: r-x
077    User: rwx, Group: ---, Others: ---  (strict)
```

**Set Permanently:**
```bash
# In ~/.bashrc or /etc/profile
umask 022
```

---

## **5. Special Permissions** ⚡

### **SUID (Set User ID)**

**Effect:** Execute file as owner, not current user

```bash
chmod u+s file                               # Set SUID
chmod 4755 file                              # Numeric (4xxx)
chmod u-s file                               # Remove SUID
```

**Display:** `-rwsr-xr-x` (s in user execute position)

**Common Example:**
```bash
ls -l /usr/bin/passwd
# -rwsr-xr-x 1 root root 68208 /usr/bin/passwd
# Allows regular users to change passwords (writes to /etc/shadow owned by root)
```

**Security Warning:** SUID on shell scripts is dangerous!

### **SGID (Set Group ID)**

**On Files:** Execute as group owner
**On Directories:** New files inherit directory's group

```bash
chmod g+s directory                          # Set SGID
chmod 2755 directory                         # Numeric (2xxx)
chmod g-s directory                          # Remove SGID
```

**Display:** `-rwxr-sr-x` (s in group execute position)

**Example:**
```bash
# Shared directory for team
sudo mkdir /shared/developers
sudo chgrp developers /shared/developers
sudo chmod 2775 /shared/developers

# Now all files created in /shared/developers will have group 'developers'
```

### **Sticky Bit**

**Effect:** Only owner can delete files (even if directory is writable by all)

```bash
chmod +t directory                           # Set sticky bit
chmod 1777 directory                         # Numeric (1xxx)
chmod -t directory                           # Remove sticky bit
```

**Display:** `drwxrwxrwt` (t in others execute position)

**Example:**
```bash
ls -ld /tmp
# drwxrwxrwt 15 root root 4096 /tmp
# All users can create files in /tmp, but only owners can delete their own files
```

### **Special Permissions Summary**

| Permission | Numeric | Symbol | Files | Directories |
|------------|---------|--------|-------|-------------|
| **SUID** | 4 | s (user) | Execute as owner | N/A |
| **SGID** | 2 | s (group) | Execute as group | Inherit group |
| **Sticky** | 1 | t (others) | N/A | Owner-only delete |

**Combined:**
```bash
chmod 4755 file     # SUID + 755
chmod 2755 dir      # SGID + 755
chmod 1777 dir      # Sticky + 777
chmod 6755 file     # SUID + SGID + 755
```

---

## **6. Access Control Lists (ACLs)** 📋

**ACLs provide fine-grained permissions beyond user/group/others.**

### **getfacl - Get ACL**

```bash
getfacl file.txt                             # View ACL
getfacl -R directory/                        # Recursive
```

**Output:**
```
# file: file.txt
# owner: john
# group: developers
user::rw-
user:alice:rw-
group::r--
mask::rw-
other::r--
```

### **setfacl - Set ACL**

```bash
setfacl -m u:alice:rw file.txt               # Give alice read/write
setfacl -m g:developers:rx directory/        # Give group execute
setfacl -m o::r file.txt                     # Others read-only
setfacl -x u:alice file.txt                  # Remove alice's ACL
setfacl -b file.txt                          # Remove all ACLs
setfacl -R -m u:alice:rw directory/          # Recursive
setfacl -d -m u:alice:rw directory/          # Default ACL (for new files)
setfacl -k directory/                        # Remove default ACL
```

**Examples:**
```bash
# Give multiple users access
setfacl -m u:alice:rwx,u:bob:rx file.txt

# Copy ACL from one file to another
getfacl file1.txt | setfacl --set-file=- file2.txt

# Set default ACL for directory (inherited by new files)
setfacl -d -m u:alice:rwx /shared/project

# View ACL
getfacl /shared/project

# Backup and restore ACLs
getfacl -R /path > acl-backup.txt
setfacl --restore=acl-backup.txt
```

---

## **7. Sudo and Root Access** 👑

### **sudo - Execute as Root**

```bash
sudo command                                 # Execute as root
sudo -u username command                     # Execute as specific user
sudo -i                                      # Login shell as root
sudo -s                                      # Shell as root
sudo -l                                      # List sudo privileges
sudo -k                                      # Invalidate sudo timestamp
sudo -v                                      # Refresh sudo timestamp
sudo !!                                      # Execute last command with sudo
```

**Examples:**
```bash
# Run command as root
sudo systemctl restart nginx

# Run as different user
sudo -u www-data touch /var/www/file.txt

# Edit protected file
sudo nano /etc/hosts

# Run multiple commands
sudo sh -c "echo 'text' > /etc/file && chmod 644 /etc/file"
```

### **visudo - Edit Sudoers File**

```bash
sudo visudo                                  # Edit /etc/sudoers (syntax checking)
```

**Sudoers File Format:**
```
# User privilege specification
root ALL=(ALL:ALL) ALL
john ALL=(ALL:ALL) ALL

# Allow group
%sudo ALL=(ALL:ALL) ALL
%admin ALL=(ALL:ALL) ALL

# No password
alice ALL=(ALL) NOPASSWD: ALL

# Specific commands
bob ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx

# Command aliases
Cmnd_Alias SERVICES = /usr/bin/systemctl restart *, /usr/bin/systemctl status *
charlie ALL=(ALL) NOPASSWD: SERVICES

# User aliases
User_Alias ADMINS = alice, bob, charlie
ADMINS ALL=(ALL:ALL) ALL
```

**Examples:**
```bash
# Allow user to run specific commands without password
devops ALL=(ALL) NOPASSWD: /usr/bin/docker, /usr/bin/kubectl

# Allow group to restart services
%developers ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# Time-limited sudo
Defaults timestamp_timeout=5

# Log sudo commands
Defaults logfile="/var/log/sudo.log"
```

### **su - Switch User**

```bash
su                                           # Switch to root
su -                                         # Login shell as root
su username                                  # Switch to user
su - username                                # Login shell as user
exit                                         # Return to previous user
```

**Difference between su and su -:**
```
su       Switch user but keep environment
su -     Login shell with user's environment
```

---

## **8. SSH Key Management** 🔑

### **ssh-keygen - Generate Keys**

```bash
ssh-keygen                                   # Interactive generation
ssh-keygen -t rsa -b 4096                    # RSA 4096-bit
ssh-keygen -t ed25519                        # Ed25519 (recommended)
ssh-keygen -t rsa -b 4096 -C "your@email.com"  # With comment
ssh-keygen -f ~/.ssh/custom_key              # Custom filename
ssh-keygen -p -f ~/.ssh/id_rsa               # Change passphrase
```

**Example:**
```bash
# Generate Ed25519 key
ssh-keygen -t ed25519 -C "john@example.com" -f ~/.ssh/id_ed25519

# Generate RSA key without passphrase (automation)
ssh-keygen -t rsa -b 4096 -N "" -f ~/.ssh/deploy_key
```

### **ssh-copy-id - Copy Public Key**

```bash
ssh-copy-id user@host                        # Copy default key
ssh-copy-id -i ~/.ssh/id_rsa.pub user@host   # Specific key
ssh-copy-id -p 2222 user@host                # Custom port
```

**Manual Copy:**
```bash
cat ~/.ssh/id_rsa.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### **SSH Directory Permissions**

```bash
chmod 700 ~/.ssh                             # SSH directory
chmod 600 ~/.ssh/id_rsa                      # Private key
chmod 644 ~/.ssh/id_rsa.pub                  # Public key
chmod 644 ~/.ssh/authorized_keys             # Authorized keys
chmod 644 ~/.ssh/known_hosts                 # Known hosts
chmod 600 ~/.ssh/config                      # SSH config
```

**Verify Permissions:**
```bash
ls -la ~/.ssh
# drwx------  2 user user 4096 .ssh/
# -rw-------  1 user user 3243 id_rsa
# -rw-r--r--  1 user user  748 id_rsa.pub
# -rw-r--r--  1 user user 2304 authorized_keys
```

---

## **9. Practical DevOps Scenarios** 🛠️

### **Scenario 1: Create Service Account**

```bash
# Create system user (no login)
sudo useradd -r -s /bin/false -d /nonexistent app-service

# Create with home directory for application
sudo useradd -r -m -d /opt/myapp -s /bin/bash myapp

# Set ownership
sudo chown -R myapp:myapp /opt/myapp

# Create systemd service
sudo nano /etc/systemd/system/myapp.service
# [Service]
# User=myapp
# Group=myapp
# WorkingDirectory=/opt/myapp
# ExecStart=/opt/myapp/start.sh
```

### **Scenario 2: Shared Development Directory**

```bash
# Create shared directory
sudo mkdir -p /shared/development

# Create group
sudo groupadd developers

# Add users to group
sudo usermod -aG developers alice
sudo usermod -aG developers bob
sudo usermod -aG developers charlie

# Set ownership and permissions
sudo chown root:developers /shared/development
sudo chmod 2775 /shared/development  # SGID + 775

# Set default ACL
sudo setfacl -d -m g:developers:rwx /shared/development
sudo setfacl -d -m o::rx /shared/development

# Verify
getfacl /shared/development
```

### **Scenario 3: Lock Compromised Account**

```bash
# 1. Lock the account
sudo passwd -l compromised-user

# 2. Expire the password
sudo chage -E 0 compromised-user

# 3. Kill all sessions
sudo pkill -u compromised-user

# 4. Prevent new logins
sudo usermod -s /sbin/nologin compromised-user

# 5. Review logs
sudo last compromised-user
sudo lastlog -u compromised-user

# 6. Check for SSH keys
sudo ls -la /home/compromised-user/.ssh/

# 7. Backup and remove if needed
sudo tar -czf compromised-user-backup.tar.gz /home/compromised-user
sudo userdel -r compromised-user
```

### **Scenario 4: Setup Sudo for Team**

```bash
# Create sudoers file for team
sudo visudo -f /etc/sudoers.d/devops-team

# Add content:
# DevOps team sudo access
%devops ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx
%devops ALL=(ALL) NOPASSWD: /usr/bin/systemctl status nginx
%devops ALL=(ALL) NOPASSWD: /usr/bin/docker
%devops ALL=(ALL) /usr/bin/vim /etc/nginx/*

# Set permissions
sudo chmod 440 /etc/sudoers.d/devops-team

# Verify
sudo visudo -c
```

### **Scenario 5: Audit User Permissions**

```bash
#!/bin/bash
# audit-users.sh

echo "=== User Permission Audit ==="
echo "Date: $(date)"
echo

echo "=== Users with UID 0 (root privileges) ==="
awk -F: '$3 == 0 {print $1}' /etc/passwd

echo
echo "=== Users with sudo access ==="
grep -Po '^sudo:\K.*' /etc/group

echo
echo "=== Users with shell access ==="
awk -F: '$7 !~ /false|nologin/ {print $1, $7}' /etc/passwd

echo
echo "=== SUID files ==="
find / -perm -4000 -type f 2>/dev/null | head -20

echo
echo "=== Recently logged in users ==="
last | head -20

echo
echo "=== Password aging info ==="
for user in $(awk -F: '$3 >= 1000 {print $1}' /etc/passwd); do
    echo "User: $user"
    sudo chage -l $user | grep "Password expires"
done
```

---

## **10. Industry Best Practices** 🏆

### **Security Best Practices**

**1. Principle of Least Privilege:**
```bash
# Use specific sudo rules, not ALL
user ALL=(ALL) NOPASSWD: /specific/command

# Create service accounts with no shell
sudo useradd -r -s /bin/false service-account

# Use groups for permissions
# Don't add users directly to sudo, use groups
```

**2. Regular Audits:**
```bash
# Weekly audit script
#!/bin/bash
# List all users with UID 0
awk -F: '$3 == 0 {print $1}' /etc/passwd

# List users with no password
sudo awk -F: '($2 == "") {print $1}' /etc/shadow

# Find SUID files
find / -perm -4000 -type f 2>/dev/null

# Check for weak permissions
find /home -type f -perm -o+w 2>/dev/null
```

**3. Password Policies:**
```bash
# Enforce password aging
sudo chage -m 7 -M 90 -W 14 username

# In /etc/login.defs:
PASS_MAX_DAYS   90
PASS_MIN_DAYS   7
PASS_WARN_AGE   14
```

**4. SSH Key Management:**
```bash
# Use strong keys
ssh-keygen -t ed25519

# Disable password authentication
# In /etc/ssh/sshd_config:
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

**5. File Permissions:**
```bash
# Secure sensitive files
chmod 600 /etc/securefile
chmod 700 ~/.ssh
chmod 644 /etc/app.conf

# Audit permissions regularly
find /etc -type f -perm -002 2>/dev/null  # World-writable
```

---

## **11. Interview Cheat Sheet** 📝

### **Quick Commands**

```bash
# User management
sudo useradd -m username    # Create user
sudo passwd username        # Set password
sudo usermod -aG group user # Add to group
sudo userdel -r username    # Delete user

# Permissions
chmod 755 file              # Set permissions
chown user:group file       # Change owner
umask 022                   # Default permissions

# Groups
sudo groupadd groupname     # Create group
sudo gpasswd -a user group  # Add user to group

# Special permissions
chmod u+s file              # SUID
chmod g+s dir               # SGID
chmod +t dir                # Sticky bit

# Sudo
sudo command                # Run as root
sudo -u user command        # Run as user
sudo visudo                 # Edit sudoers

# ACLs
setfacl -m u:user:rwx file  # Set ACL
getfacl file                # View ACL
```

### **Common Interview Questions**

**Q1: What are the three permission categories?**
- **User (u)**: File owner
- **Group (g)**: Group members
- **Others (o)**: Everyone else

**Q2: What does chmod 755 mean?**
- Owner: rwx (7)
- Group: r-x (5)
- Others: r-x (5)

**Q3: Difference between SUID and SGID?**
- **SUID**: Execute file as owner
- **SGID**: On files - execute as group; On dirs - inherit group

**Q4: How to add user to sudo group?**
```bash
sudo usermod -aG sudo username
```

**Q5: How to find files owned by user?**
```bash
find / -user username 2>/dev/null
```

**Q6: What is umask?**
- Default permission mask
- Subtracts from 666 (files) or 777 (dirs)
- umask 022 → files: 644, dirs: 755

**Q7: How to disable user account?**
```bash
sudo passwd -l username     # Lock password
sudo usermod -s /bin/false username  # Disable shell
sudo chage -E 0 username    # Expire account
```

**Q8: Difference between /etc/passwd and /etc/shadow?**
- **/etc/passwd**: User info (world-readable)
- **/etc/shadow**: Encrypted passwords (root-only)

---

## **Summary** ✅

User and permissions management is fundamental for security:

1. **Users**: `useradd`, `usermod`, `userdel`, `passwd`
2. **Groups**: `groupadd`, `usermod -aG`, `gpasswd`
3. **Permissions**: `chmod`, `chown`, `umask`
4. **Special**: SUID, SGID, Sticky bit
5. **ACLs**: `setfacl`, `getfacl`
6. **Sudo**: Controlled root access
7. **SSH**: Key-based authentication

**Key Principles:**
- Principle of least privilege
- Use groups for access control
- Regular security audits
- Strong password policies
- Prefer SSH keys over passwords
- Document all permission changes

---

**Next Topics**: [Package Management](Package_Management.md) | [Quick Reference Cheatsheet](Linux_Commands_Cheatsheet.md)
