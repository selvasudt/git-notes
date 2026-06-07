# 04 · Git Branching & Merging

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead  
> **Series:** Git Knowledge Base — File 4 of 12

---

## Table of Contents
1. [git branch](#1-git-branch)
2. [git switch / git checkout](#2-git-switch--git-checkout)
3. [git merge (Fast-Forward)](#3-git-merge-fast-forward)
4. [git merge (Three-Way)](#4-git-merge-three-way)
5. [Merge Conflict Resolution](#5-merge-conflict-resolution)
6. [git rebase](#6-git-rebase)
7. [git rebase -i (Interactive)](#7-git-rebase--i-interactive)
8. [git cherry-pick](#8-git-cherry-pick)
9. [Branching Strategies](#9-branching-strategies)
10. [Interview Questions & Answers](#10-interview-questions--answers)
11. [Quick Revision Cheatsheet](#11-quick-revision-cheatsheet)

---

# 1. git branch

## What is it?
`git branch` creates, lists, renames, and deletes branches. A **branch** is a lightweight, movable pointer to a commit — just a 41-byte file in `.git/refs/heads/` containing a SHA-1 hash.

## Why It Matters
- Branches enable isolated parallel development — each feature, bug fix, or experiment gets its own branch
- Creating a branch is O(1): one file write, zero file copies, zero network operations
- Deleting a branch does NOT delete commits — only removes the pointer

## Internal Working
```
A branch = a text file in .git/refs/heads/

cat .git/refs/heads/main
→ 32686eaaf60e5e23498b286d23a28a3387241e38   (commit SHA)

Creating a branch = writing a new file with the same SHA
Switching branches = updating .git/HEAD to point to the new branch file
Making a commit = updating the current branch file to the new commit SHA

.git/refs/heads/
  main           → c1a2b3c4...
  feature/pay    → d5e6f7a8...
  hotfix/null    → b9c0d1e2...
```

## Command Explanation

### Syntax
```bash
git branch [options] [<branchname>] [<start-point>]
```

```bash
# List local branches (* = current)
git branch

# List with last commit SHA + message
git branch -v

# List with upstream tracking + ahead/behind info
git branch -vv

# List remote-tracking branches
git branch -r

# List ALL (local + remote-tracking)
git branch -a

# Create branch (stay on current branch)
git branch feature/payment

# Create branch from specific commit/tag
git branch hotfix/v2.1.1 v2.1.0          # from tag
git branch feature/pay abc123            # from commit SHA

# Delete merged branch (safe)
git branch -d feature/payment

# Force delete (even if unmerged)
git branch -D feature/payment

# Rename current branch
git branch -m new-name

# Rename specific branch
git branch -m old-name new-name

# List branches merged into current branch (safe to delete)
git branch --merged

# List branches NOT yet merged (do NOT delete these carelessly)
git branch --no-merged
```

---

# 2. git switch / git checkout

## What is it?
`git switch` (Git 2.23+) changes the currently active branch. `git checkout` is the older equivalent that also handles file restoration (its dual-purpose nature caused confusion, which is why `git switch` and `git restore` were introduced as clearer replacements).

## Why It Matters
- You must be on the correct branch before making commits
- Switching branches updates the Working Directory to match the target branch's commit
- `git switch -c` is the most frequent branching command in daily development

## Internal Working
```
git switch feature/payment:
  1. Updates .git/HEAD → ref: refs/heads/feature/payment
  2. Reads the tree of the feature/payment commit
  3. Updates Working Directory files to match that tree
  4. Updates .git/index to match that tree

If Working Directory has uncommitted changes that conflict:
  → Git REFUSES to switch (protects your work)
  → You must: git stash, git commit, or git restore first

Detached HEAD (git switch <sha> or git checkout <tag>):
  .git/HEAD → abc123def...  (direct SHA, not a branch ref)
  New commits here are NOT on any branch → will be GC'd
```

## Command Explanation

### Syntax
```bash
git switch [options] <branch>
git switch -c <new-branch> [<start-point>]
```

```bash
# Switch to existing branch
git switch main
git switch develop

# Create AND switch in one step ⭐ (most common)
git switch -c feature/payment
git switch --create feature/payment   # explicit form

# Create from specific starting point
git switch -c hotfix/v2.1.1 v2.1.0         # from tag
git switch -c feature/pay abc123def        # from commit SHA
git switch -c local-branch origin/remote-branch  # track remote branch

# Return to previous branch (like cd -)
git switch -

# Older syntax (still valid, widely used)
git checkout -b feature/payment         # create + switch
git checkout main                       # switch to existing
git checkout -b hotfix origin/hotfix    # track remote

# Delete remote branch
git push origin --delete feature/payment

# Prune stale remote-tracking refs
git fetch --prune
```

---

# 3. git merge (Fast-Forward)

## What is it?
A **fast-forward merge** occurs when the target branch has no new commits since the feature branch diverged — the target is a direct ancestor of the feature branch tip. Git simply moves the target branch pointer forward without creating a new merge commit.

## Why It Matters
- Results in a clean linear history — no merge commit clutter
- Possible only when no divergence exists between branches
- `--no-ff` flag forces a merge commit even when FF is possible (team policy choice)

## Internal Working
```
BEFORE (main has not moved since feature branched off):
  main:    A ← B ← C
                     ↑
  feature:           C ← D ← E
                               ↑ HEAD

git switch main
git merge feature/payment

AFTER fast-forward:
  main:    A ← B ← C ← D ← E    ← main pointer moved forward
                               ↑ HEAD → main
  (No new commit object created — just pointer update)
```

## Command Explanation

### Syntax
```bash
git merge [options] <branch>
```

```bash
git switch main
git merge feature/payment
# Output: Fast-forward
#  src/Payment.java | 15 +++++++++++++++

# Prevent fast-forward — always create merge commit
git merge --no-ff feature/payment

# Preview what would be merged (dry run)
git merge --no-commit --no-ff feature/payment
git merge --abort     # abort the preview

# Verify result
git log --oneline --graph -5
```

---

# 4. git merge (Three-Way)

## What is it?
A **three-way merge** occurs when both the base branch and feature branch have new commits since the branch point. Git finds the common ancestor (merge base), compares both branch tips against it, and produces a new **merge commit** with two parents.

## Why It Matters
- Preserves complete history of both branches — branch topology visible in `git log --graph`
- Creates an explicit merge point — clear in audit trails and release tracking
- Non-overlapping changes are merged automatically; overlapping changes create conflicts

## Internal Working
```
BEFORE:
  main:    A ← B ← C ← D
                    ↑
  feature:          C ← E ← F    ← C is merge base

Three-way merge uses:
  Base:   C (common ancestor — LCA in DAG)
  Ours:   D (main tip)
  Theirs: F (feature tip)

For each file:
  Changed in Ours only  → take Ours change
  Changed in Theirs only → take Theirs change
  Changed in BOTH        → CONFLICT (must resolve manually)

AFTER:
  A ← B ← C ← D ← G    ← G is merge commit with 2 parents: D and F
                    ↑
              C ← E ← F
```

## Command Explanation

### Syntax
```bash
git merge [options] <branch>
```

```bash
git switch main
git merge feature/payment
# Merge made by the 'recursive' strategy.

# Show the merge commit
git log --oneline --graph -5

# Merge strategies
git merge -s recursive feature/payment    # default (two heads)
git merge -s octopus feat-a feat-b feat-c # merge multiple branches at once
git merge -s ours feature/payment         # always keep our version (ignore theirs)

# Recursive strategy options
git merge -X ours feature/payment         # prefer our side in all conflicts
git merge -X theirs feature/payment       # prefer their side in all conflicts

# Squash: merge all feature commits into one staged change (not committed yet)
git merge --squash feature/payment
git commit -m "feat: complete payment module"   # single commit for all feature work

# Abort in-progress merge
git merge --abort
```

---

# 5. Merge Conflict Resolution

## What is it?
A **merge conflict** occurs when Git cannot automatically determine which change to keep because both branches modified the same lines of the same file. Git inserts conflict markers and pauses the merge, requiring manual resolution.

## Why It Matters
- Conflicts are unavoidable in team development — knowing HOW to resolve them correctly is critical
- Incorrect resolution (e.g., accidentally discarding a teammate's work) causes bugs
- Understanding the three-version model (base, ours, theirs) enables principled resolution

## Internal Working
```
During three-way merge, when both branches changed same lines:
  .git/index stores THREE versions of the conflicted file:
    Stage 1: base (common ancestor) — git show :1:file
    Stage 2: ours (current branch)  — git show :2:file
    Stage 3: theirs (merging branch)— git show :3:file

  Git writes conflict markers to Working Directory:
    <<<<<<< HEAD                  ← our version start
    public void process(double amount) {
        if (amount <= 0) throw new IllegalArgumentException();
    =======                       ← separator
    public void process(double amount) {
        validate(amount);
        processWithRetry(amount, 3);
    >>>>>>> feature/payment-retry ← their version end
```

## Command Explanation

### Syntax
```bash
git merge --abort                    # abort entire merge
git mergetool                        # launch visual merge tool
git checkout --ours <file>           # take our version entirely
git checkout --theirs <file>         # take their version entirely
git add <file>                       # mark conflict as resolved
git merge --continue                 # complete merge after resolving
```

```bash
# Step 1: See which files conflict
git status
# both modified: src/Payment.java

# Step 2: Inspect all three versions
git show :1:src/Payment.java    # base (common ancestor)
git show :2:src/Payment.java    # ours (HEAD / current branch)
git show :3:src/Payment.java    # theirs (branch being merged)

# Step 3a: Resolve manually
# Open file, edit away all conflict markers, write correct final version
# Ensure NO <<<<<<< ======= >>>>>>> remain
grep -r "<<<<<<" src/   # should be empty after resolution

# Step 3b: Or take one side entirely
git checkout --ours   src/Payment.java   # our version wins
git checkout --theirs src/Payment.java   # their version wins

# Step 3c: Or use merge tool
git config --global merge.tool vscode
git mergetool                            # opens VS Code merge editor

# Step 4: Mark as resolved
git add src/Payment.java

# Step 5: Complete merge
git commit
# (Git auto-generates: "Merge branch 'feature/payment-retry'")

# Verify no conflict markers remain before committing
git diff --check
```

---

# 6. git rebase

## What is it?
`git rebase` moves or "replays" commits from one branch onto a new base commit. Instead of creating a merge commit, it rewrites the feature branch commits so they appear as if they were created after the latest base branch commits — producing a **linear history**.

## Why It Matters
- Produces clean, linear git log without merge commit noise
- Makes `git bisect` and `git blame` more effective (linear history)
- **Golden Rule:** NEVER rebase commits already pushed to a shared branch — it rewrites SHAs and breaks teammates' history

## Internal Working
```
BEFORE rebase:
  main:    A ← B ← C ← D
                    ↑
  feature:          C ← E ← F    ← C is the old base

git switch feature
git rebase main

PROCESS (per commit):
  1. Find merge base: C
  2. Save E and F as patches
  3. Reset feature to D (main's tip)
  4. Apply E's patch → creates E' (new SHA, same changes, parent=D)
  5. Apply F's patch → creates F' (new SHA, same changes, parent=E')

AFTER rebase:
  main:    A ← B ← C ← D
                           ↑
  feature:                 D ← E' ← F'   ← NEW commits (different SHAs!)

E' and F' have identical code changes to E and F
but different SHAs because their parent (D ≠ C) changed
```

## Command Explanation

### Syntax
```bash
git rebase [options] [<upstream>] [<branch>]
```

```bash
# Rebase current branch onto main
git rebase main

# Rebase onto specific remote branch
git rebase origin/main

# Continue after resolving a conflict (during rebase)
git rebase --continue

# Skip the current commit (take neither side)
git rebase --skip

# Abort rebase — restore pre-rebase state
git rebase --abort

# Pull with rebase instead of merge (keeps history clean)
git pull --rebase
git pull --rebase=interactive

# After rebasing a pushed branch (own branch only):
git push --force-with-lease origin feature/payment

# Rebase vs Merge decision:
# Feature branch cleanup before PR → rebase (linear history)
# Integrating into main/develop    → merge via PR (preserves history)
```

---

# 7. git rebase -i (Interactive)

## What is it?
`git rebase -i` (interactive rebase) opens an editor showing the last N commits and lets you rewrite local branch history: squash WIP commits, reorder, reword messages, edit, or drop commits entirely.

## Why It Matters
- The most powerful history-cleanup tool — turn messy "WIP fix typo asdf" commits into clean, atomic commits before opening a PR
- Should only be used on local/unpushed commits
- Enables professional-grade commit history that teammates and future-you will appreciate

## Internal Working
```
git rebase -i HEAD~4 opens an editor with:
  pick abc123 WIP: started payment feature
  pick def456 more payment stuff
  pick ghi789 fix typo
  pick jkl012 add tests

Actions available:
  pick   = use commit as-is
  reword = use commit, edit message
  edit   = use commit, stop for amending
  squash = melt into previous commit (keep both messages)
  fixup  = melt into previous commit (discard this message)
  drop   = delete commit entirely
  (reorder lines = reorder commits)
```

## Command Explanation

### Syntax
```bash
git rebase -i <base>
git rebase -i HEAD~<n>
```

```bash
# Interactively rebase last 4 commits
git rebase -i HEAD~4

# Rebase from where feature diverged from main
git rebase -i main

# Common: squash all WIP commits into one clean commit
# Change editor to:
#   pick abc123 WIP: started payment
#   f    def456 more payment stuff    ← fixup (discard msg)
#   f    ghi789 fix typo              ← fixup
#   f    jkl012 add tests             ← fixup
# Then amend the remaining commit message:
git commit --amend -m "feat(payment): add retry logic with unit tests"

# Reorder commits:
# Move ghi789 (bug fix) before abc123 (feature) in the editor

# Split a commit:
# Mark commit as 'edit', then when Git stops:
git reset HEAD~1           # undo the commit, keep changes unstaged
git add src/Payment.java
git commit -m "feat: add payment processing"
git add config/
git commit -m "chore: add payment config"
git rebase --continue

# After interactive rebase on pushed branch:
git push --force-with-lease origin feature/payment
```

---

# 8. git cherry-pick

## What is it?
`git cherry-pick` copies one or more specific commits from any branch and applies them onto the current branch, creating **new commits** with the same code changes but different SHAs.

## Why It Matters
- Essential for applying a hotfix to multiple release branches without merging entire feature branches
- Enables backporting specific fixes to older version lines
- Creates duplicate commits (same changes, different SHAs) — overuse leads to confusing history

## Internal Working
```
main:    A ← B ← C ← D
                           ↑ (want E's changes here)
feature:     B ← E ← F
                  ↑ cherry-pick this commit

git switch main
git cherry-pick E

Creates E' on main:
  E'.changes = E.changes (identical diff)
  E'.SHA     ≠ E.SHA     (different parent: D ≠ B)
  E'.parent  = D

main:    A ← B ← C ← D ← E'   ← E' is new commit
feature:     B ← E ← F         ← unchanged
```

## Command Explanation

### Syntax
```bash
git cherry-pick [options] <commit>...
```

```bash
# Cherry-pick single commit
git cherry-pick abc123

# Cherry-pick a range of commits (exclusive of start)
git cherry-pick abc123..def456

# Cherry-pick a range (inclusive of start)
git cherry-pick abc123^..def456

# Cherry-pick without auto-committing (stage only)
git cherry-pick -n abc123
git cherry-pick --no-commit abc123

# Edit commit message during cherry-pick
git cherry-pick -e abc123

# Continue after conflict resolution
git cherry-pick --continue

# Abort cherry-pick
git cherry-pick --abort

# Hotfix workflow: fix on develop, cherry-pick to main
git switch main
git cherry-pick <fix-commit-sha>
git tag -a v2.1.1 -m "Hotfix"
git push origin main --tags

# Port fix to multiple release branches
git switch release/v2.x
git cherry-pick <fix-commit-sha>
git switch release/v1.x
git cherry-pick <fix-commit-sha>
```

---

# 9. Branching Strategies

## What is it?
A **branching strategy** is a team agreement on how branches are named, created, merged, and deleted throughout the development lifecycle. It defines the flow from code writing to production deployment.

## Why It Matters
- Without a strategy, teams get: merge conflicts, unclear release state, broken main, deploy fear
- The right strategy depends on team size, release frequency, and product type
- Tech Lead interviews always include this question — know the tradeoffs, not just the names

## Internal Working
*(Not applicable — this is a process/convention topic, not a Git internal)*

**Strategy Comparison:**

| Strategy | Branch Structure | Best For | Merge Frequency |
|----------|-----------------|----------|-----------------|
| Git Flow | main, develop, feature/*, release/*, hotfix/* | Versioned software, scheduled releases | Infrequent |
| GitHub Flow | main + feature branches | SaaS, continuous deployment | Daily |
| Trunk-Based | main (trunk) + very short-lived branches | High-velocity teams, strong CI | Multiple times/day |
| GitLab Flow | main + environment branches | Environment-based promotion | Per environment |

## Command Explanation

### Syntax
```bash
# Git Flow pattern (manual — or use git-flow tool)
git switch -c feature/payment develop           # branch from develop
git switch develop && git merge --no-ff feature/payment   # merge back

git switch -c release/2.1.0 develop            # cut release
git switch main && git merge --no-ff release/2.1.0        # merge to main
git tag -a v2.1.0 -m "Release"
git switch develop && git merge --no-ff release/2.1.0     # backport

git switch -c hotfix/v2.1.1 main               # hotfix from main
git switch main && git merge --no-ff hotfix/v2.1.1        # merge + tag
git switch develop && git cherry-pick <fix-sha>           # backport fix
```

```bash
# GitHub Flow pattern
git switch -c feature/payment main     # branch from main
# develop → push → open PR → review → merge to main → auto-deploy

# Trunk-Based Development pattern
git switch -c fix/payment main         # max 2 days lifespan
# All features behind feature flags
# Merge to main daily, deploy immediately
```

---

# 10. Interview Questions & Answers

**Q: "What is the difference between merge and rebase?"**
> Both integrate changes from one branch into another. Merge creates a merge commit with two parents, preserving the exact non-linear history. Rebase replays commits on a new base, rewriting their SHAs to produce linear history. Merge is safe on shared branches — it doesn't rewrite history. Rebase must only be used on local/private branches because rewriting SHAs breaks history for anyone who already has those commits. My rule: rebase to clean up feature branch history before opening a PR, merge (via PR) to integrate into main.

**Q: "What is a fast-forward merge and when does it occur?"**
> A fast-forward merge occurs when the branch being merged into has no new commits since the feature branch diverged — the base is a direct ancestor of the feature tip. Git just moves the pointer forward without a new merge commit. Use `--no-ff` to always create a merge commit regardless, which some teams prefer for auditability.

**Q: "What is interactive rebase and what can you do with it?"**
> `git rebase -i HEAD~N` opens an editor with the last N commits and action prefixes. You can: `pick` (keep), `reword` (edit message), `squash` (combine with previous, keep both messages), `fixup` (combine, discard this message), `edit` (stop and amend), `drop` (delete), or reorder commits by moving lines. I use it before every PR to squash WIP commits into clean, atomic commits. The golden rule: only rebase commits not yet pushed to a shared branch.

**Q: "Branching strategy for a 50-person team doing daily deployments — what would you choose?"**
> GitHub Flow or Trunk-Based Development. With 50 engineers deploying daily, Git Flow's long-lived branches create merge conflict overhead and slow delivery. GitHub Flow: feature branches off main, PRs with 2 required reviews and CI gates, immediate deploy after merge to main. For even higher velocity or when branches regularly live > 2 days, Trunk-Based with feature flags — merge incomplete features hidden behind flags, deploy daily. The key enabler is strong CI: tests must be fast and reliable enough that engineers trust merging to main means it's deployable.

**Q: "What is cherry-pick and when would you use it?"**
> `git cherry-pick <sha>` copies a specific commit's changes onto the current branch, creating a new commit with the same diff but a different SHA. Primary use: hotfixes — a bug fix committed on develop that needs immediate backport to a production release branch, without merging the entire develop branch. Secondary use: selectively porting a feature to multiple release lines. I avoid overusing it because it creates duplicate commits that can confuse history and cause conflicts when branches eventually merge.

---

# 11. Quick Revision Cheatsheet

```bash
# ─── BRANCH MANAGEMENT ────────────────────────────────────
git branch                        # list local
git branch -vv                    # with tracking + ahead/behind
git branch -a                     # all (local + remote-tracking)
git branch -d feature/name        # delete merged branch
git branch -D feature/name        # force delete
git branch --merged               # safe to delete
git branch -m old new             # rename

# ─── SWITCH / CREATE ──────────────────────────────────────
git switch -c feature/name        # create + switch ⭐
git switch main                   # switch to existing
git switch -                      # previous branch
git switch -c branch <tag/sha>    # from specific point

# ─── MERGE ────────────────────────────────────────────────
git merge feature/name            # merge into current
git merge --no-ff feature/name    # always create merge commit
git merge --squash feature/name   # squash to single staged change
git merge --abort                 # abort in-progress merge
git mergetool                     # visual conflict resolution

# ─── CONFLICT RESOLUTION ──────────────────────────────────
git show :1:file                  # base version
git show :2:file                  # our version
git show :3:file                  # their version
git checkout --ours <file>        # take our side
git checkout --theirs <file>      # take their side

# ─── REBASE ───────────────────────────────────────────────
git rebase main                   # rebase onto main
git rebase -i HEAD~n              # interactive rebase ⭐
git rebase --continue             # after resolving conflict
git rebase --abort                # cancel rebase
git pull --rebase                 # fetch + rebase

# ─── CHERRY-PICK ──────────────────────────────────────────
git cherry-pick <sha>             # copy single commit
git cherry-pick <sha1>^..<sha2>   # copy range (inclusive)
git cherry-pick -n <sha>          # stage only (no commit)
git cherry-pick --abort           # cancel

# ─── REMOTE BRANCH ────────────────────────────────────────
git push -u origin feature/name   # push + set tracking
git push --force-with-lease       # force push (safer) ⭐
git push origin --delete branch   # delete remote branch
git fetch --prune                 # sync + clean stale refs
```

---

> **Previous:** [03 · Git File Lifecycle](./03_Git_File_Lifecycle.md)  
> **Next:** [05 · Remote Repositories & Collaboration →](./05_Remote_Repositories_and_Collaboration.md)
