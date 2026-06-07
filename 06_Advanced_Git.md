# 06 · Advanced Git — Power Commands

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead  
> **Series:** Git Knowledge Base — File 6 of 12

---

## Table of Contents
1. [git reflog](#1-git-reflog)
2. [git log (Advanced Querying)](#2-git-log-advanced-querying)
3. [git bisect](#3-git-bisect)
4. [git blame](#4-git-blame)
5. [git grep](#5-git-grep)
6. [git filter-repo (History Rewriting)](#6-git-filter-repo-history-rewriting)
7. [git worktree](#7-git-worktree)
8. [git hooks](#8-git-hooks)
9. [git archive](#9-git-archive)
10. [git submodule](#10-git-submodule)
11. [Interview Questions & Answers](#11-interview-questions--answers)
12. [Quick Revision Cheatsheet](#12-quick-revision-cheatsheet)

---

# 1. git reflog

## What is it?
`git reflog` (reference log) records every position that HEAD and branch tips have pointed to over the last 90 days. It is Git's ultimate safety net — the recovery tool for "I just destroyed something."

## Why It Matters
- Recovers from: accidental `git reset --hard`, deleted branches, bad rebases, amended commits
- Git almost never permanently deletes committed data — the reflog finds it
- Expires after 90 days (configurable) and is LOCAL only — not pushed to remotes

## Internal Working
```
Every HEAD movement is logged in .git/logs/HEAD:
  abc123 def456 Shrayansh <s@co.com> 1746000000 +0530 commit: feat: add retry
  def456 ghi789 Shrayansh <s@co.com> 1746000001 +0530 reset: moving to HEAD~3
  ghi789 abc123 Shrayansh <s@co.com> 1746000002 +0530 checkout: moving to main

Format: <old-SHA> <new-SHA> <identity> <timestamp> <action>: <description>

git reflog shows this as:
  abc123 HEAD@{0}: commit: feat: add retry logic
  def456 HEAD@{1}: reset: moving to HEAD~3
  ghi789 HEAD@{2}: checkout: moving to main

Expiry config:
  gc.reflogExpire           = 90 days (reachable commits)
  gc.reflogExpireUnreachable = 30 days (unreachable commits)
```

## Command Explanation

### Syntax
```bash
git reflog [show] [<ref>]
git reflog expire [options]
```

```bash
# View all recent HEAD movements
git reflog
git reflog show HEAD        # same

# View reflog for a specific branch
git reflog show main
git reflog show feature/payment

# Recovery: find SHA before bad operation
git reflog
# abc123 HEAD@{0}: reset: moving to HEAD~3  ← the bad reset
# def456 HEAD@{1}: commit: my lost work     ← what we want

# Restore lost commits
git reset --hard def456

# Alternative: ORIG_HEAD (set before merge/rebase/reset)
git reset --hard ORIG_HEAD

# Recover deleted branch
git reflog --all | grep "feature/payment"
git switch -c feature/payment <sha-from-reflog>

# Configure expiry
git config gc.reflogExpire 180              # keep 180 days
git config gc.reflogExpireUnreachable 90
```

---

# 2. git log (Advanced Querying)

## What is it?
`git log` traverses the commit DAG and displays commit history. Advanced options filter by author, date, message content, file path, or code change — turning it into a powerful debugging and audit tool.

## Why It Matters
- `git log -S "string"` (pickaxe) finds exactly which commit introduced or removed a piece of code — essential for debugging regressions
- `git log --graph --all` gives a complete visual map of branch history
- Proper log querying replaces hours of manual code archaeology

## Internal Working
```
git log performs DFS traversal of the commit DAG:
  Starting at HEAD (or specified ref)
  Following parent pointers backward
  Applying filters (--author, --since, --grep, -S) at each node
  Formatting output per --format specification

-S (pickaxe) implementation:
  For each commit: compute diff between commit and its parent
  If the count of <string> occurrences changed in the diff → include commit
  This finds the EXACT commit that added or removed a string
```

## Command Explanation

### Syntax
```bash
git log [options] [<revision range>] [[--] <path>...]
```

```bash
# Visual branch graph ⭐
git log --oneline --graph --all --decorate

# Filter by author
git log --author="Shrayansh"
git log --author="shrayansh@company.com"

# Filter by date range
git log --since="2026-01-01" --until="2026-06-01"
git log --since="2 weeks ago"

# Filter by commit message
git log --grep="payment"
git log --grep="feat" --grep="fix" --all-match   # AND condition

# Filter by code content — pickaxe ⭐ (find who added/removed a string)
git log -S "processPayment"
git log -S "processPayment" --pickaxe-all        # show all files in that commit
git log -G "processPayment.*amount"              # regex on diff content

# Filter by file
git log -- src/Payment.java                      # commits touching this file
git log --follow -- src/Payment.java             # follow file renames

# Range filters
git log main..feature/payment   # in feature, not in main
git log main...feature          # unique to either branch
git log v1.0.0..HEAD            # since tag v1.0.0

# Custom format
git log --pretty=format:"%h %ad | %s [%an]" --date=short

# Limit results
git log -10                     # last 10 commits
git log --no-merges             # exclude merge commits

# Find when a file was added/deleted
git log --diff-filter=A -- src/Payment.java   # A=Added
git log --diff-filter=D -- src/Payment.java   # D=Deleted
```

---

# 3. git bisect

## What is it?
`git bisect` performs a **binary search** through commit history to find the exact commit that introduced a bug or regression. Instead of manually testing each commit, bisect eliminates half the candidates at each step — O(log n) instead of O(n).

## Why It Matters
- Finding a regression in 1000 commits requires only ~10 bisect steps (log₂ 1000 ≈ 10)
- `git bisect run <script>` fully automates the search — most powerful debugging tool in Git
- Standout answer in senior interviews about debugging production regressions

## Internal Working
```
Given: good commit G (working) and bad commit B (broken)
Commits between: G ... C1 C2 C3 ... C8 C9 ... B

Step 1: Check midpoint M1 → bad → search G...M1
Step 2: Check midpoint M2 → good → search M2...M1
Step 3: Check midpoint M3 → bad → search M2...M3
Step 4: M2's child = first bad commit → FOUND

Git checks out each midpoint commit automatically.
You test and mark good/bad.
Binary search finds culprit in O(log n) steps.
```

## Command Explanation

### Syntax
```bash
git bisect start
git bisect bad [<commit>]
git bisect good [<commit>]
git bisect run <script>
git bisect reset
```

```bash
# Manual bisect
git bisect start
git bisect bad                    # current HEAD is broken
git bisect good v1.0.0            # this tag was working
# Git checks out midpoint...
# Test the code...
git bisect bad                    # this midpoint is also bad
# Git checks out new midpoint...
git bisect good                   # this one works!
# Repeat until Git announces: "abc123 is the first bad commit"

git show abc123                   # inspect the culprit
git bisect reset                  # return to original branch

# Automated bisect ⭐ (most powerful)
# Write test.sh: exits 0 if good, non-zero if bad
cat test.sh
# #!/bin/bash
# mvn test -Dtest=PaymentServiceTest -q
# exit $?

git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh          # Git automatically finds first bad commit!
git bisect reset

# Bisect on specific file changes
git bisect start -- src/Payment.java
```

---

# 4. git blame

## What is it?
`git blame` shows which commit and author last modified each line of a file. It answers "who wrote this line and when?" — essential for understanding code context and investigating bugs.

## Why It Matters
- Turns any line of code into a traceable history item — find the commit, PR, and ticket that introduced it
- Combined with `git show <sha>`, reveals the full context and reasoning behind any change
- `-C` flag detects copied/moved lines across files and commits

## Internal Working
```
git blame reads .git/objects to:
  1. Get the file's content at each commit in its history
  2. For each line in the current file:
     Find the MOST RECENT commit that changed that line
     Record: SHA, author, timestamp, line number

Output format:
  abc123  (Shrayansh 2026-05-10 14:32 +0530 1)  package com.payment;
  def456  (Ankit     2026-05-15 09:10 +0530 3)  public class Payment {
  ↑SHA    ↑author    ↑date                ↑line   ↑content
```

## Command Explanation

### Syntax
```bash
git blame [options] [<rev>] [--] <file>
```

```bash
# Blame a file
git blame src/main/java/com/payment/Payment.java

# Blame specific line range
git blame -L 10,25 src/Payment.java      # lines 10–25
git blame -L 10,+15 src/Payment.java     # line 10 + next 15 lines

# Ignore whitespace changes
git blame -w src/Payment.java

# Detect lines moved/copied within same commit
git blame -M src/Payment.java

# Detect lines moved/copied from any commit (thorough)
git blame -CCC src/Payment.java

# Show full SHA (not abbreviated)
git blame --line-porcelain src/Payment.java

# Blame a specific commit's version of a file
git blame abc123 -- src/Payment.java

# Workflow: find who changed a line and why
git blame src/Payment.java | grep "processPayment"
# abc123 (Shrayansh 2026-05-10) public void processPayment(double amount)

git show abc123    # see full context of that commit
```

---

# 5. git grep

## What is it?
`git grep` searches for patterns in tracked files (and across commits). It is faster than system `grep` in Git repos because it searches only tracked files, respects `.gitignore`, and can search across commits.

## Why It Matters
- Searches only Git-tracked files — skips `node_modules/`, `target/`, `.class` files automatically
- Can search historical commits — find when a string existed in the codebase even if deleted
- Much faster than recursive `grep` in large repos

## Internal Working
```
git grep searches by:
  1. Reading .git/index to know which files are tracked
  2. Reading blob objects for those files from .git/objects/
  3. Applying regex/pattern matching on the content
  4. For historical search: walks commit DAG and reads blobs at each commit

Unlike filesystem grep:
  - Does NOT search untracked files
  - Does NOT search ignored files
  - CAN search any past commit without checkout
```

## Command Explanation

### Syntax
```bash
git grep [options] <pattern> [<tree-ish>] [[--] <pathspec>...]
```

```bash
# Search current working tree
git grep "processPayment"

# Case-insensitive
git grep -i "payment"

# Show line numbers
git grep -n "processPayment"

# Count matches per file
git grep -c "processPayment"

# Search only Java files
git grep "processPayment" -- "*.java"

# Search in a specific commit
git grep "processPayment" HEAD~5
git grep "processPayment" v1.0.0

# Search across ALL commits (slow but thorough)
git grep "processPayment" $(git rev-list --all)

# Show context (3 lines before and after)
git grep -B3 -A3 "processPayment"

# Multiple patterns (OR)
git grep -e "processPayment" -e "handlePayment"

# Multiple patterns (AND)
git grep -e "processPayment" --and -e "amount"

# Show only filenames that match
git grep -l "processPayment"
```

---

# 6. git filter-repo (History Rewriting)

## What is it?
`git filter-repo` (modern replacement for `git filter-branch`) rewrites Git history across all commits and branches — removing files, changing author information, or restructuring the repository. It is the nuclear option for history correction.

## Why It Matters
- **REQUIRED** when a secret/credential is accidentally committed — must purge it from ALL history
- Necessary to remove large binary files that bloated the repository
- Used when splitting a monorepo into smaller repos

## Internal Working
```
git filter-repo rewrites history by:
  1. Walking ALL commits in topological order (oldest first)
  2. For each commit: applying your filter (remove file, rename, etc.)
  3. Creating NEW commit objects with new SHAs
  4. Updating all branch and tag pointers to the new SHAs
  5. Result: completely new commit graph with old graph unreferenced

After filter-repo:
  ALL commit SHAs change (because content changed)
  ALL teammates must re-clone (their history now diverges)
  Force push all branches + tags required
```

## Command Explanation

### Syntax
```bash
git filter-repo [options]
```

```bash
# Install
pip install git-filter-repo

# Remove a file from ALL history ⭐
git filter-repo --path secrets.env --invert-paths

# Remove a directory from ALL history
git filter-repo --path config/secrets/ --invert-paths

# Remove all files matching a pattern
git filter-repo --path-glob "*.pem" --invert-paths

# Change email in all commits
git filter-repo --email-callback '
    return email.replace(b"old@email.com", b"new@email.com")
'

# Extract a subdirectory into its own repo
git filter-repo --subdirectory-filter src/payment-module/

# Remove large blobs (>10MB) from history
git filter-repo --strip-blobs-bigger-than 10M

# AFTER filter-repo: force push everything
git push origin --force --all
git push origin --force --tags

# Notify ALL team members — they must re-clone:
# git fetch origin && git reset --hard origin/main
# OR: git clone <url>   (fresh clone)

# ⚠️ ALWAYS: rotate/revoke any exposed credentials BEFORE git operations
```

---

# 7. git worktree

## What is it?
`git worktree` allows checking out **multiple branches simultaneously** in separate directories, all sharing the same `.git` database. No need to clone the repo again.

## Why It Matters
- Review a PR while continuing feature work — without stashing or committing WIP
- Build a release version while developing on main
- All worktrees share objects — no wasted disk space for object duplication

## Internal Working
```
Normal: one working directory linked to .git/
  project/.git/
  project/ (main branch checked out)

With worktrees:
  project/.git/
  project/           (main branch)
  ../hotfix-tree/    (hotfix/v2.1.1 branch) ← separate directory
  ../feature-tree/   (feature/payment branch) ← separate directory

Each worktree has its own:
  - Working Directory (files)
  - HEAD (which branch it's on)
  - index (staging area)

All share the same .git/objects/ — zero object duplication
A branch can only be checked out in ONE worktree at a time
```

## Command Explanation

### Syntax
```bash
git worktree add [options] <path> [<branch>]
git worktree list
git worktree remove <path>
```

```bash
# Add worktree for existing branch
git worktree add ../hotfix-tree hotfix/v2.1.1

# Add worktree and create new branch
git worktree add -b feature/experiment ../experiment-tree

# List all worktrees
git worktree list
# /home/user/project           abc123 [main]
# /home/user/hotfix-tree       def456 [hotfix/v2.1.1]

# Work in worktree — completely independent
cd ../hotfix-tree
git status      # on hotfix/v2.1.1
git commit -m "fix: hotfix"
cd ../project   # back to main

# Remove worktree when done
git worktree remove ../hotfix-tree
git worktree prune              # clean stale metadata
```

---

# 8. git hooks

## What is it?
Git hooks are **scripts that Git executes automatically** at specific lifecycle events: before/after commit, before push, after merge, etc. They live in `.git/hooks/` and enable automation and policy enforcement.

## Why It Matters
- Enforce commit message format (Conventional Commits) before allowing commits
- Run tests or linting before allowing pushes to certain branches
- Prevent accidental direct commits to protected branches
- Share hooks across team via `core.hooksPath` or Husky

## Internal Working
```
Hook scripts in .git/hooks/:
  pre-commit      → runs before commit object is created
  commit-msg      → validates commit message (receives file path as arg)
  pre-push        → runs before git push sends data
  post-commit     → runs after commit is created
  post-merge      → runs after successful merge
  pre-rebase      → runs before rebase begins

Exit code:
  0   → hook passes, Git continues
  non-zero → hook fails, Git ABORTS the operation

.git/hooks/ is NOT committed to the repo.
Share hooks by:
  Option 1: git config core.hooksPath scripts/git-hooks/
  Option 2: Husky (Node.js)
  Option 3: pre-commit framework (Python)
```

## Command Explanation

### Syntax
```bash
# No git subcommand — hooks are shell scripts
chmod +x .git/hooks/<hookname>
git config core.hooksPath <path>
```

```bash
# Create pre-commit hook (runs linter before every commit)
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
mvn checkstyle:check -q
if [ $? -ne 0 ]; then
  echo "❌ Checkstyle failed. Fix before committing."
  exit 1
fi
EOF
chmod +x .git/hooks/pre-commit

# Create commit-msg hook (enforce Conventional Commits)
cat > .git/hooks/commit-msg << 'EOF'
#!/bin/bash
msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert)(\(.+\))?: .{1,100}"
if ! echo "$msg" | grep -qP "$pattern"; then
  echo "❌ Commit message must follow Conventional Commits:"
  echo "   feat(scope): description"
  exit 1
fi
EOF
chmod +x .git/hooks/commit-msg

# Create pre-push hook (prevent direct push to main)
cat > .git/hooks/pre-push << 'EOF'
#!/bin/bash
while read local_ref local_sha remote_ref remote_sha; do
  if [[ "$remote_ref" == "refs/heads/main" ]]; then
    echo "❌ Direct push to main is not allowed. Open a PR."
    exit 1
  fi
done
EOF
chmod +x .git/hooks/pre-push

# Share hooks with team via committed directory
mkdir -p scripts/git-hooks
cp .git/hooks/pre-commit scripts/git-hooks/
git config --local core.hooksPath scripts/git-hooks/
git add scripts/git-hooks/ && git commit -m "chore: add shared git hooks"

# Husky setup (Node.js projects)
npm install husky --save-dev
npx husky init
# Edit .husky/pre-commit and .husky/commit-msg
```

---

# 9. git archive

## What is it?
`git archive` creates a compressed archive (tar, zip) of a repository at a specific commit or tag — WITHOUT including the `.git/` directory. Used for distributing source releases or creating clean build inputs.

## Why It Matters
- Creates a clean source release without Git metadata (no `.git/` in the archive)
- CI/CD pipelines use it to create reproducible, verifiable source bundles
- Smaller than a full clone — only includes tracked files at that point

## Internal Working
```
git archive reads directly from .git/objects/:
  Given a commit or tag → find root tree
  Walk tree recursively → collect all blob paths + content
  Write to archive format (tar/zip)
  Does NOT include .git/ directory
  Does NOT include untracked or ignored files

Result: clean directory snapshot of tracked files only
```

## Command Explanation

### Syntax
```bash
git archive [options] <tree-ish> [<path>...]
```

```bash
# Export HEAD as tar.gz
git archive --format=tar.gz HEAD > release.tar.gz

# Export specific tag
git archive --format=zip v2.1.0 > payment-service-v2.1.0.zip

# Export with version prefix in archive
git archive --format=tar.gz --prefix=payment-service-v2.1.0/ \
  v2.1.0 > payment-service-v2.1.0.tar.gz

# Export only a specific directory
git archive --format=tar.gz HEAD src/ > src-only.tar.gz

# Export directly to remote via SSH
git archive HEAD | gzip | ssh server "cat > release.tar.gz"

# Verify archive contents
tar -tzf release.tar.gz | head -20
```

---

# 10. git submodule

## What is it?
`git submodule` embeds another Git repository inside your repository as a subdirectory, while keeping each repository's history separate. The parent repo stores a reference to a specific commit in the submodule repo.

## Why It Matters
- Enables reusing shared libraries across projects while keeping independent histories
- The parent repo pins the submodule to a specific commit — explicit versioning
- Notorious for causing confusion — many teams replace submodules with package managers

## Internal Working
```
Submodule creates two files in parent repo:
  .gitmodules:
    [submodule "libs/shared"]
        path = libs/shared
        url  = git@github.com:company/shared-lib.git

  .git/modules/libs/shared/  ← actual submodule .git database
  libs/shared/               ← working directory of submodule

Parent repo stores: a special "gitlink" tree entry:
  160000 commit abc123...  libs/shared   ← mode 160000 = submodule
  This is the pinned commit SHA in the submodule repo.

Updating submodule = changing which commit SHA is pinned
  → you must commit this change in the parent repo
```

## Command Explanation

### Syntax
```bash
git submodule [add | init | update | foreach | status] [options]
```

```bash
# Add a submodule
git submodule add git@github.com:company/shared-lib.git libs/shared
git add .gitmodules libs/shared
git commit -m "chore: add shared-lib submodule"

# Clone repo WITH submodules (don't forget!)
git clone --recurse-submodules git@github.com:company/main-app.git

# If you forgot --recurse-submodules
git submodule init
git submodule update

# Shorthand for both
git submodule update --init --recursive

# Update submodule to latest commit on its default branch
git submodule update --remote libs/shared
git add libs/shared
git commit -m "chore: update shared-lib to latest"

# Show submodule status
git submodule status

# Run command in all submodules
git submodule foreach git pull origin main

# Remove a submodule (multi-step)
git submodule deinit libs/shared
git rm libs/shared
rm -rf .git/modules/libs/shared
git commit -m "chore: remove shared-lib submodule"
```

---

# 11. Interview Questions & Answers

**Q: "How would you find the commit that introduced a production regression?"**
> `git bisect run`. I identify a known-good commit (e.g., the last tag before the regression) and mark it `git bisect good v1.5.0`. I mark the current broken HEAD as `git bisect bad`. I write a test script that exits 0 if the behavior is correct, non-zero if broken. Then `git bisect run ./test.sh` fully automates the binary search — Git checks out ~10 commits (for 1000-commit range) and runs my script each time. At the end it prints the first bad commit. Total time: usually under 5 minutes for any size history.

**Q: "How do you recover from an accidental `git reset --hard HEAD~5`?"**
> The reflog. `git reset --hard` moves the branch pointer but doesn't delete the commit objects immediately — they remain in `.git/objects/` and are logged in `.git/logs/HEAD`. I run `git reflog`, find the SHA labeled just before `reset: moving to HEAD~5`, and `git reset --hard <that-sha>`. All five commits are restored instantly. This works because the reflog retains entries for 90 days. The only unrecoverable scenario is if `git gc --prune=now` was run AND 90 days have passed.

**Q: "How would you remove a committed secret from Git history?"**
> First: rotate/revoke the credential immediately — assume it's compromised. Then: `git filter-repo --path secrets.env --invert-paths` to rewrite ALL commits across ALL branches, removing the file entirely. Force-push everything: `git push origin --force --all && git push origin --force --tags`. Notify all teammates to re-clone or `git fetch && git reset --hard origin/main`. For public repos: contact GitHub support for a cache purge. Finally: add the file to `.gitignore` and add secret scanning (truffleHog) to CI to prevent recurrence.

**Q: "What are Git hooks and how do you share them across a team?"**
> Git hooks are shell scripts in `.git/hooks/` that Git executes at lifecycle events. `pre-commit` runs before a commit is created, `commit-msg` validates the message, `pre-push` runs before pushing. The problem: `.git/hooks/` is not committed to the repository so hooks don't share automatically. Solutions: store hook scripts in a committed directory (e.g., `scripts/git-hooks/`) and run `git config core.hooksPath scripts/git-hooks/` in onboarding. Or use Husky (Node.js) which manages `.husky/` as committed files. Server-side hooks on GitHub Enterprise/GitLab are more reliable — they can't be bypassed by developers.

---

# 12. Quick Revision Cheatsheet

```bash
# ─── REFLOG ───────────────────────────────────────────────
git reflog                          # all HEAD movements ⭐
git reflog show main                # branch movements
git reset --hard HEAD@{n}          # restore to reflog state n
git reset --hard ORIG_HEAD          # undo last merge/rebase/reset

# ─── ADVANCED LOG ─────────────────────────────────────────
git log --oneline --graph --all --decorate   # visual DAG ⭐
git log --author="name"
git log --since="2 weeks ago"
git log -S "searchString"           # pickaxe: who added/removed this ⭐
git log -G "regex"                  # regex in diff
git log -- path/to/file             # commits touching file
git log main..feature               # commits in feature not in main

# ─── BISECT ───────────────────────────────────────────────
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
git bisect run ./test.sh            # automated binary search ⭐
git bisect reset

# ─── BLAME ────────────────────────────────────────────────
git blame src/File.java
git blame -L 10,25 src/File.java    # specific lines
git blame -CCC src/File.java        # detect copies

# ─── GREP ─────────────────────────────────────────────────
git grep -n "pattern"               # with line numbers
git grep "pattern" -- "*.java"      # in Java files only
git grep "pattern" v1.0.0           # in specific commit

# ─── FILTER-REPO ──────────────────────────────────────────
git filter-repo --path secrets.env --invert-paths   # remove file from history ⭐
git push origin --force --all && git push origin --force --tags

# ─── WORKTREE ─────────────────────────────────────────────
git worktree add ../hotfix hotfix/critical
git worktree list
git worktree remove ../hotfix

# ─── HOOKS ────────────────────────────────────────────────
chmod +x .git/hooks/pre-commit
git config core.hooksPath scripts/git-hooks/

# ─── ARCHIVE ──────────────────────────────────────────────
git archive --format=tar.gz v2.1.0 > release.tar.gz
git archive --format=zip HEAD src/ > src.zip

# ─── SUBMODULE ────────────────────────────────────────────
git clone --recurse-submodules <url>
git submodule update --init --recursive
git submodule update --remote
```

---

> **Previous:** [05 · Remote Repositories & Collaboration](./05_Remote_Repositories_and_Collaboration.md)  
> **Next:** [07 · Git Interview Master Sheet →](./07_Git_Interview_Master_Sheet.md)
