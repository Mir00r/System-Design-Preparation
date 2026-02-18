# **Git Troubleshooting – Common Issues & Solutions** 🔧🚨

Master troubleshooting techniques for common Git problems, error messages, and recovery strategies every DevOps engineer encounters in daily work.

---

## **Table of Contents** 📑
1. [Undoing Changes](#1-undoing-changes)
2. [Merge Conflicts](#2-merge-conflicts)
3. [Branch Issues](#3-branch-issues)
4. [Remote Repository Problems](#4-remote-repository-problems)
5. [Commit Mistakes](#5-commit-mistakes)
6. [Authentication Issues](#6-authentication-issues)
7. [Performance Problems](#7-performance-problems)
8. [Lost Work Recovery](#8-lost-work-recovery)
9. [Repository Corruption](#9-repository-corruption)
10. [Common Error Messages](#10-common-error-messages)
11. [Prevention Strategies](#11-prevention-strategies)
12. [Interview Cheat Sheet](#12-interview-cheat-sheet)

---

## **1. Undoing Changes** ⏮️

### **Scenario: Discard Uncommitted Changes**

```bash
# Problem: Made changes, want to discard them
git status
# modified: file.txt

# Solution 1: Discard changes in specific file
git restore file.txt
git checkout -- file.txt  # Legacy

# Solution 2: Discard all changes
git restore .
git checkout -- .  # Legacy

# Solution 3: Discard and remove untracked files
git clean -fd
# -f = force, -d = directories

# Check what would be removed
git clean -n
git clean --dry-run
```

### **Scenario: Unstage Files**

```bash
# Problem: Accidentally staged wrong files
git add .
git status
# All files staged, but you wanted only specific ones

# Solution: Unstage specific file
git restore --staged file.txt
git reset HEAD file.txt  # Legacy

# Unstage all files
git restore --staged .
git reset HEAD  # Legacy
```

### **Scenario: Undo Last Commit (Keep Changes)**

```bash
# Problem: Committed too early, want to add more changes
git commit -m "Incomplete work"
# Oh no! Forgot to add file.txt

# Solution: Undo commit, keep changes staged
git reset --soft HEAD~1

# Add forgotten file
git add file.txt

# Commit again
git commit -m "Complete work"
```

### **Scenario: Undo Last Commit (Discard Changes)**

```bash
# Problem: Committed wrong files entirely
git commit -m "Wrong commit"

# Solution 1: Undo and keep changes
git reset --mixed HEAD~1  # Default
# or
git reset HEAD~1

# Solution 2: Undo and discard everything
git reset --hard HEAD~1

# ⚠️ WARNING: --hard is destructive!
```

### **Scenario: Revert Pushed Commit**

```bash
# Problem: Pushed bad commit, need to undo
git push origin main
# Commit abc123 is bad

# Solution: Create reverting commit (safe for shared branches)
git revert abc123
git push origin main

# Revert multiple commits
git revert abc123..def456

# Revert merge commit
git revert -m 1 merge-commit-sha
```

---

## **2. Merge Conflicts** ⚔️

### **Scenario: Basic Merge Conflict**

```bash
# Problem: Merge conflict during merge
git merge feature
# CONFLICT (content): Merge conflict in app.js
# Automatic merge failed; fix conflicts and then commit the result.

# Solution:
# 1. Check conflicted files
git status
# Unmerged paths:
#   both modified:   app.js

# 2. Open file and resolve
vim app.js
# Look for conflict markers:
<<<<<<< HEAD
current branch code
=======
incoming branch code
>>>>>>> feature

# 3. Edit to keep desired code, remove markers
# Final result:
# combined or chosen code

# 4. Stage resolved file
git add app.js

# 5. Complete merge
git commit
# Or
git merge --continue

# To abort merge:
git merge --abort
```

### **Scenario: Rebase Conflicts**

```bash
# Problem: Conflicts during rebase
git rebase main
# CONFLICT (content): Merge conflict in file.js
# Resolve all conflicts manually, mark them as resolved with
# "git add/rm <conflicted_files>", then run "git rebase --continue".

# Solution:
# 1. Resolve conflict in file
vim file.js
# Remove conflict markers, fix code

# 2. Stage resolved file
git add file.js

# 3. Continue rebase
git rebase --continue

# If more conflicts, repeat steps 1-3

# To abort rebase:
git rebase --abort

# To skip this commit:
git rebase --skip
```

### **Scenario: Accept All from One Side**

```bash
# Problem: Many conflicts, want to accept all from one side

# Solution 1: Accept all from incoming branch (theirs)
git checkout --theirs file.txt
git add file.txt

# Solution 2: Accept all from current branch (ours)
git checkout --ours file.txt
git add file.txt

# For all files
git merge -X theirs feature
git merge -X ours feature
```

### **Scenario: Binary File Conflicts**

```bash
# Problem: Binary files (images, PDFs) in conflict
# CONFLICT (content): Merge conflict in image.png

# Solution: Choose one version
git checkout --ours image.png
# or
git checkout --theirs image.png

# Stage and commit
git add image.png
git commit
```

---

## **3. Branch Issues** 🌿

### **Scenario: Committed to Wrong Branch**

```bash
# Problem: Working on main, should be on feature branch
git checkout main
# Made commits
git log --oneline
# abc123 Oops, wrong branch

# Solution 1: Move commits to new branch
git branch feature-branch  # Create branch at current commit
git reset --hard HEAD~1    # Remove commit from main
git checkout feature-branch

# Solution 2: Move last N commits
git branch feature-branch
git reset --hard HEAD~3   # Move back 3 commits
git checkout feature-branch
```

### **Scenario: Can't Switch Branches (Uncommitted Changes)**

```bash
# Problem: Trying to switch with uncommitted changes
git checkout feature
# error: Your local changes to the following files would be overwritten by checkout:
#        file.txt

# Solution 1: Stash changes
git stash push -m "WIP on main"
git checkout feature
# Work on feature
git checkout main
git stash pop

# Solution 2: Commit changes
git commit -am "WIP: Work in progress"
git checkout feature

# Solution 3: Force checkout (loses changes!)
git checkout -f feature
```

### **Scenario: Branch Deleted Accidentally**

```bash
# Problem: Accidentally deleted branch
git branch -d important-feature
# Oh no!

# Solution: Find branch in reflog
git reflog
# abc123 HEAD@{5}: commit: Last commit on important-feature

# Recreate branch
git checkout -b important-feature abc123
# or
git branch important-feature abc123
```

### **Scenario: Diverged Branches**

```bash
# Problem: Local and remote branches diverged
git push origin main
# ! [rejected]        main -> main (non-fast-forward)

# Solution 1: Rebase local commits
git fetch origin
git rebase origin/main
git push origin main

# Solution 2: Merge remote changes
git pull origin main
git push origin main

# Solution 3: Force push (dangerous!)
git push --force origin main
# Better: Use force-with-lease
git push --force-with-lease origin main
```

---

## **4. Remote Repository Problems** 🌐

### **Scenario: Can't Push - Permission Denied**

```bash
# Problem
git push origin main
# ERROR: Permission to user/repo.git denied

# Solution 1: Check remote URL
git remote -v
# If HTTPS, switch to SSH
git remote set-url origin git@github.com:user/repo.git

# Solution 2: Check SSH key
ssh -T git@github.com
# If fails, add SSH key to GitHub

# Solution 3: Verify credentials
git config user.name
git config user.email
```

### **Scenario: Can't Pull - Overwrite Local Changes**

```bash
# Problem
git pull origin main
# error: Your local changes would be overwritten by merge.

# Solution 1: Stash, pull, apply
git stash
git pull origin main
git stash pop

# Solution 2: Commit first
git commit -am "Local changes"
git pull origin main

# Solution 3: Discard local changes
git reset --hard origin/main
```

### **Scenario: Remote Rejected Non-Fast-Forward**

```bash
# Problem
git push origin main
# ! [rejected] main -> main (non-fast-forward)

# Solution 1: Pull first (merge)
git pull origin main
git push origin main

# Solution 2: Pull with rebase
git pull --rebase origin main
git push origin main

# Solution 3: Force push (if you're sure!)
git push --force-with-lease origin main
```

### **Scenario: Large File Rejected**

```bash
# Problem
git push origin main
# remote: error: File large-file.zip exceeds GitHub's file size limit of 100 MB

# Solution 1: Remove file from history
git rm --cached large-file.zip
git commit --amend --no-edit
git push origin main

# Solution 2: Use Git LFS
git lfs install
git lfs track "*.zip"
git add .gitattributes
git add large-file.zip
git commit -m "Add large file with LFS"
git push origin main

# Solution 3: Remove from all history (BFG Repo-Cleaner)
brew install bfg
bfg --delete-files large-file.zip
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force origin main
```

---

## **5. Commit Mistakes** 💥

### **Scenario: Wrong Commit Message**

```bash
# Problem: Typo in last commit message
git commit -m "Fix bug in authentiction"  # Typo!

# Solution: Amend commit message
git commit --amend -m "fix: Resolve bug in authentication"

# If already pushed
git push --force-with-lease origin feature
```

### **Scenario: Forgot to Add File**

```bash
# Problem: Committed without adding file
git commit -m "Add feature"
# Oops, forgot config.js

# Solution: Add file and amend commit
git add config.js
git commit --amend --no-edit

# Or with new message
git commit --amend -m "Add feature with config"
```

### **Scenario: Committed Sensitive Data**

```bash
# Problem: Committed .env file with secrets!
git commit -am "Update config"
git log
# abc123 Update config (contains .env)

# Solution 1: If not pushed yet
git reset --soft HEAD~1
git restore --staged .env
echo ".env" >> .gitignore
git add .gitignore
git commit -m "Add .gitignore"

# Solution 2: If already pushed
# Use BFG Repo-Cleaner
bfg --delete-files .env
git reflog expire --expire=now --all
git gc --prune=now --aggressive
git push --force origin main

# IMPORTANT: Rotate all exposed credentials!
```

### **Scenario: Too Many Commits (Need to Squash)**

```bash
# Problem: 10 small commits, want 1
git log --oneline
# abc123 Fix typo
# def456 Fix another typo
# ghi789 Actually fix it
# ...

# Solution: Interactive rebase
git rebase -i HEAD~10

# In editor, change 'pick' to 'squash' or 'fixup':
pick abc123 First commit
squash def456 Fix typo
squash ghi789 Fix another typo
squash ...

# Save and close
# Edit combined commit message if needed
# Force push if already pushed
git push --force-with-lease origin feature
```

---

## **6. Authentication Issues** 🔐

### **Scenario: GitHub Authentication Failed (HTTPS)**

```bash
# Problem
git push origin main
# Username for 'https://github.com': user
# Password: 
# remote: Support for password authentication was removed on August 13, 2021.

# Solution: Use Personal Access Token (PAT)
# 1. Generate PAT on GitHub
#    Settings → Developer settings → Personal access tokens

# 2. Use PAT as password
git push origin main
# Username: your-username
# Password: <paste-PAT>

# 3. Cache credentials
git config --global credential.helper cache
git config --global credential.helper store  # Saves permanently
```

### **Scenario: SSH Key Issues**

```bash
# Problem
git push origin main
# git@github.com: Permission denied (publickey).

# Solution 1: Check SSH key exists
ls -la ~/.ssh
# Should see id_rsa and id_rsa.pub (or id_ed25519)

# Solution 2: Generate SSH key if missing
ssh-keygen -t ed25519 -C "your_email@example.com"
# Press Enter to accept defaults

# Solution 3: Add SSH key to agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Solution 4: Add public key to GitHub
cat ~/.ssh/id_ed25519.pub
# Copy output, add to GitHub Settings → SSH Keys

# Solution 5: Test SSH connection
ssh -T git@github.com
# Should see: "Hi username! You've successfully authenticated"
```

### **Scenario: Multiple GitHub Accounts**

```bash
# Problem: Using personal and work GitHub accounts
# Work commits showing personal name

# Solution: Configure per repository
cd work-repo
git config user.name "Work Name"
git config user.email "work@company.com"

# For SSH with multiple accounts
# Create ~/.ssh/config:
# Personal
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

# Work  
Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work

# Clone using specific host
git clone git@github-work:company/repo.git
git clone git@github-personal:user/repo.git
```

---

## **7. Performance Problems** 🐌

### **Scenario: Git Operations Very Slow**

```bash
# Problem: Git commands taking forever

# Solution 1: Garbage collection
git gc
git gc --aggressive

# Solution 2: Prune unreachable objects
git prune

# Solution 3: Repack repository
git repack -ad

# Solution 4: Clean up
git reflog expire --expire=30.days --all
git gc --prune=now

# Solution 5: Check repository size
git count-objects -vH
```

### **Scenario: Large Repository**

```bash
# Problem: Repository is huge (>1GB)

# Solution 1: Identify large files
git rev-list --objects --all |
  git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' |
  sed -n 's/^blob //p' |
  sort --numeric-sort --key=2 |
  tail -20

# Solution 2: Use shallow clone
git clone --depth 1 https://github.com/user/repo.git

# Solution 3: Sparse checkout
git clone --no-checkout https://github.com/user/repo.git
cd repo
git sparse-checkout init --cone
git sparse-checkout set src/ docs/
git checkout main

# Solution 4: Use Git LFS for large files
git lfs install
git lfs track "*.psd"
git lfs track "*.mp4"
```

### **Scenario: Too Many Branches**

```bash
# Problem: Hundreds of stale branches

# Solution 1: Delete merged local branches
git branch --merged | grep -v "\*\|main\|develop" | xargs -n 1 git branch -d

# Solution 2: Prune remote tracking branches
git fetch --prune
git remote prune origin

# Solution 3: Delete remote branches (careful!)
# List merged remote branches
git branch -r --merged origin/main | grep -v "main\|develop" | sed 's/origin\///'

# Delete specific remote branch
git push origin --delete old-branch
```

---

## **8. Lost Work Recovery** 🔍

### **Scenario: Accidentally Deleted Commits**

```bash
# Problem: Hard reset deleted commits
git reset --hard HEAD~3
# Oh no! Lost 3 commits

# Solution: Use reflog
git reflog
# Find lost commit SHA
# abc123 HEAD@{5}: commit: Important work

# Recover
git reset --hard abc123
# or create branch
git checkout -b recovered abc123
```

### **Scenario: Lost Stash**

```bash
# Problem: Accidentally deleted stash
git stash drop stash@{0}
# Oh no! Needed that stash

# Solution: Find in reflog
git fsck --unreachable | grep commit | cut -d ' ' -f3 | xargs git log --merges --no-walk --grep=WIP

# Or manually search reflog
git log --graph --oneline --decorate $(git fsck --no-reflog | awk '/dangling commit/ {print $3}')

# Apply recovered stash
git stash apply <commit-hash>
```

### **Scenario: Deleted Branch with Unique Commits**

```bash
# Problem: Deleted branch, commits nowhere else
git branch -D feature-branch
# Commits were only on that branch!

# Solution: Use reflog
git reflog | grep feature-branch
# abc123 checkout: moving from feature-branch to main

# Recreate branch
git checkout -b feature-branch abc123
```

### **Scenario: Lost Changes After Hard Reset**

```bash
# Problem: Did hard reset, lost uncommitted work
git reset --hard HEAD
# Lost all working directory changes

# Solution: Check if any reflog entries
git reflog
# Unlikely to help for uncommitted changes

# Prevention: Always stash before dangerous operations
git stash push -m "Before risky operation"
# Do risky operation
# Recover if needed:
git stash pop
```

---

## **9. Repository Corruption** 💔

### **Scenario: Repository Corruption**

```bash
# Problem
git status
# error: object file .git/objects/xx/xxxxx is empty
# fatal: loose object is corrupt

# Solution 1: Verify repository
git fsck --full

# Solution 2: Try to recover
rm .git/objects/xx/xxxxx
git fetch origin
git reset --hard origin/main

# Solution 3: Clone fresh copy
cd ..
git clone https://github.com/user/repo.git repo-new
cd repo-new
# Copy over local changes if any
```

### **Scenario: Corrupted Index**

```bash
# Problem
git status
# fatal: index file corrupt

# Solution: Remove and reset index
rm -f .git/index
git reset

# Verify
git status
```

### **Scenario: Corrupted Pack Files**

```bash
# Problem
git status
# error: bad pack header

# Solution: Rebuild packs
git repack -Ad
git prune-packed
```

---

## **10. Common Error Messages** 📋

### **"fatal: refusing to merge unrelated histories"**

```bash
# Problem: Trying to merge unrelated repositories
git pull origin main
# fatal: refusing to merge unrelated histories

# Solution: Allow unrelated histories
git pull origin main --allow-unrelated-histories
```

### **"fatal: not a git repository"**

```bash
# Problem: Not in git repository
git status
# fatal: not a git repository (or any of the parent directories): .git

# Solution 1: Initialize repository
git init

# Solution 2: Clone repository
git clone <url>

# Solution 3: Navigate to correct directory
cd /path/to/repo
```

### **"fatal: You have not concluded your merge"**

```bash
# Problem: Merge in progress
git merge feature
# CONFLICT...
# (forget to complete merge)
git checkout other-branch
# error: You have not concluded your merge (MERGE_HEAD exists).

# Solution 1: Complete merge
git merge --continue

# Solution 2: Abort merge
git merge --abort
```

### **"error: failed to push some refs"**

```bash
# Problem: Remote has changes you don't have
git push origin main
# error: failed to push some refs

# Solution: Pull first
git pull origin main
git push origin main

# Or with rebase
git pull --rebase origin main
git push origin main
```

### **"fatal: 'origin' does not appear to be a git repository"**

```bash
# Problem: Remote not configured
git push origin main
# fatal: 'origin' does not appear to be a git repository

# Solution: Add remote
git remote add origin https://github.com/user/repo.git
git push -u origin main
```

---

## **11. Prevention Strategies** 🛡️

### **Use Aliases for Safety:**

```bash
# Safe aliases
git config --global alias.undo-commit "reset --soft HEAD~1"
git config --global alias.unstage "reset HEAD --"
git config --global alias.discard "checkout --"
git config --global alias.amend "commit --amend --no-edit"

# Dangerous operations with confirmation
git config --global alias.nuke "!git reset --hard HEAD && git clean -fd"
```

### **Git Hooks for Protection:**

```bash
# .git/hooks/pre-commit
#!/bin/bash

# Prevent commits to main
branch=$(git rev-parse --abbrev-ref HEAD)
if [[ "$branch" == "main" || "$branch" == "master" ]]; then
    echo "Direct commits to $branch are not allowed!"
    exit 1
fi

# Check for secrets
if git diff --cached | grep -E 'API_KEY|SECRET|PASSWORD'; then
    echo "Possible secret detected! Aborting commit."
    exit 1
fi

# Make executable
chmod +x .git/hooks/pre-commit
```

### **Regular Backups:**

```bash
# Backup script
#!/bin/bash
# backup-git.sh

REPO_PATH="/path/to/repo"
BACKUP_PATH="/backups/$(date +%Y%m%d)"

# Create backup
mkdir -p "$BACKUP_PATH"
cd "$REPO_PATH"
git bundle create "$BACKUP_PATH/repo.bundle" --all

# Verify bundle
git bundle verify "$BACKUP_PATH/repo.bundle"

# To restore:
# git clone backup.bundle recovered-repo
```

### **Best Practices Checklist:**

```bash
✅ Always pull before pushing
✅ Commit frequently, push regularly
✅ Use feature branches, not main
✅ Review changes before committing
✅ Use .gitignore from start
✅ Never force push to shared branches
✅ Test before committing
✅ Write meaningful commit messages
✅ Use branch protection rules
✅ Enable 2FA on Git hosting
```

---

## **12. Interview Cheat Sheet** 🎯

### **Common Questions:**

**Q1: How to undo last commit?**
```bash
# Keep changes
git reset --soft HEAD~1

# Discard changes
git reset --hard HEAD~1

# Pushed to remote (create reverting commit)
git revert HEAD
```

**Q2: Resolve merge conflict?**
```bash
1. git merge branch        # Attempt merge
2. git status             # See conflicts
3. Edit files             # Resolve conflicts
4. git add resolved-files # Stage
5. git commit             # Complete merge
```

**Q3: Recover deleted branch?**
```bash
git reflog
git checkout -b recovered-branch <commit-hash>
```

**Q4: Committed to wrong branch?**
```bash
git branch correct-branch  # Create branch
git reset --hard HEAD~1    # Remove from current
git checkout correct-branch
```

**Q5: Remove file from Git history?**
```bash
# For recent commits
git filter-branch --tree-filter 'rm -f file' HEAD

# Better: BFG Repo-Cleaner
bfg --delete-files file
git reflog expire --expire=now --all
git gc --prune=now --aggressive
```

**Q6: Fix detached HEAD state?**
```bash
# Create branch at current commit
git checkout -b new-branch

# Or switch back to branch
git checkout main
```

**Q7: Committed sensitive data?**
```bash
# Not pushed: Reset
git reset --soft HEAD~1
git restore --staged .env

# Pushed: Use BFG
bfg --delete-files .env
# ROTATE ALL CREDENTIALS!
```

**Q8: Pull rejected (uncommitted changes)?**
```bash
# Stash, pull, pop
git stash
git pull
git stash pop
```

**Q9: Can't push (non-fast-forward)?**
```bash
# Pull with rebase
git pull --rebase origin main
git push origin main

# Or merge
git pull origin main
git push origin main
```

**Q10: Large file error?**
```bash
# Use Git LFS
git lfs install
git lfs track "*.zip"
git add .gitattributes large-file.zip
git commit -m "Add large file with LFS"
git push origin main
```

---

## **Quick Reference: Emergency Commands** 🚨

```bash
# RECOVER
git reflog                          # Find lost commits
git fsck --lost-found              # Find dangling commits
git checkout -b recovered <sha>    # Recover branch

# UNDO
git reset --soft HEAD~1            # Undo commit, keep changes
git reset --hard HEAD~1            # Undo commit, discard changes
git revert <commit>                # Safe undo (creates new commit)

# ABORT
git merge --abort                  # Abort merge
git rebase --abort                 # Abort rebase
git cherry-pick --abort            # Abort cherry-pick

# CLEAN
git clean -fd                      # Remove untracked files
git reset --hard origin/main       # Match remote exactly
git stash clear                    # Delete all stashes

# FORCE (Use with caution!)
git push --force-with-lease        # Safer force push
git reset --hard <commit>          # Force reset (destructive)
```

---

## **Next Steps** 📚

Complete your Git knowledge:

- **[Git Commands Cheatsheet](Git_Commands_Cheatsheet.md)** - Quick reference for all commands
- **[Git Fundamentals](Git_Fundamentals.md)** - Deep dive into Git architecture
- **[Git Workflows](Git_Workflows.md)** - Master professional workflows

---

**🔧 Master troubleshooting to handle any Git situation with confidence!**
