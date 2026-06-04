# 03 · Git File Lifecycle

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [02 · Git Architecture](./02_Git_Architecture.md)

---

## Table of Contents

1. [The Three Places a File Lives](#1-the-three-places-a-file-lives)
2. [The Four File States](#2-the-four-file-states)
3. [State A — Untracked → Staged (`git add`)](#3-state-a--untracked--staged-git-add)
4. [State B — Staged → Untracked (`git rm --cached`)](#4-state-b--staged--untracked-git-rm---cached)
5. [State C — Staged → Modified (Edit file)](#5-state-c--staged--modified-edit-file)
6. [State D — Staged → Unmodified (`git commit`)](#6-state-d--staged--unmodified-git-commit)
7. [State E — Unmodified → Modified (Edit file)](#7-state-e--unmodified--modified-edit-file)
8. [State F — Modified → Staged (`git add`)](#8-state-f--modified--staged-git-add)
9. [The `.git/index` — Staging Area Internals](#9-the-gitindex--staging-area-internals)
10. [Useful Status & Diff Commands](#10-useful-status--diff-commands)
11. [Advanced Staging Techniques](#11-advanced-staging-techniques)
12. [Undoing Changes — The Right Command for the Right State](#12-undoing-changes--the-right-command-for-the-right-state)
13. [Interview Questions & Model Answers](#13-interview-questions--model-answers)
14. [Quick Revision Cheatsheet](#14-quick-revision-cheatsheet)
15. [Real-World Workflows](#15-real-world-workflows)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. The Three Places a File Lives

At any moment, a file in a Git project exists in one or more of these three locations:

```
┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────┐
│  Working Directory  │   │    Staging Area      │   │  Local Repository   │
│  (editor / disk)   │   │   (.git/index)       │   │  (.git/objects/)    │
│                     │   │                      │   │                     │
│  Actual files you  │   │  Blob hashes of      │   │  Committed blob,    │
│  see and edit      │   │  files queued for    │   │  tree, commit       │
│  in IntelliJ /     │   │  next commit         │   │  objects            │
│  VS Code / Finder  │   │                      │   │                     │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
         ↑                          ↑                          ↑
    git add ──────────────────────→ │                          │
                                git commit ──────────────────→ │
```

### Working Directory

- Repository files as physical bytes on disk
- What you see in your editor (IntelliJ, VS Code, etc.)
- Changes here are NOT tracked by Git until explicitly staged

### Staging Area (Index)

- Lives in `.git/index` — a binary file
- Holds **blob hashes** of files queued for the next commit
- Acts as a **draft area** — you curate exactly what goes into the next commit
- This is one of Git's most powerful (and misunderstood) features

### Local Repository

- Lives in `.git/objects/`
- Contains all committed blobs, trees, and commit objects
- Immutable history — once committed, objects don't change

> **Interview Insight:** The staging area is unique to Git. SVN and older VCS tools commit everything that changed in the working directory. Git's staging area lets you craft precise, atomic commits — even when your working directory has unrelated changes.

---

## 2. The Four File States

```
                    git add (first time)
  ┌──────────┐ ─────────────────────────→ ┌──────────┐
  │UNTRACKED │                             │  STAGED  │ ──git commit──→ ┌────────────┐
  └──────────┘ ←──────── git rm --cached── └──────────┘                 │ UNMODIFIED │
                                                ↑   │                   └────────────┘
                                           git add  └──edit file──→ ┌──────────┐
                                                │                    │ MODIFIED │
                                           ┌────────────┐ ←─────────┴──────────┘
                                           │  MODIFIED  │
                                           └────────────┘
```

| State | Location | Description |
|---|---|---|
| **Untracked** | Working Directory only | Git sees the file but has never been asked to track it |
| **Staged** | Working Directory + Staging Area | Queued for the next commit |
| **Modified** | Working Directory + Staging Area (old version) | File edited after staging; new changes not yet staged |
| **Unmodified** | Working Dir + Staging Area + Local Repo | All three locations are in sync (last committed state) |

---

## 3. State A — Untracked → Staged (`git add`)

### What Happens Internally

When you run `git add <file>`:

1. Git reads the file content from Working Directory
2. Computes the blob SHA-1: `SHA-1("blob " + size + "\0" + content)`
3. Writes the compressed blob object to `.git/objects/`
4. Creates/updates `.git/index` with: `<mode> <sha> <stage_number> <filepath>`

```
BEFORE git add:
  Working Dir: Payment.java (exists)
  .git/index:  (empty / no entry for Payment.java)
  .git/objects: (no blob for Payment.java)

AFTER git add src/main/java/com/concepts/Payment.java:
  Working Dir: Payment.java (unchanged)
  .git/index:  100644 39ed3cc421c9... 0  src/main/java/com/concepts/Payment.java
  .git/objects: 39/ed3cc421c9137fa866cf441d0545c5c00b118c  ← new blob file
```

### `git add` Variants

```bash
# Add a specific file
git add src/main/java/com/concepts/Payment.java

# Add all files in a specific directory (and subdirectories)
git add src/

# Add ALL new/modified files from current directory downward
git add .

# Add all tracked modified files (not untracked new files)
git add -u

# Add all changes everywhere in the repo (from any directory)
git add -A

# Interactively choose hunks to stage
git add -p          # patch mode — one of the most powerful staging tools
git add --patch
```

### Verifying After `git add`

```bash
# Check staging area with SHAs
git ls-files --stage
# 100644 39ed3cc421c9137fa866cf441d0545c5c00b118c 0  src/main/java/com/concepts/Payment.java
# 100644 ad6e40b42a5647beeed2840c5e24cef0273eae77 0  src/main/java/com/concepts/Main.java

# Human-readable status
git status
# On branch main
# Changes to be committed:
#   (use "git rm --cached <file>..." to unstage)
#       new file:   src/main/java/com/concepts/Main.java
#       new file:   src/main/java/com/concepts/Payment.java
#
# Untracked files:
#   .gitignore
#   pom.xml
```

### What "Untracked" Means Precisely

After `git init` but before any `git add`:
- Git is **aware** of the files (they show up in `git status` under "Untracked files")
- Git is **NOT managing** their versions/history
- The file does NOT exist in `.git/index` yet
- No blob object has been created in `.git/objects/` yet

```bash
# Check which files exist in the object store after git add
git cat-file -p 39ed3cc421c9137fa866cf441d0545c5c00b118c
# package com.concepts;
# public class Payment { ... }
```

---

## 4. State B — Staged → Untracked (`git rm --cached`)

### What Happens Internally

`git rm --cached` **removes the entry from `.git/index`** but does NOT delete the file from the Working Directory and does NOT delete the blob from `.git/objects/`.

```bash
# Remove single file from staging area (keep on disk)
git rm --cached src/main/java/com/concepts/Main.java

# Remove entire directory from staging area
git rm --cached -r src/

# Verify: Main.java is now Untracked again
git status
# Changes to be committed:
#       new file:  src/main/java/com/concepts/Payment.java   ← still staged
#
# Untracked files:
#       src/main/java/com/concepts/Main.java                 ← back to untracked!

# Verify index no longer has Main.java
git ls-files --stage
# 100644 39ed3cc421c9... 0  src/main/java/com/concepts/Payment.java
# (Main.java entry is gone)
```

### Common Use Case: Stop Tracking an Accidentally Committed File

```bash
# Scenario: you committed secrets.env and want to stop tracking it
echo "secrets.env" >> .gitignore
git rm --cached secrets.env
git commit -m "stop tracking secrets.env"
# File remains on disk; Git will no longer track future changes
```

> **Warning:** This does NOT remove the file from Git history. Anyone who has cloned the repo before this commit can still see the secret. For secrets, use `git filter-branch` or BFG Repo Cleaner to purge history, then rotate the secrets immediately.

---

## 5. State C — Staged → Modified (Edit file)

### What Happens

When a file is in STAGED state and you **edit it again** in your editor:

- The **old version** (content at time of `git add`) remains in `.git/index`
- The **new version** (your latest edits) exists only in the Working Directory
- The same file now exists in **two different states simultaneously**

```
.git/index entry:   Payment.java → blob hash 39ed3cc... (original content)
Working Directory:  Payment.java → (new edits, NOT yet staged)
```

```bash
git status
# Changes to be committed:
#       new file:  src/main/java/com/concepts/Payment.java   ← STAGED version
#
# Changes not staged for commit:
#       modified:  src/main/java/com/concepts/Payment.java   ← MODIFIED version
```

### Critical Note: Which Version Gets Committed?

> **If you run `git commit` now, ONLY the staged version (old content) gets committed. Your new edits are NOT in the commit.**

To include the new edits:
```bash
git add src/main/java/com/concepts/Payment.java   # re-stage with new content
git commit -m "..."
```

### Seeing the Difference Between Versions

```bash
# Diff: Working Directory vs Staging Area
git diff                            # unstaged changes

# Diff: Staging Area vs Last Commit
git diff --staged                   # staged changes (what will be committed)
git diff --cached                   # same as --staged

# Diff: Working Directory vs Last Commit (combines both)
git diff HEAD
```

---

## 6. State D — Staged → Unmodified (`git commit`)

### What Happens Internally During Commit

```bash
git commit -m "add payment processing"
```

Git performs these steps:

1. **Reads `.git/index`** — gets all staged file entries (hash + path)
2. **Builds tree objects** — one per directory, recursively from leaf to root
3. **Creates root tree object** — representing complete project state
4. **Creates commit object** containing:
   - root tree SHA
   - parent commit SHA (from HEAD)
   - author info (from config)
   - committer info (from config)
   - timestamp
   - commit message
5. **Updates branch pointer** — `.git/refs/heads/main` now points to new commit SHA
6. **HEAD remains pointing to branch** — so HEAD also advances

```
BEFORE commit:
  HEAD → main → C1
  .git/index: [Payment.java → 39ed3cc...]

AFTER git commit -m "add payment":
  HEAD → main → C2   ← C2 is new commit object
  C2 → tree → blob(Payment.java v1)
  C2.parent → C1
```

### Why It's Called "Unmodified"

After a successful commit, the file exists identically in all three places:

```
Working Directory:  Payment.java (content X)
Staging Area:       Payment.java → blob(content X)
Local Repository:   Payment.java → blob(content X) ← in latest commit
```

All three are in sync → **UNMODIFIED** state.

### Commit Best Practices

```bash
# Atomic commit — one logical change per commit
git commit -m "feat: add UPI payment processing"

# Conventional Commits format (widely used in professional teams)
# <type>(<scope>): <description>
git commit -m "feat(payment): add UPI gateway integration"
git commit -m "fix(auth): resolve JWT expiry edge case"
git commit -m "refactor(db): extract repository pattern for UserDAO"
git commit -m "chore: update dependencies to latest patch versions"
git commit -m "docs: add API documentation for payment endpoints"
git commit -m "test: add unit tests for PaymentService"

# Commit with body (for complex changes)
git commit -m "feat: add retry logic to payment processor

Implements exponential backoff (3 retries, 2s base delay) when
payment gateway returns 503. Includes circuit breaker pattern.

Closes: #247
See-also: #198"

# Stage all tracked modified files AND commit in one step
git commit -a -m "fix: correct payment amount rounding"
# WARNING: -a skips staging area; bypass only for simple changes
```

---

## 7. State E — Unmodified → Modified (Edit file)

### What Happens

You edit a committed file in your editor. The Working Directory version diverges from the Staging Area and Repository versions.

```
Working Directory:  Payment.java  (new edits)
Staging Area:       Payment.java  (last committed content) ← .git/index unchanged
Local Repository:   Payment.java  (last committed content) ← .git/objects unchanged
```

```bash
git status
# Changes not staged for commit:
#   (use "git add <file>..." to update what will be committed)
#   (use "git restore <file>..." to discard changes in working directory)
#       modified:   src/main/java/com/concepts/Payment.java
```

Git detects this because it computes the SHA of the current file content and compares it to the SHA stored in `.git/index`. If they differ → MODIFIED.

---

## 8. State F — Modified → Staged (`git add`)

Same as State A internally — Git re-computes the blob, writes the new blob to `.git/objects/`, and **updates the existing entry** in `.git/index` with the new SHA.

```
BEFORE git add (re-staging):
  .git/index: Payment.java → 39ed3cc... (old content hash)

AFTER git add:
  .git/index: Payment.java → 95fbff0... (new content hash)
  .git/objects: 95/fbff0a74ddcf15c576b0920fad594c0baefe4  ← new blob
```

The old blob (`39ed3cc...`) is NOT deleted — it still exists in `.git/objects/`. It just has no index pointer anymore. It becomes unreachable (until GC) unless referenced by a previous commit.

---

## 9. The `.git/index` — Staging Area Internals

### Structure

The index is a binary file. Each entry contains:

```
<ctime> <mtime> <dev> <ino> <mode> <uid> <gid> <size> <sha1> <flags> <filepath>
```

Key fields:
- **mode** — file type and permissions (e.g., `100644`)
- **sha1** — 20-byte blob SHA-1
- **filepath** — relative path from repo root

### Index is Keyed by File Path

There is **exactly one entry per file path** (in the normal case):

```
src/main/java/com/concepts/Main.java    → Hash-1
src/main/java/com/concepts/Payment.java → Hash-2
```

If you `git add` the same file again with new changes:
- The path entry is **updated** (not duplicated) to point to the new blob hash
- The old blob still exists in objects/ until GC

### The Conflict Stage Number

The `<stage>` field (0 in normal operation) becomes 1, 2, or 3 during merge conflicts:

| Stage | Meaning |
|---|---|
| 0 | Normal (no conflict) |
| 1 | Common ancestor version (merge base) |
| 2 | "Ours" version (current branch) |
| 3 | "Theirs" version (branch being merged) |

```bash
# See stage numbers during a merge conflict
git ls-files --stage
# 100644 abc123... 1  src/Payment.java   ← base
# 100644 def456... 2  src/Payment.java   ← ours
# 100644 ghi789... 3  src/Payment.java   ← theirs
```

### Inspecting the Index

```bash
# List all staged files with SHAs and stage numbers
git ls-files --stage

# List only filenames
git ls-files

# Show untracked files
git ls-files --others

# Show ignored files
git ls-files --ignored --exclude-standard

# Show modified files (working dir vs index)
git ls-files --modified
```

---

## 10. Useful Status & Diff Commands

### `git status`

```bash
git status                  # full output
git status -s               # short format (M=modified, A=added, ?=untracked)
git status --short          # same as -s
git status --branch         # include branch info in short format

# Short format codes:
# XY filename
# X = staging area status (index)
# Y = working directory status
# M = modified, A = added, D = deleted, R = renamed, ? = untracked, ! = ignored
# Example:
# MM src/Payment.java  ← modified in both index AND working dir
# A  src/NewFile.java  ← added to index
# ?? config/secret.env ← untracked
```

### `git diff`

```bash
# Working Directory vs Staging Area (unstaged changes)
git diff

# Staging Area vs Last Commit (staged changes — what will be committed)
git diff --staged
git diff --cached           # identical to --staged

# Working Directory vs Last Commit
git diff HEAD

# Diff between two commits
git diff abc123..def456

# Diff between two branches
git diff main..feature/payment

# Diff specific file only
git diff HEAD -- src/Payment.java

# Summary of changes (stat only, no line-by-line)
git diff --stat HEAD

# Show word-level diff (useful for documentation)
git diff --word-diff
```

---

## 11. Advanced Staging Techniques

### Patch Mode (`git add -p`) — Stage Specific Hunks

One of the most powerful and underused Git features. Lets you stage **parts of a file** — not the whole file.

```bash
git add -p src/Payment.java

# Git shows each hunk and asks:
# Stage this hunk [y,n,q,a,d,/,s,e,?]?
# y = yes, stage this hunk
# n = no, skip this hunk
# s = split into smaller hunks
# e = manually edit the hunk
# q = quit, don't stage anything else
# ? = help
```

**Real-world use case:** You made two unrelated changes in one file. Stage only the first change for commit A, then the second for commit B — keeping your history clean and atomic.

### Interactive Staging (`git add -i`)

```bash
git add -i
# Opens interactive menu:
# 1: status   2: update   3: revert   4: add untracked
# 5: patch    6: diff     7: quit     8: help
```

### Stashing — Temporarily Shelving Changes

```bash
# Stash all tracked modified files
git stash

# Stash with a descriptive name
git stash save "WIP: payment retry logic"

# Stash including untracked files
git stash -u
git stash --include-untracked

# List all stashes
git stash list
# stash@{0}: WIP on main: abc123 add payment
# stash@{1}: On feature: def456 update UI

# Apply most recent stash (keeps stash in list)
git stash apply

# Apply and remove from stash list
git stash pop

# Apply specific stash
git stash apply stash@{1}

# Drop a stash
git stash drop stash@{0}

# Clear all stashes
git stash clear

# Create a branch from a stash
git stash branch feature/payment-retry stash@{0}
```

---

## 12. Undoing Changes — The Right Command for the Right State

This is one of the most important (and confusing) topics in Git. The right command depends entirely on the current state of the file.

### Decision Matrix

```
WHERE is the change I want to undo?
│
├── Working Directory only (not staged, not committed)
│   └── git restore <file>                     # discard working dir changes
│       git checkout -- <file>                 # older syntax, same effect
│
├── Staging Area (staged, not committed)
│   ├── git restore --staged <file>            # unstage (keep working dir changes)
│   │   git reset HEAD <file>                  # older syntax
│   └── git rm --cached <file>                 # unstage untracked file
│
├── Last commit (not yet pushed)
│   ├── git commit --amend                     # redo last commit
│   ├── git reset --soft HEAD~1                # undo commit, keep staged
│   ├── git reset --mixed HEAD~1               # undo commit, keep in working dir (DEFAULT)
│   └── git reset --hard HEAD~1                # undo commit, DISCARD changes ⚠️
│
└── Already pushed to remote
    ├── git revert <commit-sha>                # safe: creates new undo commit
    └── git push --force-with-lease            # DANGEROUS: rewrites shared history
```

### `git restore` (Git 2.23+, recommended)

```bash
# Discard working directory changes (revert to staged/last-commit version)
git restore src/Payment.java

# Unstage a file (move from staged back to modified)
git restore --staged src/Payment.java

# Restore a deleted file
git restore src/DeletedFile.java

# Restore from a specific commit
git restore --source abc123 src/Payment.java
```

### `git reset` Modes

```bash
# --soft: move HEAD back, keep changes STAGED
git reset --soft HEAD~1
# Use when: committed too early, want to re-commit with different message/files

# --mixed (default): move HEAD back, keep changes in WORKING DIR (unstaged)
git reset HEAD~1
git reset --mixed HEAD~1
# Use when: committed wrong files, want to re-do staging

# --hard: move HEAD back, DISCARD all changes ⚠️ DESTRUCTIVE
git reset --hard HEAD~1
# Use when: you're absolutely sure you want to throw away the work
# Recovery: git reflog → git reset --hard <sha>

# Reset to remote state (dangerous — discards all local unpushed work)
git reset --hard origin/main
```

### `git revert` — The Safe "Undo" for Shared Branches

```bash
# Create a new commit that reverses the changes of abc123
git revert abc123

# Revert without auto-committing (so you can edit the revert commit)
git revert --no-commit abc123

# Revert a merge commit (must specify which parent to revert to)
git revert -m 1 <merge-commit-sha>   # -m 1 = keep mainline (parent 1)
```

> **Interview Insight:** Use `git revert` on shared/public branches. Use `git reset` only on local/private branches. `git revert` is non-destructive — it adds to history. `git reset --hard` rewrites history and can permanently lose work.

---

## 13. Interview Questions & Model Answers

### Q1: "Explain the Git staging area and why it exists."

**Model Answer:**
> The staging area (also called the index) is an intermediate zone between the working directory and the local repository. It lives in `.git/index`. When you run `git add`, Git computes the blob SHA, writes it to `.git/objects/`, and records the mapping of filepath → blob SHA in the index. The staging area lets you precisely control what goes into each commit — you can stage some files but not others, or even stage specific hunks within a file using `git add -p`. This enables atomic, single-purpose commits even when your working directory has multiple unrelated changes. SVN and older VCS tools lack this — they commit everything changed, making it harder to maintain a clean, readable history.

### Q2: "What is the difference between `git diff`, `git diff --staged`, and `git diff HEAD`?"

**Model Answer:**
> `git diff` compares the working directory to the staging area — it shows changes you've made but NOT yet staged. `git diff --staged` (or `--cached`) compares the staging area to the last commit — it shows exactly what will be included in the next commit. `git diff HEAD` compares the working directory directly to the last commit — it combines both staged and unstaged changes. In a workflow sense: `git diff --staged` is what you review before committing to ensure you're committing what you intended.

### Q3: "What happens internally when you run `git commit`?"

**Model Answer:**
> Git reads the `.git/index` file and builds tree objects — one per directory, recursively from leaves to root. It then creates a commit object containing: the root tree SHA, the parent commit SHA (from `HEAD`), author/committer info from config, and the commit message. The commit SHA is computed from all this content. Finally, Git updates the current branch pointer (e.g., `.git/refs/heads/main`) to point to the new commit SHA. Since HEAD points to the branch, HEAD also moves forward. The working directory and staging area are unchanged — all three locations now hold identical content, so files enter UNMODIFIED state.

### Q4: "What is the difference between `git reset --soft`, `--mixed`, and `--hard`?"

**Model Answer:**
> All three move the branch pointer (HEAD) backwards to a specified commit. The difference is what happens to your changes. `--soft` keeps all changes staged in the index — use this when you want to undo the last commit but keep everything ready to re-commit, perhaps with a better message. `--mixed` (the default) keeps changes in the working directory but unstages them — use this when you want to re-examine and re-stage your changes. `--hard` discards all changes in both the staging area and working directory — it's destructive and should be used with caution; recovery is only possible via `git reflog`. On shared branches, never use reset — use `git revert` instead to preserve history.

### Q5: "What is the difference between `git rm --cached` and `git restore --staged`?"

**Model Answer:**
> `git rm --cached <file>` removes the file from the index (staging area) entirely — after this, if the file was previously committed, it will show as "deleted" in the next commit. It's used when you want to stop tracking a file altogether (e.g., accidentally committed a config file). `git restore --staged <file>` unstages a file but restores the index to the last committed state — the file goes from staged back to modified, and it will still be tracked. Use `git restore --staged` when you accidentally staged something and want to un-stage it without affecting tracking.

### Q6: "Can a file be both staged and modified at the same time?"

**Model Answer:**
> Yes, absolutely. This happens when you stage a file with `git add` and then edit it again without staging the new changes. The staging area holds the blob hash from the time you ran `git add`, while the working directory has the newer version. `git status` will show the file under both "Changes to be committed" (staged version) and "Changes not staged for commit" (working dir version). If you run `git commit` at this point, only the staged version gets committed — your newer edits are NOT included. You'd need to run `git add` again to stage the latest version before committing.

### Q7: "When would you use `git stash`?"

**Model Answer:**
> `git stash` is useful when you have uncommitted changes in your working directory but need to switch context urgently — for example, switching to another branch to fix a critical production bug, or pulling upstream changes that would conflict with your work-in-progress. Stash temporarily saves your dirty working state (tracked modified files and optionally untracked files with `-u`) onto a stack and reverts your working directory to the last committed state. You can later restore the stash with `git stash pop`. For longer-lived WIP, I prefer creating a WIP commit on a feature branch (`git commit -m "WIP: [description]"`) and amending it later — stashes can be forgotten and lost.

---

## 14. Quick Revision Cheatsheet

```bash
# ─── STAGING ──────────────────────────────────────────
git add <file>              # stage specific file
git add src/                # stage entire directory
git add .                   # stage all in current dir
git add -A                  # stage all changes repo-wide
git add -p                  # interactive hunk staging ⭐
git add -u                  # stage modifications/deletions (not new files)

# ─── INSPECT STATE ────────────────────────────────────
git status                  # full status
git status -s               # short status
git diff                    # working dir vs staging area (unstaged)
git diff --staged           # staging area vs last commit (staged)
git diff HEAD               # working dir vs last commit (all)
git ls-files --stage        # raw index contents with SHAs

# ─── COMMIT ───────────────────────────────────────────
git commit -m "msg"         # commit staged changes
git commit -am "msg"        # stage tracked + commit (skip staging area)
git commit --amend          # modify last commit (message or files)
git commit --amend --no-edit # amend keeping same message

# ─── UNSTAGE ──────────────────────────────────────────
git restore --staged <file> # unstage (modern, Git 2.23+)
git reset HEAD <file>       # unstage (older syntax)
git rm --cached <file>      # stop tracking file

# ─── DISCARD CHANGES ──────────────────────────────────
git restore <file>          # discard working dir changes
git checkout -- <file>      # discard working dir changes (older)
git clean -fd               # remove untracked files+dirs ⚠️
git clean -n                # dry run (show what would be removed)

# ─── UNDO COMMITS ─────────────────────────────────────
git reset --soft  HEAD~1    # undo commit, keep staged
git reset --mixed HEAD~1    # undo commit, keep in working dir (default)
git reset --hard  HEAD~1    # undo commit, DISCARD changes ⚠️
git revert <sha>            # safe undo (new commit) — use on shared branches

# ─── STASH ────────────────────────────────────────────
git stash                   # stash tracked changes
git stash -u                # stash including untracked
git stash list              # list stashes
git stash pop               # apply + remove latest stash
git stash apply stash@{n}   # apply specific stash
git stash drop stash@{n}    # remove specific stash
```

---

## 15. Real-World Workflows

### Workflow A: Clean Feature Development

```bash
# Start feature branch
git switch -c feature/payment-retry

# Work on changes...
# Edit PaymentService.java, RetryConfig.java, PaymentServiceTest.java

# Review what changed
git status
git diff

# Stage application code separately from test code
git add src/main/java/com/payment/PaymentService.java
git add src/main/java/com/payment/RetryConfig.java
git commit -m "feat(payment): add exponential backoff retry logic"

git add src/test/java/com/payment/PaymentServiceTest.java
git commit -m "test(payment): add retry logic unit tests"

# Push branch
git push -u origin feature/payment-retry
```

### Workflow B: Emergency Hotfix While Mid-Feature

```bash
# You're mid-feature with unstaged changes
git status
# modified: src/Payment.java (your WIP)

# Urgent: production bug found!
git stash save "WIP: payment retry feature"

# Switch to hotfix branch from main
git switch main
git pull origin main
git switch -c hotfix/null-pointer-in-payment

# Fix the bug
# ... edit, test ...
git add .
git commit -m "fix(payment): resolve NPE when amount is null"
git push origin hotfix/null-pointer-in-payment
# → Open PR → Merge → Deploy

# Back to your feature
git switch feature/payment-retry
git stash pop
# Continue where you left off
```

### Workflow C: Surgical Commit with Patch Mode

```bash
# You've made two unrelated changes to Payment.java:
# 1. Fixed a bug (lines 20-35)
# 2. Added a new feature (lines 80-100)
# You want two separate commits for clarity

git add -p src/Payment.java
# Git shows first hunk (bug fix):
# Stage this hunk [y,n,q,a,d,/,s,e,?]? y   ← stage bug fix

# Git shows second hunk (new feature):
# Stage this hunk [y,n,q,a,d,/,s,e,?]? n   ← skip feature

git commit -m "fix(payment): handle null amount edge case"

# Now stage the feature hunk
git add -p src/Payment.java
# Stage this hunk [y,n,q,a,d,/,s,e,?]? y

git commit -m "feat(payment): add payment summary calculation"
```

### Workflow D: Amending the Last Commit

```bash
# You just committed but forgot to include a file, or typo in message

# Case 1: Add forgotten file to last commit
git add src/ForgottenFile.java
git commit --amend --no-edit       # keep same message

# Case 2: Fix typo in last commit message
git commit --amend -m "feat(payment): add retry logic with exponential backoff"

# Case 3: Change last commit author (if you used wrong git identity)
git commit --amend --author="Correct Name <correct@email.com>"

# ⚠️ Only amend commits not yet pushed to shared branches
# If already pushed: git push --force-with-lease (use with caution)
```

---

## 16. Troubleshooting

| Problem | Symptom | Solution |
|---|---|---|
| Staged wrong file | File shows under "Changes to be committed" | `git restore --staged <file>` |
| Modified after staging | File shows in both staged and modified | `git add <file>` to re-stage new version |
| Lost uncommitted work | Accidentally ran `git checkout --` | ⚠️ Cannot recover — working dir changes are gone if not staged/stashed |
| Committed wrong file | File in last commit that shouldn't be | `git reset --soft HEAD~1` → unstage → re-commit |
| Accidentally committed sensitive file | `.env` or `secrets.json` in history | `git rm --cached <file>` + add to `.gitignore` + `git filter-branch` or BFG to purge history |
| `git commit -a` staged unwanted files | `-a` auto-staged unintended changes | `git reset --mixed HEAD~1` → review → selective `git add` |
| Stash lost after `git stash drop` | Stash accidentally dropped | `git fsck --unreachable \| grep commit` — stash commits may still be in objects |
| Merge conflict in index | File has stage 1/2/3 entries | Resolve conflict → `git add <file>` → `git commit` |

---

> **Previous:** [02 · Git Architecture](./02_Git_Architecture.md)  
> **Next:** [04 · Git Branching & Merging](./04_Git_Branching_and_Merging.md)
