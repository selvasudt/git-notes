# 11 · Git Troubleshooting & Disaster Recovery

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [03 · Git File Lifecycle](./03_Git_File_Lifecycle.md), [06 · Advanced Git](./06_Advanced_Git.md)

---

## Table of Contents

1. [What is Git Troubleshooting?](#1-what-is-git-troubleshooting)
2. [The Diagnostic Toolkit](#2-the-diagnostic-toolkit)
3. [Category 1 — Working Directory Problems](#3-category-1--working-directory-problems)
4. [Category 2 — Staging & Commit Problems](#4-category-2--staging--commit-problems)
5. [Category 3 — Branch Problems](#5-category-3--branch-problems)
6. [Category 4 — Merge & Rebase Problems](#6-category-4--merge--rebase-problems)
7. [Category 5 — Remote & Push Problems](#7-category-5--remote--push-problems)
8. [Category 6 — History & Data Loss](#8-category-6--history--data-loss)
9. [Category 7 — Repository Corruption](#9-category-7--repository-corruption)
10. [Category 8 — Performance Problems](#10-category-8--performance-problems)
11. [Category 9 — Authentication Problems](#11-category-9--authentication-problems)
12. [Disaster Recovery Playbook](#12-disaster-recovery-playbook)
13. [Interview Questions & Model Answers](#13-interview-questions--model-answers)
14. [Master Troubleshooting Cheatsheet](#14-master-troubleshooting-cheatsheet)

---

## 1. What is Git Troubleshooting?

## What is it?

## Why It Matters
Fast, systematic troubleshooting separates senior engineers from juniors. A 2-minute recovery vs a 2-hour panic is purely a knowledge gap.

## Internal Working
Git almost never destroys committed data. Every committed object stays in `.git/objects/` until garbage collected. The reflog records all HEAD movements for 90 days. These two facts underpin all recovery.

## Command Explanation

### Syntax
```bash
git reflog                  # FIRST tool for any data loss
git fsck --full             # check repo integrity
git status                  # always diagnose before acting
```

### The Core Principle

> **Git almost never destroys data permanently.** If it was committed, it is almost always recoverable. The reflog is your first tool; `.git/objects/` is the last resort.

### The Troubleshooting Mindset

```
STOP → DIAGNOSE → UNDERSTAND → ACT → VERIFY

1. STOP: Don't keep running commands hoping something works.
2. DIAGNOSE: git status, git log, git reflog — understand the current state.
3. UNDERSTAND: Know WHY the problem occurred before fixing it.
4. ACT: Apply the minimal fix.
5. VERIFY: Confirm the fix worked and nothing else broke.
```

---

## 2. The Diagnostic Toolkit

## What is it?
The set of Git commands to run FIRST before attempting any fix — establishing the exact current state.

## Why It Matters
Running fix commands without diagnosing first is how small problems become catastrophic. Stop → Diagnose → Understand → Act → Verify.

## Internal Working
`git status` compares working dir vs index vs last commit. `git reflog` reads `.git/logs/HEAD`. `git fsck` walks the entire object graph checking SHA validity and reference integrity.

## Command Explanation

### Syntax
```bash
# ─── ALWAYS START HERE ────────────────────────────────────
git status                          # current state of working dir + index
git log --oneline --graph --all -20 # recent commit history + branches
git reflog --all -20                # all recent HEAD + branch movements
git remote -v                       # remote configuration
git branch -vv                      # branches + tracking + ahead/behind

# ─── DEEPER DIAGNOSIS ─────────────────────────────────────
git diff                            # working dir vs index (unstaged)
git diff --staged                   # index vs last commit (staged)
git diff HEAD                       # working dir vs last commit
git stash list                      # any hidden stashes?
git ls-files --stage                # raw index contents
cat .git/HEAD                       # what HEAD points to
cat .git/refs/heads/main            # what main points to

# ─── OBJECT-LEVEL DIAGNOSIS ───────────────────────────────
git cat-file -t <sha>               # type of object
git cat-file -p <sha>               # content of object
git fsck --full                     # repository integrity check
git count-objects -vH               # object counts + sizes
```

---

## 3. Category 1 — Working Directory Problems

## What is it?
Problems where files in the Working Directory are accidentally deleted, overwritten, or polluted.

## Why It Matters
Working directory problems are the most common and most panicked — but almost always trivially fixed if the file was tracked.

## Internal Working
Tracked files can always be restored from the index (staging area) or from any commit blob. Untracked files deleted with `git clean -f` are NOT in the object store and CANNOT be recovered via Git.

## Command Explanation

### Syntax
```bash
git restore <file>                    # restore from index/HEAD
git restore --source <sha> <file>     # restore from specific commit
git clean -n                          # dry run FIRST
git clean -fd                         # remove untracked files
```

### Problem: Accidentally Deleted a Tracked File

```bash
# Symptom
git status
# deleted: src/Payment.java

# Fix: restore from last commit / staging area
git restore src/Payment.java
# OR (older syntax)
git checkout -- src/Payment.java

# Verify
ls src/Payment.java   # file is back
```

### Problem: Accidentally Overwrote a File with Wrong Content

```bash
# Symptom: file exists but has wrong content
git diff src/Payment.java      # shows your changes vs what Git has

# Fix Option 1: restore from last commit
git restore src/Payment.java

# Fix Option 2: restore from specific commit
git restore --source HEAD~3 src/Payment.java

# Fix Option 3: restore from a tag
git restore --source v2.0.0 src/Payment.java
```

### Problem: Untracked File Created by Mistake

```bash
# Remove a single untracked file
git clean -f src/AccidentalFile.java

# Remove untracked directory
git clean -fd src/accidental-dir/

# Dry run first (shows what WOULD be removed)
git clean -n
git clean -nd

# Remove everything untracked including ignored files
git clean -fdx     # ⚠️ DANGEROUS — removes .env, target/, node_modules/

# Interactive (confirms each file)
git clean -i
```

### Problem: Working Directory Has Too Many Unrelated Changes

```bash
# Symptom: started two features, now working dir is a mess

# Fix: stash selectively
git stash push -m "WIP: payment retry" src/PaymentService.java

# OR: use patch mode to stage and commit one thing at a time
git add -p               # stage only relevant hunks
git commit -m "feat: payment retry"
git add -p               # stage next set of hunks
git commit -m "feat: auth token refresh"
```

---

## 4. Category 2 — Staging & Commit Problems

## What is it?
Problems where wrong files are staged, committed with wrong messages, committed to wrong branches, or wrong content is committed.

## Why It Matters
Solving these requires knowing the exact right command — using the wrong one (e.g., `git reset --hard` instead of `--soft`) escalates the problem.

## Internal Working
Staging problems are fixed by manipulating `.git/index`. Commit problems are fixed by moving the branch pointer (reset) or creating a new undo commit (revert). ORIG_HEAD is set before any reset/merge/rebase as a safety reference.

## Command Explanation

### Syntax
```bash
git restore --staged <file>     # unstage
git reset --soft HEAD~1         # undo commit, keep staged
git commit --amend              # fix last commit
git reset --hard ORIG_HEAD      # undo last merge/rebase
```

### Problem: Staged the Wrong File

```bash
# Symptom
git status
# Changes to be committed:
#   new file: secrets.env     ← should NOT be here

# Fix: unstage (keep the file on disk)
git restore --staged secrets.env
# OR
git reset HEAD secrets.env

# Verify
git status
# Untracked files:
#   secrets.env    ← now untracked, not staged
```

### Problem: Committed the Wrong File

```bash
# Committed secrets.env in the LAST commit (not yet pushed)

# Fix Option 1: amend commit (remove file)
git rm --cached secrets.env
git commit --amend --no-edit

# Fix Option 2: reset and redo
git reset --soft HEAD~1     # uncommit, keep staged
git restore --staged secrets.env   # unstage secret file
git commit -m "original message"   # recommit without secret

# Add to .gitignore to prevent recurrence
echo "secrets.env" >> .gitignore
git add .gitignore
git commit --amend --no-edit
```

### Problem: Bad Commit Message

```bash
# Last commit not yet pushed
git commit --amend -m "feat(payment): add retry logic with exponential backoff"

# Older commit not yet pushed (interactive rebase)
git rebase -i HEAD~3
# Change 'pick' to 'reword' for the target commit
# Save → editor opens → change message → save

# Already pushed to shared branch → DON'T rewrite
# Instead: add a follow-up commit with a note, or just live with it
```

### Problem: Committed on Wrong Branch

```bash
# Made commits on main instead of feature branch
git log --oneline -3
# abc123 feat: add payment retry    ← should be on feature branch
# def456 feat: add payment config   ← should be on feature branch
# xyz789 previous correct commit

# Fix:
# 1. Create feature branch at current position
git switch -c feature/payment-retry

# 2. Return main to correct state
git switch main
git reset --hard xyz789    # main now at correct commit

# 3. Feature branch still has abc123 and def456 ✓
git switch feature/payment-retry
git log --oneline -3
# abc123 feat: add payment retry    ✓
# def456 feat: add payment config   ✓
# xyz789 base commit                ✓
```

### Problem: Forgot to Include a File in Last Commit

```bash
# Fix: amend the last commit (not yet pushed)
git add src/ForgottenFile.java
git commit --amend --no-edit    # adds file to last commit, keeps message

# If already pushed:
git add src/ForgottenFile.java
git commit -m "chore: add missing file from previous commit"
git push
# (Don't amend pushed commits — creates history conflict for others)
```

---

## 5. Category 3 — Branch Problems

## What is it?
Problems with branch state: accidentally deleted branches, detached HEAD, broken tracking relationships, diverged local/remote.

## Why It Matters
Branch problems can appear catastrophic (unmerged work "gone") but are almost always recoverable via reflog.

## Internal Working
A branch is a file in `.git/refs/heads/`. Deleting it removes the file but commits remain in `.git/objects/` until GC. The reflog logs the branch's last commit SHA — use it to recreate the branch.

## Command Explanation

### Syntax
```bash
git reflog --all | grep "feature/name"   # find lost branch SHA
git switch -c feature/name <sha>         # recreate branch
git switch -                             # escape detached HEAD
```

### Problem: Accidentally Deleted a Branch

```bash
# Symptom: deleted feature branch that had unmerged work
git branch -D feature/payment   # oops!

# Fix: find SHA in reflog
git reflog
# or
git reflog --all | grep "feature/payment"
# abc123 refs/heads/feature/payment@{0}: commit: last commit on branch

# Restore branch
git switch -c feature/payment abc123

# Verify
git log --oneline -5
```

### Problem: Detached HEAD State

```bash
# Symptom
cat .git/HEAD
# abc123def456...    ← direct SHA instead of "ref: refs/heads/..."

git status
# HEAD detached at abc123def

# Fix Option 1: go back to a branch (lose commits made in detached state)
git switch main

# Fix Option 2: save any work done in detached HEAD
git switch -c recovery-branch    # create branch at current detached position
# Now commits are safely on recovery-branch

# Prevention: never checkout a tag or SHA for development
# Use: git switch -c branch-name <tag/sha>  instead
```

### Problem: Branch Tracking is Broken

```bash
# Symptom
git push
# fatal: The current branch feature/payment has no upstream branch.

# Fix: set upstream
git push --set-upstream origin feature/payment
git push -u origin feature/payment    # shorthand

# OR: set tracking on existing remote branch
git branch --set-upstream-to=origin/feature/payment feature/payment

# Verify
git branch -vv
# * feature/payment abc123 [origin/feature/payment] last commit ✓
```

### Problem: Local Branch Diverged from Remote

```bash
# Symptom
git status
# Your branch and 'origin/feature/payment' have diverged,
# and have 1 and 2 different commits each, respectively.

# Diagnose
git log --oneline --graph origin/feature/payment...HEAD

# Fix Option 1: you haven't pushed yet, rebase onto remote
git pull --rebase

# Fix Option 2: your changes should win (overwrite remote)
git push --force-with-lease

# Fix Option 3: remote changes should win
git fetch origin
git reset --hard origin/feature/payment    # ⚠️ discards local commits
```

---

## 6. Category 4 — Merge & Rebase Problems

## What is it?
Problems occurring during merge or rebase operations: conflicts that seem unresolvable, rebases that lost commits, or duplicate commits after rebase.

## Why It Matters
Knowing how to correctly abort, inspect, and resolve merge/rebase problems is a daily skill for any engineer working in a team.

## Internal Working
During a conflict: `.git/MERGE_HEAD` stores the incoming commit SHA. Index stores three versions (stages 1/2/3). Aborting restores pre-merge state from `.git/ORIG_HEAD`.

## Command Explanation

### Syntax
```bash
git merge --abort               # abort merge
git rebase --abort              # abort rebase
git show :1:file                # base version in conflict
git show :2:file                # ours
git show :3:file                # theirs
git checkout --ours <file>      # take our side entirely
git checkout --theirs <file>    # take their side entirely
```

### Problem: Merge Conflict Won't Resolve

```bash
# Symptom: conflict markers still in file after editing

# Check all conflicted files
git status | grep "both modified"

# See the three versions
git show :1:src/Payment.java    # base (common ancestor)
git show :2:src/Payment.java    # ours
git show :3:src/Payment.java    # theirs

# Open merge tool
git mergetool

# After resolving manually (remove ALL conflict markers):
git add src/Payment.java
git merge --continue   # OR git commit

# Abort the entire merge
git merge --abort

# Verify no conflict markers remain
grep -r "<<<<<<" src/    # should return empty
grep -r ">>>>>>>" src/   # should return empty
```

### Problem: Rebase Lost My Commits

```bash
# Symptom: after rebase, some commits seem gone

# Find them in reflog
git reflog
# HEAD@{0}: rebase finished
# HEAD@{1}: rebase: commit message
# HEAD@{5}: commit: my important commit

# Recover
git reset --hard HEAD@{5}
# OR create a branch there
git switch -c recovery HEAD@{5}

# Better: use ORIG_HEAD (set before rebase/merge/reset)
git reset --hard ORIG_HEAD
```

### Problem: Rebase Created Duplicate Commits

```bash
# Symptom: same change appears twice in log after rebase
# Cause: you merged then rebased, or vice versa

git log --oneline --graph

# Fix: interactive rebase to drop duplicates
git rebase -i HEAD~10
# Drop the duplicate commits with 'd' (drop)

# Prevention: don't mix merge and rebase on the same branch
```

### Problem: Cherry-Pick Causing Conflicts

```bash
# Symptom
git cherry-pick abc123
# CONFLICT (content): Merge conflict in src/Payment.java

# Resolve conflict normally
vim src/Payment.java    # fix conflicts
git add src/Payment.java
git cherry-pick --continue

# OR: skip this commit
git cherry-pick --skip

# OR: abort entirely
git cherry-pick --abort

# Check which commits were successfully cherry-picked
git log --oneline CHERRY_PICK_HEAD..HEAD
```

### Problem: Merge Introduced a Regression

```bash
# Find what the merge commit changed
git show <merge-commit-sha>

# Diff between before and after merge
git diff <parent1-sha>..<merge-commit-sha>

# Revert the merge
git revert -m 1 <merge-commit-sha>    # -m 1 = mainline parent

# Verify revert looks correct before pushing
git show HEAD
```

---

## 7. Category 5 — Remote & Push Problems

## What is it?
Problems with remote operations: push rejections, exposed secrets, wrong remote URLs, authentication failures, slow pushes.

## Why It Matters
Remote problems block the entire team when they affect shared branches. Exposed secrets on public repos require immediate response.

## Internal Working
Push rejection: remote has commits not in local history → not a fast-forward. Fix: fetch first, then rebase or merge. Exposed secret: Git history is immutable — removal requires rewriting all descendant commit SHAs via filter-repo.

## Command Explanation

### Syntax
```bash
git pull --rebase              # fix non-fast-forward
git push --force-with-lease    # after rebasing own branch
git remote set-url origin <url> # fix wrong remote URL
git filter-repo --path secrets.env --invert-paths  # purge secret
```

### Problem: Push Rejected — Non-Fast-Forward

```bash
# Symptom
git push
# error: failed to push some refs to 'origin'
# hint: Updates were rejected because the remote contains work
# that you do not have locally.

# Diagnose
git fetch origin
git log --oneline HEAD..origin/main    # what remote has that you don't

# Fix Option 1: rebase (preferred — linear history)
git pull --rebase

# Fix Option 2: merge
git pull    # creates merge commit

# Fix Option 3: you own this branch + it was rebased
git push --force-with-lease

# Prevention: pull before starting work, sync daily
```

### Problem: Pushed Sensitive Data to Remote

```bash
# IMMEDIATE ACTION:
# 1. Revoke/rotate the credentials NOW (before any Git operations)
# 2. Assume data is compromised

# Remove from history
pip install git-filter-repo
git filter-repo --path secrets.env --invert-paths

# Force push all branches + tags
git push origin --force --all
git push origin --force --tags

# Ask team to re-clone or hard reset
git fetch origin
git reset --hard origin/main

# Contact GitHub support for cache purge (public repos)

# Add to .gitignore
echo "secrets.env" >> .gitignore
git add .gitignore
git commit -m "chore: prevent secrets from being committed"
git push
```

### Problem: Remote Repository URL Changed

```bash
# Symptom
git push
# ERROR: Repository not found.

# Diagnose
git remote -v
# origin  git@github.com:company/old-name.git

# Fix: update remote URL
git remote set-url origin git@github.com:company/new-name.git

# Verify
git remote -v
git fetch origin    # should work now
```

### Problem: SSL/TLS Certificate Error

```bash
# Symptom
git clone https://...
# SSL certificate problem: self signed certificate

# Fix (development only — not production): disable SSL verification
git config --global http.sslVerify false

# Better fix: add CA certificate
git config --global http.sslCAInfo /path/to/ca-cert.pem

# Corporate proxy fix
git config --global http.proxy http://proxy.company.com:8080
git config --global https.proxy http://proxy.company.com:8080
```

### Problem: `git push` Very Slow

```bash
# Diagnose
GIT_TRACE=1 GIT_TRANSFER_TRACE=1 git push 2>&1 | head -50

# Fix: push only specific branch
git push origin feature/payment    # not --all

# Fix: check for large files
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print substr($0,6)}' | sort -k2 -rn | head -5

# Fix: use Git LFS for large files
git lfs track "*.psd"
git add .gitattributes
git lfs migrate import --include="*.psd" --everything
```

---

## 8. Category 6 — History & Data Loss

## What is it?
Scenarios where committed work appears lost: accidental hard reset, amended commits that seem gone, commits made in detached HEAD state.

## Why It Matters
These are the most feared Git scenarios — but almost all are recoverable via reflog within the 90-day window.

## Internal Working
Commit objects persist in `.git/objects/` even when no branch points to them — they are "unreachable" but not deleted until `git gc --prune` runs. The reflog maintains references to all recent HEAD positions, keeping those commits findable.

## Command Explanation

### Syntax
```bash
git reflog                    # ALWAYS first step ⭐
git reset --hard ORIG_HEAD    # undo last merge/rebase/reset
git reset --hard HEAD@{n}     # restore to specific reflog state
git switch -c recovered <sha> # create branch at recovered commit
```

### Problem: Accidental `git reset --hard`

```bash
# Lost N commits with git reset --hard HEAD~N

# Fix: find pre-reset SHA in reflog
git reflog
# abc123 HEAD@{0}: reset: moving to HEAD~3   ← the bad reset
# def456 HEAD@{1}: commit: my important work  ← what we want

git reset --hard def456    # restore!

# Alternative: ORIG_HEAD is set before destructive operations
git reset --hard ORIG_HEAD
```

### Problem: Accidentally Ran `git clean -fd`

```bash
# Removed untracked files

# Sad truth: untracked files have never been committed
# → they are NOT in git objects store → NOT recoverable via git

# Partial recovery options:
# 1. Check IDE local history (IntelliJ: File → Local History)
# 2. Check OS trash / recycle bin
# 3. Check filesystem snapshots (Time Machine, VSS)
# 4. Check if file was open in any editor with undo history

# Prevention: ALWAYS do git clean -n (dry run) first
git clean -n     # shows what WOULD be removed
git clean -fd    # then actually remove
```

### Problem: Branch History Seems Lost After Rebase

```bash
# After rebase, feature branch has different SHAs
# Old commits seem "gone"

# Find original commits
git reflog show feature/payment
# abc123 feature/payment@{0}: rebase finished
# old456 feature/payment@{1}: commit: original commit   ← here it is

git log --oneline old456~3..old456    # verify content

# Restore to pre-rebase state
git switch feature/payment
git reset --hard feature/payment@{1}    # before rebase
```

### Problem: `git commit --amend` Lost My Changes

```bash
# Amended a commit but the amendment went wrong

# Find the pre-amend commit
git reflog
# abc123 HEAD@{0}: commit (amend): new message    ← amended version
# def456 HEAD@{1}: commit: original with changes  ← original!

# Recover original
git reset --hard def456

# OR cherry-pick specific changes from original
git cherry-pick def456
```

### Problem: Merge Commit Accidentally Deleted Good Code

```bash
# Find what the merge removed
git show <merge-sha>    # full diff of merge
git diff <parent1>..<merge-sha> -- src/Payment.java    # file-specific

# Restore specific file from before merge
git restore --source <parent1-sha> src/Payment.java
git commit -m "fix: restore Payment.java accidentally removed in merge"
```

---

## 9. Category 7 — Repository Corruption

## What is it?
Scenarios where the `.git/` object database is damaged — missing objects, broken references, or corrupted pack files.

## Why It Matters
Corruption is rare but catastrophic when it happens. Having a recovery plan and prevention strategy is essential for production-grade repos.

## Internal Working
`git fsck` verifies every object's SHA matches its content and that all referenced SHAs exist. "Missing blob/tree/commit" = true corruption. "Dangling commit" = unreferenced object = normal (unreachable after branch delete).

## Command Explanation

### Syntax
```bash
git fsck --full                    # verify entire repo
git fsck --full 2>&1 | grep "missing"   # find missing objects
git clone <remote> fresh-repo      # recover from healthy remote
```

### Diagnosing Corruption

```bash
# Check repository integrity
git fsck --full
# Possible outputs:
# "dangling blob abc123"    → unreferenced object (not corruption, just orphaned)
# "missing blob abc123"     → ACTUAL corruption
# "broken link"             → object references missing object

# Count objects
git count-objects -v

# Verify pack files
git verify-pack -v .git/objects/pack/pack-*.idx | head -20
```

### Recovery from Corruption

```bash
# Attempt 1: rebuild from remote (if remote is healthy)
git fetch origin
git reset --hard origin/main

# Attempt 2: clone fresh, then apply local changes
git clone git@github.com:company/repo.git fresh-repo
cd fresh-repo
# Manually apply local changes from corrupted repo

# Attempt 3: partial recovery with git fsck
git fsck --full 2>&1 | grep "dangling commit"
# These are commits not referenced by any branch/tag
# Identify your work, then create branches

# Attempt 4: restore from backup
# Always keep a backup/mirror of critical repositories
```

### Prevention

```bash
# Regular integrity checks (add to cron or CI)
git fsck --no-progress --no-dangling 2>&1 | grep -v "^Checking"
# Non-zero exit = corruption detected

# Mirror backup
git clone --mirror git@github.com:company/repo.git /backup/repo.git
# Schedule daily: git -C /backup/repo.git remote update

# GitHub's own backup via GitHub API or gharchive.org
```

---

## 10. Category 8 — Performance Problems

## What is it?
Git operations that are unacceptably slow: `git status` taking seconds, `git log` hanging, clone/push taking minutes.

## Why It Matters
Slow Git operations destroy developer productivity and increase CI costs. Knowing the fix for each performance symptom is a practical senior skill.

## Internal Working
`git status` slowness: compares every file's mtime against index — fixed by filesystem monitor. `git log` slowness: missing commit graph cache — fixed by `git commit-graph write`. Repo size bloat: loose objects and large blobs — fixed by `git gc` and Git LFS.

## Command Explanation

### Syntax
```bash
git config core.fsmonitor true       # faster git status
git commit-graph write --reachable   # faster git log
git gc --aggressive --prune=now      # clean up bloat
git maintenance start                # schedule background tasks
```

### Problem: `git status` is Very Slow

```bash
# Diagnose
time git status

# Fix 1: enable filesystem monitor (Git 2.37+)
git config core.fsmonitor true
git config core.untrackedCache true

# Fix 2: for Node projects — node_modules is the culprit
echo "node_modules/" >> .gitignore
git rm -r --cached node_modules/   # stop tracking if accidentally committed
git commit -m "chore: stop tracking node_modules"

# Fix 3: run maintenance
git maintenance run

# Fix 4: rebuild index
git update-index --refresh
git update-index --really-refresh
```

### Problem: `git log` is Very Slow on Large Repos

```bash
# Fix: generate commit graph (dramatically speeds up log and merge-base)
git commit-graph write --reachable --changed-paths

# Enable automatic commit graph updates
git config fetch.writeCommitGraph true
git config core.commitGraph true    # enable reading commit graph

# Verify commit graph exists
ls .git/objects/info/commit-graph
```

### Problem: Repository is Too Large

```bash
# Diagnose: find large blobs
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | sort -k3 -rn | head -10

# Fix Option 1: remove large files from history
git filter-repo --strip-blobs-bigger-than 10M

# Fix Option 2: use Git LFS going forward
git lfs install
git lfs track "*.zip" "*.jar" "*.tar.gz"
git add .gitattributes
git commit -m "chore: configure Git LFS for large files"

# Fix Option 3: aggressive gc
git gc --aggressive --prune=now

# Check result
git count-objects -vH
```

---

## 11. Category 9 — Authentication Problems

## What is it?
Git authentication failures: SSH key not found, HTTPS credentials expired/wrong, token rotation breaking existing configs.

## Why It Matters
Authentication problems completely block all remote operations. Knowing the diagnostic steps resolves them in 2–5 minutes instead of an hour of frustration.

## Internal Working
SSH: Git uses the SSH protocol; `ssh -T git@github.com` tests the connection independently of Git. HTTPS: Git delegates to the credential helper to retrieve stored username+token. Credential helpers store credentials in OS keychains.

## Command Explanation

### Syntax
```bash
ssh -T git@github.com                        # test SSH connection
ssh-add ~/.ssh/id_ed25519                    # add key to agent
git config --global credential.helper <helper>  # set credential helper
```

### Problem: `Permission denied (publickey)`

```bash
# Diagnose
ssh -vT git@github.com 2>&1 | head -30

# Fix 1: SSH key not loaded
ssh-add ~/.ssh/id_ed25519
ssh -T git@github.com    # test again

# Fix 2: SSH key not added to GitHub
cat ~/.ssh/id_ed25519.pub
# → Add to GitHub: Settings → SSH and GPG keys

# Fix 3: Multiple SSH keys — wrong one being used
# ~/.ssh/config:
# Host github.com
#   IdentityFile ~/.ssh/id_ed25519_work

ssh -T git@github.com    # verify correct key
```

### Problem: HTTPS Asks for Password Every Time

```bash
# Fix: install credential helper
git config --global credential.helper osxkeychain   # macOS
git config --global credential.helper manager       # Windows
git config --global credential.helper store         # Linux (plaintext ⚠️)

# For GitHub: use token not password
# Generate PAT: GitHub → Settings → Developer settings → Personal access tokens
# Username: your-github-username
# Password: <paste-PAT-token>
```

### Problem: `Authentication failed` After Token Rotation

```bash
# Clear cached credentials
# macOS:
git credential-osxkeychain erase
host=github.com
protocol=https
# (press enter twice)

# Windows:
# Credential Manager → Windows Credentials → remove github.com entry

# Linux:
git config --global --unset credential.helper
rm ~/.git-credentials    # if using store helper

# Re-authenticate
git fetch   # will prompt for new token
```

---

## 12. Disaster Recovery Playbook

## What is it?
A tiered playbook for recovering from Git disasters, ordered by severity from local data loss to full repository deletion.

## Why It Matters
Having a documented playbook eliminates panic-driven decisions during incidents. Every senior engineer should be able to execute Level 1–3 recovery in under 15 minutes.

## Internal Working
Recovery always relies on the same two foundations: (1) committed objects persist in `.git/objects/` until GC, (2) the reflog logs all HEAD movements for 90 days. Recovery = find the right SHA, then point HEAD or a branch at it.

## Command Explanation

### Syntax
```bash
# All recovery starts with:
git reflog                         # find the SHA
git reset --hard <sha>             # restore branch to SHA
git switch -c recovery <sha>       # or create branch at SHA
```

### DR Level 1: Local Work Lost (Not Pushed)

```
Scenario: reset --hard, deleted branch, amended wrong commit
Recovery time: 2-5 minutes

1. git reflog                     → find the SHA
2. git reset --hard <sha>         → restore HEAD
   OR git switch -c branch <sha>  → restore as branch
3. Verify: git log --oneline -10
```

### DR Level 2: Pushed Bad Code to Shared Branch

```
Scenario: pushed broken code, tests failing, team blocked
Recovery time: 5-15 minutes

1. Assess: is this a single commit or a merge?
   Single commit:  git revert <sha>     → push
   Merge commit:   git revert -m 1 <sha> → push
2. Push the revert commit
3. Notify team: "REVERT in progress, wait for green CI"
4. Tag the revert: git tag -a hotfix/revert-... if urgent deploy needed
5. Post-mortem: add tests to catch this class of issue
```

### DR Level 3: Secret/Credential Leaked to Remote

```
Scenario: API key, password, private key committed and pushed
Recovery time: 30-60 minutes

IMMEDIATE (before any git operations):
  1. Revoke/rotate credentials NOW
  2. Alert security team
  3. Check access logs for the compromised credential

GIT CLEANUP:
  1. git filter-repo --path secrets.env --invert-paths
  2. git push origin --force --all
  3. git push origin --force --tags
  4. Notify all team members to re-clone
  5. GitHub support: request cache purge (public repos)

PREVENTION:
  1. Add to .gitignore immediately
  2. Install git-secrets or truffleHog
  3. Add pre-commit hook
  4. Add CI secret scanning
```

### DR Level 4: Repository Corruption / Deletion

```
Scenario: .git deleted, objects corrupted, hosting service outage
Recovery time: Hours

OPTION A: Recover from remote (remote is the backup)
  git clone git@github.com:company/repo.git
  cd repo
  # Re-apply any local uncommitted work

OPTION B: Recover from mirror backup
  git clone /backup/repo.git working-repo
  cd working-repo
  git remote set-url origin git@github.com:company/repo.git
  git push origin --all --tags

OPTION C: Partial recovery from team members
  # Other devs have recent clones
  # Ask each to push their local branches:
  git remote add colleague git@github.com:devA/fork.git
  git fetch colleague
  git cherry-pick <commits from colleague>

PREVENTION:
  1. Mirror backup (daily automated)
  2. GitHub has their own redundancy — trust but verify
  3. Clone to separate machine periodically
```

### DR Level 5: Force Push Overwrote Team's Work

```
Scenario: git push --force overwrote remote branch with team's commits
Recovery time: 10-30 minutes

1. STOP everyone from pushing/pulling to that branch

2. Find the lost commits:
   - Check team members' local repos: git reflog on their machines
   - Colleague: git log --oneline origin/feature/payment  (before they pulled)

3. Recover:
   git remote add colleague git@...colleague-machine.git
   git fetch colleague
   # Identify the correct SHA
   git switch feature/payment
   git reset --hard <correct-sha>
   git push --force-with-lease origin feature/payment

4. Notify team to re-pull
   git fetch origin
   git reset --hard origin/feature/payment

5. Enable branch protection to prevent future --force
```

---

## 13. Interview Questions & Model Answers

### Q1: "How would you recover from accidentally running `git reset --hard HEAD~5`?"

**Model Answer:**
> `git reset --hard HEAD~5` moves the branch pointer back 5 commits, but doesn't immediately delete the commit objects. They remain in `.git/objects/` and are logged in the reflog. I'd immediately run `git reflog` to see the state just before the reset — typically listed as `HEAD@{1}` or similar. I'd find the SHA of the commit I was at before (`before-reset-sha`) and run `git reset --hard <before-reset-sha>`. Git moves the pointer back to the correct commit, all five commits are restored. The whole operation takes under a minute. The reflog retains entries for 90 days by default, so this is reliable as long as `git gc` hasn't explicitly pruned unreachable objects.

### Q2: "A developer on your team accidentally force-pushed and overwrote the main branch with an old commit. How do you recover?"

**Model Answer:**
> First, stop everyone from pulling that branch — a bad fetch could overwrite their local good state. Then hunt for the correct commits. If any team member still has the correct state locally, they can push it back: `git push --force-with-lease origin main` from their machine. If not, check the reflog on the server — GitHub's API can show recent pushes, and if they have a mirror/backup, that's the source of truth. Once we identify the correct SHA (from any developer's local `git reflog`), we reset the remote: `git switch main && git reset --hard <correct-sha> && git push --force-with-lease origin main`. After recovery, immediately enable branch protection to disallow force pushes to main — even admins. Long-term, implement at least one automated daily mirror backup.

### Q3: "What's the first thing you do when someone says 'I think I lost my work in Git'?"

**Model Answer:**
> Reassure them — Git almost never permanently deletes committed work. Then: (1) `git reflog` — this shows every position HEAD has been at over the last 90 days. If the work was ever committed, even temporarily, it appears here. (2) Look for the commit SHA right before whatever operation they ran. (3) `git reset --hard <that-sha>` or `git switch -c recovery <sha>` to restore. If the work was never committed (only in the working directory) and they ran `git clean` or `git checkout -- file`, then it's genuinely gone — Git has no record of uncommitted working directory content. In that case, check IDE local history (IntelliJ's Local History feature has saved me many times), OS trash, or filesystem snapshots.

### Q4: "How do you approach debugging a merge conflict that seems impossible to resolve?"

**Model Answer:**
> First I examine all three versions: `git show :1:file` (common ancestor), `git show :2:file` (ours), `git show :3:file` (theirs). Understanding what each branch was trying to accomplish is more important than staring at the diff. If the conflict is in deeply modified code, I use `git log --oneline <base>..<ours> -- <file>` and `git log --oneline <base>..<theirs> -- <file>` to understand the history of each change. Then I decide: is one side clearly correct? Use `git checkout --ours/--theirs`. Are both changes needed? Manually integrate them. Is the architecture wrong (both sides needed incompatible things)? That's a design discussion, not a conflict resolution — abort the merge, have the discussion, then re-attempt. For very complex conflicts, I use a three-panel merge tool like VS Code's built-in merge editor.

---

## 14. Master Troubleshooting Cheatsheet

```bash
# ─── ALWAYS DIAGNOSE FIRST ────────────────────────────────
git status                        # current state
git log --oneline --graph -10     # recent history
git reflog -20                    # all HEAD movements

# ─── WORKING DIRECTORY ────────────────────────────────────
git restore <file>                # discard working dir changes
git restore --source <sha> <file> # restore from specific commit
git clean -n                      # preview what clean would remove
git clean -fd                     # remove untracked files+dirs

# ─── STAGING ──────────────────────────────────────────────
git restore --staged <file>       # unstage file
git diff --staged                 # what's staged

# ─── COMMIT FIXES ─────────────────────────────────────────
git commit --amend                # fix last commit (not pushed!)
git reset --soft HEAD~1           # undo commit, keep staged
git reset --mixed HEAD~1          # undo commit, keep working dir
git reset --hard HEAD~1           # undo commit, DISCARD ⚠️
git reset --hard ORIG_HEAD        # undo last merge/rebase/reset

# ─── BRANCH RECOVERY ──────────────────────────────────────
git reflog                        # find lost commit SHA
git switch -c recovered <sha>     # recreate lost branch
git switch -c branch HEAD         # escape detached HEAD

# ─── MERGE / REBASE ───────────────────────────────────────
git merge --abort                 # abort merge in progress
git rebase --abort                # abort rebase in progress
git cherry-pick --abort           # abort cherry-pick
git show :1:file                  # base version in conflict
git show :2:file                  # ours in conflict
git show :3:file                  # theirs in conflict
git checkout --ours <file>        # take ours for conflicted file
git checkout --theirs <file>      # take theirs for conflicted file

# ─── REMOTE FIXES ─────────────────────────────────────────
git remote set-url origin <url>   # fix remote URL
git push --force-with-lease       # safer force push
git pull --rebase                 # fix diverged local/remote

# ─── DISASTER RECOVERY ────────────────────────────────────
git reflog                        # FIRST TOOL for data recovery ⭐
git reset --hard ORIG_HEAD        # undo last merge/rebase
git reset --hard <sha>            # restore to any past state
git fsck --full                   # check repo integrity
git filter-repo --path x --invert-paths  # remove file from history

# ─── PERFORMANCE ──────────────────────────────────────────
git gc --aggressive               # deep cleanup
git maintenance run               # background optimisation
git commit-graph write --reachable # speed up log
git config core.fsmonitor true    # faster git status
```

---

> **Previous:** [10 · Large-Scale Team Collaboration](./10_Large_Scale_Team_Collaboration.md)  
> **Next:** [12 · Git CI/CD Integration](./12_Git_CICD_Integration.md)
