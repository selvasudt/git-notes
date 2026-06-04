# 05 · Remote Repositories & Collaboration

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [04 · Git Branching & Merging](./04_Git_Branching_and_Merging.md)

---

## Table of Contents

1. [What is a Remote?](#1-what-is-a-remote)
2. [Remote Internals](#2-remote-internals)
3. [Cloning a Repository](#3-cloning-a-repository)
4. [Managing Remotes](#4-managing-remotes)
5. [Fetch, Pull, Push — Deep Dive](#5-fetch-pull-push--deep-dive)
6. [Pull Request / Merge Request Workflow](#6-pull-request--merge-request-workflow)
7. [Fork Workflow (Open-Source Contribution)](#7-fork-workflow-open-source-contribution)
8. [Authentication — SSH vs HTTPS vs Tokens](#8-authentication--ssh-vs-https-vs-tokens)
9. [Tags & Releases](#9-tags--releases)
10. [Submodules](#10-submodules)
11. [Large File Storage (Git LFS)](#11-large-file-storage-git-lfs)
12. [Interview Questions & Model Answers](#12-interview-questions--model-answers)
13. [Quick Revision Cheatsheet](#13-quick-revision-cheatsheet)
14. [Real-World Collaboration Workflows](#14-real-world-collaboration-workflows)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. What is a Remote?

### Definition

A **remote** is a reference to another Git repository — typically hosted on GitHub, GitLab, Bitbucket, or a private server. It is identified by a name (conventionally `origin`) and a URL.

```bash
cat .git/config
# [remote "origin"]
#     url = git@github.com:company/payment-service.git
#     fetch = +refs/heads/*:refs/remotes/origin/*
```

### Why Remotes Exist

| Purpose | Description |
|---|---|
| **Backup** | Off-machine copy of your entire repository |
| **Collaboration** | Shared source of truth for a team |
| **CI/CD trigger** | Pushes trigger automated build/test/deploy |
| **Code review** | PRs/MRs hosted on the remote platform |
| **Release distribution** | Tags and releases accessible to all consumers |

### Multiple Remotes

You can have multiple remotes — common in open-source workflows:

```bash
git remote -v
# origin    git@github.com:myuser/repo.git         (fetch)
# origin    git@github.com:myuser/repo.git         (push)
# upstream  git@github.com:originalorg/repo.git    (fetch)
# upstream  git@github.com:originalorg/repo.git    (push)
```

---

## 2. Remote Internals

### Remote-Tracking References

When you fetch from a remote, Git creates **remote-tracking branches** in:

```
.git/refs/remotes/origin/
├── main           → abc123...
├── develop        → def456...
└── feature/pay    → ghi789...
```

These are **local read-only snapshots** of what those branches looked like at last fetch. They are NOT automatically updated — they only change when you `git fetch` or `git pull`.

### The Fetch Refspec

The fetch refspec tells Git which remote refs to download and where to store them locally:

```bash
cat .git/config
# [remote "origin"]
#     fetch = +refs/heads/*:refs/remotes/origin/*
#     ↑          ↑                 ↑
#  force    remote refs       local storage location
```

`+refs/heads/*:refs/remotes/origin/*` means:
- Fetch ALL branches from remote (`refs/heads/*`)
- Store as `refs/remotes/origin/<branchname>` locally
- `+` = update even if not a fast-forward (force update remote-tracking refs)

### Tracking Branch Relationship

```bash
# main tracks origin/main
# This relationship is stored in .git/config:
# [branch "main"]
#     remote = origin
#     merge = refs/heads/main

git branch -vv
# * main  abc123 [origin/main: ahead 2, behind 1] latest commit message
#   ↑      ↑          ↑            ↑         ↑
# current  SHA   tracking remote  status    last commit msg
```

---

## 3. Cloning a Repository

### What `git clone` Does Internally

```bash
git clone git@github.com:company/payment-service.git
```

Git performs:
1. Creates `payment-service/` directory
2. Runs `git init` inside it
3. Adds `origin` remote pointing to the URL
4. Fetches all objects (blobs, trees, commits) from remote
5. Creates remote-tracking branches (`origin/main`, `origin/develop`, etc.)
6. Creates local `main` branch tracking `origin/main`
7. Checks out `main` into the working directory

### Clone Variants

```bash
# Clone into specific directory
git clone git@github.com:company/repo.git my-local-name

# Clone only latest commit (shallow clone) — faster, less data
git clone --depth 1 git@github.com:company/repo.git

# Shallow clone specific branch only
git clone --depth 1 --branch main git@github.com:company/repo.git

# Clone all branches (default clones all refs, but only checkout main)
git clone git@github.com:company/repo.git

# Clone a specific tag
git clone --branch v2.1.0 --depth 1 git@github.com:company/repo.git

# Clone bare repository (server-side)
git clone --bare git@github.com:company/repo.git repo.git

# Mirror clone (complete, with all refs — for backups)
git clone --mirror git@github.com:company/repo.git
```

### Shallow Clone Implications

```bash
# Shallow clones are common in CI/CD to save time
# But they have limitations:

git clone --depth 1 git@github.com:company/repo.git

# Cannot git log beyond depth 1
# Cannot git blame beyond shallow boundary
# Cannot git bisect across shallow boundary

# Convert shallow to full clone
git fetch --unshallow

# Deepen by N more commits
git fetch --deepen=50
```

---

## 4. Managing Remotes

### Adding & Removing Remotes

```bash
# Add remote
git remote add origin git@github.com:company/repo.git
git remote add upstream git@github.com:originalorg/repo.git

# Remove remote
git remote remove upstream
git remote rm upstream     # same thing

# Rename remote
git remote rename origin old-origin
git remote rename upstream origin

# Change remote URL (e.g., switching from HTTPS to SSH)
git remote set-url origin git@github.com:company/repo.git

# View remote URLs
git remote -v
git remote get-url origin
git remote get-url --all origin   # if multiple push URLs
```

### Inspecting a Remote

```bash
# Detailed remote info
git remote show origin
# * remote origin
#   Fetch URL: git@github.com:company/repo.git
#   Push  URL: git@github.com:company/repo.git
#   HEAD branch: main
#   Remote branches:
#     develop tracked
#     main    tracked
#     feature/pay tracked
#   Local branch configured for 'git pull':
#     main merges with remote main
#   Local ref configured for 'git push':
#     main pushes to main (up to date)
```

### Multiple Push URLs

```bash
# Push to two remotes simultaneously (backup to multiple hosts)
git remote set-url --add --push origin git@github.com:company/repo.git
git remote set-url --add --push origin git@bitbucket.org:company/repo.git

# Now git push pushes to both
```

---

## 5. Fetch, Pull, Push — Deep Dive

### `git fetch`

Downloads objects and refs from remote WITHOUT modifying your working directory or local branches.

```bash
# Fetch all branches from origin
git fetch origin

# Fetch specific branch
git fetch origin main

# Fetch all remotes
git fetch --all

# Fetch + delete stale remote-tracking branches
git fetch --prune
git fetch -p        # shorthand

# Fetch tags
git fetch --tags

# After fetch, your remote-tracking branches update:
# origin/main → now points to latest remote main commit
# Your local main branch is UNCHANGED until you merge/rebase
```

### `git pull`

Shorthand for `git fetch` followed by `git merge` (or `git rebase`) of the tracking branch.

```bash
# Default: fetch + merge (creates merge commit if diverged)
git pull

# Preferred: fetch + rebase (linear history)
git pull --rebase
git pull --rebase=interactive    # interactive rebase

# Pull specific branch
git pull origin main

# Pull and always merge (even if fast-forward possible)
git pull --no-ff

# Set default pull strategy globally
git config --global pull.rebase true    # always rebase on pull
git config --global pull.ff only        # only allow fast-forward pulls (safest)
```

### `git push`

Uploads local commits to remote.

```bash
# Push current branch to its tracked remote
git push

# Push specific branch to origin
git push origin main
git push origin feature/payment

# Push + set upstream tracking (first push of a new branch)
git push -u origin feature/payment
git push --set-upstream origin feature/payment

# Push all branches
git push --all origin

# Push all tags
git push --tags

# Push specific tag
git push origin v2.1.0

# Delete remote branch
git push origin --delete feature/payment
git push origin :feature/payment         # older syntax

# Force push — DANGEROUS on shared branches
git push --force origin feature/payment

# Safer force push (fails if remote has commits you haven't fetched)
git push --force-with-lease origin feature/payment

# Dry run (show what would be pushed, don't actually push)
git push --dry-run
```

### Pull Strategies Comparison

```bash
# Strategy 1: Merge (default)
git pull
# Pros: Preserves full history
# Cons: Creates merge commit → noisy log on busy repos

# Strategy 2: Rebase (recommended for feature branches)
git pull --rebase
# Pros: Linear history, no merge commits
# Cons: Rewrites local commits (safe if not shared)

# Strategy 3: Fast-forward only (strictest)
git pull --ff-only
# Pros: Never creates merge commits, never rewrites
# Cons: Fails if local and remote diverged (you must resolve manually)
# Recommended for: main branch, release branches

# Set globally
git config --global pull.rebase true
```

---

## 6. Pull Request / Merge Request Workflow

### The PR Lifecycle

```
1. CREATE BRANCH
   git switch -c feature/payment-retry

2. DEVELOP
   git commit -m "feat: add retry logic"
   git commit -m "test: add retry unit tests"

3. SYNC WITH BASE
   git fetch origin
   git rebase origin/main

4. PUSH BRANCH
   git push -u origin feature/payment-retry

5. OPEN PR
   - Title: "feat(payment): add exponential backoff retry"
   - Description: context, what changed, how to test
   - Link to issue/ticket
   - Add reviewers

6. CODE REVIEW
   - Reviewers leave comments
   - Author addresses comments with new commits
   - git push (updates the PR automatically)

7. CI PASSES
   - Tests, linting, security scans all green

8. APPROVED + MERGED
   - Merge strategy: Merge commit / Squash / Rebase

9. CLEANUP
   git switch main
   git pull
   git branch -d feature/payment-retry
```

### PR Merge Strategies (GitHub UI)

| Strategy | Creates | History | Best For |
|---|---|---|---|
| **Merge commit** | Merge commit + all original commits | Non-linear, full history | Preserving context of feature branch |
| **Squash merge** | One new commit | Linear, clean | Clean `main` history; squash noise |
| **Rebase merge** | Replayed commits (no merge commit) | Linear, individual commits preserved | When individual commits are meaningful |

### Writing Good PR Descriptions

```markdown
## Summary
Adds exponential backoff retry logic to the payment processor.
When the gateway returns 5xx errors, we now retry up to 3 times
with 2s base delay (doubles each retry).

## Motivation
Reduces payment failures during transient gateway outages.
Addresses: #PAYMENT-247

## Changes
- `PaymentService`: added `processWithRetry()` with exponential backoff
- `RetryConfig`: new config class for retry parameters
- `application.properties`: added `payment.retry.*` config keys

## Testing
- Unit tests: `PaymentServiceTest` covers retry scenarios
- Manual: tested with mock gateway returning 503

## Screenshots / Logs
(attach if relevant)

## Checklist
- [x] Tests added/updated
- [x] Documentation updated
- [x] Tested locally
- [x] No sensitive data exposed
```

### Code Review Best Practices

**As Reviewer:**
- Review intent first (does it solve the right problem?), then implementation
- Distinguish: blocker (must fix) vs suggestion (nice to have) vs nit (style preference)
- Be specific and constructive: "Consider using `Optional` here to avoid NPE" not "This is wrong"
- Approve when ready — don't let PRs sit

**As Author:**
- Keep PRs small (< 400 lines changed is the commonly cited threshold)
- One concern per PR
- Don't take review feedback personally
- Respond to every comment (even if just "done" or "will address in follow-up")

---

## 7. Fork Workflow (Open-Source Contribution)

### The Fork Model

Used in open-source where contributors don't have write access to the original repository.

```
Original Repo (upstream)           Your Fork (origin)
github.com/org/project     →fork→  github.com/you/project
       ↑                                    ↓ clone
       │                               Your Machine
       │ PR                           git remote add upstream
       └───────────────────────────────── git push origin feature
```

### Step-by-Step Fork Workflow

```bash
# 1. Fork on GitHub (click Fork button)

# 2. Clone YOUR fork
git clone git@github.com:you/project.git
cd project

# 3. Add original repo as upstream remote
git remote add upstream git@github.com:org/project.git

# 4. Verify remotes
git remote -v
# origin    git@github.com:you/project.git    (fetch)
# origin    git@github.com:you/project.git    (push)
# upstream  git@github.com:org/project.git    (fetch)
# upstream  git@github.com:org/project.git    (push)

# 5. Create feature branch
git switch -c fix/payment-null-check

# 6. Make changes + commit
git add .
git commit -m "fix: add null check for payment amount"

# 7. Sync with upstream before pushing
git fetch upstream
git rebase upstream/main

# 8. Push to YOUR fork
git push origin fix/payment-null-check

# 9. Open PR from your fork's branch → upstream/main on GitHub

# 10. Keep fork in sync with upstream
git fetch upstream
git switch main
git merge upstream/main      # or: git rebase upstream/main
git push origin main
```

---

## 8. Authentication — SSH vs HTTPS vs Tokens

### SSH Authentication (Recommended for Developers)

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "you@company.com"
# Private key: ~/.ssh/id_ed25519  (NEVER share)
# Public key:  ~/.ssh/id_ed25519.pub  (add to GitHub)

# Add to SSH agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Add public key to GitHub:
# GitHub → Settings → SSH and GPG keys → New SSH key

# Test connection
ssh -T git@github.com
# Hi username! You've successfully authenticated.

# Clone via SSH
git clone git@github.com:company/repo.git
```

### HTTPS with Personal Access Token (PAT)

```bash
# Generate token: GitHub → Settings → Developer settings → Personal access tokens

# Clone via HTTPS
git clone https://github.com/company/repo.git

# When prompted for password, use the TOKEN (not your GitHub password)

# Store credentials (so you don't type token every time)
git config --global credential.helper store         # stores in plaintext ⚠️
git config --global credential.helper osxkeychain   # macOS Keychain (secure)
git config --global credential.helper manager       # Windows Credential Manager
```

### SSH Config for Multiple Accounts

```bash
# ~/.ssh/config
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_personal

Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_work

# Use in clone:
git clone git@github-work:company/repo.git     # uses work key
git clone git@github-personal:you/repo.git     # uses personal key
```

### SSH vs HTTPS Comparison

| Aspect | SSH | HTTPS + Token |
|---|---|---|
| Authentication | Key pair | Username + token |
| Setup effort | Medium (key gen + upload) | Easy |
| Security | Very high | High (token-based) |
| Firewall friendly | Sometimes blocked (port 22) | Always works (port 443) |
| CI/CD friendly | Needs key management | Easier (env var tokens) |
| Recommended for | Local development | CI/CD pipelines |

---

## 9. Tags & Releases

### Tag Types

```bash
# Lightweight tag (just a pointer — no object)
git tag v1.0.0
cat .git/refs/tags/v1.0.0   → commit SHA directly

# Annotated tag (full object with metadata + message) ← USE THIS FOR RELEASES
git tag -a v1.0.0 -m "Release 1.0.0: initial production release"
```

### Semantic Versioning in Tags

```
v<MAJOR>.<MINOR>.<PATCH>[-<prerelease>]

v1.0.0      ← initial release
v1.1.0      ← backward-compatible new feature
v1.1.1      ← bug fix
v2.0.0      ← breaking change
v2.0.0-rc.1 ← release candidate
v2.0.0-beta.1 ← beta
```

### Tagging Workflow

```bash
# Tag current HEAD
git tag -a v1.0.0 -m "Release 1.0.0"

# Tag a specific commit (retroactive tagging)
git tag -a v1.0.0 abc123 -m "Release 1.0.0"

# List all tags
git tag
git tag -l "v1.*"          # filter by pattern

# Show tag details
git show v1.0.0

# Push tags to remote (tags NOT pushed with git push by default)
git push origin v1.0.0     # push single tag
git push --tags            # push ALL tags ⚠️ pushes all, including temp tags

# Safer: push only annotated tags
git push origin 'refs/tags/v*'

# Delete tag
git tag -d v1.0.0               # delete local
git push origin --delete v1.0.0  # delete remote

# Checkout tag (enters detached HEAD)
git checkout v1.0.0

# Create branch from tag
git switch -c release/v1.0.0 v1.0.0
```

### `git describe` — Human-Readable Version String

```bash
# Useful for auto-generating version strings in CI/CD
git describe --tags
# v1.2.3-14-gabc123def
# ↑       ↑  ↑
# tag  14 commits  abbreviated SHA
#      since tag

git describe --tags --abbrev=0    # just the tag name: v1.2.3
```

---

## 10. Submodules

### What Are Submodules?

Submodules allow you to include another Git repository inside your repository as a sub-directory, while keeping each repository's history separate.

```bash
# Add submodule
git submodule add git@github.com:company/shared-lib.git libs/shared

# .gitmodules file is created:
# [submodule "libs/shared"]
#     path = libs/shared
#     url  = git@github.com:company/shared-lib.git
```

### Working with Submodules

```bash
# Clone repo with submodules
git clone --recurse-submodules git@github.com:company/main-app.git

# If you forgot --recurse-submodules:
git submodule init
git submodule update

# Update all submodules to latest
git submodule update --remote

# Show submodule status
git submodule status

# Run command in all submodules
git submodule foreach git pull origin main
```

### Submodule Pitfalls

> **Interview Insight:** Submodules are notorious for causing confusion. Common issues:
> - Cloning without `--recurse-submodules` leaves empty subdirectories
> - Updating a submodule changes the parent repo (you must commit the pointer update)
> - `git pull` doesn't automatically update submodules
> - Many teams replace submodules with package managers (npm, Maven, pip) or monorepos

---

## 11. Large File Storage (Git LFS)

### The Problem

Git stores entire file content as blobs. Large binary files (videos, datasets, ML models, game assets) inflate the repository size dramatically — every version is fully stored.

### Git LFS Solution

Git LFS replaces large files in the repository with **text pointer files** and stores the actual content on a separate LFS server.

```bash
# Install Git LFS
git lfs install

# Track file types with LFS
git lfs track "*.psd"
git lfs track "*.mp4"
git lfs track "models/**/*.bin"

# .gitattributes is created/updated:
# *.psd filter=lfs diff=lfs merge=lfs -text
# *.mp4 filter=lfs diff=lfs merge=lfs -text

git add .gitattributes
git commit -m "chore: configure Git LFS for large assets"

# Normal git add/commit/push now works transparently
git add large-file.psd
git commit -m "add design file"
git push   # LFS content uploaded to LFS server

# Show LFS tracked files
git lfs ls-files

# Show LFS status
git lfs status
```

---

## 12. Interview Questions & Model Answers

### Q1: "What is the difference between `git fetch` and `git pull`?"

**Model Answer:**
> `git fetch` downloads all new objects (commits, branches, tags) from the remote and updates remote-tracking branches (like `origin/main`) — but it does NOT touch your working directory or local branches. It's a safe, read-only operation you can run anytime. `git pull` is `git fetch` followed by either `git merge` or `git rebase` of the tracked remote branch into your current local branch. `git pull` changes your working directory and history. I prefer `git pull --rebase` over default `git pull` because it maintains linear history. In CI pipelines, I use `git fetch` explicitly to have full control over what happens next.

### Q2: "Explain the fork-and-PR workflow for open-source contributions."

**Model Answer:**
> In the fork workflow, contributors don't have write access to the original (upstream) repository. You fork it — creating a full copy under your GitHub account. You clone your fork locally and add the original as a second remote called `upstream`. You create a feature branch, make changes, commit, and push to YOUR fork (origin). Then you open a Pull Request from your fork's branch to the upstream's main branch. The maintainers review and can merge your PR. To keep your fork current with upstream changes, you periodically `git fetch upstream` and merge or rebase your main branch from `upstream/main`, then push to your fork's origin. This model scales perfectly for open-source with thousands of contributors, none of whom need write access to the canonical repo.

### Q3: "What is `git push --force-with-lease` and why is it safer than `--force`?"

**Model Answer:**
> `git push --force` overwrites the remote branch unconditionally, regardless of what's there. If a teammate pushed commits after your last fetch, `--force` deletes their work. `git push --force-with-lease` adds a safety check: it only overwrites the remote if the remote's current tip matches what you last fetched. If someone else pushed in the meantime, it fails with an error, prompting you to `git fetch` first and reconcile. This prevents accidentally destroying teammates' work. I use `--force-with-lease` after rebasing my feature branches and never use `--force` except in dire situations where I've explicitly verified the remote state.

### Q4: "When would you use SSH vs HTTPS for Git authentication?"

**Model Answer:**
> For local development on personal or work machines, I prefer SSH. You generate a key pair, add the public key to GitHub once, and never type passwords. SSH is also more secure — the private key never leaves your machine. For CI/CD pipelines, HTTPS with Personal Access Tokens (PATs) or deploy keys is more practical — you store the token as an environment variable/secret, and it's easier to rotate and scope (fine-grained permissions). For enterprise environments with GitHub Enterprise behind a firewall, HTTPS often works more reliably since SSH on port 22 may be blocked, though SSH over port 443 is also configurable.

### Q5: "What is a shallow clone and when would you use it?"

**Model Answer:**
> A shallow clone (`git clone --depth 1`) downloads only the most recent N commits rather than the entire history. The objects for older commits are not downloaded, making the clone much faster and smaller. This is extremely useful in CI/CD pipelines where you only need the latest code to build and test — you don't need 5 years of history to run unit tests. GitHub Actions and most CI systems do shallow clones by default for exactly this reason. The trade-off is that you can't run `git log` beyond the shallow boundary, can't `git bisect` across it, and `git blame` only works within the shallow depth. If you need the full history later, `git fetch --unshallow` converts it.

### Q6: "How do you handle a situation where your `git push` is rejected because the remote has changes?"

**Model Answer:**
> This rejection means someone else pushed to the same branch after my last fetch. The safest approach depends on the branch type. For shared branches like `main` or `develop`: `git pull --rebase` — this fetches the remote changes and replays my local commits on top of them, maintaining linear history. If there are conflicts, I resolve them per commit during the rebase. For my own feature branches after a rebase: `git push --force-with-lease` — since I intentionally rewrote history and I own this branch. I never `git push --force` on shared branches. The `--ff-only` pull strategy can also be configured globally to prevent accidental local merges.

---

## 13. Quick Revision Cheatsheet

```bash
# ─── REMOTE MANAGEMENT ────────────────────────────────
git remote -v                          # list remotes
git remote add origin <url>            # add remote
git remote set-url origin <new-url>    # change URL
git remote remove upstream             # remove remote
git remote show origin                 # inspect remote details

# ─── CLONE ────────────────────────────────────────────
git clone <url>                        # full clone
git clone --depth 1 <url>             # shallow clone
git clone --recurse-submodules <url>   # with submodules

# ─── FETCH / PULL / PUSH ──────────────────────────────
git fetch --prune                      # fetch + clean stale refs
git pull --rebase                      # fetch + rebase ⭐
git pull --ff-only                     # fetch + fast-forward only (safest)
git push -u origin feature/name        # push + set tracking
git push --force-with-lease            # safer force push
git push --tags                        # push all tags

# ─── TAGS ─────────────────────────────────────────────
git tag -a v1.0.0 -m "message"        # annotated tag ⭐
git tag -l "v1.*"                      # list matching tags
git push origin v1.0.0                # push single tag
git push origin --delete v1.0.0       # delete remote tag
git describe --tags                    # version string from tags

# ─── FORK WORKFLOW ────────────────────────────────────
git remote add upstream <original-url>
git fetch upstream
git rebase upstream/main
git push origin feature/name
# → Open PR to upstream

# ─── SUBMODULES ───────────────────────────────────────
git submodule add <url> path/
git submodule update --init --recursive
git submodule foreach git pull origin main

# ─── SSH ──────────────────────────────────────────────
ssh-keygen -t ed25519 -C "email"
ssh -T git@github.com
```

---

## 14. Real-World Collaboration Workflows

### Workflow A: Daily Developer Sync

```bash
# Start of day: sync feature branch with main
git switch feature/payment
git fetch origin                         # download latest without touching branches
git rebase origin/main                   # replay my commits on top of latest main

# If conflicts:
# resolve → git add → git rebase --continue

# Push updated branch (after rebase, SHAs changed)
git push --force-with-lease origin feature/payment
```

### Workflow B: CI/CD Shallow Clone

```yaml
# GitHub Actions example (shallow clone for speed)
- name: Checkout
  uses: actions/checkout@v4
  with:
    fetch-depth: 1           # shallow clone
    # fetch-depth: 0         # full history (needed for git describe, changelog)

# When you need tags for versioning:
- name: Checkout with tags
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
    fetch-tags: true
```

### Workflow C: Release Tagging

```bash
# Prepare release
git switch main
git pull --ff-only

# Final version bump and changelog
vim CHANGELOG.md
git add CHANGELOG.md
git commit -m "chore: update changelog for v2.1.0"

# Create annotated release tag
git tag -a v2.1.0 -m "Release v2.1.0

Changes:
- feat: exponential backoff retry for payment processor
- fix: null check for payment amount
- perf: optimise database query in PaymentRepository

Breaking changes: none"

# Push commit + tag
git push origin main
git push origin v2.1.0

# Create GitHub release (via CLI)
gh release create v2.1.0 --title "v2.1.0" --notes-file RELEASE_NOTES.md
```

### Workflow D: Syncing a Fork with Upstream

```bash
# Scheduled task or before starting new work
git fetch upstream                        # download upstream changes
git switch main
git rebase upstream/main                  # apply upstream changes to local main
git push origin main                      # update your fork on GitHub

# Now create feature branch from up-to-date main
git switch -c fix/issue-456
```

---

## 15. Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| `Permission denied (publickey)` | SSH key not added or not in agent | `ssh-add ~/.ssh/id_ed25519`; verify with `ssh -T git@github.com` |
| `git push` rejected — non-fast-forward | Remote has commits you don't have | `git pull --rebase` then push |
| `git clone` very slow | Large repo, full history | `git clone --depth 1 <url>` |
| Remote branch not showing after fetch | Stale remote-tracking refs | `git fetch --prune` |
| Pushed wrong branch | Pushed feature to main directly | `git push origin --delete main` (if no branch protection) or revert via PR |
| Submodule directory is empty | Cloned without `--recurse-submodules` | `git submodule update --init --recursive` |
| `Updates were rejected because the tip of your current branch is behind` | Remote advanced beyond local | `git pull --rebase` or `git fetch` + resolve |
| Wrong user in commits on GitHub | `user.email` doesn't match GitHub email | Set correct local config: `git config --local user.email "correct@email.com"` |
| Tag not visible on remote | Tags not pushed | `git push origin v1.0.0` or `git push --tags` |
| `git pull` creating unwanted merge commits | Default pull.rebase not set | `git config --global pull.rebase true` |

---

> **Previous:** [04 · Git Branching & Merging](./04_Git_Branching_and_Merging.md)  
> **Next:** [06 · Advanced Git — Rewriting History & Power Commands](./06_Advanced_Git.md)
