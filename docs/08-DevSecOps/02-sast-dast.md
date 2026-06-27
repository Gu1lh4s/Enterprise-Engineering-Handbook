# SAST & DAST — Static and Dynamic Application Security Testing

> **Category:** DevSecOps
> **Version:** 1.0.0

---

## Table of Contents

1. [Security Testing Paradigms](#1-security-testing-paradigms)
2. [SAST — Static Application Security Testing](#2-sast--static-application-security-testing)
3. [DAST — Dynamic Application Security Testing](#3-dast--dynamic-application-security-testing)
4. [IAST — Interactive AST](#4-iast--interactive-ast)
5. [SCA — Software Composition Analysis](#5-sca--software-composition-analysis)
6. [Tool Configuration and Integration](#6-tool-configuration-and-integration)
7. [Finding Triage Process](#7-finding-triage-process)
8. [Metrics and SLAs](#8-metrics-and-slas)
9. [Reference Pipeline](#9-reference-pipeline)

---

## 1. Security Testing Paradigms

### When Each Test Type Finds Bugs

```
                  Design → Code → Build → Test → Deploy → Operate
SAST                         ←→  ←→
DAST                                           ←→  ←→
IAST                                      ←→  ←→  ←→
Pentesting                                          ←→        ←→
Bug Bounty                                                     ←→

Shift-Left: Find bugs as early as possible (cheaper to fix)
Cost ratio: Design(1x) → Code(6x) → Test(15x) → Deploy(30x) → Post-release(100x)
```

### Comparison

| | SAST | DAST | IAST | SCA |
|---|---|---|---|---|
| Code needed | Yes | No | Yes | Yes |
| Running app needed | No | Yes | Yes | No |
| Language dependent | Yes | No | Somewhat | Yes |
| False positive rate | High | Medium | Low | Low |
| Runtime bugs | No | Yes | Yes | No |
| Auth logic bugs | Limited | Yes | Yes | No |
| Business logic | No | Limited | Yes | No |
| Speed | Fast | Slow | Medium | Fast |
| CI/CD integration | Easy | Requires env | Complex | Easy |

---

## 2. SAST — Static Application Security Testing

SAST analyzes source code without executing it. Finds patterns associated with security vulnerabilities.

### How SAST Works

```
Source code → Parser → AST (Abstract Syntax Tree) → Taint analysis → Findings

Taint analysis:
- Source: where untrusted data enters (req.body, req.params, req.query)
- Sink: where data is used dangerously (SQL query, innerHTML, eval, exec)
- Taint propagation: track data flow from source to sink
- Finding: if untrusted data reaches a dangerous sink without sanitization

Example:
const username = req.body.username  // Source: tainted input
const query = `SELECT * FROM users WHERE name = '${username}'`  // Sink: SQL injection
db.query(query)  // Tainted data reaches SQL sink → FINDING: SQL injection
```

### Semgrep (Language-Agnostic SAST)

```yaml
# .semgrep/rules.yml — custom rules for your codebase

rules:
  # Detect innerHTML with potentially tainted data
  - id: unsafe-innerhtml
    pattern-either:
      - pattern: $X.innerHTML = $USER_INPUT
      - pattern: $X.innerHTML += $USER_INPUT
    message: "Potential XSS: $USER_INPUT assigned to innerHTML without sanitization"
    severity: WARNING
    languages: [javascript, typescript]
    metadata:
      cwe: CWE-79
      owasp: A03:2021

  # Detect SQL string concatenation
  - id: sql-injection-string-concat
    patterns:
      - pattern: |
          $DB.query($QUERY + $INPUT)
      - pattern: |
          `... ${...} ...`
    message: "Potential SQL injection: user input in SQL string"
    severity: ERROR
    languages: [javascript, typescript]

  # Detect hardcoded passwords
  - id: hardcoded-password
    pattern-either:
      - pattern: password = "..."
      - pattern: secret = "..."
      - pattern: api_key = "..."
    pattern-not-either:
      - pattern: password = ""
      - pattern: password = process.env.$VAR
    message: "Potential hardcoded credential"
    severity: ERROR
    languages: [javascript, typescript, python]

  # Detect eval usage
  - id: no-eval
    pattern-either:
      - pattern: eval($X)
      - pattern: Function($X)
      - pattern: setTimeout($X, ...)
    pattern-not:
      - pattern: setTimeout(function() {...}, ...)
      - pattern: setTimeout(() => {...}, ...)
    message: "Code execution via eval/Function — potential injection"
    severity: WARNING
    languages: [javascript, typescript]
```

```bash
# Run Semgrep locally
semgrep --config=.semgrep/rules.yml --config=p/javascript --config=p/typescript src/

# Run with OWASP Top 10 rules
semgrep --config=p/owasp-top-ten src/

# Run with auto-discovery of language-specific rules
semgrep --config=auto src/

# Output as SARIF (for GitHub Security tab)
semgrep --config=p/javascript --sarif > semgrep.sarif
```

### CodeQL (GitHub's Deep SAST)

```yaml
# .github/workflows/codeql.yml
name: CodeQL Analysis

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # Weekly full scan

jobs:
  analyze:
    name: Analyze
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read

    strategy:
      matrix:
        language: ['javascript']  # Also: python, java, go, csharp, cpp, ruby

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

      - name: Initialize CodeQL
        uses: github/codeql-action/init@4fa2a7953630fd2f3fb380f21be14ede0169dd4f
        with:
          languages: ${{ matrix.language }}
          queries: +security-extended,security-and-quality
          # security-extended: broader coverage
          # security-and-quality: security + maintainability issues

      - name: Autobuild
        uses: github/codeql-action/autobuild@4fa2a7953630fd2f3fb380f21be14ede0169dd4f

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@4fa2a7953630fd2f3fb380f21be14ede0169dd4f
        with:
          category: "/language:${{matrix.language}}"
```

### SAST for Different Languages

```bash
# Python — Bandit
pip install bandit
bandit -r src/ -f json -o bandit-results.json
bandit -r src/ -l -ii  # Only medium-confidence, medium-severity and above

# Go — gosec
gosec ./...
gosec -fmt sarif -out gosec.sarif ./...

# Java — SpotBugs + Find Security Bugs plugin
mvn com.github.spotbugs:spotbugs-maven-plugin:check

# PHP — PHPStan + security extensions
phpstan analyse src/ --level 9

# Ruby — Brakeman
brakeman -o output.json

# .NET — Security Code Scan
dotnet tool install -g security-scan
security-scan MySolution.sln
```

### SAST False Positive Management

SAST generates many false positives. Strategy:

```
1. Start with high-confidence, high-severity findings only
2. Triage each finding once:
   - True positive: fix immediately
   - False positive: add suppression comment
   - Accepted risk: document with business justification

3. Suppression examples:

# Python (Bandit)
password = get_env_var()  # nosec B105 — not a hardcoded password

// JavaScript (Semgrep)
eval(safeExpression)  // nosemgrep: no-eval — safeExpression is whitespace-only

// Java (SpotBugs)
@SuppressFBWarnings(value = "SQL_INJECTION", justification = "Using parameterized query")

4. Track suppression rationale — review annually (suppressed findings may become real)
5. Measure: false positive rate, mean time to fix, finding resolution rate
```

---

## 3. DAST — Dynamic Application Security Testing

DAST tests the running application by sending malicious inputs and observing responses.

### Burp Suite Pro (Gold Standard)

```
Burp Suite workflow:
1. Configure browser proxy → Burp (127.0.0.1:8080)
2. Browse application → Burp records HTTP traffic in Target > Site Map
3. Set scope: include only your application's domain
4. Crawl: Spider or manual navigation
5. Scan: Active Scan sends attack payloads to each parameter

Active Scanner checks:
- SQL injection (16 types)
- XSS (reflected, stored, DOM)
- XXE (XML external entities)
- SSRF
- Path traversal
- OS command injection
- HTTP header injection
- Open redirect
- CORS misconfiguration
- TLS issues
- Information disclosure
```

### OWASP ZAP (Free Alternative)

```bash
# ZAP Docker — automated DAST in CI

# Baseline scan (passive, fast, no attacks)
docker run -v $(pwd):/zap/wrk owasp/zap2docker-stable:latest \
  zap-baseline.py \
  -t https://staging.example.com \
  -J zap-baseline-report.json \
  -r zap-baseline-report.html

# Full scan (active attacks — only on environments you own)
docker run -v $(pwd):/zap/wrk owasp/zap2docker-stable:latest \
  zap-full-scan.py \
  -t https://staging.example.com \
  -J zap-full-report.json

# API scan (OpenAPI spec)
docker run -v $(pwd):/zap/wrk owasp/zap2docker-stable:latest \
  zap-api-scan.py \
  -t /zap/wrk/openapi.json \
  -f openapi \
  -J zap-api-report.json
```

### DAST in CI/CD

```yaml
# GitHub Actions DAST job
dast:
  name: DAST (ZAP)
  runs-on: ubuntu-latest
  needs: [deploy-staging]  # Must run against deployed environment
  
  steps:
    - name: ZAP Baseline Scan
      uses: zaproxy/action-baseline@v0.13.0
      with:
        target: 'https://staging.example.com'
        rules_file_name: '.zap/rules.tsv'  # Suppress known false positives
        fail_action: true                   # Fail build on new MEDIUM+ findings
        allow_issue_writing: true           # Create GitHub issues for findings
    
    - name: Upload ZAP Report
      uses: actions/upload-artifact@v4
      with:
        name: zap-report
        path: report_html.html
```

### Authenticated DAST

```python
# ZAP API — authenticated scan
from zapv2 import ZAPv2

zap = ZAPv2(apikey='your-api-key', proxies={'http': 'http://127.0.0.1:8080', 'https': 'http://127.0.0.1:8080'})

# Log in via Selenium/requests, capture session
import requests

session = requests.Session()
session.post('https://staging.example.com/api/auth/login', json={
    'email': 'test@example.com',
    'password': 'test-password-staging-only'
})

# Get cookies and set in ZAP
cookies = session.cookies.get_dict()
zap.httpsessions.add_session_token('staging.example.com', 'session_id')

# Now scan with authentication
zap.spider.scan('https://staging.example.com')
zap.ascan.scan('https://staging.example.com')
```

---

## 4. IAST — Interactive AST

IAST instruments the running application (agent in the process), monitoring data flows in real time as tests execute.

```
Architecture:
Test runner → Application (with IAST agent) → Findings in real time

IAST agent hooks:
- Input sources: HTTP request parameters, headers, cookies
- Dangerous sinks: SQL queries, HTML output, file system, subprocess
- Tracks data flow in memory (no network overhead)

Products:
- Contrast Security (mature, enterprise)
- Seeker by Synopsys
- Hdiv Security

Advantages over SAST:
- Near-zero false positives (actually traced data flow in execution)
- Business logic aware (knows actual execution context)
- Works with frameworks (not just raw code patterns)

Disadvantages:
- Requires agent in application → performance overhead (~5-15%)
- Must have test coverage to find issues
- Licensing cost
```

---

## 5. SCA — Software Composition Analysis

SCA identifies known vulnerabilities in open source dependencies.

### npm audit / pip-audit / safety

```bash
# npm — built-in audit
npm audit
npm audit --json > audit-report.json
npm audit fix          # Auto-fix compatible updates
npm audit fix --force  # Force upgrades (may break API compatibility)

# Fail CI on high/critical
npm audit --audit-level=high
# Exit code 1 if any finding meets the threshold

# pip — audit
pip install pip-audit
pip-audit
pip-audit --format=json > pip-audit-report.json
pip-audit --vulnerability-service=osv  # Use OSV database

# Go
govulncheck ./...

# Java (Maven)
mvn org.owasp:dependency-check-maven:check

# Ruby
bundle audit check --update
```

### Snyk (SCA + SAST + Container)

```yaml
# Snyk in GitHub Actions
- name: Snyk Security Test
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    args: --severity-threshold=high
    
- name: Snyk Monitor (track in Snyk dashboard)
  uses: snyk/actions/node@master
  env:
    SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
  with:
    command: monitor
```

```bash
# CLI usage
snyk test                    # Test dependencies
snyk test --all-projects     # Monorepo: test all sub-projects
snyk container test my-image:latest  # Test Docker image
snyk iac test infra/         # Test Terraform/Kubernetes
snyk code test               # SAST
snyk monitor                 # Submit to Snyk dashboard for ongoing monitoring
```

### Dependency Update Automation

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "America/Sao_Paulo"
    open-pull-requests-limit: 10
    labels:
      - "dependencies"
      - "security"
    
    # Auto-merge patch updates (safer)
    allow:
      - dependency-type: "all"
    
    # Group related updates to reduce PR noise
    groups:
      aws-sdk:
        patterns:
          - "@aws-sdk/*"
      testing:
        patterns:
          - "jest*"
          - "@jest/*"
          - "vitest*"
    
    # Ignore specific major upgrades until ready
    ignore:
      - dependency-name: "next"
        versions: ["15.x"]  # Test upgrade separately

  - package-ecosystem: "docker"
    directory: "/"
    schedule:
      interval: "weekly"
```

---

## 6. Tool Configuration and Integration

### Centralized Security Results (SARIF)

```yaml
# All security tools output SARIF format → upload to GitHub Security tab
# One unified view of all security findings

# SAST findings (Semgrep, CodeQL, ESLint-security)
# DAST findings (ZAP)
# Container scan findings (Trivy, Grype)
# All visible in GitHub → Security → Code scanning

- name: Upload SARIF results
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: tool-results.sarif
    category: semgrep  # Label for which tool produced this
```

### ESLint Security Plugin

```javascript
// .eslintrc.js
module.exports = {
  plugins: ['security'],
  extends: ['plugin:security/recommended'],
  rules: {
    // No insecure regex (ReDoS)
    'security/detect-unsafe-regex': 'error',
    // No non-literal regexp
    'security/detect-non-literal-regexp': 'error',
    // No eval
    'security/detect-eval-with-expression': 'error',
    // No unsafe deserialization
    'security/detect-unsafe-yaml': 'error',
    // No non-literal fs calls
    'security/detect-non-literal-fs-filename': 'error',
    // No pseudoRandom (use crypto.randomBytes)
    'security/detect-pseudoRandomBytes': 'error',
    // No SQL injection via query string concatenation
    'security/detect-possible-timing-attacks': 'warn',
  }
}
```

---

## 7. Finding Triage Process

```
1. RECEIVE: Tool finds issue → creates GitHub Security alert (GHAS)

2. REVIEW:
   - Is this a true positive or false positive?
   - What is the actual exploitability in context?
   - What is the realistic impact?

3. CLASSIFY:
   Critical:  Exploitable with high impact (RCE, SQLi, auth bypass)
   High:      Exploitable with medium impact (XSS, IDOR, info disclosure)
   Medium:    Exploitable with low impact or theoretical
   Low:       Defense-in-depth or best practice violation
   Informational: No direct exploitability

4. ASSIGN:
   - Critical/High → Security team + engineering team → P1/P2 ticket
   - Medium → Backlog with target sprint
   - Low → Team to track and address opportunistically

5. FIX or ACCEPT:
   - Fix: Implement the remediation
   - Accept: Document business justification (risk acceptance form)
   - False positive: Mark as dismissed with explanation

6. VERIFY:
   - Run the tool again on the fix
   - Manual verification for Critical/High
```

---

## 8. Metrics and SLAs

### Finding Resolution SLAs

| Severity | SLA | Escalation |
|---|---|---|
| Critical | 24 hours | CISO + VP Eng if not met |
| High | 7 days | Security team if not met |
| Medium | 30 days | Team lead if not met |
| Low | 90 days | Track in backlog |

### Security Testing Metrics (Dashboard)

```
Coverage:
- % of repos with SAST configured
- % of repos with dependency scanning
- % of production deployments with DAST gate

Findings:
- Open findings by severity (trend chart)
- Mean time to remediation (MTTR) by severity
- Backlog age (% of findings older than SLA)
- False positive rate per tool (monthly)

Quality:
- New critical/high findings per sprint
- Findings introduced vs. resolved (net trend)
```

---

## 9. Reference Pipeline

```yaml
# Full AppSec pipeline integrated into CI/CD
name: Application Security

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Fast security checks (run first, block PR fast)
  fast-security:
    name: Fast Security Checks
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

      # Secret detection (fastest)
      - name: Secret scan (gitleaks)
        uses: gitleaks/gitleaks-action@v2

      # Dependency audit (fast)
      - name: Dependency audit
        run: npm audit --audit-level=high

      # ESLint security rules
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - name: ESLint security
        run: npx eslint src/ --rulesdir .eslint-rules/ --ext .ts,.tsx --format @microsoft/eslint-formatter-sarif --output-file eslint-security.sarif

      - uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: eslint-security.sarif
          category: eslint-security

  # Deep SAST (longer, not blocking PR but blocking merge to main)
  sast:
    name: SAST (CodeQL + Semgrep)
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write

    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683

      # CodeQL
      - uses: github/codeql-action/init@4fa2a7953630fd2f3fb380f21be14ede0169dd4f
        with: { languages: javascript, queries: +security-extended }
      - uses: github/codeql-action/autobuild@4fa2a7953630fd2f3fb380f21be14ede0169dd4f
      - uses: github/codeql-action/analyze@4fa2a7953630fd2f3fb380f21be14ede0169dd4f

      # Semgrep
      - name: Semgrep
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/javascript
            p/typescript
            p/owasp-top-ten
            p/security-audit
            .semgrep/

  # SCA — Dependency vulnerability (after SAST, as it's slower with NPM audit)
  sca:
    name: SCA (Snyk)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif
      - uses: github/codeql-action/upload-sarif@v3
        with: { sarif_file: snyk.sarif, category: snyk }

  # DAST — only on staging, after deployment
  dast:
    name: DAST (ZAP)
    runs-on: ubuntu-latest
    needs: [deploy-staging]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      - name: ZAP Full Scan
        uses: zaproxy/action-full-scan@v0.11.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: '.zap/rules.tsv'
          fail_action: true
          allow_issue_writing: true
```
