# 02 · Git Architecture

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [01 · Git Introduction & Setup](./01_Git_Setup_and_Introduction.md)

---

## Table of Contents

1. [The Big Picture — How Git Stores Data](#1-the-big-picture--how-git-stores-data)
2. [Object Type 1 — Blob](#2-object-type-1--blob)
3. [Object Type 2 — Tree](#3-object-type-2--tree)
4. [Object Type 3 — Commit](#4-object-type-3--commit)
5. [Object Type 4 — Tag (Annotated)](#5-object-type-4--tag-annotated)
6. [The Object Database — `.git/objects/`](#6-the-object-database--gitobjects)
7. [SHA-1 Hashing — Content-Addressable Storage](#7-sha-1-hashing--content-addressable-storage)
8. [The DAG — Directed Acyclic Graph](#8-the-dag--directed-acyclic-graph)
9. [HEAD, Branches, and References](#9-head-branches-and-references)
10. [Pack Files — Optimised Storage](#10-pack-files--optimised-storage)
11. [Interview Questions & Model Answers](#11-interview-questions--model-answers)
12. [Quick Revision Cheatsheet](#12-quick-revision-cheatsheet)
13. [Real-World Implications of Git's Architecture](#13-real-world-implications-of-gits-architecture)

---

## 1. The Big Picture — How Git Stores Data

### Snapshots, Not Diffs

Most people assume Git stores *diffs* (like `patch` files). This is wrong.

> **Git stores the complete snapshot of every file at each commit — not a delta from the previous version.**

However, Git is smart: if a file is unchanged between two commits, the new commit simply **re-uses the same blob object** (via the same SHA hash pointer) rather than duplicating data.

```
Commit C1:
  tree → blob(Main.java v1)  ← new blob
         blob(Payment.java v1) ← new blob

Commit C2:  (only Payment.java changed)
  tree → blob(Main.java v1)    ← SAME blob pointer (no duplication!)
         blob(Payment.java v2) ← new blob
```

This is why Git is extremely fast at branching and why storage is surprisingly efficient.

### Four Object Types

```
┌─────────────────────────────────────────────────────┐
│              Git Object Store (.git/objects/)        │
│                                                     │
│   BLOB        TREE        COMMIT       TAG           │
│  (file       (folder     (snapshot    (named        │
│  content)    structure)  metadata)    commit)        │
└─────────────────────────────────────────────────────┘
```

Every object is:
- **Immutable** — never modified after creation
- **Content-addressed** — identified by SHA-1 hash of its content
- **Stored in** `.git/objects/`

---

## 2. Object Type 1 — Blob

### What is it?

A **blob** (Binary Large Object) stores the **raw content of a single file**. Nothing else — no file name, no permissions, no path.

### What a Blob Does NOT Store

| Information | Stored By |
|---|---|
| File name | Tree |
| File path | Tree |
| Directory structure | Tree |
| Permissions | Tree |
| Timestamps | Commit |

### How a Blob is Created

```
File: Main.java
Content:
  package com.concepts;
  public class Main {
      public static void main(String[] args) {
          System.out.println("Hello world!");
      }
  }
```

Git computes:

```
Key = SHA-1( "blob" + " " + content_size_in_bytes + "\0" + content )
    = SHA-1( "blob 100\0package com.concepts;\n..." )
    = 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c
```

Value = raw bytes of file content (zlib-compressed)

```
Result stored in:
.git/objects/42/f9a3297b7e90d22ef17e255c4aca705b4a6f4c
              ↑↑ ↑───────────────────────────────────
          folder  filename (remaining 38 chars)
          (first 2 chars)
```

### Why Split into 2+38?

If all objects were in one directory, with millions of files, the filesystem would slow down significantly (slower `readdir`, slower metadata operations). Splitting into 256 subdirectories (00–ff) keeps each directory small.

### Inspecting a Blob

```bash
# Pretty-print blob content
git cat-file -p 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c

# Show object type
git cat-file -t 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c
# blob

# Show object size (bytes)
git cat-file -s 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c

# Hash a file WITHOUT writing to object store (dry run)
git hash-object Main.java

# Hash AND write to object store
git hash-object -w Main.java
```

### Key Property: Content-Identical Files Share One Blob

```
README.md  → "Hello World"  → SHA: abc123
LICENSE.md → "Hello World"  → SHA: abc123  ← SAME blob!
```

Two files with identical content share a single blob. Git deduplicates automatically.

---

## 3. Object Type 2 — Tree

### What is it?

A **tree** object represents a **directory**. It stores:
- A list of blob pointers (files) and their names/permissions
- Pointers to other tree objects (sub-directories)

### Tree Structure (Recursive)

```
Parent Folder/
├── README.md
└── src/
    └── Main.java
```

Git creates:

```
Tree (root/parent folder)
  → 100644 blob 7x0e745c4... README.md
  → 040000 tree a1b2c3d4e... src

Tree (src/)
  → 100644 blob 9d8e7f6c... Main.java

Blob (README.md content)
Blob (Main.java content)
```

### File Mode Numbers

| Mode | Type | Meaning |
|---|---|---|
| `100644` | blob | Regular file |
| `100755` | blob | Executable file |
| `120000` | blob | Symbolic link |
| `040000` | tree | Directory |
| `160000` | commit | Git submodule |

### Tree SHA Computation

```
Key = SHA-1( "tree" + " " + size + "\0" + entries )
```

Where each entry is: `<mode> <name>\0<sha_bytes>`

Because tree content includes child SHAs, if ANY file changes anywhere in the hierarchy, the tree SHA changes all the way up to the root tree. This is how Git knows the entire project state changed.

### Inspecting a Tree

```bash
# List top-level tree of HEAD
git ls-tree HEAD

# Recursive listing (all files)
git ls-tree -r HEAD

# List tree of a specific commit
git ls-tree a1b2c3d4

# Show only names
git ls-tree --name-only HEAD

# Example output:
# 100644 blob 7x0e745c4...  README.md
# 040000 tree a1b2c3d4e...  src
```

---

## 4. Object Type 3 — Commit

### What is it?

A **commit** is Git's snapshot of the project at a specific point in time. It contains:

| Field | Description |
|---|---|
| `tree` | SHA of the root tree object (entire project state) |
| `parent` | SHA of parent commit(s) — empty for first commit, two for merge |
| `author` | name + email + timestamp (who wrote the changes) |
| `committer` | name + email + timestamp (who applied the commit) |
| `message` | commit message |

### Commit SHA Computation

```
Key = SHA-1( "commit" + " " + size + "\0" + commit_content )

commit_content =
  tree   f9e8d7c6...
  parent c1a2b3c4...         ← absent for first commit
  author Shrayansh <...> 1746000000 +0530
  committer Shrayansh <...> 1746000000 +0530
  
  first commit

= c1a2b3c4d5e6...
```

### Why Author ≠ Committer?

```
Open-source workflow:
  Contributor (Author)  → writes patch, sends PR
  Maintainer (Committer) → reviews, applies patch with git am / git cherry-pick
```

On personal projects they're usually identical. In large open-source projects they differ.

### The Commit Chain (DAG)

```
C1 ← C2 ← C3 ← C4 (HEAD → main)
```

Each commit points to its **parent**. This forms a **Directed Acyclic Graph**:
- Directed: edges go from child to parent (newest to oldest)
- Acyclic: no cycles (you cannot be your own ancestor)

### Inspecting a Commit

```bash
# View commit object content
git cat-file -p HEAD
# tree   f9e8d7c6...
# parent c1a2b3c4...
# author Shrayansh <...> 1746000000 +0530
# committer Shrayansh <...> 1746000000 +0530
#
# first commit

# Show commit type
git cat-file -t HEAD
# commit

# Pretty log
git log --oneline --graph --all --decorate

# Full diff for a commit
git show <commit-sha>

# List files changed in a commit
git diff-tree --no-commit-id -r --name-only <commit-sha>
```

### What Makes a Commit SHA Change?

Since SHA is derived from the content, changing ANY of these changes the commit SHA:

- The commit message
- The author/committer name or email
- The timestamp
- The parent commit SHA
- The root tree SHA (i.e., any file anywhere in the project)

> **Interview Insight:** This is why `git rebase` produces **new commits** with different SHAs even if the code changes are identical — the parent SHA is different, so the commit SHA is different. This is also why rewriting history (amend, rebase, filter-branch) breaks history for anyone who already has the old SHAs.

---

## 5. Object Type 4 — Tag (Annotated)

### What is it?

An **annotated tag** is a full Git object (unlike lightweight tags which are just ref pointers). It stores:
- The SHA of the tagged object (usually a commit)
- Tagger name, email, timestamp
- Tag message
- Optional GPG signature

```bash
# Create annotated tag
git tag -a v1.0.0 -m "Release version 1.0.0"

# Inspect tag object
git cat-file -p v1.0.0
# object  c1a2b3c4...   ← the commit SHA
# type    commit
# tag     v1.0.0
# tagger  Shrayansh <...> 1746000000 +0530
#
# Release version 1.0.0

# Lightweight tag (NOT an object — just a pointer)
git tag v1.0.0-light   ← just a file in .git/refs/tags/
```

### Annotated vs Lightweight Tags

| Feature | Annotated Tag | Lightweight Tag |
|---|---|---|
| Git object? | Yes | No (just a ref file) |
| Has message? | Yes | No |
| Has tagger info? | Yes | No |
| Can be GPG-signed? | Yes | No |
| Appears in `git describe`? | Yes | With `--tags` flag |
| **Use for releases?** | **Yes** | No |

---

## 6. The Object Database — `.git/objects/`

### Layout

```
.git/objects/
├── 42/
│   └── f9a3297b7e90d22ef17e255c4aca705b4a6f4c   ← blob (Main.java)
├── 39/
│   └── ed3cc421c9137fa866cf441d0545c5c00b118c   ← blob (Payment.java)
├── 7c/
│   └── ...                                       ← tree (src/)
├── f9/
│   └── ...                                       ← tree (root)
├── c1/
│   └── ...                                       ← commit
├── info/
└── pack/
    ├── pack-xxx.idx
    └── pack-xxx.pack
```

### Storage Format

All objects are **zlib-compressed** before storage. This is why you cannot `cat` them directly — they're binary. Use `git cat-file` instead.

### Checking Object Existence

```bash
# Check if an object exists
git cat-file -e <sha> && echo "exists"

# Find all objects
git rev-list --objects --all | head -20

# Count all objects
git count-objects -v
```

---

## 7. SHA-1 Hashing — Content-Addressable Storage

### The Formula

```
SHA-1( object_type + " " + byte_size + null_byte + content )
```

| Object | Header |
|---|---|
| Blob | `blob <size>\0` |
| Tree | `tree <size>\0` |
| Commit | `commit <size>\0` |
| Tag | `tag <size>\0` |

### Why SHA-1?

Git uses SHA-1 to:
1. **Deduplicate** — same content always produces same hash → no storage duplication
2. **Verify integrity** — any bit flip changes the hash → corruption is immediately detectable
3. **Reference objects** — the hash IS the address in the object store

### SHA-1 Collision Concerns (2017+)

In 2017, Google demonstrated a SHA-1 collision (SHAttered attack). Git's response:
- Git 2.13+ added collision detection
- The `sha256` object format is available in experimental Git versions (`git init --object-format=sha256`)
- GitHub, GitLab, and most large repos are migrating toward SHA-256 object format

> **Interview Insight:** The practical risk of SHA-1 collisions in Git is extremely low for normal usage. The collision attack requires enormous compute resources to craft a malicious file with a matching SHA-1, and Git has additional safeguards. But production-grade organisations increasingly prefer SHA-256 format for new repos.

### Verifying Data Integrity

```bash
# Verify entire repository integrity
git fsck --full

# Verify without progress output (CI-friendly)
git fsck --no-progress

# Example output:
# Checking object directories: 100%
# Checking connectivity: done.
```

---

## 8. The DAG — Directed Acyclic Graph

### What is it?

Git's commit history is a **DAG** — a graph where:
- **Nodes** = commits
- **Edges** = parent pointers (directed: child → parent)
- **Acyclic** = no commit can be its own ancestor

### Linear History

```
A ← B ← C ← D  (HEAD → main)
```

### History with a Branch and Merge

```
      E ← F  (feature)
     /         \
A ← B ← C ← D ← G  (HEAD → main)
                  ↑
              merge commit (has 2 parents: D and F)
```

### History with Rebase (Linear after rebase)

```
Before rebase:
      E ← F  (feature)
     /
A ← B ← C ← D  (main)

After rebase:
A ← B ← C ← D ← E' ← F'  (feature, rebased)
                  ↑
              new commits (different SHAs even though same changes)
```

### Implications for Git Commands

| Command | Operates on DAG by |
|---|---|
| `git log` | Traversing parent pointers (DFS) |
| `git merge` | Finding common ancestor (LCA), combining |
| `git rebase` | Re-replaying commits on new base |
| `git bisect` | Binary search on DAG |
| `git cherry-pick` | Copying a commit node to another branch |

---

## 9. HEAD, Branches, and References

### HEAD

`HEAD` is a **pointer to the current branch** (or directly to a commit in detached HEAD state).

```bash
cat .git/HEAD
# ref: refs/heads/main   ← symbolic ref (normal state)

# After git checkout <commit-sha>:
cat .git/HEAD
# c1a2b3c4d5e6...        ← direct SHA (detached HEAD)
```

### Branches as Pointers

A branch in Git is simply a **file containing a 40-character SHA-1**:

```bash
cat .git/refs/heads/main
# 32686eaaf60e5e23498b286d23a28a3387241e38   ← SHA of latest commit on main
```

This is why:
- Creating a branch is **O(1)** — just create a small text file
- Deleting a branch doesn't delete commits — just removes the pointer file
- Branches are **cheap** in Git; create them liberally

### After a Commit: Branch Pointer Advances

```
Before commit:
  HEAD → main → C3

After git commit:
  HEAD → main → C4  (C4 is new commit, main advances automatically)
```

### Remote-Tracking Branches

```bash
cat .git/refs/remotes/origin/main
# a1b2c3...   ← SHA of main on remote, as of last fetch

# They live in:
.git/refs/remotes/
└── origin/
    ├── main
    └── develop
```

Remote-tracking branches are **read-only locally** — they update only when you `git fetch`.

### Inspecting References

```bash
# List all refs
git show-ref

# List remote branches
git branch -r

# List all branches (local + remote)
git branch -a

# Show what HEAD points to
git symbolic-ref HEAD          # → refs/heads/main
git rev-parse HEAD             # → full SHA
git rev-parse --short HEAD     # → short SHA
```

---

## 10. Pack Files — Optimised Storage

### The Problem with Loose Objects

Every `git add` creates a new loose object file. After many commits, you can accumulate tens of thousands of loose objects — each a separate file. This is inefficient.

### Pack Files

Git periodically runs **garbage collection** (or you can trigger it manually) to bundle loose objects into **pack files** — compressed binary files with a delta-compressed format.

```bash
# Manually trigger pack file creation
git gc

# Show pack file statistics
git count-objects -v
# count: 12          ← loose objects
# size: 48           ← loose objects size (KB)
# in-pack: 2847      ← objects in pack files
# packs: 1           ← number of pack files
# size-pack: 892     ← pack files size (KB)

# Aggressively optimise (slow but thorough)
git gc --aggressive
```

Pack files use **delta compression** — storing the differences between similar objects (e.g., two versions of a large file). This is where Git actually achieves diff-like storage efficiency.

---

## 11. Interview Questions & Model Answers

### Q1: "How does Git store data internally?"

**Model Answer:**
> Git uses a content-addressable object store in `.git/objects/`. Every piece of data is stored as one of four object types: blob (file content), tree (directory structure), commit (project snapshot + metadata), or tag (named reference). Each object is identified by the SHA-1 hash of its content. This means identical content always produces the same hash, enabling automatic deduplication. Git does not store diffs between file versions — it stores complete snapshots, but unchanged files re-use existing blob objects so storage is efficient.

### Q2: "What is the difference between a blob and a tree in Git?"

**Model Answer:**
> A blob stores the raw content of a single file — no filename, no path, no permissions. A tree stores a directory listing: it maps filenames and permissions to blob SHAs (for files) and other tree SHAs (for subdirectories). This separation allows Git to deduplicate content across filenames — if two files have identical content, they share one blob. The tree provides the naming and structural context that the blob deliberately lacks.

### Q3: "What is a commit object and what does it contain?"

**Model Answer:**
> A commit object contains: (1) the SHA of the root tree — representing the complete project state; (2) parent commit SHA(s) — zero for the first commit, one for normal commits, two for merge commits; (3) author name/email/timestamp; (4) committer name/email/timestamp; and (5) the commit message. The commit SHA is a hash of all these fields, which means any change to any field — including the timestamp or parent SHA — produces a completely different commit SHA. This is why rebasing produces new commits even with identical code changes.

### Q4: "What is HEAD in Git?"

**Model Answer:**
> HEAD is a special pointer in `.git/HEAD` that tells Git which commit is currently checked out. Normally it's a symbolic reference pointing to a branch name (`ref: refs/heads/main`), meaning it indirectly points to whatever commit that branch points to. When you make a new commit, the current branch pointer advances to the new commit, and since HEAD points to the branch, HEAD also advances. If you checkout a specific commit SHA directly (`git checkout <sha>`), HEAD detaches from any branch and points directly to that commit — this is called "detached HEAD state." In detached HEAD state, new commits you make are not on any branch and can be garbage-collected.

### Q5: "Why is Git branching so much faster than SVN branching?"

**Model Answer:**
> In SVN, creating a branch copies files on the server — an O(n) operation proportional to the project size. In Git, a branch is simply a 41-byte file containing a SHA-1 hash of a commit. Creating a branch is an O(1) operation — Git just writes one small file. Similarly, switching branches in Git is fast because Git replays the difference between the two tree snapshots rather than checking out files from a server. This fundamental architectural difference is why Git enables feature branching as a standard workflow, whereas SVN branching was expensive enough that most teams avoided it.

### Q6: "What is a detached HEAD state and how do you fix it?"

**Model Answer:**
> Detached HEAD occurs when HEAD points directly to a commit SHA rather than to a branch name. Common causes: `git checkout <commit-sha>`, `git checkout <tag>`, or checking out a remote branch directly. Any commits you make in this state are not attached to any branch — they exist but are unreachable from any branch and will be garbage-collected. To fix it: if you want to keep the work, create a new branch at that point with `git switch -c new-branch-name`. If you don't need the commits, just `git switch main` to return to your branch.

### Q7: "How does Git ensure data integrity?"

**Model Answer:**
> Git uses SHA-1 hashing on every object. The SHA is derived from the content itself, so any corruption — even a single bit flip — produces a different hash. When Git reads an object, it re-computes the SHA and compares it against the filename (which IS the SHA). A mismatch means corruption. You can verify an entire repository with `git fsck --full`. Additionally, since commit objects include the parent commit SHA and the root tree SHA, any tampering with history is detectable — modifying a single commit changes its SHA, which breaks the chain (all descendant commits would have a different parent SHA and thus different SHAs themselves).

---

## 12. Quick Revision Cheatsheet

```bash
# ─── INSPECT OBJECTS ────────────────────────────────────
git cat-file -t <sha>          # type of object (blob/tree/commit/tag)
git cat-file -p <sha>          # pretty-print object content
git cat-file -s <sha>          # object size in bytes
git hash-object <file>         # compute SHA without storing
git hash-object -w <file>      # compute SHA AND store object

# ─── INSPECT TREE / INDEX ───────────────────────────────
git ls-tree HEAD               # list root tree of HEAD
git ls-tree -r HEAD            # recursive: all files
git ls-files --stage           # list staging area with SHAs

# ─── INSPECT COMMITS ────────────────────────────────────
git log --oneline --graph --all --decorate
git show <sha>                 # diff + metadata for a commit
git rev-parse HEAD             # full SHA of HEAD
git rev-parse --short HEAD     # short SHA
git cat-file -p HEAD           # raw commit object

# ─── INSPECT HEAD & REFS ────────────────────────────────
cat .git/HEAD                  # see what HEAD points to
git symbolic-ref HEAD          # branch HEAD points to
git show-ref                   # all refs + SHAs
git branch -a                  # all branches

# ─── REPOSITORY HEALTH ──────────────────────────────────
git fsck --full                # verify integrity
git count-objects -v           # object count + sizes
git gc                         # garbage collect + pack

# ─── OBJECT FORMULA REMINDER ────────────────────────────
# SHA-1("blob " + size + "\0" + content)   → blob
# SHA-1("tree " + size + "\0" + entries)   → tree
# SHA-1("commit " + size + "\0" + content) → commit
```

---

## 13. Real-World Implications of Git's Architecture

### 1. Why Rebasing is "Dangerous" on Shared Branches

Because every commit's SHA is derived from its parent's SHA, rebasing rewrites ALL descendent commit SHAs. If teammates have already pulled those commits, their history diverges — force-push (`git push --force`) is required, which can cause data loss for others.

**Rule:** Never rebase commits that have been pushed to a shared branch.

### 2. Why `git gc` Matters in Large Repos

Loose objects accumulate with every `git add`. In repos with long histories (monorepos, game studios with large assets), garbage collection is critical to keep the repo size manageable.

```bash
# On large repos, schedule this in CI or cron
git gc --auto           # Git auto-runs this when thresholds are met
git gc --aggressive     # Deeper compression (slower)
git prune               # Remove unreachable objects
```

### 3. Why Deleting a Branch Doesn't Delete Commits

A branch is just a pointer. Deleting it removes the pointer, not the commits. Commits without any branch or tag pointing to them become **unreachable** and are collected by `git gc` after a grace period (default: 30 days via `gc.pruneExpire`).

```bash
# Recover commits after accidental branch delete
git reflog             # find the SHA of the deleted branch tip
git switch -c recovered-branch <sha>   # recreate branch
```

### 4. `.gitignore` vs Untracked vs Ignored

```
File States:
  Untracked  → Git sees it, not managing it
  Ignored    → Git completely ignores it (listed in .gitignore)
  Tracked    → Git manages it (staged or committed)
```

```bash
# Check why a file is ignored
git check-ignore -v <filename>

# Show all ignored files
git status --ignored

# Add to gitignore AFTER accidentally tracking it
echo "secrets.env" >> .gitignore
git rm --cached secrets.env    # remove from tracking (keeps file on disk)
git commit -m "stop tracking secrets.env"
```

### 5. The Reflog — Your Safety Net

The reflog records every position HEAD has been at, including deleted branches and amended commits.

```bash
git reflog              # all HEAD movements
git reflog show main    # movements of main branch specifically

# Recover anything from last 90 days (default expiry)
git checkout <sha-from-reflog>
```

---

> **Previous:** [01 · Git Introduction & Setup](./01_Git_Setup_and_Introduction.md)  
> **Next:** [03 · Git File Lifecycle](./03_Git_File_Lifecycle.md)
