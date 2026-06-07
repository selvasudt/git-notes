# 05 · Remote Repositories & Collaboration

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead  
> **Series:** Git Knowledge Base — File 5 of 12

---

## Table of Contents
1. [git remote](#1-git-remote)
2. [git clone](#2-git-clone)
3. [git fetch](#3-git-fetch)
4. [git pull](#4-git-pull)
5. [git push](#5-git-push)
6. [Remote-Tracking Branches](#6-remote-tracking-branches)
7. [Pull Request / Merge Request Workflow](#7-pull-request--merge-request-workflow)
8. [Fork Workflow](#8-fork-workflow)
9. [SSH vs HTTPS Authentication](#9-ssh-vs-https-authentication)
10. [git tag (Release Tagging)](#10-git-tag-release-tagging)
11. [Interview Questions & Answers](#11-interview-questions--answers)
12. [Quick Revision Cheatsheet](#12-quick-revision-cheatsheet)

---

# 1. git remote

## What is it?
`git remote` manages references to remote repositories. A **remote** is a named URL pointing to another Git repository — typically on GitHub, GitLab, or a private server. The default remote name when cloning is `origin`.

## Why It Matters
- Remotes are the coordination point for team collaboration — the shared source of truth
- Multiple remotes are common: `origin` (your fork) + `upstream` (original repo) in open-source workflows
- Getting the remote URL wrong is a common setup problem

## Internal Working
```
Remote configuration stored in .git/config:
  [remote "origin"]
      url = git@github.com:company/payment-service.git
      fetch = +refs/heads/*:refs/remotes/origin/*

The fetch refspec: +refs/heads/*:refs/remotes/origin/*
  + = force update (even non-fast-forward)
  refs/heads/* = all remote branches
  refs/remotes/origin/* = store locally as remote-tracking refs
```

## Command Explanation

### Syntax
```bash
git remote [add | remove | rename | set-url | show | -v] [name] [url]
```

```bash
# List remotes with URLs
git remote -v

# Add a remote
git remote add origin git@github.com:company/repo.git
git remote add upstream git@github.com:original/repo.git

# Change remote URL (e.g., HTTPS → SSH)
git remote set-url origin git@github.com:company/repo.git

# Rename remote
git remote rename origin old-origin

# Remove remote
git remote remove upstream

# Inspect remote (branches, tracking, HEAD)
git remote show origin
```

---

# 2. git clone

## What is it?
`git clone` downloads a complete copy of a remote repository — all objects (blobs, trees, commits, tags), all branches, and all history — and sets up `origin` pointing to the source URL.

## Why It Matters
- Starting point for every developer joining an existing project
- `--depth 1` (shallow clone) is critical for CI/CD speed — reduces clone time from minutes to seconds
- Understanding what clone does internally explains remote-tracking branches and tracking relationships

## Internal Working
```
git clone git@github.com:company/repo.git performs:
  1. Creates repo/ directory
  2. Runs git init inside it
  3. Adds [remote "origin"] to .git/config
  4. Fetches all objects (blobs, trees, commits, tags) from remote
  5. Creates remote-tracking branches:
       .git/refs/remotes/origin/main
       .git/refs/remotes/origin/develop
  6. Creates local main branch tracking origin/main
  7. Checks out main into Working Directory
```

## Command Explanation

### Syntax
```bash
git clone [options] <url> [<directory>]
```

```bash
# Standard clone
git clone git@github.com:company/payment-service.git

# Clone into custom directory name
git clone git@github.com:company/repo.git my-local-name

# Shallow clone — latest commit only (fast, for CI/CD)
git clone --depth 1 git@github.com:company/repo.git

# Shallow clone of specific branch only
git clone --depth 1 --branch main git@github.com:company/repo.git

# Clone a specific tag
git clone --branch v2.1.0 --depth 1 git@github.com:company/repo.git

# Clone with all submodules
git clone --recurse-submodules git@github.com:company/repo.git

# Bare clone (server-side — no working directory)
git clone --bare git@github.com:company/repo.git repo.git

# Mirror clone (complete backup — all refs)
git clone --mirror git@github.com:company/repo.git

# Convert shallow clone to full history
git fetch --unshallow
```

---

# 3. git fetch

## What is it?
`git fetch` downloads new objects and refs from a remote repository and updates remote-tracking branches (e.g., `origin/main`) — WITHOUT touching your local branches or Working Directory.

## Why It Matters
- Safe, read-only operation — never modifies your local work
- Best practice: always fetch before merging/rebasing to ensure you have the latest remote state
- Essential in CI/CD to see what's changed without altering local state

## Internal Working
```
git fetch origin:
  1. Connects to remote at origin's URL
  2. Downloads any new objects (blobs, trees, commits)
  3. Updates .git/refs/remotes/origin/* files:
       .git/refs/remotes/origin/main → new SHA
       .git/refs/remotes/origin/develop → new SHA
  4. Your local branches (main, develop) are NOT changed
  5. Your Working Directory is NOT changed

Compare:
  Before fetch: origin/main → abc123
  After  fetch: origin/main → def456  (new commits downloaded)
  Local main:   still at abc123  (unchanged until you merge/rebase)
```

## Command Explanation

### Syntax
```bash
git fetch [options] [<remote>] [<refspec>]
```

```bash
# Fetch all branches from origin
git fetch origin

# Fetch and remove stale remote-tracking refs (pruning)
git fetch --prune
git fetch -p             # shorthand

# Fetch all remotes
git fetch --all

# Fetch specific branch only
git fetch origin main

# Fetch all tags
git fetch --tags

# Fetch and write commit graph (speeds up log)
git fetch origin && git commit-graph write --reachable

# See what was fetched
git log HEAD..origin/main --oneline     # commits on remote not yet local
git diff HEAD origin/main               # diff between local and remote
```

---

# 4. git pull

## What is it?
`git pull` is a shorthand for `git fetch` followed by either `git merge` or `git rebase` of the remote-tracking branch into your current local branch.

## Why It Matters
- Most common daily command for syncing with team changes
- Default behavior (`git pull` = fetch + merge) creates unwanted merge commits on busy repos
- `git pull --rebase` is the professional default — linear history, no merge commits

## Internal Working
```
git pull --rebase origin main:
  Step 1: git fetch origin main
    → downloads new commits → updates origin/main
  Step 2: git rebase origin/main
    → replays your local commits on top of fetched commits
    → produces linear history

git pull (default merge):
  Step 1: git fetch origin main
  Step 2: git merge origin/main
    → creates merge commit if branches diverged
    → messy history on busy repos

Set default globally:
  git config --global pull.rebase true    ← always rebase on pull
  git config --global pull.ff only        ← only allow fast-forward (safest)
```

## Command Explanation

### Syntax
```bash
git pull [options] [<remote>] [<branch>]
```

```bash
# Pull with rebase (recommended — linear history)
git pull --rebase

# Pull with merge (default — may create merge commit)
git pull

# Pull specific branch
git pull origin main

# Pull fast-forward only (fails if diverged — prevents accidental merges)
git pull --ff-only

# Pull all remotes
git pull --all

# Set global default to rebase
git config --global pull.rebase true

# Set global default to ff-only (strict)
git config --global pull.ff only
```

---

# 5. git push

## What is it?
`git push` uploads local commits to a remote repository, updating the remote branch to match your local branch.

## Why It Matters
- The way your work becomes visible to teammates and CI/CD systems
- Push rejection (non-fast-forward) is a common daily problem — knowing how to handle it safely is essential
- `--force-with-lease` vs `--force` is a critical safety distinction for team workflows

## Internal Working
```
git push origin main:
  1. Git checks: is local main ahead of remote origin/main?
     - If yes (fast-forward): sends new commits, remote advances
     - If no (diverged): REJECTED — remote has commits you don't have

  2. Sends pack of new objects to remote
  3. Remote updates its refs/heads/main pointer
  4. Your .git/refs/remotes/origin/main updates to match

--force-with-lease safety check:
  Before overwriting remote, checks: does remote's current SHA
  match what you last fetched (your origin/main)?
  If yes → safe to force push
  If no  → ABORT (someone pushed after your last fetch)
```

## Command Explanation

### Syntax
```bash
git push [options] [<remote>] [<refspec>]
```

```bash
# Push current branch to tracked remote
git push

# Push specific branch to origin
git push origin main
git push origin feature/payment

# Push and set upstream tracking (first push of new branch)
git push -u origin feature/payment
git push --set-upstream origin feature/payment

# Push all local branches
git push --all origin

# Push all tags
git push --tags

# Push specific tag
git push origin v2.1.0

# Force push SAFELY (fails if remote has commits you haven't fetched)
git push --force-with-lease origin feature/payment

# Force push DANGEROUSLY (overwrites remote unconditionally) ⚠️
git push --force origin feature/payment

# Delete remote branch
git push origin --delete feature/payment
git push origin :feature/payment     # older syntax

# Dry run (show what would be pushed)
git push --dry-run
```

---

# 6. Remote-Tracking Branches

## What is it?
**Remote-tracking branches** are local read-only references that mirror the state of branches on a remote, as of the last `git fetch`. They live in `.git/refs/remotes/` and are named `<remote>/<branch>` (e.g., `origin/main`).

## Why It Matters
- They are the local "memory" of what the remote looked like last time you fetched
- They only update when you run `git fetch` or `git pull` — not automatically
- They enable offline work — you can compare with `origin/main` without network access

## Internal Working
```
.git/refs/remotes/origin/
  main     → abc123...  (SHA of main on remote, as of last fetch)
  develop  → def456...
  feature/payment → ghi789...

These are NOT branches you commit to.
They update ONLY on git fetch / git pull.

[branch "main"]              ← in .git/config
    remote = origin
    merge = refs/heads/main  ← tracking relationship

git branch -vv shows:
  * main  abc123 [origin/main: ahead 2, behind 1] latest commit msg
                  ↑ tracking   ↑ ahead=local has 2 unpushed
                               ↑ behind=remote has 1 unfetched
```

## Command Explanation

### Syntax
```bash
git branch --set-upstream-to=<remote>/<branch> [<local-branch>]
git branch -u <remote>/<branch>
```

```bash
# View tracking info for all branches
git branch -vv

# Set tracking on existing branch
git branch --set-upstream-to=origin/main main
git branch -u origin/feature/payment feature/payment

# Remove tracking relationship
git branch --unset-upstream

# See all remote-tracking branches
git branch -r

# Prune stale remote-tracking refs (remote branch deleted)
git fetch --prune
git remote prune origin

# Check ahead/behind counts
git status          # shows "ahead N, behind M"
git rev-list --count HEAD..origin/main   # commits behind
git rev-list --count origin/main..HEAD   # commits ahead
```

---

# 7. Pull Request / Merge Request Workflow

## What is it?
A **Pull Request (PR)** (GitHub) or **Merge Request (MR)** (GitLab) is a platform feature — not a Git command — that creates a code review interface for merging a feature branch into a base branch. It combines: diff display, review comments, CI status checks, and approval gates.

## Why It Matters
- The primary quality gate in modern software teams — no code merges to main without a PR
- PRs enforce: peer review, CI validation, CODEOWNERS approval, branch protection rules
- Small, focused PRs (< 400 lines) are reviewed thoroughly; large PRs are rubber-stamped

## Internal Working
```
PR is a PLATFORM feature on top of Git:
  git push origin feature/payment
  → GitHub shows: git diff main..feature/payment
  → Runs CI on feature/payment branch
  → Enables: comments, approvals, status checks

Merge strategies (configured per repo):
  Merge commit:  git merge --no-ff feature/payment
  Squash merge:  git merge --squash feature/payment → single new commit
  Rebase merge:  git rebase feature/payment onto main → linear, no merge commit
```

## Command Explanation

### Syntax
```bash
# Using GitHub CLI (gh)
gh pr create [options]
gh pr review [options]
gh pr merge [options]
```

```bash
# Create PR via GitHub CLI
gh pr create \
  --title "feat(payment): add retry logic" \
  --body "Closes #247. Adds exponential backoff retry." \
  --base main \
  --head feature/payment

# Create as draft (WIP — not ready for review)
gh pr create --draft --title "WIP: payment retry"

# View open PRs
gh pr list

# Check out someone's PR locally for review
gh pr checkout 123

# Approve a PR
gh pr review 123 --approve

# Merge PR (squash)
gh pr merge 123 --squash --delete-branch

# Git workflow before opening PR:
git fetch origin
git rebase origin/main          # sync with latest main
git push -u origin feature/payment
# → Open PR on GitHub UI or via gh CLI
```

---

# 8. Fork Workflow

## What is it?
The **fork workflow** is used in open-source where contributors lack write access to the original repo. A contributor forks (creates a personal copy) the repo on GitHub, clones their fork, makes changes, and opens a PR from their fork to the original (upstream) repository.

## Why It Matters
- Enables thousands of contributors to work on a project without write access
- Forks are complete independent copies — you own your fork, push freely
- Understanding the two-remote setup (`origin` = your fork, `upstream` = original) is essential for open-source contribution

## Internal Working
```
Original repo (upstream):   github.com/org/project
                   → fork →
Your fork (origin):         github.com/you/project
                   → clone →
Your machine (local):       ~/project/
  git remote -v:
    origin   git@github.com:you/project.git     (your fork)
    upstream git@github.com:org/project.git     (original)
```

## Command Explanation

### Syntax
```bash
git remote add upstream <original-repo-url>
git fetch upstream
git rebase upstream/main
```

```bash
# Complete fork workflow:

# 1. Fork on GitHub (click Fork button in browser)

# 2. Clone YOUR fork
git clone git@github.com:you/project.git
cd project

# 3. Add upstream remote
git remote add upstream git@github.com:org/project.git

# 4. Verify remotes
git remote -v
# origin    git@github.com:you/project.git   (fetch + push)
# upstream  git@github.com:org/project.git   (fetch only)

# 5. Create feature branch
git switch -c fix/payment-null-check

# 6. Make changes and commit
git add . && git commit -m "fix: add null check for payment amount"

# 7. Sync with upstream before pushing
git fetch upstream
git rebase upstream/main

# 8. Push to YOUR fork (not upstream)
git push origin fix/payment-null-check

# 9. Open PR: your fork/fix → upstream/main (via GitHub UI)

# 10. Keep fork in sync with upstream
git fetch upstream
git switch main
git rebase upstream/main
git push origin main        # update your fork's main
```

---

# 9. SSH vs HTTPS Authentication

## What is it?
Git supports two authentication protocols for remote operations: **SSH** (key-pair based) and **HTTPS** (username + token based). The choice affects security, convenience, and CI/CD integration.

## Why It Matters
- Wrong authentication setup causes `Permission denied` errors (most common Git setup problem)
- Multiple GitHub accounts on one machine requires SSH config aliases
- CI/CD pipelines typically use HTTPS + tokens (easier to rotate, scope)

## Internal Working
```
SSH:
  Client has: ~/.ssh/id_ed25519 (private key — NEVER share)
  GitHub has: id_ed25519.pub   (public key — added to account)
  Auth: SSH handshake proves private key without transmitting it
  URL format: git@github.com:company/repo.git

HTTPS + Token:
  GitHub generates Personal Access Token (PAT) with specific scopes
  Used as password in HTTPS basic auth
  URL format: https://github.com/company/repo.git
  Credential helper stores token so you don't retype every time
```

## Command Explanation

### Syntax
```bash
ssh-keygen -t ed25519 -C "email"
ssh -T git@github.com
git config --global credential.helper <helper>
```

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "you@company.com"
# Private: ~/.ssh/id_ed25519
# Public:  ~/.ssh/id_ed25519.pub  → add to GitHub: Settings > SSH Keys

# Add to SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Test SSH connection
ssh -T git@github.com
# Hi username! You've successfully authenticated.

# Multiple GitHub accounts via SSH config
# ~/.ssh/config:
# Host github-work
#   HostName github.com
#   User git
#   IdentityFile ~/.ssh/id_ed25519_work
#
# Host github-personal
#   HostName github.com
#   User git
#   IdentityFile ~/.ssh/id_ed25519_personal

# Clone using SSH alias
git clone git@github-work:company/repo.git

# HTTPS credential helpers
git config --global credential.helper osxkeychain  # macOS (secure)
git config --global credential.helper manager      # Windows
git config --global credential.helper store        # Linux (plaintext ⚠️)

# Switch existing repo from HTTPS to SSH
git remote set-url origin git@github.com:company/repo.git
```

---

# 10. git tag (Release Tagging)

## What is it?
`git tag` creates a named reference to a specific commit. **Annotated tags** are full Git objects with tagger info, timestamp, and message — used for releases. **Lightweight tags** are just pointer files — used for bookmarks.

## Why It Matters
- Tags are how you mark production releases — `v2.1.0` always points to exactly what was deployed
- Annotated tags survive garbage collection and can be GPG-signed for security
- Tag pushes trigger release pipelines in CI/CD

## Internal Working
```
Annotated tag — a full Git object:
  .git/objects/t9/x8y7z6... (tag object)
    object  c1a2b3c4...      ← tagged commit SHA
    type    commit
    tag     v2.1.0
    tagger  Bot <ci@co.com> 1746000000
    Release v2.1.0

  .git/refs/tags/v2.1.0 → t9x8y7z6...  (points to tag object)

Lightweight tag — just a pointer file:
  .git/refs/tags/v2.1.0-light → c1a2b3c4...  (points directly to commit)
  No object in .git/objects/
```

## Command Explanation

### Syntax
```bash
git tag [options] [<tagname>] [<commit>]
```

```bash
# Create annotated tag (USE FOR RELEASES)
git tag -a v2.1.0 -m "Release v2.1.0"

# Tag a specific commit (retroactive)
git tag -a v2.1.0 abc123 -m "Release v2.1.0"

# GPG-signed tag
git tag -s v2.1.0 -m "Release v2.1.0"

# Lightweight tag (bookmarks only — NOT for releases)
git tag v2.1.0-draft

# List all tags (alphabetical)
git tag

# List with SemVer sort (correct order!)
git tag -l --sort=-version:refname "v*"

# Inspect tag object
git show v2.1.0
git cat-file -p v2.1.0

# Get commit SHA from tag
git rev-parse v2.1.0^{}

# Push single tag to remote
git push origin v2.1.0

# Push all tags
git push --tags

# Delete local tag
git tag -d v2.1.0

# Delete remote tag
git push origin --delete v2.1.0

# Checkout tag (enters detached HEAD)
git checkout v2.1.0

# Create branch from tag (safer)
git switch -c release/v2.1.0 v2.1.0

# Generate version string
git describe --tags --abbrev=0    # → v2.1.0
git describe --tags               # → v2.1.0-14-gabc123
```

---

# 11. Interview Questions & Answers

**Q: "What is the difference between git fetch and git pull?"**
> `git fetch` downloads new objects and updates remote-tracking branches (e.g., `origin/main`) but does NOT modify your local branches or Working Directory — it is purely read-only. `git pull` is `git fetch` followed by either `git merge` or `git rebase` into your current branch, which DOES change your local branch and Working Directory. I prefer `git pull --rebase` over default `git pull` because rebase produces linear history without merge commit noise. In CI/CD, I use explicit `git fetch` to have full control over what happens next.

**Q: "Explain the fork workflow for open-source contribution."**
> You fork the repo on GitHub — creating your own copy. Clone YOUR fork locally and add the original as a second remote called `upstream`. Work on a feature branch, push to YOUR fork (`origin`), then open a PR from your fork's branch to the upstream's main branch. To keep in sync: `git fetch upstream && git rebase upstream/main`. This scales to thousands of contributors — nobody needs write access to the canonical repo, and maintainers control what merges through PR review.

**Q: "What is --force-with-lease and why is it safer than --force?"**
> `git push --force` overwrites the remote branch unconditionally — if a teammate pushed after your last fetch, their work is destroyed. `--force-with-lease` adds a safety check: it only overwrites if the remote's current tip matches what you last fetched (your local `origin/main` value). If someone pushed in between, it fails and prompts you to fetch first. Always use `--force-with-lease` when force-pushing rebased feature branches. Never force-push shared branches like `main`.

**Q: "What is the difference between annotated and lightweight tags?"**
> A lightweight tag is just a file in `.git/refs/tags/` containing a commit SHA — a simple named pointer. An annotated tag is a full Git object stored in `.git/objects/`, containing tagger name, email, timestamp, a message, and optionally a GPG signature. For releases, always use annotated tags: they carry metadata (who tagged, when, why), they can be GPG-signed, they work correctly with `git describe`, and they communicate formal intent. Use lightweight tags only for personal temporary bookmarks.

---

# 12. Quick Revision Cheatsheet

```bash
# ─── REMOTE MANAGEMENT ────────────────────────────────────
git remote -v                          # list remotes
git remote add origin <url>            # add remote
git remote set-url origin <url>        # change URL
git remote remove upstream             # remove remote
git remote show origin                 # detailed remote info

# ─── CLONE ────────────────────────────────────────────────
git clone <url>                        # full clone
git clone --depth 1 <url>             # shallow (CI/CD)
git clone --recurse-submodules <url>   # with submodules

# ─── FETCH / PULL / PUSH ──────────────────────────────────
git fetch --prune                      # fetch + clean stale refs
git pull --rebase                      # fetch + rebase ⭐
git pull --ff-only                     # fetch + fast-forward only
git push -u origin feature/name        # push + set tracking
git push --force-with-lease            # safer force push ⭐

# ─── TRACKING ─────────────────────────────────────────────
git branch -vv                         # show tracking + ahead/behind
git branch -u origin/main main         # set tracking
git fetch --prune                      # remove stale remote-tracking refs

# ─── TAGS ─────────────────────────────────────────────────
git tag -a v2.1.0 -m "msg"            # annotated tag ⭐
git tag -l --sort=-version:refname "v*" # list SemVer sorted
git push origin v2.1.0                # push single tag
git push origin --delete v2.1.0       # delete remote tag
git describe --tags --abbrev=0         # latest tag name

# ─── FORK WORKFLOW ────────────────────────────────────────
git remote add upstream <original>
git fetch upstream
git rebase upstream/main
git push origin feature/name
# → Open PR to upstream

# ─── SSH SETUP ────────────────────────────────────────────
ssh-keygen -t ed25519 -C "email"
ssh-add ~/.ssh/id_ed25519
ssh -T git@github.com
```

---

> **Previous:** [04 · Git Branching & Merging](./04_Git_Branching_and_Merging.md)  
> **Next:** [06 · Advanced Git →](./06_Advanced_Git.md)
