# Supply Chain Security

> **Category:** DevSecOps
> **Version:** 1.0.0
> **Standards:** SLSA v1.0, NIST SP 800-204D, EO 14028

---

## Table of Contents

1. [The Supply Chain Threat Landscape](#1-the-supply-chain-threat-landscape)
2. [Dependency Security](#2-dependency-security)
3. [Build System Security](#3-build-system-security)
4. [Artifact Integrity](#4-artifact-integrity)
5. [SBOM — Software Bill of Materials](#5-sbom--software-bill-of-materials)
6. [Container Supply Chain](#6-container-supply-chain)
7. [Registries and Package Security](#7-registries-and-package-security)
8. [SLSA Framework](#8-slsa-framework)
9. [Incident Response for Supply Chain](#9-incident-response-for-supply-chain)
10. [Checklist](#10-checklist)

---

## 1. The Supply Chain Threat Landscape

### Major Incidents Timeline

```
2020 — SolarWinds (SUNBURST)
  - Attacker compromised SolarWinds' build system
  - Malicious code injected into Orion software update (18,000+ customers)
  - 9 federal agencies and 100 private companies breached
  - Detection: 9 months after initial compromise

2021 — Codecov bash uploader compromise
  - Docker image build leaked credentials to attacker
  - Attacker modified the bash uploader script
  - 23,000+ customers unknowingly ran malicious script in CI
  - Exfiltrated environment variables (including secrets, tokens)

2022 — PyTorch malicious dependency
  - Attacker published "torchtriton" to PyPI before PyTorch did
  - pip dependency resolution chose malicious version over legitimate
  - "Dependency confusion" attack
  - Exfiltrated SSH keys, git configs, environment variables

2024 — XZ Utils (CVE-2024-3094)
  - Multi-year social engineering: attacker gained maintainer trust
  - Backdoor hidden in build system (not source code)
  - Targeted systemd-linked sshd on specific Linux distributions
  - Caught by coincidental performance testing 2 days before release
```

### Attack Surfaces

```
Source code:
  - Malicious PR from external contributor
  - Compromised developer credentials → malicious commit
  - Dependency hijacking (typosquatting, dependency confusion)

Build system:
  - Compromised CI/CD environment → modify build artifacts
  - Leaked secrets from build logs → attacker uses credentials
  - Poisoned caches or build tools

Packages:
  - Malicious package published to npm/PyPI (typosquat or account takeover)
  - Account takeover of popular package maintainer
  - Post-publish modification of artifact on registry

Distribution:
  - CDN compromise → modify served files
  - Mirror compromise
  - DNS hijacking of update servers

Deployment:
  - Unsigned artifact deployed → no integrity check
  - Pull from latest (no version pinning) → pulls malicious update
```

---

## 2. Dependency Security

### Dependency Pinning

```json
// package.json — use exact versions in production (no ^ or ~)
{
  "dependencies": {
    "express": "4.18.2",          // PINNED — exact version
    "lodash": "4.17.21",
    "@supabase/supabase-js": "2.49.0"
  },
  "devDependencies": {
    "jest": "^29.0.0"             // Dev deps can use ranges (lower risk)
  }
}
```

```bash
# Lock files are mandatory — never commit without them
# package-lock.json (npm) or yarn.lock (Yarn) or pnpm-lock.yaml (pnpm)

# Verify lockfile integrity on install (CI)
npm ci  # Instead of npm install — respects lockfile exactly, fails if lockfile out of sync

# Python — pin to exact versions
pip freeze > requirements.txt
pip install -r requirements.txt  # Exact versions

# Poetry lockfile
poetry install --no-dev --no-root  # Use lockfile in CI

# Verify checksums are in lockfile
npm audit --json  # Shows resolved versions and SHA checksums
```

### Dependency Confusion Attack Prevention

```bash
# Problem: Private package "my-company/auth" at internal registry
# Attacker: publishes "auth" to npmjs.com (public)
# pip/npm may resolve to the public one if not configured correctly

# npm — scope packages to internal registry
# .npmrc
@mycompany:registry=https://npm.internal.mycompany.com
//npm.internal.mycompany.com/:_authToken=${NPM_TOKEN}

# Publish all internal packages under a scope
# @mycompany/auth  ← attacker cannot publish to this scope on npmjs.com (you own it)

# Python — restrict to internal index
# pip.conf
[global]
index-url = https://pypi.internal.mycompany.com/simple/
# Do NOT set extra-index-url (vulnerable to confusion)

# Worst practice:
pip install --extra-index-url https://internal.example.com/simple/ auth
# pip resolves to the version with highest version number across ALL indexes
# Attacker publishes auth==9999.0.0 to PyPI → wins the version comparison
```

### Typosquatting Detection

```python
# Common typosquatting patterns:
# cryptography → cryptograhy, cryptographyy, crytpography
# requests → requsts, reuqests, request (singular)
# Beautiful Soup → beautifilsoup4, beautifulsoup, beautifulsoup45

# Check package names before installing
# Tool: pip install pip-audit
# pip-audit detects packages that are similar to popular ones

# npm: npm install npm-diff  -- compare with known-good versions
# Check package metadata: age, downloads, repository URL, author

# Pre-installation checks:
import subprocess
import json

def check_npm_package_safety(package_name: str) -> dict:
    result = subprocess.run(['npm', 'show', package_name, '--json'], capture_output=True, text=True)
    info = json.loads(result.stdout)
    
    return {
        'name': info.get('name'),
        'version': info.get('version'),
        'created': info.get('time', {}).get('created'),
        'downloads_weekly': info.get('downloads'),  # Requires separate API call
        'repository': info.get('repository', {}).get('url'),
        'author': info.get('author'),
        'maintainers_count': len(info.get('maintainers', [])),
    }
```

---

## 3. Build System Security

### Ephemeral Build Environments

```yaml
# Build should run in a clean, ephemeral environment every time
# No persistent state between builds (cache is controlled and verified)

# GitHub Actions: github-hosted runners are ephemeral (new VM each run)
# Self-hosted runners: DANGER — shared state between runs unless hardened

# Self-hosted runner hardening:
# 1. Each job in a separate VM (not container — VMs provide stronger isolation)
# 2. Runner token scoped to repo, not org
# 3. Restrict to private repos only
# 4. No persistent workspace between runs
# 5. Network egress restricted (can only reach known registries, not arbitrary internet)
```

### Build Isolation

```yaml
# Restrict network access during build (prevent exfiltration or malicious downloads)
# --network=none for Docker builds when possible
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Build in isolated container
        run: |
          docker build \
            --network=none \     # No internet access during build
            --no-cache \         # Don't use potentially poisoned cache
            -t app:${{ github.sha }} \
            .
```

### Secrets in Build

```bash
# Never pass secrets as build args — they appear in docker history
docker build --build-arg API_KEY=secret .  # WRONG: visible in image layers

# Use BuildKit secrets (not stored in image layers)
echo "mysecret" | docker buildx build \
  --secret id=api_key,src=- \
  -t myimage .

# In Dockerfile:
RUN --mount=type=secret,id=api_key \
    API_KEY=$(cat /run/secrets/api_key) && \
    run-command-needing-api-key
# Secret is available during build but not in the final image
```

---

## 4. Artifact Integrity

### Signing with sigstore/cosign

```bash
# Generate keyless signature (uses OIDC — no private key to manage)
# Signature recorded in Rekor (public transparency log)

# Sign a container image
cosign sign --yes ghcr.io/myorg/myapp@sha256:abc123...

# Verify the signature
cosign verify \
  --certificate-identity-regexp="https://github.com/myorg/myapp/.github/workflows/" \
  --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
  ghcr.io/myorg/myapp@sha256:abc123...

# Sign a file/binary
cosign sign-blob --yes --output-signature=binary.sig binary
cosign verify-blob \
  --signature=binary.sig \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com \
  binary

# Traditional key-based signing (when keyless isn't suitable)
cosign generate-key-pair  # Creates cosign.key + cosign.pub
cosign sign --key cosign.key ghcr.io/myorg/myapp:latest
cosign verify --key cosign.pub ghcr.io/myorg/myapp:latest
```

### Admit Policy in Kubernetes (Connaisseur / Kyverno)

```yaml
# Kyverno policy: require all images to be signed by cosign
# Only signed images can run in the cluster
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-signed-images
spec:
  validationFailureAction: Enforce
  rules:
    - name: check-image-signature
      match:
        any:
          - resources:
              kinds: [Pod]
      verifyImages:
        - imageReferences:
            - "ghcr.io/myorg/*"
          attestors:
            - count: 1
              entries:
                - keyless:
                    subject: "https://github.com/myorg/*/.github/workflows/*"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev
```

### Checksums and Verification

```bash
# Generate checksums for release artifacts
sha256sum binary-linux-amd64 > SHA256SUMS
sha256sum binary-darwin-arm64 >> SHA256SUMS
gpg --clearsign SHA256SUMS  # Sign the checksum file

# Release: upload SHA256SUMS and SHA256SUMS.asc

# Users verify:
gpg --import maintainer-pubkey.asc
gpg --verify SHA256SUMS.asc SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing

# In CI: verify before using
curl -LO https://releases.example.com/v1.0.0/binary
curl -LO https://releases.example.com/v1.0.0/SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing  # Fails build if hash mismatch
```

---

## 5. SBOM — Software Bill of Materials

An SBOM is a formal, machine-readable inventory of software components.

### SBOM Formats

```
SPDX (Software Package Data Exchange):
- SPDX 2.3 is ISO standard (ISO/IEC 5962:2021)
- Formats: SPDX-TV (tag-value), JSON, RDF, YAML
- Widely supported

CycloneDX:
- OWASP project
- Focused on security use cases (vulnerability mapping)
- Formats: XML, JSON
- Better for VEX (Vulnerability Exploitability eXchange)
```

### Generating SBOMs

```bash
# syft — comprehensive SBOM generator
syft scan ghcr.io/myorg/myapp:latest -o spdx-json=sbom.spdx.json
syft scan /path/to/code -o cyclonedx-json=sbom.cyclonedx.json
syft scan . -o table  # Human readable

# grype — scan SBOM for vulnerabilities
grype sbom:sbom.spdx.json
grype sbom:sbom.spdx.json --fail-on high

# Microsoft sbom-tool (cross-platform)
sbom-tool generate -b . -bc . -pn MyProject -pv 1.0.0 -ps MyOrg

# Node.js
npx @cyclonedx/cyclonedx-npm --output-file sbom.json
cyclonedx-npm --output-format=json --output-file=sbom.json

# Python
pip install cyclonedx-bom
cyclonedx-py --output-format json > sbom.json
```

```yaml
# Attach SBOM as attestation to container image
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: ghcr.io/myorg/myapp@${{ steps.build.outputs.digest }}
    format: spdx-json
    output-file: sbom.spdx.json

- name: Attest SBOM
  run: |
    cosign attest --yes \
      --type spdxjson \
      --predicate sbom.spdx.json \
      ghcr.io/myorg/myapp@${{ steps.build.outputs.digest }}

# Retrieve SBOM from image attestation
cosign download attestation ghcr.io/myorg/myapp:latest | jq -r '.payload' | base64 -d | jq '.predicate'
```

---

## 6. Container Supply Chain

### Base Image Security

```dockerfile
# Pin base image to specific digest (immutable)
# WRONG — tag is mutable
FROM node:20-alpine

# CORRECT — digest is immutable
FROM node:20-alpine@sha256:8f44a6c0b9b9e14d8b0f0e26a19f4e75c6c9e3e7f9e4c0b7e5f8c2d1e4a7b3c6

# Or use a curated base (Google Distroless, Chainguard Images)
FROM cgr.dev/chainguard/node:latest  # Chainguard: minimal, no CVEs, signed

# Check base image for vulnerabilities before building
# Fail if base has CRITICAL CVEs before building your image
FROM node:20-alpine AS check
RUN apk update && apk upgrade  # Keep base updated

# Better: Rebuild base image weekly and run through your scanner
```

### Image Promotion Policy

```
Dev registry ← Build from PR/commit
  ↓ (after scan passes)
Staging registry ← Deployed to staging
  ↓ (after staging tests pass + manual approval)
Production registry ← Deployed to production

Never pull directly from docker hub in production:
- Use a proxy/mirror registry (Harbor, ECR, Artifact Registry)
- Scan images on ingestion into your registry
- Only allow pre-scanned, approved images to run in production

# Harbor proxy cache config:
# Configure Harbor as proxy for docker.io
# All pulls go through Harbor, Harbor scans and caches
```

---

## 7. Registries and Package Security

### Private Registry Setup

```yaml
# npm enterprise / Verdaccio for internal packages
# Package flows: Developer → npm publish → Private Registry → Production build

# verdaccio config.yaml
storage: /verdaccio/storage
auth:
  htpasswd:
    file: /verdaccio/htpasswd
    max_users: 1000

uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    timeout: 30s

packages:
  '@mycompany/*':
    access: $all
    publish: $authenticated
    # No upstream — these are private packages only
  
  '**':
    access: $all
    publish: $authenticated
    proxy: npmjs  # All others: proxy from npmjs

security:
  api:
    jwt:
      sign:
        expiresIn: 29d
```

### npm Token Security

```bash
# Types of npm tokens:
# 1. Automation tokens — for CI (no 2FA required)
# 2. Granular access tokens — per-package, per-IP CIDR, per-environment

# Create automation token (scoped to specific org)
npm token create --cidr=10.0.0.0/8 --read-only  # Read-only + IP restricted

# Token in CI — never commit
# GitHub Actions: store as secret, use as env var
export NPM_TOKEN="${{ secrets.NPM_TOKEN }}"
echo "//registry.npmjs.org/:_authToken=${NPM_TOKEN}" > .npmrc

# Revoke token immediately if exposed
npm token revoke <token>
```

---

## 8. SLSA Framework

SLSA (Supply chain Levels for Software Artifacts) — structured framework for supply chain security.

### SLSA v1.0 Tracks

```
SLSA Build L1:
  - Documentation of build process
  - Build script available
  - Provenance document available (creator, timestamp)

SLSA Build L2:
  - Hosted build platform (not local developer machine)
  - Signed provenance
  - Minimal trust in build platform

SLSA Build L3:
  - Hardened build platform
  - Non-falsifiable provenance
  - Controls against insider attacks
  - Hermetic build (no network access during build)
  - Reproducible build (same inputs → same output, bit-for-bit)
```

### Achieving SLSA Build L2 with GitHub Actions

```yaml
# GitHub Actions is a SLSA L2-capable platform when configured correctly

name: Release (SLSA L2)

on:
  push:
    tags: ['v*']

jobs:
  build:
    outputs:
      hashes: ${{ steps.hash.outputs.hashes }}
    
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      
      - name: Build
        id: build
        run: |
          npm ci
          npm run build
          npm pack  # Creates my-package-1.0.0.tgz
      
      - name: Hash artifacts
        id: hash
        run: |
          HASH=$(sha256sum *.tgz | base64 -w0)
          echo "hashes=$HASH" >> "$GITHUB_OUTPUT"
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v4
        with:
          name: package
          path: '*.tgz'

  provenance:
    needs: [build]
    permissions:
      id-token: write
      contents: write
      actions: read
    
    uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v2.0.0
    with:
      base64-subjects: "${{ needs.build.outputs.hashes }}"
      upload-assets: true
    
    # This generates a signed SLSA provenance document
    # Signed with the build's OIDC identity
    # Published to the release and to Rekor transparency log
```

### Verifying SLSA Provenance

```bash
# Install SLSA verifier
go install github.com/slsa-framework/slsa-verifier/v2/cli/slsa-verifier@latest

# Verify a download
slsa-verifier verify-artifact \
  --provenance-path my-package-1.0.0.intoto.jsonl \
  --source-uri github.com/myorg/myrepo \
  --source-tag v1.0.0 \
  my-package-1.0.0.tgz
```

---

## 9. Incident Response for Supply Chain

### Compromised Dependency Response

```
T+0h: Alert received (npm advisory, GitHub Dependabot, internal report)

T+1h: Assessment
  - What version is affected?
  - Are we using the affected version? (check package-lock.json)
  - What does the malicious code do? (exfiltration, backdoor, cryptominer)
  - Is there a safe version? (upgrade path)

T+2h: Contain
  - Block the malicious package version at registry level (corporate proxy)
  - Stop any running builds using the dependency
  - Rotate any credentials that may have been exfiltrated
  - Block known malicious C2 domains at firewall

T+4h: Remediate
  - Update to safe version (if available)
  - Or remove and replace with alternative
  - Deploy clean builds

T+24h: Review
  - Were any production environments affected?
  - Was any data exfiltrated?
  - Does this trigger breach notification (GDPR/LGPD 72h)?
  - Post-mortem and process improvements

Post-incident:
  - Add blocklist entry for malicious package version in registry proxy
  - Review other dependencies for similar risks
  - Update Dependabot/Renovate to auto-update
```

---

## 10. Checklist

**Dependency Security**
- [ ] All dependencies pinned to exact versions
- [ ] Lockfiles committed and verified in CI (`npm ci`, not `npm install`)
- [ ] `npm audit --audit-level=high` in CI pipeline (blocks on high/critical)
- [ ] Automated dependency updates (Dependabot/Renovate) configured
- [ ] Internal packages scoped to organization name (prevent dependency confusion)
- [ ] Private registry proxy with scanning on ingestion

**Build Security**
- [ ] CI/CD runs on ephemeral environments (new VM per job)
- [ ] No persistent workspace between unrelated builds
- [ ] Secrets passed via env vars from vault, not build args
- [ ] Minimal network access during build (whitelist approach)
- [ ] Build steps pinned to specific SHAs (not mutable tags)

**Artifact Integrity**
- [ ] Container images signed with cosign
- [ ] Non-container artifacts signed with cosign or GPG
- [ ] Checksums generated and published for releases
- [ ] Image signing verified before deployment (Kyverno/OPA policy)
- [ ] SBOMs generated and attached to each release

**SBOM**
- [ ] SBOM generated for each release (SPDX or CycloneDX format)
- [ ] SBOM scanned for vulnerabilities (Grype/osv-scanner)
- [ ] SBOM published with release artifacts

**Registry and Distribution**
- [ ] Production images pulled from internal registry (not public docker hub)
- [ ] Base images scanned and updated weekly
- [ ] Container registry with vulnerability scanning enabled
- [ ] Only signed, scanned images accepted in Kubernetes cluster
