# 01 · Git Introduction & Setup

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** None — this is the entry point of the Git knowledge base

---

## Table of Contents

1. [What is Version Control?](#1-what-is-version-control)
2. [Types of VCS](#2-types-of-vcs)
3. [What is Git?](#3-what-is-git)
4. [What is a Git Repository?](#4-what-is-a-git-repository)
5. [Git Installation & Verification](#5-git-installation--verification)
6. [Git Configuration (System · Global · Local)](#6-git-configuration-system--global--local)
7. [Initialising a Repository — `git init`](#7-initialising-a-repository--git-init)
8. [What is GitHub? (Git vs GitHub)](#8-what-is-github-git-vs-github)
9. [Interview Questions & Model Answers](#9-interview-questions--model-answers)
10. [Quick Revision Cheatsheet](#10-quick-revision-cheatsheet)
11. [Real-World Workflows](#11-real-world-workflows)
12. [Troubleshooting Common Setup Issues](#12-troubleshooting-common-setup-issues)

---

## 1. What is Version Control?

### Definition

A **Version Control System (VCS)** tracks changes to files over time so you can recall specific versions later, understand *what* changed, *when* it changed, *who* changed it, and *why* it changed.

### The Problem Without VCS

```
paymentService.java          ← which one is production?
paymentService_final.java
paymentService_final_final.java
paymentService_final_monday.java
```

Without VCS you rely on manual file naming conventions — error-prone, unscalable, and impossible to diff or rollback cleanly.

### With VCS — Structured History

```
File: paymentService.java

Version 1  ─ Author: Shreya   ─ 4 May 2026  ─ "basic pay() method"
Version 2  ─ Author: Shreya   ─ 5 May 2026  ─ "add UPI flow"         ← parent: v1
Version 3  ─ Author: Ankit    ─ 6 May 2026  ─ "add validation"        ← parent: v2
```

VCS answers **WHAT · WHEN · WHO · WHY** for every change.

### Why It Matters (Real-World)

| Scenario | Without VCS | With VCS |
|---|---|---|
| Production bug introduced | Grep through emails | `git blame` + `git bisect` |
| Rollback needed | Restore from backup (hours) | `git revert` / `git reset` (seconds) |
| Two devs edit same file | Overwrite each other | Merge with conflict resolution |
| Audit / compliance | Manual log | Full immutable history |
| Release branching | Copy folders | Named branches + tags |

---

## 2. Types of VCS

### 2.1 Centralised VCS (CVCS) — e.g. SVN, Perforce

```
         ┌─────────────┐
         │   Server    │  ← single source of truth
         │  (history)  │
         └──────┬──────┘
       ┌────────┼────────┐
   Dev A      Dev B    Dev C
 (working   (working  (working
  copy)      copy)     copy)
```

**Drawback:** Single point of failure. No network → cannot commit.

### 2.2 Distributed VCS (DVCS) — Git, Mercurial

```
         ┌─────────────┐
         │  Remote     │  ← shared, but NOT single point of failure
         │  (GitHub)   │
         └──────┬──────┘
       ┌────────┼────────┐
   Dev A      Dev B    Dev C
 (full repo) (full repo) (full repo)
```

Every developer holds the **complete repository** — full history, all branches. You can commit, branch, and view history entirely offline.

> **Interview Insight:** Git is a DVCS. Every clone is a full backup of the repo. This is why Git is resilient — if GitHub goes down, every developer still has a complete copy locally.

---

## 3. What is Git?

### Definition

Git is a **free, open-source, distributed version control system** created by Linus Torvalds in 2005 (to manage Linux kernel development after the BitKeeper licence was revoked).

### Core Design Goals

| Goal | How Git Achieves It |
|---|---|
| Speed | Local operations (no network for commit/log/diff) |
| Data integrity | SHA-1 hashing on every object |
| Non-linear development | Lightweight branching and merging |
| Fully distributed | Every clone is a complete repo |
| Large project support | Designed for the Linux kernel (millions of lines) |

### Git's Internal Model (Mental Model — essential for interviews)

Git does **NOT** store file diffs. It stores **snapshots**.

```
Commit C1  →  snapshot of entire project at that point in time
Commit C2  →  snapshot of entire project (unchanged files share blob objects)
Commit C3  →  snapshot of entire project
```

Each commit points to the previous commit, forming a **Directed Acyclic Graph (DAG)**.

> **Interview Insight:** "Git stores snapshots, not diffs" is one of the most important things to understand about Git's internal model. SVN stores diffs; Git stores content-addressed snapshots, which makes branching and merging dramatically faster.

---

## 4. What is a Git Repository?

### Definition

A **Git Repository (repo)** is a directory tracked and managed by Git. It contains:
- Your project files (Working Directory)
- A hidden `.git/` folder — the actual Git database

### Structure of `.git/` Directory

```
.git/
├── HEAD          ← pointer to current branch ref
├── config        ← local config (user.name, user.email, aliases)
├── index         ← staging area (binary file)
├── objects/      ← ALL Git objects (blobs, trees, commits, tags)
│   ├── 42/
│   │   └── f9a3297b7e90d22ef17e255c4aca705b4a6f4c  ← blob
│   ├── info/
│   └── pack/
├── refs/
│   ├── heads/    ← local branch pointers
│   │   └── main
│   └── tags/     ← tag pointers
├── logs/         ← reflog data
└── hooks/        ← pre-commit, pre-push scripts
```

> **Interview Insight:** When you delete the `.git/` folder you destroy the entire version history. The project files remain but Git tracking is completely gone. This is why `.git/` is sacred.

### Types of Repositories

| Type | Description | Use Case |
|---|---|---|
| **Local** | On your machine | Day-to-day development |
| **Remote** | On a server (GitHub/GitLab/Bitbucket) | Team collaboration, backup |
| **Bare** | No working directory, only `.git/` contents | Server-side repos (what GitHub stores internally) |
| **Fork** | A copy of another repo under your account | Open-source contributions |

---

## 5. Git Installation & Verification

### Installation

```bash
# macOS (Homebrew)
brew install git

# Ubuntu / Debian
sudo apt update && sudo apt install git -y

# Windows
# Download from: https://git-scm.com/download/win

# Verify installation
git --version
# git version 2.39.5 (Apple Git-154)
```

### Verify Git is Working in a Project

```bash
# Inside a non-git folder
git branch
# fatal: not a git repository (or any of the parent directories): .git

# This error means: you haven't run git init yet → correct behaviour
```

---

## 6. Git Configuration (System · Global · Local)

### Three Configuration Levels

```
System  (/etc/gitconfig)          ← applies to ALL users on the machine
  └── Global (~/.gitconfig)       ← applies to ALL repos for current user
        └── Local (.git/config)   ← applies only to THIS repository  ← highest priority
```

**Priority:** Local > Global > System (lower file always wins)

### Setting Username and Email

```bash
# Local (recommended when working across multiple GitHub accounts)
git config --local user.name  "Shreya"
git config --local user.email "shreya@company.com"

# Global (most common setup for personal machines)
git config --global user.name  "Shreya"
git config --global user.email "shreya@personal.com"

# System (admin use only)
git config --system user.name  "CI-Bot"
```

> **Why this matters:** `user.name` and `user.email` are embedded in every commit object. They appear in `git log`, GitHub commit history, and blame views. They are NOT authentication credentials — they are metadata only.

### Viewing Configuration

```bash
git config --list                   # show all effective configs
git config --list --show-origin     # show config + which file it came from
git config user.name                # show single key
```

### Useful Additional Configurations

```bash
# Set default branch name to 'main'
git config --global init.defaultBranch main

# Set preferred editor
git config --global core.editor "vim"
git config --global core.editor "code --wait"   # VS Code

# Create command aliases
git config --local alias.st   status
git config --local alias.co   checkout
git config --local alias.br   branch
git config --local alias.lg   "log --oneline --graph --all --decorate"

# Now: git st  → git status
#      git lg  → beautiful commit graph

# Auto-correct line endings (cross-platform teams)
git config --global core.autocrlf input    # macOS/Linux
git config --global core.autocrlf true     # Windows
```

### Where is Config Stored?

```bash
# Global config file
cat ~/.gitconfig

# Local config file (inside repo)
cat .git/config
```

---

## 7. Initialising a Repository — `git init`

### What It Does Internally

```bash
git init
# Initialized empty Git repository in /Users/shreya/Projects/LearnGit/.git/
```

Internally Git:
1. Creates the `.git/` directory
2. Creates `HEAD` file pointing to `refs/heads/main` (or `master`)
3. Creates empty `objects/` and `refs/` directories
4. Creates default `config` file with local settings

### Anatomy: Before vs After `git init`

```
BEFORE git init:          AFTER git init:
LearnGit/                 LearnGit/
├── src/                  ├── .git/            ← created by git init
├── pom.xml               │   ├── HEAD
└── .gitignore            │   ├── config
                          │   ├── objects/
                          │   └── refs/
                          ├── src/
                          ├── pom.xml
                          └── .gitignore
```

### States After `git init`

After `git init` but **before any `git add`**:
- All files are in **UNTRACKED** state
- Git is aware they exist in the Working Directory
- Git is NOT managing their history yet
- Staging Area (`.git/index`) does NOT exist yet
- Local Repository (`.git/objects/`) is empty

```bash
git status
# On branch main
# No commits yet
# Untracked files:
#   (use "git add <file>..." to include in what will be committed)
#       src/
#       pom.xml
#       .gitignore
```

### `git init --bare` (Advanced)

```bash
git init --bare /srv/repos/myproject.git
```

Creates a repo with no working directory — only the `.git/` contents at the top level. Used for server-side repositories (Git hosting, CI/CD systems). You never directly edit files in a bare repo.

---

## 8. What is GitHub? (Git vs GitHub)

### The Distinction

| | Git | GitHub |
|---|---|---|
| **Type** | Version control tool | Cloud hosting platform |
| **Creator** | Linus Torvalds (2005) | Tom Preston-Werner et al. (2008) |
| **Runs on** | Local machine | Web (Microsoft-owned) |
| **Requires network?** | No (local operations) | Yes |
| **Alternatives** | Mercurial, SVN, Perforce | GitLab, Bitbucket, Gitea |

> **Interview Insight:** "Git and GitHub are NOT the same thing." This is a surprisingly common interview question at all levels. Git is the tool; GitHub is a service that hosts Git repositories and adds collaboration features on top.

### What GitHub Adds on Top of Git

- **Pull Requests (PRs)** — code review workflow
- **Issues & Projects** — project management
- **Actions** — CI/CD pipelines
- **Protected Branches** — enforce review before merge
- **Webhooks** — trigger external services on push
- **Code scanning** — security analysis
- **GitHub Packages** — artifact registry

### Alternatives to GitHub

| Platform | Owned By | Key Differentiator |
|---|---|---|
| GitLab | GitLab Inc. | Built-in CI/CD, self-hostable |
| Bitbucket | Atlassian | Tight Jira integration |
| Azure DevOps | Microsoft | Enterprise, Active Directory |
| Gitea | Open-source | Lightweight, self-hosted |
| AWS CodeCommit | Amazon | AWS-native, IAM auth |

---

## 9. Interview Questions & Model Answers

### Q1: "What is the difference between Git and SVN?"

**Model Answer:**
> Git is a distributed VCS — every developer has a full copy of the repository including complete history. SVN is centralised — history lives only on the server. With Git, you can commit, branch, and view logs offline. Branching in Git is a O(1) operation (just creating a pointer); in SVN it copies files server-side. Git also uses content-addressed storage (SHA hashing) ensuring data integrity, while SVN uses sequential revision numbers.

### Q2: "What happens when you run `git init`?"

**Model Answer:**
> `git init` creates a `.git/` directory in the current folder. This directory IS the Git database. It contains: `HEAD` (pointer to current branch), `objects/` (all blobs, trees, commits), `refs/` (branch and tag pointers), `config` (local configuration), and `index` (staging area — created later on first `git add`). After `git init`, all existing files are in UNTRACKED state — Git is aware they exist but is not managing their versions yet.

### Q3: "What is a bare repository and when would you use it?"

**Model Answer:**
> A bare repository has no working directory — it contains only the contents that would normally be in `.git/`. You use bare repos as the central remote: when you `git push`, you push to a bare repo. GitHub, GitLab, and Bitbucket internally host bare repositories. You'd set one up on a private server with `git init --bare`. Developers never directly edit files in a bare repo; they clone it, work locally, and push back.

### Q4: "Why does Git embed `user.name` and `user.email` in commits?"

**Model Answer:**
> They serve as human-readable metadata identifying the author and committer of each commit. They're baked into the commit object hash — changing them retroactively changes the commit SHA. They're NOT authentication credentials; Git authentication is handled separately (SSH keys, tokens, OAuth). The distinction author vs committer matters in open-source: the author is the person who wrote the patch; the committer is the maintainer who applied it.

### Q5: "What is the difference between `git config --local`, `--global`, and `--system`?"

**Model Answer:**
> These are three scope levels. `--system` applies to all users on the machine (stored in `/etc/gitconfig`). `--global` applies to all repositories for the current user (`~/.gitconfig`). `--local` applies only to the current repository (`.git/config`). The priority order is local > global > system — a local setting always overrides a global one. In practice: use `--global` for personal machines, `--local` when working with multiple Git identities (personal vs work accounts on the same machine).

---

## 10. Quick Revision Cheatsheet

```bash
# ─── VERIFY ────────────────────────────────────────────
git --version                          # check git is installed
git config --list --show-origin        # list all config + source files

# ─── CONFIGURE ─────────────────────────────────────────
git config --global user.name  "Name"
git config --global user.email "email@example.com"
git config --global init.defaultBranch main
git config --global core.editor "code --wait"
git config --local  alias.st status   # git st → git status

# ─── INITIALISE ────────────────────────────────────────
git init                               # init new repo in current dir
git init my-project                    # init into new folder
git init --bare /srv/repos/proj.git    # bare repo (server-side)

# ─── INSPECT ───────────────────────────────────────────
git status                             # working tree + staging status
git log --oneline --graph --all        # visual commit history
ls -la .git/                           # inspect git internals
cat .git/HEAD                          # see current branch pointer
cat .git/config                        # see local config
```

---

## 11. Real-World Workflows

### Workflow A: New Project from Scratch

```bash
# 1. Create project directory and initialise
mkdir payment-service && cd payment-service
git init
git config --local user.name  "Shreya"
git config --local user.email "shreya@company.com"

# 2. Create .gitignore before first commit
echo "target/" >> .gitignore
echo ".env"   >> .gitignore
echo "*.class" >> .gitignore

# 3. Initial commit
git add .gitignore
git commit -m "chore: initialise repo with gitignore"

# 4. Connect to remote
git remote add origin git@github.com:company/payment-service.git
git push -u origin main
```

### Workflow B: Clone Existing Project

```bash
# Clone via SSH (preferred for teams — no password prompts)
git clone git@github.com:company/payment-service.git
cd payment-service

# Configure identity if not set globally
git config --local user.name  "Ankit"
git config --local user.email "ankit@company.com"
```

### Workflow C: Multiple GitHub Accounts on One Machine

```bash
# ~/.ssh/config
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_personal

Host github-work
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_work

# Clone using alias
git clone git@github-work:company/repo.git

# Set local config per repo
git config --local user.email "ankit@company.com"
```

---

## 12. Troubleshooting Common Setup Issues

| Problem | Cause | Solution |
|---|---|---|
| `fatal: not a git repository` | `git init` not run | Run `git init` in project root |
| `Author identity unknown` | No user.name/email set | `git config --global user.name "..."` |
| Commits show wrong author | Local config not set | `git config --local user.name "..."` |
| `.git` folder not visible | Hidden files not shown | Mac: `Cmd+Shift+.`; Linux: `ls -la` |
| `git init` on wrong directory | Wrong path | `rm -rf .git` then re-init in correct dir |
| Line ending issues (Windows) | CRLF vs LF | `git config --global core.autocrlf true` |

---

> **Next:** [02 · Git Architecture →](./02_Git_Architecture.md)
