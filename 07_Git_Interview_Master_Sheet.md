# 07 · Git Interview Master Sheet

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead — Final interview prep  
> **Series:** Git Knowledge Base — File 7 of 12

---

## Table of Contents
1. [The 10 Must-Know Concepts](#1-the-10-must-know-concepts)
2. [Internals — 60-Second Summary](#2-internals--60-second-summary)
3. [Interview Questions by Level](#3-interview-questions-by-level)
4. [Scenario-Based Questions](#4-scenario-based-questions)
5. [Command Decision Trees](#5-command-decision-trees)
6. [Common Traps & Misconceptions](#6-common-traps--misconceptions)
7. [One-Page Cheatsheet](#7-one-page-cheatsheet)
8. [Topics to Study Beyond This Series](#8-topics-to-study-beyond-this-series)

---

# 1. The 10 Must-Know Concepts

## What is it?
The ten Git concepts that appear most frequently in senior and tech lead interviews, each requiring knowledge of the **what, why, and internal mechanism**.

## Why It Matters
Interviewers distinguish seniors from juniors by depth: a junior knows commands, a senior knows internals and tradeoffs.

## Internal Working
*(Refer to individual topic files for full internal working of each concept)*

## Command Explanation

### Syntax
*(Refer to individual topic files — this section consolidates the concepts)*

| # | Concept | Depth Required |
|---|---------|---------------|
| 1 | **Git Object Model** | Blob, Tree, Commit, Tag — SHA-1, content-addressable storage |
| 2 | **The Three Areas** | Working Directory, Staging Area (index), Local Repository |
| 3 | **File Lifecycle** | Untracked → Staged → Unmodified → Modified + all transitions |
| 4 | **Merge vs Rebase** | When to use each; golden rule; DAG implications |
| 5 | **Branching Strategies** | Git Flow vs GitHub Flow vs Trunk-Based — pros/cons/when |
| 6 | **HEAD and References** | Symbolic refs, detached HEAD, remote-tracking refs |
| 7 | **Reflog** | Recovery tool; 90-day window; what it records |
| 8 | **Force Push Safety** | `--force-with-lease` vs `--force`; when each is appropriate |
| 9 | **Conflict Resolution** | Three-way merge; stage numbers; `--ours` vs `--theirs` |
| 10 | **History Rewriting Safety** | When safe; shared branch golden rules; filter-repo |

---

# 2. Internals — 60-Second Summary

## What is it?
A rapid-fire mental model for explaining Git internals in any interview context.

## Why It Matters
Interviewers at senior level will ask "explain how Git works internally" — this is the model answer structure.

## Internal Working
```
Git = content-addressable object store in .git/objects/

FOUR OBJECT TYPES:
  blob   = file content only (no name, no path)
  tree   = directory (maps names + permissions → blob/tree SHAs)
  commit = snapshot (root tree SHA + parent SHA + author + message)
  tag    = named commit reference with metadata

SHA-1(object_type + " " + size + "\0" + content) → 40-char hex address
  → identical content = identical SHA = automatic deduplication
  → any change = different SHA = immutability + integrity

THREE AREAS:
  Working Directory → .git/index (staging) → .git/objects/ (repository)
  git add:    blob written to objects/, filepath→SHA added to index
  git commit: trees built from index, commit object created, branch pointer advances

DAG (Directed Acyclic Graph):
  Commits form a DAG — each points to parent(s)
  Branch = 41-byte file containing one commit SHA (O(1) to create)
  HEAD   = pointer to current branch (or detached = direct SHA)
  Merge  = new commit with TWO parents
  Rebase = replay commits on new base (new SHAs, same changes)

RECOVERY:
  reflog = 90-day log of all HEAD movements → almost nothing is permanently lost
```

## Command Explanation

### Syntax
```bash
# Verify your understanding of internals
git cat-file -p HEAD              # see raw commit object
git cat-file -p HEAD^{tree}       # see root tree
git ls-files --stage              # see index contents
cat .git/HEAD                     # see HEAD pointer
git log --oneline --graph --all   # see the DAG
```

---

# 3. Interview Questions by Level

## What is it?
Categorised questions with model answers for Junior, Mid-Level, Senior, and Tech Lead interview levels.

## Why It Matters
Knowing the level-appropriate depth prevents over-explaining to juniors or under-explaining to seniors.

## Internal Working
*(No Git internals — this section is question/answer pairs)*

## Command Explanation

### Syntax
*(Model answers reference commands from previous files)*

**JUNIOR LEVEL (1–2 years):**

Q: "What is Git and what is it used for?"
> Git is a distributed version control system that tracks changes to files over time. It lets multiple developers collaborate on the same codebase by providing branching, merging, history tracking, and the ability to revert to any past state.

Q: "What is the difference between `git add` and `git commit`?"
> `git add` moves changes to the Staging Area (index) — it prepares changes to be committed. `git commit` permanently records the staged snapshot into the Local Repository. Nothing is committed until you explicitly run `git commit`.

Q: "How do you undo the last commit without losing your changes?"
> `git reset --soft HEAD~1` — this moves the branch pointer back one commit but keeps all changes staged in the index.

---

**MID-LEVEL (2–5 years):**

Q: "What is the difference between merge and rebase?"
> Merge creates a merge commit with two parents, preserving the full non-linear branch history. Rebase replays commits on a new base, rewriting their SHAs to produce a linear history. Merge is safe on shared branches. Rebase must only be used on local/private branches because rewriting SHAs breaks history for teammates who already have those commits.

Q: "What is `git reset --soft`, `--mixed`, and `--hard`?"
> All three move the branch pointer backwards. `--soft` keeps changes staged. `--mixed` (default) keeps changes unstaged in working directory. `--hard` discards all changes in both index and working directory. Use `--hard` with extreme caution — only recoverable via reflog.

Q: "What is the staging area and why does it exist?"
> The staging area (`.git/index`) is an intermediate area between the Working Directory and Local Repository. It lets you craft atomic, precise commits — stage some files but not others, or stage specific lines within a file using `git add -p`. This enables clean single-purpose commits even when your working directory has multiple unrelated changes.

---

**SENIOR LEVEL (5+ years):**

Q: "Explain Git's internal object model."
> Git stores everything as one of four object types in `.git/objects/`: blob (file content only), tree (directory — maps names to blob/tree SHAs), commit (snapshot pointing to root tree + parent commit + metadata), and tag (named annotated reference). Each is identified by SHA-1 of its content — content-addressable storage. Identical content → identical SHA → deduplication. Any change → different SHA → immutability and integrity detection.

Q: "Why does rebasing create new commits even when the code changes are identical?"
> A commit's SHA is computed from ALL its content including the parent SHA. When you rebase, commits are replayed onto a new base — the parent SHA changes. Different parent → different commit content → different SHA. The code diff may be identical but the commit IS a different object.

Q: "How would you find a performance regression introduced sometime in the last 500 commits?"
> `git bisect run <benchmark-script>`. Mark current as bad, last known-good tag as good. Write a script that exits 0 if performance is acceptable, non-zero if degraded. `git bisect run ./benchmark.sh` automatically binary-searches — ~9 commits tested for 500 history. Git announces the first bad commit.

---

**TECH LEAD LEVEL:**

Q: "What branching strategy would you recommend for a 50-engineer team doing 10 deploys per day?"
> Trunk-Based Development with feature flags. At 50 engineers with 10 daily deploys, any long-lived branches create conflict overhead and slow delivery. Short-lived feature branches (< 2 days) merge to main behind feature flags. CI must be fast and reliable — every merge to main is deployable. Branch protection enforces: 2 reviewers, all CI checks pass, CODEOWNERS approval for domain-critical paths.

Q: "How do you handle a compromised secret in Git history?"
> Immediate: rotate/revoke the credential — assume compromised. Git cleanup: `git filter-repo --path secrets.env --invert-paths`, force-push all branches/tags, notify all teammates to re-clone, GitHub support cache purge for public repos. Prevention: add to `.gitignore`, add pre-commit hook and CI secret scanning (truffleHog), migrate to a secrets manager (Vault, AWS Secrets Manager).

---

# 4. Scenario-Based Questions

## What is it?
Real-world incident and problem scenarios that test practical Git judgment, not just command knowledge.

## Why It Matters
Senior interviews favor scenario questions because they reveal actual decision-making process, not memorized answers.

## Internal Working
*(No Git internals — scenario analysis)*

## Command Explanation

### Syntax
*(Commands vary per scenario — shown inline)*

**Scenario 1: "You pushed a bug to main. Production is down."**
```bash
# Option A: Revert (SAFE — new commit, history preserved)
git revert <bad-commit-sha>       # creates undo commit
git push origin main
# → tag + redeploy

# Option B: Deploy previous tag (fastest)
# Trigger CI/CD to deploy previous release tag — no git needed

# Option C: Reset main (LAST RESORT — notify team immediately)
git reset --hard v2.0.9
git push --force-with-lease origin main
# Everyone must: git fetch && git reset --hard origin/main
```

**Scenario 2: "Developer accidentally deleted a branch with 2 weeks of unmerged work."**
```bash
git reflog --all | grep "feature/big-feature"
# xyz789 refs/heads/feature/big-feature@{0}: commit: last commit

git switch -c feature/big-feature xyz789    # recreate branch ✓
```

**Scenario 3: "Two developers edited the same file — merge conflict in CI."**
```bash
git fetch origin
git rebase origin/main          # surface conflicts locally
# resolve conflicts per commit
git add <resolved-files>
git rebase --continue
git push --force-with-lease     # update PR branch
# → CI reruns on resolved branch
```

**Scenario 4: "A 3-month-old feature branch has 200 WIP commits. How to open a clean PR?"**
```bash
git fetch origin
git rebase -i origin/main       # rebase onto current main first
# In editor: fixup all WIP commits into logical atomic commits
# Each commit = one concern: feat, test, docs, config
git push --force-with-lease origin feature/big-feature
# → Open PR with clean, reviewable commits
```

---

# 5. Command Decision Trees

## What is it?
Decision flowcharts for the most common Git dilemmas: "I want to undo something" and "I want to integrate changes."

## Why It Matters
Interviewers often present "what would you do if..." scenarios — decision trees make the answer instant and systematic.

## Internal Working
*(No internals — decision logic)*

## Command Explanation

### Syntax

**"I want to undo something:"**
```
What do I want to undo?
│
├── Unstage a file (keep working dir changes)
│   └── git restore --staged <file>
│
├── Discard working dir changes for a file ⚠️
│   └── git restore <file>
│
├── Undo last commit — keep staged
│   └── git reset --soft HEAD~1
│
├── Undo last commit — keep unstaged
│   └── git reset --mixed HEAD~1
│
├── Undo last commit — DISCARD ALL ⚠️
│   └── git reset --hard HEAD~1
│
├── Undo a pushed commit (SAFE for shared branches)
│   └── git revert <sha>
│
└── Recover something "lost"
    └── git reflog → git reset --hard <sha>
```

**"I want to integrate changes from another branch:"**
```
Target branch is...
│
├── Shared (main, develop) ← NEVER rebase
│   ├── Their feature → us  → git merge feature/name (via PR)
│   └── Need specific commit → git cherry-pick <sha>
│
└── My private feature branch
    ├── Want linear history?    → git rebase main
    ├── Want to clean WIP commits? → git rebase -i HEAD~n
    └── Want to sync with remote?  → git pull --rebase
```

---

# 6. Common Traps & Misconceptions

## What is it?
The most frequently wrong answers and misunderstandings seen in Git interviews and on teams.

## Why It Matters
Knowing the traps prevents you from saying something wrong in an interview or on the job.

## Internal Working
*(No Git internals — conceptual corrections)*

## Command Explanation

### Syntax
*(Commands shown inline with each trap)*

```
TRAP 1: "Git and GitHub are the same thing."
TRUTH: Git = local CLI tool. GitHub = cloud hosting platform. Separate products.

TRAP 2: "Deleting a branch deletes its commits."
TRUTH: Branches are pointers. Deleting a branch removes the pointer, not the
       commits. They persist in objects/ until gc runs (after 30-90 days).
       Recovery: git reflog → git switch -c branch <sha>

TRAP 3: "git pull is always safe."
TRUTH: git pull = fetch + merge. Default behavior creates merge commits.
       Use: git pull --rebase  OR  git pull --ff-only

TRAP 4: "Rebase is always better than merge."
TRUTH: Rebase rewrites SHAs. On shared branches this breaks teammates' history.
       Golden rule: rebase only LOCAL/PRIVATE commits.

TRAP 5: "git reset --hard is unrecoverable."
TRUTH: Mostly recoverable via git reflog within 90 days.
       TRULY unrecoverable: untracked files removed by git clean -f.

TRAP 6: "Git stores diffs between commits."
TRUTH: Git stores complete SNAPSHOTS. Efficiency comes from re-using identical
       blob objects across commits (not from storing diffs).
       Pack files use delta compression for storage — but logical model = snapshots.

TRAP 7: "git commit -a stages everything."
TRUTH: -a stages only TRACKED modified/deleted files.
       New untracked files still need explicit git add.

TRAP 8: "Force push is always bad."
TRUTH: --force-with-lease on your own feature branches after rebasing = correct.
       --force on shared branches = bad.
       Never force push main, develop, or release/* branches.

TRAP 9: "The staging area is just a temporary copy."
TRUTH: The staging area IS the specification of the next commit.
       A file can be simultaneously modified in working dir AND staged with
       older content. git commit takes STAGED version, not working dir version.

TRAP 10: "A shallow clone is incomplete."
TRUTH: A shallow clone has FULL FILE CONTENT for the checked-out state.
       What's missing is deeper COMMIT HISTORY. For building software = fine.
       For git blame / git bisect / git describe = needs fuller history.
```

---

# 7. One-Page Cheatsheet

## What is it?
A complete single-view reference of the most important Git commands grouped by task — for last-minute review before interviews or daily reference.

## Why It Matters
Interviewers often ask "walk me through your typical Git workflow" — this cheatsheet covers it all.

## Internal Working
*(No internals — command reference)*

## Command Explanation

### Syntax
```bash
# ═══════════════════════════════════════════════════════════
# GIT MASTER CHEATSHEET
# ═══════════════════════════════════════════════════════════

# SETUP & INIT
git init                          # init new repo
git clone --depth 1 <url>         # shallow clone (CI)
git config --global user.name "Name"
git config --global pull.rebase true

# STAGING & COMMITTING
git status -sb                    # compact status
git add -p                        # interactive hunk staging ⭐
git diff --staged                 # what will be committed
git commit -m "feat(scope): msg"  # conventional commit
git commit --amend                # fix last commit

# BRANCHING
git switch -c feature/name        # create + switch ⭐
git switch -                      # previous branch
git branch -vv                    # tracking + ahead/behind
git branch -d feature/name        # delete merged branch

# MERGING & REBASING
git merge --no-ff feature/name    # merge with commit
git rebase main                   # rebase onto main
git rebase -i HEAD~n              # interactive rebase ⭐
git cherry-pick <sha>             # copy specific commit

# UNDOING
git restore --staged <file>       # unstage
git restore <file>                # discard working dir ⚠️
git reset --soft  HEAD~1          # undo commit, keep staged
git reset --mixed HEAD~1          # undo commit, keep unstaged
git reset --hard  HEAD~1          # undo commit, DISCARD ⚠️
git revert <sha>                  # safe undo → new commit ⭐
git reset --hard ORIG_HEAD        # undo last merge/rebase

# RECOVERY ← MOST IMPORTANT ⭐
git reflog                        # find ANY lost commit

# REMOTE
git fetch --prune                 # fetch + clean stale refs
git pull --rebase                 # fetch + rebase ⭐
git push -u origin feature/name   # push + set tracking
git push --force-with-lease       # safer force push ⭐

# TAGS
git tag -a v2.1.0 -m "msg"       # annotated tag ⭐
git push origin v2.1.0            # push tag

# HISTORY
git log --oneline --graph --all --decorate  ⭐
git log -S "string"               # pickaxe search ⭐
git blame -L 10,25 src/File.java  # line authorship
git bisect run ./test.sh          # binary search ⭐

# INTERNALS
git cat-file -p HEAD              # raw commit object
git ls-files --stage              # index contents
git fsck --full                   # integrity check

# ═══════════════════════════════════════════════════════════
# GOLDEN RULES
# 1. Never rebase shared branches (main, develop, release/*)
# 2. Use --force-with-lease, never --force on shared branches
# 3. Use git revert (not reset) on pushed commits
# 4. Always git clean -n (dry run) before git clean -f
# 5. Rotate secrets BEFORE git filter-repo cleanup
# 6. When in doubt: git reflog
# ═══════════════════════════════════════════════════════════
```

---

# 8. Topics to Study Beyond This Series

## What is it?
Advanced Git topics not covered in this series that appear in principal/staff engineer and architecture-level interviews.

## Why It Matters
These topics signal depth beyond typical senior level and demonstrate infrastructure/platform thinking.

## Internal Working
*(No internals — topic guide)*

## Command Explanation

### Syntax
```bash
# Partial Clone (monorepo optimisation)
git clone --filter=blob:none <url>          # no blobs until needed
git clone --filter=tree:0 <url>             # no trees until needed

# Sparse Checkout (checkout subset of monorepo)
git sparse-checkout init --cone
git sparse-checkout set services/payment libs/shared

# Scalar (built-in monorepo performance tool, Git 2.38+)
scalar clone <url>                          # optimised large-repo clone

# Commit graph (speed up log + merge-base)
git commit-graph write --reachable
git config fetch.writeCommitGraph true

# Git maintenance (background optimisation)
git maintenance start

# Signed commits (GPG verification)
git config --global commit.gpgsign true
git verify-commit HEAD

# SHA-256 object format (future of Git)
git init --object-format=sha256
```

**Topics list:**
- Partial Clone + Sparse Checkout (monorepo at scale)
- Git Virtual File System / GVFS (Microsoft-scale repos)
- Scalar (built-in Git monorepo optimisation)
- GitOps with ArgoCD / Flux
- Semantic Release + Conventional Commits automation
- Git LFS (Large File Storage) for binary assets
- GPG-signed commits and tags for release verification
- CODEOWNERS + GitHub Rulesets for org-level policy
- git bundle (offline repo transfer)
- git subtree (alternative to submodules)
- pre-commit framework (shareable hook management)
- truffleHog / git-secrets (secret scanning)
- BFG Repo Cleaner (fast alternative to filter-repo)

---

> **Start of Series:** [01 · Git Introduction & Setup](./01_Git_Setup_and_Introduction.md)  
> **Next:** [08 · Git Commit Messages & Conventions →](./08_Git_Commit_Messages_and_Conventions.md)
