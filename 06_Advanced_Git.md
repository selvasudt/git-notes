# 06 · Advanced Git — Rewriting History & Power Commands

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [05 · Remote Repositories & Collaboration](./05_Remote_Repositories_and_Collaboration.md)

---

## Table of Contents

1. [The Reflog — Your Ultimate Safety Net](#1-the-reflog--your-ultimate-safety-net)
2. [Advanced `git log` — Querying History](#2-advanced-git-log--querying-history)
3. [`git bisect` — Binary Search for Bugs](#3-git-bisect--binary-search-for-bugs)
4. [`git blame` — Line-Level Authorship](#4-git-blame--line-level-authorship)
5. [`git grep` — Search Codebase History](#5-git-grep--search-codebase-history)
6. [Rewriting History — `filter-branch` & `filter-repo`](#6-rewriting-history--filter-branch--filter-repo)
7. [`git worktree` — Multiple Working Directories](#7-git-worktree--multiple-working-directories)
8. [Git Hooks — Automation at Every Step](#8-git-hooks--automation-at-every-step)
9. [`git archive` — Exporting Snapshots](#9-git-archive--exporting-snapshots)
10. [Advanced Reset & Restore Patterns](#10-advanced-reset--restore-patterns)
11. [`.gitignore` — Patterns & Advanced Usage](#11-gitignore--patterns--advanced-usage)
12. [`git config` — Power Configurations](#12-git-config--power-configurations)
13. [Interview Questions & Model Answers](#13-interview-questions--model-answers)
14. [Quick Revision Cheatsheet](#14-quick-revision-cheatsheet)
15. [Production Incident Playbook](#15-production-incident-playbook)

---

## 1. The Reflog — Your Ultimate Safety Net

### What is the Reflog?

The **reflog** (reference log) records every position that HEAD and branch tips have pointed to over the last 90 days (by default). It is your recovery tool for "I just destroyed something" situations.

```bash
# View full HEAD reflog
git reflog
# or
git reflog show HEAD

# Example output:
# abc123 (HEAD -> main) HEAD@{0}: commit: feat: add retry logic
# def456               HEAD@{1}: reset: moving to HEAD~1
# ghi789               HEAD@{2}: commit: WIP: payment changes
# jkl012               HEAD@{3}: checkout: moving from feature to main
# mno345               HEAD@{4}: commit: initial payment setup
```

### Reflog vs Log

| `git log` | `git reflog` |
|---|---|
| Shows commit history reachable from current HEAD | Shows ALL HEAD movements, including discarded commits |
| Shows only committed snapshots | Shows resets, rebases, checkouts, merges, amendments |
| History visible to everyone | Local only — not shared or pushed |
| Permanent (as long as referenced) | Expires after 90 days (configurable) |

### Recovery Scenarios

```bash
# ── SCENARIO 1: Accidental git reset --hard ──────────────────
git reset --hard HEAD~3   # oops, lost 3 commits

git reflog
# abc123 HEAD@{0}: reset: moving to HEAD~3
# def456 HEAD@{1}: commit: feat: add payment retry    ← want this
# ghi789 HEAD@{2}: commit: test: add payment tests
# jkl012 HEAD@{3}: commit: fix: null check

git reset --hard def456   # restore to before the bad reset

# ── SCENARIO 2: Deleted branch with unmerged work ────────────
git branch -D feature/payment   # oops, unmerged work gone

git reflog
# abc123 HEAD@{4}: checkout: moving from feature/payment to main
# xyz789 HEAD@{5}: commit: WIP: retry logic   ← branch tip

git switch -c feature/payment xyz789   # recreate branch!

# ── SCENARIO 3: Bad rebase ────────────────────────────────────
git rebase main   # rebase went wrong

git reflog
# abc123 HEAD@{0}: rebase finished: refs/heads/feature
# def456 HEAD@{1}: rebase: payment retry
# ORIG_HEAD_sha HEAD@{6}: commit: last good state before rebase

git reset --hard ORIG_HEAD   # ORIG_HEAD is set before rebase/merge
# OR
git reset --hard HEAD@{6}
```

### Reflog Configuration

```bash
# Change expiry (default: 90 days unreachable, 30 days others)
git config gc.reflogExpire 180
git config gc.reflogExpireUnreachable 90

# View reflog for specific branch
git reflog show main
git reflog show origin/main
```

---

## 2. Advanced `git log` — Querying History

### Essential Log Formats

```bash
# Compact one-liner
git log --oneline

# Visual branch graph
git log --oneline --graph --all --decorate

# Full detail
git log -p                          # with full diff
git log --stat                      # with file change summary
git log --shortstat                 # minimal stats

# Custom format (most flexible)
git log --pretty=format:"%h %ad | %s%d [%an]" --date=short
# %h  = short SHA
# %H  = full SHA
# %ad = author date
# %s  = subject (first line of message)
# %d  = ref names (branches, tags)
# %an = author name
# %ae = author email
# %cn = committer name
```

### Filtering the Log

```bash
# By author
git log --author="Shrayansh"
git log --author="shrayansh@company.com"

# By date range
git log --since="2026-01-01"
git log --until="2026-06-01"
git log --since="2 weeks ago"
git log --after="yesterday"

# By commit message
git log --grep="payment"             # commits mentioning "payment"
git log --grep="feat" --grep="fix" --all-match  # both must match

# By file
git log -- src/Payment.java          # commits touching this file
git log --follow -- src/Payment.java # follow renames

# By code content (pickaxe search) ⭐
git log -S "processPayment"          # commits that added/removed this string
git log -G "processPayment.*amount"  # commits where diff matches regex

# Range
git log main..feature/payment        # commits in feature not in main
git log main...feature/payment       # commits unique to either branch
git log v1.0.0..HEAD                 # commits since tag v1.0.0

# Limit
git log -10                          # last 10 commits
git log --skip=5 -10                 # skip 5, show next 10

# Exclude merges
git log --no-merges

# Only merges
git log --merges
```

### Finding the Commit That Introduced a File

```bash
# When was this file created?
git log --diff-filter=A -- src/Payment.java

# When was this file deleted?
git log --diff-filter=D -- src/Payment.java

# Diff filter codes:
# A=Added, D=Deleted, M=Modified, R=Renamed, C=Copied, T=Type changed
```

---

## 3. `git bisect` — Binary Search for Bugs

### What It Does

`git bisect` performs a **binary search through commit history** to find the exact commit that introduced a bug or regression. Instead of manually checking each commit, bisect eliminates half the candidates at each step.

```
Commits: A B C D E F G H I J (latest, broken)
         ↑                 ↑
       good             bad (current)

Step 1: Check E (middle) → bad → narrow to A-E
Step 2: Check B (middle of A-E) → good → narrow to C-E
Step 3: Check D → bad → narrow to C-D
Step 4: Check C → good → D is the first bad commit!

O(log n) instead of O(n)
```

### Manual Bisect Workflow

```bash
# Start bisect
git bisect start

# Mark current (broken) state as bad
git bisect bad

# Mark a known good commit
git bisect good v1.0.0        # tag
git bisect good abc123        # specific SHA

# Git checks out the midpoint commit
# Test the code...
# If broken:
git bisect bad

# If working:
git bisect good

# Repeat until Git identifies the first bad commit:
# "abc123 is the first bad commit"

# Show the bad commit details
git show abc123

# End bisect (return to original branch)
git bisect reset
```

### Automated Bisect (Most Powerful)

```bash
# Write a test script that exits 0 (good) or non-zero (bad)
# test.sh:
#!/bin/bash
mvn test -Dtest=PaymentServiceTest -q
# exits 0 if tests pass, 1 if they fail

git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh
# Git automatically bisects to find the first bad commit!

git bisect reset
```

> **Interview Insight:** `git bisect run` is a standout answer in interviews about debugging production regressions. It shows understanding of binary search efficiency and practical debugging workflows.

---

## 4. `git blame` — Line-Level Authorship

### What It Does

Shows which commit (and author) last modified each line of a file.

```bash
# Blame a file
git blame src/main/java/com/payment/Payment.java

# Output:
# ^abc123 (Shrayansh  2026-05-10 14:32:11 +0530 1)  package com.payment;
# def456  (Ankit      2026-05-15 09:10:00 +0530 2)
# def456  (Ankit      2026-05-15 09:10:00 +0530 3)  public class Payment {
# ghi789  (Shreya     2026-05-20 16:45:22 +0530 4)      public void process() {

# Blame specific lines
git blame -L 10,25 src/Payment.java      # lines 10-25 only
git blame -L 10,+15 src/Payment.java     # line 10, next 15 lines

# Show full SHA (not abbreviated)
git blame --line-porcelain src/Payment.java

# Ignore whitespace changes
git blame -w src/Payment.java

# Detect moved/copied lines (within same commit)
git blame -M src/Payment.java

# Detect moved/copied lines (across any commit)
git blame -C src/Payment.java
git blame -CCC src/Payment.java    # most thorough copy detection
```

### Blame Workflow: Finding the Cause of a Bug

```bash
# Step 1: Find the problematic line
git blame src/Payment.java | grep "processPayment"
# abc123 (Shrayansh 2026-05-10) public void processPayment(double amount) {

# Step 2: Get context around that commit
git show abc123

# Step 3: Find what changed around that time
git log --oneline abc123~5..abc123

# Step 4: Check if this was intentional
git log --oneline -p abc123 -- src/Payment.java
```

---

## 5. `git grep` — Search Codebase History

```bash
# Search current working tree
git grep "processPayment"

# Case-insensitive
git grep -i "payment"

# Show line numbers
git grep -n "processPayment"

# Count matches per file
git grep -c "processPayment"

# Search only specific file types
git grep "processPayment" -- "*.java"

# Search in a specific commit
git grep "processPayment" HEAD~5

# Search across all commits (slow but thorough)
git grep "processPayment" $(git rev-list --all)

# Show context lines (3 before and after)
git grep -B3 -A3 "processPayment"

# Multiple patterns (OR)
git grep -e "processPayment" -e "handlePayment"

# Multiple patterns (AND)
git grep -e "processPayment" --and -e "amount"
```

> **Advantage over `grep`:** `git grep` searches only tracked files, respects `.gitignore`, and can search across commits. It's significantly faster than recursive `grep` in large repos.

---

## 6. Rewriting History — `filter-branch` & `filter-repo`

### When You Need to Rewrite History

- Accidentally committed secrets/credentials (API keys, passwords)
- Need to remove a large file from all history
- Need to change email in all commits (e.g., personal → work)
- Splitting a monorepo into separate repos

### `git filter-repo` (Modern — Recommended)

```bash
# Install
pip install git-filter-repo

# Remove a file from ALL history
git filter-repo --path secrets.env --invert-paths

# Remove a directory from ALL history
git filter-repo --path config/secrets/ --invert-paths

# Change email in all commits
git filter-repo --email-callback '
    return email.replace(b"old@email.com", b"new@email.com")
'

# Change author name in all commits
git filter-repo --name-callback '
    return name.replace(b"Old Name", b"New Name")
'

# Extract a subdirectory into its own repo
git filter-repo --subdirectory-filter src/payment-module/

# After filter-repo, force push all branches (history rewritten)
git push --force --all
git push --force --tags
```

### `git filter-branch` (Legacy — Avoid for New Projects)

```bash
# Remove file from all history (legacy approach)
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch secrets.env' \
  --prune-empty --tag-name-filter cat -- --all

# Clean up after filter-branch
git reflog expire --expire=now --all
git gc --prune=now --aggressive

# Force push
git push origin --force --all
```

### Post-Rewrite Steps

```bash
# After any history rewrite:
# 1. Force push ALL branches and tags
git push origin --force --all
git push origin --force --tags

# 2. Notify ALL team members
# They must re-clone or reset their local repos:
git fetch origin
git reset --hard origin/main

# 3. Revoke and rotate any exposed credentials IMMEDIATELY
# History rewrite alone is NOT sufficient — assume secrets are compromised

# 4. Check GitHub's cached views haven't exposed data
# Contact GitHub support if needed for complete purge
```

---

## 7. `git worktree` — Multiple Working Directories

### What Is It?

`git worktree` lets you check out **multiple branches simultaneously** in separate directories — without cloning the repo again.

### Use Cases

- Review a PR while continuing your own feature work
- Build a specific version while developing on main
- Run tests on main while fixing a bug on a hotfix branch

```bash
# Add a worktree for a branch
git worktree add ../hotfix-worktree hotfix/null-pointer

# Add worktree and create new branch
git worktree add -b feature/experiment ../experiment-worktree

# List all worktrees
git worktree list
# /Users/shreya/project          abc123 [main]
# /Users/shreya/hotfix-worktree  def456 [hotfix/null-pointer]

# Work in the separate directory
cd ../hotfix-worktree
# git status, git commit, etc. — independent working tree

# Remove worktree when done
git worktree remove ../hotfix-worktree
git worktree prune    # clean up stale worktree metadata
```

> **Advantage over multiple clones:** All worktrees share the same `.git` directory and object store. No extra disk space for objects; branch updates in one worktree are visible in all others.

---

## 8. Git Hooks — Automation at Every Step

### What Are Hooks?

Git hooks are **scripts that Git executes automatically** at specific points in the Git workflow. They live in `.git/hooks/`.

```
.git/hooks/
├── pre-commit          ← runs before commit is created
├── prepare-commit-msg  ← runs before commit message editor opens
├── commit-msg          ← validates commit message
├── post-commit         ← runs after commit is created
├── pre-push            ← runs before git push
├── pre-rebase          ← runs before rebase begins
├── post-merge          ← runs after successful merge
└── post-checkout       ← runs after git checkout
```

### Client-Side Hook Examples

```bash
# pre-commit: run tests before every commit
# .git/hooks/pre-commit
#!/bin/bash
echo "Running tests..."
mvn test -q
if [ $? -ne 0 ]; then
    echo "❌ Tests failed. Commit aborted."
    exit 1
fi
echo "✅ Tests passed."

# Make executable
chmod +x .git/hooks/pre-commit
```

```bash
# commit-msg: enforce Conventional Commits format
# .git/hooks/commit-msg
#!/bin/bash
commit_message=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|test|chore|perf|ci|build|revert)(\(.+\))?: .{1,100}"
if ! echo "$commit_message" | grep -qP "$pattern"; then
    echo "❌ Commit message must follow Conventional Commits format:"
    echo "   <type>(<scope>): <description>"
    echo "   e.g.: feat(payment): add retry logic"
    exit 1
fi
```

```bash
# pre-push: prevent pushing to main directly
# .git/hooks/pre-push
#!/bin/bash
protected_branch="main"
current_branch=$(git symbolic-ref HEAD | sed -e 's,.*/\(.*\),\1,')
if [ "$current_branch" = "$protected_branch" ]; then
    echo "❌ Direct push to '$protected_branch' is not allowed."
    echo "   Please create a feature branch and open a PR."
    exit 1
fi
```

### Sharing Hooks with Team (Hooks aren't committed with `.git/`)

```bash
# Option 1: Store hooks in repo, configure path
mkdir -p scripts/git-hooks
cp .git/hooks/pre-commit scripts/git-hooks/
git add scripts/git-hooks/
git config --local core.hooksPath scripts/git-hooks/

# Option 2: Use Husky (Node.js projects)
npm install husky --save-dev
npx husky init
# .husky/pre-commit file is committed to the repo

# Option 3: Use pre-commit framework (Python)
pip install pre-commit
# .pre-commit-config.yaml committed to repo
pre-commit install
```

### Server-Side Hooks (on GitHub/GitLab Enterprise)

| Hook | Trigger | Use Case |
|---|---|---|
| `pre-receive` | Before accepting push | Block force pushes, enforce commit policy |
| `update` | Per branch during push | Enforce branch naming conventions |
| `post-receive` | After push completes | Trigger CI/CD, send notifications |

---

## 9. `git archive` — Exporting Snapshots

```bash
# Export current HEAD as a tar.gz
git archive --format=tar.gz HEAD > release.tar.gz

# Export specific tag
git archive --format=zip v2.1.0 > payment-service-v2.1.0.zip

# Export specific directory only
git archive --format=tar.gz HEAD src/ > src-only.tar.gz

# Export with prefix (useful for releasing)
git archive --format=tar.gz --prefix=payment-service-v2.1.0/ \
  v2.1.0 > payment-service-v2.1.0.tar.gz

# Export directly to remote via SSH (no intermediate file)
git archive HEAD | gzip | ssh server "cat > release.tar.gz"
```

> **Use case:** CI/CD pipelines that need a clean source tree without `.git/` directory (e.g., building Docker images, generating source releases).

---

## 10. Advanced Reset & Restore Patterns

### Reset to Specific File Version from Any Commit

```bash
# Restore a file to its state 3 commits ago
git restore --source HEAD~3 src/Payment.java

# Restore from a specific commit SHA
git restore --source abc123 src/Payment.java

# Restore from a specific tag
git restore --source v1.0.0 src/Payment.java

# Restore a deleted file
git restore src/DeletedFile.java

# Restore from another branch
git restore --source feature/payment src/Payment.java
```

### Advanced Reset Patterns

```bash
# Squash last 5 commits into one (soft reset then commit)
git reset --soft HEAD~5
git commit -m "feat: complete payment module implementation"

# Reset single file in index (unstage without losing working dir changes)
git reset HEAD src/Payment.java      # older syntax
git restore --staged src/Payment.java # modern syntax

# Reset branch to match remote exactly (DESTRUCTIVE)
git fetch origin
git reset --hard origin/main

# Reset to parent's state for specific file
git checkout HEAD~1 -- src/Payment.java
```

---

## 11. `.gitignore` — Patterns & Advanced Usage

### Pattern Syntax

```gitignore
# Ignore specific file
secrets.env

# Ignore by extension
*.class
*.log
*.jar

# Ignore directory
target/
.idea/
node_modules/

# Ignore anywhere in tree
**/*.class

# Negate (un-ignore a previously ignored pattern)
!important.log

# Ignore only at root
/config.local.properties

# Ignore files in specific directory
src/**/*.tmp

# Comments
# This is a comment
```

### Levels of `.gitignore`

```
1. .gitignore in repo root         ← committed, shared with team
2. .gitignore in subdirectories    ← committed, applies to subdirectory
3. ~/.gitignore_global             ← personal, not committed
4. .git/info/exclude               ← local, not committed, not shared
```

```bash
# Set global gitignore
git config --global core.excludesFile ~/.gitignore_global

# ~/.gitignore_global content (editor/OS specific):
.DS_Store
.idea/
*.swp
Thumbs.db
```

### Common `.gitignore` Patterns by Language/Tool

```gitignore
# Java / Maven
target/
*.class
*.jar
*.war

# Node.js
node_modules/
dist/
.env
.env.local

# Python
__pycache__/
*.pyc
*.pyo
.venv/
.env

# IDE
.idea/
.vscode/
*.iml

# OS
.DS_Store
Thumbs.db
```

### Debugging `.gitignore`

```bash
# Why is this file ignored?
git check-ignore -v src/config/local.properties
# .gitignore:5:*.properties   src/config/local.properties

# Show all ignored files
git status --ignored

# Force add an ignored file (bypass .gitignore)
git add -f src/config/important.properties
```

---

## 12. `git config` — Power Configurations

### Must-Know Configuration

```bash
# ── IDENTITY ──────────────────────────────────────────
git config --global user.name  "Shrayansh"
git config --global user.email "shrayansh@company.com"

# ── EDITOR & DIFF TOOLS ───────────────────────────────
git config --global core.editor "code --wait"
git config --global diff.tool   vscode
git config --global merge.tool  vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# ── PULL STRATEGY ─────────────────────────────────────
git config --global pull.rebase true           # always rebase on pull
git config --global pull.ff    only            # only fast-forward

# ── PUSH ──────────────────────────────────────────────
git config --global push.default current       # push current branch to same-name remote
git config --global push.autoSetupRemote true  # auto set upstream on first push

# ── BRANCH ────────────────────────────────────────────
git config --global init.defaultBranch main

# ── REBASE ────────────────────────────────────────────
git config --global rebase.autoStash true      # auto stash before rebase
git config --global rebase.autoSquash true     # auto fixup! commits

# ── ALIASES ───────────────────────────────────────────
git config --global alias.st   "status -sb"
git config --global alias.co   "checkout"
git config --global alias.br   "branch -vv"
git config --global alias.lg   "log --oneline --graph --all --decorate"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.last "log -1 HEAD --stat"
git config --global alias.who  "shortlog -n -s --no-merges"
git config --global alias.aliases "config --get-regexp alias"

# Complex alias with shell command
git config --global alias.cleanup \
  "!git branch --merged main | grep -v 'main\\|*' | xargs git branch -d"

# ── PERFORMANCE ───────────────────────────────────────
git config --global core.preloadIndex true
git config --global core.fscache       true   # Windows
git config --global gc.auto            256

# ── COLOUR ────────────────────────────────────────────
git config --global color.ui auto

# ── CREDENTIAL ────────────────────────────────────────
git config --global credential.helper osxkeychain   # macOS
git config --global credential.helper store          # Linux (plaintext ⚠️)
```

### Useful Inspection

```bash
git config --list --show-origin --show-scope  # all config with source + scope
git config --global --edit                    # open global config in editor
git config --local  --edit                    # open local config in editor
```

---

## 13. Interview Questions & Model Answers

### Q1: "How would you find the commit that introduced a performance regression?"

**Model Answer:**
> I'd use `git bisect`. First identify a commit where performance was acceptable (a known-good point — a tag like `v1.5.0` works well). Mark the current broken state as bad with `git bisect bad HEAD`. Then mark the good state with `git bisect good v1.5.0`. Git performs a binary search — checking out the midpoint commit each time. For each checkout, I run the performance benchmark. If performance is good, `git bisect good`; if bad, `git bisect bad`. Git narrows down logarithmically. For automation, I'd write a script that runs the benchmark and exits 0/1 based on a threshold, then use `git bisect run ./perf-test.sh`. This typically finds the culprit commit in 10-15 steps even across thousands of commits.

### Q2: "How do you recover from an accidental `git reset --hard`?"

**Model Answer:**
> The reflog is the answer. `git reset --hard` moves HEAD to a different commit, making the previous commits unreachable from branches — but they're still in the object store and logged in the reflog. I'd run `git reflog` to find the SHA of the commit I was at before the reset (it appears as the entry just before `reset: moving to ...`). Then `git reset --hard <that-sha>` restores me to that state. The reflog retains entries for 90 days by default, so this works unless a lot of time has passed or `git gc` has run. This is why the reflog is the most important recovery tool in Git.

### Q3: "What are Git hooks and how would you use them in a team?"

**Model Answer:**
> Git hooks are scripts that Git executes at specific lifecycle points. Client-side hooks include `pre-commit` (run linters/tests before allowing a commit), `commit-msg` (enforce commit message format like Conventional Commits), and `pre-push` (prevent direct pushes to main). The challenge with hooks is that `.git/hooks/` isn't committed to the repository, so they don't automatically share with the team. Solutions: store hook scripts in a tracked directory (e.g., `scripts/git-hooks/`) and configure `core.hooksPath` to point there, or use tools like Husky (for Node.js projects) or the `pre-commit` framework which manage this. For truly enforced policies, server-side hooks on GitHub Enterprise or GitLab are more reliable — they can't be bypassed by developers.

### Q4: "How would you remove an accidentally committed secret from Git history?"

**Model Answer:**
> First, treat the secret as compromised — revoke and rotate it immediately, regardless of what Git operations you perform. History rewriting doesn't guarantee the secret hasn't been seen. For the Git cleanup: use `git-filter-repo` (the modern tool — much faster than `git filter-branch`). Run `git filter-repo --path secrets.env --invert-paths` to remove the file from all commits across all branches. Then force-push all branches and tags. All team members must re-clone or do a hard reset to the rewritten remote branches. Also request a GitHub support cache purge if it's a public repository. The key message: rotate the secret first, Git cleanup second.

### Q5: "What is `git bisect run` and why is it powerful?"

**Model Answer:**
> `git bisect run <script>` fully automates the bisect process. You provide a script that returns exit code 0 when the current state is "good" and any non-zero code when it's "bad". Git then automatically runs this script after each checkout during the binary search, marking the result, and continuing until it finds the first bad commit — all without any manual intervention. This is extremely powerful for regression testing because you can run a full test suite as the bisect script. For a repository with 1000 commits between known-good and known-bad states, bisect tests only ~10 commits (log₂ 1000 ≈ 10) instead of all 1000. The script could run unit tests, integration tests, performance benchmarks, or any other automated check.

### Q6: "How do you handle a situation where a secret was committed to a public GitHub repo?"

**Model Answer:**
> Immediate action: rotate/revoke the compromised credentials right now — before any Git operations. Assume they've been harvested by automated scanners (GitHub has them; malicious actors do too). For Git cleanup: if the commit was very recent and not yet pulled by others, `git reset --soft HEAD~1` + remove the file + recommit + force push might suffice. If it's been in history for any time or was public, use `git-filter-repo --path <secret-file> --invert-paths`, force-push all branches/tags, and ask all contributors to re-clone. Request a GitHub cache purge via support. Update `.gitignore` to prevent recurrence. Add secret scanning (GitHub's built-in, or tools like truffleHog, detect-secrets) to CI/CD. Use environment variables or a secrets manager (HashiCorp Vault, AWS Secrets Manager) instead of file-based secrets going forward.

---

## 14. Quick Revision Cheatsheet

```bash
# ─── REFLOG ───────────────────────────────────────────
git reflog                          # all HEAD movements
git reflog show main                # main branch movements
git reset --hard HEAD@{n}          # restore to reflog state n

# ─── LOG POWER ────────────────────────────────────────
git log --oneline --graph --all --decorate  ⭐
git log --author="name"
git log --since="2 weeks ago"
git log -S "searchString"          # pickaxe: who added/removed this
git log -G "regex"                 # regex in diff
git log -- path/to/file            # commits touching file
git log main..feature              # commits in feature not in main

# ─── BISECT ───────────────────────────────────────────
git bisect start
git bisect bad
git bisect good v1.0.0
git bisect run ./test.sh           # automated ⭐
git bisect reset

# ─── BLAME ────────────────────────────────────────────
git blame src/File.java
git blame -L 10,25 src/File.java   # specific lines
git blame -w                        # ignore whitespace
git blame -CCC                      # detect copies

# ─── GREP ─────────────────────────────────────────────
git grep -n "pattern"              # with line numbers
git grep -S "text" HEAD~10         # in specific commit
git grep "pattern" -- "*.java"     # in Java files only

# ─── HOOKS ────────────────────────────────────────────
ls .git/hooks/
chmod +x .git/hooks/pre-commit
git config core.hooksPath scripts/git-hooks/

# ─── WORKTREE ─────────────────────────────────────────
git worktree add ../hotfix hotfix/critical
git worktree list
git worktree remove ../hotfix

# ─── CONFIG ALIASES ───────────────────────────────────
git config --global alias.lg "log --oneline --graph --all --decorate"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.who "shortlog -n -s --no-merges"
```

---

## 15. Production Incident Playbook

### Scenario 1: Identify Regression in Production

```bash
# Find last known-good deploy tag
git tag -l "deploy-*" | sort | tail -5
# deploy-2026-05-01
# deploy-2026-05-08
# deploy-2026-05-15  ← last good
# deploy-2026-05-22  ← regression appeared

git bisect start
git bisect bad deploy-2026-05-22
git bisect good deploy-2026-05-15
git bisect run ./scripts/test-regression.sh
# First bad commit: abc123def
git show abc123def
git bisect reset
```

### Scenario 2: Emergency Rollback

```bash
# Option 1: Revert the bad commit (safest — creates new commit)
git revert abc123def
git push origin main

# Option 2: Deploy previous tag (if CI/CD supports it)
git checkout v2.0.9
# → trigger deployment pipeline

# Option 3: Reset main to last known good (DANGEROUS — force push to main)
git reset --hard v2.0.9
git push --force-with-lease origin main
# Alert team immediately — their local histories will diverge
```

### Scenario 3: Who Changed This and Why?

```bash
# Find the line
git blame src/Payment.java | grep "processPayment"
# abc123 (Shrayansh 2026-05-18 10:22:00) public void processPayment(

# Get full commit context
git show abc123

# Find the ticket/PR linked
git log --oneline abc123~3..abc123
# Check PR description, linked issue
```

### Scenario 4: Recover From Botched Deployment Merge

```bash
# Merge went wrong, main is broken, need to undo the merge
git log --oneline -5
# abc123 (HEAD → main) Merge branch 'feature/bad-feature'
# def456 last stable commit before merge

# Option 1: Revert the merge commit (safest)
git revert -m 1 abc123   # -m 1 = keep mainline parent
git push origin main

# Option 2: Reset to pre-merge (if not yet pulled by others)
git reset --hard def456
git push --force-with-lease origin main

# Notify team
```

---

> **Previous:** [05 · Remote Repositories & Collaboration](./05_Remote_Repositories_and_Collaboration.md)  
> **Next:** [07 · Git Interview Master Sheet](./07_Git_Interview_Master_Sheet.md)
