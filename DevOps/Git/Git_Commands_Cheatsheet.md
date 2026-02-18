# **Git Commands Cheatsheet – Quick Reference** 📋⚡

Comprehensive quick reference for all essential Git commands - your go-to guide for daily Git operations in DevOps environments.

---

## **Table of Contents** 📑
1. [Setup & Configuration](#1-setup--configuration)
2. [Creating & Cloning](#2-creating--cloning)
3. [Basic Operations](#3-basic-operations)
4. [Branching](#4-branching)
5. [Merging](#5-merging)
6. [Remote Operations](#6-remote-operations)
7. [Stashing](#7-stashing)
8. [History & Logs](#8-history--logs)
9. [Undoing Changes](#9-undoing-changes)
10. [Advanced Commands](#10-advanced-commands)
11. [Useful Aliases](#11-useful-aliases)
12. [One-Liners](#12-one-liners)

---

## **1. Setup & Configuration** ⚙️

### **Initial Setup:**
```bash
# Set user name
git config --global user.name "Your Name"

# Set email
git config --global user.email "your.email@example.com"

# Set default editor
git config --global core.editor "vim"
git config --global core.editor "code --wait"  # VS Code

# Set default branch name
git config --global init.defaultBranch main

# Line ending (Linux/Mac)
git config --global core.autocrlf input

# Line ending (Windows)
git config --global core.autocrlf true
```

### **View Configuration:**
```bash
# List all config
git config --list
git config -l

# List with file locations
git config --list --show-origin

# Get specific value
git config user.name
git config user.email

# Edit config file
git config --global --edit
```

### **Configuration Levels:**
```bash
# System (all users)
git config --system <key> <value>

# Global (current user)
git config --global <key> <value>

# Local (current repository)
git config --local <key> <value>
```

---

## **2. Creating & Cloning** 🎬

### **Initialize Repository:**
```bash
# Create new repository
git init

# Initialize with specific branch
git init -b main

# Initialize in specific directory
git init my-project

# Create bare repository (for servers)
git init --bare repo.git
```

### **Clone Repository:**
```bash
# Clone repository
git clone <url>

# Clone to specific directory
git clone <url> my-folder

# Clone specific branch
git clone -b develop <url>

# Shallow clone (limited history)
git clone --depth 1 <url>

# Clone with submodules
git clone --recursive <url>

# Clone single branch
git clone --single-branch -b main <url>
```

---

## **3. Basic Operations** 📝

### **Status & Info:**
```bash
# Show status
git status

# Short status
git status -s

# Show branch info
git status -b
```

### **Staging:**
```bash
# Stage specific file
git add file.txt

# Stage all files
git add .

# Stage all (including deleted)
git add -A

# Stage modified and deleted (not new)
git add -u

# Interactive staging
git add -i

# Patch mode (stage parts)
git add -p
```

### **Committing:**
```bash
# Commit with message
git commit -m "Commit message"

# Commit all modified files
git commit -am "Message"

# Amend last commit
git commit --amend

# Amend without changing message
git commit --amend --no-edit

# Commit with detailed message
git commit  # Opens editor

# Empty commit
git commit --allow-empty -m "Trigger CI"
```

### **Viewing Changes:**
```bash
# Show unstaged changes
git diff

# Show staged changes
git diff --staged
git diff --cached

# Show changes in specific file
git diff file.txt

# Show changes between commits
git diff commit1 commit2

# Show changes between branches
git diff branch1 branch2

# Word diff
git diff --word-diff

# Stat summary
git diff --stat
```

---

## **4. Branching** 🌿

### **Branch Operations:**
```bash
# List branches
git branch

# List all (local + remote)
git branch -a

# List remote only
git branch -r

# Create branch
git branch feature-name

# Create and switch
git checkout -b feature-name
git switch -c feature-name  # New syntax

# Switch branch
git checkout main
git switch main  # New syntax

# Delete branch
git branch -d feature-name   # Safe
git branch -D feature-name   # Force

# Rename branch
git branch -m old-name new-name

# Show branches with last commit
git branch -v

# Show merged branches
git branch --merged

# Show unmerged branches
git branch --no-merged

# Track remote branch
git checkout --track origin/feature
```

---

## **5. Merging** 🔀

### **Merge Operations:**
```bash
# Merge branch into current
git merge feature-name

# Merge with merge commit (no ff)
git merge --no-ff feature-name

# Fast-forward only
git merge --ff-only feature-name

# Squash merge
git merge --squash feature-name

# Abort merge
git merge --abort

# Merge with strategy
git merge -X theirs feature-name
git merge -X ours feature-name

# Continue after conflict resolution
git merge --continue
```

### **Rebase:**
```bash
# Rebase onto branch
git rebase main

# Interactive rebase
git rebase -i HEAD~5
git rebase -i main

# Abort rebase
git rebase --abort

# Continue rebase
git rebase --continue

# Skip commit during rebase
git rebase --skip

# Rebase onto specific base
git rebase --onto new-base old-base
```

---

## **6. Remote Operations** 🌐

### **Remote Management:**
```bash
# List remotes
git remote
git remote -v

# Add remote
git remote add origin <url>

# Remove remote
git remote remove origin

# Rename remote
git remote rename origin new-name

# Change remote URL
git remote set-url origin <new-url>

# Show remote details
git remote show origin
```

### **Fetch & Pull:**
```bash
# Fetch from remote
git fetch origin

# Fetch all remotes
git fetch --all

# Fetch and prune
git fetch --prune

# Pull (fetch + merge)
git pull origin main

# Pull with rebase
git pull --rebase origin main

# Pull fast-forward only
git pull --ff-only
```

### **Push:**
```bash
# Push to remote
git push origin main

# Push and set upstream
git push -u origin feature

# Push all branches
git push --all

# Push tags
git push --tags

# Push specific tag
git push origin v1.0.0

# Delete remote branch
git push origin --delete feature

# Force push (dangerous!)
git push --force

# Safer force push
git push --force-with-lease
```

---

## **7. Stashing** 💾

### **Stash Operations:**
```bash
# Stash changes
git stash
git stash push

# Stash with message
git stash push -m "WIP: Feature work"

# Stash including untracked
git stash -u

# Stash including ignored
git stash -a

# List stashes
git stash list

# Show stash content
git stash show
git stash show -p

# Apply stash
git stash apply
git stash apply stash@{1}

# Apply and remove
git stash pop

# Remove stash
git stash drop stash@{0}

# Clear all stashes
git stash clear

# Create branch from stash
git stash branch feature-name
```

---

## **8. History & Logs** 📜

### **Viewing History:**
```bash
# Show commit history
git log

# One line per commit
git log --oneline

# Show graph
git log --graph

# Graph all branches
git log --graph --all --oneline --decorate

# Show last N commits
git log -n 5
git log -5

# Show patches (diffs)
git log -p

# Show stats
git log --stat

# By author
git log --author="John Doe"

# By date
git log --since="2 weeks ago"
git log --after="2024-01-01" --before="2024-12-31"

# By message
git log --grep="fix"

# By file
git log -- path/to/file

# Show commits adding/removing string
git log -S"function_name"
```

### **Showing Commits:**
```bash
# Show commit details
git show HEAD
git show abc123

# Show commit stats
git show --stat abc123

# Show file in commit
git show abc123:path/to/file
```

### **Reflog:**
```bash
# Show reflog
git reflog

# Show reflog for branch
git reflog show main

# With dates
git reflog --date=relative
```

---

## **9. Undoing Changes** ⏮️

### **Restore & Reset:**
```bash
# Discard changes in file
git restore file.txt
git checkout -- file.txt  # Legacy

# Unstage file
git restore --staged file.txt
git reset HEAD file.txt  # Legacy

# Discard all changes
git restore .

# Clean untracked files
git clean -fd
git clean -n  # Dry run
```

### **Reset:**
```bash
# Soft reset (keep changes staged)
git reset --soft HEAD~1

# Mixed reset (keep changes unstaged)
git reset HEAD~1
git reset --mixed HEAD~1

# Hard reset (discard changes)
git reset --hard HEAD~1

# Reset to specific commit
git reset --hard abc123

# Reset to remote
git reset --hard origin/main
```

### **Revert:**
```bash
# Create reverting commit
git revert HEAD
git revert abc123

# Revert multiple commits
git revert abc123..def456

# Revert merge commit
git revert -m 1 merge-sha

# Revert without committing
git revert -n HEAD
```

---

## **10. Advanced Commands** ⚡

### **Cherry-Pick:**
```bash
# Apply commit to current branch
git cherry-pick abc123

# Cherry-pick multiple
git cherry-pick abc123 def456

# Cherry-pick range
git cherry-pick abc123..def456

# Cherry-pick without committing
git cherry-pick -n abc123

# Abort cherry-pick
git cherry-pick --abort
```

### **Tags:**
```bash
# List tags
git tag

# Create lightweight tag
git tag v1.0.0

# Create annotated tag
git tag -a v1.0.0 -m "Version 1.0.0"

# Tag specific commit
git tag -a v1.0.0 abc123 -m "Version 1.0.0"

# Show tag
git show v1.0.0

# Delete tag
git tag -d v1.0.0

# Delete remote tag
git push origin --delete v1.0.0

# Push tag
git push origin v1.0.0

# Push all tags
git push --tags
```

### **Bisect:**
```bash
# Start bisect
git bisect start

# Mark bad commit
git bisect bad

# Mark good commit
git bisect good abc123

# Mark current as good/bad
git bisect good
git bisect bad

# Automated bisect
git bisect run npm test

# End bisect
git bisect reset
```

### **Submodules:**
```bash
# Add submodule
git submodule add <url> path/

# Initialize submodules
git submodule init

# Update submodules
git submodule update

# Update from remote
git submodule update --remote

# Clone with submodules
git clone --recursive <url>

# Update all submodules
git submodule update --init --recursive

# Run command in all submodules
git submodule foreach git pull origin main
```

---

## **11. Useful Aliases** 🎯

### **Productivity Aliases:**
```bash
# Add to ~/.gitconfig or run these commands:

# Status and branch
git config --global alias.st status
git config --global alias.br branch
git config --global alias.co checkout
git config --global alias.cm commit

# Pretty log
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.lol "log --graph --decorate --pretty=oneline --abbrev-commit"
git config --global alias.lola "log --graph --decorate --pretty=oneline --abbrev-commit --all"

# Useful shortcuts
git config --global alias.last "log -1 HEAD"
git config --global alias.unstage "reset HEAD --"
git config --global alias.uncommit "reset --soft HEAD~1"
git config --global alias.amend "commit --amend --no-edit"

# Branch management
git config --global alias.branches "branch -a"
git config --global alias.remotes "remote -v"
git config --global alias.contributors "shortlog -sn"

# Cleanup
git config --global alias.cleanup "!git branch --merged | grep -v '\\*\\|main\\|develop' | xargs -n 1 git branch -d"

# Show diff
git config --global alias.diffs "diff --staged"
```

### **Using Aliases:**
```bash
# After setting up aliases:
git st                    # git status
git lg                    # Pretty log
git co -b feature         # git checkout -b feature
git cm -m "Message"       # git commit -m "Message"
git unstage file.txt      # git reset HEAD file.txt
git uncommit              # git reset --soft HEAD~1
```

---

## **12. One-Liners** 🚀

### **Cleanup & Maintenance:**
```bash
# Delete all merged branches
git branch --merged | grep -v "\*\|main\|develop" | xargs -n 1 git branch -d

# Delete all local branches except main
git branch | grep -v "main" | xargs git branch -D

# Prune remote branches
git fetch --prune

# Garbage collection
git gc --aggressive --prune=now

# Find large files
git rev-list --objects --all | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' | sed -n 's/^blob //p' | sort --numeric-sort --key=2 | tail -20
```

### **Information:**
```bash
# Count commits
git rev-list --count HEAD

# Show authors ranked by commits
git shortlog -sn

# Show file change frequency
git log --all --format=format: --name-only | egrep -v '^$' | sort | uniq -c | sort -rg | head -20

# Show who changed file most
git log --follow -p -- path/to/file

# Repository size
git count-objects -vH

# List all files ever added
git log --pretty=format: --name-only --diff-filter=A | sort -u
```

### **Commit Analysis:**
```bash
# Show empty commits
git log --all --oneline --diff-filter=A --diff-filter=D | grep -v "^[a-f0-9]\{7\} [Mm]erge"

# Find commit by message
git log --all --grep="keyword"

# Find commit that added/removed line
git log -S"some code" -p

# Show commits between dates
git log --since="2024-01-01" --until="2024-01-31"

# Commits not on main
git log main..feature

# Show only merge commits
git log --merges

# Show no-merge commits
git log --no-merges
```

### **Diff & Changes:**
```bash
# Show files changed between commits
git diff --name-only commit1 commit2

# Show files changed between branches
git diff --name-status main feature

# Show changes by author
git log --author="John" -p

# Show total insertions/deletions
git log --shortstat --author="John" | grep -E "fil(e|es) changed" | awk '{files+=$1; inserted+=$4; deleted+=$6} END {print "files changed:", files, "lines inserted:", inserted, "lines deleted:", deleted}'
```

### **Branch Management:**
```bash
# Show branches containing commit
git branch --contains abc123

# Show branches not containing commit
git branch --no-contains abc123

# Sort branches by commit date
git branch --sort=-committerdate

# Track all remote branches
git branch -r | grep -v '\->' | while read remote; do git branch --track "${remote#origin/}" "$remote"; done
```

### **Workflow Shortcuts:**
```bash
# Undo last commit but keep changes
git reset --soft HEAD~1

# Change last commit message
git commit --amend -m "New message"

# Add file to last commit
git add forgotten.txt && git commit --amend --no-edit

# Sync fork with upstream
git fetch upstream && git rebase upstream/main && git push origin main --force-with-lease

# Create branch from stash
git stash branch feature-branch stash@{0}
```

---

## **Emergency Commands** 🚨

```bash
# RECOVER
git reflog                              # Find lost commits
git checkout -b recovered <sha>         # Recover branch

# UNDO
git reset --soft HEAD~1                 # Undo commit, keep changes
git reset --hard HEAD~1                 # Undo commit, discard changes
git revert <commit>                     # Safe undo (new commit)

# ABORT
git merge --abort                       # Abort merge
git rebase --abort                      # Abort rebase  
git cherry-pick --abort                 # Abort cherry-pick

# CLEAN
git clean -fd                           # Remove untracked files
git reset --hard origin/main            # Match remote exactly

# FORCE (Caution!)
git push --force-with-lease             # Safer force push
git reset --hard <commit>               # Force reset
```

---

## **Git Flow Commands** 🌊

```bash
# Initialize git-flow
git flow init

# Features
git flow feature start feature-name
git flow feature finish feature-name
git flow feature publish feature-name

# Releases
git flow release start 1.0.0
git flow release finish 1.0.0

# Hotfixes
git flow hotfix start hotfix-name
git flow hotfix finish hotfix-name
```

---

## **Keyboard Shortcuts (Terminal)** ⌨️

```bash
# Bash/Zsh shortcuts
Ctrl+R          # Search command history
Ctrl+A          # Move to line start
Ctrl+E          # Move to line end
Ctrl+U          # Clear line before cursor
Ctrl+K          # Clear line after cursor
Ctrl+W          # Delete word before cursor
Alt+.           # Insert last argument from previous command

# Git interactive shortcuts (rebase, add -p)
y               # Yes
n               # No
q               # Quit
s               # Split hunk
e               # Edit hunk
?               # Help
```

---

## **Configuration Templates** 📋

### **Global .gitconfig:**
```ini
[user]
    name = Your Name
    email = your.email@example.com
    signingkey = YOUR_GPG_KEY

[core]
    editor = vim
    autocrlf = input
    excludesfile = ~/.gitignore_global

[init]
    defaultBranch = main

[pull]
    rebase = false

[push]
    default = current
    followTags = true

[merge]
    conflictstyle = diff3

[diff]
    colorMoved = zebra

[color]
    ui = auto

[alias]
    st = status
    co = checkout
    br = branch
    cm = commit
    lg = log --oneline --graph --all --decorate
    unstage = reset HEAD --
    last = log -1 HEAD
    
[commit]
    gpgsign = true
```

### **.gitignore Template:**
```bash
# Dependencies
node_modules/
vendor/
venv/

# Build outputs
dist/
build/
*.o
*.class
*.exe

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db
desktop.ini

# Logs
*.log
logs/
npm-debug.log*

# Testing
coverage/
.nyc_output/

# Temporary
tmp/
temp/
*.tmp
```

---

## **Quick Reference Tables** 📊

### **Reset Modes:**
| Command | HEAD | Index | Working Dir |
|---------|------|-------|-------------|
| `--soft` | ✅ | ❌ | ❌ |
| `--mixed` | ✅ | ✅ | ❌ |
| `--hard` | ✅ | ✅ | ✅ |

### **Merge Strategies:**
| Strategy | Description |
|----------|-------------|
| `--ff` | Fast-forward if possible (default) |
| `--no-ff` | Always create merge commit |
| `--ff-only` | Only fast-forward, fail otherwise |
| `--squash` | Squash commits into one |
| `-X ours` | Prefer current branch in conflicts |
| `-X theirs` | Prefer merging branch in conflicts |

### **Diff Types:**
| Command | Compares |
|---------|----------|
| `git diff` | Working dir vs index |
| `git diff --staged` | Index vs HEAD |
| `git diff HEAD` | Working dir vs HEAD |
| `git diff branch1 branch2` | Two branches |
| `git diff commit1 commit2` | Two commits |

---

## **Next Steps** 📚

Dive deeper into specific topics:

- **[Git Fundamentals](Git_Fundamentals.md)** - Architecture and internals
- **[Git Basic Commands](Git_Basic_Commands.md)** - Detailed command explanations
- **[Git Advanced Commands](Git_Advanced_Commands.md)** - Power user techniques
- **[Git Workflows](Git_Workflows.md)** - Professional development strategies
- **[Git Troubleshooting](Git_Troubleshooting.md)** - Common issues and solutions

---

**📋 Bookmark this cheatsheet for quick reference during daily work!**
