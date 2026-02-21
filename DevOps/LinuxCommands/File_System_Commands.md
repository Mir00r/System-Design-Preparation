# **Linux File System Commands** 📂

**Complete guide for file and directory operations in Linux**

---

## **Table of Contents** 📑
1. [Navigation Commands](#1-navigation-commands)
2. [File & Directory Operations](#2-file--directory-operations)
3. [Finding Files](#3-finding-files)
4. [File Permissions](#4-file-permissions)
5. [Compression & Archives](#5-compression--archives)
6. [Disk Usage](#6-disk-usage)
7. [Links (Symbolic & Hard)](#7-links-symbolic--hard)
8. [File Attributes](#8-file-attributes)
9. [DevOps Use Cases](#9-devops-use-cases)
10. [Troubleshooting](#10-troubleshooting)
11. [Best Practices](#11-best-practices)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Navigation Commands** 🧭

### **Basic Navigation**

```bash
# Print working directory
pwd
# Output: /home/user/projects

# Change directory
cd /var/log                  # Absolute path
cd logs                      # Relative path
cd ..                        # Parent directory
cd ../..                     # Two levels up
cd -                         # Previous directory
cd ~                         # Home directory
cd                           # Home directory (shortcut)
cd ~username                 # Another user's home

# List directory contents
ls                           # Basic list
ls -l                        # Long format (detailed)
ls -a                        # Include hidden files
ls -lh                       # Human-readable sizes
ls -lah                      # All files, human-readable
ls -lt                       # Sort by modification time
ls -lS                       # Sort by size
ls -lR                       # Recursive listing
ls -ld directory             # Directory itself, not contents
ls -li                       # Show inode numbers

# List with specific patterns
ls *.txt                     # All .txt files
ls [abc]*                    # Files starting with a, b, or c
ls file?.txt                 # file1.txt, file2.txt, etc.
```

### **Advanced Listing**

```bash
# Tree view (install first: apt install tree)
tree                         # Current directory tree
tree -L 2                    # Max depth 2
tree -a                      # Show hidden files
tree -d                      # Directories only
tree -h                      # Human-readable sizes
tree -p                      # Show permissions
tree -u                      # Show file owner
tree -D                      # Print last modification time

# Disk usage summary
du -sh *                     # Size of each item
du -h --max-depth=1         # One level deep
du -ah | sort -rh | head -20 # Top 20 largest

# List only directories
ls -d */                     # Simple method
find . -maxdepth 1 -type d  # Using find
```

---

## **2. File & Directory Operations** 📄

### **Creating Files & Directories**

```bash
# Create empty file
touch file.txt               # Create new or update timestamp
touch file1.txt file2.txt    # Multiple files
touch {file1,file2,file3}.txt # Brace expansion

# Create directory
mkdir mydir                  # Single directory
mkdir -p path/to/deep/dir   # Create parent directories
mkdir -m 755 mydir          # With specific permissions
mkdir dir1 dir2 dir3        # Multiple directories
mkdir -p project/{src,bin,lib,docs}  # Create structure

# Create file with content
echo "Hello World" > file.txt         # Overwrite
echo "Append this" >> file.txt        # Append
cat > file.txt <<EOF                   # Heredoc
Line 1
Line 2
EOF

# Create from template
cp template.txt newfile.txt
```

### **Copying Files**

```bash
# Basic copy
cp source.txt dest.txt              # Copy file
cp file.txt /path/to/destination/   # Copy to directory
cp -v file.txt dest.txt             # Verbose
cp -i file.txt dest.txt             # Interactive (prompt before overwrite)
cp -n file.txt dest.txt             # No clobber (don't overwrite)
cp -u source.txt dest.txt           # Update (copy only if newer)

# Copy directories
cp -r source_dir/ dest_dir/         # Recursive copy
cp -a source_dir/ dest_dir/         # Archive (preserve all attributes)

# Copy multiple files
cp file1.txt file2.txt destination/
cp *.txt destination/
cp {file1,file2,file3}.txt dest/

# Copy with backup
cp --backup=numbered file.txt dest.txt  # Creates file.txt.~1~

# Advanced copying
cp -p file.txt dest.txt             # Preserve attributes (timestamp, ownership)
cp -l file.txt link.txt             # Hard link instead of copy
cp -s file.txt symlink.txt          # Symbolic link
```

### **Moving & Renaming**

```bash
# Basic move/rename
mv oldname.txt newname.txt          # Rename
mv file.txt /path/to/destination/   # Move
mv -v file.txt dest/                # Verbose
mv -i file.txt dest/                # Interactive
mv -n file.txt dest/                # No clobber
mv -u file.txt dest/                # Update only

# Move multiple files
mv *.txt destination/
mv file1 file2 file3 destination/

# Rename with pattern
for file in *.txt; do
  mv "$file" "${file%.txt}.md"
done

# Bulk rename using rename command
rename 's/\.txt$/.md/' *.txt        # Perl regex
rename 'y/A-Z/a-z/' *               # Lowercase all filenames
```

### **Deleting Files**

```bash
# Remove files
rm file.txt                         # Delete file
rm -i file.txt                      # Interactive (confirm)
rm -f file.txt                      # Force (no prompt, ignore nonexistent)
rm -v file.txt                      # Verbose
rm *.txt                            # Delete all .txt files
rm -rf directory/                   # Recursive force (DANGEROUS!)

# Remove directories
rmdir emptydir                      # Remove empty directory
rm -r directory/                    # Remove directory and contents
rm -rf directory/                   # Force recursive remove

# Safe deletion alternatives
trash file.txt                      # Move to trash (if installed)
mv file.txt ~/.trash/               # Manual trash

# Delete old files
find . -name "*.log" -mtime +30 -delete    # Older than 30 days
find . -type f -mtime +7 -exec rm {} \;    # Alternative method
```

---

## **3. Finding Files** 🔍

### **find Command**

```bash
# Basic find
find .                              # All files in current directory
find /path                          # All files in path
find . -name "file.txt"            # Find by name (exact)
find . -iname "file.txt"           # Case-insensitive
find . -name "*.txt"               # Pattern matching
find . -name "file*"               # Wildcard

# Find by type
find . -type f                      # Files only
find . -type d                      # Directories only
find . -type l                      # Symbolic links
find . -type f -name "*.sh"        # Shell scripts

# Find by size
find . -size +100M                  # Larger than 100MB
find . -size -1M                    # Smaller than 1MB
find . -size 50M                    # Exactly 50MB
find . -size +1G                    # Larger than 1GB

# Find by time
find . -mtime -7                    # Modified in last 7 days
find . -mtime +30                   # Modified more than 30 days ago
find . -atime -7                    # Accessed in last 7 days
find . -ctime -7                    # Changed in last 7 days
find . -mmin -60                    # Modified in last 60 minutes

# Find by permissions
find . -perm 644                    # Exactly 644
find . -perm -644                   # At least 644
find . -perm /u+w                   # User writable

# Find by owner
find . -user john                   # Owned by user john
find . -group dev                   # Owned by group dev
find . -nouser                      # No owner (orphaned)

# Execute commands on found files
find . -name "*.tmp" -delete        # Delete all .tmp files
find . -name "*.log" -exec rm {} \; # Alternative delete
find . -type f -exec chmod 644 {} \; # Change permissions
find . -name "*.txt" -exec grep "error" {} \; # Search content
find . -name "*.txt" -ok rm {} \;   # Confirm before each deletion

# Complex find queries
find . -name "*.log" -size +10M -mtime +7 -delete
# Delete log files > 10MB and older than 7 days

find . -type f \( -name "*.tmp" -o -name "*.cache" \) -delete
# Delete .tmp OR .cache files

find . -type f -name "*.sh" -exec chmod +x {} \;
# Make all shell scripts executable
```

### **locate Command**

```bash
# Update database (run as root)
sudo updatedb

# Find files
locate file.txt                     # Fast search
locate -i file.txt                  # Case-insensitive
locate -c file.txt                  # Count matches
locate -r '\.txt$'                  # Regex pattern
locate -n 10 file                   # Limit to 10 results

# Note: locate uses database, may not show recent files
```

### **which & whereis**

```bash
# which - Find command location
which python                        # /usr/bin/python
which -a python                     # All matching paths

# whereis - Find binary, source, man pages
whereis python                      # python: /usr/bin/python /usr/lib/python
whereis -b python                   # Binary only
whereis -m python                   # Manual only
whereis -s python                   # Source only
```

---

## **4. File Permissions** 🔐

### **Understanding Permissions**

```
Permission Format:
-rwxrwxrwx

Position  | Meaning
--------- | -------
1         | File type (- file, d directory, l link)
2-4       | Owner permissions (rwx)
5-7       | Group permissions (rwx)
8-10      | Others permissions (rwx)

r = read (4)
w = write (2)
x = execute (1)
```

### **chmod - Change Permissions**

```bash
# Symbolic mode
chmod u+x script.sh                 # Add execute for owner
chmod g-w file.txt                  # Remove write for group
chmod o+r file.txt                  # Add read for others
chmod a+x script.sh                 # Add execute for all
chmod u+rw,g+r,o-rwx file.txt      # Multiple changes
chmod u=rwx,g=rx,o=r file.txt      # Set exact permissions

# Numeric mode
chmod 644 file.txt                  # rw-r--r--
chmod 755 script.sh                 # rwxr-xr-x
chmod 600 secret.txt                # rw-------
chmod 777 file.txt                  # rwxrwxrwx (NOT recommended)
chmod 400 readonly.txt              # r--------

# Recursive
chmod -R 755 directory/             # Apply to all files/subdirs
chmod -R u+w directory/             # Add write for owner recursively

# Common DevOps permissions
chmod 644 config.yml                # Config files
chmod 600 private.key               # SSH keys, secrets
chmod 755 script.sh                 # Executable scripts
chmod 700 ~/.ssh                    # SSH directory
chmod 750 /var/www/html             # Web directories
```

### **chown - Change Ownership**

```bash
# Change owner
sudo chown user file.txt            # Change owner
sudo chown user:group file.txt      # Change owner and group
sudo chown :group file.txt          # Change group only
sudo chown -R user:group directory/ # Recursive

# Real examples
sudo chown www-data:www-data /var/www/html
sudo chown -R nginx:nginx /usr/share/nginx/html
sudo chown deploy:deploy /opt/app
```

### **chgrp - Change Group**

```bash
# Change group
sudo chgrp developers file.txt      # Change group
sudo chgrp -R developers project/   # Recursive
```

### **umask - Default Permissions**

```bash
# View current umask
umask                               # 0022

# Set umask (subtract from 666 for files, 777 for directories)
umask 022                           # Files: 644, Dirs: 755
umask 077                           # Files: 600, Dirs: 700
umask 002                           # Files: 664, Dirs: 775

# Permanent umask (add to ~/.bashrc or /etc/profile)
echo "umask 022" >> ~/.bashrc
```

---

## **5. Compression & Archives** 🗜️

### **tar - Archive Files**

```bash
# Create archive
tar -cvf archive.tar files/         # Create tar archive
tar -czvf archive.tar.gz files/     # Create and compress (gzip)
tar -cjvf archive.tar.bz2 files/    # Create and compress (bzip2)
tar -cJvf archive.tar.xz files/     # Create and compress (xz)

# Extract archive
tar -xvf archive.tar                # Extract tar
tar -xzvf archive.tar.gz            # Extract gzip tar
tar -xjvf archive.tar.bz2           # Extract bzip2 tar
tar -xJvf archive.tar.xz            # Extract xz tar
tar -xzvf archive.tar.gz -C /path/  # Extract to specific directory

# List contents
tar -tvf archive.tar                # List without extracting
tar -tzvf archive.tar.gz            # List gzip archive

# Append to archive
tar -rvf archive.tar newfile.txt    # Add file to existing archive

# Extract specific files
tar -xzvf archive.tar.gz file.txt   # Extract only file.txt
tar -xzvf archive.tar.gz --wildcards '*.txt'  # Extract all .txt files

# Common tar flags
# c = create
# x = extract
# t = list
# v = verbose
# f = file
# z = gzip
# j = bzip2
# J = xz
```

### **gzip - Compress Files**

```bash
# Compress
gzip file.txt                       # Creates file.txt.gz (removes original)
gzip -k file.txt                    # Keep original file
gzip -9 file.txt                    # Maximum compression
gzip -1 file.txt                    # Fastest compression
gzip -r directory/                  # Compress all files recursively

# Decompress
gunzip file.txt.gz                  # Decompress
gzip -d file.txt.gz                 # Alternative decompress
gunzip -k file.txt.gz               # Keep compressed file

# View compressed file
zcat file.txt.gz                    # View contents
zless file.txt.gz                   # Page through contents
zgrep "pattern" file.txt.gz         # Search in compressed file
```

### **bzip2 - Better Compression**

```bash
# Compress
bzip2 file.txt                      # Creates file.txt.bz2
bzip2 -k file.txt                   # Keep original
bzip2 -9 file.txt                   # Maximum compression

# Decompress
bunzip2 file.txt.bz2               # Decompress
bzip2 -d file.txt.bz2              # Alternative

# View
bzcat file.txt.bz2                 # View contents
bzless file.txt.bz2                # Page through
bzgrep "pattern" file.txt.bz2      # Search
```

### **zip & unzip**

```bash
# Create zip
zip archive.zip file.txt            # Zip single file
zip archive.zip file1.txt file2.txt # Multiple files
zip -r archive.zip directory/       # Recurse into directories
zip -9 archive.zip files/           # Maximum compression
zip -e secret.zip files/            # Encrypt (password)

# Extract zip
unzip archive.zip                   # Extract all
unzip archive.zip -d /path/         # Extract to directory
unzip -l archive.zip                # List contents
unzip -q archive.zip                # Quiet mode
unzip archive.zip file.txt          # Extract specific file

# Update zip
zip -u archive.zip newfile.txt      # Add or update file
```

---

## **6. Disk Usage** 💿

### **df - Disk Space**

```bash
# Show disk space
df                                  # All filesystems
df -h                               # Human-readable
df -H                               # SI units (1000 instead of 1024)
df -i                               # Inode information
df -T                               # Show filesystem type
df /path                            # Specific filesystem

# Output columns
df -h
# Filesystem | Size | Used | Avail | Use% | Mounted on
```

### **du - Directory Usage**

```bash
# Show directory usage
du                                  # Current directory
du -h                               # Human-readable
du -sh                              # Summary only
du -sh *                            # Size of each item
du -ah                              # All files
du -h --max-depth=1                 # One level deep
du -h --max-depth=2 | sort -rh      # Two levels, sorted

# Find largest directories
du -h | sort -rh | head -10         # Top 10 largest
du -ah | sort -rh | head -20        # Top 20 files/dirs

# Exclude patterns
du -h --exclude="*.tmp"             # Exclude .tmp files
du -h --exclude-from=exclude.txt    # Exclude from file

# Show with timestamps
du -sh --time *                     # With modification time
```

### **ncdu - Interactive Disk Usage**

```bash
# Install first
sudo apt install ncdu

# Use ncdu (ncurses du)
ncdu /                              # Scan root
ncdu /var/log                       # Scan specific directory
ncdu -x / # Same filesystem only

# Interactive navigation:
# Arrow keys to navigate
# d to delete
# n to sort by name
# s to sort by size
# q to quit
```

---

## **7. Links (Symbolic & Hard)** 🔗

### **Symbolic Links (Soft Links)**

```bash
# Create symbolic link
ln -s /path/to/original linkname    # Create symlink
ln -s /var/log/app/app.log latest.log

# Real-world examples
ln -s /usr/bin/python3.11 /usr/bin/python  # Python version
ln -s /opt/app/current /app          # Application directory
ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/mysite

# List symlinks
ls -l linkname # Shows: linkname -> /path/to/original
find . -type l                       # Find all symlinks
find . -type l -ls                   # Detailed symlink listing

# Remove symlink
rm linkname                          # Remove link (not original!)
unlink linkname                      # Alternative

# Update symlink
ln -sf /new/path linkname           # Force update symlink
```

### **Hard Links**

```bash
# Create hard link
ln /path/to/original hardlink       # Create hard link

# Characteristics
# - Both files point to same inode
# - Deleting one doesn't affect the other
# - Can't cross filesystems
# - Can't link directories

# View hard links
ls -li file1 file2                  # Same inode number
find . -samefile file1              # Find all hard links to file1
find . -inum 12345                  # Find by inode number
```

### **Symlink vs Hard Link**

```
Symbolic Link:
✅ Can link directories
✅ Can cross filesystems
✅ Shows broken link if target deleted
❌ Extra overhead (indirection)

Hard Link:
✅ No overhead (direct inode reference)
✅ Survives original file deletion
❌ Can't link directories
❌ Can't cross filesystems

Use symlinks for most cases.
Use hard links for backup/deduplication.
```

---

## **8. File Attributes** 🏷️

### **file - Determine File Type**

```bash
# Check file type
file filename                       # Determine file type
file *                              # All files in directory
file -b filename                    # Brief mode (no filename)
file -i filename                    # MIME type
file -z compressed.gz               # Look inside compressed files
```

### **stat - File Statistics**

```bash
# Detailed file information
stat file.txt

# Output includes:
# - Size
# - Inode
# - Links
# - Permissions (octal and symbolic)
# - Owner/Group
# - Timestamps (access, modify, change)

# Format output
stat -c '%n %s' file.txt           # Name and size
stat -c '%a' file.txt              # Permissions (octal)
stat -c '%U:%G' file.txt           # Owner:Group
```

### **lsattr & chattr - Extended Attributes**

```bash
# List attributes
lsattr file.txt                     # Show attributes
lsattr -d directory/                # Directory itself

# Change attributes
sudo chattr +i file.txt             # Immutable (can't modify/delete)
sudo chattr -i file.txt             # Remove immutable
sudo chattr +a file.txt             # Append-only
sudo chattr +A file.txt             # No atime update

# Common attributes:
# i = immutable
# a = append only
# A = no atime updates
# d = no dump
# s = secure deletion
```

---

## **9. DevOps Use Cases** 🚀

### **Log Management**

```bash
# Find and clean old logs
find /var/log -name "*.log" -mtime +30 -exec gzip {} \;
find /var/log -name "*.gz" -mtime +90 -delete

# Monitor log file sizes
du -sh /var/log/* | sort -rh | head -10

# Archive application logs
tar -czf "logs-$(date +%Y%m%d).tar.gz" /var/log/app/*.log
```

### **Deployment Scripts**

```bash
#!/bin/bash
# Deployment script

APP_DIR="/opt/myapp"
BACKUP_DIR="/backup/myapp"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

# Backup current version
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/app-${TIMESTAMP}.tar.gz" "$APP_DIR"

# Deploy new version
cd "$APP_DIR" || exit
git pull origin main

# Cleanup old backups (keep last 10)
cd "$BACKUP_DIR" || exit
ls -t | tail -n +11 | xargs rm -f
```

### **Disk Space Monitoring**

```bash
#!/bin/bash
# Alert if disk usage > 80%

THRESHOLD=80
USAGE=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

if [ "$USAGE" -gt "$THRESHOLD" ]; then
    echo "WARNING: Disk usage is ${USAGE}%"
    # Send alert (email, Slack, etc.)
    du -sh /* | sort -rh | head -10
fi
```

### **File Synchronization**

```bash
# rsync - Efficient file sync
rsync -avz /source/ /destination/     # Archive, verbose, compress
rsync -avz --delete source/ dest/     # Delete extraneous files
rsync -avz user@remote:/path/ /local/ # Remote to local
rsync -avz --exclude='*.log' src/ dst/ # Exclude patterns

# Backup with rsync
rsync -avz --link-dest=/backup/previous /data/ /backup/current/
```

---

## **10. Troubleshooting** 🔧

### **Permission Denied Errors**

```bash
# Check permissions
ls -l file.txt

# Fix permissions
chmod 644 file.txt                  # File permissions
chmod 755 directory/                # Directory permissions

# Fix ownership
sudo chown user:group file.txt

# Check effective permissions
namei -l /path/to/file              # Show permissions of all path components
```

### **Disk Full Issues**

```bash
# Find what's using space
df -h                               # Overall disk usage
du -sh /*  | sort -rh               # Top-level directories
du -ah /var | sort -rh | head -20   # Largest files/dirs

# Find large files
find / -type f -size +1G 2>/dev/null
find /var -type f -size +100M -exec ls -lh {} \; | sort -k5 -rh

# Check for deleted but open files
lsof | grep deleted
lsof +L1                            # Files with no link count

# Clear space
sudo journalctl --vacuum-size=100M  # Limit journal size
sudo apt clean                      # Clean package cache
docker system prune -a              # Clean Docker (if installed)
```

### **File Not Found**

```bash
# Search for file
find / -name "filename" 2>/dev/null
locate filename
whereis command
which command

# Check if file was deleted
grep "filename" /root/.bash_history
```

---

## **11. Best Practices** ⭐

### **Do's**

```
✅ Use absolute paths in scripts
✅ Quote variable expansions: "$variable"
✅ Check if command/file exists before using
✅ Use --verbose flag when learning
✅ Test destructive commands with --dry-run first
✅ Use version control for important files
✅ Regular backups before major changes
✅ Document permission requirements
✅ Use meaningful file names
✅ Organize files in logical directory structures
```

### **Don'ts**

```
❌ Don't use rm -rf without double-checking
❌ Don't run chmod 777 on everything
❌ Don't store secrets in plain text files
❌ Don't use spaces in filenames (use underscores/hyphens)
❌ Don't modify files in /var/log directly
❌ Don't ignore file permissions
❌ Don't hardcode paths (use variables)
❌ Don't delete without backup
```

### **Safety Tips**

```bash
# Safe deletion
alias rm='rm -i'                    # Prompt before delete
mkdir ~/.trash
alias rm='mv -t ~/.trash'          # Move to trash instead

# Safe overwrite
cp -i source dest                   # Prompt before overwrite
mv -i old new                       # Prompt before overwrite

# Test before execute
find . -name "*.tmp" -print        # Test before -delete

# Backup before changes
cp file.txt file.txt.backup        # Quick backup
tar -czf backup.tar.gz directory/  # Archive backup
```

---

## **12. Interview Cheat Sheet** 🎯

### **Q1: Difference between hard and symbolic links?**
```
Symbolic Link (Soft Link):
- Points to pathname
- Can cross filesystems
- Can link directories
- Breaks if target deleted
- Command: ln -s target link

Hard Link:
- Points to same inode
- Same filesystem only
- Can't link directories
- Survives target deletion
- Command: ln target link

Example:
ln -s /var/log/app/current.log latest.log  # Symlink
ln data.txt data-hardlink.txt              # Hard link
```

### **Q2: How to find files modified in last 7 days?**
```
find /path -type f -mtime -7

Explanation:
- /path: Starting directory
- -type f: Files only
- -mtime -7: Modified less than 7 days ago

Other time options:
-atime: Access time
-ctime: Change time (metadata)
-mmin: Minutes instead of days

Example:
find /var/log -name "*.log" -mtime -1  # Logs from last 24h
```

### **Q3: Explain file permissions (644, 755)?**
```
Format: rwx rwx rwx (Owner Group Others)
Numeric: r=4, w=2, x=1

644 = rw-r--r--
- Owner: read(4) + write(2) = 6
- Group: read(4) = 4
- Others: read(4) = 4
Use: Regular files, configs

755 = rwxr-xr-x
- Owner: read(4) + write(2) + execute(1) = 7
- Group: read(4) + execute(1) = 5
- Others: read(4) + execute(1) = 5
Use: Executables, directories

600 = rw-------
- Owner only can read/write
Use: Private keys, secrets

Commands:
chmod 644 file.txt
chmod 755 script.sh
chmod 600 ~/.ssh/id_rsa
```

### **Q4: How to free up disk space?**
```
1. Find what's using space:
   df -h                          # Overall usage
   du -sh /* | sort -rh | head    # Top directories
   
2. Find large files:
   find / -type f -size +1G 2>/dev/null
   
3. Clear caches:
   sudo apt clean                 # APT cache
   docker system prune -a         # Docker
   sudo journalctl --vacuum-time=7d  # Logs
   
4. Remove old files:
   find /tmp -mtime +7 -delete
   find /var/log -name "*.gz" -mtime +90 -delete
   
5. Check for deleted but open files:
   lsof +L1
```

### **Q5: Common commands for everyday use?**
```
Navigation:
cd /path        # Change directory
pwd             # Print working directory
ls -lah         # List files

File Operations:
cp src dst      # Copy
mv old new      # Move/rename
rm file         # Delete
mkdir -p path   # Create directory
touch file      # Create empty file

Searching:
find . -name "*.txt"           # Find files
grep "pattern" file            # Search in file
locate filename                # Fast find

Permissions:
chmod 755 file                 # Change permissions
chown user:group file          # Change owner

Disk Usage:
d -h            # Disk space
du -sh *        # Directory sizes

Compression:
tar -czf archive.tar.gz dir/   # Create archive
tar -xzf archive.tar.gz        # Extract archive
```

---

## **Next Steps** 📚

- **[Text Processing Commands](Text_Processing_Commands.md)** - grep, sed, awk, log analysis
- **[System Monitoring](System_Monitoring_Commands.md)** - Performance and diagnostics  
- **[Process Management](Process_Management.md)** - Process control and scheduling
- **[Network Commands](Network_Commands.md)** - Networking and troubleshooting

---

**📂 Master File System Commands for Efficient Linux Administration!**

*Understanding file operations is fundamental for every DevOps engineer and system administrator.*
