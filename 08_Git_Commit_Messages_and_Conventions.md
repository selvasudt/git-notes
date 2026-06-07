# 08 · Git Commit Messages & Conventions

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [03 · Git File Lifecycle](./03_Git_File_Lifecycle.md)

---

## Table of Contents

1. [What is a Commit Message?](#1-what-is-a-commit-message)
2. [Why It Matters](#2-why-it-matters)
3. [Internal Working — How Git Stores Messages](#3-internal-working--how-git-stores-messages)
4. [Anatomy of a Good Commit Message](#4-anatomy-of-a-good-commit-message)
5. [Conventional Commits Specification](#5-conventional-commits-specification)
6. [The 7 Rules of a Great Commit Message](#6-the-7-rules-of-a-great-commit-message)
7. [Bad vs Good — Real Examples](#7-bad-vs-good--real-examples)
8. [Commit Message Templates](#8-commit-message-templates)
9. [Enforcing Standards with Hooks](#9-enforcing-standards-with-hooks)
10. [Automated Changelog & Versioning](#10-automated-changelog--versioning)
11. [Amending & Editing Messages](#11-amending--editing-messages)
12. [Interview Questions & Model Answers](#12-interview-questions--model-answers)
13. [Quick Revision Cheatsheet](#13-quick-revision-cheatsheet)

---

## 1. What is a Commit Message?

## What is it?

A **commit message** is the human-readable metadata attached to every Git commit. It explains **what** changed and **why** — not **how** (the diff shows the how).

### Purpose

- Communicate intent to teammates during code review
- Explain reasoning to your future self
- Enable automated tooling (changelog generation, semantic versioning, CI filters)
- Serve as an audit trail for compliance and debugging
- Power `git log`, `git bisect`, `git blame` workflows

---

## 2. Why It Matters

## What is it?
Commit message quality directly determines whether Git history is an asset or a liability.

## Why It Matters

### The Real Cost of Bad Commit Messages

```
git log --oneline (a real repo with bad practices):
  a1b2c3 fix
  d4e5f6 update
  g7h8i9 WIP
  j0k1l2 changes
  m3n4o5 stuff
  p6q7r8 asdf
  s9t0u1 final fix
  v2w3x4 final final fix
```

This is noise. It tells you nothing about what happened, why it happened, or which commit introduced a bug. `git bisect` becomes guesswork; `git blame` reveals nothing useful.

### The Value of Good Commit Messages

```
git log --oneline (same repo, good practices):
  a1b2c3 feat(payment): add exponential backoff retry logic
  d4e5f6 fix(auth): resolve JWT expiry edge case on DST change
  g7h8i9 refactor(db): extract repository pattern for UserDAO
  j0k1l2 test(payment): add unit tests for retry scenarios
  m3n4o5 chore: upgrade Spring Boot to 3.2.1
  p6q7r8 docs(api): document payment endpoint request/response
```

Now you can:
- Understand project evolution without reading diffs
- Auto-generate a changelog from commit history
- Filter CI jobs by commit type (`feat` → run full suite; `docs` → skip)
- Use `git log --grep="feat(payment)"` to find all payment features

---

## 3. Internal Working — How Git Stores Messages

## What is it?
The commit message is embedded inside the commit object and contributes to the commit SHA.

## Why It Matters
Changing a message changes the SHA — amending requires force-push on shared branches.

## Internal Working

### Commit Object Structure

The message is stored directly inside the **commit object** in `.git/objects/`:

```
tree   f9e8d7c6...              ← root tree SHA
parent c1a2b3c4...              ← parent commit SHA
author  Shrayansh <s@co.com> 1746000000 +0530
committer Shrayansh <s@co.com> 1746000000 +0530
                                ← blank line separator (mandatory)
feat(payment): add retry logic

Implements exponential backoff (3 retries, 2s base delay) when
payment gateway returns 503. Circuit breaker resets after 60s.

Closes: #247
```

### SHA Implications

The commit SHA is computed from the **entire commit object content**, which includes the message. Therefore:

- Changing the commit message changes the commit SHA
- `git commit --amend -m "new message"` produces a **new commit** with a different SHA
- This is why amending pushed commits requires `--force-with-lease`

### Multiline Messages in the Object Store

```bash
# View raw commit object
git cat-file -p HEAD

# The message starts after the blank line
# Subject line (first line) = everything up to first \n
# Body = everything after the blank line
```

---

## 4. Anatomy of a Good Commit Message

## What is it?
The three-part structure (subject + body + footer) is the universal standard for professional commit messages.

## Why It Matters
Proper structure enables automated changelog generation, searchability, and clear code review context.

## Internal Working
Git uses the first line (before first blank line) as the commit subject — shown in `git log --oneline`. Everything after the blank line is the body, shown in `git show`.

## Command Explanation

### Syntax
```bash
git commit -m "subject"
git commit  # opens editor for full subject+body+footer
```

### The Three-Part Structure

```
<type>(<scope>): <subject>          ← SUBJECT LINE (50 chars max)
                                    ← BLANK LINE (mandatory separator)
<body>                              ← BODY (72 chars per line, optional)
                                    ← BLANK LINE
<footer>                            ← FOOTER (optional: issue refs, breaking changes)
```

### Subject Line Rules

```
feat(payment): add exponential backoff retry logic
├──┘ ├──────┘  ├─────────────────────────────────┘
type  scope              description
                    (imperative mood, no period, ≤50 chars)
```

- **Type** — what kind of change (feat, fix, refactor, etc.)
- **Scope** — which part of the codebase (optional but recommended)
- **Description** — imperative mood ("add", not "added" or "adds"), no capital first letter, no period

### Body Rules

```
Implements exponential backoff (3 retries, 2s base delay) when
payment gateway returns 503. Circuit breaker resets after 60s.

Added RetryConfig class to externalise retry parameters.
Integration test covers all retry scenarios including exhaustion.
```

- Wrap at 72 characters (for `git log` readability in terminals)
- Explain **what** and **why**, not how (the diff shows how)
- Separate from subject with a blank line

### Footer Rules

```
Closes: #247
See-also: #198
BREAKING CHANGE: PaymentService constructor now requires RetryConfig param
Co-authored-by: Ankit <ankit@company.com>
Reviewed-by: Shreya <shreya@company.com>
```

---

## 5. Conventional Commits Specification

## What is it?
Conventional Commits is a specification that adds machine-readable structure to commit messages, enabling automated versioning and changelog generation.

## Why It Matters
Tools like `semantic-release` read commit types to auto-determine the next semantic version (feat→minor, fix→patch, BREAKING→major) and generate changelogs.

## Internal Working
The commit message is parsed by tools like `commitlint` and `semantic-release`. The type determines SemVer bump; the scope adds context; BREAKING CHANGE in footer triggers major bump.

## Command Explanation

### Syntax
```bash
git commit -m "<type>(<scope>): <description>"
```

### Overview

[Conventional Commits](https://www.conventionalcommits.org/) is a lightweight convention on top of commit messages. It provides a set of rules for creating an explicit commit history that can be used by automated tools.

### Type Reference

| Type | Description | Triggers SemVer |
|---|---|---|
| `feat` | New feature | MINOR (1.x.0) |
| `fix` | Bug fix | PATCH (1.0.x) |
| `BREAKING CHANGE` | Breaking API change (in footer) | MAJOR (x.0.0) |
| `refactor` | Code change without feature or fix | None |
| `perf` | Performance improvement | None (or PATCH) |
| `test` | Adding/fixing tests | None |
| `docs` | Documentation only | None |
| `style` | Formatting, whitespace (no logic) | None |
| `chore` | Maintenance, dependency updates | None |
| `ci` | CI/CD configuration | None |
| `build` | Build system changes | None |
| `revert` | Reverts a previous commit | Depends |

### Scope Examples

```
feat(auth): ...         ← authentication module
fix(payment): ...       ← payment module
refactor(db): ...       ← database layer
test(api): ...          ← API tests
chore(deps): ...        ← dependencies
ci(github): ...         ← GitHub Actions
```

### Breaking Changes

```bash
# Method 1: Exclamation mark after type/scope
feat(payment)!: change processPayment signature to async

# Method 2: BREAKING CHANGE in footer
feat(payment): change processPayment to return CompletableFuture

BREAKING CHANGE: processPayment() now returns CompletableFuture<PaymentResult>
instead of PaymentResult. All callers must be updated to handle async response.

Migration: PaymentService.processPayment(amount)
        → PaymentService.processPayment(amount).get() for blocking
        → PaymentService.processPayment(amount).thenAccept(...) for async
```

---

## 6. The 7 Rules of a Great Commit Message

## What is it?
Seven universally accepted rules (based on Chris Beams' guidelines) for writing professional commit messages.

## Why It Matters
Following these rules makes `git log` readable, enables tooling, and respects your teammates' (and future self's) time.

## Internal Working
Git itself enforces no message format — all enforcement is via `commit-msg` hooks, `commitlint`, or CI validation.

## Command Explanation

### Syntax
```bash
git commit -m "subject"            # inline (subject only)
git commit                         # opens editor (subject + body + footer)
```

Based on Chris Beams' widely-cited guidelines:

### Rule 1: Separate subject from body with a blank line

```bash
# BAD: no blank line — git log --oneline shows entire thing
git commit -m "fix payment bug this was caused by null amount"

# GOOD: blank line separates subject (shown in oneline) from body
git commit -m "fix(payment): prevent NPE for null amount

Amount was not validated before processing. Added null check
and IllegalArgumentException for amounts <= 0.

Fixes: #521"
```

### Rule 2: Limit subject line to 50 characters

```
feat(payment): add retry logic with exponential backoff      ← 52 chars (trim!)
feat(payment): add exponential backoff retry                 ← 47 chars ✓

GitHub and most git tools truncate subject lines beyond 72 chars.
The 50-char limit is the target; 72 is the hard max.
```

### Rule 3: Capitalise the subject line

```
# BAD
feat(payment): add retry logic
fix(payment): fix null pointer

# GOOD (after type/scope prefix — capitalise the description)
feat(payment): Add retry logic         ← some teams prefer this
# OR (no capitalisation after colon — also widely used)
feat(payment): add retry logic
```

Note: Most Conventional Commits implementations use lowercase after the colon. Pick one and be consistent.

### Rule 4: Do not end subject line with a period

```
# BAD
feat(payment): add retry logic.

# GOOD
feat(payment): add retry logic
```

### Rule 5: Use imperative mood in subject

```
# Imperative (correct) — "If applied, this commit will..."
feat: add payment retry logic
fix: resolve null pointer in payment
refactor: extract payment helper methods
docs: update API documentation

# Past tense (wrong)
feat: added payment retry logic
fix: fixed null pointer in payment

# Present tense (wrong)
feat: adding payment retry logic
fix: fixes null pointer in payment
```

### Rule 6: Wrap body at 72 characters

```
# BAD — one long line
This commit adds retry logic to the payment processor implementing exponential backoff with a maximum of 3 retries and 2 second base delay.

# GOOD — wrapped at 72 chars
This commit adds retry logic to the payment processor. It implements
exponential backoff with a maximum of 3 retries and a 2-second base
delay. The circuit breaker pattern resets after 60 seconds of
successful processing.
```

### Rule 7: Use the body to explain what and why, not how

```
# BAD body — explains HOW (the diff already shows this)
Changed the processPayment method to add a for loop from 0 to 3
and inside the loop added a try-catch block and Thread.sleep().

# GOOD body — explains WHAT and WHY
Payment gateway intermittently returns 503 during peak traffic.
Without retry logic, ~0.3% of valid payment attempts fail. This
change adds three retry attempts with exponential backoff (2s, 4s,
8s delays) to silently handle transient failures, reducing visible
payment errors to near zero.
```

---

## 7. Bad vs Good — Real Examples

## What is it?
Concrete before/after examples of commit messages at bad, mediocre, and professional quality levels.

## Why It Matters
Seeing the contrast makes the rules concrete and actionable.

## Internal Working
N/A — examples only.

## Command Explanation

### Syntax
```bash
git log --oneline -10    # see how messages appear in history
git show HEAD            # see full message + diff
```

### Example 1: Bug Fix

```bash
# BAD
git commit -m "fix bug"

# MEDIOCRE
git commit -m "fix NPE in payment service"

# GOOD
git commit -m "fix(payment): prevent NPE when currency is absent

PaymentService.processPayment() assumed currency field was always
present. When currency is null (older API clients), a NPE was thrown
before amount validation could occur.

Added null coalescing to default currency to USD when absent.
Added unit test for missing-currency scenario.

Fixes: #PROD-4521
Reported-by: monitoring alert at 2026-06-04 09:23 UTC"
```

### Example 2: New Feature

```bash
# BAD
git commit -m "added stuff to payment"

# MEDIOCRE
git commit -m "add retry to payment gateway calls"

# GOOD
git commit -m "feat(payment): add exponential backoff retry to gateway calls

Payment gateway SLA is 99.5% — occasional 503s are expected. Without
retry logic, transient failures reach the user as hard errors.

This implements 3 retries with 2s base delay (doubles each attempt).
Circuit breaker opens after 5 consecutive failures; resets after 60s.

Configuration (application.properties):
  payment.retry.max-attempts=3
  payment.retry.base-delay-ms=2000
  payment.circuit-breaker.failure-threshold=5

Closes: #247
Pair-programmed-with: Ankit <ankit@company.com>"
```

### Example 3: Refactor

```bash
# BAD
git commit -m "refactor code"

# GOOD
git commit -m "refactor(payment): extract PaymentRepository from PaymentService

PaymentService was handling both business logic and data access,
making it untestable without a real database. Extracted all DB
interactions to a dedicated PaymentRepository interface.

- PaymentService now depends on PaymentRepository (injected)
- Added PaymentRepositoryImpl as the JPA implementation
- All existing tests updated to use mock repository
- No functional changes"
```

---

## 8. Commit Message Templates

## What is it?
A commit template is a default message shown in the editor every time you run `git commit` — a prompt reminding developers of the expected format.

## Why It Matters
Reduces friction for adopting message standards — developers see the format on every commit without memorising it.

## Internal Working
Git reads `commit.template` config key, finds the file, and pre-populates the commit editor buffer with its contents.

## Command Explanation

### Syntax
```bash
git config --global commit.template ~/.gitmessage
git config --local  commit.template .gitmessage
```

### Setting a Global Template

```bash
# Create template file
cat > ~/.gitmessage << 'EOF'
# <type>(<scope>): <subject> (50 chars max)
# |<----  Using a maximum of 50 characters  ---->|

# Blank line (DO NOT REMOVE)

# Explain WHY this change is needed (72 chars per line max):
# |<----   Try to limit each line to a maximum of 72 characters   ---->|

# Optional: reference issues, PRs, co-authors
# Closes: #
# BREAKING CHANGE:
# Co-authored-by: Name <email>

# --- TYPE REFERENCE ---
# feat:     New feature
# fix:      Bug fix
# refactor: Code restructure (no feature/fix)
# perf:     Performance improvement
# test:     Add/update tests
# docs:     Documentation only
# style:    Formatting (no logic change)
# chore:    Maintenance, dependencies
# ci:       CI/CD changes
# revert:   Revert a commit
EOF

# Set as global template
git config --global commit.template ~/.gitmessage
```

### Project-Level Template

```bash
# .gitmessage in repo root
echo "commit.template = .gitmessage" >> .git/config

# Or set per-repo
git config --local commit.template .gitmessage
```

---

## 9. Enforcing Standards with Hooks

## What is it?
A `commit-msg` hook is a shell script in `.git/hooks/commit-msg` that Git runs after you write a commit message — it can validate the format and abort the commit if the message is non-compliant.

## Why It Matters
Local enforcement ensures bad messages never enter the repo. Combined with CI-level `commitlint`, enforcement is complete and cannot be bypassed.

## Internal Working
Git passes the commit message FILE PATH as the first argument to `commit-msg`. The hook reads the file, validates the content, and exits 0 (pass) or non-zero (fail/abort commit).

## Command Explanation

### Syntax
```bash
chmod +x .git/hooks/commit-msg
git config core.hooksPath scripts/git-hooks/
```

### `commit-msg` Hook — Validate Conventional Commits

```bash
#!/bin/bash
# .git/hooks/commit-msg

commit_msg_file=$1
commit_msg=$(cat "$commit_msg_file")

# Skip merge commits
if echo "$commit_msg" | grep -qE "^Merge "; then
    exit 0
fi

# Skip revert commits
if echo "$commit_msg" | grep -qE "^Revert "; then
    exit 0
fi

# Conventional Commits pattern
PATTERN="^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert)(\([a-z0-9/-]+\))?(!)?: .{1,100}"

if ! echo "$commit_msg" | grep -qE "$PATTERN"; then
    echo ""
    echo "❌ COMMIT REJECTED: Message does not follow Conventional Commits format."
    echo ""
    echo "   Expected: <type>(<scope>): <description>"
    echo "   Example:  feat(payment): add retry logic"
    echo ""
    echo "   Types: feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert"
    echo ""
    echo "   Your message: $commit_msg"
    echo ""
    exit 1
fi

# Check subject line length (50 chars after type/scope)
subject=$(head -1 "$commit_msg_file")
if [ ${#subject} -gt 72 ]; then
    echo "⚠️  WARNING: Subject line is ${#subject} chars. Recommended max: 72."
fi

exit 0
```

```bash
chmod +x .git/hooks/commit-msg
```

### Using `commitlint` (Node.js Projects)

```bash
npm install --save-dev @commitlint/{cli,config-conventional}

# commitlint.config.js
echo "module.exports = { extends: ['@commitlint/config-conventional'] };" \
  > commitlint.config.js

# Husky integration
npx husky add .husky/commit-msg 'npx --no -- commitlint --edit "$1"'
```

---

## 10. Automated Changelog & Versioning

## What is it?
`semantic-release` and `git-cliff` are tools that read Conventional Commits history to automatically determine the next version number and generate `CHANGELOG.md`.

## Why It Matters
Eliminates manual versioning decisions — every release is semantically correct based on actual commit types. Reduces human error and debate over version numbers.

## Internal Working
These tools run `git log` to read commit messages since the last tag. They parse commit types (feat→minor bump, fix→patch bump, BREAKING CHANGE→major bump), compute the next version, create a tag, and generate changelog text from the commit messages.

## Command Explanation

### Syntax
```bash
npx semantic-release               # automated release in CI
git cliff --latest                 # generate changelog since last tag
git cliff v2.0.0..v2.1.0          # between specific tags
```

### `standard-version` / `semantic-release`

These tools read your Conventional Commits history and:
1. Determine the next semantic version (feat → minor, fix → patch, BREAKING CHANGE → major)
2. Generate/update `CHANGELOG.md`
3. Create a version commit and tag

```bash
# Install semantic-release
npm install --save-dev semantic-release @semantic-release/git @semantic-release/changelog

# .releaserc.json
{
  "branches": ["main"],
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    "@semantic-release/changelog",
    "@semantic-release/npm",
    "@semantic-release/git"
  ]
}

# Run in CI (on push to main)
npx semantic-release
```

### Generated CHANGELOG Example

```markdown
# Changelog

## [2.1.0] - 2026-06-04

### Features
- **payment:** add exponential backoff retry logic (#247)
- **auth:** support OAuth2 PKCE flow (#251)

### Bug Fixes
- **payment:** prevent NPE when currency is absent (#PROD-4521)
- **auth:** resolve JWT expiry edge case on DST change (#238)

### Performance
- **db:** optimise PaymentRepository query with index (#243)

## [2.0.1] - 2026-05-20

### Bug Fixes
- **payment:** fix rounding error for amounts > 1000 (#229)
```

---

## 11. Amending & Editing Messages

## What is it?
`git commit --amend` replaces the last commit (including its message) with a new commit. Interactive rebase (`rebase -i`) edits older commit messages.

## Why It Matters
Fixing bad messages before they are pushed to shared branches is important — once pushed, amending rewrites SHAs and requires force-push.

## Internal Working
`git commit --amend` creates a BRAND NEW commit object with a new SHA (even if only the message changed) and updates the branch pointer. The old commit becomes unreferenced and is eventually GC'd.

## Command Explanation

### Syntax
```bash
git commit --amend -m "new message"
git commit --amend --no-edit
git rebase -i HEAD~n    # reword older commits
```

### Amend Last Commit Message

```bash
# Change message of last commit (not yet pushed)
git commit --amend -m "feat(payment): add exponential backoff retry"

# Open editor to amend
git commit --amend

# Amend without changing message
git commit --amend --no-edit
```

### Edit Older Commit Messages (Interactive Rebase)

```bash
# Edit message of commit 3 ago
git rebase -i HEAD~3

# In editor, change 'pick' to 'reword' for target commit:
pick abc123 bad message here      → reword abc123 bad message here
pick def456 another commit
pick ghi789 third commit

# Save and close — Git opens editor for each 'reword' commit
# Type new message, save → rebase continues

# ⚠️ Rewrites SHAs — only do this on unpushed commits
```

### Fix Multiple Commit Messages in Bulk

```bash
# Change author/email across all commits (after git filter-repo install)
git filter-repo --commit-callback '
  if commit.author_email == b"wrong@email.com":
    commit.author_email = b"correct@email.com"
    commit.committer_email = b"correct@email.com"
'
```

---

## 12. Interview Questions & Model Answers

### Q1: "What makes a good commit message and why does it matter?"

**Model Answer:**
> A good commit message follows the structure: subject line (≤50 chars, imperative mood, no period), blank line, optional body (explains what and why, 72 chars per line), optional footer (issue refs, breaking change notes). The subject should complete the sentence "If applied, this commit will..." The body explains motivation — the diff shows the how; the message should explain the why. This matters because Git is the long-term audit trail of a project. Three months later, `git blame` or `git log` should tell you why a line exists, not just who added it. Good messages enable automated changelog generation, semantic versioning, and better code reviews.

### Q2: "What is Conventional Commits and why would a team adopt it?"

**Model Answer:**
> Conventional Commits is a specification that adds structure to commit messages: `<type>(<scope>): <description>`. Types like `feat`, `fix`, `refactor`, `chore`, `docs`, `test`, `ci` classify each change. The primary benefit is automation — tools like `semantic-release` parse the commit history to determine the next semantic version (feat → minor bump, fix → patch bump, BREAKING CHANGE → major bump) and generate a changelog automatically. This removes manual versioning decisions and ensures releases are always semantically meaningful. Secondary benefits: better `git log` readability, CI job filtering by commit type, and clearer code review context.

### Q3: "How would you enforce commit message standards in a team?"

**Model Answer:**
> Multiple layers: (1) **Local hooks** — `commit-msg` hook validates the message format before the commit is created. Not foolproof since hooks live in `.git/` and aren't committed. Share via `core.hooksPath` pointing to a committed directory, or use Husky/commitlint for Node.js projects. (2) **CI validation** — a pipeline step that runs `commitlint` on every PR's commits against the base branch. This catches any bypassed local hooks. (3) **PR template** — remind developers of the format in the PR description template. (4) **Education** — set a global commit message template (`git config commit.template`) that shows the format every time a developer opens the commit editor. The CI layer is the most reliable enforcement since it can't be bypassed.

---

## 13. Quick Revision Cheatsheet

```bash
# ─── WRITE MESSAGES ──────────────────────────────────────
git commit -m "feat(scope): description"           # inline
git commit                                          # open editor (uses template)
git commit --amend -m "corrected message"          # fix last commit message
git commit --amend                                 # open editor to amend last

# ─── EDIT OLDER MESSAGES ─────────────────────────────────
git rebase -i HEAD~3                               # reword older commits
# change 'pick' to 'reword' for target commit

# ─── SEARCH BY MESSAGE ───────────────────────────────────
git log --grep="feat(payment)"                     # find commits by message
git log --grep="Closes: #247"                      # find by issue reference
git log --oneline --grep="BREAKING"                # find breaking changes

# ─── VALIDATE FORMAT ─────────────────────────────────────
# commitlint (Node.js):
npx commitlint --from HEAD~1 --to HEAD --verbose

# ─── CONFIGURE TEMPLATE ──────────────────────────────────
git config --global commit.template ~/.gitmessage
git config --local  commit.template .gitmessage

# ─── CONVENTIONAL COMMITS TYPES ──────────────────────────
# feat, fix, refactor, perf, test, docs, style, chore, ci, build, revert

# ─── SUBJECT LINE RULES (memory aid) ─────────────────────
# ≤50 chars · imperative mood · no period · capitalise description
# "If applied, this commit will: <your subject>"
```

---

> **Previous:** [07 · Git Interview Master Sheet](./07_Git_Interview_Master_Sheet.md)  
> **Next:** [09 · Production Release Management](./09_Production_Release_Management.md)
