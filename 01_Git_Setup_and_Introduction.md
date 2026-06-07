# 01 · Git Introduction & Setup

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead  
> **Series:** Git Knowledge Base — File 1 of 12

---

## Table of Contents
1. [Version Control System (VCS)](#1-version-control-system-vcs)
2. [Git](#2-git)
3. [Git Repository](#3-git-repository)
4. [GitHub vs Git](#4-github-vs-git)
5. [git init](#5-git-init)
6. [git config](#6-git-config)
7. [Interview Questions & Answers](#7-interview-questions--answers)
8. [Quick Revision Cheatsheet](#8-quick-revision-cheatsheet)

---

# 1. Version Control System (VCS)

## What is it?
A **Version Control System** tracks and records changes to files over time so you can recall specific versions, compare changes, identify who changed what, and restore older states when required.

## Why It Matters
- Answers **WHAT** changed, **WHEN**, **WHO** changed it, and **WHY**
- Enables multiple developers to work in parallel without overwriting each other
- Provides rollback capability when bugs are introduced
- Serves as the complete audit trail for compliance and debugging
- Eliminates the "final_v2_REALLY_FINAL.java" naming problem

**Without VCS vs With VCS:**
```
Without VCS:                        With VCS:
paymentService.java                 v1 — Author: Shreya — 4 May — "basic pay()"
paymentService_final.java           v2 — Author: Shreya — 5 May — "add UPI"
paymentService_final2.java          v3 — Author: Ankit  — 6 May — "validation"
paymentService_FINAL_monday.java
```

## Internal Working
VCS stores history as a sequence of snapshots or deltas. Two major models:

**Centralised VCS (SVN, Perforce):**
```
Server (all history) ← single point of failure
  ├── Dev A (working copy only)
  ├── Dev B (working copy only)
  └── Dev C (working copy only)
```

**Distributed VCS (Git, Mercurial):**
```
Remote (shared hub)
  ├── Dev A (FULL repository — complete history)
  ├── Dev B (FULL repository — complete history)
  └── Dev C (FULL repository — complete history)
```
Every clone IS a full backup.

## Command Explanation
No specific commands here — VCS is a concept. See `git init` to initialise a Git VCS.

---

# 2. Git

## What is it?
**Git** is a free, open-source, distributed version control system created by Linus Torvalds in 2005 to manage Linux kernel development. It tracks file changes using a content-addressable object database.

## Why It Matters
- **Speed:** All core operations (commit, log, diff, branch) are local — no network required
- **Integrity:** Every object is SHA-1 hashed — corruption is immediately detectable
- **Non-linear development:** Lightweight branches enable parallel feature development
- **Industry standard:** Used by virtually every software team worldwide

## Internal Working
Git does **NOT** store diffs. It stores **snapshots** of the entire project at each commit.

```
Commit C1 → snapshot of ALL files at that point in time
Commit C2 → snapshot of ALL files (unchanged files re-use existing blob objects)
Commit C3 → snapshot of ALL files
```

Every object is stored in `.git/objects/` and identified by SHA-1 hash of its content — this is called **content-addressable storage**.

The four object types:
```
blob    → stores raw file content
tree    → stores directory structure (maps names → blobs/trees)
commit  → stores snapshot metadata (tree + parent + author + message)
tag     → stores named annotated reference to a commit
```

## Command Explanation

### Syntax
```bash
git --version
```

**Options:**
```bash
git --version          # show installed Git version
git --help             # list common commands
git help <command>     # detailed help for a command
git <command> --help   # same as above
```

**Example:**
```bash
git --version
# git version 2.39.5 (Apple Git-154)
```

---

# 3. Git Repository

## What is it?
A **Git repository (repo)** is any directory that Git is tracking. It contains your project files (Working Directory) plus a hidden `.git/` folder which IS the entire Git database — all history, branches, tags, and configuration.

## Why It Matters
- The `.git/` folder IS the repository — deleting it destroys all history
- Every clone of a remote repo contains the complete `.git/` database locally
- Understanding repo structure is essential for diagnosing and recovering from problems

## Internal Working
```
project/                       ← Working Directory (your files)
└── .git/                      ← Git Database (DO NOT MANUALLY EDIT)
    ├── HEAD                   ← pointer to current branch
    ├── config                 ← local configuration
    ├── index                  ← staging area (binary)
    ├── objects/               ← ALL Git objects (blobs, trees, commits, tags)
    │   ├── 42/                ← folder = first 2 chars of SHA
    │   │   └── f9a3297b...    ← file = remaining 38 chars of SHA
    │   ├── info/
    │   └── pack/              ← packed objects (compressed)
    ├── refs/
    │   ├── heads/             ← local branch pointers
    │   │   └── main           ← contains SHA of latest commit on main
    │   ├── remotes/           ← remote-tracking refs
    │   └── tags/              ← tag pointers
    ├── logs/                  ← reflog data
    └── hooks/                 ← automation scripts
```

**Types of repositories:**

| Type | Description | Use Case |
|------|-------------|----------|
| Local | On your machine with working dir | Day-to-day development |
| Remote | On a server (GitHub/GitLab) | Collaboration, backup |
| Bare | No working dir, only `.git/` contents | Server-side hosting |

## Command Explanation

### Syntax
```bash
git init [directory] [options]
```
*(Full coverage in Section 5 — `git init`)*

---

# 4. GitHub vs Git

## What is it?
- **Git** — the version control *tool* (CLI program installed on your machine)
- **GitHub** — a cloud *platform* that hosts Git repositories and adds collaboration features (PRs, Issues, Actions) on top of Git

## Why It Matters
Confusing them is a common interview mistake. Git works entirely without GitHub. GitHub is worthless without Git. They are completely separate things.

## Internal Working
```
Git (local tool):                   GitHub (cloud service):
  git commit                          Pull Requests (code review UI)
  git branch                          Issues (bug/feature tracking)
  git merge                           Actions (CI/CD pipelines)
  git log                             Protected Branches (quality gates)
  ← all local, no network needed →    ← all server-side, needs network →
```

GitHub alternatives: **GitLab** (self-hostable, built-in CI/CD), **Bitbucket** (Atlassian/Jira integration), **Gitea** (lightweight, self-hosted), **Azure DevOps** (enterprise/Microsoft).

## Command Explanation

### Syntax
```bash
# Connect local repo to GitHub remote
git remote add origin <url>
git push -u origin main
```

**No Git command installs or configures "GitHub" — it is a web service you access via browser or `gh` CLI.**

---

# 5. git init

## What is it?
`git init` initialises a new Git repository in the current directory (or a specified path). It creates the `.git/` folder which begins the Git database for that project.

## Why It Matters
- Starting point of every new Git-managed project
- Without it, no Git commands work (`fatal: not a git repository`)
- Running it in the wrong directory is a common mistake that requires cleanup

## Internal Working
When `git init` runs:
1. Creates `.git/` directory
2. Creates `HEAD` file: `ref: refs/heads/main` (points to default branch)
3. Creates empty `objects/` directory (no objects yet)
4. Creates empty `refs/heads/` and `refs/tags/` directories
5. Creates default `config` file with `[core]` settings

**State after `git init` — before any `git add`:**
- All files are **UNTRACKED** — Git is aware they exist but not managing them
- `.git/index` does NOT exist yet (created on first `git add`)
- `.git/objects/` is empty (no blobs created yet)

```bash
git status
# On branch main
# No commits yet
# Untracked files:
#   src/
#   pom.xml
```

## Command Explanation

### Syntax
```bash
git init [<directory>] [--bare] [--initial-branch=<name>]
```

**Options & Examples:**
```bash
# Initialise in current directory
git init

# Initialise in a new named directory
git init my-project

# Initialise with specific default branch name
git init --initial-branch=main
git init -b main                      # shorthand

# Initialise a bare repo (server-side — no working directory)
git init --bare /srv/repos/project.git

# Verify initialisation
ls -la .git/

# Check status immediately after init
git status
# On branch main / No commits yet / Untracked files: ...
```

**Real-world sequence:**
```bash
mkdir payment-service && cd payment-service
git init -b main
git config --local user.name  "Shreya"
git config --local user.email "shreya@company.com"
echo "target/" > .gitignore
git add .gitignore
git commit -m "chore: initialise repo"
git remote add origin git@github.com:company/payment-service.git
git push -u origin main
```

---

# 6. git config

## What is it?
`git config` reads and writes configuration values for Git. It controls identity (name/email embedded in commits), editor preference, aliases, pull strategy, and hundreds of other behaviours.

## Why It Matters
- `user.name` and `user.email` are **baked into every commit's SHA** — wrong identity = wrong attribution in `git blame`, GitHub commit history, and audit logs
- Configuration is layered — local overrides global overrides system (critical for multi-account machines)
- Aliases and pull strategies set here have daily productivity impact

## Internal Working
Three configuration files, each with a scope:

```
/etc/gitconfig            ← system   (all users, all repos)
~/.gitconfig              ← global   (current user, all repos)
.git/config               ← local    (current repo only) ← HIGHEST PRIORITY
```

Priority: **local > global > system** — lower file always wins.

`user.name` and `user.email` are stored as plain text in the config file and read at commit time. They become part of the commit object:
```
author  Shreya <shreya@company.com> 1746000000 +0530
```
This string is included in the SHA-1 computation, so changing the config changes future commit SHAs.

## Command Explanation

### Syntax
```bash
git config [--system | --global | --local] <key> <value>
git config [--system | --global | --local] <key>
git config --list [--show-origin] [--show-scope]
git config --edit [--system | --global | --local]
git config --unset <key>
```

**Identity (required before first commit):**
```bash
# Set for all repos on this machine (most common)
git config --global user.name  "Shreya Jain"
git config --global user.email "shreya@company.com"

# Set only for this repo (overrides global — use for work/personal separation)
git config --local user.name  "Shreya"
git config --local user.email "shreya@personal.com"
```

**Editor and tools:**
```bash
git config --global core.editor "code --wait"          # VS Code
git config --global core.editor "vim"                  # Vim
git config --global merge.tool  vscode
git config --global diff.tool   vscode
git config --global difftool.vscode.cmd 'code --wait --diff $LOCAL $REMOTE'
```

**Pull and push strategy:**
```bash
git config --global pull.rebase true          # rebase on pull (recommended)
git config --global pull.ff    only           # fast-forward only (safest)
git config --global push.default current      # push to same-name remote branch
git config --global push.autoSetupRemote true # auto-set upstream on first push
```

**Default branch:**
```bash
git config --global init.defaultBranch main
```

**Aliases:**
```bash
git config --global alias.st   "status -sb"
git config --global alias.co   "checkout"
git config --global alias.br   "branch -vv"
git config --global alias.lg   "log --oneline --graph --all --decorate"
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.last "log -1 HEAD --stat"

# Usage:
git st    # → git status -sb
git lg    # → visual commit graph
git undo  # → undo last commit, keep staged
```

**Inspect configuration:**
```bash
git config --list                           # all effective config
git config --list --show-origin             # with source file
git config --list --show-origin --show-scope # with scope label
git config user.name                        # single key value
git config --global --edit                  # open global config in editor
cat .git/config                             # raw local config file
```

**Unset / remove:**
```bash
git config --global --unset user.email
git config --local  --unset-all alias.st    # remove all values for key
```

---

# 7. Interview Questions & Answers

**Q: "What is the difference between Git and GitHub?"**
> Git is a distributed version control tool that runs locally. GitHub is a cloud hosting platform for Git repositories with collaboration features (PRs, Actions, Issues). Git works without GitHub. They are separate products by separate organisations (Git: the Linux community; GitHub: Microsoft-owned).

**Q: "What happens when you run `git init`?"**
> Git creates a `.git/` directory containing: `HEAD` (pointing to `refs/heads/main`), empty `objects/` (object store), empty `refs/heads/` and `refs/tags/` (branch/tag pointers), and a default `config` file. All existing files become UNTRACKED — Git is aware they exist but not managing their history. The staging area index does not exist until the first `git add`.

**Q: "What is the difference between `--local`, `--global`, and `--system` in git config?"**
> Three scope levels with descending priority: `--system` applies to all users on the machine (`/etc/gitconfig`). `--global` applies to all repos for the current user (`~/.gitconfig`). `--local` applies only to the current repository (`.git/config`). Local always overrides global which always overrides system. Use global for your usual identity; use local when working with a different GitHub account on a specific project.

**Q: "Why do commits show the wrong author name even though I set git config globally?"**
> A `--local` config in that repository is overriding the global setting. Run `git config --list --show-origin` to see which config file each setting comes from. The local `.git/config` takes priority over `~/.gitconfig`. Fix: `git config --local user.name "Correct Name"` in that repo.

**Q: "What is a bare repository and when would you use one?"**
> A bare repository has no working directory — it contains only the contents that would normally be inside `.git/`. Created with `git init --bare`. Used for server-side repos that act as a shared remote — like what GitHub stores internally. Developers never directly edit files in a bare repo; they clone it, work locally, and push back.

---

# 8. Quick Revision Cheatsheet

```bash
# ─── VERIFY SETUP ─────────────────────────────────────────
git --version                                # check git installed
git config --list --show-origin              # all config + source files

# ─── CONFIGURE IDENTITY ───────────────────────────────────
git config --global user.name  "Your Name"
git config --global user.email "you@email.com"
git config --global init.defaultBranch main
git config --global pull.rebase true
git config --global core.editor "code --wait"

# ─── USEFUL ALIASES ───────────────────────────────────────
git config --global alias.st   "status -sb"
git config --global alias.lg   "log --oneline --graph --all --decorate"
git config --global alias.undo "reset --soft HEAD~1"

# ─── INITIALISE ───────────────────────────────────────────
git init                        # init in current dir
git init my-project             # init in new dir
git init --bare /path/repo.git  # bare repo (server-side)

# ─── INSPECT ──────────────────────────────────────────────
cat .git/HEAD                   # see current branch pointer
cat .git/config                 # see local config
ls -la .git/                    # inspect git database structure
git status                      # working dir + staging state

# ─── SCOPES PRIORITY ──────────────────────────────────────
# local (.git/config) > global (~/.gitconfig) > system (/etc/gitconfig)
```

---

> **Next:** [02 · Git Architecture →](./02_Git_Architecture.md)
