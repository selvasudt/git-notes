# 04 · Git Branching & Merging

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [03 · Git File Lifecycle](./03_Git_File_Lifecycle.md)

---

## Table of Contents

1. [What is a Branch?](#1-what-is-a-branch)
2. [Branch Internals](#2-branch-internals)
3. [Creating & Switching Branches](#3-creating--switching-branches)
4. [Types of Merge](#4-types-of-merge)
5. [Fast-Forward Merge](#5-fast-forward-merge)
6. [Three-Way Merge](#6-three-way-merge)
7. [Merge Conflicts — Resolution Deep Dive](#7-merge-conflicts--resolution-deep-dive)
8. [Rebase vs Merge](#8-rebase-vs-merge)
9. [Interactive Rebase](#9-interactive-rebase)
10. [Cherry-Pick](#10-cherry-pick)
11. [Branching Strategies](#11-branching-strategies)
12. [Remote Branches & Tracking](#12-remote-branches--tracking)
13. [Interview Questions & Model Answers](#13-interview-questions--model-answers)
14. [Quick Revision Cheatsheet](#14-quick-revision-cheatsheet)
15. [Real-World Workflows](#15-real-world-workflows)
16. [Troubleshooting](#16-troubleshooting)

---

## 1. What is a Branch?

### Definition

A **branch** is a lightweight, movable pointer to a commit. Nothing more. It is a 41-byte file containing a SHA-1 hash.

```bash
cat .git/refs/heads/main
# 32686eaaf60e5e23498b286d23a28a3387241e38
```

### Why Branches Exist

- **Isolation** — develop features without affecting stable code
- **Parallel development** — multiple features in flight simultaneously
- **Experimentation** — try risky ideas safely; discard if they fail
- **Release management** — maintain multiple production versions
- **Code review** — PRs/MRs are branch-based

### The Branch Metaphor (and Why It's Wrong)

Many people think of branches as "copies" of code. They're not. A branch is simply a **named pointer to a commit**. Creating a branch costs:
- One file write (~41 bytes)
- Zero file copies
- Zero network operations

This is why Git branching is instant, and why you should **branch early and branch often**.

---

## 2. Branch Internals

### Branch as a File

```
.git/refs/heads/
├── main          → 32686eaa...
├── develop       → a1b2c3d4...
└── feature/pay   → f9e8d7c6...
```

Each file contains exactly one SHA — the commit that branch currently points to.

### HEAD Points to the Current Branch

```
Normal state:
  HEAD → refs/heads/main → C4

After git switch feature/pay:
  HEAD → refs/heads/feature/pay → C3
```

HEAD is a **symbolic ref** — it points to a branch, which points to a commit.

### How Branches Move

When you make a commit on a branch:
1. New commit `C5` is created
2. `C5.parent = C4` (previous HEAD commit)
3. `.git/refs/heads/main` is updated to `C5`'s SHA
4. HEAD still points to `main` → HEAD now implicitly points to `C5`

Branches always point to their **latest** commit and advance automatically on each new commit.

---

## 3. Creating & Switching Branches

### Modern Commands (Git 2.23+)

```bash
# Create branch (stay on current branch)
git branch feature/payment

# Switch to existing branch
git switch feature/payment

# Create AND switch in one step ⭐ (most common)
git switch -c feature/payment
git switch --create feature/payment

# Create branch from specific commit/tag/branch
git switch -c hotfix/v2.1.1 v2.1.0           # from tag
git switch -c hotfix/v2.1.1 abc123def        # from commit SHA
git switch -c feature/pay origin/feature/pay  # from remote branch

# Return to previous branch (like cd -)
git switch -
```

### Legacy Commands (Still Works, Widely Used)

```bash
git checkout -b feature/payment              # create + switch
git checkout feature/payment                 # switch only
git checkout -b feature/pay origin/feature/pay  # track remote
```

### Listing Branches

```bash
git branch                  # local branches (* = current)
git branch -v               # with last commit SHA + message
git branch -vv              # with upstream tracking info
git branch -r               # remote-tracking branches
git branch -a               # all (local + remote)

# Branches merged into current branch (safe to delete)
git branch --merged

# Branches NOT yet merged (be careful deleting these)
git branch --no-merged
```

### Deleting Branches

```bash
# Delete merged branch (safe)
git branch -d feature/payment

# Force delete (even if not merged) ⚠️
git branch -D feature/payment

# Delete remote branch
git push origin --delete feature/payment
git push origin :feature/payment            # older syntax

# Prune stale remote-tracking refs
git fetch --prune
git remote prune origin
```

### Renaming a Branch

```bash
# Rename current branch
git branch -m new-name

# Rename specific branch
git branch -m old-name new-name

# Rename remote branch (delete old + push new)
git push origin --delete old-name
git push origin new-name
git push --set-upstream origin new-name
```

---

## 4. Types of Merge

Git has two primary merge strategies:

| Merge Type | Creates New Commit? | Preserves Branch History? | When Used |
|---|---|---|---|
| Fast-Forward | No | No (linear history) | Feature ahead of base with no divergence |
| Three-Way (Merge Commit) | Yes | Yes (explicit merge point) | Branches have diverged |

---

## 5. Fast-Forward Merge

### When It Happens

Fast-forward is possible when the target branch has not diverged from the source branch — i.e., the target is a **direct ancestor** of the source.

```
BEFORE merge (main has not moved since feature branched off):
  main:    A ← B ← C
                     ↑
  feature:           C ← D ← E
                               ↑ HEAD

AFTER git merge feature (fast-forward):
  main:    A ← B ← C ← D ← E
                               ↑ HEAD → main
  (feature branch pointer also still at E)
```

Git simply moves the `main` pointer forward to `E`. No new commit is created.

### Commands

```bash
git switch main
git merge feature/payment
# Updating abc123..def456
# Fast-forward
#  src/Payment.java | 15 +++++++++++++++
```

### Preventing Fast-Forward (Preserving History)

```bash
# Always create merge commit even if FF is possible
git merge --no-ff feature/payment
```

**When to use `--no-ff`:**
- Team policy requires explicit merge commits for every feature
- You want branch history visible in `git log --graph`
- Regulated environments requiring audit trails of feature integrations

---

## 6. Three-Way Merge

### When It Happens

When both the base branch and the feature branch have new commits since the branch point.

```
BEFORE merge:
  main:    A ← B ← C ← D
                    ↑
  feature:          C ← E ← F
                              ↑ HEAD

(C is the merge base — common ancestor)

AFTER git merge feature:
  main:    A ← B ← C ← D ← G
                    ↑       ↑
  feature:          C ← E ← F
  (G is a merge commit with 2 parents: D and F)
```

Git performs a **three-way merge**:
- **Ours** (D) — tip of main
- **Theirs** (F) — tip of feature
- **Base** (C) — common ancestor

Git compares changes from Base→Ours and Base→Theirs:
- Non-overlapping changes → merged automatically
- Overlapping changes to same lines → **CONFLICT** (requires manual resolution)

### Creating a Merge Commit

```bash
git switch main
git merge feature/payment
# Merge made by the 'recursive' strategy.
#  src/Payment.java | 20 ++++++++++++++++++++
#
# git log shows:
# *   G (HEAD → main) Merge branch 'feature/payment'
# |\
# | * F add payment validation
# | * E add retry logic
# * | D update config
# |/
# * C initial payment setup
```

### Merge Strategies

```bash
git merge -s recursive feature/payment      # default (two heads)
git merge -s octopus feature-a feature-b    # merge multiple branches at once
git merge -s ours feature/payment           # always take ours (ignore theirs)
git merge -s subtree                        # for merging subtrees

# Recursive strategy options
git merge -X ours feature/payment           # prefer our side in conflicts
git merge -X theirs feature/payment         # prefer their side in conflicts
git merge -X patience feature/payment       # better diff for large changes
```

---

## 7. Merge Conflicts — Resolution Deep Dive

### What Causes a Conflict

Git cannot automatically merge when:
1. Both branches modified the **same lines** of the same file
2. One branch deleted a file while the other modified it
3. Both branches added a file with the same name but different content

### Conflict Markers

```java
<<<<<<< HEAD                           ← start of OUR version (current branch)
public void processPayment(double amount) {
    if (amount <= 0) throw new IllegalArgumentException("Invalid amount");
    gateway.process(amount);
=======                                ← separator
public void processPayment(double amount) {
    validate(amount);
    gateway.processWithRetry(amount, 3);
>>>>>>> feature/payment-retry          ← end of THEIR version (branch being merged)
```

The three sections:
- `<<<<<<< HEAD` to `=======` → your current branch's version
- `=======` to `>>>>>>> branch-name` → the incoming branch's version

### Step-by-Step Conflict Resolution

```bash
# Step 1: Start merge (conflict occurs)
git merge feature/payment-retry
# CONFLICT (content): Merge conflict in src/Payment.java
# Automatic merge failed; fix conflicts and then commit the result.

# Step 2: See which files have conflicts
git status
# both modified: src/main/java/com/payment/Payment.java

# Step 3: Open the file and resolve manually
# (edit the file — remove conflict markers, keep the correct code)

# Step 4: Stage the resolved file
git add src/main/java/com/payment/Payment.java

# Step 5: Complete the merge
git commit
# (Git auto-generates merge commit message)

# OR abort the merge entirely and go back to pre-merge state
git merge --abort
```

### Using a Merge Tool

```bash
# Configure merge tool
git config --global merge.tool vimdiff    # or: meld, kdiff3, opendiff, vscode

# Launch merge tool for all conflicts
git mergetool

# VS Code as merge tool
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'
```

### Conflict Resolution Strategies

```bash
# Accept all OURS (current branch wins for all conflicts)
git checkout --ours   src/Payment.java
git add src/Payment.java

# Accept all THEIRS (incoming branch wins for all conflicts)
git checkout --theirs src/Payment.java
git add src/Payment.java

# View the three versions side by side
git show :1:src/Payment.java    # base (common ancestor)
git show :2:src/Payment.java    # ours
git show :3:src/Payment.java    # theirs
```

### Preventing Conflicts Proactively

1. **Sync frequently** — `git pull --rebase` from main before opening a PR
2. **Small PRs** — fewer lines changed = fewer conflicts
3. **Clear file ownership** — reduce overlapping edits via architecture
4. **Feature flags** — merge partial features behind flags instead of long-lived branches
5. **Communicate** — tell teammates when you're editing shared files

---

## 8. Rebase vs Merge

### The Core Difference

```
MERGE: preserves exact history (non-linear, explicit merge commit)
REBASE: rewrites history to appear linear (replays commits on new base)
```

### Rebase Visualised

```
BEFORE rebase:
  main:    A ← B ← C ← D
                    ↑
  feature:          C ← E ← F

git switch feature
git rebase main

AFTER rebase:
  main:    A ← B ← C ← D
                              ↑
  feature:                    D ← E' ← F'
  (E' and F' are NEW commits — same changes, different SHAs, parent is D)
```

### Merge vs Rebase Comparison

| Aspect | Merge | Rebase |
|---|---|---|
| History | Non-linear (DAG) | Linear |
| Merge commit | Created | Not created |
| Commit SHAs | Preserved | Changed (new commits) |
| Safe on shared branches | ✅ Yes | ❌ No (rewrites history) |
| Conflict points | Once (at merge) | Per commit being replayed |
| `git log` readability | Cluttered with merge commits | Clean, linear |
| Auditability | Full branch history visible | Branch history lost |
| Best for | `main`/`develop` integration | Feature branch cleanup before PR |

### The Golden Rule of Rebase

> **Never rebase commits that have been pushed to a shared/public branch.**

When you rebase, you rewrite commit SHAs. If your teammates have pulled those commits, they now have a different history than you. Reconciling this requires `git push --force` and is a painful team experience.

### `git rebase` Commands

```bash
# Rebase current branch onto main
git rebase main

# Rebase with specific upstream
git rebase origin/main

# Continue after resolving a conflict during rebase
git rebase --continue

# Skip the current commit during rebase
git rebase --skip

# Abort rebase and return to pre-rebase state
git rebase --abort

# Pull with rebase instead of merge (keeps history clean)
git pull --rebase
git pull --rebase=interactive   # rebase interactively
```

---

## 9. Interactive Rebase

### What It Does

`git rebase -i` lets you **rewrite your local branch history** before sharing it. You can:
- Reword commit messages
- Squash multiple commits into one
- Reorder commits
- Edit (stop and amend) commits
- Drop commits entirely
- Split commits

### Usage

```bash
# Interactively rebase last N commits
git rebase -i HEAD~3      # last 3 commits
git rebase -i HEAD~5      # last 5 commits

# Rebase from the point where feature branched off main
git rebase -i main
```

### The Interactive Editor

```
pick abc123 feat: add payment validation
pick def456 fix typo in payment
pick ghi789 add unit tests for payment

# Rebase abc123..ghi789 onto xyz (3 commands)
#
# Commands:
# p, pick   = use commit as-is
# r, reword = use commit, but edit the commit message
# e, edit   = use commit, but stop for amending
# s, squash = melt commit into previous commit (keeps both messages)
# f, fixup  = melt commit into previous commit (discard this message)
# d, drop   = remove commit entirely
# b, break  = stop here (continue with 'git rebase --continue')
```

### Common Interactive Rebase Workflows

#### Squash Feature Commits Before PR

```
BEFORE (messy local history):
  abc123  WIP: started payment
  def456  WIP: more payment stuff
  ghi789  fix: oops wrong variable
  jkl012  add tests finally
  mno345  fix: tests were failing

GOAL: single clean commit for PR

git rebase -i HEAD~5

# Change to:
pick abc123 WIP: started payment
f    def456 WIP: more payment stuff      ← fixup (discard message)
f    ghi789 fix: oops wrong variable     ← fixup
f    jkl012 add tests finally            ← fixup
f    mno345 fix: tests were failing      ← fixup

# Result: one commit with message "WIP: started payment"
# Then reword it:
git commit --amend -m "feat: add UPI payment processing with retry logic"
```

#### Reorder Commits

```
# Change order in the editor:
pick ghi789 add unit tests        ← moved up
pick abc123 feat: add feature
pick def456 docs: update readme
```

#### Split a Commit

```
# Mark a commit for editing:
edit abc123 feat: add payment and update config

# Git stops at this commit, HEAD is at abc123
# Undo the commit (keep changes in working dir)
git reset HEAD~1

# Now stage and commit in pieces
git add src/Payment.java
git commit -m "feat: add payment processing"

git add config/application.properties
git commit -m "chore: update payment gateway config"

# Continue rebase
git rebase --continue
```

---

## 10. Cherry-Pick

### What It Does

`git cherry-pick` copies one or more commits from another branch onto the current branch, creating **new commits** with the same changes but different SHAs.

```
main:    A ← B ← C ← D
                            ↑ (want commit E's changes here)
feature:     B ← E ← F
                  ↑ cherry-pick this

AFTER git cherry-pick E:
main:    A ← B ← C ← D ← E'
                              ↑ (E' = same changes as E, different SHA)
feature:     B ← E ← F (unchanged)
```

### Usage

```bash
# Cherry-pick single commit
git cherry-pick abc123

# Cherry-pick range of commits (exclusive of start)
git cherry-pick abc123..def456

# Cherry-pick range (inclusive of start)
git cherry-pick abc123^..def456

# Cherry-pick without auto-committing (stage only)
git cherry-pick -n abc123
git cherry-pick --no-commit abc123

# Cherry-pick and edit commit message
git cherry-pick -e abc123

# Continue after conflict
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort
```

### Real-World Use Cases

```bash
# Hotfix: fix was applied to develop, need it in main
git switch main
git cherry-pick <fix-commit-sha>
git push origin main

# Port a feature to multiple release branches
git switch release/v2.0
git cherry-pick <feature-commit-sha>

git switch release/v1.9
git cherry-pick <feature-commit-sha>
```

> **Interview Insight:** Cherry-pick is useful for isolated fixes but should not replace proper branching strategy. Overuse leads to divergent histories that become hard to maintain. If you find yourself cherry-picking often, consider whether your branching strategy needs revision.

---

## 11. Branching Strategies

### A. Git Flow

Best for: versioned software with scheduled releases (libraries, apps with release cycles)

```
main          ─────────────────────────────────────────────── (production)
                 ↑ merge                             ↑ merge
hotfix/       ──────                                ──────
                                    ↑ merge
release/v1.2               ─────────────────
                              ↑ merge                             
develop       ──────────────────────────────────────────────── (integration)
                 ↑ merge             ↑ merge
feature/A     ──────────
feature/B                        ────────────────
```

**Branches:**
- `main` — production-ready, tagged with version numbers
- `develop` — integration branch; all features merge here
- `feature/*` — individual features, branch from `develop`
- `release/*` — release preparation (bug fixes only), branch from `develop`
- `hotfix/*` — emergency production fixes, branch from `main`

**Pros:** Clear separation, supports multiple versions  
**Cons:** Complex, slow for fast-moving teams, merge hell

### B. GitHub Flow

Best for: web apps, SaaS, continuous deployment teams

```
main    ─────────────────────────────────────── (always deployable)
            ↑ PR+merge    ↑ PR+merge    ↑ PR+merge
feature/A ──────
feature/B         ────────────
feature/C                          ──────────────
```

**Rules:**
1. Anything in `main` is deployable
2. Create branch from `main` for every change
3. Open PR early, discuss, iterate
4. Deploy to staging from branch before merging
5. Merge to `main` only after review + CI passes
6. Deploy immediately after merge

**Pros:** Simple, fast, suits CI/CD  
**Cons:** Needs strong CI/CD, not suited for versioned releases

### C. GitLab Flow (Environment Branches)

```
main        ──────────────────────────────────
                ↓ auto-deploy
staging     ──────────────────────────────────
                ↓ manual promote
production  ──────────────────────────────────
```

Feature branches → main → auto-deploy to staging → manual promotion to production.

### D. Trunk-Based Development

Best for: high-velocity teams, experienced engineers, strong CI/CD

```
main (trunk)  ─────────────────────────────── (very short-lived branches)
                ↑↑ merge frequently (daily)
feature/A   ────
feature/B       ────
feature/C           ────
```

**Rules:**
- Feature branches live for < 1-2 days
- All work hides behind **feature flags** until ready
- CI must pass before any merge
- Team commits directly to trunk (experienced teams)

**Pros:** No merge conflicts, fastest integration, simplest flow  
**Cons:** Requires maturity, feature flags, and rock-solid CI/CD

### Strategy Selection Guide

```
Is your product versioned with scheduled releases?
  Yes → Git Flow

Do you practice continuous deployment?
  Yes, team < 20 devs → GitHub Flow
  Yes, team > 20 devs → Trunk-Based Development

Do you need environment promotion gates?
  Yes → GitLab Flow

Is this open source with external contributors?
  Yes → Fork + PR workflow (GitHub/GitLab Flow variant)
```

---

## 12. Remote Branches & Tracking

### Remote-Tracking Branches

When you clone or fetch, Git creates **remote-tracking branches** — read-only local references that mirror the remote state:

```
.git/refs/remotes/origin/
├── main          → abc123...  (state of origin/main at last fetch)
├── develop       → def456...
└── feature/pay   → ghi789...
```

These update ONLY when you `git fetch` or `git pull`.

### Tracking Relationships

A local branch can **track** a remote branch. This enables:
- `git push` without specifying remote/branch
- `git pull` without specifying remote/branch
- `git status` shows ahead/behind counts

```bash
# Set tracking when pushing for first time
git push -u origin feature/payment
git push --set-upstream origin feature/payment    # same thing

# Set tracking on existing branch
git branch --set-upstream-to=origin/main main
git branch -u origin/feature/payment feature/payment

# View tracking configuration
git branch -vv
# * feature/payment  abc123 [origin/feature/payment: ahead 2, behind 1] add retry
#   main             def456 [origin/main] latest main

# Remove tracking
git branch --unset-upstream
```

### Fetch vs Pull vs Push

```bash
# FETCH: download remote changes, do NOT merge
git fetch origin                    # fetch all branches
git fetch origin main               # fetch specific branch
git fetch --all                     # fetch all remotes
git fetch --prune                   # fetch + remove stale remote-tracking refs

# PULL: fetch + merge (or rebase) into current branch
git pull                            # fetch + merge (default)
git pull --rebase                   # fetch + rebase (cleaner history)
git pull origin main                # explicit remote + branch

# PUSH: upload local commits to remote
git push origin main                # push to specific branch
git push                            # push to tracked upstream
git push --force-with-lease         # force push (safer than --force)
git push --force                    # force push ⚠️ DANGEROUS on shared branches
git push --tags                     # push all tags
git push origin v1.0.0              # push specific tag
```

### Force Push Safety

```bash
# --force-with-lease: fails if remote has commits you haven't fetched
# This prevents accidentally overwriting a teammate's push
git push --force-with-lease origin feature/payment

# --force: overwrites remote unconditionally ⚠️
# Only use on your own private branches
git push --force origin feature/payment
```

---

## 13. Interview Questions & Model Answers

### Q1: "What is the difference between merge and rebase?"

**Model Answer:**
> Both integrate changes from one branch into another, but they work differently. Merge creates a new "merge commit" with two parents, preserving the exact history of both branches — the branch topology remains visible in `git log --graph`. Rebase replays commits from your branch onto the tip of the target branch, rewriting their SHAs, resulting in a linear history as if the feature was developed sequentially after the base. Merge is safe on shared branches because it doesn't rewrite history. Rebase should only be used on local/private branches because rewriting SHAs breaks the history for anyone who already has those commits. My team policy is: rebase to clean up feature branch history before opening a PR, merge (via PR) to integrate into main.

### Q2: "What is a fast-forward merge and when does it occur?"

**Model Answer:**
> A fast-forward merge occurs when the branch being merged into has no new commits since the feature branch diverged — i.e., the base branch tip is a direct ancestor of the feature branch tip. In this case, Git simply moves the base branch pointer forward to the feature branch tip without creating a merge commit. No merge commit is needed because there's no divergence to reconcile. You can prevent fast-forward with `git merge --no-ff` to always create a merge commit, which some teams prefer for auditability. Fast-forward is the default when it's possible.

### Q3: "How do you resolve a merge conflict in Git?"

**Model Answer:**
> When a merge conflict occurs, Git marks the conflicting sections in the file with `<<<<<<<`, `=======`, and `>>>>>>>` markers. My process is: first, run `git status` to identify all conflicting files. Then open each file and understand why both sides changed the same section — usually this requires understanding the intent of both changes. I edit the file to the correct final state (which might be one side, the other, or a combination), then remove all conflict markers. Once resolved, I `git add` the file to mark it resolved. After all conflicts are resolved, `git commit` completes the merge. For complex conflicts, I use `git mergetool` with a visual diff tool. I also use `git show :1:file` to see the common ancestor, `:2:file` for ours, `:3:file` for theirs. Post-resolution, I always run the test suite before committing.

### Q4: "What branching strategy does your team use and why?"

**Model Answer (GitHub Flow example):**
> We use GitHub Flow because we practice continuous deployment. Every feature and bug fix gets its own short-lived branch off main. We open PRs early — sometimes as draft PRs — so the team can provide feedback before the work is done. Our CI pipeline runs tests, lint, and security scans on every push. We deploy the branch to a staging environment before merging. Once approved and CI passes, we merge via squash-merge to keep main's history clean and linear. We deploy to production immediately after merge. This works well because main is always in a deployable state — there's no manual release process. For a library or versioned product, I'd consider Git Flow instead to manage multiple release lines.

### Q5: "What is `git cherry-pick` and when would you use it?"

**Model Answer:**
> `git cherry-pick <sha>` copies a specific commit's changes onto the current branch, creating a new commit with the same diff but a different SHA. I use it in two main scenarios: (1) Hotfixes — when a bug fix is committed on a development branch and needs to be backported to a production release branch immediately, rather than merging the entire feature branch. (2) Selective feature porting — when specific functionality from a long-running feature branch needs to land in main independently. However, I try not to overuse cherry-pick because it creates duplicate commits (same changes, different SHAs) that can cause confusion and conflicts later. If I find myself cherry-picking regularly, it usually signals a branching strategy problem.

### Q6: "What is interactive rebase and what can you do with it?"

**Model Answer:**
> `git rebase -i HEAD~N` opens an interactive editor showing the last N commits with action prefixes. You can: `pick` (keep as-is), `reword` (change message), `squash` (combine with previous commit, keeping both messages), `fixup` (combine with previous, discarding this message), `edit` (stop and amend), `drop` (delete the commit), or reorder commits by reordering lines. I use it most often before opening a PR to squash "WIP" commits and fix typos into a clean, atomic commit series — making the PR easier to review and maintaining a readable project history. The golden rule: only use interactive rebase on commits that haven't been pushed to a shared branch.

---

## 14. Quick Revision Cheatsheet

```bash
# ─── BRANCH MANAGEMENT ────────────────────────────────
git branch                        # list local branches
git branch -a                     # list all branches
git branch -vv                    # with tracking + ahead/behind
git switch -c feature/name        # create + switch ⭐
git switch main                   # switch to existing branch
git switch -                      # switch to previous branch
git branch -d feature/name        # delete (safe)
git branch -D feature/name        # delete (force)
git branch -m old-name new-name   # rename
git branch --merged               # branches merged into current

# ─── MERGE ────────────────────────────────────────────
git merge feature/payment         # merge into current branch
git merge --no-ff feature/payment # always create merge commit
git merge --squash feature/payment # squash all commits into one staged change
git merge --abort                 # abort in-progress merge
git mergetool                     # launch visual merge tool

# ─── REBASE ───────────────────────────────────────────
git rebase main                   # rebase current branch onto main
git rebase -i HEAD~3              # interactive rebase last 3 commits ⭐
git rebase --continue             # continue after resolving conflict
git rebase --abort                # abort rebase
git pull --rebase                 # pull with rebase instead of merge

# ─── CHERRY-PICK ──────────────────────────────────────
git cherry-pick abc123            # copy single commit
git cherry-pick abc123..def456    # copy range
git cherry-pick -n abc123         # stage only (no commit)
git cherry-pick --abort           # abort

# ─── REMOTE ───────────────────────────────────────────
git fetch --prune                 # fetch + clean stale refs
git pull --rebase                 # fetch + rebase
git push -u origin feature/name   # push + set tracking
git push --force-with-lease       # force push (safer)
git push origin --delete branch   # delete remote branch
```

---

## 15. Real-World Workflows

### Workflow A: PR-Ready Feature Branch

```bash
# Start feature
git switch main
git pull --rebase
git switch -c feature/payment-retry

# Develop with frequent small commits
git commit -m "WIP: add retry config"
git commit -m "WIP: implement retry loop"
git commit -m "fix: off-by-one in retry count"
git commit -m "add unit tests"
git commit -m "fix: tests passing now"

# Before opening PR: clean up history
git rebase -i main
# Squash all WIP commits:
# pick  WIP: add retry config
# f     WIP: implement retry loop
# f     fix: off-by-one in retry count
# f     add unit tests
# f     fix: tests passing now
# Reword to: "feat(payment): add exponential backoff retry with unit tests"

# Sync with main
git fetch origin
git rebase origin/main

# Push
git push -u origin feature/payment-retry
# → Open PR on GitHub
```

### Workflow B: Hotfix on Production

```bash
# Production is on main at tag v2.1.0
git switch main
git pull origin main

git switch -c hotfix/null-pointer-payment

# Fix the bug
vim src/Payment.java
git add src/Payment.java
git commit -m "fix(payment): prevent NPE when amount is null

Amount validation was missing when the currency field was absent.
Adds null check before processing.

Fixes: #PROD-4521"

# Run tests
mvn test

# Merge to main
git switch main
git merge --no-ff hotfix/null-pointer-payment
git tag -a v2.1.1 -m "Hotfix: prevent NPE in payment processing"
git push origin main --tags

# Backport to develop
git switch develop
git cherry-pick <fix-commit-sha>
git push origin develop

# Clean up
git branch -d hotfix/null-pointer-payment
git push origin --delete hotfix/null-pointer-payment
```

### Workflow C: Keeping Feature Branch Up-to-Date

```bash
# Long-lived feature branch (avoid long-lived branches, but sometimes necessary)
git switch feature/big-redesign

# Regularly sync with main to minimise divergence
git fetch origin
git rebase origin/main       # preferred: linear history

# If conflicts:
# resolve each conflict per commit
# git add <resolved-file>
# git rebase --continue

# After rebase, force push (feature branch is yours)
git push --force-with-lease origin feature/big-redesign
```

---

## 16. Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| "Not a git repository" | Wrong directory | `cd` to project root |
| Merge conflict won't resolve | Conflict markers still in file | Remove ALL `<<<<<<<` `=======` `>>>>>>>` markers |
| "Your branch has diverged" | Local and remote have different commits | `git pull --rebase` or `git fetch` + `git rebase origin/main` |
| Accidentally merged wrong branch | Merge commit created | `git revert -m 1 <merge-sha>` (safe) or `git reset --hard HEAD~1` (local only) |
| Branch deleted with unmerged work | Used `git branch -D` | `git reflog` → find SHA → `git switch -c recovered <sha>` |
| Rebase conflicts on every commit | Feature branch very diverged | Consider `git merge` instead, or sync more frequently |
| `git push` rejected after rebase | History rewritten | `git push --force-with-lease` (only on your own branches) |
| Detached HEAD state | Checked out a commit SHA directly | `git switch -c new-branch` (to keep work) or `git switch main` (to discard) |
| Wrong commit message already pushed | Pushed to shared branch | `git revert` old commit; add new commit with correct message |
| Squash gone wrong in rebase | All commits squashed accidentally | `git reflog` → find pre-rebase SHA → `git reset --hard <sha>` |

---

> **Previous:** [03 · Git File Lifecycle](./03_Git_File_Lifecycle.md)  
> **Next:** [05 · Remote Repositories & Collaboration](./05_Remote_Repositories_and_Collaboration.md)
