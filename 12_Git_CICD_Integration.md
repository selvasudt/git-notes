# 12 · Git CI/CD Integration

> **Audience:** Senior SWE · Backend Engineer · DevOps · Tech Lead interviews  
> **Prerequisite:** [05 · Remote Repositories & Collaboration](./05_Remote_Repositories_and_Collaboration.md)

---

## Table of Contents

1. [What is Git CI/CD Integration?](#1-what-is-git-cicd-integration)
2. [Why It Matters](#2-why-it-matters)
3. [Internal Working — Git Events as Triggers](#3-internal-working--git-events-as-triggers)
4. [Git Events & Webhooks](#4-git-events--webhooks)
5. [GitHub Actions — Deep Dive](#5-github-actions--deep-dive)
6. [GitLab CI/CD — Deep Dive](#6-gitlab-cicd--deep-dive)
7. [CI Best Practices with Git](#7-ci-best-practices-with-git)
8. [CD Patterns with Git](#8-cd-patterns-with-git)
9. [GitOps](#9-gitops)
10. [Branch-Based Environments](#10-branch-based-environments)
11. [Security in Git CI/CD](#11-security-in-git-cicd)
12. [Interview Questions & Model Answers](#12-interview-questions--model-answers)
13. [Quick Revision Cheatsheet](#13-quick-revision-cheatsheet)
14. [Real-World Pipeline Examples](#14-real-world-pipeline-examples)

---

## 1. What is Git CI/CD Integration?

## What is it?
Git CI/CD integration uses Git events (pushes, PRs, tag pushes) as automatic triggers for pipelines that build, test, and deploy software.

## Why It Matters
CI/CD removes manual steps between writing code and deploying it — every commit is verifiably tested, every release is automated, every deployment is traceable to a specific Git SHA.

## Internal Working
Git hosts (GitHub, GitLab) register webhooks. A `git push` fires an HTTP POST to the CI service with the event payload (commit SHA, ref, changed files). The CI reads the pipeline config (`.github/workflows/*.yml`), spins up a runner, clones at that exact SHA, and executes steps.

## Command Explanation

### Syntax
```bash
# In CI environment these are available automatically:
echo $GITHUB_SHA          # full commit SHA (GitHub Actions)
echo $CI_COMMIT_SHA       # full commit SHA (GitLab CI)
echo $GITHUB_REF          # refs/heads/main or refs/tags/v2.1.0
```

### Definition

**Git CI/CD integration** means using Git events (pushes, PRs, tags) as **triggers** for automated pipelines that build, test, and deploy software. Git is the single source of truth — every action the CI/CD system takes is traceable to a specific commit SHA.

### The Pipeline Principle

```
Developer Action (Git) → Event → CI/CD Pipeline → Outcome

git push origin feature/payment  → PR CI:   build + test + lint
git push origin main             → CD:      build + test + deploy staging
git push origin tag v2.1.0       → Release: build + test + deploy production
```

---

## 2. Why It Matters

## What is it?
The concrete business impact of CI/CD integration with Git.

## Why It Matters

## Internal Working
N/A — business impact comparison.

## Command Explanation

### Syntax
```bash
# Tag push triggers production deploy:
git tag -a v2.1.0 -m "Release"
git push origin v2.1.0    # → CI detects tag → runs release pipeline
```

```
Without Git CI/CD:              With Git CI/CD:
  Manual builds                   Every push auto-builds
  "Works on my machine"           Tested in clean environment every time
  Release = stressful event       Release = push a tag
  Unknown what's deployed         Git SHA pinned to every deployment
  Broken code on main             Main always green (gates enforced)
  Manual rollback                 Automated rollback on health check failure
```

---

## 3. Internal Working — Git Events as Triggers

## What is it?
The technical flow from a `git push` command to a running CI pipeline.

## Why It Matters
Understanding the event flow explains why pipelines can be slow to start (runner provisioning), why SHAs are the build identity, and why builds are reproducible.

## Internal Working

### How GitHub Actions Receives Events

```
git push origin feature/payment
       ↓
GitHub receives push via HTTPS/SSH
       ↓
GitHub fires webhook to Actions service
       ↓
Actions service reads: .github/workflows/*.yml
       ↓
Matches on: push, branches: [feature/*]
       ↓
Spins up runner, checks out code at that exact commit SHA
       ↓
Executes pipeline steps
       ↓
Reports status back to GitHub commit (green/red check)
```

### The Commit SHA as the Build Identity

Every CI/CD run is anchored to a specific commit SHA:

```bash
# In any CI runner, these are available:
echo $GITHUB_SHA         # full commit SHA
echo $GITHUB_REF         # refs/heads/main or refs/tags/v2.1.0
echo $GITHUB_HEAD_REF    # PR branch name
echo $GITHUB_BASE_REF    # PR target branch
```

This means builds are **reproducible** — you can re-run the pipeline against any past commit and get the same result (assuming deterministic builds).

---

## 4. Git Events & Webhooks

## What is it?
Webhooks are HTTP callbacks from the Git host to the CI system, fired on specific Git events (push, PR open, tag create).

## Why It Matters
Choosing the right event trigger determines: what gets tested, what gets deployed, and how fast feedback reaches developers.

## Internal Working
GitHub stores webhook configurations per repository. On each qualifying event, GitHub sends an HTTP POST to the registered URL with a JSON payload containing: commit SHA, ref name, changed files, and repository info.

## Command Explanation

### Syntax
```yaml
# GitHub Actions trigger syntax:
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  push:
    tags: ['v*']
```

### Key Git Events for CI/CD

| Event | Trigger | Typical Pipeline |
|---|---|---|
| `push` to feature branch | Developer pushes | Build + Unit Tests + Lint |
| Pull Request opened | PR created | Build + Tests + Security Scan + Review gates |
| Pull Request updated | New push to PR branch | Re-run all PR checks |
| `push` to main | PR merged | Full test suite + Deploy to staging |
| Tag pushed (`v*`) | Release created | Full pipeline + Deploy to production |
| Schedule (cron) | Time-based | Nightly security scan, dependency audit |
| Manual dispatch | Triggered manually | Ad-hoc builds, redeployments |

### GitHub Webhook Payload (Key Fields)

```json
{
  "ref": "refs/heads/feature/payment",
  "before": "abc123def456...",         // previous SHA
  "after":  "def456ghi789...",         // new SHA (what to build)
  "repository": { "full_name": "company/repo" },
  "pusher": { "name": "shrayansh" },
  "commits": [
    {
      "id": "def456ghi789...",
      "message": "feat(payment): add retry logic",
      "author": { "name": "Shrayansh" }
    }
  ]
}
```

---

## 5. GitHub Actions — Deep Dive

## What is it?
GitHub Actions is GitHub's built-in CI/CD system. Workflows are YAML files in `.github/workflows/` that define triggered pipelines of jobs and steps.

## Why It Matters
GitHub Actions is the most widely used CI/CD platform. Understanding its structure, triggers, caching, and matrix builds is essential for DevOps interviews.

## Internal Working
GitHub Actions reads `.github/workflows/*.yml` on every triggering event. A runner (GitHub-hosted or self-hosted) is provisioned, the repo is cloned at the triggering commit SHA, and steps execute in sequence within each job.

## Command Explanation

### Syntax
```yaml
# Minimal workflow structure:
name: CI
on:
  push:
    branches: [main]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: echo "Build step"
```

### Workflow File Structure

```yaml
# .github/workflows/ci.yml
name: CI                           # workflow name (shown in Actions tab)

on:                                # trigger events
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]
  workflow_dispatch:               # manual trigger button

env:                               # workflow-level environment variables
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io

jobs:
  build:                           # job name
    runs-on: ubuntu-latest         # runner OS
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0           # 0 = full history (needed for git describe)
          # fetch-depth: 1         # shallow clone (faster, no history)

      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'           # cache ~/.m2

      - name: Build
        run: mvn clean package -DskipTests

      - name: Test
        run: mvn test

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()               # upload even if tests failed
        with:
          name: test-results
          path: target/surefire-reports/
```

### PR-Specific Workflow

```yaml
# .github/workflows/pr.yml
name: PR Validation

on:
  pull_request:
    branches: [main]
    types: [opened, synchronize, reopened]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # needed to compare with base branch

      - name: Check changed files
        id: changed
        run: |
          CHANGED=$(git diff --name-only origin/${{ github.base_ref }}...HEAD)
          echo "files=$CHANGED" >> $GITHUB_OUTPUT
          if echo "$CHANGED" | grep -q "^src/"; then
            echo "src_changed=true" >> $GITHUB_OUTPUT
          fi

      - name: Run tests (if src changed)
        if: steps.changed.outputs.src_changed == 'true'
        run: mvn test

      - name: Lint commit messages
        run: |
          # Check all commits in this PR follow Conventional Commits
          git log origin/${{ github.base_ref }}..HEAD --format="%s" | \
          while read msg; do
            echo "$msg" | grep -qE \
              "^(feat|fix|docs|style|refactor|perf|test|chore|ci|build|revert)(\(.+\))?: .+" \
              || (echo "BAD MESSAGE: $msg" && exit 1)
          done

      - name: Comment PR with test coverage
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '✅ All checks passed. Coverage: 87%'
            })
```

### Release Pipeline (Tag-Triggered)

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags:
      - 'v[0-9]+.[0-9]+.[0-9]+'    # match v2.1.0 but not v2.1.0-rc.1

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write    # needed to create GitHub Release
      packages: write    # needed to push to GitHub Container Registry

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Extract version
        id: version
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_OUTPUT

      - name: Build release artifact
        run: |
          mvn clean package -DskipTests
          mvn versions:set -DnewVersion=${{ steps.version.outputs.VERSION }}

      - name: Run full test suite
        run: mvn verify

      - name: Build and push Docker image
        run: |
          docker build \
            --label "git.sha=${{ github.sha }}" \
            --label "version=${{ steps.version.outputs.VERSION }}" \
            -t ${{ env.REGISTRY }}/company/payment-service:${{ steps.version.outputs.VERSION }} \
            -t ${{ env.REGISTRY }}/company/payment-service:latest \
            .
          docker push ${{ env.REGISTRY }}/company/payment-service:${{ steps.version.outputs.VERSION }}
          docker push ${{ env.REGISTRY }}/company/payment-service:latest

      - name: Generate changelog
        run: git cliff --latest --output RELEASE_NOTES.md

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          name: "Release ${{ steps.version.outputs.VERSION }}"
          body_path: RELEASE_NOTES.md
          files: |
            target/*.jar
          draft: false
          prerelease: ${{ contains(steps.version.outputs.VERSION, '-') }}

      - name: Deploy to production
        run: |
          kubectl set image deployment/payment-service \
            payment-service=${{ env.REGISTRY }}/company/payment-service:${{ steps.version.outputs.VERSION }}
          kubectl rollout status deployment/payment-service --timeout=300s
        env:
          KUBECONFIG: ${{ secrets.KUBECONFIG }}
```

### Matrix Builds

```yaml
jobs:
  test:
    strategy:
      matrix:
        java: ['17', '21']
        os: [ubuntu-latest, windows-latest]
      fail-fast: false    # run all combinations even if one fails
    
    runs-on: ${{ matrix.os }}
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java }}
          distribution: 'temurin'
      - run: mvn test
```

### Caching in GitHub Actions

```yaml
# Cache Maven dependencies
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: maven-

# Cache Node modules
- uses: actions/cache@v4
  with:
    path: node_modules
    key: node-${{ hashFiles('package-lock.json') }}
    restore-keys: node-

# Cache Docker layers
- uses: actions/cache@v4
  with:
    path: /tmp/.buildx-cache
    key: docker-${{ github.sha }}
    restore-keys: docker-
```

---

## 6. GitLab CI/CD — Deep Dive

## What is it?
GitLab CI/CD is GitLab's built-in pipeline system configured via `.gitlab-ci.yml`. It supports stages, templates (YAML anchors), environments, and manual approval gates.

## Why It Matters
GitLab CI is dominant in enterprise and self-hosted environments. Its environment promotion model (staging → production with manual gate) is a common interview topic.

## Internal Working
GitLab registers a GitLab Runner (shared or project-specific). On push, GitLab reads `.gitlab-ci.yml`, evaluates `only`/`except`/`rules` conditions, and queues matching jobs. Runners poll for jobs, execute, and report status back.

## Command Explanation

### Syntax
```yaml
# Minimal .gitlab-ci.yml:
stages:
  - build
  - test
  - deploy

build:
  stage: build
  script:
    - mvn package -DskipTests
  only:
    - merge_requests
    - main
```

### `.gitlab-ci.yml` Structure

```yaml
# .gitlab-ci.yml

stages:
  - build
  - test
  - security
  - deploy-staging
  - deploy-production

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=$CI_PROJECT_DIR/.m2/repository"
  REGISTRY: $CI_REGISTRY_IMAGE

# Global cache
cache:
  paths:
    - .m2/repository/

# Templates (reusable anchors)
.java-template: &java-template
  image: eclipse-temurin:21
  before_script:
    - java -version
    - mvn -version

build:
  <<: *java-template
  stage: build
  script:
    - mvn clean package -DskipTests
  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 hour
  only:
    - merge_requests
    - main
    - tags

test:
  <<: *java-template
  stage: test
  script:
    - mvn verify
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
    paths:
      - target/site/jacoco/
  coverage: '/Total.*?([0-9]{1,3})%/'

security-scan:
  stage: security
  image: owasp/dependency-check:latest
  script:
    - dependency-check.sh --project "$CI_PROJECT_NAME"
      --scan . --format JSON --out reports/
  artifacts:
    paths: [reports/]
  allow_failure: true    # don't block pipeline, but report

deploy-staging:
  stage: deploy-staging
  script:
    - kubectl set image deployment/payment-service
        payment-service=$REGISTRY:$CI_COMMIT_SHA
  environment:
    name: staging
    url: https://staging.company.com
  only:
    - main

deploy-production:
  stage: deploy-production
  script:
    - kubectl set image deployment/payment-service
        payment-service=$REGISTRY:$CI_COMMIT_TAG
  environment:
    name: production
    url: https://company.com
  when: manual             # requires human approval
  only:
    - tags
```

### GitLab CI/CD Variables (Git-Specific)

```bash
CI_COMMIT_SHA           # full commit SHA
CI_COMMIT_SHORT_SHA     # short SHA (first 8 chars)
CI_COMMIT_BRANCH        # branch name (for push events)
CI_COMMIT_TAG           # tag name (for tag events)
CI_COMMIT_MESSAGE       # full commit message
CI_MERGE_REQUEST_IID    # MR number
CI_PIPELINE_SOURCE      # push, merge_request_event, schedule, etc.
```

---

## 7. CI Best Practices with Git

## What is it?
The set of Git-specific best practices that make CI pipelines secure, fast, and reproducible.

## Why It Matters
Bad CI practices cause: supply chain attacks (unpinned actions), slow builds (no caching), untraceable deployments (no SHA embedding), and credential leaks (secrets in code).

## Internal Working
Pinning actions to SHA (`uses: actions/checkout@abc123`) prevents a compromised action tag from injecting malicious code. Shallow clones (`--depth 1`) reduce clone time by downloading only the latest commit's objects.

## Command Explanation

### Syntax
```bash
# Pin GitHub Actions to SHA (not mutable tag):
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# Affected files detection in CI:
CHANGED=$(git diff --name-only origin/${BASE_BRANCH}...HEAD)

# Embed SHA in Docker image:
docker build --label "git.sha=${GITHUB_SHA}" -t myimage .
```

### 1. Always Pin the Exact Commit SHA

```yaml
# BAD: uses mutable tag (latest could change)
uses: actions/checkout@main

# GOOD: pin to exact SHA (immutable)
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
```

### 2. Use Shallow Clones in CI (Unless You Need History)

```yaml
# Default: shallow clone (1 commit) — fast
- uses: actions/checkout@v4
  with:
    fetch-depth: 1

# When you need tags (for git describe, semver):
- uses: actions/checkout@v4
  with:
    fetch-depth: 0
    fetch-tags: true

# When you need to compare with base branch (for affected files):
- uses: actions/checkout@v4
  with:
    fetch-depth: 0  # need enough history to find merge-base
```

### 3. Detect Changed Files to Optimise CI

```bash
# In CI, only run affected tests
CHANGED=$(git diff --name-only origin/${BASE_BRANCH}...HEAD)

if echo "$CHANGED" | grep -q "^services/payment/"; then
    echo "Running payment service tests"
    mvn test -pl services/payment
fi

if echo "$CHANGED" | grep -q "^services/auth/"; then
    echo "Running auth service tests"
    mvn test -pl services/auth
fi
```

### 4. Always Report the Git SHA in Deployments

```bash
# Embed SHA in Docker image label
docker build \
  --label "git.sha=${GITHUB_SHA}" \
  --label "git.branch=${GITHUB_REF_NAME}" \
  --label "build.time=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  -t myimage:latest .

# Embed in application (Spring Boot)
# application.properties:
management.info.git.mode=full
spring.git.properties=true
# → /actuator/info shows git.commit.id, git.branch, etc.
```

### 5. Never Store Secrets in Git

```bash
# BAD — hardcoded secret in workflow
env:
  DB_PASSWORD: "mypassword123"

# GOOD — use GitHub Secrets
env:
  DB_PASSWORD: ${{ secrets.DB_PASSWORD }}

# GOOD — use OIDC (no static secrets for cloud providers)
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/github-actions-role
    aws-region: us-east-1
# → Uses short-lived OIDC tokens, no static AWS keys needed
```

### 6. Fail Fast, Fail Clearly

```yaml
# Add timeout to prevent hung jobs
jobs:
  test:
    timeout-minutes: 30      # fail if job takes > 30 min

steps:
  - name: Test
    run: mvn test
    timeout-minutes: 20      # fail if tests take > 20 min
```

---

## 8. CD Patterns with Git

## What is it?
The different models for how Git events trigger deployment: push-based (CI deploys directly), pull-based/GitOps (agent watches Git), tag-based (release on tag push), and canary/blue-green strategies.

## Why It Matters
Choosing the right CD pattern determines deployment safety, rollback speed, and auditability. GitOps is the modern standard for Kubernetes.

## Internal Working
Push-based: CI runner has credentials and calls deploy API directly after build. Pull-based (GitOps): an agent in the cluster watches a Git repo and applies changes — no inbound credentials needed in CI.

## Command Explanation

### Syntax
```bash
# Push-based deploy:
kubectl set image deployment/svc container=image:${GITHUB_SHA}
kubectl rollout status deployment/svc --timeout=300s

# Tag-based release trigger:
git tag -a v2.1.0 -m "Release"
git push origin v2.1.0    # triggers release pipeline
```

### Pattern 1: Push-Based Deployment (Most Common)

```
CI/CD System pushes to target (Kubernetes, server, cloud)

git push → CI builds → CI runs: kubectl apply / helm upgrade / aws deploy
```

```yaml
- name: Deploy to Kubernetes
  run: |
    kubectl set image deployment/payment-service \
      container=$REGISTRY/payment-service:${{ github.sha }}
    kubectl rollout status deployment/payment-service --timeout=300s
    
    # Rollback on failure
    kubectl rollout undo deployment/payment-service
```

### Pattern 2: Pull-Based Deployment (GitOps — Preferred for K8s)

```
Git repo is the desired state → Agent in cluster pulls and applies

git push manifest changes → GitOps agent detects drift → applies changes
```

See [Section 9 — GitOps](#9-gitops) for full detail.

### Pattern 3: Deploy on Tag

```yaml
on:
  push:
    tags: ['v*']

# Only production deploys happen on tag push
# All other pushes to main only deploy to staging
```

### Pattern 4: Blue-Green Deployment via Git

```bash
# In CI/CD after build:
# 1. Deploy new version to "green" environment
kubectl apply -f k8s/green-deployment.yaml

# 2. Run smoke tests against green
./scripts/smoke-test.sh https://green.company.com

# 3. Switch traffic (update service selector)
kubectl patch service payment-service \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# 4. Monitor for 15 min
sleep 900
./scripts/health-check.sh

# 5. On failure: switch back
kubectl patch service payment-service \
  -p '{"spec":{"selector":{"slot":"blue"}}}'
```

### Pattern 5: Canary Release

```bash
# Deploy new version to 10% of pods
kubectl apply -f k8s/canary-deployment.yaml   # 1 of 10 replicas = new version

# Monitor error rate for 30 min
sleep 1800
ERROR_RATE=$(./scripts/check-error-rate.sh)

if [ "$ERROR_RATE" -gt "1" ]; then
  # Rollback canary
  kubectl delete deployment payment-canary
  echo "Canary failed: error rate $ERROR_RATE%"
  exit 1
fi

# Promote canary to full deployment
kubectl apply -f k8s/full-deployment.yaml
kubectl delete deployment payment-canary
```

---

## 9. GitOps

## What is it?
GitOps is a continuous delivery paradigm where Git is the single source of truth for infrastructure and deployment configuration. An agent continuously reconciles the actual cluster state with the desired state declared in Git.

## Why It Matters
GitOps provides: full audit trail (every deployment = a Git commit), automatic drift correction, pull-based security model (no inbound credentials from CI), and instant rollback (revert the config commit).

## Internal Working
An ArgoCD or Flux agent runs inside the Kubernetes cluster. It watches a Git repo at a configured interval. When it detects a diff between the repo's manifests and the cluster's actual state, it applies the Git version. This is a PULL model — the cluster fetches from Git, not the other way around.

## Command Explanation

### Syntax
```bash
# Update GitOps repo from CI (triggers ArgoCD sync):
sed -i "s|image:.*|image: myapp:${GITHUB_SHA}|" k8s/deployment.yaml
git commit -am "ci: update image to ${GITHUB_SHA}"
git push
# ArgoCD detects change → syncs cluster → done
```

### What Is GitOps?

GitOps is a CD paradigm where **Git is the single source of truth for infrastructure and deployment configuration**. The desired state of the system is declared in Git; an agent continuously reconciles actual state with desired state.

```
Developer → git push (update Kubernetes manifest) 
→ GitOps agent (ArgoCD/Flux) detects diff
→ Applies changes to cluster
→ Reports sync status back to Git (commit status)
```

### GitOps Principles

1. **Declarative** — system state described declaratively (YAML manifests, Helm values)
2. **Versioned** — all changes via Git commits (full audit trail)
3. **Pulled** — agent pulls from Git (not push from CI)
4. **Continuously reconciled** — agent constantly ensures cluster = Git state

### ArgoCD Example

```yaml
# argocd-app.yaml — Application definition in ArgoCD
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: payment-service
spec:
  project: default
  source:
    repoURL: git@github.com:company/platform.git
    targetRevision: main          # or a specific tag
    path: k8s/payment-service    # Kubernetes manifests path
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true                 # delete resources removed from Git
      selfHeal: true              # re-apply if someone manually changes cluster
    syncOptions:
      - CreateNamespace=true
```

### GitOps Repository Structure

```
infrastructure-repo/
├── apps/
│   ├── payment-service/
│   │   ├── base/                   # base Kubernetes manifests
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── staging/            # staging-specific overrides
│   │       │   └── kustomization.yaml  # image: payment-service:main-abc123
│   │       └── production/         # production-specific overrides
│   │           └── kustomization.yaml  # image: payment-service:v2.1.0
├── clusters/
│   ├── staging/
│   └── production/
└── README.md
```

### CI → GitOps Integration

```yaml
# CI pipeline: after building image, update the GitOps repo
- name: Update GitOps repo with new image
  run: |
    git clone git@github.com:company/infrastructure.git
    cd infrastructure
    
    # Update image tag in staging overlay
    sed -i "s|payment-service:.*|payment-service:${{ github.sha }}|g" \
      apps/payment-service/overlays/staging/kustomization.yaml
    
    git config user.email "ci-bot@company.com"
    git config user.name "CI Bot"
    git commit -am "ci: update payment-service to ${{ github.sha }}"
    git push
    # → ArgoCD detects change → auto-deploys to staging
```

---

## 10. Branch-Based Environments

## What is it?
Branch-based environments automatically create and destroy isolated deployment environments tied to Git branch lifecycle events (PR opened → environment created, PR merged → environment destroyed).

## Why It Matters
PR preview environments let reviewers test changes in a real environment before merge — catches issues impossible to find in code review alone.

## Internal Working
CI detects PR open event (`github.event.action == 'opened'`), creates a Kubernetes namespace named after the PR number, deploys the branch's Docker image there, posts the URL as a PR comment. On PR close, the namespace is deleted.

## Command Explanation

### Syntax
```bash
# Create preview environment:
kubectl create namespace pr-${PR_NUMBER}
helm upgrade --install app-preview ./helm \
  --namespace pr-${PR_NUMBER} \
  --set image.tag=${GITHUB_SHA} \
  --set ingress.host="pr-${PR_NUMBER}.preview.company.com"

# Cleanup on PR close:
kubectl delete namespace pr-${PR_NUMBER}
```

### Preview Environments (Per PR)

```yaml
# .github/workflows/preview.yml
on:
  pull_request:
    types: [opened, synchronize, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy preview environment
        run: |
          PREVIEW_NAME="pr-${{ github.event.number }}"
          kubectl create namespace $PREVIEW_NAME --dry-run=client -o yaml | kubectl apply -f -
          
          helm upgrade --install payment-service-preview ./helm/payment-service \
            --namespace $PREVIEW_NAME \
            --set image.tag=${{ github.sha }} \
            --set ingress.host="${PREVIEW_NAME}.preview.company.com"
          
      - name: Comment preview URL
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚀 Preview: https://pr-${{ github.event.number }}.preview.company.com'
            })

  cleanup-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Delete preview environment
        run: |
          kubectl delete namespace pr-${{ github.event.number }}
```

### Environment Promotion Flow

```
Feature branch push
  → CI: build + test + lint
  → Deploy to PR preview environment
  → PR approved + CI green
  → Merge to main
  → CI: build + test
  → Deploy to staging (auto)
  → QA approval (manual gate)
  → Deploy to production (on tag push or manual trigger)
```

---

## 11. Security in Git CI/CD

## What is it?
Security practices for Git-integrated CI/CD pipelines: OIDC authentication (no static secrets), dependency pinning, secret scanning, and least-privilege access.

## Why It Matters
CI/CD pipelines have elevated privileges — they build, push, and deploy. A compromised pipeline is a full supply chain attack. Security here is not optional.

## Internal Working
OIDC: GitHub Actions requests a short-lived JWT from GitHub's OIDC provider. AWS/GCP/Azure exchanges this JWT for cloud credentials. No static keys stored anywhere — credentials expire automatically after minutes.

## Command Explanation

### Syntax
```yaml
# OIDC in GitHub Actions (no static AWS keys):
permissions:
  id-token: write
  contents: read
steps:
  - uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123:role/github-actions
      aws-region: us-east-1
```

### OIDC — No Static Secrets (Best Practice)

```yaml
# GitHub Actions → AWS without static credentials
permissions:
  id-token: write    # needed for OIDC
  contents: read

jobs:
  deploy:
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions
          aws-region: us-east-1
          # → GitHub gets short-lived OIDC token, exchanges for AWS credentials
          # → No static AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY needed

      - name: Deploy
        run: aws ecs update-service --cluster production --service payment ...
```

### Dependency Pinning in Workflows

```yaml
# Pin ALL GitHub Actions to specific SHA (not mutable tags)
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
uses: actions/setup-java@3232623d9a428cc5baf68dbfe737b8dd8ef9e8a   # v4.7.0

# Use Dependabot to automatically keep these updated:
# .github/dependabot.yml:
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

### Secret Scanning in CI

```yaml
# Run truffleHog in CI to detect secrets
- name: Secret scan
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.pull_request.base.sha }}
    head: ${{ github.event.pull_request.head.sha }}
    extra_args: --debug --only-verified
```

---

## 12. Interview Questions & Model Answers

### Q1: "How do Git events trigger CI/CD pipelines?"

**Model Answer:**
> When you push to GitHub, GitHub receives the push over HTTPS/SSH and fires a webhook event to the CI service (GitHub Actions, Jenkins, GitLab CI). The webhook payload contains the commit SHA, branch/tag reference, and changed files. The CI system reads the pipeline configuration (`.github/workflows/*.yml`) and matches the event to defined triggers. A runner is spun up, the repository is checked out at the exact commit SHA, and the pipeline steps execute. The result (pass/fail) is reported back to GitHub via the Commit Status API, which powers the branch protection "status checks must pass" gate. This makes every commit verifiably tested before it can merge to main.

### Q2: "What is GitOps and how does it differ from traditional CI/CD?"

**Model Answer:**
> GitOps is a continuous delivery model where Git is the single source of truth for both application code and infrastructure/deployment configuration. In traditional push-based CI/CD, the CI system pushes changes to the target environment after building. In GitOps, an agent (like ArgoCD or Flux) running inside the target cluster continuously watches a Git repository and pulls any changes, applying them automatically. The key differences: (1) Git is fully declarative — you describe desired state, not imperative commands. (2) It's pull-based, so the cluster network only needs outbound access to Git, not inbound from CI. (3) Full audit trail — every deployment is a Git commit. (4) Drift detection — the agent continuously reconciles; if someone manually changes the cluster, it gets reverted to the Git-defined state. GitOps is particularly well-suited for Kubernetes environments.

### Q3: "How do you handle secrets in CI/CD pipelines that use Git?"

**Model Answer:**
> Never put secrets in Git — not even encrypted. The right approach depends on the cloud: (1) Use OIDC where possible — GitHub Actions can get short-lived AWS/GCP/Azure credentials without storing static keys, using federated identity. (2) Platform secrets for everything else — GitHub Secrets, GitLab CI variables marked as protected/masked, or a secrets manager (HashiCorp Vault, AWS Secrets Manager) that the CI runner fetches from at runtime. (3) Pin all GitHub Actions to specific SHAs — a compromised action could exfiltrate secrets. (4) Use minimal permissions — each job's GitHub token should only have the permissions it actually needs (via `permissions:` block). (5) Audit secrets regularly — rotate them, and use secret scanning tools like truffleHog in CI to detect any accidental commits.

### Q4: "How do you implement a deployment pipeline that deploys staging on every main merge and production only on tag push?"

**Model Answer:**
> Two separate workflow files in `.github/workflows/`. The staging workflow triggers on `push: branches: [main]` — it builds, runs the full test suite, and deploys to staging automatically. The production workflow triggers on `push: tags: ['v*']` — it builds, runs tests, generates a changelog, creates a GitHub Release, and deploys to production. The tag must be created explicitly by an authorised person (`git tag -a v2.1.0 -m "..." && git push origin v2.1.0`). This creates a clear promotion boundary: every main merge is automatically in staging, but production requires a deliberate versioning decision. I'd add a manual approval gate on the production deploy step using GitHub Environments with required reviewers, so even a tag push requires human confirmation before touching production.

---

## 13. Quick Revision Cheatsheet

```bash
# ─── GIT IN CI: KEY COMMANDS ──────────────────────────────
# Get current SHA
echo $GITHUB_SHA                        # GitHub Actions
echo $CI_COMMIT_SHA                     # GitLab CI

# Shallow clone (fast)
git clone --depth 1 <url>

# Full clone (for tags/changelog)
git clone --depth 0 <url>  # = fetch-depth: 0 in GitHub Actions

# Compare with base branch (find changed files)
git diff --name-only origin/${BASE_BRANCH}...HEAD

# Generate version string from tag
git describe --tags --always --dirty

# Tag and push (release trigger)
git tag -a v2.1.0 -m "Release v2.1.0"
git push origin v2.1.0

# ─── GITHUB ACTIONS PATTERNS ──────────────────────────────
# Trigger on push to main:
on:
  push:
    branches: [main]

# Trigger on tag push:
on:
  push:
    tags: ['v*']

# Trigger on PR:
on:
  pull_request:
    branches: [main]

# Manual trigger:
on:
  workflow_dispatch:

# ─── ENVIRONMENT VARIABLES ────────────────────────────────
# Embed SHA in Docker image:
docker build --label "git.sha=$GITHUB_SHA" ...

# In Kubernetes deployment:
kubectl set image deployment/svc container=image:$GITHUB_SHA
kubectl rollout status deployment/svc --timeout=300s
```

---

## 14. Real-World Pipeline Examples

### Full Java Microservice Pipeline

```yaml
# .github/workflows/pipeline.yml
name: Full Pipeline

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  # ── JOB 1: Build & Test ──────────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Test
        run: mvn verify

      - name: Upload coverage
        uses: codecov/codecov-action@v4
        with:
          files: target/site/jacoco/jacoco.xml

  # ── JOB 2: Security Scan ─────────────────────────────────
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      - name: OWASP Dependency Check
        run: mvn org.owasp:dependency-check-maven:check
      - name: Secret scan
        uses: trufflesecurity/trufflehog@main

  # ── JOB 3: Build Docker Image ────────────────────────────
  build-image:
    needs: [test, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || startsWith(github.ref, 'refs/tags/')
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      - run: mvn package -DskipTests
      
      - name: Build and push image
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            docker login ghcr.io -u ${{ github.actor }} --password-stdin
          
          IMAGE=ghcr.io/company/payment-service
          docker build -t $IMAGE:${{ github.sha }} -t $IMAGE:latest .
          docker push $IMAGE:${{ github.sha }}
          docker push $IMAGE:latest

  # ── JOB 4: Deploy Staging (on main merge) ─────────────────
  deploy-staging:
    needs: build-image
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment:
      name: staging
      url: https://staging.company.com
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/payment-service \
            payment-service=ghcr.io/company/payment-service:${{ github.sha }}
          kubectl rollout status deployment/payment-service -n staging --timeout=5m
        env:
          KUBECONFIG: ${{ secrets.STAGING_KUBECONFIG }}

  # ── JOB 5: Deploy Production (on tag push) ────────────────
  deploy-production:
    needs: build-image
    runs-on: ubuntu-latest
    if: startsWith(github.ref, 'refs/tags/v')
    environment:
      name: production
      url: https://company.com
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate release notes
        run: git cliff --latest --output RELEASE_NOTES.md

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          body_path: RELEASE_NOTES.md

      - name: Deploy to production
        run: |
          TAG=${GITHUB_REF#refs/tags/}
          # Re-tag SHA image with version tag
          docker pull ghcr.io/company/payment-service:${{ github.sha }}
          docker tag ghcr.io/company/payment-service:${{ github.sha }} \
                     ghcr.io/company/payment-service:$TAG
          docker push ghcr.io/company/payment-service:$TAG
          
          kubectl set image deployment/payment-service \
            payment-service=ghcr.io/company/payment-service:$TAG
          kubectl rollout status deployment/payment-service -n production --timeout=5m
        env:
          KUBECONFIG: ${{ secrets.PRODUCTION_KUBECONFIG }}

      - name: Notify on failure
        if: failure()
        run: |
          curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
            -d '{"text":"🚨 Production deploy FAILED for ${{ github.ref }}"}'
```

---

> **Previous:** [11 · Git Troubleshooting & Disaster Recovery](./11_Git_Troubleshooting_and_Disaster_Recovery.md)  
> **Series Complete** ✅
