# NIST Frameworks

> **Category:** Compliance / Security Architecture
> **Version:** 1.0.0
> **Level:** Staff Engineer / Security Lead

---

## Table of Contents

1. [NIST CSF 2.0 Overview](#1-nist-csf-20-overview)
2. [Govern Function](#2-govern-function)
3. [Identify Function](#3-identify-function)
4. [Protect Function](#4-protect-function)
5. [Detect Function](#5-detect-function)
6. [Respond Function](#6-respond-function)
7. [Recover Function](#7-recover-function)
8. [NIST SP 800-53 Controls](#8-nist-sp-800-53-controls)
9. [NIST SP 800-63 Digital Identity](#9-nist-sp-800-63-digital-identity)
10. [NIST AI Risk Management Framework](#10-nist-ai-risk-management-framework)
11. [Mapping to ISO 27001](#11-mapping-to-iso-27001)

---

## 1. NIST CSF 2.0 Overview

```
NIST Cybersecurity Framework 2.0 (CSF 2.0) — released February 2024

Audience: ALL organizations (not just critical infrastructure; updated from CSF 1.1)
Purpose: Common language for managing cybersecurity risk

6 Functions (high-level outcomes):
  GV - GOVERN   → Policy, strategy, roles, oversight (NEW in CSF 2.0)
  ID - IDENTIFY → Know your assets, risks, and supply chain
  PR - PROTECT  → Safeguards to limit impact
  DE - DETECT   → Find cybersecurity events
  RS - RESPOND  → Act on detected incidents
  RC - RECOVER  → Restore capabilities after incidents

Tiers (maturity levels):
  Tier 1 (Partial):    Ad-hoc, reactive, no formal process
  Tier 2 (Risk Informed): Risk-aware but not integrated across org
  Tier 3 (Repeatable): Formal policy, consistently applied
  Tier 4 (Adaptive):   Continuously improving, learns from incidents

Relationship to other frameworks:
  CSF maps to → ISO 27001, NIST 800-53, CIS Controls, SOC 2, PCI-DSS
  CSF ≠ compliance; CSF = risk management philosophy
```

---

## 2. Govern Function

```
GV.OC - Organizational Context
  Understand organizational mission, stakeholders, legal/regulatory requirements
  
  Implementation:
    → Document: who are your customers, what data do you hold, what laws apply
    → Map regulatory requirements: GDPR, LGPD, PCI-DSS, HIPAA (if applicable)
    → Define risk appetite: "we accept risks below score X"

GV.RM - Risk Management Strategy
  Priorities, constraints, risk tolerances established and communicated
  
  Implementation:
    → Risk appetite statement: "We will not accept risks with residual score > 15"
    → Risk register reviewed quarterly
    → Board/management sign-off on risk acceptance decisions

GV.RR - Roles, Responsibilities, Authorities
  Cybersecurity roles defined, understood, and authorized
  
  Implementation:
    CISO:         Owns security strategy, reports to CEO/Board
    DPO:          Required under GDPR/LGPD, reports independently
    Security Eng: Implements controls, runs security reviews
    DevSecOps:    Integrates security into development pipeline
    All staff:    Security awareness (AUP signed, annual training)

GV.SC - Cybersecurity Supply Chain Risk
  Third-party risks identified and managed
  
  Implementation:
    → Supplier tier classification (see ISO 27001 §10)
    → Vendor security assessments before onboarding critical suppliers
    → SBOM (Software Bill of Materials) for software dependencies
```

---

## 3. Identify Function

### Asset Inventory (ID.AM)

```typescript
// NIST CSF ID.AM-1: Physical devices and systems inventoried
// NIST CSF ID.AM-2: Software platforms and applications inventoried
// NIST CSF ID.AM-7: Inventories of IT and OT assets (CSF 2.0)

// Asset types to inventory:
const ASSET_CATEGORIES = {
  code_and_software: [
    'Application source code (GitHub repos)',
    'Third-party libraries (package.json, go.mod)',
    'Container images (Docker Hub, ECR)',
    'Infrastructure-as-Code (Terraform, Helm charts)',
  ],
  data: [
    'Customer PII (users table)',
    'Financial data (payment records)',
    'Authentication credentials (hashed passwords)',
    'Session tokens (Redis)',
    'Audit logs (CloudWatch/Loki)',
  ],
  infrastructure: [
    'AWS Account IDs and regions',
    'Kubernetes clusters',
    'Database instances (RDS)',
    'Cache clusters (ElastiCache)',
    'CDN distributions (CloudFront)',
    'DNS zones (Route53)',
  ],
  people: [
    'Engineering team members with production access',
    'Third-party contractors with code access',
    'Privileged accounts (admin, root)',
  ],
}
```

### Risk Assessment (ID.RA)

```
ID.RA-1: Assets with potential vulnerabilities identified
  → SBOM + CVE scanning (Grype, Snyk)
  → Infrastructure vulnerability scanning (Inspector, Tenable)
  → Code vulnerability scanning (CodeQL, Semgrep)
  → Penetration testing (annual minimum)

ID.RA-2: Threat intelligence received, analyzed
  → Subscribe: CISA alerts, vendor security advisories, NVD
  → Internal: monitor for IoCs in logs (SIEM rules)

ID.RA-3: Internal and external threats identified and documented
  → Threat modeling per feature (STRIDE methodology)
  → Annual threat landscape review

STRIDE Threat Modeling:
  S - Spoofing:      Attacker impersonates valid user/system
  T - Tampering:     Unauthorized data modification
  R - Repudiation:   Deny performing an action (→ audit logs)
  I - Info Disclosure: Unauthorized data access
  D - Denial of Service: Availability attacks
  E - Elevation of Privilege: Gain higher access than authorized
```

---

## 4. Protect Function

### Identity Management (PR.AA)

```
PR.AA-1: Identities and credentials managed
  Checklist:
  [ ] All identities in centralized IdP (Okta, Azure AD, Auth0)
  [ ] No shared credentials (each person has unique account)
  [ ] Service-to-service: machine identities (not human accounts)
  [ ] Credentials rotated: API keys (90 days), DB passwords (90 days), TLS certs (auto-renew)
  [ ] Default credentials changed: never use default admin/admin

PR.AA-2: Identities authenticated before access
  [ ] MFA required for: all admin access, production access, privileged accounts
  [ ] MFA type: TOTP (minimum), hardware key (FIDO2/WebAuthn for high-risk)
  [ ] Session management: idle timeout 15 min for admin, JWT expiry 15 min

PR.AA-3: Users, services, hardware authenticated
  Humans:   Username + password + TOTP
  Services: Short-lived tokens (OIDC), mTLS, API keys scoped to specific permissions
  Hardware: Certificate-based authentication, TPM attestation

PR.AA-5: Access permissions managed
  Principle of Least Privilege:
  [ ] Users: access only resources needed for their role
  [ ] Services: IAM roles with minimal permissions (no S3:*)
  [ ] Developers: no standing production access (JIT via Teleport/Boundary)
  [ ] Database: application user has SELECT/INSERT/UPDATE only (not DROP/ALTER)
```

### Awareness and Training (PR.AT)

```
PR.AT-1: All users informed and trained
  Required training:
  [ ] Security awareness (annual): phishing, social engineering, password hygiene
  [ ] Secure coding (for engineers, annual): OWASP Top 10, injection, XSS
  [ ] Data handling (annual): classification, GDPR rights, breach reporting
  [ ] New hire: complete before production access

Phishing simulation:
  → Run quarterly phishing simulations (KnowBe4, Proofpoint)
  → Track click rates; trend downward over time
  → Repeat training for users who fail simulations
  → Target rate: < 5% click rate
```

### Data Security (PR.DS)

```typescript
// PR.DS-1: Data at rest protected
const ENCRYPTION_REQUIREMENTS = {
  database: {
    method: 'AES-256 (AWS RDS encrypted storage)',
    keyManagement: 'AWS KMS with customer-managed key',
    enabled: true,
  },
  backups: {
    method: 'AES-256 (S3 SSE-KMS)',
    keyManagement: 'AWS KMS',
    enabled: true,
  },
  laptops: {
    method: 'FileVault (macOS) / BitLocker (Windows)',
    enforcement: 'MDM policy (Kandji/Jamf/Intune)',
    enabled: true,
  },
  applicationSecrets: {
    method: 'AWS Secrets Manager / HashiCorp Vault',
    keyManagement: 'AWS KMS',
    enabled: true,
  },
}

// PR.DS-2: Data in transit protected
// Enforce TLS 1.2+ everywhere, reject older versions
// nginx: ssl_protocols TLSv1.2 TLSv1.3;
// AWS ALB: Policy ELBSecurityPolicy-TLS13-1-2-2021-06
// Internal service-to-service: mTLS (Istio/Linkerd in Kubernetes)
```

---

## 5. Detect Function

### Anomaly Detection (DE.AE)

```typescript
// DE.AE-2: Anomalies and events analyzed
// DE.AE-3: Event data aggregated and correlated

// SIEM rules (examples in Sigma format for any SIEM)
const DETECTION_RULES = [
  {
    name: 'Brute Force Login Attempt',
    description: 'More than 10 failed logins from same IP in 5 minutes',
    detection: {
      event: 'auth.login.failed',
      threshold: { count: 10, window: '5m', groupBy: 'source_ip' },
    },
    severity: 'medium',
    response: 'Auto-block IP via WAF, alert security team',
  },
  
  {
    name: 'Admin Action Outside Business Hours',
    description: 'Admin role action performed outside 6am-10pm local time',
    detection: {
      event: 'admin.*',
      condition: 'hour(timestamp) < 6 OR hour(timestamp) > 22',
    },
    severity: 'high',
    response: 'Alert on-call, verify with user via separate channel',
  },
  
  {
    name: 'Mass Data Export',
    description: 'Single user exports > 1000 records in 1 hour',
    detection: {
      event: 'data.export',
      threshold: { count: 1000, window: '1h', groupBy: 'user_id' },
    },
    severity: 'high',
    response: 'Alert security team, review export contents',
  },
  
  {
    name: 'Impossible Travel',
    description: 'Same user authenticates from geographically distant locations in short time',
    detection: {
      event: 'auth.login.success',
      condition: 'geo_distance(previous_login.ip, current_login.ip) > 5000km AND time_diff < 2h',
    },
    severity: 'critical',
    response: 'Force re-authentication, alert user and security',
  },
]
```

### Continuous Monitoring (DE.CM)

```yaml
# Monitoring checklist (what to detect)
technical_monitoring:
  - Failed authentication attempts (rate and absolute count)
  - Privilege escalation events
  - New privileged account created
  - Production config changes
  - Database schema changes
  - Outbound connections to new destinations
  - Certificate expiry (30, 14, 7, 1 day warnings)
  - Dependency vulnerabilities (new CVEs for your dependencies)
  - Cloud resource changes (CloudTrail anomalies)

application_monitoring:
  - Error rate spike (> 5%)
  - Latency spike (p99 > 3x baseline)
  - New endpoints appearing in access logs
  - Large payload sizes (potential exfiltration)
  - Cross-tenant data access attempts (RLS violations)

supply_chain_monitoring:
  - New package versions with high CVEs
  - Unexpected package registry changes
  - CI pipeline integrity (SHA pin violations)
```

---

## 6. Respond Function

```
RS.MA - Incident Management
  RS.MA-1: Incidents reported by users and automated systems investigated
  RS.MA-2: Incidents contained
  RS.MA-3: Incidents eradicated
  RS.MA-4: Incidents recovered from (coordinate with RC)
  RS.MA-5: Post-incident activities performed

Incident Playbooks (by type):
  Playbook 1: Phishing email received
  Playbook 2: Ransomware detected
  Playbook 3: Data breach / unauthorized data access
  Playbook 4: DDoS attack
  Playbook 5: Compromised credentials
  Playbook 6: Insider threat
  Playbook 7: Supply chain compromise (compromised dependency)

RS.CO - Incident Communication
  Internal:    Security team → Engineering → Management → Board (by severity)
  Customers:   If data affected → notification per contract SLA and law
  Regulators:  GDPR: 72h to supervisory authority; LGPD: reasonable timeframe
  Law enforcement: If criminal activity suspected
  Public:      Only if legally required or reputationally necessary (PR involved)
```

---

## 7. Recover Function

```
RC.RP - Incident Recovery Plan
  RC.RP-1: Recovery plan executed during and after incident
    → Documented playbooks per incident type
    → Recovery steps tested (tabletop exercises)
  
  RC.RP-3: Recovery activities and progress communicated to stakeholders
    → Status page (status.example.com) for customer-facing incidents
    → Internal Slack channel for engineering updates
    → Stakeholder mailing list for major incidents

RC.RP-4: Recovery plan updated based on lessons learned
  → Blameless post-mortem within 72h
  → Action items tracked to closure
  → Plan updated before next quarter

Recovery Priorities:
  1. Contain: stop ongoing damage first
  2. Eradicate: remove root cause
  3. Recover: restore service in safe state
  4. Validate: verify no residual compromise
  5. Monitor: heightened monitoring post-recovery
```

---

## 8. NIST SP 800-53 Controls

```
NIST SP 800-53 Rev 5: Security and Privacy Controls for Information Systems
Purpose: Control catalog for US federal systems (also widely adopted commercially)

Control families (most relevant for SaaS):

AC - Access Control (26 controls)
  AC-2:  Account Management → JML process, quarterly reviews
  AC-3:  Access Enforcement → RBAC, RLS
  AC-6:  Least Privilege → minimal permissions, no standing admin access
  AC-17: Remote Access → VPN/Teleport required, MFA enforced

AU - Audit and Accountability (16 controls)
  AU-2:  Event Logging → what events to log
  AU-9:  Protection of Audit Information → immutable logs (CloudWatch log groups)
  AU-12: Audit Record Generation → logs for all security-relevant events

CM - Configuration Management (12 controls)
  CM-7:  Least Functionality → disable unused features, ports, services
  CM-8:  System Component Inventory → SBOM, asset register

IA - Identification and Authentication (13 controls)
  IA-5:  Authenticator Management → password policy, MFA
  IA-8:  Non-Organizational Users → external user authentication standards

SC - System and Communications Protection (51 controls)
  SC-8:  Transmission Confidentiality → TLS everywhere
  SC-12: Cryptographic Key Establishment → KMS, key rotation
  SC-28: Protection of Information at Rest → encryption at rest

SI - System and Information Integrity (23 controls)
  SI-3:  Malware Protection → AV on endpoints, container scanning
  SI-4:  System Monitoring → SIEM, anomaly detection
  SI-10: Information Input Validation → input validation at all boundaries

Control impact levels:
  LOW:     Loss of CIA would have limited adverse effect
  MODERATE: Serious adverse effect
  HIGH:    Severe or catastrophic adverse effect
  
Most SaaS: MODERATE impact (handles PII, revenue-generating)
US federal systems: often HIGH or MODERATE
```

---

## 9. NIST SP 800-63 Digital Identity

```
NIST SP 800-63-3: Digital Identity Guidelines
Four documents:
  800-63A: Enrollment and Identity Proofing (IAL)
  800-63B: Authentication and Lifecycle (AAL)
  800-63C: Federation and Assertions (FAL)

Authenticator Assurance Levels (AAL):
  AAL1: Single-factor (password or OTP)
        → Risk: low; most consumer apps
  
  AAL2: MFA required
        → Risk: moderate; business apps with sensitive data
        → Implementation: password + TOTP OR password + hardware key
        → Our default: AAL2 for all users
  
  AAL3: Hardware cryptographic authenticator (FIDO2/WebAuthn)
        → Risk: high; admin access, financial systems, healthcare
        → Implementation: security key (YubiKey) only
        → Our implementation: admin access requires WebAuthn

Password Policy (updated NIST guidance — very different from old rules):
  DO:
    ✓ Minimum 8 characters (15+ recommended for admin)
    ✓ Check against known breached passwords (HaveIBeenPwned API)
    ✓ Allow long passphrases (support 64+ character passwords)
    ✓ Allow all printable ASCII + Unicode
    ✓ Offer password strength meter
  
  DON'T (outdated, counterproductive):
    ✗ Mandatory periodic rotation (causes weaker passwords: Passw0rd1 → Passw0rd2)
    ✗ Complexity requirements (leads to predictable patterns)
    ✗ Password hints (attackers use them)
    ✗ Security questions (guessable, not memorable)
    ✗ Truncate passwords (allow full length)
```

```typescript
// NIST 800-63B compliant password check
async function validatePassword(password: string, userEmail: string): Promise<ValidationResult> {
  const errors: string[] = []
  
  if (password.length < 8) {
    errors.push('Minimum 8 characters required')
  }
  
  if (password.length > 64) {
    errors.push('Maximum 64 characters allowed')
  }
  
  // NIST requirement: check against known breached passwords
  const isBreached = await checkHaveIBeenPwned(password)
  if (isBreached) {
    errors.push('This password has appeared in data breaches. Choose a different password.')
  }
  
  // Don't allow password = email or username (context-specific)
  if (password.toLowerCase().includes(userEmail.toLowerCase().split('@')[0])) {
    errors.push('Password cannot contain your email address')
  }
  
  return { valid: errors.length === 0, errors }
}

async function checkHaveIBeenPwned(password: string): Promise<boolean> {
  // k-Anonymity: only send first 5 chars of SHA1 hash
  const hash = crypto.createHash('sha1').update(password).digest('hex').toUpperCase()
  const prefix = hash.slice(0, 5)
  const suffix = hash.slice(5)
  
  const response = await fetch(`https://api.pwnedpasswords.com/range/${prefix}`)
  const body = await response.text()
  
  return body.split('\r\n').some(line => line.startsWith(suffix))
}
```

---

## 10. NIST AI Risk Management Framework

```
NIST AI RMF 1.0 (January 2023)
Purpose: Manage risks for AI systems throughout their lifecycle

Audience: AI developers, deployers, and operators
Context: Increasingly referenced in AI regulations worldwide

Core structure: GOVERN → MAP → MEASURE → MANAGE

AI Risk Categories:
  Accuracy & Performance:   Model drift, training-serving skew, edge cases
  Bias & Fairness:          Disparate impact on protected classes
  Privacy:                  PII in training data, inference attacks
  Security:                 Prompt injection, model extraction, adversarial inputs
  Transparency:             Explainability, auditability of AI decisions
  Safety:                   Physical/psychological harm from AI outputs
  Robustness:               Performance under distribution shift
  Accountability:           Who is responsible for AI decisions

For AI/ML features in SaaS:
  1. Document AI use cases and risk tier (low/medium/high)
  2. Bias testing before deployment (demographic parity, equalized odds)
  3. Explainability: log why model made decision (for appeals)
  4. Human oversight: high-risk decisions require human review
  5. Model monitoring: drift detection, performance degradation alerts
  6. Incident process: what happens when AI makes wrong/harmful decision
```

---

## 11. Mapping to ISO 27001

```
NIST CSF ↔ ISO 27001 Control Mapping (selected):

NIST CSF             ISO 27001:2022 Annex A
─────────────────────────────────────────────────────────────────────
GV.OC (Context)      5.1 Policies, 5.4 Management responsibilities
GV.RM (Risk Mgmt)    Clause 6.1 Risk management
ID.AM (Assets)       5.9 Inventory of assets
ID.RA (Risk Assess)  Clause 6.1.2 Risk assessment
PR.AA (Identity)     8.2 Privileged access, 5.16 Identity management
PR.AT (Training)     6.3 Awareness, 6.4 Competence
PR.DS (Data Sec)     8.24 Cryptography, 5.33 Data protection
PR.PS (Secure Config) 8.9 Configuration management, 8.8 Tech vuln mgmt
DE.AE (Anomalies)    8.16 Monitoring, 5.25 Assessment of info events
DE.CM (Monitoring)   8.15 Logging, 8.16 Monitoring activities
RS.MA (Incident)     5.26 Response to incidents, 5.27 Learning from incidents
RS.CO (Comms)        5.26 Response to information security incidents
RC.RP (Recovery)     5.30 ICT readiness for business continuity

Dual-framework strategy:
  Pursuing ISO 27001? → Use NIST CSF as your risk language
  Map both: CSF gives structure; ISO 27001 gives auditability
  Many certifiers accept CSF evidence for ISO 27001 controls
```
