# 03 · Git File Lifecycle

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead  
> **Series:** Git Knowledge Base — File 3 of 12

---

## Table of Contents
1. [File Lifecycle Overview](#1-file-lifecycle-overview)
2. [git add](#2-git-add)
3. [git status](#3-git-status)
4. [git diff](#4-git-diff)
5. [git commit](#5-git-commit)
6. [git rm --cached](#6-git-rm---cached)
7. [git restore](#7-git-restore)
8. [git reset](#8-git-reset)
9. [git stash](#9-git-stash)
10. [git clean](#10-git-clean)
11. [Interview Questions & Answers](#11-interview-questions--answers)
12. [Quick Revision Cheatsheet](#12-quick-revision-cheatsheet)

---

# 1. File Lifecycle Overview

## What is it?
Every file in a Git project moves through defined **states** as it travels across the three areas (Working Directory → Staging Area → Local Repository).

## Why It Matters
Understanding states and transitions is the foundation for correctly using `git add`, `git commit`, `git restore`, `git reset`, and `git stash`. Every "I lost my work" or "I committed the wrong thing" problem maps to a misunderstanding of these states.

## Internal Working

**The Four States:**
```
UNTRACKED  → Git sees file in Working Dir; no entry in index; no blob in objects/
STAGED     → git add run; blob in objects/; entry in .git/index; ready for commit
UNMODIFIED → file exists identically in Working Dir + index + last commit
MODIFIED   → file in Working Dir differs from its blob SHA in index
```

**The Full Lifecycle Diagram:**
```
Working Directory          Staging Area (.git/index)     Local Repo (.git/objects/)
─────────────────          ──────────────────────────    ──────────────────────────

[UNTRACKED]
    │
    │  git add (first time)
    ▼
[STAGED]  ─────────────────────────────────────────────  git commit  ──► [UNMODIFIED]
    ▲                                                                          │
    │  git add (re-stage)             edit file                               │ edit file
    │                                     │                                   ▼
[STAGED] ◄──────────────────────── [MODIFIED]  ◄──────────────────── [MODIFIED]
    │
    │  git rm --cached
    ▼
[UNTRACKED]
```

## Command Explanation
*(Each command covered in its own section below)*

---

# 2. git add

## What is it?
`git add` moves files from the Working Directory to the Staging Area (index). For new files: Untracked → Staged. For modified files: Modified → Staged.

## Why It Matters
- The staging area lets you craft atomic commits — stage only related changes even when your working directory has many unrelated edits
- `git add -p` (patch mode) is one of the most powerful developer tools — lets you stage specific lines within a file
- Understanding what `git add` does internally explains why committing after edit without re-staging gives you stale staged content

## Internal Working
When `git add src/Payment.java` runs:
```
1. Read Payment.java content from Working Directory
2. Compute blob SHA: SHA-1("blob " + size + "\0" + content)
   = 39ed3cc421c9137fa866cf441d0545c5c00b118c
3. Write compressed blob to .git/objects/39/ed3cc421c9...
4. Update .git/index entry:
   100644 39ed3cc421c9... 0  src/main/java/com/concepts/Payment.java
   (mode) (blob SHA)     (stage) (path)

If re-staged after edits:
   Same path entry is UPDATED (not duplicated) with new blob SHA.
   Old blob remains in objects/ until gc.
```

## Command Explanation

### Syntax
```bash
git add [options] [<pathspec>...]
```

```bash
# Stage specific file
git add src/main/java/com/payment/Payment.java

# Stage an entire directory (and all subdirectories)
git add src/

# Stage all new + modified files in current directory downward
git add .

# Stage all tracked modified/deleted files (NOT new untracked files)
git add -u
git add --update

# Stage all changes repo-wide (new + modified + deleted)
git add -A
git add --all

# INTERACTIVE HUNK STAGING — stage specific lines within a file ⭐
git add -p src/Payment.java
git add --patch src/Payment.java
# Prompts per hunk: y=yes n=no s=split e=edit q=quit ?=help

# Interactive menu mode
git add -i

# Verify what was staged
git ls-files --stage          # shows blob SHAs + file paths in index
git diff --staged             # shows staged diff
```

**Patch mode hunk options:**
```
y = stage this hunk
n = skip this hunk
s = split into smaller hunks (if possible)
e = manually edit the hunk in your editor
q = quit — don't stage any more hunks
a = stage this hunk and all remaining in file
d = don't stage this hunk or any remaining in file
? = show help
```

---

# 3. git status

## What is it?
`git status` shows the current state of the Working Directory and Staging Area: which files are staged, which are modified but unstaged, and which are untracked.

## Why It Matters
- First command to run when diagnosing any Git situation
- Shows the three-way relationship between Working Dir, Index, and last commit
- Tells you exactly what will and won't be in the next commit

## Internal Working
```
git status computes:
  1. Working Dir vs Index    → "Changes not staged for commit" (MODIFIED)
  2. Index vs Last Commit    → "Changes to be committed" (STAGED)
  3. Untracked files         → files not in index at all

For each tracked file:
  - Hash current file content → compare to blob SHA in index
  - If different → MODIFIED
  - If index SHA ≠ last commit SHA → STAGED (for that file)
```

## Command Explanation

### Syntax
```bash
git status [options] [<pathspec>...]
```

```bash
# Full output
git status

# Short/compact format
git status -s
git status --short

# Short format with branch info
git status -sb

# Show ignored files too
git status --ignored

# Short format codes:
# XY filename
# X = index status   (left column)
# Y = working status (right column)
# M = Modified  A = Added  D = Deleted  R = Renamed
# ? = Untracked ! = Ignored
#
# Examples:
# M  file.java     → staged modification (index changed, working matches index)
# MM file.java     → staged modification + additional working dir change
# A  newfile.java  → new file staged
# ?? config.env    → untracked file
# !! node_modules/ → ignored
```

---

# 4. git diff

## What is it?
`git diff` shows the line-by-line differences between versions of files across the three areas (Working Directory, Staging Area, commits).

## Why It Matters
- Before committing: use `git diff --staged` to review exactly what will go into the commit
- During debugging: use `git diff <sha1> <sha2>` to isolate what changed between versions
- The most precise way to understand what's staged vs what's not

## Internal Working
```
git diff          → compares Working Dir blobs vs Index blobs
git diff --staged → compares Index blobs vs Last Commit blobs
git diff HEAD     → compares Working Dir blobs vs Last Commit blobs

All comparisons work by reading the relevant blob content and
running a Myers diff algorithm to produce unified diff output.
```

## Command Explanation

### Syntax
```bash
git diff [options] [<commit>] [--] [<path>...]
```

```bash
# Working Dir vs Staging Area (unstaged changes)
git diff

# Staging Area vs Last Commit (staged changes — what WILL be committed)
git diff --staged
git diff --cached          # identical to --staged

# Working Dir vs Last Commit (all changes)
git diff HEAD

# Between two commits
git diff abc123..def456

# Between two branches
git diff main..feature/payment

# Specific file only
git diff HEAD -- src/Payment.java

# Summary statistics only (no line-by-line)
git diff --stat HEAD

# Word-level diff (useful for documentation)
git diff --word-diff

# Name-only list of changed files
git diff --name-only HEAD

# Name + change type
git diff --name-status HEAD
```

---

# 5. git commit

## What is it?
`git commit` takes the current contents of the Staging Area (index) and permanently records them as a new commit object in the Local Repository, advancing the current branch pointer.

## Why It Matters
- A commit is immutable once created — it cannot be modified without changing its SHA
- Good commits are atomic: one logical change, clear message, all tests passing
- The staging area separation means you control EXACTLY what goes into each commit

## Internal Working
```
When git commit runs:
  1. Reads .git/index entries (all staged files: mode + blob SHA + path)
  2. Builds tree objects bottom-up (leaf dirs first, then parent dirs)
  3. Creates root tree object pointing to all subtrees/blobs
  4. Creates commit object:
       tree   <root-tree-SHA>
       parent <current-HEAD-commit-SHA>
       author <from git config user.name/email> <timestamp>
       committer <same>
       <blank line>
       <commit message>
  5. Writes commit object to .git/objects/
  6. Updates .git/refs/heads/<current-branch> to new commit SHA
  7. HEAD remains pointing to branch → HEAD implicitly advances

After commit: all three areas are in sync → files are UNMODIFIED
```

## Command Explanation

### Syntax
```bash
git commit [options] [-m <message>]
```

```bash
# Standard commit with inline message
git commit -m "feat(payment): add exponential backoff retry"

# Open editor for full subject + body + footer
git commit

# Stage all tracked modified files AND commit in one step
git commit -a -m "fix: correct payment rounding"
# ⚠️ Does NOT stage new untracked files; use git add for those

# Amend the last commit (not yet pushed)
git commit --amend -m "corrected message"
git commit --amend --no-edit   # add staged files, keep same message

# Commit with author override
git commit --author="Ankit <ankit@co.com>" -m "feat: payment"

# Empty commit (useful for CI triggers)
git commit --allow-empty -m "ci: trigger pipeline"

# Verify what will be committed before committing
git diff --staged        # review staged diff
git status               # review file list

# After commit, verify
git log --oneline -3     # see new commit in history
git show HEAD            # see full diff of new commit
```

**Conventional Commit format (industry standard):**
```bash
git commit -m "feat(payment): add retry logic

Implements exponential backoff (3 retries, 2s base delay) when
payment gateway returns 503. Circuit breaker resets after 60s.

Closes: #247"
# type(scope): subject     ← 50 chars max
#                          ← blank line
# body                     ← why, what (72 chars/line)
#                          ← blank line
# footer                   ← issue refs, breaking changes
```

---

# 6. git rm --cached

## What is it?
`git rm --cached` removes a file from the Staging Area (index) without deleting it from the Working Directory. It "untracks" a file — moves it from Staged/Unmodified back to Untracked.

## Why It Matters
- Essential when you accidentally staged or committed a file that should be ignored (config files, secrets, build artifacts)
- Keeps the file on disk while removing Git's tracking of it
- Always pair with adding the file to `.gitignore` to prevent re-staging

## Internal Working
```
BEFORE git rm --cached secrets.env:
  .git/index: 100644 abc123... 0  secrets.env  ← tracked
  .git/objects/: abc123... blob still exists
  Working Dir: secrets.env (exists)

AFTER git rm --cached secrets.env:
  .git/index: (no entry for secrets.env)  ← removed from index
  .git/objects/: abc123... blob still exists (unreferenced until gc)
  Working Dir: secrets.env (STILL EXISTS — only index entry removed)

git status now shows:
  Changes to be committed: deleted: secrets.env  ← if previously committed
  Untracked files: secrets.env                   ← now untracked
```

**⚠️ Warning:** This does NOT remove the file from Git history. If it was previously committed, it's still in past commits. Use `git filter-repo` to purge from history.

## Command Explanation

### Syntax
```bash
git rm --cached [options] <file>
```

```bash
# Untrack a single file
git rm --cached secrets.env

# Untrack an entire directory
git rm --cached -r config/secrets/

# Untrack all .class files
git rm --cached **/*.class

# Complete workflow for accidentally tracked secret:
echo "secrets.env" >> .gitignore   # prevent future staging
git rm --cached secrets.env         # remove from index
git add .gitignore
git commit -m "chore: stop tracking secrets.env"
# ⚠️ ALSO: revoke the exposed credential immediately

# Check: file should still exist on disk
ls secrets.env     # still there
git status         # now shows as untracked or deleted (if previously committed)
```

---

# 7. git restore

## What is it?
`git restore` (Git 2.23+) is the modern command for discarding file changes — either in the Working Directory or in the Staging Area. It replaces the confusing overload of `git checkout` for file operations.

## Why It Matters
- Cleanly separates "undo working directory changes" from "undo staging"
- Safer than `git checkout --` (which had ambiguous syntax)
- Can restore files from any commit, not just HEAD

## Internal Working
```
git restore <file>:
  Takes the blob SHA from .git/index for that file
  Writes it to the Working Directory (overwrites current file)
  → DESTRUCTIVE: working dir changes are LOST

git restore --staged <file>:
  Takes the blob SHA from the last commit (HEAD)
  Updates .git/index entry for that file to the HEAD version
  → Working Directory is UNCHANGED (file stays modified)
  → Index reverts to HEAD version → file appears as "modified" not "staged"
```

## Command Explanation

### Syntax
```bash
git restore [options] [--source=<tree>] [<pathspec>...]
```

```bash
# Discard working directory changes (restore to staged/HEAD version)
# ⚠️ DESTRUCTIVE — working dir changes are LOST
git restore src/Payment.java

# Unstage a file (move from Staged back to Modified)
# Safe — working directory changes are KEPT
git restore --staged src/Payment.java

# Restore from a specific commit (not just HEAD)
git restore --source HEAD~3 src/Payment.java
git restore --source abc123 src/Payment.java
git restore --source v2.0.0 src/Payment.java

# Restore a deleted file
git restore src/DeletedFile.java

# Restore from another branch
git restore --source feature/payment src/Payment.java

# Unstage AND discard working dir changes
git restore --staged --worktree src/Payment.java

# Restore entire directory
git restore src/

# Old syntax (still works, less clear):
git checkout -- src/Payment.java        # same as: git restore src/Payment.java
git reset HEAD src/Payment.java         # same as: git restore --staged src/Payment.java
```

---

# 8. git reset

## What is it?
`git reset` moves the current branch pointer (HEAD) to a specified commit, with three modes that control what happens to the Staging Area and Working Directory.

## Why It Matters
- Most powerful and most dangerous undo command in Git
- Three modes have completely different safety profiles: soft (safe) → mixed (safe) → hard (destructive)
- Used correctly it enables fixing commits, unstaging, and recovering from bad merges
- **Only use on commits not yet pushed to shared branches** — use `git revert` on shared history

## Internal Working
```
All three modes move HEAD + branch pointer to <target>:
  .git/refs/heads/main → <target SHA>

--soft:
  Index: UNCHANGED → still shows the old staged content
  Working Dir: UNCHANGED
  Effect: commits "popped" back to staged state

--mixed (DEFAULT):
  Index: RESET to match <target> → files show as unstaged modifications
  Working Dir: UNCHANGED
  Effect: commits "popped" back to working dir (unstaged)

--hard:
  Index: RESET to match <target>
  Working Dir: RESET to match <target> ⚠️ ALL changes DISCARDED
  Effect: complete rewind, working dir matches <target> commit

ORIG_HEAD: Before any reset/merge/rebase, Git saves current HEAD to ORIG_HEAD
           → git reset --hard ORIG_HEAD to undo the operation
```

## Command Explanation

### Syntax
```bash
git reset [--soft | --mixed | --hard] [<commit>]
git reset [<commit>] -- <pathspec>   # unstage specific file (mixed mode only)
```

```bash
# ── SOFT: undo commit, keep changes STAGED ──────────────
git reset --soft HEAD~1      # undo last commit
git reset --soft HEAD~3      # undo last 3 commits
git reset --soft abc123      # reset to specific commit

# ── MIXED (DEFAULT): undo commit, keep changes UNSTAGED ─
git reset HEAD~1             # same as --mixed
git reset --mixed HEAD~1

# ── HARD: undo commit, DISCARD ALL CHANGES ⚠️ ────────────
git reset --hard HEAD~1
git reset --hard abc123
git reset --hard origin/main   # align with remote (lose all local changes)

# ── UNSTAGE A FILE (mixed mode on path) ─────────────────
git reset HEAD src/Payment.java   # unstage single file (old syntax)
# Modern: git restore --staged src/Payment.java

# ── RECOVERY ────────────────────────────────────────────
git reset --hard ORIG_HEAD   # undo last merge/rebase/reset
git reflog                   # find SHA before the bad reset
git reset --hard <sha>       # restore to that state

# ── SQUASH LAST N COMMITS INTO ONE ──────────────────────
git reset --soft HEAD~5
git commit -m "feat: complete payment module"
```

**Decision guide:**
```
What do I want after reset?
  Keep changes staged:    --soft    (safe, recommit immediately)
  Keep changes unstaged:  --mixed   (safe, re-review before staging)
  Discard everything:     --hard    (DESTRUCTIVE, cannot recover without reflog)
```

---

# 9. git stash

## What is it?
`git stash` temporarily saves the current dirty Working Directory state (and optionally the Staging Area) to a stack, then reverts to the clean last-committed state, allowing you to switch context and restore later.

## Why It Matters
- Enables switching branches or pulling updates without committing incomplete work
- Creates a stack of saved states — can stash multiple contexts
- `git stash pop` restores the most recent stash and removes it from the stack

## Internal Working
```
git stash internally creates special commit objects:
  stash@{0}: a WIP merge commit with 2–3 parents:
    Parent 1: HEAD (base)
    Parent 2: index state (staged changes)
    Parent 3: working dir state (unstaged changes, if -u used)

Stored in .git/refs/stash (single stash head)
and .git/logs/refs/stash (stash stack via reflog)

git stash pop:
  1. Applies the stash commit changes back to working dir + index
  2. Removes the stash entry from .git/refs/stash stack
```

## Command Explanation

### Syntax
```bash
git stash [push | pop | apply | list | drop | show | branch] [options]
```

```bash
# Stash all tracked modified files
git stash
git stash push                         # explicit form

# Stash with a descriptive name
git stash push -m "WIP: payment retry logic"

# Stash including untracked files
git stash push -u
git stash push --include-untracked

# Stash including ignored files (rare)
git stash push -a
git stash push --all

# Stash only specific files
git stash push src/Payment.java

# List all stashes
git stash list
# stash@{0}: WIP on main: abc123 add payment
# stash@{1}: On feature: def456 update UI

# Show stash contents
git stash show                         # summary of latest
git stash show stash@{1}               # specific stash
git stash show -p                      # full diff

# Apply latest stash (KEEPS it in stash list)
git stash apply

# Apply latest stash AND remove from list
git stash pop

# Apply specific stash
git stash apply stash@{1}

# Remove specific stash
git stash drop stash@{0}

# Remove ALL stashes
git stash clear

# Create a branch from a stash (best way to resume abandoned WIP)
git stash branch feature/payment-retry stash@{0}
```

---

# 10. git clean

## What is it?
`git clean` removes **untracked files and directories** from the Working Directory — files that Git has never been asked to track and that are not in `.gitignore`.

## Why It Matters
- Quickly resets working directory to a clean state (e.g., after a failed build left artifacts)
- **Irreversible** — untracked files are NOT in the object store, so they CANNOT be recovered via git
- Always run `git clean -n` (dry run) first to preview

## Internal Working
```
git clean identifies files by:
  - Files NOT in .git/index (untracked)
  - Files NOT matching .gitignore patterns (unless -x flag)

After git clean -fd:
  Untracked files are deleted from disk with OS delete call.
  No git objects were ever created for them.
  CANNOT be recovered via git reflog or git fsck.
```

## Command Explanation

### Syntax
```bash
git clean [options] [<path>...]
```

```bash
# DRY RUN FIRST — always! Shows what would be removed without removing
git clean -n
git clean --dry-run

# Remove untracked files (not directories)
git clean -f

# Remove untracked files AND directories
git clean -fd

# Remove untracked + ignored files (e.g., .env, target/, node_modules/)
git clean -fdx    # ⚠️ VERY DESTRUCTIVE

# Remove only ignored files (keep untracked non-ignored)
git clean -fdX    # capital X

# Interactive mode (confirm each file)
git clean -i

# Remove from specific directory only
git clean -fd src/generated/
```

---

# 11. Interview Questions & Answers

**Q: "Explain the Git staging area and why it exists."**
> The staging area (index, `.git/index`) is an intermediate layer between the Working Directory and Local Repository. `git add` computes a blob, writes it to `.git/objects/`, and records the filepath → blob SHA mapping in the index. The staging area lets you craft precise, atomic commits — you can stage some files but not others, or even stage specific lines within a file using `git add -p`, while leaving other changes unstaged. This enables clean, single-purpose commits even when your working directory has multiple unrelated edits. SVN lacks this — it commits everything changed.

**Q: "What is the difference between `git diff`, `git diff --staged`, and `git diff HEAD`?"**
> `git diff` compares Working Directory to Staging Area — shows changes NOT yet staged. `git diff --staged` compares Staging Area to last commit — shows exactly what WILL be in the next commit. `git diff HEAD` compares Working Directory directly to last commit — combines both staged and unstaged changes. In practice: review `git diff --staged` before every commit to confirm you're committing exactly what you intend.

**Q: "What is the difference between `git reset --soft`, `--mixed`, and `--hard`?"**
> All three move the branch pointer backwards. `--soft` keeps all changes staged in the index — use when you want to undo a commit but re-commit immediately with a better message or different files. `--mixed` (default) keeps changes in Working Directory but unstaged — use when you want to re-review and selectively re-stage. `--hard` discards all changes in both index and Working Directory — destructive, only recoverable via reflog. On shared branches, never use reset — use `git revert` instead to preserve history for others.

**Q: "Can a file be both staged and modified at the same time?"**
> Yes. This happens when you `git add` a file, then edit it again without staging the new changes. The index holds the blob SHA from the time of `git add`; the Working Directory has newer content. `git status` shows the file under both "Changes to be committed" and "Changes not staged for commit". If you `git commit` now, only the staged (older) version is committed. You need to `git add` again to stage the latest content.

**Q: "When would you use `git stash` vs a WIP commit?"**
> `git stash` for short interruptions (switching branch to review a PR, pulling upstream changes). For longer-lived WIP, I prefer a WIP commit: `git commit -m "WIP: payment retry [not for review]"` on the feature branch, then `git commit --amend` later. WIP commits survive machine restarts, are visible in reflog, and can be pushed as backup. Stashes can be forgotten and lost if you switch machines or if `git stash clear` is run accidentally. For abandoning work entirely: stash gives you the option to never come back; a branch preserves it with full history.

---

# 12. Quick Revision Cheatsheet

```bash
# ─── STAGING ──────────────────────────────────────────────
git add <file>              # stage specific file
git add src/                # stage directory
git add .                   # stage all in current dir
git add -A                  # stage ALL changes repo-wide
git add -p                  # interactive hunk staging ⭐
git add -u                  # stage modifications + deletions (not new files)

# ─── INSPECT STATE ────────────────────────────────────────
git status                  # full status
git status -sb              # short + branch info
git diff                    # unstaged changes
git diff --staged           # staged changes (what will commit)
git diff HEAD               # all changes vs last commit
git ls-files --stage        # raw index with SHAs

# ─── COMMIT ───────────────────────────────────────────────
git commit -m "type(scope): description"
git commit --amend          # fix last commit (not pushed!)
git commit --amend --no-edit # add staged to last commit, same message

# ─── UNSTAGE / RESTORE ────────────────────────────────────
git restore --staged <file> # unstage (modern)
git restore <file>          # discard working dir changes ⚠️
git restore --source <sha> <file>  # restore from specific commit

# ─── UNDO COMMITS ─────────────────────────────────────────
git reset --soft  HEAD~1    # undo commit → keep staged
git reset --mixed HEAD~1    # undo commit → keep working dir (default)
git reset --hard  HEAD~1    # undo commit → DISCARD ⚠️
git reset --hard ORIG_HEAD  # undo last merge/rebase

# ─── UNTRACK ──────────────────────────────────────────────
git rm --cached <file>      # remove from index, keep on disk
git rm --cached -r dir/     # remove directory from index

# ─── STASH ────────────────────────────────────────────────
git stash push -m "desc"    # save WIP
git stash -u                # save WIP + untracked
git stash list              # see all stashes
git stash pop               # apply + remove latest
git stash apply stash@{n}   # apply specific

# ─── CLEAN ────────────────────────────────────────────────
git clean -n                # dry run (preview) FIRST
git clean -fd               # remove untracked files + dirs
git clean -fdx              # also remove ignored files ⚠️

# ─── STATE TRANSITION SUMMARY ─────────────────────────────
# Untracked  → Staged:     git add
# Staged     → Untracked:  git rm --cached
# Staged     → Modified:   edit file (re-add to update staged version)
# Staged     → Unmodified: git commit
# Unmodified → Modified:   edit file
# Modified   → Staged:     git add
# Modified   → Unmodified: git restore <file>
# Staged     → Staged:     git add (re-stages with new content)
```

---

> **Previous:** [02 · Git Architecture](./02_Git_Architecture.md)  
> **Next:** [04 · Git Branching & Merging →](./04_Git_Branching_and_Merging.md)
