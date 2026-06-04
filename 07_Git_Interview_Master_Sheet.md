# 07 · Git Interview Master Sheet

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead — Final interview prep  
> **Prerequisite:** Files 01–06 of this series

---

## Table of Contents

1. [The 10 Concepts Every Senior Engineer Must Know](#1-the-10-concepts-every-senior-engineer-must-know)
2. [Concept Map — Everything in One View](#2-concept-map--everything-in-one-view)
3. [Interview Questions by Seniority Level](#3-interview-questions-by-seniority-level)
4. [Scenario-Based Questions](#4-scenario-based-questions)
5. [DevOps-Specific Questions](#5-devops-specific-questions)
6. [Tech Lead / Architecture Questions](#6-tech-lead--architecture-questions)
7. [Common Traps & Misconceptions](#7-common-traps--misconceptions)
8. [Command Decision Trees](#8-command-decision-trees)
9. [Git Internals — 60-Second Summary](#9-git-internals--60-second-summary)
10. [One-Page Cheatsheet](#10-one-page-cheatsheet)
11. [Topics to Study Beyond This Series](#11-topics-to-study-beyond-this-series)

---

## 1. The 10 Concepts Every Senior Engineer Must Know

These are non-negotiable for any senior/lead interview. Master the **what, why, and internal mechanism** for each.

| # | Concept | Must Know |
|---|---|---|
| 1 | **Git Object Model** | Blob, Tree, Commit, Tag — SHA-1 content-addressed storage |
| 2 | **The Three Areas** | Working Directory, Staging Area (Index), Local Repository |
| 3 | **File Lifecycle** | Untracked → Staged → Unmodified → Modified |
| 4 | **Merge vs Rebase** | When to use each; the golden rule of rebase |
| 5 | **Branching Strategies** | Git Flow, GitHub Flow, Trunk-Based — pros/cons |
| 6 | **HEAD and References** | Symbolic refs, detached HEAD, remote-tracking refs |
| 7 | **Reflog** | Recovery tool; what it records; 90-day window |
| 8 | **Force Push Safety** | `--force-with-lease` vs `--force`; when to use |
| 9 | **Conflict Resolution** | Three-way merge, stage numbers, resolution strategies |
| 10 | **History Rewriting Safety** | When it's safe; shared branch golden rules |

---

## 2. Concept Map — Everything in One View

```
                         GIT KNOWLEDGE MAP
                         ─────────────────

INTERNALS                          WORKFLOW
─────────                          ────────
Object Types                       Daily: add → commit → push
  ├── Blob (file content)          Branching: switch -c → develop → PR
  ├── Tree (directory)             Emergency: stash → fix → pop
  ├── Commit (snapshot)            Release: tag -a → push --tags
  └── Tag (named commit)
                                   COLLABORATION
Storage                            ─────────────
  ├── .git/objects/ (blobs,trees)  GitHub Flow: branch → PR → merge
  ├── .git/index (staging area)    Fork Flow: fork → upstream → PR
  ├── .git/refs/ (branches, tags)  Code Review: atomic PRs, good messages
  └── .git/HEAD (current ref)
                                   INTEGRATION
File States                        ───────────
  Untracked                        Merge: three-way, preserves history
  ↓ git add                        Rebase: linear, rewrites SHAs
  Staged                           Cherry-pick: selective copy
  ↓ git commit                     Squash: combine commits
  Unmodified
  ↓ edit                           RECOVERY
  Modified                         ────────
  ↓ git add                        Reflog: 90 days of all movements
  Staged                           git reset: soft/mixed/hard
                                   git revert: safe undo for shared
SHA Mechanics                      git restore: file-level recovery
  SHA-1(type+size+content)
  → content-addressable            HISTORY SURGERY
  → deduplication                  ──────────────
  → integrity                      bisect: binary search
  → immutability                   blame: line authorship
                                   log -S: pickaxe search
DAG                                filter-repo: rewrite history
  Commits form directed            interactive rebase: clean up
  acyclic graph
  Branches = pointers              TEAM POLICIES
  HEAD = current pointer           ───────────────
                                   Branch protection rules
                                   Required reviews
                                   Status checks (CI)
                                   Hooks: pre-commit, commit-msg
```

---

## 3. Interview Questions by Seniority Level

### Junior Level (1-2 years)

| Question | Key Points |
|---|---|
| What is Git? | DVCS, tracks changes, local full history |
| What is `git init`? | Creates `.git/`, initialises empty repo |
| Difference: `git add` vs `git commit` | Stage vs permanently record |
| What is a branch? | Lightweight pointer to a commit |
| How do you undo the last commit? | `git reset --soft HEAD~1` |
| What is `.gitignore`? | Patterns for files Git should not track |
| What is `git clone`? | Download entire repo (all history) |
| What is a merge conflict? | Both branches edited same lines |

### Mid-Level (2-5 years)

| Question | Key Points |
|---|---|
| Merge vs Rebase? | Preserves vs rewrites history; golden rule |
| What is the staging area? | Draft area between working dir and commits |
| `git reset` modes? | soft/mixed/hard — what each preserves |
| `git revert` vs `git reset`? | Safe (new commit) vs destructive (rewrite) |
| What is `git stash`? | Temporarily save dirty state |
| What is detached HEAD? | HEAD points to commit, not branch |
| What is `--force-with-lease`? | Safer force push — checks remote state first |
| What is interactive rebase? | Rewrite local history: squash, reorder, drop |

### Senior Level (5+ years)

| Question | Key Points |
|---|---|
| Explain Git's internal object model | Blob, Tree, Commit, Tag; SHA-1; `.git/objects/` |
| How does a commit SHA change? | Parent SHA, author, timestamp, tree SHA all contribute |
| Branching strategies for large teams? | Trunk-Based vs Git Flow — context matters |
| How to find a performance regression? | `git bisect run <script>` |
| How do you handle a compromised secret in history? | Rotate first; filter-repo; force push; re-clone team |
| DAG and its implications? | Merge finds LCA; rebase replays; cherry-pick copies |
| Git hooks for team enforcement? | Pre-commit, commit-msg, pre-push; sharing via hooksPath or Husky |
| Git in a monorepo? | Sparse checkout, shallow clone, partial clone |

### Tech Lead / Architect Level

| Question | Key Points |
|---|---|
| Design a branching strategy for a 50-person team? | Assess release model, deployment frequency, team maturity |
| How to migrate SVN to Git? | `git svn clone`, preserve history, retrain team |
| Git at scale — monorepo challenges? | Partial clone, sparse checkout, CODEOWNERS, virtual filesystems |
| How to enforce commit quality across the org? | Shared hooks, pre-receive server hooks, PR templates, CI checks |
| Disaster recovery with Git? | Mirror repos, bare clones, reflog, ORIG_HEAD |

---

## 4. Scenario-Based Questions

### Scenario 1: "You pushed a bug to main. Production is down. What do you do?"

**Model Answer:**
> First, assess: is this a recent commit that can be reverted cleanly, or a complex change? If recent and cleanly isolated: `git revert <bad-commit-sha>` — this creates a new commit undoing the change, is safe on shared branches, and preserves full history. Commit with a clear message ("revert: payment NPE fix rollback — fixes PROD-521"), push, trigger deployment. If revert is complex or the build takes too long, consider deploying the previous tagged release directly from CI/CD — faster than fixing forward. After recovery: conduct a blameless post-mortem, improve testing to prevent recurrence.

### Scenario 2: "A developer says their local main is 50 commits behind origin/main. How do they catch up?"

**Model Answer:**
> `git fetch origin` to download the latest remote changes (never modifies local branches). Then `git log main..origin/main` to see what commits are coming. Then `git rebase origin/main` (preferred for linear history) or `git merge origin/main`. If their local main has diverged (they committed locally to main — which they shouldn't have), `git pull --rebase` handles it. If they want to cleanly align with remote: `git reset --hard origin/main` (discards any local commits). After this, they should create feature branches for new work rather than committing directly to local main.

### Scenario 3: "A developer accidentally force-pushed and deleted a colleague's commits from the shared feature branch. Recovery?"

**Model Answer:**
> The colleague still has the commits locally. Immediate steps: don't panic, don't pull. The colleague should run `git push --force-with-lease origin feature/shared` to restore their commits (if no one has pulled the broken state yet). If others have pulled the bad state: colleague pushes their version, affected devs run `git fetch; git reset --hard origin/feature/shared`. For future prevention: enable branch protection rules on GitHub to require pull requests and disable force pushes. Educate the team: never `--force` on shared branches; use `--force-with-lease` only on your own branches.

### Scenario 4: "You want to apply only one specific bug fix from a feature branch (which isn't ready to merge) to main for a hotfix. How?"

**Model Answer:**
> `git cherry-pick <commit-sha>` copies the specific fix commit onto main. First, find the commit: `git log feature/payment --oneline` to identify the exact fix SHA. Then on main: `git cherry-pick abc123`. If there are conflicts, resolve them, `git add`, `git cherry-pick --continue`. The result is a new commit on main with the same changes but a different SHA. I'd note in the commit message that this is a cherry-pick from the feature branch (and include the original SHA for traceability). The feature branch keeps the original commit for when it eventually merges.

### Scenario 5: "The team is having too many merge conflicts. What changes would you propose?"

**Model Answer:**
> Frequent conflicts signal process issues. I'd investigate: (1) **Long-lived feature branches** — branches open for 2+ weeks constantly diverge from main. Solution: smaller PRs, feature flags to merge partial work. (2) **Poor code ownership** — multiple people editing the same files. Solution: clearer module boundaries, CODEOWNERS file, avoid cross-cutting changes. (3) **Infrequent syncing** — developers don't pull from main regularly. Solution: enforce `git pull --rebase` from main daily; use `git config pull.rebase true`. (4) **No rebase before PR** — stale PRs conflict on merge. Solution: require branches to be up-to-date before merge (GitHub "Require branches to be up to date" setting). (5) **Monolithic changes** — massive refactors conflict with everything. Solution: parallel refactoring techniques, strangler fig pattern.

---

## 5. DevOps-Specific Questions

### Q: "How does Git fit into a CI/CD pipeline?"

**Model Answer:**
> Git is the trigger for the entire pipeline. A `git push` to a branch fires a webhook to the CI server (GitHub Actions, Jenkins, GitLab CI). The CI system: (1) does a shallow clone (`--depth 1`) for speed, (2) runs the build, (3) runs tests, (4) runs security/lint scans, (5) reports status back to GitHub (commit status API). For trunk-based teams, every push to main triggers deployment to staging. A `git tag -a v2.1.0` triggers a release pipeline. Pull requests have their own CI run — only PRs with passing CI and required reviews can merge (enforced via branch protection rules).

### Q: "How do you version releases with Git?"

**Model Answer:**
> Semantic versioning with annotated tags: `git tag -a v<MAJOR>.<MINOR>.<PATCH> -m "Release message"`. Major = breaking change, Minor = backward-compatible feature, Patch = bug fix. CI/CD reads the tag and uses it as the build version. `git describe --tags` generates version strings like `v1.2.3-14-gabc1234` (14 commits after v1.2.3). Pre-release versions use suffixes: `v2.0.0-rc.1`, `v2.0.0-beta.1`. For automated version bumping, tools like `semantic-release` parse Conventional Commits to automatically determine and apply the next version number and generate changelogs.

### Q: "How do you handle database migrations with Git branches?"

**Model Answer:**
> Database migrations are directional and often irreversible — they need special handling. Approaches: (1) **Sequential numbered migrations** — `V001__create_payment_table.sql`, `V002__add_retry_count.sql`. Tool like Flyway/Liquibase applies them in order. Migrations are committed to the repo with the feature they support. (2) **Branch conflicts** — two developers both create `V003__*.sql`. Solution: migration numbering based on timestamp (`V20260610143022__add_index.sql`) to avoid clashes. (3) **Rollback strategy** — design all migrations with a corresponding rollback script; Liquibase supports rollback natively. (4) **Blue-green deployments** — run database migrations separately from code deploys; ensure backward compatibility for the deployment window.

### Q: "What is sparse checkout and when would you use it?"

**Model Answer:**
> Sparse checkout lets you check out only a subset of a repository's files — useful in monorepos where you only need a specific service or module. `git sparse-checkout init --cone` then `git sparse-checkout set services/payment` checks out only the payment service directory. Combined with shallow clones, this dramatically reduces CI/CD checkout time for large monorepos. Also useful for large repos with generated files or build artifacts in specific directories you never need to modify locally.

---

## 6. Tech Lead / Architecture Questions

### Q: "How would you onboard a 10-person team from SVN to Git?"

**Model Answer:**
> Key phases: (1) **Migration** — use `git svn clone` to import SVN history into Git, preserving all commits with authors/timestamps. Set up remote (GitHub/GitLab). (2) **Branching strategy decision** — for a team new to Git, GitHub Flow is simpler than Git Flow. Define branch naming conventions (feature/, hotfix/, release/). (3) **Training** — focus on mental model changes: local commits are free (no server needed), branches are cheap, commit often, push/PR when ready. Common SVN habits to unlearn: committing directly to trunk, updating before every commit, fear of branching. (4) **Tooling** — configure IDE Git integration, set up global .gitignore, install Git Credential Manager. (5) **Enforcement** — branch protection rules, commit message standards, PR templates. (6) **Gradual rollout** — one team first, then organisation-wide.

### Q: "Design a Git strategy for a SaaS product that needs to support multiple enterprise customers on different versions."

**Model Answer:**
> This requires maintaining multiple stable release lines simultaneously — Git Flow with long-lived release branches is appropriate here. Structure: `main` as the source of truth with all features. `release/v2.x` for major version lines — this branch receives only bug fixes (cherry-picked from main or hotfix branches). Each enterprise customer is pinned to a `release/v2.x` branch. New features land in main, stabilise, then release/v3.x is cut. Hotfixes: fix in the relevant `release/vX.x` branch, then cherry-pick up to main. Tagging: `v2.3.1-enterprise` for specific enterprise releases. CI/CD: each release branch has its own deployment pipeline. The trade-off is significant maintenance overhead — consider whether feature flags plus single-version SaaS is achievable instead.

### Q: "What is the CODEOWNERS file and how does it support team development?"

**Model Answer:**
> `CODEOWNERS` is a GitHub/GitLab feature where you define which team or individual owns which parts of the codebase. Example: `src/payment/ @payment-team`, `*.md @docs-team`. When a PR touches files in a CODEOWNERS path, the listed owners are automatically requested as reviewers. Combined with branch protection rules requiring CODEOWNERS review, this ensures the right experts review changes to critical subsystems. It scales code review responsibility — the payment team reviews payment code, the security team reviews auth code — without requiring team leads to manually assign every PR. It also serves as living documentation of ownership.

---

## 7. Common Traps & Misconceptions

### Trap 1: "Git and GitHub are the same thing."
> Git is the version control tool (CLI). GitHub is a hosting platform and collaboration service. Alternatives to GitHub: GitLab, Bitbucket, Gitea.

### Trap 2: "Deleting a branch deletes its commits."
> Branches are pointers. Deleting a branch removes the pointer; the commits remain in `.git/objects/` until garbage collected (after 30-90 days). Recovery via `git reflog`.

### Trap 3: "`git pull` is safe to run anytime."
> `git pull` = `git fetch` + `git merge`. It can create unwanted merge commits. Use `git pull --rebase` or `git fetch` + explicit rebase/merge.

### Trap 4: "Rebase is always better than merge."
> Rebase rewrites history. On shared branches this breaks teammates' histories. Rule: rebase only local/private commits; merge to integrate into shared branches.

### Trap 5: "`git reset --hard` is recoverable."
> Mostly true — via reflog — but dangerous. Commits not in any branch and not in reflog (older than expiry) are permanently lost after `git gc`.

### Trap 6: "Git stores diffs between commits."
> Git stores **snapshots** (complete file content as blobs). Efficiency comes from re-using identical blobs across commits, not from storing diffs. (Pack files do use delta compression, but this is a storage optimisation, not the logical model.)

### Trap 7: "`git commit -a` stages everything."
> `git commit -a` stages only **tracked modified/deleted** files. It does NOT stage new untracked files. You still need `git add` for new files.

### Trap 8: "Force push is always bad."
> Force push is appropriate on your own private feature branches after rebasing. It's dangerous on shared branches. Use `--force-with-lease` instead of `--force` for added safety.

### Trap 9: "The staging area is just a temporary copy."
> The staging area (`.git/index`) is a precise specification of the next commit. You can have a file simultaneously modified in working directory and staged with older content — `git commit` only takes the staged version.

### Trap 10: "A shallow clone is incomplete."
> A shallow clone has full content for the checked-out branch at the requested depth — all files are present and correct. What's missing is deeper commit history. For building software, shallow clones are completely functional.

---

## 8. Command Decision Trees

### "I want to undo something. Which command?"

```
What do I want to undo?
│
├── Unstage a file (keep changes in working dir)
│   └── git restore --staged <file>
│
├── Discard working directory changes (dangerous — cannot recover)
│   └── git restore <file>
│
├── Undo last commit (keep changes)
│   ├── Keep staged?  → git reset --soft HEAD~1
│   └── Keep unstaged? → git reset --mixed HEAD~1 (default)
│
├── Undo last commit (discard changes) ⚠️
│   └── git reset --hard HEAD~1
│
├── Undo a pushed commit (safe)
│   └── git revert <sha>          ← creates new undo commit
│
└── Recover something I thought was lost
    └── git reflog → git reset --hard <sha>
```

### "I want to integrate changes from another branch."

```
Which branch is the target?
│
├── Shared branch (main, develop)
│   ├── Their feature → ours?     → git merge feature/name (via PR)
│   └── Need specific commit only? → git cherry-pick <sha>
│
└── My private feature branch
    ├── Want linear history?       → git rebase main
    ├── Want to squash my WIPs?   → git rebase -i HEAD~n
    └── Want to sync with remote? → git pull --rebase
```

### "I want to view history."

```
What do I want to see?
│
├── All commits visually          → git log --oneline --graph --all --decorate
├── Commits by author             → git log --author="name"
├── Commits touching a file       → git log -- path/to/file
├── Who changed this line?        → git blame src/File.java
├── Which commit added this code? → git log -S "code string"
├── Find first bad commit         → git bisect start/bad/good/run
└── My recent HEAD movements      → git reflog
```

---

## 9. Git Internals — 60-Second Summary

> Use this as a rapid-fire response to "Explain how Git works internally."

**Git is a content-addressable object store.** Every piece of data is stored as one of four object types in `.git/objects/`, each identified by a SHA-1 hash of its content.

**Blob** = file content only (no name, no path).  
**Tree** = directory listing, mapping names to blob/tree SHAs.  
**Commit** = snapshot pointing to root tree + parent commit SHA + author/message metadata.  
**Tag** = named, annotated reference to a commit.

When you `git add`, Git computes the blob SHA, writes the blob to `.git/objects/`, and records `filepath → SHA` in `.git/index` (the staging area).  
When you `git commit`, Git builds trees from the index, creates a commit object pointing to the root tree, and advances the current branch pointer.  
A **branch** is just a file containing one commit SHA — creating a branch is O(1).  
**HEAD** is a symbolic reference pointing to the current branch (or directly to a commit in detached HEAD state).  
Commits form a **DAG** — a directed acyclic graph where each commit points to its parent(s). Branching creates diverging paths; merging creates commits with two parents; rebasing replays commits on a new base (rewriting SHAs).  
The **reflog** records every position HEAD has occupied — your safety net for recovery.

---

## 10. One-Page Cheatsheet

```
GIT MASTER CHEATSHEET
═══════════════════════════════════════════════════════════════════

SETUP & INIT
  git init                          init new repo
  git clone <url>                   clone remote repo
  git config --global user.name ""  set identity
  git config --global user.email "" set identity

STAGING & COMMITTING
  git status [-s]                   show state
  git add .                         stage all
  git add -p                        interactive hunk staging ⭐
  git commit -m "msg"               commit staged
  git commit --amend                redo last commit
  git diff [--staged]               unstaged [staged] changes

BRANCHING
  git switch -c feature/name        create + switch ⭐
  git switch main                   switch branch
  git branch -vv                    list + tracking info
  git branch -d feature/name        delete merged
  git branch -D feature/name        force delete

MERGING & REBASING
  git merge feature/name            merge into current
  git merge --no-ff feature/name    always merge commit
  git rebase main                   rebase onto main
  git rebase -i HEAD~n              interactive rebase ⭐
  git cherry-pick <sha>             copy specific commit

REMOTE
  git fetch --prune                 fetch + clean refs
  git pull --rebase                 fetch + rebase ⭐
  git push -u origin feature        push + set tracking
  git push --force-with-lease       safer force push

TAGS
  git tag -a v1.0.0 -m "msg"       annotated tag ⭐
  git push origin v1.0.0            push tag

UNDOING
  git restore --staged <file>       unstage
  git restore <file>                discard working dir
  git reset --soft  HEAD~1          undo commit, keep staged
  git reset --mixed HEAD~1          undo commit, keep unstaged
  git reset --hard  HEAD~1          undo commit, DISCARD ⚠️
  git revert <sha>                  safe undo → new commit

RECOVERY
  git reflog                        all HEAD movements ⭐
  git reset --hard HEAD@{n}         restore to reflog state

HISTORY
  git log --oneline --graph --all   visual history ⭐
  git log -S "string"               pickaxe search
  git log -- path/to/file           file history
  git blame src/File.java           line authorship
  git bisect start/bad/good/run     binary search ⭐

INSPECTION
  git cat-file -p <sha>             inspect git object
  git ls-tree HEAD                  list tree
  git ls-files --stage              list index with SHAs
  git fsck --full                   verify integrity

STASH
  git stash [-u]                    stash [+ untracked]
  git stash pop                     apply + remove
  git stash list                    list stashes

═══════════════════════════════════════════════════════════════════
GOLDEN RULES
  1. Never rebase shared branches
  2. Prefer --force-with-lease over --force
  3. Use git revert (not reset) on pushed commits
  4. Small, atomic, well-described commits
  5. Rotate secrets before git cleanup
  6. When in doubt: git reflog
═══════════════════════════════════════════════════════════════════
```

---

## 11. Topics to Study Beyond This Series

### Git at Scale
- **Partial Clone** (`git clone --filter=blob:none`) — clone without downloading file content up front
- **Sparse Checkout** — only checkout specific directories in monorepos
- **Git Virtual File System (GVFS)** — Microsoft's solution for very large repos (Windows codebase)
- **Scalar** — built-in Git monorepo performance tool (ships with Git 2.38+)

### Specialised Workflows
- `git submodule` deep dive — complex scenarios and pitfalls
- `git subtree` — alternative to submodules (no separate repo required)
- `git bundle` — offline transfer of Git history

### Security
- **Signed Commits** — GPG signing (`git commit -S`) for commit authenticity
- **Signed Tags** — `git tag -s` for verified releases
- **GitHub's secret scanning** — automated detection of committed credentials
- **SBOM generation** from Git history

### Tooling
- **git-delta** — beautiful diff viewer
- **lazygit** — terminal UI for Git
- **gh CLI** — GitHub's official CLI (manage PRs, releases from terminal)
- **pre-commit** — framework for managing Git hooks
- **Conventional Commits + semantic-release** — automated versioning from commit messages
- **git-crypt** — transparent encryption of files in Git repos

### Platform-Specific
- **GitHub Actions** — CI/CD with Git events as triggers
- **GitHub Protected Branches** — enforce PR reviews, status checks
- **GitHub CODEOWNERS** — automatic review assignment
- **GitLab CI/CD** — `.gitlab-ci.yml` pipeline configuration
- **Gitea/Forgejo** — self-hosted Git platforms

---

> **Start of Series:** [01 · Git Introduction & Setup](./01_Git_Setup_and_Introduction.md)

---

*This knowledge base was built from first-principles notes enhanced for senior engineering interviews.*  
*Topics: Version Control · Git Architecture · File Lifecycle · Branching · Merging · Remote Collaboration · Advanced Commands · Production Workflows*
