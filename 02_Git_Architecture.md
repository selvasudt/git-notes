# 02 · Git Architecture

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead  
> **Series:** Git Knowledge Base — File 2 of 12

---

## Table of Contents
1. [Git Object Model](#1-git-object-model)
2. [Blob Object](#2-blob-object)
3. [Tree Object](#3-tree-object)
4. [Commit Object](#4-commit-object)
5. [Tag Object (Annotated)](#5-tag-object-annotated)
6. [SHA-1 Content-Addressable Storage](#6-sha-1-content-addressable-storage)
7. [The DAG — Directed Acyclic Graph](#7-the-dag--directed-acyclic-graph)
8. [HEAD and References](#8-head-and-references)
9. [The Three Areas](#9-the-three-areas)
10. [Interview Questions & Answers](#10-interview-questions--answers)
11. [Quick Revision Cheatsheet](#11-quick-revision-cheatsheet)

---

# 1. Git Object Model

## What is it?
Git's object model is a **content-addressable key-value store** inside `.git/objects/`. Every piece of data — file contents, directory structures, commit snapshots, and release tags — is stored as one of four immutable object types, each identified by a SHA-1 hash of its content.

## Why It Matters
- Understanding object types explains WHY Git commands behave the way they do
- Explains why branching is O(1), why rebasing changes SHAs, why merging finds a common ancestor
- Prerequisite for understanding staging area, commits, history rewriting, and recovery

## Internal Working
```
.git/objects/
  42/f9a3297b7e90d22ef17e255c4aca705b4a6f4c  ← blob
  7c/e3d4b12a...                              ← tree
  c1/a2b3c4d5...                              ← commit
  t9/x8y7z6...                                ← tag
  info/
  pack/
    pack-xxx.idx
    pack-xxx.pack   ← compressed bundle of many objects
```

All objects are:
- **Immutable** — never modified after creation
- **Content-addressed** — SHA is derived from content itself
- **zlib-compressed** — use `git cat-file`, not `cat`, to read them

## Command Explanation

### Syntax
```bash
git cat-file -t <sha>    # show object type
git cat-file -p <sha>    # pretty-print object content
git cat-file -s <sha>    # show object size in bytes
```

```bash
git cat-file -t HEAD              # → commit
git cat-file -p HEAD              # → commit object content
git cat-file -t HEAD^{tree}       # → tree
git fsck --full                   # verify all object integrity
git count-objects -vH             # object count + sizes
```

---

# 2. Blob Object

## What is it?
A **blob** (Binary Large Object) stores the **raw content of a single file**. Nothing else — no filename, no file path, no permissions, no timestamps. Just bytes.

## Why It Matters
- Because blobs store only content (not name), two files with identical content share ONE blob — automatic deduplication
- Understanding blobs explains why renaming a file creates no new blob; only content changes do
- Every `git add` creates a blob — staging area = collection of blob SHAs

## Internal Working
```
File: Main.java
Content: public class Main { ... }

Git computes:
  Key = SHA-1("blob " + content_size + "\0" + content)
      = SHA-1("blob 100\0public class Main { ... }")
      = 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c

Stored at:
  .git/objects/42/f9a3297b7e90d22ef17e255c4aca705b4a6f4c
                ↑↑  ↑─────────────────────────────────
           folder  filename (38 chars)
          (2 chars)

Value = zlib-compressed(raw bytes of file content)
```

**Why split 2 + 38 chars?** With millions of objects in one directory, filesystem operations (readdir, lookup) slow dramatically. 256 subdirectories (00–ff) keep each directory small and fast.

**Deduplication example:**
```
README.md  → "Hello World" → SHA: abc123
LICENSE.md → "Hello World" → SHA: abc123  ← SAME blob, no duplication
```

## Command Explanation

### Syntax
```bash
git hash-object [options] <file>
git cat-file [options] <sha>
```

```bash
# Compute blob SHA without storing
git hash-object Main.java

# Compute AND write blob to object store
git hash-object -w Main.java

# Read blob content
git cat-file -p 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c

# Confirm type is blob
git cat-file -t 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c
# blob

# See blob size
git cat-file -s 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c
```

---

# 3. Tree Object

## What is it?
A **tree** object represents a **directory**. It stores a list of entries mapping filenames and permissions to blob SHAs (for files) and other tree SHAs (for subdirectories).

## Why It Matters
- Trees store what blobs deliberately omit: filename, path, and permissions
- Tree objects are recursive — a tree can point to other trees (subdirectories) and blobs (files)
- When any file changes, the tree SHAs change all the way up to the root tree — this is how Git knows the full project state

## Internal Working
```
Project structure:
  parent-folder/
    README.md
    src/
      Main.java

Git creates:
  Tree (src/):
    100644 blob 9d8e7f6c...  Main.java

  Tree (parent-folder/):
    100644 blob 7x0e745c...  README.md
    040000 tree a1b2c3d4...  src          ← points to src/ tree

Key = SHA-1("tree " + size + "\0" + entries)

Each entry format: <mode> <name>\0<20-byte-SHA>
```

**File mode values:**
| Mode | Meaning |
|------|---------|
| `100644` | Regular file |
| `100755` | Executable file |
| `120000` | Symbolic link |
| `040000` | Directory (subtree) |
| `160000` | Git submodule |

## Command Explanation

### Syntax
```bash
git ls-tree [options] <tree-ish> [path]
```

```bash
# List root tree of HEAD
git ls-tree HEAD

# Recursive — list all files
git ls-tree -r HEAD

# Show only filenames
git ls-tree --name-only HEAD

# List tree of a specific commit
git ls-tree abc123def

# Inspect a specific tree object
git cat-file -p HEAD^{tree}
# 100644 blob 7x0e745c...  README.md
# 040000 tree a1b2c3d4...  src
```

---

# 4. Commit Object

## What is it?
A **commit** object is Git's snapshot of the entire project at a specific point in time. It ties together the root tree (full project state), parent commit(s) (history linkage), and metadata (author, committer, timestamp, message).

## Why It Matters
- Every commit's SHA depends on ALL its fields — changing the message, parent, author, or tree changes the SHA
- This is why `git rebase` produces new commits even with identical code changes (the parent SHA differs)
- The commit chain forms the DAG that powers `git log`, `git bisect`, `git merge`

## Internal Working
```
Commit object content:
  tree   f9e8d7c6...              ← SHA of root tree (full project state)
  parent c1a2b3c4...              ← parent commit SHA (absent for first commit)
  author  Shrayansh <s@co.com> 1746000000 +0530
  committer Shrayansh <s@co.com> 1746000000 +0530
                                  ← BLANK LINE (mandatory)
  feat(payment): add retry logic

Key = SHA-1("commit " + size + "\0" + above_content)

What makes the SHA change:
  - Commit message wording
  - Author/committer name, email, or timestamp
  - Parent commit SHA (so rebasing changes ALL descendant SHAs)
  - Root tree SHA (any file change anywhere)
```

**Author vs Committer:**
```
Author    = who WROTE the change (open-source contributor)
Committer = who APPLIED the change (maintainer who ran git am/cherry-pick)
On personal projects these are always the same person.
```

## Command Explanation

### Syntax
```bash
git commit [options] [-m <message>]
git cat-file -p <commit-sha>
git show [options] <commit-sha>
```

```bash
# Create commit with inline message
git commit -m "feat(payment): add retry logic"

# Open editor for full message with body/footer
git commit

# Commit all tracked modified files (skip staging)
git commit -a -m "fix: correct rounding error"

# Amend last commit (not yet pushed)
git commit --amend -m "corrected message"
git commit --amend --no-edit    # add staged changes, keep message

# Inspect a commit object
git cat-file -p HEAD
# tree   f9e8d7c6...
# parent c1a2b3c4...
# author Shrayansh ...

# Show full diff of a commit
git show abc123

# Show files changed in a commit
git show --stat abc123

# Show commit type
git cat-file -t HEAD
# commit
```

---

# 5. Tag Object (Annotated)

## What is it?
An **annotated tag** is a full Git object (unlike lightweight tags which are just ref pointers). It stores the tagger's name, email, timestamp, a tag message, and optionally a GPG signature, pointing to a specific commit.

## Why It Matters
- Annotated tags are permanent objects in `.git/objects/` — they survive GC and carry metadata
- Always use annotated tags for releases — they record WHO tagged, WHEN, and WHY
- Can be GPG-signed to cryptographically verify release authenticity
- Lightweight tags are just files; annotated tags are objects with history

## Internal Working
```
Annotated tag object content:
  object  c1a2b3c4...        ← SHA of the tagged commit
  type    commit
  tag     v2.1.0
  tagger  CI-Bot <ci@co.com> 1746000000 +0000

  Release v2.1.0 — payment retry feature

Stored at: .git/objects/t9/x8y7z6...

Lightweight tag (NOT an object):
  .git/refs/tags/v2.1.0-light → c1a2b3c4...   (directly the commit SHA)
```

## Command Explanation

### Syntax
```bash
git tag [options] [<tagname>] [<commit>]
```

```bash
# Annotated tag on HEAD (USE FOR RELEASES)
git tag -a v2.1.0 -m "Release v2.1.0"

# Annotated tag on specific commit (retroactive)
git tag -a v2.1.0 abc123 -m "Release v2.1.0"

# GPG-signed tag
git tag -s v2.1.0 -m "Release v2.1.0"

# Lightweight tag (bookmarks only, NOT for releases)
git tag v2.1.0-light

# List tags (SemVer sorted)
git tag -l --sort=-version:refname "v*"

# Inspect tag object
git show v2.1.0
git cat-file -p v2.1.0

# Get commit SHA from tag
git rev-parse v2.1.0^{}

# Push tag to remote
git push origin v2.1.0

# Delete tag
git tag -d v2.1.0                   # local
git push origin --delete v2.1.0     # remote

# Verify GPG signature
git tag -v v2.1.0
```

---

# 6. SHA-1 Content-Addressable Storage

## What is it?
Git uses **SHA-1 hashing** to compute a 40-character hexadecimal identifier for every object, derived entirely from the object's content. The hash IS the address — hence "content-addressable storage."

## Why It Matters
- **Deduplication:** Identical content → identical SHA → one stored object, no duplicates
- **Integrity:** Any bit corruption changes the hash → detected instantly
- **Immutability:** You cannot change an object's content without changing its address
- **History tamper detection:** Changing any commit changes its SHA and ALL descendant commit SHAs

## Internal Working
```
Hash formula for each object type:
  blob:   SHA-1("blob "   + byte_size + "\0" + content)
  tree:   SHA-1("tree "   + byte_size + "\0" + entries)
  commit: SHA-1("commit " + byte_size + "\0" + content)
  tag:    SHA-1("tag "    + byte_size + "\0" + content)

Example:
  SHA-1("blob 100\0package com.concepts;\npublic class Main...")
  = 42f9a3297b7e90d22ef17e255c4aca705b4a6f4c

Storage:
  First 2 chars → subdirectory: .git/objects/42/
  Last 38 chars → filename:     f9a3297b7e90d22ef17e255c4aca705b4a6f4c
```

**SHA-1 collision notes (2017+):** Google demonstrated SHA-1 collision (SHAttered). Git added collision detection. SHA-256 object format available (`git init --object-format=sha256`). Practical risk for normal repos remains very low.

## Command Explanation

### Syntax
```bash
git hash-object [options] <file>
git fsck [options]
```

```bash
# Compute SHA without storing
git hash-object Main.java

# Compute AND store blob
git hash-object -w Main.java

# Verify ALL repository objects for corruption
git fsck --full

# CI-friendly integrity check (no progress output)
git fsck --no-progress

# Count objects (healthy repo check)
git count-objects -vH

# Manually reconstruct SHA (for understanding)
# echo -n "blob 12\0Hello World\n" | sha1sum
```

---

# 7. The DAG — Directed Acyclic Graph

## What is it?
Git's commit history is a **Directed Acyclic Graph (DAG)**: commits are nodes, parent pointers are directed edges (child → parent), and there are no cycles (no commit can be its own ancestor).

## Why It Matters
- `git log` = DFS traversal of the DAG following parent pointers
- `git merge` = find Lowest Common Ancestor (LCA) of two DAG nodes
- `git rebase` = replay commits on a new base node in the DAG
- `git bisect` = binary search on the DAG
- Understanding DAG explains why rebase creates NEW commits (different parent → different SHA)

## Internal Working
```
Linear history DAG:
  A ← B ← C ← D  (HEAD → main)

After feature branch and merge:
       E ← F  (feature)
      /         \
A ← B ← C ← D ← G  (HEAD → main)
                  ↑
              Merge commit: has 2 parents (D and F)

After rebase (feature replayed on D):
A ← B ← C ← D ← E' ← F'  (feature rebased)
                  ↑
    New commits — same changes, different SHAs (parent is now D)
```

## Command Explanation

### Syntax
```bash
git log [options]
git merge-base <commit1> <commit2>
```

```bash
# Visualise the full DAG
git log --oneline --graph --all --decorate

# Find common ancestor of two branches (LCA)
git merge-base main feature/payment

# Show all commits reachable from HEAD
git rev-list HEAD

# Show commits in feature but not in main
git log main..feature/payment --oneline

# Show commits unique to either branch
git log main...feature/payment --oneline
```

---

# 8. HEAD and References

## What is it?
**HEAD** is a special pointer in `.git/HEAD` that tells Git which commit is "currently checked out." It normally points to a branch name (symbolic ref), which in turn points to a commit SHA.

**References (refs)** are human-readable names for commit SHAs: branches, tags, and remote-tracking refs.

## Why It Matters
- When you commit, Git moves the CURRENT BRANCH pointer forward — HEAD follows automatically because it points to the branch
- "Detached HEAD" (HEAD → SHA directly) means new commits are not on any branch and will be garbage collected
- Every branch, tag, and remote-tracking ref is just a text file containing a 40-char SHA

## Internal Working
```
Normal state:
  .git/HEAD = "ref: refs/heads/main"      ← symbolic ref
  .git/refs/heads/main = "32686eaa..."    ← commit SHA

After git switch feature/payment:
  .git/HEAD = "ref: refs/heads/feature/payment"
  .git/refs/heads/feature/payment = "abc123..."

Detached HEAD (git checkout abc123):
  .git/HEAD = "abc123def456..."           ← direct SHA, no branch

After new commit C5 on main:
  .git/refs/heads/main advances to C5's SHA automatically
  HEAD still points to main → HEAD implicitly points to C5

Branch = a text file with 41 bytes (40-char SHA + newline)
Creating a branch = O(1) — just write one small file
```

## Command Explanation

### Syntax
```bash
git symbolic-ref HEAD
git rev-parse HEAD
git show-ref
```

```bash
# See what HEAD points to
cat .git/HEAD
# ref: refs/heads/main

# Get full SHA of HEAD
git rev-parse HEAD
# 32686eaaf60e5e23498b286d23a28a3387241e38

# Get short SHA
git rev-parse --short HEAD

# See branch HEAD points to
git symbolic-ref HEAD
# refs/heads/main

# See branch HEAD points to (name only)
git symbolic-ref --short HEAD
# main

# List all refs with SHAs
git show-ref

# List local branches with SHAs + messages
git branch -v

# List all branches + remote tracking
git branch -a

# Escape detached HEAD by creating a branch
git switch -c recovery-branch     # creates branch at current detached position
```

---

# 9. The Three Areas

## What is it?
Git manages files across three distinct areas:
1. **Working Directory** — actual files on disk (what you see in your editor)
2. **Staging Area (Index)** — `.git/index` — draft of the next commit
3. **Local Repository** — `.git/objects/` — permanent committed history

## Why It Matters
- The staging area is Git's superpower — lets you craft precise, atomic commits even when multiple unrelated changes exist in working directory
- Every `git add` updates the staging area; every `git commit` writes the staging area to the repository
- Misunderstanding these three areas causes most beginner Git confusion

## Internal Working
```
Working Directory          Staging Area              Local Repository
(editor / disk)            (.git/index)              (.git/objects/)
────────────────           ────────────              ────────────────
Payment.java (edits)  →    Payment.java →blob hash   commit C1 → tree → blobs
Main.java (edits)     →    Main.java    →blob hash   commit C2 → tree → blobs
                      ↑                          ↑
                   git add                    git commit

git add:    reads Working Dir → computes blob → writes to objects/ → updates index
git commit: reads index → builds trees → creates commit → advances branch pointer
```

## Command Explanation

### Syntax
```bash
git ls-files --stage     # inspect staging area (index) contents
git diff                  # Working Dir vs Staging Area
git diff --staged         # Staging Area vs Last Commit
git diff HEAD             # Working Dir vs Last Commit
```

```bash
# Show raw index contents with SHAs
git ls-files --stage
# 100644 39ed3cc421c9... 0  src/main/java/com/concepts/Payment.java
# 100644 ad6e40b42a56... 0  src/main/java/com/concepts/Main.java

# What changed in working dir but NOT yet staged
git diff

# What IS staged (what will be committed)
git diff --staged
git diff --cached           # same as --staged

# Everything changed vs last commit
git diff HEAD
```

---

# 10. Interview Questions & Answers

**Q: "How does Git store data internally?"**
> Git uses a content-addressable object store in `.git/objects/`. Every piece of data is stored as one of four immutable object types: blob (file content only), tree (directory structure mapping names to blob/tree SHAs), commit (project snapshot pointing to root tree + parent commit + metadata), or annotated tag (named reference with metadata). Each object is identified by SHA-1 of its content. Identical content always produces the same SHA — enabling deduplication. Git stores complete snapshots, not diffs — unchanged files re-use existing blob objects via SHA pointers.

**Q: "What is the difference between a blob and a tree?"**
> A blob stores only raw file content — no filename, no path, no permissions. A tree stores directory structure: it maps filenames and permissions to blob SHAs (for files) and other tree SHAs (for subdirectories). This separation enables deduplication: two files with identical content share one blob even if they have different names. The tree provides the naming and structural context the blob deliberately omits.

**Q: "What is HEAD and what is detached HEAD state?"**
> HEAD is a special pointer in `.git/HEAD` pointing to the currently checked-out branch, which in turn points to the latest commit on that branch. Normally it's a symbolic ref: `ref: refs/heads/main`. Detached HEAD occurs when HEAD points directly to a commit SHA rather than to a branch name — happens when you `git checkout <sha>` or `git checkout <tag>`. In this state, new commits you make are not on any branch and will be garbage-collected unless you create a branch with `git switch -c new-branch`.

**Q: "Why does rebasing produce new commits even when the code changes are identical?"**
> A commit's SHA is computed from ALL its content: the root tree SHA, the parent commit SHA, author/committer info, and the message. When you rebase, your commits are replayed onto a new base — the parent SHA changes. A different parent SHA → different commit content → different SHA. So the "same" code change becomes a new commit object with a new SHA. This is why rebasing rewrites history and why force-push is needed after rebasing a pushed branch.

**Q: "Why is Git branching so much faster than SVN branching?"**
> In SVN, creating a branch copies files on the server — O(n) proportional to repo size. In Git, a branch is a 41-byte text file containing one SHA-1 hash. Creating a branch is O(1) — just write one small file. Switching branches in Git replays the diff between two tree snapshots locally, without any server interaction. This fundamental architectural difference is why Git treats branches as disposable, lightweight tools used freely throughout development.

---

# 11. Quick Revision Cheatsheet

```bash
# ─── OBJECT INSPECTION ────────────────────────────────────
git cat-file -t <sha>          # type: blob / tree / commit / tag
git cat-file -p <sha>          # pretty-print content
git cat-file -s <sha>          # size in bytes
git hash-object <file>         # compute SHA (don't store)
git hash-object -w <file>      # compute SHA and store

# ─── TREE / INDEX ─────────────────────────────────────────
git ls-tree HEAD               # root tree at HEAD
git ls-tree -r HEAD            # all files (recursive)
git ls-files --stage           # staging area with SHAs + stage numbers

# ─── COMMIT ───────────────────────────────────────────────
git cat-file -p HEAD           # raw commit object
git show HEAD                  # commit diff + metadata
git log --oneline --graph --all --decorate   # visual DAG

# ─── HEAD & REFS ──────────────────────────────────────────
cat .git/HEAD                  # what HEAD points to
git symbolic-ref HEAD          # branch name HEAD points to
git rev-parse HEAD             # full SHA of HEAD
git show-ref                   # all refs + SHAs
git branch -v                  # local branches + last commit

# ─── INTEGRITY ────────────────────────────────────────────
git fsck --full                # verify all objects
git count-objects -vH          # object count + sizes
git gc                         # garbage collect + pack loose objects

# ─── SHA FORMULA (memory) ─────────────────────────────────
# SHA-1("blob "   + size + "\0" + content)  → blob
# SHA-1("tree "   + size + "\0" + entries)  → tree
# SHA-1("commit " + size + "\0" + content)  → commit
```

---

> **Previous:** [01 · Git Introduction & Setup](./01_Git_Setup_and_Introduction.md)  
> **Next:** [03 · Git File Lifecycle →](./03_Git_File_Lifecycle.md)
