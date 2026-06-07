# 09 · Production Release Management with Git

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [04 · Git Branching & Merging](./04_Git_Branching_and_Merging.md)

---

## Table of Contents

1. [What is Release Management?](#1-what-is-release-management)
2. [Why It Matters](#2-why-it-matters)
3. [Internal Working — Tags & Release Pointers](#3-internal-working--tags--release-pointers)
4. [Semantic Versioning (SemVer)](#4-semantic-versioning-semver)
5. [Command Explanation — Tags](#5-command-explanation--tags)
6. [Release Branch Strategies](#6-release-branch-strategies)
7. [Hotfix Management](#7-hotfix-management)
8. [Changelog Generation](#8-changelog-generation)
9. [Release Automation](#9-release-automation)
10. [Rollback Strategies](#10-rollback-strategies)
11. [Multi-Version Maintenance](#11-multi-version-maintenance)
12. [Interview Questions & Model Answers](#12-interview-questions--model-answers)
13. [Quick Revision Cheatsheet](#13-quick-revision-cheatsheet)
14. [Real-World Release Workflows](#14-real-world-release-workflows)
15. [Troubleshooting](#15-troubleshooting)

---

## 1. What is Release Management?

## What is it?

**Release management** in Git is the set of practices, workflows, and tooling used to:
- Package and version a specific state of the codebase
- Promote that version through environments (dev → staging → production)
- Track exactly what code is live in production at any point
- Roll back safely when something goes wrong
- Maintain multiple production versions simultaneously (for enterprise/versioned products)

### Core Components

```
Release Management = Tagging + Branching Strategy + Changelog + Automation + Rollback Plan
```

---

## 2. Why It Matters

## What is it?
A comparison of release management capabilities.

## Why It Matters

## Internal Working
Release management in Git is built on tags (immutable refs to commits), branches (environment/version pointers), and CI/CD integration (tag push triggers pipelines).

## Command Explanation

### Syntax
```bash
git tag -a v2.1.0 -m "msg"   # mark a release
git push origin v2.1.0        # trigger release pipeline
```

| Without Release Management | With Release Management |
|---|---|
| "Which commit is live?" — unknown | Tag `v2.1.0` always points to production SHA |
| Rollback = guess and hope | `git checkout v2.0.9` → redeploy exact previous state |
| Hotfix applied to wrong version | Hotfix branch from release tag, cherry-picked precisely |
| Changelog written manually (incomplete) | Auto-generated from commit history |
| Deploy timing unclear | CI/CD triggers on tag push |
| Multi-version support chaotic | Long-lived release branches per major version |

---

## 3. Internal Working — Tags & Release Pointers

## What is it?
An annotated tag is a full Git object storing the tagger info, timestamp, message, and a pointer to a commit.

## Why It Matters
Annotated tags are the immutable, permanent markers of releases — they survive GC, carry metadata, and can be GPG-signed.

## Internal Working

### Annotated Tag Object

An annotated tag is a **full Git object** stored in `.git/objects/`, distinct from the commit it points to:

```
Tag Object (v2.1.0):
  object  c1a2b3c4...          ← SHA of the tagged commit
  type    commit
  tag     v2.1.0
  tagger  CI-Bot <ci@co.com> 1746000000 +0000

  Release v2.1.0 — payment retry feature

  Changes:
  - feat(payment): exponential backoff retry
  - fix(auth): DST timezone edge case
```

```bash
# Tag SHA ≠ Commit SHA
git rev-parse v2.1.0            # → tag object SHA
git rev-parse v2.1.0^{}         # → commit SHA (dereference tag)
git cat-file -p v2.1.0          # → inspect tag object
```

### Lightweight vs Annotated Tags — Internal Difference

```
Lightweight tag:
  .git/refs/tags/v2.1.0-light → c1a2b3c4...   (directly the commit SHA)
  No object in .git/objects/

Annotated tag:
  .git/refs/tags/v2.1.0 → t9x8y7z6...         (tag object SHA)
  .git/objects/t9/x8y7z6...  (full tag object)
  tag object → c1a2b3c4...   (commit SHA)
```

> **Interview Insight:** Always use annotated tags for releases. They are permanent objects in the Git database, can carry a message, tagger info, and timestamp, and can be GPG-signed. Lightweight tags are just pointers and can be silently overwritten.

---

## 4. Semantic Versioning (SemVer)

## What is it?
Semantic Versioning is a versioning scheme: `MAJOR.MINOR.PATCH` where each part signals the type of change.

## Why It Matters
Communicates breaking changes, new features, and bug fixes to consumers of your software through the version number alone.

## Internal Working
Git tags use SemVer labels. Tools like `semantic-release` automatically determine the next version by parsing Conventional Commit types from git log.

## Command Explanation

### Syntax
```bash
git tag -a v2.1.0 -m "Release"
git tag -l --sort=-version:refname "v*"
```

### Format

```
v<MAJOR>.<MINOR>.<PATCH>[-<pre-release>][+<build-metadata>]

v2.1.0
v2.1.1-rc.1
v2.1.1-beta.2
v3.0.0-alpha.1
```

### When to Bump

| Change Type | Version Bump | Example |
|---|---|---|
| Breaking API change | MAJOR | v2.0.0 → v3.0.0 |
| Backward-compatible new feature | MINOR | v2.0.0 → v2.1.0 |
| Backward-compatible bug fix | PATCH | v2.0.0 → v2.0.1 |
| Pre-release / RC | Pre-release suffix | v2.1.0-rc.1 |

### Pre-release Naming Convention

```
v2.1.0-alpha.1   ← early testing, may have breaking changes
v2.1.0-beta.1    ← feature complete, bug fixes only
v2.1.0-rc.1      ← release candidate, production-ready candidate
v2.1.0           ← stable release
v2.1.1           ← patch release
```

### SemVer with Conventional Commits

```
Commits since last release → Auto-determined next version:
  fix: ...        → PATCH bump  (v2.0.0 → v2.0.1)
  feat: ...       → MINOR bump  (v2.0.0 → v2.1.0)
  BREAKING CHANGE → MAJOR bump  (v2.0.0 → v3.0.0)
```

---

## 5. git tag — Release Tagging

## What is it?
`git tag` creates named references to specific commits. Annotated tags are full objects; lightweight tags are just pointer files.

## Why It Matters
Tags are the permanent markers of what went to production — always use annotated tags for releases.

## Internal Working
Annotated tag object stored in `.git/objects/`, lightweight tag stored only in `.git/refs/tags/` with no object.

## Command Explanation

### Syntax

```bash
git tag [options] [tagname] [commit]
```

### Creating Tags

```bash
# Annotated tag on HEAD (USE FOR RELEASES)
git tag -a v2.1.0 -m "Release v2.1.0"

# Annotated tag with multi-line message
git tag -a v2.1.0 -F release-notes.txt

# Annotated tag on specific commit (retroactive)
git tag -a v2.1.0 abc123def -m "Release v2.1.0"

# Lightweight tag (NOT recommended for releases)
git tag v2.1.0-light

# GPG-signed tag (for high-security releases)
git tag -s v2.1.0 -m "Release v2.1.0"
git tag -v v2.1.0     # verify GPG signature
```

### Listing & Inspecting Tags

```bash
# List all tags (sorted alphabetically)
git tag

# List tags matching pattern
git tag -l "v2.*"
git tag -l "v2.1.*"

# List tags with version sort (correct for SemVer!)
git tag -l --sort=-version:refname "v*"
# v2.1.0
# v2.0.1
# v2.0.0
# v1.9.3

# Show tag details + tagged commit
git show v2.1.0

# Get commit SHA from tag (dereference)
git rev-parse v2.1.0^{}

# Latest tag reachable from HEAD
git describe --tags --abbrev=0

# Generate version string
git describe --tags
# v2.1.0-14-gabc123def   (14 commits past tag, at SHA abc123def)
```

### Pushing Tags

```bash
# Push single tag
git push origin v2.1.0

# Push all local tags
git push --tags

# Push only annotated tags (excludes lightweight)
git push origin 'refs/tags/*'

# Better: push only tags matching pattern
git push origin 'refs/tags/v*'

# Delete remote tag
git push origin --delete v2.1.0
git push origin :refs/tags/v2.1.0   # older syntax

# Delete local tag
git tag -d v2.1.0
```

### Checking Out a Tag

```bash
# Checkout tag → enters detached HEAD state
git checkout v2.1.0

# Create branch from tag (better than detached HEAD)
git switch -c release/v2.1.0 v2.1.0

# Verify you're on the right commit
git log --oneline -1
git describe --tags --exact-match   # must match tag exactly
```

---

## 6. Release Branch Strategies

## What is it?
Different approaches to managing releases through Git branches depending on the release model (continuous vs scheduled).

## Why It Matters
The wrong strategy causes release pain: either too slow (Git Flow for CI/CD) or too chaotic (GitHub Flow for versioned libraries).

## Internal Working
Branch strategies define: which commits go to production, how hotfixes are applied, and how versions are tracked in the DAG.

## Command Explanation

### Syntax
```bash
git switch -c release/2.1.0 develop   # Git Flow
git tag -a v2.1.0                     # tag the release
git merge --no-ff release/2.1.0       # merge to main
```

### Strategy A: Tag-Based Releases (GitHub Flow / Continuous Deployment)

Best for SaaS with no versioned releases. Every merge to `main` is a potential release.

```
main:  A ← B ← C ← D ← E
                     ↑       ↑
                  v2.0.0   v2.1.0

No release branches needed.
CI/CD deploys automatically on merge to main.
Tag applied after successful deployment.
```

```bash
# Release process
git switch main && git pull --ff-only
# Run smoke tests, approve
git tag -a v2.1.0 -m "Release v2.1.0" HEAD
git push origin v2.1.0
# CI/CD picks up tag push → deploys to production
```

### Strategy B: Release Branches (Git Flow)

Best for software with scheduled releases (mobile apps, libraries, enterprise software).

```
main:     A ← B ← C ──────────────────────────── M  ← v3.0.0
                    ↓ branch
release/v2.x:       C ← D(fix) ← E(fix) ──────── F
                    ↑                  ↑         ↑
                  branch          v2.1.0     v2.1.1

develop:  A ← B ← C ← F ← G ← H ← I ...
```

```bash
# Cut release branch from develop
git switch develop
git pull --ff-only
git switch -c release/2.1.0

# Only bug fixes on release branch
git commit -m "fix(payment): resolve rounding error"

# When ready to release
git tag -a v2.1.0 -m "Release v2.1.0"
git push origin release/2.1.0 --tags

# Merge back to main AND develop (to keep fixes in develop)
git switch main
git merge --no-ff release/2.1.0
git push origin main

git switch develop
git merge --no-ff release/2.1.0
git push origin develop
```

### Strategy C: Environment Branches (GitLab Flow)

```
feature → main → staging → production
                 ↑ auto-deploy  ↑ manual promote
```

```bash
# Promote to staging (auto via CI on merge to main)
git switch main && git merge --no-ff feature/payment

# Promote to production
git switch production
git merge --no-ff main     # or cherry-pick specific commits
git push origin production
# → CI deploys production branch to production environment
```

---

## 7. Hotfix Management

## What is it?
A hotfix is an emergency fix applied to production code by branching from the production tag — bypassing the normal development pipeline.

## Why It Matters
Hotfixes must reach production immediately without carrying in-progress feature work. Branching from the production TAG (not develop or main) ensures exactly this.

## Internal Working
Hotfix branch is created from the production tag SHA — not from main (which may have unreleased commits). Fix is applied, tagged, merged to main, and cherry-picked to develop.

## Command Explanation

### Syntax
```bash
git switch -c hotfix/v2.1.1 v2.1.0    # branch FROM the production tag
git tag -a v2.1.1 -m "Hotfix"
git push origin main --tags
git cherry-pick <sha>                  # backport to develop
```

### What is a Hotfix?

A **hotfix** is an emergency fix applied directly to production code — bypassing the normal feature development pipeline.

### Hotfix Process (Git Flow)

```
main (v2.1.0):  A ← B ← C ─────────────────── G ← v2.1.1
                                ↓ branch from tag
hotfix/v2.1.1:                  C ← D(fix)     ↑ merge to main
                                                ↑ merge to develop
develop:        A ← B ← C ← E ← F ←──────────── G (cherry-pick)
```

```bash
# Step 1: Branch from the production tag
git switch -c hotfix/v2.1.1 v2.1.0

# Step 2: Apply fix
vim src/main/java/com/payment/PaymentService.java
git add .
git commit -m "fix(payment): prevent NPE when amount is null

Amount validation was missing for the currency-absent code path.
Adds null coalescing before processPayment() call.

Fixes: #PROD-4521
Severity: P1 — 0.3% of payments failing since v2.1.0 deploy"

# Step 3: Run tests
mvn test

# Step 4: Tag the hotfix
git tag -a v2.1.1 -m "Hotfix v2.1.1 — payment NPE"

# Step 5: Merge to main (production)
git switch main
git merge --no-ff hotfix/v2.1.1
git push origin main

# Step 6: Push tag
git push origin v2.1.1

# Step 7: Backport to develop (so fix is in future releases)
git switch develop
git cherry-pick <fix-commit-sha>
git push origin develop

# Step 8: Clean up
git branch -d hotfix/v2.1.1
git push origin --delete hotfix/v2.1.1
```

### Hotfix Decision Matrix

```
Severity?
├── P1 (production down / major data issue)
│   └── Hotfix immediately → deploy → tag
├── P2 (significant feature broken)
│   └── Hotfix branch → QA → deploy next business hour
└── P3 (minor issue, workaround exists)
    └── Normal feature branch → next scheduled release
```

---

## 8. Changelog Generation

## What is it?
A changelog is a human-readable record of all notable changes per version, either written manually or auto-generated from commit history.

## Why It Matters
Users and operators need to know what changed between versions — especially breaking changes, new features, and important fixes.

## Internal Working
Auto-generated changelogs work by running `git log v1.0.0..v2.0.0 --oneline --no-merges` and filtering/formatting commits by type prefix.

## Command Explanation

### Syntax
```bash
git cliff --latest              # since last tag
git cliff v2.0.0..v2.1.0       # between tags
npx conventional-changelog -p angular -i CHANGELOG.md -s
```

### Manual CHANGELOG.md Structure

```markdown
# Changelog
All notable changes to this project will be documented in this file.
Format: [Semantic Versioning](https://semver.org/)

## [Unreleased]
### Added
- Payment retry logic (exponential backoff)

## [2.1.0] - 2026-06-04
### Added
- feat(payment): exponential backoff retry (#247)
- feat(auth): OAuth2 PKCE flow support (#251)

### Fixed
- fix(payment): NPE when currency field absent (#PROD-4521)
- fix(auth): JWT expiry edge case on DST change (#238)

### Changed
- refactor(db): extract PaymentRepository from PaymentService (#243)

## [2.0.1] - 2026-05-20
### Fixed
- fix(payment): rounding error for amounts > 1000 (#229)

## [2.0.0] - 2026-05-01
### Changed
- BREAKING: PaymentService.processPayment() now async (returns CompletableFuture)

[Unreleased]: https://github.com/company/repo/compare/v2.1.0...HEAD
[2.1.0]: https://github.com/company/repo/compare/v2.0.1...v2.1.0
[2.0.1]: https://github.com/company/repo/compare/v2.0.0...v2.0.1
```

### Auto-Generate from Git Log

```bash
# Generate changelog between two tags
git log v2.0.1..v2.1.0 --oneline --no-merges \
  | grep -E "^[a-f0-9]+ (feat|fix|perf|refactor)" \
  | awk '{$1=""; print "- "$0}'

# Using git-cliff (Rust tool — recommended)
git cliff --latest                    # changes since last tag
git cliff v2.0.1..v2.1.0             # between two tags
git cliff --output CHANGELOG.md       # write to file

# Using conventional-changelog (Node.js)
npx conventional-changelog -p angular -i CHANGELOG.md -s
```

---

## 9. Release Automation

## What is it?
Automated release pipelines triggered by Git tag pushes that build, test, publish artifacts, and deploy to production without manual steps.

## Why It Matters
Manual releases are error-prone and slow. Automated releases triggered by `git push origin v2.1.0` make releases a 10-second operation.

## Internal Working
CI/CD systems (GitHub Actions, GitLab CI) register webhooks on the Git host. A tag push event (`refs/tags/v*`) triggers the release workflow which reads `GITHUB_REF` to extract the version.

## Command Explanation

### Syntax
```bash
git tag -a v2.1.0 -m "Release"
git push origin v2.1.0           # triggers CI release pipeline
```

### GitHub Actions — Full Release Pipeline

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v*'          # Trigger on any tag starting with v

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0      # Need full history for changelog

      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Build
        run: mvn clean package -DskipTests

      - name: Test
        run: mvn test

      - name: Extract version from tag
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Generate Changelog
        run: |
          git cliff --latest --output RELEASE_NOTES.md

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          name: Release ${{ steps.version.outputs.VERSION }}
          body_path: RELEASE_NOTES.md
          files: target/*.jar
          generate_release_notes: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Deploy to Production
        run: ./scripts/deploy.sh ${{ steps.version.outputs.VERSION }}
        env:
          DEPLOY_KEY: ${{ secrets.DEPLOY_KEY }}
```

### Automated Version Bumping with `semantic-release`

```yaml
# .github/workflows/semantic-release.yml
name: Semantic Release

on:
  push:
    branches: [main]

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          persist-credentials: false

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: npm ci

      - name: Semantic Release
        uses: cycjimmy/semantic-release-action@v4
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          NPM_TOKEN: ${{ secrets.NPM_TOKEN }}
```

---

## 10. Rollback Strategies

## What is it?
A rollback strategy defines how to safely return production to a known-good state when a release causes problems.

## Why It Matters
Every release must have a documented rollback plan. The right strategy depends on severity, time pressure, and whether DB migrations are involved.

## Internal Working
`git revert` creates a new commit (safe, history preserved). `git reset --hard <tag>` moves the branch pointer back (dangerous, requires force-push, notifies team). Redeploying a previous tag uses CI/CD without touching Git history.

## Command Explanation

### Syntax
```bash
git revert <sha>                 # safe: new undo commit
git revert -m 1 <merge-sha>      # revert a merge commit
git reset --hard v2.0.9          # hard rollback ⚠️
git push --force-with-lease      # after hard rollback
```

### Strategy 1: Redeploy Previous Tag (Fastest)

```bash
# Identify the last known good version
git tag -l --sort=-version:refname "v*" | head -5

# Check what that version contains
git log --oneline v2.0.9..v2.1.0    # what changed between versions
git show v2.0.9                      # inspect that tag

# Trigger CI/CD to deploy previous version
# (Usually done through CI/CD UI, not git directly)
# OR manually:
git checkout v2.0.9
# → deploy this build artifact
```

### Strategy 2: Revert Commit (Safe — New Commit)

```bash
# Find the bad commit
git log --oneline v2.0.9..v2.1.0

# Revert specific commit
git switch main
git revert abc123def              # creates new commit undoing abc123
git push origin main
git tag -a v2.1.1 -m "Hotfix: revert bad payment change"
git push origin v2.1.1
```

### Strategy 3: Revert Merge Commit

```bash
# Revert an entire feature that was merged
git log --merges --oneline -5
# abc123 Merge branch 'feature/bad-feature'

git revert -m 1 abc123            # -m 1 = keep mainline (parent 1)
# Creates new commit undoing everything the merge introduced
git push origin main
```

### Strategy 4: `git reset --hard` (Last Resort — Force Push)

```bash
# ⚠️ Only if you're CERTAIN no one has pulled the broken commits
git reset --hard v2.0.9
git push --force-with-lease origin main

# Notify ALL team members immediately — their history is now diverged
# Everyone must: git fetch && git reset --hard origin/main
```

### Rollback Decision Tree

```
Production is broken. What do I do?
│
├── Is the issue in a single recent commit?
│   └── git revert <sha> → push → deploy
│
├── Is the issue in a merged feature branch?
│   └── git revert -m 1 <merge-sha> → push → deploy
│
├── Is there a known-good previous version artifact?
│   └── Redeploy previous tag from CI/CD (fastest)
│
├── Are multiple commits involved and hard to isolate?
│   └── git reset --hard <last-good-tag>
│       → git push --force-with-lease (notify team!)
│
└── Database migration involved?
    └── Run DB rollback script first, then git revert/reset
        (ensure backward-compatible schema before code rollback)
```

---

## 11. Multi-Version Maintenance

### Maintaining v1.x and v2.x Simultaneously

```
main:         ──────────── v3.0.0-dev
                 
release/v2.x: ──── v2.0.0 ── v2.1.0 ── v2.1.1(hotfix)
                 
release/v1.x: ── v1.9.0 ── v1.9.1(hotfix)
```

```bash
# Create and maintain release branches
git switch -c release/v1.x v1.9.0
git push -u origin release/v1.x

# Customer reports bug in v1.x
git switch release/v1.x
git switch -c hotfix/v1.x/session-expiry

# Apply fix
git commit -m "fix(auth): resolve session expiry for v1.x clients"
git switch release/v1.x
git merge --no-ff hotfix/v1.x/session-expiry
git tag -a v1.9.1 -m "Hotfix v1.9.1"
git push origin release/v1.x --tags

# Check if fix applies to v2.x as well
git switch release/v2.x
git cherry-pick <fix-commit-sha>
git tag -a v2.1.1 -m "Hotfix v2.1.1"
git push origin release/v2.x --tags

# Merge to main if relevant
git switch main
git cherry-pick <fix-commit-sha>
```

---

## 12. Interview Questions & Model Answers

### Q1: "How does your team manage releases with Git?"

**Model Answer:**
> We use a tag-based release process aligned with GitHub Flow since we practice continuous deployment. Every merge to `main` passes through CI (tests, security scans, staging deploy). When we're ready to cut a release, we create an annotated tag (`git tag -a v2.1.0 -m "Release notes..."`) and push it. This tag push triggers our release CI pipeline: builds the artifact, runs smoke tests against staging, generates the changelog with `git cliff`, creates a GitHub Release with the notes, and deploys to production. The tag is the immutable record of exactly what went to production. For rollback, we either `git revert` the offending commit (for code issues) or trigger a CI deploy of the previous tag. We never force-push to main.

### Q2: "What is the difference between an annotated tag and a lightweight tag?"

**Model Answer:**
> A lightweight tag is simply a named pointer to a commit — a file in `.git/refs/tags/` containing the commit SHA. A annotated tag is a full Git object stored in `.git/objects/`, containing the tagger's name, email, timestamp, a message, and optionally a GPG signature. It points to a commit via indirection. For releases, always use annotated tags because: they're permanent objects (survive `git gc` differently), they carry metadata (who tagged it, when, why), they can be GPG-signed for integrity verification, they work correctly with `git describe`, and they communicate intent — this is a formal release, not a casual bookmark.

### Q3: "How do you handle a hotfix on production while the team is mid-sprint developing new features?"

**Model Answer:**
> Branch from the production tag, not from develop or main if they contain in-progress features. `git switch -c hotfix/v2.1.1 v2.1.0`. Apply and test the minimal fix on this branch. Tag it (`git tag -a v2.1.1 -m "..."`) and deploy from this branch — this ensures production gets only the fix, not any in-progress sprint work. Then merge the hotfix back to both main and develop so the fix is included in all future releases. If develop has already diverged significantly, cherry-pick the fix commit onto develop rather than merging the whole hotfix branch. The key principle: production should always be deployable from a known, stable tag, not from the tip of a development branch.

### Q4: "How would you automate the release versioning process?"

**Model Answer:**
> Using `semantic-release` with Conventional Commits. `semantic-release` analyzes commit messages since the last release: `fix` commits → patch bump, `feat` commits → minor bump, `BREAKING CHANGE` in footer → major bump. It then creates the tag, generates the changelog, creates the GitHub Release, and publishes to the package registry — all automatically in CI on merge to main. The only human input is writing correct commit messages. This eliminates manual version decisions, ensures releases are semantically meaningful, and generates accurate changelogs. For Java projects, I pair this with Maven's versions plugin. For gradual adoption, teams can start with `standard-version` which does the same without requiring CI integration.

---

## 13. Quick Revision Cheatsheet

```bash
# ─── TAGGING ──────────────────────────────────────────────
git tag -a v2.1.0 -m "Release v2.1.0"          # annotated tag on HEAD
git tag -a v2.1.0 abc123 -m "msg"               # tag specific commit
git tag -l --sort=-version:refname "v*"          # list sorted correctly
git show v2.1.0                                  # inspect tag
git push origin v2.1.0                           # push single tag
git push origin 'refs/tags/v*'                   # push all v* tags
git tag -d v2.1.0                               # delete local tag
git push origin --delete v2.1.0                 # delete remote tag
git rev-parse v2.1.0^{}                         # get commit SHA from tag

# ─── RELEASE WORKFLOW ─────────────────────────────────────
git describe --tags                              # version string (v2.1.0-5-gabc123)
git describe --tags --abbrev=0                   # latest tag only: v2.1.0
git log v2.0.9..v2.1.0 --oneline --no-merges    # what changed between versions
git diff v2.0.9..v2.1.0 -- src/                 # diff between versions

# ─── HOTFIX ───────────────────────────────────────────────
git switch -c hotfix/v2.1.1 v2.1.0             # branch from tag
git tag -a v2.1.1 -m "Hotfix v2.1.1"           # tag the fix
git cherry-pick <sha>                            # backport to develop/main

# ─── ROLLBACK ─────────────────────────────────────────────
git revert <sha>                                 # safe: new commit
git revert -m 1 <merge-sha>                      # revert merge
git reset --hard v2.0.9                          # hard rollback ⚠️
git push --force-with-lease origin main          # after hard rollback

# ─── CHANGELOG ────────────────────────────────────────────
git cliff --latest                               # changes since last tag
git log v2.0.9..v2.1.0 --oneline --no-merges    # manual changelog source
```

---

## 14. Real-World Release Workflows

### Workflow A: SaaS Continuous Deployment Release

```bash
# All features merged to main via PRs, all CI passing

# QA on staging signs off
# Create release
git switch main
git pull --ff-only

# Final sanity: what's going out?
git log $(git describe --tags --abbrev=0)..HEAD --oneline --no-merges

# Tag the release
git tag -a v2.1.0 -m "Release v2.1.0

Features:
- feat(payment): exponential backoff retry (#247)
- feat(auth): OAuth2 PKCE flow (#251)

Fixes:
- fix(payment): NPE when currency absent (#PROD-4521)

Full changelog: https://github.com/company/repo/blob/main/CHANGELOG.md"

# Push tag — triggers CI release pipeline
git push origin v2.1.0

# Monitor deployment
# → CI: builds artifact, smoke tests staging, deploys production
# → Datadog/Grafana: watch error rate for 15 min post-deploy
# → If clean: release complete
# → If issues: git revert → git push → new tag → redeploy
```

### Workflow B: Library/SDK Release (Git Flow)

```bash
# Feature development done on develop branch
# Create release branch
git switch develop
git pull --ff-only
git switch -c release/2.1.0

# Only bug fixes allowed now
# Run full test suite, update version in pom.xml
mvn versions:set -DnewVersion=2.1.0
git add pom.xml
git commit -m "chore(release): bump version to 2.1.0"

# Update CHANGELOG.md
vim CHANGELOG.md
git add CHANGELOG.md
git commit -m "docs: update changelog for 2.1.0"

# Merge to main
git switch main
git merge --no-ff release/2.1.0 -m "chore: merge release/2.1.0 to main"
git tag -a v2.1.0 -m "Release v2.1.0"

# Merge back to develop
git switch develop
git merge --no-ff release/2.1.0

# Push everything
git push origin main develop --tags

# Publish to Maven Central via CI
# Delete release branch
git branch -d release/2.1.0
git push origin --delete release/2.1.0
```

---

## 15. Troubleshooting

| Problem | Cause | Solution |
|---|---|---|
| Tag not triggering CI | Tag not pushed | `git push origin v2.1.0` |
| Wrong commit tagged | Didn't verify HEAD before tagging | `git tag -d v2.1.0` → verify → re-tag |
| Tag already exists remotely | Trying to re-tag | `git tag -d v2.1.0` → `git push origin --delete v2.1.0` → re-create |
| `git describe` shows wrong version | Lightweight tag in the way | Use annotated tags only for releases |
| Hotfix applied to wrong branch | Branched from develop, not tag | `git switch -c hotfix/x <tag>` (always branch from production tag) |
| Changelog missing commits | Commits not following Conventional Commits | Enforce `commitlint` in CI |
| Merge conflict when backporting hotfix | Develop diverged significantly | Use `git cherry-pick` instead of merge |
| Rollback caused DB errors | DB migration not rolled back | Always roll back DB before code; design backward-compatible migrations |

---

> **Previous:** [08 · Git Commit Messages & Conventions](./08_Git_Commit_Messages_and_Conventions.md)  
> **Next:** [10 · Large-Scale Team Collaboration](./10_Large_Scale_Team_Collaboration.md)
