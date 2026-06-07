# 10 · Large-Scale Team Collaboration with Git

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [05 · Remote Repositories & Collaboration](./05_Remote_Repositories_and_Collaboration.md)

---

## Table of Contents

1. [What is Large-Scale Collaboration?](#1-what-is-large-scale-collaboration)
2. [Why It Matters](#2-why-it-matters)
3. [Internal Working — Collaboration Mechanics](#3-internal-working--collaboration-mechanics)
4. [Branch Protection Rules](#4-branch-protection-rules)
5. [CODEOWNERS](#5-codeowners)
6. [PR / MR Best Practices](#6-pr--mr-best-practices)
7. [PR Templates](#7-pr-templates)
8. [Code Review Culture](#8-code-review-culture)
9. [Git Workflows for Large Teams](#9-git-workflows-for-large-teams)
10. [Monorepo vs Polyrepo](#10-monorepo-vs-polyrepo)
11. [Monorepo Tooling & Git Optimisations](#11-monorepo-tooling--git-optimisations)
12. [Access Control & Security](#12-access-control--security)
13. [Scaling Git Performance](#13-scaling-git-performance)
14. [Interview Questions & Model Answers](#14-interview-questions--model-answers)
15. [Quick Revision Cheatsheet](#15-quick-revision-cheatsheet)
16. [Real-World Collaboration Scenarios](#16-real-world-collaboration-scenarios)

---

## 1. What is Large-Scale Collaboration?

## What is it?

Large-scale Git collaboration refers to the practices, tooling, and workflows that enable teams of 10–10,000+ engineers to work on shared codebases safely, efficiently, and with high confidence in code quality.

### Scale Dimensions

| Dimension | Small Team | Large Team |
|---|---|---|
| Developers | 2–10 | 50–10,000+ |
| PRs per day | 1–10 | 100–1,000+ |
| Repository size | < 1 GB | 10 GB – 1 TB+ |
| Branches active | 5–20 | 100–10,000+ |
| Deploy frequency | Weekly | Multiple times/day |

---

## 2. Why It Matters

## What is it?
The business impact of collaboration problems at scale.

## Why It Matters

## Internal Working
At scale, collaboration is enforced through: branch protection rules (platform-level), CODEOWNERS (auto-review assignment), PR templates (context consistency), and CI gates (automated quality). These work together as complementary layers.

## Command Explanation
### Syntax
```bash
# All enforcement is platform-level (GitHub/GitLab settings)
# + git hooks for local enforcement
# + CI validation for server-side enforcement
```

Without deliberate collaboration practices:
- **Merge conflicts** become a daily tax on productivity
- **Quality regressions** slip through without proper review gates
- **Security incidents** from unreviewed code reaching production
- **Knowledge silos** — only one person understands each subsystem
- **Deploy fear** — teams afraid to push because anything could break
- **History pollution** — thousands of "fix", "update" commits with no context

---

## 3. Internal Working — Collaboration Mechanics

## What is it?
How Git's distributed model enables parallel development across large teams without stepping on each other.

## Why It Matters
Understanding the mechanics explains why certain workflows scale and others collapse under team growth.

## Internal Working

### How Remote Branches Enable Collaboration

```
Developer A (local):
  feature/payment  → abc123

Developer B (local):
  feature/auth     → def456

GitHub Remote (shared):
  origin/main                → xyz789
  origin/feature/payment     → abc123  ← A pushed here
  origin/feature/auth        → def456  ← B pushed here
  origin/feature/db-optimize → ghi012  ← C pushed here
```

Each developer works in isolation on their own branch. The remote is the **coordination point** — branches are shared via push/fetch, but nobody's local main is modified by another person's push.

### The Pull Request as a Git Operation

A PR/MR is NOT a Git concept — it is a **platform feature** (GitHub, GitLab) built on top of Git. Internally:

```
When you open a PR from feature/payment → main:
  GitHub shows: git diff main..feature/payment
  GitHub runs:  git merge --no-ff feature/payment (if merge commit strategy)
            OR: git rebase feature/payment onto main (if rebase strategy)
            OR: git merge --squash feature/payment (if squash strategy)
```

The PR adds: review workflow, discussion threads, CI status gates, approval requirements — none of which exist in Git itself.

---

## 4. Branch Protection Rules

## What is it?
Branch protection rules are platform-level policies (GitHub, GitLab, Bitbucket) that prevent certain Git operations on critical branches — enforcing quality gates no developer can bypass.

## Why It Matters
The single most impactful safeguard for team code quality: no code merges to main without passing all checks, regardless of seniority or urgency claims.

## Internal Working
Branch protection is enforced at the Git HOST level (GitHub/GitLab), not inside Git itself. When a push or merge is attempted, the platform checks all configured rules before accepting the operation.

## Command Explanation

### Syntax
```bash
# Configured via GitHub UI: Settings → Branches → Branch protection rules
# Or via GitHub API / Terraform / GitHub CLI:
gh api repos/:owner/:repo/branches/main/protection -X PUT --input rules.json
```

### What They Are

Branch protection rules are **platform-level policies** (GitHub, GitLab, Bitbucket) that prevent certain operations on critical branches. They enforce quality gates that no individual developer can bypass.

### GitHub Branch Protection Configuration

```yaml
# Via GitHub UI: Settings → Branches → Add rule
# Or via GitHub API / Terraform:

Branch: main
Rules:
  ✅ Require a pull request before merging
     - Required approvals: 2                    ← minimum reviewers
     - Dismiss stale reviews on new push        ← re-review after changes
     - Require review from CODEOWNERS           ← domain expert review
  ✅ Require status checks to pass
     - build (GitHub Actions)                   ← must compile
     - test (GitHub Actions)                    ← must pass tests
     - lint (GitHub Actions)                    ← style rules
     - security-scan (Snyk/Dependabot)          ← no known vulnerabilities
  ✅ Require branches to be up to date          ← no stale PRs
  ✅ Require signed commits                     ← GPG-verified authors
  ✅ Include administrators                     ← no one bypasses rules
  ✅ Restrict who can push to matching branches ← only specific teams
  ✅ Require linear history                     ← no merge commits to main
  ✅ Require deployments to succeed before merge ← staging must pass
  ❌ Allow force pushes                         ← NEVER allow on main
  ❌ Allow deletions                            ← NEVER delete main
```

### GitLab Protected Branches (Equivalent)

```yaml
# .gitlab-ci.yml is used for CI gates
# Protected branches configured in Settings → Repository → Protected Branches

Branch: main
  Allowed to push:    No one (all go through MR)
  Allowed to merge:   Maintainers
  Code owner approval: Required
```

### Rulesets (GitHub — More Powerful)

```yaml
# GitHub Rulesets (2023+) — more flexible than branch protection
# Can apply to multiple branches via patterns
# Can be applied at org level across all repos

Ruleset: "production-branches"
Targets: main, release/*
Rules:
  - require_pull_request: { required_approving_review_count: 2 }
  - required_status_checks: { contexts: [build, test, deploy-staging] }
  - non_fast_forward: true       # require linear history
  - update: { update_allows_fetch_and_merge: true }
```

---

## 5. CODEOWNERS

## What is it?
The `CODEOWNERS` file defines which team or individual automatically owns specific parts of the codebase. When a PR touches an owned file, those owners are auto-requested as required reviewers.

## Why It Matters
Scales code review responsibility — payment team reviews payment code, security team reviews auth code — without tech leads manually monitoring every PR.

## Internal Working
GitHub parses the diff's changed file paths against CODEOWNERS patterns (last matching rule wins, like `.gitignore`). Matched owners are added as required reviewers. With branch protection "Require CODEOWNERS review", the PR cannot merge until those owners approve.

## Command Explanation

### Syntax
```
# File: .github/CODEOWNERS or /CODEOWNERS
# Pattern              Owner
*                      @default-team
src/payment/           @payment-team
*.tf                   @devops-team
.github/workflows/     @devops-team
```

### What Is It?

The `CODEOWNERS` file defines which team or individual **automatically owns** specific parts of the codebase. When a PR touches an owned file, those owners are automatically requested as reviewers.

### Location

```
CODEOWNERS can be placed in:
  /CODEOWNERS          (root — most common)
  /.github/CODEOWNERS
  /docs/CODEOWNERS
```

### Syntax

```
# Pattern                    Owner(s)
# ──────────────────────────────────────────────────────────────

# Default owner for everything
*                             @company/backend-team

# Payment module — payment team must review
src/main/java/com/payment/    @company/payment-team

# Auth — security team must review
src/main/java/com/auth/       @company/security-team @senior-security-dev

# Infrastructure / CI — devops team
.github/                      @company/devops-team
infrastructure/               @company/devops-team

# Database migrations — DBA must review
src/main/resources/db/        @company/dba-team @company/backend-leads

# Public API contracts — architects must review
api/openapi/                  @company/architecture-team

# Documentation — anyone from docs team
docs/                         @company/docs-team

# Specific critical file
src/main/java/com/payment/PaymentService.java   @senior-payment-dev @tech-lead

# Terraform — infra team only
*.tf                          @company/infra-team
```

### CODEOWNERS + Branch Protection

```
Branch protection:
  ✅ Require review from CODEOWNERS

Effect:
  PR touching src/payment/ → @payment-team auto-requested
  PR cannot merge until @payment-team approves
  Even if 5 other people approve, still blocked without CODEOWNERS approval
```

### GitHub API — Query CODEOWNERS

```bash
# Check who owns a file
gh api \
  /repos/company/repo/contents/.github/CODEOWNERS \
  | jq -r .content | base64 -d

# List all pending CODEOWNERS reviews
gh pr list --search "review:required"
```

---

## 6. PR / MR Best Practices

## What is it?
PR best practices are guidelines for size, description quality, lifecycle management, and review etiquette that make code review efficient and effective at scale.

## Why It Matters
Poor PR practices are the leading cause of review bottlenecks: too-large PRs get rubber-stamped, too-vague descriptions waste reviewer time, stale PRs block team progress.

## Internal Working
PRs are platform features on top of Git. Internally: a PR compares `git diff base..head`. Merge strategies (merge commit, squash, rebase) correspond to specific Git merge commands executed by the platform.

## Command Explanation

### Syntax
```bash
gh pr create --title "feat(payment): add retry" --draft
gh pr ready <number>          # convert draft to ready
gh pr merge <number> --squash --delete-branch
```

### The Right Size for a PR

```
< 200 lines changed:  Easy to review thoroughly
200–400 lines:        Acceptable; needs good description
400–800 lines:        Hard to review well; split if possible
> 800 lines:          Almost never fully reviewed; split it
```

> **Data:** Research by SmartBear (Code Collaborator) shows review quality degrades sharply after 400 lines. Reviewers can process ~300–500 lines in under 60 minutes; beyond that, defect detection rate drops.

### What Goes in One PR

```
GOOD — one concern per PR:
  ✅ "Add retry logic to PaymentService"
  ✅ "Fix NPE in AuthService"
  ✅ "Refactor UserDAO to use repository pattern"

BAD — multiple concerns:
  ❌ "Add retry logic, fix auth bug, and update dependencies"
  ❌ "Various fixes and improvements"
```

### Draft PRs (Work-in-Progress)

```bash
# Open as draft — signals WIP, blocks accidental merge
gh pr create --draft --title "WIP: payment retry" --body "Early feedback welcome"

# Convert to ready when done
gh pr ready <pr-number>
```

Use draft PRs to:
- Get early architecture feedback before full implementation
- Unblock dependent teams who need to see the interface
- Share work-in-progress without implying it's review-ready

### PR Lifecycle Checklist

```markdown
## Pre-Open
- [ ] Branch is up-to-date with main (`git rebase origin/main`)
- [ ] All CI checks pass locally (`mvn test`)
- [ ] No console.log/System.out.println left in code
- [ ] No TODO comments without ticket reference
- [ ] Self-reviewed the diff (via `git diff main..HEAD`)

## On Open
- [ ] Title follows Conventional Commits format
- [ ] Description explains WHY, not just WHAT
- [ ] Linked to JIRA/GitHub issue
- [ ] Added appropriate labels (feat, fix, breaking-change)
- [ ] Correct milestone set (if applicable)

## During Review
- [ ] Responded to all reviewer comments
- [ ] Addressed or deferred each concern with explanation
- [ ] Requested re-review after significant changes

## Pre-Merge
- [ ] All CI checks green
- [ ] Required approvals obtained (including CODEOWNERS)
- [ ] Branch up-to-date with main (required status check)
- [ ] Squashed WIP commits if needed
```

---

## 7. PR Templates

## What is it?
A PR template is a Markdown file (`.github/pull_request_template.md`) that pre-fills the PR description when a developer opens a new PR on GitHub.

## Why It Matters
Reduces friction for writing good PR descriptions — developers fill in a checklist rather than starting from blank. Ensures consistent context: linked issue, testing steps, checklist.

## Internal Working
GitHub detects `.github/pull_request_template.md` and pre-populates the PR body field. Multiple templates can be used via URL parameter `?template=hotfix.md`.

## Command Explanation

### Syntax
```bash
# File location:
.github/pull_request_template.md          # default template
.github/PULL_REQUEST_TEMPLATE/feature.md  # named templates
.github/PULL_REQUEST_TEMPLATE/hotfix.md
```

### GitHub PR Template

Create `.github/pull_request_template.md`:

```markdown
## Summary
<!-- What does this PR do? Why is it needed? -->


## Type of Change
- [ ] feat: New feature
- [ ] fix: Bug fix
- [ ] refactor: Code refactor (no feature/fix)
- [ ] perf: Performance improvement
- [ ] test: Tests only
- [ ] docs: Documentation
- [ ] chore: Maintenance / dependencies
- [ ] breaking: Breaking change

## Related Issues
<!-- Closes #, Fixes #, Related to # -->
Closes: #

## Changes Made
<!-- Brief list of key changes -->
-
-
-

## How to Test
<!-- Steps to verify this works -->
1.
2.
3.

## Screenshots / Logs
<!-- For UI changes or complex behaviour -->


## Checklist
- [ ] Tests added/updated and passing
- [ ] Documentation updated (if applicable)
- [ ] No secrets or sensitive data in code
- [ ] Backward compatible (or BREAKING CHANGE noted)
- [ ] Performance impact considered
- [ ] Reviewed own diff before requesting review
```

### Multiple PR Templates (GitHub)

```
.github/PULL_REQUEST_TEMPLATE/
├── feature.md          ← For feature PRs
├── bugfix.md           ← For bug fix PRs
├── hotfix.md           ← For emergency fixes
└── dependency.md       ← For dependency updates
```

```markdown
<!-- In PR, append ?template=hotfix.md to URL to select template -->
```

---

## 8. Code Review Culture

## What is it?
Code review culture defines the team norms around how reviews are conducted: tone, comment classification, SLAs, and shared goals.

## Why It Matters
Tech leads are judged on team culture as much as technical decisions. A healthy review culture accelerates learning, catches bugs, and prevents knowledge silos.

## Internal Working
N/A — this is a process topic. The technical mechanism is GitHub's PR comment thread system on top of Git diff output.

## Command Explanation

### Syntax
```bash
gh pr review <number> --approve
gh pr review <number> --request-changes --body "blocking: ..."
gh pr review <number> --comment --body "question: ..."
```

### The Reviewer's Responsibility

```
Code review is NOT about finding fault.
Code review IS about:
  - Sharing knowledge (reviewer learns about feature; author gets fresh eyes)
  - Catching logical bugs before production
  - Ensuring code is maintainable by the whole team
  - Enforcing architectural decisions consistently
```

### Comment Classification (Communicate Intent)

```
blocking:     Must be fixed before merge
              "blocking: This will cause an NPE when amount is null."

suggestion:   Consider it; author decides
              "suggestion: Could we use Optional here instead?"

nit:          Style preference; very minor; author can skip
              "nit: Missing space before opening brace."

question:     Seeking understanding; not blocking
              "question: Why do we use 3 retries specifically?"

praise:       Acknowledge good work
              "praise: Clean solution to the retry problem!"

FYI:          Information sharing; no action required
              "FYI: Java 21 has a built-in retry utility in java.util.concurrent"
```

### Review SLAs (Service Level Agreements)

Large teams need defined expectations:

```
PR size:    SLA for first review:
< 200 lines      4 business hours
200–400 lines    8 business hours
> 400 lines      Next business day

Stale PR (no review after 2 days): 
  → Ping tech lead
  → Consider splitting the PR

Merge within:
  After approval: 24 hours (or it needs re-review)
```

---

## 9. Git Workflows for Large Teams

## What is it?
Workflows adapted for 50–10,000+ engineers: Trunk-Based Development, Scaled GitHub Flow, and environment-branch models.

## Why It Matters
Workflows that work for 5 engineers collapse at 50. Trunk-based + feature flags is the only model proven at Google/Facebook/Netflix scale.

## Internal Working
Trunk-based relies on: short-lived branches (< 2 days), feature flags to hide incomplete work, and fast CI (< 10 min) to make frequent merges safe.

## Command Explanation

### Syntax
```bash
git switch -c fix/payment main   # branch from trunk
# implement behind feature flag
git push -u origin fix/payment
# → PR → merge same day → deploy
```

### Trunk-Based Development (Recommended for High-Velocity Teams)

```
main (trunk):  ──A──B──C──D──E──F──G──H──  (deploys continuously)
                ↑↑↑   ↑↑  ↑↑   ↑↑
           feature branches (< 2 days lifespan)

Rules:
1. Branch from main, merge to main within 1–2 days
2. ALL work behind feature flags (can merge incomplete features)
3. CI must pass before merge
4. No broken code on main — ever
5. At scale: commit directly to trunk (senior engineers only)
```

```bash
# Feature flag example (Java)
if (FeatureFlags.isEnabled("payment-retry", userId)) {
    return paymentService.processWithRetry(amount);
} else {
    return paymentService.process(amount);
}

# Toggle in config (no deploy needed to enable/disable)
feature.flags.payment-retry=true
```

### Scaled Trunk-Based (for 100+ engineers)

```
main:           ──────────────────────────────
                 ↑↑ PRs merge frequently (daily)
team-a/feature: ──────
team-b/fix:           ──
team-c/refactor:        ────────
```

Each team owns their feature flags. Release trains or automated deploy on tag.

### GitHub Flow (10–50 engineers, continuous deployment)

```
main:  ─────────────────────────────────
           ↑PR    ↑PR    ↑PR    ↑PR
feature/A ─────
feature/B        ───────
feature/C               ────────────
```

- Feature branches 2–5 days typically
- PR opened early as draft for discussion
- Branch protection enforces CI + 1–2 reviews
- Deploy immediately after merge

---

## 10. Monorepo vs Polyrepo

## What is it?
Monorepo = all services and libraries in one Git repository. Polyrepo = each service in its own repository.

## Why It Matters
This is a critical architecture decision affecting developer experience, CI/CD speed, dependency management, and team autonomy for years.

## Internal Working
Monorepo: single `.git/` database, all history in one place, sparse checkout and partial clone needed at scale. Polyrepo: separate `.git/` per repo, independent histories, cross-repo changes require multiple PRs.

## Command Explanation

### Syntax
```bash
# Monorepo sparse checkout
git sparse-checkout init --cone
git sparse-checkout set services/payment libs/shared

# Affected files detection (CI optimisation)
git diff --name-only origin/main...HEAD
```

### Monorepo

All services/packages in one repository.

```
company/platform/
├── services/
│   ├── payment-service/
│   ├── auth-service/
│   ├── notification-service/
├── libs/
│   ├── shared-models/
│   ├── api-client/
│   └── test-utils/
├── infrastructure/
│   ├── terraform/
│   └── kubernetes/
└── tools/
    └── build-scripts/
```

**Advantages:**
- Atomic cross-service changes in one PR
- Shared tooling and CI configuration
- Easier code reuse (no version pinning)
- Single source of truth for all code
- Simplified dependency management

**Disadvantages:**
- Slower `git clone` and CI (solution: sparse checkout, shallow clone)
- CI must be smart about what to build (solution: affected-files detection)
- Access control harder (solution: CODEOWNERS, branch protection per path)
- Risk of merge conflicts at scale

### Polyrepo

Each service in its own repository.

```
company/payment-service        ← separate repo
company/auth-service           ← separate repo
company/notification-service   ← separate repo
company/shared-models          ← separate repo (versioned library)
```

**Advantages:**
- Fast clone/CI per repo
- Clear service ownership
- Independent release cycles
- Strict access control per repo

**Disadvantages:**
- Cross-service changes need multiple PRs (hard to keep atomic)
- Dependency versioning complexity (shared-models v1.2 vs v1.3 across services)
- Tooling fragmentation (each repo may evolve differently)
- Discovery — "where is this code?" is harder

### Decision Guide

```
Choose Monorepo when:
  - Tight coupling between services (frequent cross-service changes)
  - Strong desire for atomic changes
  - Shared tooling/CI is a priority
  - Team < 500 engineers (Google/Meta-scale monorepos need custom tooling)

Choose Polyrepo when:
  - Services are truly independent (different tech stacks)
  - Different release/deployment cadences per service
  - Strict team/access isolation required
  - You can solve dependency management cleanly
```

---

## 11. Monorepo Tooling & Git Optimisations

## What is it?
Git features and external tools (Nx, Turborepo, Bazel) that make large monorepos performant for hundreds of engineers.

## Why It Matters
Without optimisation, a 50GB monorepo with 10,000 files makes every `git status`, `git clone`, and CI run unacceptably slow.

## Internal Working
Sparse checkout: Git reads `.git/info/sparse-checkout` to determine which paths to materialise in the working directory. Partial clone: Git fetches blob/tree objects lazily on demand rather than upfront.

## Command Explanation

### Syntax
```bash
git clone --filter=blob:none <url>       # partial clone
git sparse-checkout init --cone
git sparse-checkout set services/payment
git commit-graph write --reachable       # speed up log
git maintenance start                    # background optimisation
```

### Sparse Checkout (Check Out Subset of Files)

```bash
# Only checkout the payment-service directory
git clone --no-checkout git@github.com:company/platform.git
cd platform
git sparse-checkout init --cone
git sparse-checkout set services/payment-service libs/shared-models
git checkout main

# Add more paths later
git sparse-checkout add infrastructure/terraform

# Show current sparse-checkout config
git sparse-checkout list
```

### Partial Clone (Download Only What You Need)

```bash
# Clone without blob objects (metadata only — blobs fetched on demand)
git clone --filter=blob:none git@github.com:company/platform.git

# Clone without tree objects (even less downloaded)
git clone --filter=tree:0 git@github.com:company/platform.git

# Combine with sparse checkout for maximum efficiency
git clone --filter=blob:none --no-checkout git@github.com:company/platform.git
cd platform
git sparse-checkout init --cone
git sparse-checkout set services/payment-service
git checkout main
```

### Affected Files Detection (CI Optimisation)

```bash
# In CI: only build/test what changed
CHANGED_FILES=$(git diff --name-only origin/main...HEAD)

if echo "$CHANGED_FILES" | grep -q "^services/payment"; then
    echo "Running payment service tests"
    cd services/payment-service && mvn test
fi

if echo "$CHANGED_FILES" | grep -q "^services/auth"; then
    echo "Running auth service tests"
    cd services/auth-service && mvn test
fi

if echo "$CHANGED_FILES" | grep -q "^libs/shared-models"; then
    echo "Shared models changed — running ALL tests"
    mvn test --file pom.xml
fi
```

### Nx / Turborepo / Bazel (Monorepo Build Tools)

```bash
# Nx (JavaScript/TypeScript monorepos)
npx nx affected:test            # only test affected projects
npx nx affected:build           # only build affected projects
npx nx graph                    # visualise project dependency graph

# Turborepo
npx turbo run test --filter=[HEAD^1]   # test only changed packages

# Bazel (Google-origin, language-agnostic)
bazel build //services/payment:all
bazel test //services/payment/...
```

---

## 12. Access Control & Security

## What is it?
Access control in Git collaboration means: who can push to which branches, who must review which files, and how commits are verified as authentic.

## Why It Matters
Security breaches, accidental data exposure, and compliance failures often originate from inadequate Git access controls.

## Internal Working
Git itself has no access control — it is enforced by the hosting platform (GitHub/GitLab). GPG-signed commits use Git's object model: the signature is stored in the commit object content.

## Command Explanation

### Syntax
```bash
git config --global commit.gpgsign true
git config --global user.signingkey <KEY-ID>
git verify-commit HEAD
git verify-tag v2.1.0
```

### Repository Permissions

```
GitHub Teams → Repository Access:

team/payment-devs:
  company/platform: Write    ← can push feature branches, open PRs

team/payment-leads:
  company/platform: Maintain ← can manage branches, merge PRs

team/security:
  company/platform: Read     ← can view, comment on PRs

team/devops:
  company/platform: Admin    ← can change settings, manage branch protection
```

### Per-Path Access via CODEOWNERS + Branch Protection

```
CODEOWNERS enforces domain-specific review:
  services/payment/    → @payment-team
  infrastructure/      → @devops-team
  .github/workflows/   → @devops-team    ← CI changes need DevOps approval
  security/            → @security-team  ← security changes need SecEng approval
```

### Git Commit Signing (GPG)

```bash
# Generate GPG key
gpg --full-generate-key
gpg --list-secret-keys --keyid-format LONG

# Configure Git to sign
git config --global user.signingkey <KEY-ID>
git config --global commit.gpgsign true    # sign every commit automatically
git config --global tag.gpgsign true       # sign every tag

# Sign a specific commit
git commit -S -m "feat: add payment retry"

# Sign a tag
git tag -s v2.1.0 -m "Release v2.1.0"

# Verify
git verify-commit HEAD
git verify-tag v2.1.0
```

```
GitHub shows:  [Verified] next to signed commits
               [Unverified] for unsigned or bad signature
```

### Secret Scanning

```yaml
# GitHub Advanced Security — enabled in repo settings
# Automatically scans for known secret patterns:
# API keys, tokens, certificates, passwords

# .github/secret_scanning.yml — add custom patterns
patterns:
  - name: "Internal API Key"
    regex: "COMPANY_API_[A-Z0-9]{32}"
    secret_group: 1
```

---

## 13. Scaling Git Performance

## What is it?
Git configuration and commands that maintain acceptable performance as repositories grow to gigabytes and thousands of developers.

## Why It Matters
A slow `git status` (5+ seconds) destroys developer productivity. A slow CI clone (10+ minutes) kills deployment frequency.

## Internal Working
Performance degrades from: loose object accumulation (fix: `git gc`), large working directories (fix: `core.fsmonitor`), and missing commit graph (fix: `git commit-graph write`).

## Command Explanation

### Syntax
```bash
git gc --aggressive              # deep cleanup
git commit-graph write --reachable  # speed up git log
git config core.fsmonitor true   # faster git status
git maintenance start            # schedule background tasks
```

### For Large Repositories

```bash
# Measure current repo size
git count-objects -vH
# count: 0
# size: 0 bytes
# in-pack: 89,423
# packs: 3
# size-pack: 1.23 GiB

# Aggressive garbage collection and repack
git gc --aggressive --prune=now

# Find large files in history
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print substr($0,6)}' \
  | sort -k2 -rn \
  | head -20

# Remove large files from all history
git filter-repo --strip-blobs-bigger-than 10M

# Enable filesystem monitor (speeds up git status in large repos)
git config core.fsmonitor true         # Git 2.37+
git config core.untrackedCache true
```

### Git Maintenance (Background Optimisation)

```bash
# Run Git maintenance on a schedule (Git 2.31+)
git maintenance start            # register with system scheduler

# Runs these tasks automatically:
# - prefetch:     daily fetch from remotes
# - gc:           weekly garbage collection
# - commit-graph: incremental commit graph updates
# - loose-objects: pack loose objects

# Show maintenance status
git maintenance run --task=gc
```

### Commit Graph (Speed Up `git log`, `git merge-base`)

```bash
# Generate commit graph file
git commit-graph write --reachable --changed-paths

# Verify commit graph
git commit-graph verify

# Auto-update on fetch
git config fetch.writeCommitGraph true
```

---

## 14. Interview Questions & Model Answers

### Q1: "How do you ensure code quality in a large distributed team?"

**Model Answer:**
> Multiple complementary layers. (1) **Branch protection** — enforce minimum 2 reviewers, CI must pass, branches must be up-to-date, CODEOWNERS approval required for domain-specific code. No one, including admins, bypasses this. (2) **CODEOWNERS** — payment code reviewed by payment team, auth code by security team. Auto-requested reviewers mean no PR gets missed. (3) **PR standards** — PR template ensures context, size limits (< 400 lines), draft PRs for early feedback. Review SLAs prevent PRs going stale. (4) **CI gates** — unit tests, integration tests, lint, security scan, coverage threshold all run automatically. (5) **Commit message linting** — `commitlint` in CI ensures searchable, meaningful history. (6) **Post-merge** — monitoring alerts, canary deployments, rollback capability.

### Q2: "Monorepo vs polyrepo — what would you recommend and why?"

**Model Answer:**
> It depends on the coupling and team structure. For a product with tightly coupled services that often change together, a monorepo is superior — atomic cross-service changes, shared tooling, easier refactoring. For truly independent services with different tech stacks, release cycles, or teams, polyrepo avoids the overhead of monorepo tooling at the cost of cross-service coordination complexity. My recommendation for a typical SaaS product: start with a monorepo. Most teams overestimate how independent their services actually are. Add Nx or Turborepo for affected-files detection in CI to avoid building everything on every PR. If a specific service genuinely needs independence (different language, team, or regulatory boundary), extract it to a separate repo at that point — not prematurely.

### Q3: "How would you set up branch protection for a team of 50 engineers?"

**Model Answer:**
> For `main`: require PRs (no direct push, including admins), 2 required reviewers, dismiss stale reviews on new push, require CODEOWNERS review, require all CI checks (build, test, lint, security scan), require branches up-to-date before merge, disallow force pushes, disallow deletion. For `release/*`: similar but with restricted push access — only release managers can create these branches. For `develop`: require PRs with 1 reviewer, CI must pass, no CODEOWNERS requirement (less strict for integration branch). Additionally, I'd use GitHub Rulesets at the organisation level to apply consistent policies across all repos — so new repos created by any team inherit the baseline. Document these rules in the team's engineering handbook so expectations are clear.

### Q4: "How do you handle CI/CD at scale in a monorepo?"

**Model Answer:**
> The key is **affected-files detection** — only build and test what changed. In CI, we compute `git diff --name-only origin/main...HEAD` to get the list of changed files. Each service or library has a defined ownership boundary. If only `services/payment` changed, we only run payment service tests. If `libs/shared-models` changed (a dependency of many services), we run all tests since many services could be affected. Tools like Nx and Turborepo automate this with a project dependency graph. Build caching is also critical — Nx caches build outputs by content hash, so unchanged modules reuse cached results even if the PR touches many files. Remote caching (Nx Cloud, Turborepo Remote Cache) extends this across all CI runs and developer machines.

### Q5: "What is CODEOWNERS and how does it work internally?"

**Model Answer:**
> CODEOWNERS is a file (typically at `/.github/CODEOWNERS`) that maps path patterns to GitHub users or teams. When a PR is opened, GitHub parses the diff's changed files, matches each file path against CODEOWNERS patterns (last matching rule wins, like `.gitignore`), and automatically adds the matched owners as required reviewers. Combined with branch protection's "Require review from CODEOWNERS" setting, the PR cannot merge until those owners approve — regardless of how many other approvals exist. This is powerful for cross-cutting concerns: any change to `.github/workflows/` automatically requires DevOps approval, any change to auth code requires security team review. It scales code review responsibility without requiring team leads to manually monitor every PR.

---

## 15. Quick Revision Cheatsheet

```bash
# ─── CODEOWNERS ───────────────────────────────────────────
# File: .github/CODEOWNERS or /CODEOWNERS
# Pattern → Owner
*                    @default-team
src/payment/         @payment-team
*.tf                 @devops-team

# ─── SPARSE CHECKOUT ──────────────────────────────────────
git clone --no-checkout <url>
git sparse-checkout init --cone
git sparse-checkout set services/payment libs/shared
git checkout main
git sparse-checkout add infrastructure/

# ─── PARTIAL CLONE ────────────────────────────────────────
git clone --filter=blob:none <url>        # no blobs (fetch on demand)
git clone --filter=tree:0 <url>           # no trees

# ─── AFFECTED FILES (CI) ──────────────────────────────────
git diff --name-only origin/main...HEAD
git diff --name-only HEAD~1 HEAD          # last commit

# ─── REPOSITORY HEALTH ────────────────────────────────────
git count-objects -vH                     # size summary
git gc --aggressive --prune=now           # deep cleanup
git maintenance start                     # schedule background maintenance
git commit-graph write --reachable        # speed up log/merge

# ─── SIGNED COMMITS ───────────────────────────────────────
git config --global commit.gpgsign true
git config --global user.signingkey <KEY>
git verify-commit HEAD
git verify-tag v2.1.0

# ─── LARGE FILE DETECTION ─────────────────────────────────
git rev-list --objects --all \
  | git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
  | awk '/^blob/ {print substr($0,6)}' \
  | sort -k2 -rn | head -10
```

---

## 16. Real-World Collaboration Scenarios

### Scenario A: Onboarding a New Engineer

```bash
# 1. Give access (GitHub Org → Add member → Assign team)
# 2. Engineer clones repo
git clone git@github.com:company/platform.git

# 3. For monorepo: sparse checkout their service
git sparse-checkout init --cone
git sparse-checkout set services/payment-service libs/shared-models

# 4. Set local identity
git config --local user.name  "New Engineer"
git config --local user.email "new.engineer@company.com"

# 5. Install hooks
git config core.hooksPath .github/hooks
npm install   # installs Husky if Node project

# 6. First task: create feature branch, open draft PR early
git switch -c feature/NEP-101-first-task
# ... implement ...
gh pr create --draft --title "feat(payment): NEP-101 first task"
```

### Scenario B: Cross-Team Coordination

```bash
# Team A needs an API from Team B that doesn't exist yet

# Team B: create interface in shared lib, open PR
git switch -c feat/shared-payment-interface
# Add interface to libs/shared-models/
git commit -m "feat(shared): add PaymentGatewayInterface contract"
git push origin feat/shared-payment-interface
gh pr create --title "feat(shared): PaymentGatewayInterface contract" \
  --body "Team A needs this interface. Blocking: TEAM-A-PR-123"

# Team A: branch from Team B's branch (don't wait for merge)
git fetch origin
git switch -c feat/payment-impl origin/feat/shared-payment-interface
# Implement against the interface
git commit -m "feat(payment): implement PaymentGatewayInterface"
gh pr create --title "feat(payment): gateway implementation" \
  --body "Depends on: #456 (Team B's interface PR). Merge after that."

# When Team B's PR merges, Team A rebases:
git fetch origin
git rebase origin/main
git push --force-with-lease origin feat/payment-impl
```

### Scenario C: Large Refactor Across Many Files

```bash
# Problem: renaming a method used in 50 files
# Bad approach: one massive 50-file PR (hard to review)

# Good approach: strangler fig pattern
# 1. Add new method alongside old (backward compatible)
git switch -c refactor/add-process-payment-v2
git commit -m "refactor: add processPaymentV2() alongside processPayment()"
gh pr create   # small PR, easy to review

# 2. Migrate callers incrementally (separate PRs per module)
git switch -c refactor/migrate-auth-to-v2
# Update only auth service to use V2
git commit -m "refactor(auth): migrate to processPaymentV2()"
gh pr create   # one module at a time

# 3. Remove old method when all callers migrated
git switch -c refactor/remove-process-payment-v1
git commit -m "refactor!: remove deprecated processPayment()"
# This is the breaking change PR — now reviewers see only the removal
```

---

> **Previous:** [09 · Production Release Management](./09_Production_Release_Management.md)  
> **Next:** [11 · Git Troubleshooting & Disaster Recovery](./11_Git_Troubleshooting_and_Disaster_Recovery.md)
