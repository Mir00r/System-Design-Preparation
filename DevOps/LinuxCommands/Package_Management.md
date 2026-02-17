# **Linux Package Management – Complete Guide for DevOps** 📦🚀

Master package management across different Linux distributions - essential skills for DevOps engineers installing software, managing dependencies, and maintaining system consistency.

---

## **Table of Contents** 📑
1. [Package Management Fundamentals](#1-package-management-fundamentals)
2. [APT (Debian/Ubuntu)](#2-apt-debianubuntu)
3. [YUM/DNF (RHEL/CentOS/Fedora)](#3-yumdnf-rhelcentosfedora)
4. [Snap Packages](#4-snap-packages)
5. [Flatpak](#5-flatpak)
6. [AppImage](#6-appimage)
7. [Source Compilation](#7 -source-compilation)
8. [Repository Management](#8-repository-management)
9. [Practical DevOps Scenarios](#9-practical-devops-scenarios)
10. [Industry Best Practices](#10-industry-best-practices)
11. [Interview Cheat Sheet](#11-interview-cheat-sheet)

---

## **1. Package Management Fundamentals** 🎯

### **Package Managers by Distribution**

| Distribution | Package Manager | Package Format |
|--------------|-----------------|----------------|
| **Ubuntu/Debian** | APT, dpkg | .deb |
| **RHEL/CentOS** | YUM, RPM | .rpm |
| **Fedora** | DNF, RPM | .rpm |
| **Arch** | Pacman | .pkg.tar.xz |
| **SUSE** | Zypper, RPM | .rpm |
| **Alpine** | APK | .apk |

### **Package Management Layers**

```
High-Level:  apt, yum, dnf     (Dependency resolution, repositories)
             ↓
Low-Level:   dpkg, rpm         (Install/remove individual packages)
             ↓
Files:       .deb, .rpm         (Package archives)
```

### **Key Concepts**

**Repository:** Server hosting packages
**Dependency:** Required package for another to work
**Cache:** Local copy of package metadata
**Hold:** Prevent package from being upgraded

---

## **2. APT (Debian/Ubuntu)** 📦

### **apt - Modern Package Manager**

**Update and Upgrade:**
```bash
sudo apt update                              # Update package index
sudo apt upgrade                            # Upgrade all packages
sudo apt full-upgrade                        # Upgrade with dependency changes
sudo apt dist-upgrade                        # Distribution upgrade
sudo apt autoremove                          # Remove unused dependencies
sudo apt autoclean                           # Clean old package files
```

**Install and Remove:**
```bash
sudo apt install package                     # Install package
sudo apt install package1 package2 package3  # Multiple packages
sudo apt install package=version             # Specific version
sudo apt install -y package                  # Assume yes (non-interactive)
sudo apt install --no-install-recommends pkg # Without recommended packages
sudo apt reinstall package                   # Reinstall package
sudo apt remove package                      # Remove package
sudo apt purge package                       # Remove with config files
sudo apt remove --purge package              # Same as purge
```

**Search and Info:**
```bash
apt search keyword                           # Search for package
apt show package                             # Show package details
apt list                                     # List all packages
apt list --installed                         # List installed packages
apt list --upgradable                        # List upgradable packages
apt-cache policy package                     # Show version information
apt-cache depends package                    # Show dependencies
apt-cache rdepends package                   # Show reverse dependencies
```

**Practical Examples:**
```bash
# Install nginx
sudo apt update && sudo apt install -y nginx

# Install specific version
sudo apt install nginx=1.18.0-0ubuntu1

# Search for python packages
apt search python3 | grep -i dev

# Show package info
apt show docker.io

# List installed packages
apt list --installed | grep nginx

# Remove package and dependencies
sudo apt remove --purge nginx
sudo apt autoremove
```

### **dpkg - Low-Level Package Manager**

```bash
dpkg -i package.deb                          # Install .deb package
dpkg -r package                              # Remove package
dpkg -P package                              # Purge package
dpkg -l                                      # List installed packages
dpkg -l | grep package                       # Search installed
dpkg -L package                              # List package files
dpkg -S /path/to/file                        # Find package owning file
dpkg -s package                              # Package status
dpkg --configure -a                          # Fix broken packages
dpkg --get-selections                        # List selections
```

**Fix Broken Dependencies:**
```bash
sudo dpkg -i package.deb
# If dependency error:
sudo apt install -f                          # Fix dependencies
```

**Examples:**
```bash
# Install .deb file
wget https://example.com/package.deb
sudo dpkg -i package.deb
sudo apt install -f  # Fix dependencies

# Find which package provides file
dpkg -S /usr/bin/python3

# List all files from package
dpkg -L nginx

# Check package status
dpkg -s nginx | grep Status
```

### **apt-get and apt-cache (Legacy)**

**apt-get:**
```bash
sudo apt-get update                          # Update index
sudo apt-get install package                 # Install
sudo apt-get remove package                  # Remove
sudo apt-get purge package                   # Purge
sudo apt-get upgrade                         # Upgrade
sudo apt-get dist-upgrade                    # Distribution upgrade
sudo apt-get autoremove                      # Remove unused
sudo apt-get clean                           # Clean cache
sudo apt-get autoclean                       # Clean old cache
```

**apt-cache:**
```bash
apt-cache search package                     # Search
apt-cache show package                       # Show info
apt-cache policy package                     # Show versions
apt-cache depends package                    # Dependencies
apt-cache rdepends package                   # Reverse dependencies
```

**Note:** `apt` combines `apt-get` and `apt-cache` with better UI.

### **Package Holding**

```bash
# Hold package from upgrade
sudo apt-mark hold nginx
sudo apt-mark unhold nginx                   # Unhold

# Show held packages
apt-mark showhold

# Manual hold
sudo apt-mark manual package                 # Mark as manually installed
sudo apt-mark auto package                   # Mark as automatically installed
```

---

## **3. YUM/DNF (RHEL/CentOS/Fedora)** 📦

### **yum - Yellowdog Updater Modified (CentOS 7)**

**Update and Upgrade:**
```bash
sudo yum update                              # Update all packages
sudo yum upgrade                             # Same as update
sudo yum check-update                        # Check for updates
sudo yum update-minimal                      # Minimal update
```

**Install and Remove:**
```bash
sudo yum install package                     # Install package
sudo yum install -y package                  # Assume yes
sudo yum install package1 package2           # Multiple packages
sudo yum localinstall package.rpm            # Install local .rpm
sudo yum reinstall package                   # Reinstall package
sudo yum remove package                      # Remove package
sudo yum autoremove                          # Remove unused dependencies
```

**Search and Info:**
```bash
yum search keyword                           # Search packages
yum info package                             # Show package info
yum list                                     # List all packages
yum list installed                           # List installed
yum list available                           # List available
yum provides /path/to/file                   # Find package for file
yum deplist package                          # List dependencies
```

**Group Management:**
```bash
yum grouplist                                # List package groups
sudo yum groupinstall "Development Tools"    # Install group
sudo yum groupremove "Development Tools"     # Remove group
yum groupinfo "Development Tools"            # Group info
```

**Repository Management:**
```bash
yum repolist                                 # List enabled repos
yum repolist all                             # List all repos
sudo yum-config-manager --enable repo        # Enable repo
sudo yum-config-manager --disable repo       # Disable repo
sudo yum clean all                           # Clean cache
sudo yum makecache                           # Rebuild cache
```

**Examples:**
```bash
# Install Docker
sudo yum install -y docker

# Install development tools
sudo yum groupinstall "Development Tools"

# Search for nginx
yum search nginx

# Find package providing command
yum provides /usr/bin/git

# List installed packages
yum list installed | grep python
```

### **dnf - Dandified YUM (Fedora, RHEL 8+)**

```bash
sudo dnf update                              # Update all
sudo dnf upgrade                             # Upgrade all
sudo dnf install package                     # Install
sudo dnf install -y package                  # Assume yes
sudo dnf reinstall package                   # Reinstall
sudo dnf remove package                      # Remove
sudo dnf autoremove                          # Remove unused

# Search and info
dnf search package                           # Search
dnf info package                             # Show info
dnf list                                     # List all
dnf list installed                           # List installed
dnf list available                           # List available
dnf provides /path/to/file                   # Find package

# Group management
dnf group list                               # List groups
sudo dnf group install "Development Tools"   # Install group
sudo dnf group remove "Development Tools"    # Remove group

# Repository management
dnf repolist                                 # List repos
dnf repolist all                             # All repos
sudo dnf config-manager --enable repo        # Enable repo
sudo dnf config-manager --disable repo       # Disable repo
sudo dnf clean all                           # Clean cache
sudo dnf makecache                           # Rebuild cache
```

**dnf Advantages over yum:**
- Faster (C, not Python)
- Better dependency resolution
- More accurate
- Default in Fedora/RHEL 8+

### **rpm - RPM Package Manager**

```bash
rpm -i package.rpm                           # Install
rpm -U package.rpm                           # Upgrade
rpm -e package                               # Erase/remove
rpm -qa                                      # Query all installed
rpm -qa | grep package                       # Search installed
rpm -qi package                              # Query package info
rpm -ql package                              # Query package files
rpm -qf /path/to/file                        # Query file owner
rpm -qR package                              # Query requirements (dependencies)
rpm -V package                               # Verify package
rpm -Va                                      # Verify all packages
```

**Examples:**
```bash
# Install .rpm file
sudo rpm -ivh package.rpm

# Upgrade package
sudo rpm -Uvh package.rpm

# List all installed packages
rpm -qa

# Find package owning file
rpm -qf /usr/bin/docker

# List files in package
rpm -ql docker

# Check package dependencies
rpm -qR docker
```

---

## **4. Snap Packages** 📸

**Snap** - Universal package format by Canonical

**Install Snapd:**
```bash
# Ubuntu (usually pre-installed)
sudo apt install snapd

# CentOS/RHEL 7/8
sudo yum install snapd
sudo systemctl enable --now snapd.socket
sudo ln -s /var/lib/snapd/snap /snap

# Fedora
sudo dnf install snapd
```

**Snap Commands:**
```bash
snap find package                            # Search
snap list                                    # List installed snaps
snap info package                            # Show info
sudo snap install package                    # Install
sudo snap install --classic package          # Install with classic confinement
sudo snap install --channel=edge package     # Install from edge channel
sudo snap refresh package                    # Update snap
sudo snap refresh                            # Update all snaps
sudo snap remove package                     # Remove snap
sudo snap revert package                     # Revert to previous version
snap changes                                 # Show recent changes
snap version                                 # Snap version
```

**Examples:**
```bash
# Install Docker
sudo snap install docker

# Install VS Code
sudo snap install --classic code

# Update all snaps
sudo snap refresh

# Remove snap
sudo snap remove docker

# List installed snaps
snap list
```

**Snap Channels:**
```
stable     Production-ready
candidate  Release candidate
beta       Beta testing
edge       Latest development
```

---

## **5. Flatpak** 📦

**Flatpak** - Universal package format with sandboxing

**Install Flatpak:**
```bash
# Ubuntu/Debian
sudo apt install flatpak

# Fedora
sudo dnf install flatpak

# CentOS/RHEL
sudo yum install flatpak
```

**Add Flathub Repository:**
```bash
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
```

**Flatpak Commands:**
```bash
flatpak search package                       # Search
flatpak list                                 # List installed
flatpak info package                         # Show info
flatpak install flathub package              # Install from Flathub
sudo flatpak install package                 # System-wide install
flatpak install --user package               # User install
flatpak update                               # Update all
flatpak update package                       # Update specific
flatpak uninstall package                    # Uninstall
flatpak remote-list                          # List remotes
flatpak run package                          # Run application
```

**Examples:**
```bash
# Install Spotify
flatpak install flathub com.spotify.Client

# Install GIMP
flatpak install flathub org.gimp.GIMP

# Update all
flatpak update

# Run application
flatpak run com.spotify.Client

# List installed
flatpak list

# Uninstall
flatpak uninstall com.spotify.Client
```

---

## **6. AppImage** 🖼️

**AppImage** - Portable application format

**No Installation Required:**
```bash
# Download AppImage
wget https://example.com/app.AppImage

# Make executable
chmod +x app.AppImage

# Run
./app.AppImage
```

**Integration:**
```bash
# Install AppImageLauncher (optional, for desktop integration)
sudo add-apt-repository ppa:appimagelauncher-team/stable
sudo apt update
sudo apt install appimagelauncher
```

**Example:**
```bash
# Download Etcher
wget https://github.com/balena-io/etcher/releases/download/v1.7.9/balenaEtcher-1.7.9-x64.AppImage

# Make executable
chmod +x balenaEtcher-1.7.9-x64.AppImage

# Run
./balenaEtcher-1.7.9-x64.AppImage
```

---

## **7. Source Compilation** 🔨

**Compile from Source:**

```bash
# Install build tools
# Ubuntu/Debian
sudo apt install build-essential

# RHEL/CentOS
sudo yum groupinstall "Development Tools"

# Fedora
sudo dnf groupinstall "Development Tools"
```

**Standard Build Process:**
```bash
# 1. Download source
wget https://example.com/package-1.0.tar.gz

# 2. Extract
tar -xzf package-1.0.tar.gz
cd package-1.0

# 3. Configure
./configure
./configure --prefix=/usr/local  # Custom install location

# 4. Compile
make

# 5. Install
sudo make install

# 6. Uninstall (if supported)
sudo make uninstall
```

**Alternative: checkinstall**
```bash
# Install checkinstall
sudo apt install checkinstall  # Ubuntu/Debian
sudo yum install checkinstall  # RHEL/CentOS

# Use instead of 'make install'
sudo checkinstall              # Creates .deb or .rpm package
```

**Example - Compile nginx from source:**
```bash
# Install dependencies
sudo apt install build-essential libpcre3-dev libssl-dev zlib1g-dev

# Download
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar -xzf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# Configure
./configure --prefix=/usr/local/nginx --with-http_ssl_module

# Compile
make

# Install
sudo make install

# Run
/usr/local/nginx/sbin/nginx
```

---

## **8. Repository Management** 🗄️

### **APT Repositories (Debian/Ubuntu)**

**Repository Files:**
```bash
/etc/apt/sources.list            # Main sources
/etc/apt/sources.list.d/         # Additional sources
```

**Add Repository:**
```bash
# Method 1: add-apt-repository
sudo add-apt-repository ppa:nginx/stable
sudo apt update

# Method 2: Edit sources.list
sudo nano /etc/apt/sources.list
# Add: deb http://example.com/ubuntu focal main

# Method 3: Create new file
echo "deb http://example.com/ubuntu focal main" | sudo tee /etc/apt/sources.list.d/example.list

# Add GPG key
wget -qO - https://example.com/gpg | sudo apt-key add -
# Or (new method)
wget -qO - https://example.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/example.gpg
```

**Remove Repository:**
```bash
sudo add-apt-repository --remove ppa:nginx/stable
# Or
sudo rm /etc/apt/sources.list.d/example.list
sudo apt update
```

**Examples:**
```bash
# Add Docker repository
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update
sudo apt install docker-ce

# Add Nginx PPA
sudo add-apt-repository ppa:nginx/stable
sudo apt update
sudo apt install nginx
```

### **YUM/DNF Repositories (RHEL/CentOS/Fedora)**

**Repository Files:**
```bash
/etc/yum.repos.d/                # Repository definitions
```

**Add Repository:**
```bash
# Add EPEL repository (CentOS/RHEL)
sudo yum install epel-release

# Create custom repo file
sudo nano /etc/yum.repos.d/custom.repo
# Add:
# [custom-repo]
# name=Custom Repository
# baseurl=https://example.com/repo/
# enabled=1
# gpgcheck=1
# gpgkey=https://example.com/gpg

# Add GPG key
sudo rpm --import https://example.com/gpg
```

**Disable/Enable Repository:**
```bash
sudo yum-config-manager --disable repo-name
sudo yum-config-manager --enable repo-name

# DNF
sudo dnf config-manager --disable repo-name
sudo dnf config-manager --enable repo-name
```

**Examples:**
```bash
# Add Docker repository (CentOS)
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce

# Add EPEL
sudo yum install epel-release

# List repositories
yum repolist
```

---

## **9. Practical DevOps Scenarios** 🛠️

### **Scenario 1: Automate Package Installation**

```bash
#!/bin/bash
# install-devops-tools.sh

# Update system
echo "Updating system..."
sudo apt update && sudo apt upgrade -y

# Install essential packages
PACKAGES=(
    git
    vim
    curl
    wget
    htop
    net-tools
    docker.io
    python3-pip
    build-essential
)

echo "Installing packages: ${PACKAGES[*]}"
sudo apt install -y "${PACKAGES[@]}"

# Clean up
sudo apt autoremove -y
sudo apt autoclean

echo "Installation complete!"
```

### **Scenario 2: Maintain Consistent Packages Across Servers**

```bash
# On source server - export installed packages
dpkg --get-selections > installed-packages.txt

# Transfer to target server
scp installed-packages.txt user@target:/tmp/

# On target server - install same packages
sudo dpkg --set-selections < /tmp/installed-packages.txt
sudo apt-get dselect-upgrade
```

### **Scenario 3: Package Version Pinning**

**APT Pinning:**
```bash
# Create pin file
sudo nano /etc/apt/preferences.d/nginx-pin

# Add content:
Package: nginx
Pin: version 1.18.*
Pin-Priority: 1001

# Update and install
sudo apt update
sudo apt install nginx
```

**YUM Version Lock:**
```bash
# Install plugin
sudo yum install yum-plugin-versionlock

# Lock version
sudo yum versionlock nginx

# List locked
sudo yum versionlock list

# Unlock
sudo yum versionlock delete nginx
```

### **Scenario 4: Clean Up Unused Packages**

```bash
#!/bin/bash
# cleanup-packages.sh

echo "Removing unused packages..."
sudo apt autoremove -y

echo "Cleaning package cache..."
sudo apt clean
sudo apt autoclean

echo "Removing old kernels..."
dpkg -l 'linux-*' | sed '/^ii/!d;/'"$(uname -r | sed "s/\(.*\)-\([^0-9]\+\)/\1/")"'/d;s/^[^ ]* [^ ]* \([^ ]*\).*/\1/;/[0-9]/!d' | xargs sudo apt-get -y purge

echo "Disk space freed:"
df -h /
```

### **Scenario 5: Security Updates Only**

```bash
# Ubuntu/Debian - unattended-upgrades
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades

# Configure (auto security updates)
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades

# Manual security update
sudo apt update
sudo apt upgrade -y

# CentOS/RHEL - security updates only
sudo yum update --security
```

---

## **10. Industry Best Practices** 🏆

### **1. Always Update Before Install**

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install package

# RHEL/CentOS/Fedora
sudo yum update && sudo yum install package
```

### **2. Use Package Managers, Not Manual Downloads**

```bash
# Good
sudo apt install nginx

# Bad
wget nginx.tar.gz
tar -xzf nginx.tar.gz
# (harder to update, no dependency management)
```

### **3. Automate Updates (with caution)**

```bash
# Security updates only
sudo apt install unattended-upgrades

# Configure in /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
```

### **4. Test Updates in Staging**

```bash
# Never update production directly
# 1. Test in dev
# 2. Test in staging
# 3. Then production

# Use version pinning for critical packages
```

### **5. Document Package Sources**

```bash
# Keep list of custom repositories
cat /etc/apt/sources.list.d/*

# Document installed packages
dpkg -l > installed-packages-$(date +%Y%m%d).txt
```

### **6. Regular Cleanup**

```bash
# Weekly cleanup script
#!/bin/bash
sudo apt autoremove -y
sudo apt autoclean
sudo journalctl --vacuum-time=7d
```

### **7. Security Best Practices**

```bash
# Only install from trusted sources
# Verify GPG keys
# Keep system updated
# Use official repositories when possible
# Audit installed packages regularly

# Check for security updates
sudo apt list --upgradable | grep -i security
```

---

## **11. Interview Cheat Sheet** 📝

### **Quick Command Reference**

**APT (Ubuntu/Debian):**
```bash
sudo apt update                  # Update package index
sudo apt install package         # Install
sudo apt remove package          # Remove
sudo apt search package          # Search
apt list --installed             # List installed
```

**YUM/DNF (RHEL/CentOS/Fedora):**
```bash
sudo yum update                  # Update all
sudo yum install package         # Install
sudo yum remove package          # Remove
yum search package               # Search
yum list installed               # List installed
```

**Package Files:**
```bash
dpkg -i package.deb              # Install .deb
rpm -ivh package.rpm             # Install .rpm
dpkg -l                          # List installed (Debian)
rpm -qa                          # List installed (RHEL)
```

### **Common Interview Questions**

**Q1: How to update all packages?**
```bash
# Ubuntu/Debian
sudo apt update && sudo apt upgrade

# RHEL/CentOS
sudo yum update
```

**Q2: How to find which package provides a file?**
```bash
# Debian/Ubuntu
dpkg -S /path/to/file

# RHEL/CentOS
yum provides /path/to/file
rpm -qf /path/to/file
```

**Q3: How to list all installed packages?**
```bash
# Debian/Ubuntu
dpkg -l
apt list --installed

# RHEL/CentOS
rpm -qa
yum list installed
```

**Q4: How add a repository?**
```bash
# Ubuntu/Debian
sudo add-apt-repository ppa:name/ppa

# RHEL/CentOS
sudo yum-config-manager --add-repo URL
```

**Q5: Difference between apt and apt-get?**
- **apt**: Modern, user-friendly, progress bar
- **apt-get**: Legacy, script-friendly, stable interface

**Q6: How to prevent package from updating?**
```bash
# Ubuntu/Debian
sudo apt-mark hold package

# RHEL/CentOS
sudo yum versionlock package
```

**Q7: How to clean package cache?**
```bash
# Ubuntu/Debian
sudo apt clean
sudo apt autoclean

# RHEL/CentOS
sudo yum clean all
```

---

## **Summary** ✅

Package management is essential for system administration:

1. **APT** (Debian/Ubuntu): `apt`, `dpkg`
2. **YUM/DNF** (RHEL/CentOS/Fedora): `yum`, `dnf`, `rpm`
3. **Universal**: `snap`, `flatpak`, `AppImage`
4. **Source**: `configure`, `make`, `make install`
5. **Repositories**: Manage package sources

**Key Principles:**
- Always update before install
- Use official repositories
- Test updates in staging
- Automate security updates
- Regular cleanup
- Document customizations

---

**Next Topics**: [Linux Commands  Cheatsheet](Linux_Commands_Cheatsheet.md)
