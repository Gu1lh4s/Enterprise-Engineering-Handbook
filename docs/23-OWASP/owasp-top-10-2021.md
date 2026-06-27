# OWASP Top 10 — 2021

> **Category:** Security
> **Version:** 1.0.0
> **Source:** OWASP Top 10 2021 (official release)
> **Next update cycle:** 2024/2025 (verify owasp.org)

---

## Overview

The OWASP Top 10 is the most widely referenced application security standard. Updated every 3–4 years by community data, it represents the most critical web application security risks based on:

- Incidence data from 500,000+ applications tested by 40+ partner organizations
- 8 categories from frequency data
- 2 categories from community survey (emerging threats)

### 2021 vs 2017 Changes

| 2021 | 2017 |
|---|---|
| A01: Broken Access Control (↑#5) | A1: Injection |
| A02: Cryptographic Failures (↑#3) | A2: Broken Authentication |
| A03: Injection (↓#1, includes XSS) | A3: Sensitive Data Exposure |
| A04: Insecure Design (new) | A4: XML External Entities |
| A05: Security Misconfiguration (↑#6) | A5: Broken Access Control |
| A06: Vulnerable Components (↑#9) | A6: Security Misconfiguration |
| A07: Auth & Identification Failures (↓#2) | A7: XSS |
| A08: Software Integrity Failures (new) | A8: Insecure Deserialization |
| A09: Logging & Monitoring Failures (↑#10) | A9: Using Components with Known Vulnerabilities |
| A10: Server-Side Request Forgery (new) | A10: Insufficient Logging & Monitoring |

---

## A01: Broken Access Control

**Previously:** #5 (2017) → **#1** (2021, most common)

**What it is:** Access control enforces policies to prevent users from acting outside their intended permissions. Broken access control means users can act as other users, access admin functions, or read/modify other users' data.

### Common Manifestations

```
1. IDOR — Insecure Direct Object Reference
   GET /api/invoices/1234  → changes ID to 1235 → gets another user's invoice

2. Missing Function Level Access Control
   GET /admin/users → frontend hides the link but the endpoint is accessible
   DELETE /api/users/5 → no admin check on the endpoint

3. Privilege Escalation
   POST /api/profile with {"role": "admin"} → accepted by server

4. JWT claim tampering
   Modify JWT payload without signature verification (if alg=none accepted)

5. CORS misconfiguration
   Allowing arbitrary origins to make credentialed cross-site requests

6. Directory traversal
   GET /api/files?name=../../../etc/passwd
```

### Detection and Prevention

```typescript
// IDOR Prevention — always scope queries to authenticated user
// WRONG
app.get('/api/orders/:id', auth, async (req, res) => {
  const order = await Order.findById(req.params.id)  // Anyone's order
  res.json(order)
})

// CORRECT
app.get('/api/orders/:id', auth, async (req, res) => {
  const order = await Order.findOne({
    _id: req.params.id,
    userId: req.user.id  // Always filter by owner
  })
  if (!order) return res.status(404).json({ error: 'Not found' })
  res.json(order)
})

// Missing Function Level Access Control — explicit authorization
function requireRole(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user?.role)) {
      return res.status(403).json({ error: 'Forbidden' })
    }
    next()
  }
}

app.delete('/api/users/:id', auth, requireRole('admin'), async (req, res) => {
  await User.deleteById(req.params.id)
  res.json({ success: true })
})
```

**Testing:**
```bash
# IDOR test — change the ID and observe response
curl -H "Authorization: Bearer $USER_A_TOKEN" https://api.example.com/api/orders/1234
curl -H "Authorization: Bearer $USER_A_TOKEN" https://api.example.com/api/orders/1235  # User B's order

# Admin endpoint access with non-admin token
curl -H "Authorization: Bearer $USER_TOKEN" https://api.example.com/admin/users
```

**CWE:** CWE-284, CWE-285, CWE-639
**Incidence rate:** 94% of applications tested had some form of broken access control

---

## A02: Cryptographic Failures

**Previously:** A3 "Sensitive Data Exposure" → Renamed to highlight root cause

**What it is:** Failures in cryptography (or lack thereof) that lead to exposure of sensitive data. Includes weak algorithms, improper key management, unencrypted data in transit or at rest.

### Attack Scenarios

```
1. Cleartext protocol — HTTP instead of HTTPS
   Network attacker intercepts login credentials in transit

2. Weak algorithms
   MD5/SHA1 password hashing → rainbow table attack → credentials recovered
   RC4 encryption → ciphertext analysis → decryption
   DES/3DES → brute force → decryption

3. Missing encryption at rest
   Database backup stolen → PII/PAN data exposed in plaintext

4. Hard-coded secrets
   API key in source code → pushed to GitHub → found by attacker

5. Insufficient key management
   Same key used for years → key exposure = historical data compromised

6. Improper certificate validation
   Code that ignores TLS errors → man-in-the-middle attack

7. Weak random number generation
   Math.random() for session tokens → predictable, guessable
```

### Cryptographic Controls

```typescript
// Cryptographically secure random — Node.js
import crypto from 'crypto'

const sessionId = crypto.randomBytes(32).toString('hex')      // 256-bit session ID
const apiKey = crypto.randomBytes(24).toString('base64url')   // URL-safe API key
const resetToken = crypto.randomBytes(32).toString('hex')     // Password reset token

// NEVER use Math.random() for security-sensitive values
// Math.random() → NOT cryptographically secure

// Symmetric encryption — AES-256-GCM (authenticated encryption)
function encrypt(plaintext: string, key: Buffer): { ciphertext: string; iv: string; tag: string } {
  const iv = crypto.randomBytes(12)  // 96-bit IV for GCM
  const cipher = crypto.createCipheriv('aes-256-gcm', key, iv)
  
  let ciphertext = cipher.update(plaintext, 'utf8', 'hex')
  ciphertext += cipher.final('hex')
  const tag = cipher.getAuthTag().toString('hex')
  
  return {
    ciphertext,
    iv: iv.toString('hex'),
    tag,
  }
}

function decrypt(encrypted: { ciphertext: string; iv: string; tag: string }, key: Buffer): string {
  const decipher = crypto.createDecipheriv(
    'aes-256-gcm',
    key,
    Buffer.from(encrypted.iv, 'hex')
  )
  decipher.setAuthTag(Buffer.from(encrypted.tag, 'hex'))
  
  let plaintext = decipher.update(encrypted.ciphertext, 'hex', 'utf8')
  plaintext += decipher.final('utf8')
  return plaintext
}

// Key derivation from password (for encryption key)
function deriveKey(password: string, salt: Buffer): Buffer {
  return crypto.pbkdf2Sync(password, salt, 600000, 32, 'sha256')
}
```

**TLS Configuration (Nginx):**
```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305';
ssl_prefer_server_ciphers off;  # Let client choose in TLS 1.3
ssl_session_timeout 1d;
ssl_session_cache shared:MozSSL:10m;
ssl_stapling on;
ssl_stapling_verify on;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

**CWE:** CWE-259, CWE-327, CWE-331
**Incidence:** 67% of applications tested

---

## A03: Injection

**Previously:** #1 (now includes XSS which was A7)

**What it is:** Injection occurs when untrusted data is sent to an interpreter as part of a command or query. Subtypes: SQL, NoSQL, OS command, LDAP, XPath, HTML, ORM injection.

See dedicated documents:
- [SQL Injection — Full Reference](01-sql-injection.md)
- [Cross-Site Scripting — Full Reference](02-xss.md)

### OS Command Injection

```typescript
// VULNERABLE
import { exec } from 'child_process'

app.post('/api/convert', async (req, res) => {
  const { filename } = req.body
  exec(`convert ${filename} output.jpg`, ...)  // Shell injection via filename
  // filename = 'image.png; rm -rf /'
})

// SECURE — use execFile with argument array (no shell)
import { execFile } from 'child_process'

app.post('/api/convert', async (req, res) => {
  const { filename } = req.body
  
  // Validate filename — only allow safe characters
  if (!/^[\w\-. ]+$/.test(filename)) {
    return res.status(400).json({ error: 'Invalid filename' })
  }
  
  execFile('convert', [filename, 'output.jpg'], ...)  // No shell interpolation
})
```

### LDAP Injection

```typescript
// VULNERABLE
const filter = `(&(uid=${username})(password=${password}))`
client.search(base, { filter }, callback)
// username = *)(uid=*)  → bypasses authentication

// SECURE — escape LDAP filter metacharacters
function escapeLdapFilter(value: string): string {
  return value
    .replace(/\\/g, '\\5c')
    .replace(/\*/g, '\\2a')
    .replace(/\(/g, '\\28')
    .replace(/\)/g, '\\29')
    .replace(/\0/g, '\\00')
}

const filter = `(&(uid=${escapeLdapFilter(username)})(password=${escapeLdapFilter(password)}))`
```

---

## A04: Insecure Design

**New in 2021** — distinct from implementation bugs; flaws in architecture and design

**What it is:** Design flaws that are not implementation mistakes — wrong assumptions about trust, missing controls, or business logic flaws baked in from the start. Cannot be fixed by secure coding alone; require redesign.

### Common Design Failures

```
1. Missing rate limiting on credential recovery (password reset)
   → Allow unlimited OTP attempts → brute force 6-digit code (10^6 attempts)

2. Trust boundary violations
   → Mobile app sends user_id in request; server trusts it without verification
   → API assumes request came from the UI because it "looks right"

3. Business logic flaws
   → Negative quantity in shopping cart → refund without purchase
   → Discount codes with no uniqueness or rate limiting
   → Multi-step workflow can skip steps (start checkout, skip payment, confirm order)

4. Missing security controls for sensitive operations
   → Password change does not require current password
   → Email change does not verify new email
   → Account deletion is instant with no grace period

5. Insecure default configuration
   → Admin panel enabled in production
   → Debug endpoints exposed (/actuator, /.env, /debug)
```

### Threat Modeling (Integration into Design)

**STRIDE model:**
```
S — Spoofing (authentication): Who is making this request? Can they prove it?
T — Tampering (integrity): Can data be modified in transit? At rest?
R — Repudiation (non-repudiation): Can actions be denied? Is there an audit trail?
I — Information Disclosure (confidentiality): What data is exposed?
D — Denial of Service (availability): Can the system be overwhelmed?
E — Elevation of Privilege (authorization): Can users gain higher permissions?
```

**For each user story, ask:**
1. What can go wrong? (STRIDE analysis)
2. What is the impact if it does?
3. What controls prevent it?
4. How do we verify the controls work?

### Secure by Default Design Principles

```
1. Principle of Least Privilege
   → Users, processes, services get only the permissions they need
   → DB user for web app cannot DROP TABLE

2. Defense in Depth
   → Multiple layers; no single control is relied upon
   → Input validation + parameterized queries + WAF + anomaly detection

3. Fail Securely
   → On error, default to deny (not allow)
   → If auth check throws exception, deny access (not grant)

4. Complete Mediation
   → Every access to every object is checked (not just at login)
   → Server validates permissions on every request, not just first request

5. Separation of Duties
   → No single person/process can complete a high-risk action alone
   → Deploys require two approvals; financial transactions require dual control
```

---

## A05: Security Misconfiguration

**Previously:** #6

**What it is:** Security misconfiguration is the most common issue. Includes default credentials, unnecessary features enabled, verbose error messages, missing patches, insecure cloud storage.

### Common Misconfigurations

```
1. Default credentials
   admin/admin, admin/password, root/root on databases, routers, CMS
   AWS console with no MFA on root account

2. Unnecessary features enabled
   S3 bucket with public list access
   Debug endpoints (/status, /health, /actuator/env) in production
   Directory listing on web server
   TRACE/OPTIONS HTTP methods on API

3. Error messages expose internals
   Stack traces with file paths, DB schema, internal IPs
   SQL error messages exposing table/column names

4. Missing security headers
   No Content-Security-Policy, no X-Frame-Options, no HSTS

5. Cloud misconfiguration
   Publicly accessible RDS instance
   S3 bucket with public write access
   Security groups allowing 0.0.0.0/0 on port 22 (SSH)
   IAM roles with excessive permissions (*:*)

6. Unpatched systems
   Outdated packages with known CVEs
   Unpatched operating systems
```

### Infrastructure as Code Security Checks

```yaml
# Checklist for AWS/GCP/Azure deployments

# S3 — Block all public access
aws s3api put-public-access-block \
  --bucket my-bucket \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# RDS — No public accessibility
# In Terraform:
resource "aws_db_instance" "main" {
  publicly_accessible = false  # Never true in production
  deletion_protection = true
  storage_encrypted   = true
  # ...
}

# Security Group — No SSH from anywhere
resource "aws_security_group_rule" "ssh" {
  type        = "ingress"
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["10.0.0.0/8"]  # VPC CIDR only, never 0.0.0.0/0
}
```

**Detection — automated scanning:**
```bash
# Security headers check
curl -I https://example.com | grep -E "(Strict-Transport|Content-Security|X-Frame|X-Content)"

# Open ports
nmap -sV --script=default example.com

# OWASP ZAP passive scan for misconfigurations
zap-cli active-scan --scanners all https://example.com

# AWS Config rules for continuous compliance
aws configservice put-config-rule --config-rule file://s3-public-access-rule.json
```

---

## A06: Vulnerable and Outdated Components

**Previously:** #9

**What it is:** Using components (libraries, frameworks, OS) with known vulnerabilities. Includes not tracking versions, not scanning for vulnerabilities, not testing compatibility of updates.

### The Software Bill of Materials (SBOM)

```json
// Generate SBOM with npm
{
  "name": "my-app",
  "version": "1.0.0",
  "dependencies": {
    "express": "4.18.2",   // What version?
    "lodash": "4.17.21",   // Any CVEs?
    "axios": "1.6.8"       // Up to date?
  }
}
```

### Automated Vulnerability Scanning

```bash
# npm audit — check for known vulnerabilities
npm audit
npm audit --audit-level=high  # Fail CI on high/critical

# Snyk — deeper analysis with remediation
npx snyk test
npx snyk monitor  # Track in Snyk platform

# GitHub Dependabot
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "security"
    ignore:
      - dependency-name: "lodash"
        versions: ["4.17.21"]  # Known safe version pin

# Python — safety / pip-audit
pip-audit
safety check

# Docker image scanning
docker scout cves my-image:latest
trivy image my-image:latest
```

### Dependency Management Policy

```
1. Track all direct and transitive dependencies (SBOM)
2. Subscribe to security advisories for critical dependencies
3. Automate dependency updates (Dependabot, Renovate)
4. Test updates in staging before production
5. Pin versions in production; range in dev
6. Review LICENSE — GPL contamination risk for commercial products
7. Prefer well-maintained packages (last commit, stars, issues response time)
8. Minimize dependencies — every package is a potential attack surface
```

---

## A07: Identification and Authentication Failures

**Previously:** #2 "Broken Authentication"

**What it is:** Failures that allow attackers to compromise passwords, keys, session tokens, or exploit implementation flaws to assume other users' identities.

See full treatment in [Authentication & Authorization](../07-Security/03-authentication.md).

### Quick Reference

```
Authentication Failures:
- Default/weak passwords allowed
- Ineffective credential stuffing protection
- Plaintext or weakly hashed passwords
- Missing MFA
- URL with session token (exposed in logs, Referer headers)

Session Management Failures:
- No session invalidation on logout
- Session token does not rotate after login (fixation)
- Long-lived session tokens without inactivity timeout
- Guessable session IDs (sequential, MD5 of timestamp)
```

---

## A08: Software and Data Integrity Failures

**New in 2021** — merged CI/CD pipeline attacks with insecure deserialization

**What it is:** Code and infrastructure that does not protect against integrity violations. Includes insecure deserialization, unverified software updates, CI/CD pipeline compromise.

### Insecure Deserialization

```java
// Java — VULNERABLE: deserializing untrusted data
ObjectInputStream ois = new ObjectInputStream(inputStream);
Object obj = ois.readObject();  // Executes code in readObject() of the incoming object
// Attacker can send crafted serialized gadget chain → RCE

// SECURE: use JSON instead of Java serialization
// Or use allow-listing for deserialized class names
ObjectInputStream ois = new ObjectInputStream(inputStream) {
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws ClassNotFoundException, IOException {
        if (!ALLOWED_CLASSES.contains(desc.getName())) {
            throw new InvalidClassException("Unauthorized: ", desc.getName());
        }
        return super.resolveClass(desc);
    }
};
```

```python
# Python — VULNERABLE: pickle deserialization
import pickle
data = request.get_data()
obj = pickle.loads(data)  # Arbitrary code execution via __reduce__

# SECURE: use JSON
import json
data = json.loads(request.get_data())
```

### CI/CD Pipeline Security (Supply Chain Attacks)

```yaml
# GitHub Actions — secure pipeline
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read    # Principle of least privilege
      id-token: write   # For OIDC
    
    steps:
      # Pin third-party actions to commit SHA, not tag
      # Tags are mutable; SHAs are immutable
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2
      
      # Verify integrity of downloaded artifacts
      - name: Verify artifact
        run: |
          sha256sum -c expected-checksums.txt
          gpg --verify artifact.sig artifact.tar.gz
      
      # Sign your artifacts
      - uses: sigstore/cosign-installer@main
      - name: Sign image
        run: cosign sign --yes ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ env.DIGEST }}
```

**SolarWinds-style supply chain attack prevention:**
```
1. Pin dependencies to exact versions (lockfiles)
2. Use private artifact registries (npm proxy, PyPI mirror)
3. Scan all artifacts on ingestion
4. Verify GPG/cosign signatures on packages
5. Require 2+ approvals for dependency updates
6. Isolate CI/CD from production (no direct network path)
7. Use short-lived credentials in pipelines (OIDC, not stored secrets)
```

---

## A09: Security Logging and Monitoring Failures

**Previously:** #10 "Insufficient Logging & Monitoring"

**What it is:** Without logging and monitoring, breaches go undetected. Average breach detection time was 207 days (IBM 2022). Organizations cannot detect attacks in progress, cannot investigate incidents, cannot meet compliance requirements.

### What to Log

```typescript
// Security events that MUST be logged
const SECURITY_EVENTS = [
  'LOGIN_SUCCESS',
  'LOGIN_FAILURE',
  'LOGOUT',
  'PASSWORD_CHANGE',
  'PASSWORD_RESET_REQUEST',
  'PASSWORD_RESET_COMPLETE',
  'MFA_ENABLED',
  'MFA_DISABLED',
  'MFA_FAILURE',
  'ACCOUNT_LOCKED',
  'PERMISSION_DENIED',  // Authorization failures
  'ADMIN_ACTION',       // Any admin operation
  'DATA_EXPORT',        // Bulk data access
  'API_KEY_CREATED',
  'API_KEY_DELETED',
  'SUSPICIOUS_REQUEST', // WAF/rate limiter triggers
]

interface SecurityEvent {
  timestamp: string        // ISO 8601
  eventType: string
  userId?: string
  sessionId?: string
  ip: string
  userAgent: string
  resource?: string
  action?: string
  result: 'success' | 'failure'
  reason?: string          // For failures
  severity: 'info' | 'warning' | 'critical'
}

function logSecurityEvent(event: Omit<SecurityEvent, 'timestamp'>) {
  const entry: SecurityEvent = {
    ...event,
    timestamp: new Date().toISOString(),
  }
  
  // Write to immutable log store (append-only)
  securityLogger.info(JSON.stringify(entry))
  
  // Alert on critical events
  if (event.severity === 'critical') {
    alerting.send({ channel: 'security', ...entry })
  }
}
```

### What NOT to Log

```typescript
// NEVER log these (PII/sensitive data)
const NEVER_LOG = [
  'password',
  'password_hash',
  'credit_card_number',
  'cvv',
  'ssn',
  'passport_number',
  'private_key',
  'secret_key',
  'access_token',   // Log token ID (jti) instead
  'refresh_token',
]

// Redact sensitive fields
function sanitizeForLogging(obj: Record<string, unknown>): Record<string, unknown> {
  const result = { ...obj }
  for (const key of NEVER_LOG) {
    if (key in result) result[key] = '[REDACTED]'
  }
  return result
}
```

### Alerting and Detection

```yaml
# Example alert rules (Grafana/AlertManager)

# Too many failed logins from one IP
- alert: BruteForceDetected
  expr: |
    sum(rate(login_failures_total[5m])) by (ip) > 10
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "Brute force attempt from {{ $labels.ip }}"

# Admin login outside business hours
- alert: AdminLoginOffHours
  expr: |
    login_success_total{role="admin"} and (hour() < 8 or hour() > 20)
  labels:
    severity: warning

# Spike in permission denied errors (scanning activity)
- alert: AuthzSpike
  expr: |
    sum(rate(permission_denied_total[5m])) > 50
  labels:
    severity: critical
```

---

## A10: Server-Side Request Forgery (SSRF)

**New in 2021** — elevated by cloud metadata endpoint attacks

**What it is:** SSRF flaws occur when a web application fetches a remote resource based on user-supplied URL. The attacker can force the server to make requests to internal services, cloud metadata endpoints, or localhost — bypassing firewalls.

### Attack Scenarios

```
1. Cloud metadata (AWS EC2)
   Input URL: http://169.254.169.254/latest/meta-data/iam/security-credentials/
   Server fetches → returns temporary AWS credentials → full cloud account compromise

2. Internal service access
   Input URL: http://internal-api.prod.svc.cluster.local/admin/users
   Server fetches → returns internal admin API response

3. Port scanning via SSRF
   Input URL: http://192.168.1.1:22  → checks if SSH is open
   Input URL: http://192.168.1.1:3306 → checks if MySQL is open

4. File read via file:// scheme
   Input URL: file:///etc/passwd
   Server reads local file and returns contents

5. AWS IMDSv1 → full credential theft
   http://169.254.169.254/latest/meta-data/iam/security-credentials/role-name
   Returns AccessKeyId, SecretAccessKey, Token with expiration
```

### SSRF Prevention

```typescript
import dns from 'dns/promises'
import { URL } from 'url'

const PRIVATE_RANGES = [
  /^127\./,               // Loopback
  /^10\./,                // Private A
  /^172\.(1[6-9]|2\d|3[0-1])\./,  // Private B
  /^192\.168\./,          // Private C
  /^169\.254\./,          // Link-local (AWS metadata)
  /^::1$/,                // IPv6 loopback
  /^fc00:/,               // IPv6 private
  /^fd/,                  // IPv6 private
  /^0\./,                 // This host
]

function isPrivateIp(ip: string): boolean {
  return PRIVATE_RANGES.some(range => range.test(ip))
}

async function safeRequest(urlString: string): Promise<Response> {
  // 1. Parse and validate URL
  let url: URL
  try {
    url = new URL(urlString)
  } catch {
    throw new Error('Invalid URL')
  }
  
  // 2. Only allow http and https schemes
  if (!['http:', 'https:'].includes(url.protocol)) {
    throw new Error('Only HTTP/HTTPS allowed')
  }
  
  // 3. Allowlist approach — only allow specific trusted domains
  const ALLOWED_HOSTS = new Set(['api.trusted.com', 'cdn.trusted.com'])
  if (!ALLOWED_HOSTS.has(url.hostname)) {
    throw new Error(`Host ${url.hostname} not in allowlist`)
  }
  
  // 4. Resolve hostname and check IP is not private
  const addresses = await dns.resolve4(url.hostname)
  for (const ip of addresses) {
    if (isPrivateIp(ip)) {
      throw new Error(`Host resolves to private IP: ${ip}`)
    }
  }
  
  // 5. Make request (no redirects to different domain)
  const response = await fetch(urlString, {
    redirect: 'manual',  // Handle redirects manually to prevent redirect-based bypass
  })
  
  if (response.status >= 300 && response.status < 400) {
    const redirect = response.headers.get('location')
    if (redirect) {
      return safeRequest(redirect)  // Recursive check on redirect URL
    }
  }
  
  return response
}
```

**AWS IMDSv2 (mitigates SSRF impact):**
```bash
# Require IMDSv2 — forces use of session token (PUT request first)
# SSRF via GET cannot get credentials because IMDSv2 requires a PUT first

aws ec2 modify-instance-metadata-options \
  --instance-id i-1234567890abcdef0 \
  --http-tokens required \
  --http-endpoint enabled
```

---

## Remediation Priority Matrix

| Risk | Exploitability | Impact | Fix Difficulty |
|---|---|---|---|
| A01: Broken Access Control | High | Critical | Medium |
| A02: Cryptographic Failures | Medium | Critical | Low |
| A03: Injection | High | Critical | Low |
| A04: Insecure Design | High | High | High |
| A05: Security Misconfiguration | Easy | High | Low |
| A06: Vulnerable Components | Medium | High | Low |
| A07: Auth Failures | Medium | Critical | Medium |
| A08: Integrity Failures | Medium | Critical | High |
| A09: Logging Failures | Low | Medium | Low |
| A10: SSRF | High | Critical | Medium |

---

## Implementation Checklist

**A01 — Access Control**
- [ ] All endpoints require authentication where applicable
- [ ] All authenticated endpoints enforce authorization
- [ ] DB queries filter by authenticated user ID (IDOR prevention)
- [ ] No user-supplied user IDs trusted server-side
- [ ] Admin functions require admin role check

**A02 — Cryptographic Failures**
- [ ] HTTPS enforced (HSTS header, HTTP→HTTPS redirect)
- [ ] Passwords hashed with Argon2/bcrypt
- [ ] Sensitive data encrypted at rest
- [ ] AES-256-GCM for symmetric encryption
- [ ] Secrets not in source code or environment files committed to git

**A03 — Injection**
- [ ] All DB queries use parameterized statements
- [ ] All HTML output is escaped (or uses framework auto-escape)
- [ ] OS commands use execFile with array args (no shell)
- [ ] User input validated before processing

**A04 — Insecure Design**
- [ ] Threat model documented for each major feature
- [ ] Rate limiting on all sensitive operations
- [ ] Password reset/OTP has limited attempts
- [ ] Business logic reviewed for bypass scenarios

**A05 — Security Misconfiguration**
- [ ] Security headers configured (CSP, HSTS, X-Frame-Options)
- [ ] No default credentials in any component
- [ ] Debug endpoints disabled in production
- [ ] Error responses do not expose stack traces or internals
- [ ] Cloud storage is private by default

**A06 — Vulnerable Components**
- [ ] Dependency vulnerability scan in CI pipeline
- [ ] Automated dependency updates (Dependabot/Renovate)
- [ ] No high/critical CVEs in production dependencies
- [ ] Docker base images scanned and updated

**A07 — Auth Failures**
- [ ] MFA available and encouraged
- [ ] Session invalidated on logout (server-side)
- [ ] Brute force protection on login and password reset
- [ ] Secure session cookies (HttpOnly, Secure, SameSite)

**A08 — Integrity Failures**
- [ ] Dependencies pinned to verified versions (lockfiles)
- [ ] Container image signatures verified
- [ ] No deserialization of untrusted data with Java/Python native serializers
- [ ] CI/CD pipeline protected with branch protection and secret scanning

**A09 — Logging Failures**
- [ ] Authentication events logged (success and failure)
- [ ] Authorization failures logged
- [ ] Admin actions logged
- [ ] Logs written to tamper-evident store
- [ ] Alerting configured for critical security events
- [ ] Sensitive data never logged

**A10 — SSRF**
- [ ] User-supplied URLs validated against allowlist
- [ ] Private IP ranges blocked on outbound requests
- [ ] Cloud metadata endpoints protected (IMDSv2, network policy)
- [ ] URL schemes restricted (only http/https)

---

## References

- [OWASP Top 10 2021 Official](https://owasp.org/Top10/)
- [OWASP Testing Guide v4.2](https://owasp.org/www-project-web-security-testing-guide/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [CWE/SANS Top 25 Most Dangerous Software Weaknesses](https://cwe.mitre.org/top25/)
- [NIST SP 800-53 Rev 5 — Security and Privacy Controls](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [IBM Cost of a Data Breach Report 2023](https://www.ibm.com/reports/data-breach)
- [Verizon Data Breach Investigations Report](https://www.verizon.com/business/resources/reports/dbir/)
